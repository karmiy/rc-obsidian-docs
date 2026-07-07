```js
// 入口分支
Free Task / Flow Task 页面
        |
        v
业务薄入口
当前 Free Task = TaskAgentStore
未来 Flow Task = Flow 自己的薄入口
        |
        v
检查 feature flag: rc.ai.desk.task_chat_acp
        |
        +-- true  -> Task ACP chat path      新 ACP path
        |
        +-- false -> legacy TaskAgentStore   旧 path，保持现状
```
```js
// 新 ACP FE modules
// 1. 通用 task chat runtime 准备层，不属于 ACP
frontend/src/services/task-chat-runtime/

TaskChatRuntimePreparation.ts

// 负责:
// - 构造 task chat prompt params
// - 合并 AI Desk 业务 Agent 的 environment
// - task chat metadata 小工具
// 注意:
// - 这里的 Agent / TaskAgent 是 AI Desk 业务概念
// - codex / cursor / claude 这类叫 runtime 或 cliCommand

// 2. ACP task chat 业务编排层，只负责 ACP + cloud/reload/backfill
frontend/src/services/task-acp-chat/

TaskACPChatController.ts
TaskACPChatReloadService.ts
TaskACPChatCloudRepository.ts
TaskACPChatCloudBackfillService.ts
TaskACPChatEventSelector.ts
TaskACPChatTypes.ts

// 3. Task ACP prompt / route mapping，当前仍在 agent service 下
frontend/src/services/agent/

TaskACPRuntimeAdapter.ts
TaskACPRuntimePromptCodec.ts
```
```js
// 整体关系图
TaskAgentStore / future Flow thin entry
   |
   +-- TaskChatRuntimePreparationService
   |     通用业务准备:
   |     task + task agents + context options -> prompt params / env / metadata helpers
   |
   +-- TaskACPChatController
         ACP 业务编排:
         send / reload / cloud append / recorded backfill / event selection
         |
         +-- TaskACPRuntimeAdapter
         |     task metadata / prompt params / mode -> ACP runtime prompt/request
         |
         +-- TaskACPChatReloadService
         |     reload 本地历史读取
         |     loadRecordedEvents -> daemon ~/.aidesktop recorded events
         |     loadReplayEvents -> daemon ACP session/load replay fallback
         |
         +-- TaskACPChatCloudRepository
         |     ConversationEvent <-> AcpRuntimeEvent
         |     cloud load / append / 过滤 legacy or ACP
         |
         +-- TaskACPChatCloudBackfillService
         |     接收 controller 已读取的 recorded events（```
event.source.ingest_source === 'live'
```，不包括 repair 的 recorded）
         |     和 cloud diff，补缺
         |
         +-- TaskACPChatEventSelector
               personal/shared thread 的 UI 显示选择

// 依赖方向:
// TaskAgentStore / future Flow thin entry 可以同时调用
//   - TaskChatRuntimePreparationService
//   - TaskACPChatController
//
// TaskACPChatController 不 import TaskChatRuntimePreparationService。
// 两者是兄弟模块，避免 ACP 业务编排层反向依赖 runtime 准备层。
```
```js
// Send flow
用户发消息
  |
  v
Free Task / Flow Task UI
  |
  v
业务薄入口
  |
  +-- flag false
  |     -> legacy TaskAgentStore send
  |
  +-- flag true
        |
        v
        TaskChatRuntimePreparationService
        |
        +-- build prompt params
        +-- build AI Desk Agent environment
        |
        v
        TaskACPChatController.sendPreparedLocalTurn()
        |
        +-- TaskACPRuntimeAdapter
        |     从 prompt params + mode 拼 ACP runtime prompt/request
        |
        +-- MCP config / Knowledge Capture / metadata patch
        |
        v
        daemon ACP live
        |
        v
        CLI raw ACP event
        |
        v
        daemon 转 AcpRuntimeEvent
        |
        +-- daemon 写 ~/.aidesktop
        |
        +-- 返回前端
              |
              +-- ACPTimelineView 渲染
              |
              +-- TaskACPChatCloudRepository.append()
                    AcpRuntimeEvent -> cloud ConversationEvent
```
```js
// Reload flow
页面刷新 / 重新进入 task
  |
  v
Free Task / Flow Task UI
  |
  v
业务薄入口
  |
  +-- flag false
  |     -> legacy TaskAgentStore restore
  |
  +-- flag true
        |
        v
        TaskACPChatController.reload()
        |
        +--------------------------------------------------+
        | 先 resolve thread，再按 viewMode 读取历史          |
        |                                                  |
        | 1. TaskACPChatCloudRepository.resolveDisplayThread() |
        |    live share 未开启 -> 当前 viewer 的 personal thread |
        |    live share 已开启 -> liveShare.threadId(shared thread) |
        |                                                  |
        | 2. TaskACPChatCloudRepository.load()              |
        |    BE ConversationEvent -> cloud AcpRuntimeEvent[] |
        |    只读 ACP events，过滤旧 legacy cloud events      |
        |                                                  |
        | 3. viewMode = personal 时                         |
        |    TaskACPChatReloadService.loadRecordedEvents()   |
        |    daemon ~/.aidesktop -> recorded events          |
        |                                                  |
        | 4. personal 且 recorded 为空时才 lazy fallback     |
        |    TaskACPChatReloadService.loadReplayEvents()     |
        |    daemon ACP session/load -> replay events        |
        |                                                  |
        | 5. personal 且允许 backfill 时                     |
        |    TaskACPChatCloudBackfillService.run()           |
        |    复用同一份 recorded events 和 cloud diff（```
event.source.ingest_source === 'live'
```，不包括 repair 的 recorded）         |
        |    补 cloud 缺失                                   |
        |                                                  |
        | 6. viewMode = shared 时                            |
        |    只读取 shared cloud events                      |
        |    不读取 recorded / replay，不做 recorded backfill |
        +--------------------------------------------------+
                  |
                  v
        TaskACPChatEventSelector.select()
                  |
                  +-- viewMode = personal:
                  |     recorded events 有数据 -> UI 用 recorded
                  |     否则 replay events 有数据 -> UI 用 replay
                  |     否则 -> UI 用 personal cloud events
                  |
                  +-- viewMode = shared:
                        UI 用 shared cloud events
                        不读 recorded 做主显示
                        不用 replay 做主显示
                        因为 recorded/replay 只代表当前机器/provider session，
                        shared cloud 才包含 owner + participants 的完整共享会话
```
```js
// Cloud storage flow
live / backfill 需要写 cloud
  |
  v
TaskACPChatCloudRepository.append()
  |
  v
ConversationEvent
  |
  +-- clientEventId = AcpRuntimeEvent.id
  +-- eventType = acp_runtime_event
  +-- source = user / agent / system
  +-- content = 可选摘要
  +-- payload = 完整 AcpRuntimeEvent + task scope
  +-- seq = BE 自己分配的 thread 顺序
```
```js
// Backfill flow
TaskACPChatCloudBackfillService.run()
  |
  +-- 输入 cloud events
  |
  +-- 输入 controller 已读取的 recorded events
  |     同一份 recordedEvents 也会给 UI selector 复用
  |
  +-- diff:
  |     recorded event id 不在 cloud clientEventId 里（```
event.source.ingest_source === 'live'
```，不包括 repair 的 recorded）
  |
  v
TaskACPChatCloudRepository.appendMissing()
  |
  v
cloud ConversationEvent 补齐
```
```js
// Free/Flow task 接入差异
Free Task:
  scope = { surface: 'free_task', taskId }

Flow Task:
  scope = { surface: 'flow_task', taskId, stepId?, runId?, scopeKey? }
```
