```js
// 模块
ai-desk-daemon/daemon/internal/daemonapp/

server.go
  注册 ACP 路由

agent_acp_server.go
  daemon ACP HTTP/WS API 入口

agent_acp_runner.go
  管 ACP provider 进程、live session、turn 执行

agent_acp_client.go
  JSON-RPC client，和 provider ACP server 通过 stdio 通信

agent_acp_events.go / agent_acp_runtime_events.go
  raw ACP update -> AcpRuntimeEvent

agent_acp_history.go
  ~/.aidesktop recorded events 存取

agent_acp_session_history.go
  ACP session/load replay 历史
```
```js
// Live flow
前端 WebSocket
  /api/v1/acp/conversations/:conversationId/stream
        |
        v
agent_acp_server.handleACPRuntimeStream
        |
        +-- start / resume
        |     |
        |     v
        |   AgentACPRunner.StartConversationSession()
        |     |
        |     +-- 启动或复用 ACP provider 进程
        |     +-- initialize / authenticate
        |     +-- session/new 或 session/load
        |     +-- 返回 ready + sessionId
        |
        +-- user_message
              |
              v
            AgentACPRunner.SendConversationMessage()
              |
              v
            ACPJSONRPCClient.Prompt()
              |
              v
            provider ACP server
              |
              v
            raw ACP session/update
              |
              v
            ACPEventMapper
              |
              v
            AcpRuntimeEvent
              |
              +-- 写 ~/.aidesktop
              |
              +-- WebSocket 返回前端 runtime_event
              |
              v
            turn complete
              |
              v
            daemon 写 ready，但 WebSocket 不断开
```
```js
// Reload session/load flow
前端 HTTP GET
  /api/v1/acp/conversations/:conversationId/events?history_source=acp_session
        |
        v
agent_acp_server.writeACPRuntimeEvents
        |
        v
AgentACPRunner.LoadConversationSessionHistory()
        |
        v
ACP provider session/load
        |
        v
replay raw ACP events
        |
        v
AcpRuntimeEvent[]
        |
        v
返回前端 replay events
```
```js
// Reload recorded events flow
前端 HTTP GET
  /api/v1/acp/conversations/:conversationId/events
  不带 history_source=acp_session
        |
        v
DefaultHistoryStore().List()
        |
        v
读取 ~/.aidesktop recorded AcpRuntimeEvent
        |
        v
返回前端 recorded events
```
```js
// Stop flow
前端 stop
  |
  v
WebSocket message: stop
  |
  v
agent_acp_server
  |
  +-- cancel active turn
  +-- mapper.Cancelled("Stopped by user")
  +-- persistingEmitter 写 cancelled AcpRuntimeEvent 到 ~/.aidesktop
  +-- runner.PauseConversation()
  +-- writeState("stopped")
  +-- return，WebSocket 断开
```
```js
// 生命周期
// 规则
同一个 conversationId + 同一套 runtime 参数 + session 还 alive
  -> 复用同一个 ACP provider 进程/session
  示例：
	- turn 1 完成 -> session idle，但不关
	- turn 2 继续 -> 复用同一个 provider session/prompt

不同 conversationId
  -> 通常是不同 ACP provider 进程/session

同 conversationId 但 cwd/env/command/mcp 变了
  -> 旧 session 关闭，重新起一个
// 清理策略
idle 30 分钟会被清理
默认最多保留 3 个 live sessions
AI_DESK_ACP_MAX_LIVE_SESSIONS 可以改最大数量
stop / pause / error / provider 退出也会关闭或丢弃
daemon 退出时全部关闭
```