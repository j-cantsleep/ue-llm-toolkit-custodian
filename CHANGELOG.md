# Changelog

Local changes to the upstream plugin. All changes are additive or defensive — no breaking API changes.

## Blueprint Modify Tool — Schema Hardening

**File:** `Plugin/UELLMToolkit/Source/UELLMToolkit/Private/MCP/Tools/MCPTool_BlueprintModify.h`

Expanded the `blueprint_modify` tool schema to document behavior that was previously implicit or silent:

- **`node_params` structure** — Now explicitly documents that `node_params` is a required nested object, not flat top-level params, and describes the `pin_values` sub-object for setting pin defaults at creation time.
- **`CallFunction` `target_class` resolution** — Documents which short library names resolve (5 hardcoded: `KismetSystemLibrary`, `KismetMathLibrary`, `KismetStringLibrary`, `GameplayStatics`, `AnimInstance`) and directs use of full script paths for everything else.
- **`Event` node constraints** — Clarifies that only `BeginPlay`, `Tick`, `EndPlay` are hardcoded; other event names must exist as `BlueprintImplementableEvent` or `BlueprintNativeEvent` on the parent C++ class.
- **`VariableSet` asymmetry** — Notes that `VariableGet` can read C++ parent properties but `VariableSet` cannot write them; workaround is calling a `BlueprintCallable` setter.
- **Extended `node_type` list** — Added undocumented supported node types: `ModifyBone`, `TwoBoneIK`, `ControlRig` (AnimGraph), plus aliases `IfThenElse`, `GetVariable`, `SetVariable`.

## Blueprint Modify Tool — node_params Validation

**File:** `Plugin/UELLMToolkit/Source/UELLMToolkit/Private/MCP/Tools/MCPTool_BlueprintModify.cpp`

Added validation in `ExecuteAddNode` that rejects unknown keys in `node_params` with a helpful error message. Previously, unknown keys were silently ignored, producing broken nodes without feedback.

Valid keys are whitelisted: `function`, `target_class`, `event`, `action_path`, `variable`, `num_outputs`, `bone_name`, `control_rig_class`, `pin_values`. Any other key returns an error listing the valid set and pointing at `pin_values` as the correct location for pin default values.