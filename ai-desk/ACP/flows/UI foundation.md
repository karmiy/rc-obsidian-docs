```js
// 模块
frontend/src/foundation/acp/

runtime/
  ACPRuntimeClient
  ACPRuntimeSessionManager
  ACPRuntimeConversationBridge
  useACPRuntimeConversation
  ACPRuntimeEventStore
  ACPTimelineTransformer
  ACPTimelineReducer
  ACPActivityGrouping
  ACPTurnTiming
  ACPRuntimeConversationStatus

components/
  ACPTimelineView
  ACPPendingTurnLoader
  ACPTurnFooter
  ACPClipboard
```
```js
// 职责
ACPRuntimeClient
  最底层 daemon client，只负责 HTTP/WS 调 daemon ACP 接口。

ACPRuntimeSessionManager
  runtime 会话状态管理器。
  管 connection、pending turn、active turn、event store、load/send。

ACPRuntimeConversationBridge
  把“某个业务 conversation scope”接到 SessionManager。
  对外给 snapshot / sendTurn / subscribe。

useACPRuntimeConversation
  React hook 包装 Bridge，给 UI/业务层用。

ACPRuntimeEventStore
  存当前 conversation 的 AcpRuntimeEvent 列表和 snapshot。

ACPTimelineTransformer
  把 AcpRuntimeEvent 转成更适合 UI reducer 的 timeline event。

ACPTimelineReducer
  把 timeline event 归并成稳定的 timeline item。

ACPActivityGrouping
  管 activity/tool group 展示结构。

ACPTurnTiming
  只算 turn 时间，不管业务。
  live / session_load 的时间显示规则也在这里附近收敛。

ACPTimelineView
  纯 foundation UI。
  接收 timeline props，渲染消息、activity、footer、copy button。
```
```js
// 整体链路
业务层
  Free Task / Flow Task / 未来别的业务
        |
        | 传入 runtime request、conversation id、events、display hooks
        v
ACP foundation runtime
        |
        v
ACP foundation components
        |
        v
UI timeline
```
```js
// Live send flow
业务层 send()
  |
  v
ACPRuntimeConversationBridge.sendTurn()
  |
  v
ACPRuntimeSessionManager
  |
  +-- 创建/复用 conversation state
  +-- 标记 pending turn / active turn
  +-- 管理连接生命周期
  |
  v
ACPRuntimeClient
  |
  v
daemon ACP live 接口
  |
  v
daemon 返回 AcpRuntimeEvent
  |
  v
ACPRuntimeSessionManager.ingestEvents()
  |
  v
ACPRuntimeEventStore
  |
  v
ACPRuntimeConversationBridge snapshot
  |
  v
useACPRuntimeConversation / 业务 hook
  |
  v
ACPTimelineView
```
```js
// Reload flow
业务层 restore/reload()
  |
  v
ACPRuntimeSessionManager.loadEvents()
  |
  v
ACPRuntimeClient.loadEvents()
  |
  v
daemon ACP session/load
  |
  v
replay AcpRuntimeEvent[]
  |
  v
ACPRuntimeEventStore
  |
  v
ACPRuntimeConversationBridge snapshot
  |
  v
ACPTimelineView
```
```js
// Render flow
AcpRuntimeEvent[]
  |
  v
ACPTimelineTransformer
  |
  |  runtime event -> timeline event
  |  比如 message / thought / activity / status / error
  v
ACPTimelineReducer
  |
  |  timeline event -> stable timeline item
  |  合并 delta、更新 tool/activity 状态
  v
ACPActivityGrouping
  |
  |  把相关 activity/tool call 组织成 group
  v
ACPTurnTiming
  |
  |  算 active turn、worked for、发生时间
  v
ACPTimelineView
  |
  +-- ACPPendingTurnLoader
  +-- ACPTurnFooter
  +-- ACPClipboard
```
```js
// 连接生命周期
正常情况下：
  多个 turn 复用同一个 WebSocket + 同一个 ACP live session

stop / 页面刷新 / 网络断开：
  WebSocket 会断
  下次发消息会重新开 WebSocket
  但可以靠 conversationId / sessionId 继续
```