# Copilot 流式输出实现详解

本文档详细说明 copilot 功能如何实现流式输出（Streaming Output）。

## 架构概览

Copilot 的流式输出采用 **WebSocket + LangGraph 事件流** 的架构：

```
前端 (React) 
  ↓ WebSocket
后端 API (FastAPI)
  ↓ 调用
ChatAgent (LangGraph)
  ↓ astream_events
LLM (流式生成)
  ↓ 实时事件
SessionManager
  ↓ WebSocket
前端实时显示
```

## 核心组件

### 1. 后端流式输出引擎

#### 1.1 WebSocket 端点 (`backend/app/api/v1/copilot.py`)

```python
@router.websocket("/ws/chat/{session_id}")
async def chat_websocket(websocket: WebSocket, session_id: str, token: str):
    # 1. 建立 WebSocket 连接
    await chat_session_manager.connect_websocket(session_id, websocket)
    
    # 2. 接收用户消息
    data = await websocket.receive_json()
    user_message = data.get("content", "")
    
    # 3. 调用 agent.chat() 并设置 stream=True
    async for event in agent.chat(
        message=user_message,
        user_id=user.id,
        session_id=session_id,
        thread_id=unique_thread_id,
        stream=True  # 启用流式输出
    ):
        # 4. 处理不同类型的事件
        if event["type"] == "token":
            # 流式 token，立即发送给前端
            await chat_session_manager.stream_token(
                session_id,
                event["content"]
            )
        elif event["type"] == "complete":
            # 流式完成，发送完整消息和证据数据
            await chat_session_manager.stream_complete(
                session_id,
                event["content"],
                evidence=event.get("metadata", {}).get("evidence")
            )
```

**关键点**：
- 使用 FastAPI 的 `@router.websocket` 装饰器创建 WebSocket 端点
- 通过 `stream=True` 参数启用流式模式
- 实时处理并转发 `token` 和 `complete` 事件

#### 1.2 ChatAgent 流式处理 (`backend/app/langgraph_agents/chat_agent.py`)

```python
async def _stream_response(
    self,
    input_state: ChatState,
    config: Dict[str, Any]
) -> AsyncIterator[Dict[str, Any]]:
    """
    核心流式响应方法
    使用 LangGraph 的 astream_events 监听 LLM 的流式输出
    """
    accumulated_content = ""
    
    # 使用 astream_events 监听所有事件
    async for event in self.graph.astream_events(input_state, config, version="v2"):
        event_type = event.get("event")
        
        # 监听 LLM 的流式输出事件
        if event_type == "on_chat_model_stream":
            chunk = event.get("data", {}).get("chunk")
            
            # 从 chunk 中提取 token 内容
            token = None
            if hasattr(chunk, "content"):
                token = chunk.content
            elif isinstance(chunk, dict):
                token = chunk.get("content") or chunk.get("text")
            
            if token:
                accumulated_content += token
                # 立即 yield token，不等待完整响应
                yield {
                    "type": "token",
                    "content": token,
                    "metadata": {
                        "accumulated": accumulated_content,
                        "token_index": token_count
                    }
                }
        
        # 处理节点完成事件
        elif event_type == "on_chain_end":
            event_name = event.get("name", "unknown")
            
            # 检查是否是最终合成节点
            if event_name in ["final_synthesizer", "result_formatter"]:
                output = event.get("data", {}).get("output", {})
                final_markdown = output.get("final_markdown", "")
                
                # 如果没有收到流式 token，使用 fallback 流式传输
                if final_markdown and not accumulated_content:
                    # 按字符分割以保留 markdown 格式
                    for char in final_markdown:
                        yield {
                            "type": "token",
                            "content": char,
                            "metadata": {}
                        }
                        await asyncio.sleep(0.01)  # 模拟流式效果
    
    # 流式完成，发送完整消息和证据数据
    yield {
        "type": "complete",
        "content": accumulated_content,
        "metadata": {
            "evidence": evidence_data  # 包含决策证据链
        }
    }
```

**关键点**：
- 使用 `astream_events(version="v2")` 监听 LangGraph 的所有事件
- 监听 `on_chat_model_stream` 事件获取 LLM 的实时 token
- 使用 `AsyncIterator` 实现异步生成器，支持流式 yield
- 提供 fallback 机制：如果 LLM 不支持流式，从 `final_markdown` 按字符流式传输

#### 1.3 SessionManager 消息发送 (`backend/app/langgraph_agents/session_manager.py`)

```python
class ChatSessionManager:
    async def stream_token(
        self,
        session_id: str,
        token: str
    ) -> bool:
        """流式发送单个 token 到客户端"""
        return await self.send_message(session_id, {
            "type": "token",
            "content": token,
            "timestamp": datetime.utcnow().isoformat()
        })
    
    async def stream_complete(
        self,
        session_id: str,
        full_message: str,
        evidence: Optional[Dict[str, Any]] = None
    ) -> bool:
        """发送流式完成信号"""
        message = {
            "type": "done",
            "content": full_message,
            "markdown": full_message,
            "timestamp": datetime.utcnow().isoformat()
        }
        if evidence:
            message["evidence"] = evidence  # 包含决策证据链
        return await self.send_message(session_id, message)
    
    async def send_message(
        self,
        session_id: str,
        message: Dict[str, Any]
    ) -> bool:
        """通过 WebSocket 发送消息"""
        websocket = self.connections.get(session_id)
        if not websocket:
            return False
        
        try:
            await websocket.send_json(message)  # FastAPI WebSocket 发送 JSON
            return True
        except Exception as e:
            logger.error(f"Error sending message: {e}")
            return False
```

**关键点**：
- 维护 WebSocket 连接映射：`{session_id: websocket}`
- 使用 `websocket.send_json()` 发送 JSON 格式的消息
- 支持多种消息类型：`token`、`done`、`error`、`log`

### 2. 前端流式接收

#### 2.1 WebSocket 客户端 (`frontend/src/utils/chatWebSocket.ts`)

```typescript
export class ChatWebSocketClient {
  private ws: WebSocket | null = null
  
  connect(): void {
    // 构建 WebSocket URL
    const protocol = window.location.protocol === "https:" ? "wss:" : "ws:"
    const wsUrl = `${protocol}//${window.location.host}/api/v1/copilot/ws/chat/${sessionId}?token=${token}`
    
    this.ws = new WebSocket(wsUrl)
    this.ws.onmessage = this.handleMessage.bind(this)
  }
  
  private handleMessage(event: MessageEvent): void {
    const data: WebSocketMessage = JSON.parse(event.data)
    
    switch (data.type) {
      case "token":
        // 流式 token 到达，立即更新 UI
        this.config.onToken?.(data.content)
        break
      
      case "done":
        // 流式完成，包含完整消息和证据数据
        this.config.onComplete?.(data.markdown || data.content, data.evidence)
        if (data.evidence) {
          this.config.onEvidence?.(data.evidence)
        }
        break
      
      case "error":
        this.config.onError?.(data.content)
        break
    }
  }
  
  sendMessage(content: string): void {
    this.ws?.send(JSON.stringify({
      type: "message",
      content
    }))
  }
}
```

**关键点**：
- 使用原生 `WebSocket` API 建立连接
- 监听 `onmessage` 事件接收流式数据
- 通过回调函数 (`onToken`, `onComplete`) 实时更新 UI

#### 2.2 UI 组件集成

前端组件通过回调函数接收流式 token：

```typescript
const wsClient = new ChatWebSocketClient({
  sessionId: session.id,
  token: authToken,
  onToken: (token: string) => {
    // 实时追加 token 到消息内容
    setCurrentMessage(prev => prev + token)
  },
  onComplete: (fullMessage: string, evidence?: DecisionEvidenceData) => {
    // 流式完成，保存完整消息
    saveMessage(fullMessage, evidence)
  }
})
```

## 流式输出流程

### 完整流程图

```
1. 用户发送消息
   ↓
2. 前端通过 WebSocket 发送消息
   ↓
3. 后端 WebSocket 端点接收消息
   ↓
4. 调用 agent.chat(stream=True)
   ↓
5. ChatAgent._stream_response() 启动
   ↓
6. 使用 graph.astream_events() 监听事件
   ↓
7. LLM 开始生成，触发 on_chat_model_stream 事件
   ↓
8. 提取 token → yield {"type": "token", "content": token}
   ↓
9. WebSocket 端点接收 token 事件
   ↓
10. SessionManager.stream_token() 发送到前端
   ↓
11. 前端 WebSocket 接收 token
   ↓
12. 调用 onToken 回调，实时更新 UI
   ↓
13. 重复步骤 7-12，直到 LLM 生成完成
   ↓
14. 触发 complete 事件，发送完整消息和证据数据
   ↓
15. 前端接收 done 消息，保存完整对话
```

### 时序图

```
前端                WebSocket端点          ChatAgent          LangGraph          LLM
 │                      │                    │                  │                 │
 │── send message ──────>│                    │                  │                 │
 │                      │── chat(stream=True)─>│                  │                 │
 │                      │                    │── astream_events ─>│                 │
 │                      │                    │                  │── generate ──────>│
 │                      │                    │                  │                 │
 │                      │                    │<── on_chat_model_stream ────────────│
 │                      │                    │  (token: "Hello") │                 │
 │                      │<── yield token ────│                  │                 │
 │<── {"type":"token"}──│                    │                  │                 │
 │  (更新 UI)            │                    │                  │                 │
 │                      │                    │<── on_chat_model_stream ────────────│
 │                      │                    │  (token: " world")│                 │
 │                      │<── yield token ────│                  │                 │
 │<── {"type":"token"}──│                    │                  │                 │
 │  (更新 UI)            │                    │                  │                 │
 │                      │                    │                  │                 │
 │                      │                    │<── complete ──────│                 │
 │                      │<── yield complete ─│                  │                 │
 │<── {"type":"done"}───│                    │                  │                 │
 │  (保存消息)           │                    │                  │                 │
```

## 关键技术点

### 1. LangGraph 事件流

使用 `astream_events()` 方法监听 LangGraph 执行过程中的所有事件：

```python
async for event in self.graph.astream_events(input_state, config, version="v2"):
    event_type = event.get("event")  # 事件类型
    event_name = event.get("name")    # 节点名称
    event_data = event.get("data")    # 事件数据
```

**重要事件类型**：
- `on_chat_model_stream`: LLM 流式输出 token
- `on_chain_start`: 节点开始执行
- `on_chain_end`: 节点执行完成
- `on_tool_start/end`: 工具调用

### 2. 异步生成器 (AsyncIterator)

使用 `async def` 和 `yield` 实现异步生成器，支持流式输出：

```python
async def _stream_response(...) -> AsyncIterator[Dict[str, Any]]:
    async for event in self.graph.astream_events(...):
        if event_type == "on_chat_model_stream":
            token = extract_token(event)
            yield {"type": "token", "content": token}  # 立即 yield，不阻塞
```

### 3. WebSocket 双向通信

- **前端 → 后端**：发送用户消息
- **后端 → 前端**：流式发送 token 和完成信号

### 4. Fallback 机制

如果 LLM 不支持流式输出，从 `final_markdown` 按字符流式传输：

```python
if final_markdown and not accumulated_content:
    # 按字符分割以保留 markdown 格式
    for char in final_markdown:
        yield {"type": "token", "content": char}
        await asyncio.sleep(0.01)  # 模拟流式效果
```

## 性能优化

1. **立即转发**：收到 token 后立即发送，不等待累积
2. **异步处理**：使用 `async/await` 避免阻塞
3. **连接管理**：SessionManager 维护连接池，支持多会话
4. **心跳保活**：前端每 30 秒发送 ping，保持连接活跃

## 错误处理

1. **WebSocket 断开**：自动重连机制（最多 5 次）
2. **序列化错误**：使用 `ensure_serializable()` 处理特殊类型
3. **LLM 错误**：通过 `error` 事件类型通知前端

## 总结

Copilot 的流式输出实现基于以下核心技术：

1. **WebSocket**：实现双向实时通信
2. **LangGraph astream_events**：监听 LLM 流式生成事件
3. **异步生成器**：支持流式 yield token
4. **事件驱动架构**：通过事件类型区分不同的消息类型

这种架构实现了真正的流式输出，用户可以看到 AI 逐字生成回复，而不是等待完整响应后才显示，大大提升了用户体验。
