# Runtime Selector Contract

## 目标

`RuntimeSelector` 不再对外吐 `AgentPreset`，而是统一收/吐 `RuntimeSelection`。

```ts
type RuntimeSelection =
  | { type: "local"; source: "preset"; presetId: string; cliType: string; model?: string | null }
  | { type: "local"; source: "model"; presetId?: string | null; cliType: string; model: string }
  | { type: "remote"; source: "preset"; presetId: string; cliType: string; model?: string | null }
  | { type: "shared"; source: "preset" | "model"; presetId?: string | null; cliType: string; model?: string | null; sharedMachineId: string };
```

- `remote`：当前仍然来自 remote preset，`presetId` 是主引用，`cliType` 来自 `remoteCliCommand`。
- `shared`：不是单纯 preset，而是一个本地 runtime 选择加 `sharedMachineId`。
- `model`：和 `preset` 互斥。选 model 时，不因为某个 preset 的 model 相同就高亮 preset。

## 1. 新建任务选择后

```mermaid
flowchart LR
  UI["RuntimeSelector"]
  Local["Local: preset 或 cli+model"]
  Remote["Remote: preset"]
  Shared["Shared: selection + machine"]
  Selection["RuntimeSelection"]
  Draft["Composer draft value"]
  Submit["Submit task"]
  Metadata["metadata.agentChat.runtimeSelection"]
  Legacy["legacy mirror fields"]
  Resolve["resolveRuntimeExecutionParams"]
  Run["ACP / local / remote runtime"]

  UI --> Local
  UI --> Remote
  UI --> Shared
  Local --> Selection
  Remote --> Selection
  Shared --> Selection
  Selection --> Draft
  Draft --> Submit
  Submit --> Metadata
  Submit --> Legacy
  Metadata --> Resolve
  Legacy --> Resolve
  Resolve --> Run
```

提交时建议同时写：

```ts
metadata.agentChat = {
  runtimeSelection,

  // backward compatible fields
  selectedPresetId,
  selected_preset_id,
  cli_type,
  cliType,
  runtime_type,
  runtimeType,
  model,
};
```

## 2. 已有 Task 刷新后

```mermaid
flowchart LR
  Task["Task metadata"]
  NewField["agentChat.runtimeSelection"]
  OldFields["旧字段 selectedPresetId + cli_type + runtime_type + model"]
  Parse["resolveRuntimeSelectionFromMetadata"]
  Selection["RuntimeSelection"]
  Selector["RuntimeSelector value"]
  PresetState["source=preset: 高亮 preset"]
  ModelState["source=model: 高亮 model"]

  Task --> NewField
  Task --> OldFields
  NewField --> Parse
  OldFields --> Parse
  Parse --> Selection
  Selection --> Selector
  Selector --> PresetState
  Selector --> ModelState
```

读取顺序：

1. 优先读 `agentChat.runtimeSelection`。
2. 没有时，用旧字段组装 `RuntimeSelection`。
3. 如果旧字段只有 `selectedPresetId`，用 preset 解析出 `type / cliType / model`。
4. 如果 preset 找不到但还有 `cli_type + model`，fallback 成 `source: "model"`。

## 3. Selector 内部选中规则

```mermaid
flowchart TD
  Input["value: RuntimeSelection"]
  IsPreset["source is preset"]
  IsModel["source is model"]
  MatchPreset["按 presetId exact match"]
  MatchModel["按 cliType + model exact match"]
  Fallback["fallback 默认 runtime"]
  ShowPreset["只选中 Preset"]
  ShowModel["只选中 Model"]

  Input --> IsPreset
  Input --> IsModel
  IsPreset --> MatchPreset
  IsModel --> MatchModel
  MatchPreset --> ShowPreset
  MatchModel --> ShowModel
  MatchPreset --> Fallback
  MatchModel --> Fallback
```

关键点：`Preset` 和 `Model` 是 selection source，不是两个可同时 checked 的列表。

## 4. Flow Task / Step 选择

```mermaid
flowchart LR
  StepUI["Step runtime selector"]
  StepSelection["step.runtimeSelection"]
  TaskDefault["task.defaultRuntimeSelection"]
  Effective["resolveEffectiveRuntimeSelection"]
  Params["resolveRuntimeExecutionParams"]
  ACP["ACP foundation"]
  LegacyStart["legacy start agent path"]

  StepUI --> StepSelection
  StepSelection --> Effective
  TaskDefault --> Effective
  Effective --> Params
  Params --> ACP
  Params --> LegacyStart
```

建议顺序：

1. Step 有自己的 `runtimeSelection` 时用 step。
2. Step 没有时用 task default。
3. Task default 没有时用用户全局默认。
4. 执行前统一解析成 `cliCommand + model + runtimeType + transport`。

## 5. 函数边界

```mermaid
flowchart LR
  Metadata["metadata.agentChat"]
  A["resolveRuntimeSelectionFromMetadata"]
  Selection["RuntimeSelection"]
  B["resolveRuntimeExecutionParams"]
  Params["RuntimeExecutionParams"]
  C["legacyAgentChatRuntimeFields"]
  Legacy["selectedPresetId / cli_type / runtime_type / model"]

  Metadata --> A
  A --> Selection
  Selection --> B
  B --> Params
  Selection --> C
  C --> Legacy
```

建议函数：

```ts
resolveRuntimeSelectionFromMetadata(meta, agentStore): RuntimeSelection | null
resolveRuntimeExecutionParams(selection, agentStore): RuntimeExecutionParams
legacyAgentChatRuntimeFields(selection, params): Record<string, unknown>
runtimeSelectionFromPreset(preset): RuntimeSelection
runtimeSelectionFromCliModel(cliType, model, basePresetId?): RuntimeSelection
```

## 结论

- UI 层只表达用户选择：`RuntimeSelection`。
- Metadata 层存结构化 `runtimeSelection`，并短期保留旧字段。
- 执行层只吃解析后的 `RuntimeExecutionParams`。
- 不在 `RuntimeSelector` 内伪造 preset；如果旧 API 临时还要 `selectedPresetId`，放在 task/store adapter 层兼容。
