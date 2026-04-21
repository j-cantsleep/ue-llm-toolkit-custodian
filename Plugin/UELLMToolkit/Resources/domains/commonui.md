# Common UI Domain — Multiplatform Widget System

Operational reference for Epic's Common UI plugin — input routing, focus, activatable widget lifecycle, platform styling.

## Hard Rules

1. **Plugin must be enabled** in `.uproject`. Module dependency (`CommonUI`, `UMG`, `EnhancedInput`) must be in `Build.cs`. Full rebuild after either change.
2. **Inherit from Common UI base classes, never `UUserWidget`.** Plain UMG widgets bypass input routing and focus management entirely. Base classes: `UCommonUserWidget`, `UCommonActivatableWidget`, `UCommonButtonBase`.
3. **Game Viewport Client Class must be `UCommonGameViewportClient`** (Project Settings → General Settings → Game Viewport Client Class). Input routing dies without this.
4. **Never call `SetInputMode*` on the PlayerController when Common UI is active.** It breaks Common UI's internal state. Use `GetDesiredInputConfig()` on the activatable widget instead.
5. **Input Data / Action Data Table / Controller Data must be wired in Project Settings.** Missing any = buttons show no icons, back/confirm actions don't fire, gamepad nav broken.
6. **Focus is not automatic.** Override `GetDesiredFocusTarget()` on activatable screens; delay focus requests by at least one tick after push.
7. **Activatable widgets are reused, not destroyed.** Don't put per-activation logic in `NativeOnInitialized` or the constructor — use `NativeOnActivated` / `NativeOnDeactivated`, which fire every activation cycle.
8. **Back action requires `BackHandler` enabled on the activatable** AND an override of `NativeOnHandleBackAction()`. Both must be present or back input routes past the widget.

## Module & Plugin Setup

`.uproject`:
```json
{ "Plugins": [{ "Name": "CommonUI", "Enabled": true }] }
```

`Build.cs` dependencies:
```csharp
PublicDependencyModuleNames.AddRange(new[] {
    "CommonUI",        // Widgets, activatable system, input routing
    "CommonInput",     // Input device detection, action bindings
    "UMG",             // Underlying UMG widget framework
    "EnhancedInput"    // For gameplay input side (UI uses Common UI's own action system)
});
```

Adding these requires a **full rebuild**. See `domains/code.md` → Build Strategy.

## Project Settings Checklist

Set all of these before expecting anything to work:

| Setting | Location | Value |
|---|---|---|
| Game Viewport Client Class | Project Settings → General Settings → Game Viewport Client Class | `CommonGameViewportClient` |
| Input Data | Project Settings → Engine → Common Input Settings → Default Input Data | A `UCommonUIInputData`-derived asset (custom) |
| Action Data Table | Set on the `UCommonUIInputData` asset | `UDataTable` with row type `CommonInputActionDataBase` |
| Controller Data | Project Settings → Engine → Common Input Settings → Default Classes | One `UCommonInputBaseControllerData` per supported platform |

### Input Data asset shape

Create a new Blueprint class deriving from `CommonUIInputData`. It references:
- **Default Click Action** — row name in the action data table, typically `"DefaultClick"`
- **Default Back Action** — row name in the action data table, typically `"DefaultBack"`
- **Action Data Table** — the `UDataTable` asset itself

### Action Data Table

Row type: `CommonInputActionDataBase`. Each row has:
- Display Name (localized)
- Icons per input type (keyboard, gamepad, touch)
- Key bindings per input type
- Holdable flag
- Key for navigation flag

Minimum rows: `DefaultClick`, `DefaultBack`. Add project-specific rows as needed.

## Class Hierarchy

| Class | Role |
|---|---|
| `UCommonUserWidget` | Base for non-activatable widgets that still need Common UI routing (HUD elements, subwidgets) |
| `UCommonActivatableWidget` | Base for any screen/panel that can be pushed onto a stack, focused, or handle back action |
| `UCommonButtonBase` | Button with Common UI features (focus-aware triggering, action binding, hold-to-trigger) |
| `UCommonActionWidget` | Displays the current platform's icon+label for a given action row |
| `UCommonBoundActionButton` | Button that binds to a named action and displays its icon |
| `UCommonBoundActionBar` | Container that shows all active activatable widgets' action bindings |
| `UCommonActivatableWidgetStack` | Stack container; all widgets stay alive, topmost is active |
| `UCommonActivatableWidgetQueue` | Queue container; one at a time, previous destroyed before next shown |
| `UCommonInputSubsystem` | Runtime access to current input type, platform, broadcast on device changes |
| `UCommonGameViewportClient` | Required viewport class; routes input through Common UI before falling through |

## Activatable Widget Lifecycle

```
NativeOnInitialized     ← once per widget instance lifetime (rare; widget is reused)
NativeConstruct         ← when added to viewport/widget tree
  [ActivateWidget]       ← push onto stack triggers this; fires OnActivated delegates
    NativeOnActivated    ← override point; focus, bind dynamic data, start animations
  [DeactivateWidget]     ← pop or sibling push triggers this
    NativeOnDeactivated  ← override point; unbind, stop animations
NativeDestruct          ← when actually removed from tree (rare)
```

**Reuse behavior:** Stacks keep activatable widgets alive when covered. Expect `OnActivated`/`OnDeactivated` to fire multiple times during a widget instance's life. Don't put "first-time" initialization in `OnActivated` without guarding it.

## Input Routing Model

```
Hardware input
   ↓
PlayerController / viewport
   ↓
UCommonGameViewportClient      ← required; otherwise skips Common UI
   ↓
UCommonInputSubsystem          ← identifies input type, dispatches to action system
   ↓
Active activatable stack       ← topmost visible activatable gets priority
   ↓
Focused widget                 ← only this widget receives nav/click
   ↓
Fallback to gameplay (if InputMode allows)
```

Widgets not in the active stack, or not focused, **do not receive input** regardless of visibility.

## Focus Management

Focus is manual. Two main hooks:

```cpp
// In your activatable widget
UWidget* UMyScreen::GetDesiredFocusTarget() const
{
    return FirstButton; // Widget the screen wants focused when activated
}
```

```cpp
// Request Common UI to re-apply focus (e.g. after content rebuilds)
UCommonUISubsystemBase* CUISub = GetGameInstance()->GetSubsystem<UCommonUISubsystemBase>();
// Or more commonly on the activatable itself:
RequestRefreshFocus();
```

**Delay trap.** Calling `RequestRefreshFocus` immediately after pushing a widget may fire before the widget is fully constructed in the stack. Common workaround: delay by one tick via `FTimerManager::SetTimerForNextTick` or a Blueprint `Delay 0.0`.

## Back Action Handling

Enable on the activatable:

```cpp
UMyScreen::UMyScreen()
{
    bIsBackHandler = true;  // or set BackHandler via editor
}

bool UMyScreen::NativeOnHandleBackAction()
{
    DeactivateWidget(); // or pop stack, or custom dismiss logic
    return true;        // handled; stop propagation
}
```

Return `false` to let back action propagate to the next activatable in the stack.

## Input Config (`GetDesiredInputConfig`)

Every activatable can declare how it wants input routed:

```cpp
TOptional<FUIInputConfig> UMyScreen::GetDesiredInputConfig() const
{
    FUIInputConfig Config;
    Config.InputMode       = ECommonInputMode::Menu;   // Menu | Game | All
    Config.MouseCaptureMode = EMouseCaptureMode::NoCapture;
    Config.bHideCursorDuringViewportCapture = false;
    return Config;
}
```

| `ECommonInputMode` | Routes to |
|---|---|
| `Game` | Gameplay only (no UI input) |
| `Menu` | UI only (gameplay input suspended) |
| `All` | Both (HUD with underlying gameplay) |

When the active activatable changes, Common UI reads the new top widget's `GetDesiredInputConfig` and applies it. Never manually call `APlayerController::SetInputModeGameOnly/UIOnly/GameAndUI`.

## Stack vs Queue

| Container | Behavior | Use when |
|---|---|---|
| `UCommonActivatableWidgetStack` | Push adds to top, pop removes top; all widgets persist in memory; top is active | Hierarchical menus where back should return to previous screen |
| `UCommonActivatableWidgetQueue` | One active at a time; previous is destroyed before next is shown | Serialized notifications, tutorial overlays |

Stack is the default for menu systems. Queue is for non-overlapping one-shot UIs.

### Root Content Widget pattern

A stack with nothing in it leaves `GetDesiredInputConfig` undefined — gameplay input can stay captured. Create a minimal `RootContentWidgetClass` (empty `UCommonActivatableWidget` with `InputMode = Game`, `MouseCapture = NoCapture`) and assign it on the stack so an "empty" stack still has a valid input config.

## Enhanced Input Integration

UI input and gameplay input are separate subsystems. Common UI has its own action-data-table-driven input system; Enhanced Input drives gameplay.

- **Gameplay input** → IMC + IA assets (see `contexts/enhanced_input.md`)
- **UI input** (clicks, back, nav) → `CommonInputActionDataBase` rows referenced by the Input Data asset

Both can be active simultaneously via `ECommonInputMode::All`. For a HUD showing gameplay underneath, active activatable declares `All`; for a full-screen menu that should pause gameplay input, declare `Menu`.

Viewport client routes input to Common UI first; unclaimed input falls through to the gameplay input stack.

## Platform / Device Styling

`UCommonInputSubsystem::GetCurrentInputType()` returns the last-used input device:

```cpp
ECommonInputType Type = GetGameInstance()
    ->GetSubsystem<UCommonUIActionRouterBase>()
    ->GetActiveInputType();
```

`UCommonInputBaseControllerData` classes (one per platform/controller) define icon sets and key glyphs. Common UI auto-swaps icons on `UCommonActionWidget` / `UCommonBoundActionButton` when the active input device changes. Register the controller data classes in Project Settings or the input subsystem picks up defaults.

Bind to `UCommonInputSubsystem::OnInputMethodChangedNative` to react to device swaps at runtime (e.g., show keyboard tooltip when switching from gamepad).

## UE 5.7 Notes

- Common UI's Enhanced Input integration is documented in Epic's "Using CommonUI With Enhanced Input" page — the viewport client does the routing; Enhanced Input IMC can coexist.
- As of UE 5.5+, `PushWidget` also activates the widget. **Do not call `ActivateWidget` after `PushWidget`** — double-init breaks `OnActivated` expectations. Older tutorials still show the double-call pattern; ignore them.
- `UCommonBoundActionButton` now supports `InputActionDataRow` directly in editor — no separate `UCommonBoundActionBar` entry needed for simple bindings.

## Common Pitfalls

- **Nothing happens when you launch PIE and add a widget.** Viewport Client Class wasn't changed to `CommonGameViewportClient`.
- **Widget shows but receives no input.** Widget inherits from `UUserWidget` instead of a Common UI base class.
- **Buttons show no icons / back action doesn't fire.** Project Settings → Common Input → Input Data isn't set, or the referenced action data table lacks `DefaultClick` / `DefaultBack` rows.
- **Gameplay input gets stuck disabled after closing a menu.** Someone called `APlayerController::SetInputModeUIOnly()`. Remove it; let Common UI's `GetDesiredInputConfig` handle it.
- **Gamepad navigation is dead on a menu.** Focus wasn't set — override `GetDesiredFocusTarget` or call `RequestRefreshFocus` one tick after push.
- **Back button does nothing.** `bIsBackHandler` wasn't set on the activatable, or `NativeOnHandleBackAction` wasn't overridden (or returned `false` and nothing upstream handled it).
- **Widget's "first-time" data fails to load on second activation.** Logic was placed in `NativeOnInitialized` or the constructor, but the widget got reused. Move per-activation logic to `NativeOnActivated`.
- **`PushWidget` then `ActivateWidget` breaks init (5.5+).** Push already activates. Drop the explicit `ActivateWidget` call.
- **Empty stack leaves gameplay input captured.** Assign a `RootContentWidgetClass` (empty activatable with `Game` input mode) to the stack.
- **Different icons don't appear on gamepad.** No `UCommonInputBaseControllerData` registered for the gamepad platform, or the gamepad class isn't in Project Settings → Common Input → Default Controller Data.
- **MVVM bindings don't update after re-activation.** ViewModel was bound in `NativeConstruct` and not re-bound in `NativeOnActivated` after a deactivation/reactivation cycle.
- **Multiple activatables all listen for the same back action, only one handles it.** `NativeOnHandleBackAction` returned `true` on a lower-priority widget that shouldn't have consumed it. Return `false` when the widget isn't the intended back target.

## References

- `contexts/slate.md` — low-level Slate widgets underlying UMG and Common UI
- `contexts/enhanced_input.md` — gameplay input (Common UI input is a separate system that coexists)
- Epic docs (authoritative, consult directly):
  - Common UI Quickstart Guide
  - Input Fundamentals for CommonUI
  - CommonUI Input Technical Guide
  - Using CommonUI With Enhanced Input
