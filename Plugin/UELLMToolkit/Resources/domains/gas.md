# GAS Domain — Gameplay Ability System

Operational reference for the Gameplay Ability System — ASC placement, attributes, abilities, effects, tags, replication.

## Hard Rules

1. **GAS executes authoritatively on the server.** `GiveAbility`, `ApplyGameplayEffect*`, `TryActivateAbility*` are server-gated. Check `HasAuthority()` or route through a server RPC.
2. **Never modify attributes directly.** All changes flow through an `FGameplayEffectSpec`. Hardcoded `Health -= X` breaks replication, buffs, immunities, and prediction.
3. **GameplayTags live in `Config/DefaultGameplayTags.ini`.** Runtime `FGameplayTag::RequestGameplayTag` looks them up; tags not declared in ini silently resolve to invalid.
4. **Use `DOREPLIFETIME_CONDITION_NOTIFY` with `REPNOTIFY_Always`** for replicated attributes. Plain `DOREPLIFETIME` drops predicted-vs-actual deltas and breaks UI bindings.
5. **Always `CommitAbility` first** in `ActivateAbility`. If it returns false, `EndAbility` and return — do not execute ability logic.
6. **`AddLooseGameplayTags` does not replicate.** For replicated runtime tags, apply a duration/infinite GE that grants the tag.
7. **Init the ASC on BOTH `PossessedBy` (server) AND `OnRep_PlayerState` (client).** Missing either side = silent no-op on that side.

## Module & Plugin Setup

`.uproject`: `{"Name": "GameplayAbilities", "Enabled": true}`

`Build.cs` dependencies:

```csharp
PublicDependencyModuleNames.AddRange(new[] {
    "GameplayAbilities",  // ASC, UGameplayAbility, UGameplayEffect
    "GameplayTags",       // FGameplayTag, FGameplayTagContainer
    "GameplayTasks"       // Required by ability task system
});
```

Adding these requires a **full rebuild** — live coding won't pick them up. See `domains/code.md` → Build Strategy.

## ASC Placement Decision

| Target | Placement | Why |
|---|---|---|
| Single-player / listen server player | Character | Simpler; ASC dies with pawn (OK — no respawn) |
| Dedicated server / persistent design | PlayerState | Survives respawn; decoupled from pawn identity |
| AI / NPC | Character | No respawn; no persistence needed |

PlayerState is the more future-proof default. Code below shows the PlayerState path; the Character path is a near-identical shrink (ASC + AttributeSet as `ACharacter` subobjects).

## Core Setup (PlayerState Recommended Path)

```cpp
// MyPlayerState.h
class AMyPlayerState : public APlayerState, public IAbilitySystemInterface
{
    GENERATED_BODY()
public:
    AMyPlayerState();
    virtual UAbilitySystemComponent* GetAbilitySystemComponent() const override;

protected:
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category="Abilities")
    TObjectPtr<UAbilitySystemComponent> AbilitySystemComponent;

    UPROPERTY()
    TObjectPtr<UAttributeSet> AttributeSet;
};

// MyPlayerState.cpp
AMyPlayerState::AMyPlayerState()
{
    AbilitySystemComponent = CreateDefaultSubobject<UAbilitySystemComponent>(TEXT("ASC"));
    AbilitySystemComponent->SetIsReplicated(true);
    AbilitySystemComponent->SetReplicationMode(EGameplayEffectReplicationMode::Mixed);
    AttributeSet = CreateDefaultSubobject<UMyAttributeSet>(TEXT("AttributeSet"));
}
```

## The Init Ritual

`InitAbilityActorInfo(Owner, Avatar)` wires the ASC to its owner and avatar. **Call it on both sides.**

```cpp
// Server
void AMyCharacter::PossessedBy(AController* NewController)
{
    Super::PossessedBy(NewController);
    if (AMyPlayerState* PS = GetPlayerState<AMyPlayerState>())
        PS->GetAbilitySystemComponent()->InitAbilityActorInfo(PS, this);
}

// Owning client
void AMyCharacter::OnRep_PlayerState()
{
    Super::OnRep_PlayerState();
    if (AMyPlayerState* PS = GetPlayerState<AMyPlayerState>())
        PS->GetAbilitySystemComponent()->InitAbilityActorInfo(PS, this);
}
```

If the avatar changes at runtime (pawn swap, respawn), re-run `InitAbilityActorInfo` with the new avatar. The Owner stays the PlayerState.

## Attribute Sets

### Declaration

```cpp
UCLASS()
class UMyAttributeSet : public UAttributeSet
{
    GENERATED_BODY()
public:
    UPROPERTY(BlueprintReadOnly, Category="Attributes", ReplicatedUsing=OnRep_Health)
    FGameplayAttributeData Health;
    ATTRIBUTE_ACCESSORS(UMyAttributeSet, Health)

    UPROPERTY(BlueprintReadOnly, Category="Attributes", ReplicatedUsing=OnRep_MaxHealth)
    FGameplayAttributeData MaxHealth;
    ATTRIBUTE_ACCESSORS(UMyAttributeSet, MaxHealth)

    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& Out) const override;
    virtual void PreAttributeChange(const FGameplayAttribute&, float& NewValue) override;
    virtual void PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data) override;

protected:
    UFUNCTION() virtual void OnRep_Health(const FGameplayAttributeData& Old);
    UFUNCTION() virtual void OnRep_MaxHealth(const FGameplayAttributeData& Old);
};
```

### Replication

```cpp
#include "Net/UnrealNetwork.h"

void UMyAttributeSet::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& Out) const
{
    Super::GetLifetimeReplicatedProps(Out);
    DOREPLIFETIME_CONDITION_NOTIFY(UMyAttributeSet, Health,    COND_None, REPNOTIFY_Always);
    DOREPLIFETIME_CONDITION_NOTIFY(UMyAttributeSet, MaxHealth, COND_None, REPNOTIFY_Always);
}

void UMyAttributeSet::OnRep_Health(const FGameplayAttributeData& Old)
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(UMyAttributeSet, Health, Old);
}
```

### Clamping — Two Places, Different Purposes

| Method | When | Use for |
|---|---|---|
| `PreAttributeChange` | Before any change is applied | Display-layer clamps (current ≤ max) |
| `PostGameplayEffectExecute` | After a GE finishes | Base-value clamps + reactive events (death on Health=0) |

```cpp
void UMyAttributeSet::PreAttributeChange(const FGameplayAttribute& Attr, float& NewValue)
{
    Super::PreAttributeChange(Attr, NewValue);
    if (Attr == GetHealthAttribute())
        NewValue = FMath::Clamp(NewValue, 0.f, GetMaxHealth());
}

void UMyAttributeSet::PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data)
{
    Super::PostGameplayEffectExecute(Data);
    if (Data.EvaluatedData.Attribute == GetHealthAttribute())
    {
        SetHealth(FMath::Clamp(GetHealth(), 0.f, GetMaxHealth()));
        // Signal death via tag/event — do NOT destroy the pawn here.
    }
}
```

## Gameplay Abilities

### Skeleton

```cpp
UCLASS()
class UMyAbility : public UGameplayAbility
{
    GENERATED_BODY()
protected:
    virtual void ActivateAbility(
        const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayAbilityActivationInfo ActivationInfo,
        const FGameplayEventData* TriggerEventData) override;
};
```

### Activation Pattern

```cpp
void UMyAbility::ActivateAbility(...)
{
    if (!CommitAbility(Handle, ActorInfo, ActivationInfo))
    {
        EndAbility(Handle, ActorInfo, ActivationInfo, /*bReplicate*/ true, /*bWasCancelled*/ true);
        return;
    }

    // Ability logic here.

    // Instant abilities end here. Async abilities end from an AbilityTask callback.
    EndAbility(Handle, ActorInfo, ActivationInfo, true, false);
}
```

### Granting (Server Only)

```cpp
UPROPERTY(EditAnywhere, BlueprintReadOnly, Category="Abilities")
TArray<TSubclassOf<UGameplayAbility>> DefaultAbilities;

void AMyPlayerState::GiveStartupAbilities()
{
    if (!HasAuthority() || !AbilitySystemComponent) return;
    for (TSubclassOf<UGameplayAbility> Cls : DefaultAbilities)
        AbilitySystemComponent->GiveAbility(FGameplayAbilitySpec(Cls, /*Level*/ 1, INDEX_NONE, this));
}
```

## Activation Paths

**By class:**
```cpp
ASC->TryActivateAbilityByClass(UMyAbility::StaticClass());
```

**By tag (preferred — decouples from class references):**
```cpp
FGameplayTagContainer Tags;
Tags.AddTag(FGameplayTag::RequestGameplayTag(TEXT("Ability.Attack.Melee")));
ASC->TryActivateAbilitiesByTag(Tags);
```

**By input ID (with `FGameplayAbilitySpec` InputID + Enhanced Input):**
```cpp
// Granted with an InputID, invoked from an Enhanced Input action handler:
ASC->AbilityLocalInputPressed(static_cast<int32>(EAbilityInputID::Attack));
ASC->AbilityLocalInputReleased(static_cast<int32>(EAbilityInputID::Attack));
```
IMC/IA details: `contexts/enhanced_input.md`.

## Gameplay Effects

### Duration Policies

| Policy | Behavior | Typical use |
|---|---|---|
| `Instant` | One-shot; modifies base value immediately | Damage, healing |
| `Duration` | Bounded lifetime; reverts on expiry | Temporary buffs/debuffs |
| `Infinite` | Persists until removed manually | Passive auras, state flags |

### Applying a Spec with Dynamic Magnitude (`SetByCaller`)

Hardcoding the magnitude inside the GE asset locks the value. For runtime-calculated damage, use `SetByCaller`:

```cpp
FGameplayEffectContextHandle Ctx = ASC->MakeEffectContext();
Ctx.AddSourceObject(this);

FGameplayEffectSpecHandle Spec = ASC->MakeOutgoingSpec(DamageEffectClass, /*Level*/ 1, Ctx);
if (Spec.IsValid())
{
    Spec.Data->SetSetByCallerMagnitude(
        FGameplayTag::RequestGameplayTag(TEXT("Data.Damage")),
        ComputedDamage);
    ASC->ApplyGameplayEffectSpecToTarget(*Spec.Data.Get(), TargetASC);
}
```

The GE asset must declare `Data.Damage` (or whatever tag you chose) as a `SetByCaller` modifier magnitude — otherwise the value is silently ignored.

### Cost & Cooldown

Set `CostGameplayEffectClass` and `CooldownGameplayEffectClass` on the ability CDO. `CommitAbility` applies both automatically. Do **not** implement cooldowns with `Delay` nodes or timers — they bypass the block/query hooks.

Conventional naming: `GE_Cost_<Resource>_<Amount>`, `GE_Cooldown_<AbilityName>`. Cooldown GEs grant a `Cooldown.<AbilityName>` tag; the ability's `BlockAbilitiesWithTag` references that same tag so `CommitAbility` fails while cooling down.

## Gameplay Tags

### Declare in ini

```ini
[/Script/GameplayTags.GameplayTagsSettings]
+GameplayTagList=(Tag="Ability.Attack.Melee",DevComment="Melee attack")
+GameplayTagList=(Tag="State.Stunned",DevComment="Character is stunned")
+GameplayTagList=(Tag="Cooldown.Fireball",DevComment="Fireball on cooldown")
+GameplayTagList=(Tag="Data.Damage",DevComment="SetByCaller damage magnitude")
```

### Ability CDO Tag Fields

```cpp
UMyAbility::UMyAbility()
{
    AbilityTags.AddTag          (FGameplayTag::RequestGameplayTag(TEXT("Ability.Attack.Melee")));
    ActivationOwnedTags.AddTag  (FGameplayTag::RequestGameplayTag(TEXT("State.Attacking")));
    BlockAbilitiesWithTag.AddTag(FGameplayTag::RequestGameplayTag(TEXT("State.Stunned")));
    ActivationBlockedTags.AddTag(FGameplayTag::RequestGameplayTag(TEXT("State.Dead")));
}
```

### Loose Tags

`AddLooseGameplayTags`/`RemoveLooseGameplayTag` set tags locally without going through a GE. They **do not replicate**. Use for local/debug states only. For replicated runtime tags, apply a duration/infinite GE that grants the tag.

## Replication Modes

Set once on the ASC.

| Mode | Replicates | Use when |
|---|---|---|
| `Full` | Everything, to everyone | Single-player only |
| `Mixed` | Full to owning client, minimal to simulated proxies | Player-owned ASCs |
| `Minimal` | Only tags | AI/NPCs, simulated proxies |

```cpp
ASC->SetReplicationMode(EGameplayEffectReplicationMode::Mixed);
```

## Debugging

### In-game commands

```text
showdebug abilitysystem              # Abilities, effects, tags, attributes on current target
AbilitySystem.Debug.NextCategory     # Cycle detail panels
AbilitySystem.DebugNextTarget        # Next pawn under inspection
```

### Verbose logging

```cpp
LogAbilitySystem.SetVerbosity(ELogVerbosity::VeryVerbose);
```

### Plugin integration

`gameplay_debug run_sequence` can drive input-bound abilities in PIE and capture GAS log output. See `domains/debug.md` for the sequence format. Combine with a `console` step that runs `showdebug abilitysystem` to snapshot ASC state mid-sequence.

## UE 5.7 Notes

- `DOREPLIFETIME_CONDITION_NOTIFY` with `REPNOTIFY_Always` remains required for attributes — older guides using plain `DOREPLIFETIME` drop deltas the prediction system needs.
- Attribute-based PredictionKey handling is unchanged in 5.7.
- `UAbilitySystemBlueprintLibrary::SendGameplayEventToActor` is the canonical path for event-driven activation; older direct paths still work but are less diagnostic-friendly.

## Common Pitfalls

- **Silent no-op on owning client** — forgot `InitAbilityActorInfo` in `OnRep_PlayerState`.
- **Attribute UI freezes or flickers** — used plain `DOREPLIFETIME` instead of `DOREPLIFETIME_CONDITION_NOTIFY` with `REPNOTIFY_Always`.
- **`TryActivateAbility` returns true, nothing happens** — ability activated then ended because `CommitAbility` failed (cost, cooldown, or blocking tag). Check `LogAbilitySystem` verbose.
- **Cooldown never blocks re-activation** — cooldown GE not set on CDO, or its granted tag doesn't match the ability's `BlockAbilitiesWithTag`.
- **`SetByCaller` damage lands as zero** — GE asset didn't declare the tag as a `SetByCaller` modifier magnitude, or caller forgot `SetSetByCallerMagnitude` on the spec.
- **Tags "added" but queries return false** — `AddLooseGameplayTags` on a simulated proxy doesn't replicate. Use a duration/infinite GE instead.
- **ASC appears uninitialized after pawn swap/respawn** — re-run `InitAbilityActorInfo` with the new avatar.
- **`GiveAbility` silently drops** — called without `HasAuthority()`, or before ASC init.
- **Performance cliff with many NPCs** — replication mode set to `Full` or `Mixed` on enemies. Use `Minimal`.
- **Damage GE reverts on its own** — used `Duration` policy instead of `Instant`. Damage must be Instant to modify base value.

## References

- `contexts/replication.md` — DOREPLIFETIME/OnRep semantics, RPC patterns
- `domains/debug.md` — PIE automation via `gameplay_debug run_sequence`
- `domains/code.md` — TObjectPtr/UPROPERTY conventions, build strategy
