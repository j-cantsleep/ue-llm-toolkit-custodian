# Interface Domain — UInterface / IInterface Patterns

Operational reference for Unreal's interface system — `UINTERFACE`/`IInterface` declarations, `Execute_` dispatch, `TScriptInterface` usage, C++ vs Blueprint considerations.

## Hard Rules

1. **Interfaces only work on UObjects.** The implementing class must derive from UObject (directly or transitively).
2. **Declare both halves: `UInterface` and `IInterface`.** The `U`-prefixed class is the reflection stub; the `I`-prefixed class is where methods live. Missing either breaks registration.
3. **Mark `UINTERFACE(Blueprintable)` to allow BP implementation.** Without it, BPs cannot add the interface in Class Settings.
4. **Mark `UINTERFACE(BlueprintType)` to use the interface as a BP variable/parameter type.** Missing this makes `TScriptInterface<IFoo>` properties non-assignable in the editor.
5. **Always call `BlueprintNativeEvent` / `BlueprintImplementableEvent` methods via the `Execute_` prefix.** Direct calls skip the reflection dispatch — they hit the C++ default and silently bypass the BP override.
6. **Use `Implements<UFoo>()` or `ImplementsInterface`, not `Cast<IFoo>`.** `Cast` only sees C++-side implementers; it misses BP-only implementations.
7. **Store interface references as `TScriptInterface<IFoo>`.** Raw `IFoo*` is GC-unsafe and loses BP dispatch.
8. **Pure virtual methods (`= 0`) are C++-only.** A BP class implementing an interface with any pure-virtual method will fail to compile.

## Deciding Whether to Use an Interface

| Situation | Choice |
|---|---|
| Unrelated classes share a behavior (any Actor could be "interactable") | **Interface** |
| Family of related classes share behavior AND data | **Abstract base class** |
| A class needs reusable, composable functionality | **Actor Component** |
| Publishing events to many listeners | **Delegate / Event Dispatcher** |
| "Does this object support X?" queries across a heterogeneous world | **Interface** |
| Cross-system decoupling (UI ↔ gameplay, ability ↔ pawn) | **Interface** (matches `CLAUDE.md` Blueprint Interfaces rule) |

Interfaces are contracts for **behavior**, not storage for data. Put no fields, no state — methods only.

## C++ Interface Pattern

### Declaration

```cpp
// MyInteractable.h
#pragma once
#include "UObject/Interface.h"
#include "MyInteractable.generated.h"

UINTERFACE(MinimalAPI, Blueprintable, BlueprintType, meta=(DisplayName="Interactable"))
class UMyInteractable : public UInterface
{
    GENERATED_BODY()
};

class MYMODULE_API IMyInteractable
{
    GENERATED_BODY()
public:
    // ---- C++ only (no BP override possible) ----
    virtual bool IsReadyForInteraction() const = 0;  // pure virtual

    // ---- BP can override, C++ provides default ----
    UFUNCTION(BlueprintNativeEvent, BlueprintCallable, Category="Interactable")
    void OnInteract(AActor* Instigator);
    virtual void OnInteract_Implementation(AActor* Instigator);

    // ---- BP must implement (no C++ default) ----
    UFUNCTION(BlueprintImplementableEvent, BlueprintCallable, Category="Interactable")
    FText GetInteractPromptText() const;
};
```

### Implementation

```cpp
// MyDoor.h
UCLASS()
class AMyDoor : public AActor, public IMyInteractable
{
    GENERATED_BODY()
public:
    // Override the pure virtual
    virtual bool IsReadyForInteraction() const override { return !bLocked; }

    // Override the BlueprintNativeEvent C++ default
    virtual void OnInteract_Implementation(AActor* Instigator) override;

    // GetInteractPromptText is BlueprintImplementableEvent — implement in a BP subclass only
};
```

### `UINTERFACE` specifiers worth knowing

| Specifier | Meaning |
|---|---|
| `Blueprintable` | BPs can implement this interface |
| `BlueprintType` | Interface can be used as variable / parameter type in BP |
| `MinimalAPI` | Reflection-only exposure; smaller module footprint |
| `meta=(CannotImplementInterfaceInBlueprint)` | C++ implementation only |

## Method Type Decisions

| Declaration | BP can call? | BP can implement? | C++ default possible? |
|---|---|---|---|
| `virtual Foo() = 0` (pure virtual) | No | No | No — must override |
| `virtual Foo()` without specifier | No | No | Yes |
| `UFUNCTION(BlueprintCallable) virtual` | Yes | No | Yes (required) |
| `UFUNCTION(BlueprintNativeEvent)` | Yes | Yes | Yes (in `_Implementation`) |
| `UFUNCTION(BlueprintImplementableEvent)` | Yes | Yes | No (BP-only) |

**Rule of thumb:** prefer `BlueprintNativeEvent` whenever a sensible C++ default exists. `BlueprintImplementableEvent` is correct only when there's no meaningful fallback (e.g., asking a widget for its custom text format).

## Blueprint-Only Interfaces

If no C++ is ever needed:

1. Content Browser → Blueprint Class → Blueprint Interface
2. Add functions with inputs/outputs in the interface asset
3. Target BP → Class Settings → Implemented Interfaces → Add
4. Override in the graph under the "Interfaces" category

BP-only interfaces are fine for BP-only projects but limit reusability. C++ callers can still invoke them via the same `Execute_` pattern — the reflection layer is identical.

A common naming convention is to prefix Blueprint interface assets with `BPI_` to distinguish them from concrete Blueprints.

## Calling Interface Methods

### The `Execute_` prefix — for `BlueprintNativeEvent` / `BlueprintImplementableEvent`

```cpp
UObject* Target = SomeActor;
if (Target && Target->Implements<UMyInteractable>())
{
    // CORRECT — dispatches to BP override if present, else C++ default
    IMyInteractable::Execute_OnInteract(Target, Instigator);
    FText Prompt = IMyInteractable::Execute_GetInteractPromptText(Target);
}
```

**Never call `OnInteract_Implementation` directly on the object.** It skips the reflection dispatch and always runs the C++ version, bypassing any Blueprint override — a silent bug with no compiler warning.

### Pure virtual methods (C++ only)

```cpp
// Cast works here because pure virtuals can't be BP-implemented anyway
if (IMyInteractable* Iface = Cast<IMyInteractable>(Target))
{
    if (Iface->IsReadyForInteraction()) { /* ... */ }
}
```

### Implements<> check

```cpp
// Preferred — sees both C++ and BP implementers
if (Target->Implements<UMyInteractable>()) { ... }

// Equivalent longhand
if (Target->GetClass()->ImplementsInterface(UMyInteractable::StaticClass())) { ... }
```

Note the `U` prefix on `Implements<UMyInteractable>()` — the `I` version won't compile.

## Reference Types and Safety

### `TScriptInterface<IFoo>` — the BP-safe container

```cpp
UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Refs")
TScriptInterface<IMyInteractable> InteractableRef;
```

Holds both the raw `UObject*` and the `IInterface*` pointer. Key benefits:

- Compatible with Blueprint (raw `IFoo*` is not).
- Works with Blueprint-only implementers (raw `IFoo*` can't be obtained from those).
- Safe to `UPROPERTY`-serialize and GC-track.

Access:

```cpp
if (InteractableRef)
{
    // Via Execute_ — BP-override-aware (preferred)
    IMyInteractable::Execute_OnInteract(InteractableRef.GetObject(), this);

    // Direct — C++ virtual dispatch only, skips BP override (usually wrong)
    InteractableRef->OnInteract_Implementation(this);
}
```

### Raw pointers — avoid in `UPROPERTY`

Raw `IFoo*` is not a UObject pointer; the GC can't track it. The owning UObject can be collected with the interface pointer left dangling. Use `TScriptInterface<IFoo>` for anything persisting beyond a stack frame, or re-resolve from the owning `UObject*` each time.

## Multiple Interfaces & Name Disambiguation

A class can implement multiple interfaces. Name collisions must be disambiguated:

```cpp
class AMyActor : public AActor, public IInterfaceA, public IInterfaceB
{
    GENERATED_BODY()
public:
    // Both declare void DoThing(); disambiguate with explicit interface scope:
    virtual void IInterfaceA::DoThing() override { /* A-side */ }
    virtual void IInterfaceB::DoThing() override { /* B-side */ }
};
```

Better: keep interface method names distinct in the first place.

## UE 5.7 Notes

- `GENERATED_BODY()` replaces the old `GENERATED_UINTERFACE_BODY()` / `GENERATED_IINTERFACE_BODY()` pair — use only the modern form.
- `TObjectPtr` is not used inside interface declarations themselves (they're abstract), but any `UPROPERTY` referencing implementers should use `TObjectPtr<T>` for concrete types or `TScriptInterface<IFoo>` for interface polymorphism.
- No interface-pattern changes in 5.7 vs 5.3+; older guides remain structurally valid.
- `Implements<UFoo>()` is preferred over `ImplementsInterface(UFoo::StaticClass())` in 5.x — shorter, equivalent.

## Common Pitfalls

- **`OnFoo` called but BP override never fires.** Caller used `Target->OnFoo_Implementation(...)` instead of `IMyInterface::Execute_OnFoo(Target, ...)`. Direct calls bypass the reflection dispatch.
- **Interface check fails for BP-only implementers.** Used `Cast<IMyInterface>(Target)`. `Cast` only sees C++ implementers; use `Target->Implements<UMyInterface>()` instead.
- **Linker error on an interface method.** Pure-virtual (`= 0`) not implemented on a C++ implementer. Either implement it or use the `PURE_VIRTUAL(Class::Method, )` macro in the header to provide a stub.
- **BP can't add the interface in Class Settings.** `UINTERFACE` missing the `Blueprintable` specifier.
- **`TScriptInterface<IFoo>` property shows no assignable classes in editor.** `UINTERFACE` missing `BlueprintType`.
- **Interface method marked `const` but `_Implementation` isn't (or vice versa).** Signatures must match exactly, including `const`. Mismatch compiles but the override never runs.
- **Changing an interface method's signature silently breaks implementers.** No compiler warning for BP-side signature drift. Audit every implementer after any signature change.
- **Raw `IFoo*` class member causes GC dangling.** Use `TScriptInterface<IFoo>` for persistent references, or re-resolve from the owning `UObject*`.
- **Multi-return BP interface method loses values.** BP represents returns as output pins; declare multiple returns as explicit out-parameters, not via struct-by-value with multiple named fields.
- **`Execute_` symbol not found at call site.** Interface header not included. Include `"MyInterface.h"` — forward-declaration is not sufficient.
- **Interface method category differs between declaration and override.** Editor shows the function in inconsistent places. Keep `Category="..."` consistent.
- **Ambiguous method calls when implementing two interfaces with same name.** Disambiguate with `IInterfaceA::Method` syntax or rename.
- **Forgot `UINTERFACE` / `IInterface` pair; only one declared.** Compile may succeed but registration fails at runtime. Always declare both.
- **Interface method with `TSubclassOf<T>` parameter has T outside the current module.** Add the module dependency; otherwise BP picks up a compilation error with a confusing location.

## Industry Best Practices

- **Narrow interfaces.** 1–5 methods, single focused responsibility. Fat interfaces ("everything a character can do") are rarely implemented uniformly and resist change.
- **Compose over inherit.** Several focused interfaces (`IInteractable`, `IDamageable`, `IInspectable`) beat one large one. Classes pick what they need.
- **Interfaces don't know their implementers.** Dependency always points toward the interface.
- **No data in interfaces.** No fields, no state. Contracts for behavior only. Use a base class or component for shared state.
- **Prefer `BlueprintNativeEvent` over `BlueprintImplementableEvent`** when a sensible default exists. Designers keep override power; C++ covers unhandled cases.
- **Document the contract in header comments.** When is the method called? What invariants hold before and after? Implementers need to know.
- **`TScriptInterface<IFoo>` for parameters crossing module or BP boundaries.** Maximum compatibility.
- **Name interfaces after capability, not category.** `IInteractable`, `IDamageable`, `IFocusable` — not `IEntity` or `IActorStuff`.
- **Version shipped interfaces carefully.** Adding a member forces every implementer to update. For open-ended extension, use a single generic `HandleEvent(FName, FInstancedStruct)` method and dispatch internally.
- **Interfaces for cross-cut behavior, base classes for taxonomy.** Actor kinds → base class hierarchy. Orthogonal capabilities → interfaces.

## References

- `contexts/blueprint.md` — Blueprint graph system reference (complements this file for BP-side workflow; plugin-loaded)
- `domains/code.md` — `UPROPERTY` / `UFUNCTION` / `TObjectPtr` conventions (plugin-loaded)
- `domains/blueprints.md` — node IDs, targeted queries on BP graphs (plugin-loaded)
