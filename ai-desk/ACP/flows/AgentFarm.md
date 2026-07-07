```js
// AgentFarm remote ACP 总览
// 目标:
// - AgentFarm live run 期间，把 AgentFarm ACP event 转成标准 AcpRuntimeEvent
// - live 事件实时推回 FE，同时写入 cloud conversation_events
// - 刷新/重新进入 task 时，不再找 AgentFarm 拿历史，只读 cloud acp_runtime_event
// - UI 始终只消费 AcpRuntimeEvent，不直接理解 AgentFarm raw event
```

```js
// Live flow: AgentFarm ACP send
ChatPanel / Flow Task UI
  |
  v
frontend/src/services/task-acp-chat/
  chat-task/TaskACPChatRuntimeAdapter.ts      [扩展 route: agentfarm_acp]
  flow-task/TaskACPFlowRuntimeAdapter.ts      [扩展 route: agentfarm_acp]
  |
  v
frontend/src/services/task-acp-chat/common/
  TaskACPChatRuntimeController.ts             [扩展 send: remote ACP sender]
  |
  v
frontend/src/services/task-acp-chat/common/
  TaskACPRemoteRuntimeSender.ts               [新增]
  |
  v
frontend/src/services/RemoteRuntimeService.ts [扩展 runtime_event chunk]
  |
  | WS /api/v1/remote-runtime/executions/stream
  |
  | payload:
  |   runtimeTransport = "agentmesh"
  |   runtimeProtocol  = "acp"
  |   cliType          = "codex" / "claude"
  |
  v
backend/app/api/v1/remote_runtime.py          [扩展接收/返回 runtime_event]
  |
  v
backend/app/services/remote_runtime_bridge.py [扩展分流]
  |
  | if runtimeTransport == agentmesh
  | && runtimeProtocol == acp
  |
  v
backend/app/services/agentmesh_acp_runtime_transport.py [新增]
  |
  v
AgentFarm SDK ACP subscribe
  |
  v
backend/app/services/agentmesh_acp_event_mapper.py      [新增]
  |
  | AgentFarm raw/normalized event
  |   -> AcpRuntimeEvent
  |
  +-------------------------+--------------------------+
                            |
                            v
```

```js
// Live flow: mapper 后分两路
AgentMeshACPEventMapper
  |
  +-- persist path
  |     |
  |     v
  |   backend/app/services/task_acp_conversation_event_service.py [新增]
  |     |
  |     v
  |   conversation_events
  |     event_type = "acp_runtime_event"
  |     payload.acpRuntimeEvent = AcpRuntimeEvent
  |     payload.runtimeOrigin = "agentfarm"
  |     payload.runtimeTransport = "agentmesh"
  |     payload.runtimeProtocol = "acp"
  |
  +-- realtime path
        |
        v
      WS chunk:
      {
        "type": "runtime_event",
        "event": AcpRuntimeEvent
      }
        |
        v
      frontend/src/foundation/acp/             [不动]
        ACPRuntimeEventStore
        ACPTimelineTransformer
        ACPTimelineReducer
        ACPTimelineView
```

```js
// Reload flow: 刷新 / 重新进入 task
Page refresh / enter task
  |
  v
frontend/src/services/task-acp-chat/
  chat-task/useTaskACPChatRuntime.ts
  flow-task/useTaskACPFlowRuntime.ts
  |
  v
frontend/src/services/task-acp-chat/common/
  TaskACPChatRuntimeController.reload()        [扩展]
  |
  +-- agentfarm_acp / remote ACP
  |     |
  |     v
  |   TaskACPChatCloudRepository.ts            [已有]
  |     |
  |     v
  |   GET /api/v1/tasks/{taskId}/conversations/{threadId}/events
  |     query: stepId / runId / afterSeq
  |     |
  |     v
  |   conversation_events
  |     event_type = "acp_runtime_event"
  |     payload.runtimeOrigin = "agentfarm"
  |
  +-- mixed local + remote personal
        |
        +-- local selected = recorded > replay
        +-- cloud selected = conversation_events.acp_runtime_event
        |
        v
      TaskACPChatEventMerger.ts                [新增/替代 selector]
        dedupe + timestamp sort
        |
        v
      frontend/src/foundation/acp/             [不动]
        ACPRuntimeEventStore -> ACPTimelineView
```

```js
// Flow Task Team Activity reload
Team Activity
  |
  +-- All
  |     显示当前 step 下所有可见 contributions 的 ACP events 合并结果
  |
  +-- 某个 contribution
        显示该 contribution/thread 自己的 ACP events

Team Activity select All / contribution
  |
  v
frontend/src/services/task-acp-chat/flow-task/
  useTaskACPFlowContributionView.ts
  |
  v
frontend/src/services/task-acp-chat/flow-task/
  TaskACPFlowContributionTimelineService.ts
  |
  v
frontend/src/services/task-chat-runtime/flow-task/
  TaskFlowContributionEventsService.ts         [旧，只负责查 contribution events]
  |
  v
GET /api/v1/tasks/{taskId}/conversations/{threadId}/events?stepId=...
  |
  v
conversation_events
  event_type = "acp_runtime_event"
  |
  v
TaskACPFlowContributionTimelineService
  |
  +-- select contribution:
  |     当前 thread events -> timeline
  |
  +-- select All:
        多个 contribution cloud events
        -> dedupe
        -> timestamp sort
        -> rewrite virtual scope
        -> timeline
```

```js
// Team Activity 发消息规则
如果当前选中 All:
  用户发消息仍然写入“当前用户自己的 contribution/thread”
  All 只是一个虚拟合并视图，不是一个真实 thread

如果当前选中自己的 contribution:
  可以发消息
  写入自己的 contribution/thread

如果当前选中别人的 contribution:
  不允许发消息
  input 隐藏或禁用
```

```js
// Cloud payload 需要明确标记来源
ConversationEvent
  eventType = "acp_runtime_event"
  source    = "user" / "agent"
  role      = "user" / "assistant"
  content   = 可选摘要
  payload:
    {
      schemaVersion: 1,
      surface: "task-agent-chat" | "task-step-chat",
      eventKind: "acp_runtime_event",

      // 新增/要求明确:
      runtimeOrigin: "agentfarm",      // local / agentfarm 后续也可扩展
      runtimeTransport: "agentmesh",
      runtimeProtocol: "acp",

      acpRuntimeEvent: AcpRuntimeEvent,

      scope: {
        taskId,
        threadId,
        viewMode,
        stepId,
        runId,
        scopeKey,
        actorExtensionId,
        actorAccountId
      }
    }
```

```js
// 文件夹分界
frontend/src/services/task-chat-runtime/
  只做 task 业务上下文准备和 contribution 查询
  不放 ACP event mapping
  不知道 AgentFarm raw event

frontend/src/services/task-acp-chat/
  负责 ACP 业务路线:
  - local_acp / agentfarm_acp route
  - send / reload
  - cloud append / cloud load
  - local + cloud event merge
  - Flow Team Activity contribution timeline

frontend/src/foundation/acp/
  只消费标准 AcpRuntimeEvent
  不知道 Chat Task / Flow Task
  不知道 AgentFarm
  不知道 conversation_events

backend/app/services/agentmesh_acp_*.py
  负责 AgentFarm ACP runtime:
  - subscribe AgentFarm ACP
  - normalize AgentFarm event -> AcpRuntimeEvent
  - stream runtime_event chunk to FE

backend/app/services/task_acp_conversation_event_service.py
  负责 AcpRuntimeEvent -> conversation_events
  是 backend 版 TaskACPChatCloudRepository 写入能力
```

```js
// 一句话
AgentFarm SDK 只参与 live run。
刷新历史只读 conversation_events.acp_runtime_event。
FE ACP UI 只认 AcpRuntimeEvent，不认 AgentFarm 原始格式。
```
