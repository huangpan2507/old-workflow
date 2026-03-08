# Workflow问题根源分析

## 问题1：WebSocket通知未发送

### 现象
- 审批提交后，浏览器A中的节点状态仍然是`pending`
- 日志中未看到 `[Workflow Callback] Sent WebSocket notification` 的消息

### 可能原因

**原因1：日志被过滤**
- 用户使用的命令：`docker logs -f foundation-backend-container 2>&1 | grep -E "\[WF\]"`
- `[Workflow Callback]` 日志不包含 `[WF]` 前缀，所以被过滤掉了
- **需要检查**：使用命令 `docker logs --tail 500 foundation-backend-container 2>&1 | grep -i "workflow callback"` 查看是否有相关日志

**原因2：代码提前返回**
- 在 `_process_workflow_approval_callback` 的409-413行，如果节点状态不是`pending`，会提前return
- 但是日志显示 `process_human_decision` 被执行了，说明没有提前返回

**原因3：WebSocket发送失败但被catch了**
- 代码中有try-catch（515-520行），如果发送失败只记录warning
- **需要检查**：查看是否有warning日志

### 验证方法

```bash
# 1. 查看完整的Workflow Callback日志（不过滤）
docker logs --tail 1000 foundation-backend-container 2>&1 | grep -i "workflow callback"

# 2. 查看WebSocket相关的warning日志
docker logs --tail 1000 foundation-backend-container 2>&1 | grep -i "websocket\|failed to send"

# 3. 查看特定task_id的所有日志（不过滤）
TASK_ID="271ba766-4ff6-40ff-ac19-f36a0f486b76"
docker logs --tail 2000 foundation-backend-container 2>&1 | grep "${TASK_ID}"
```

---

## 问题2：'NoneType' object has no attribute 'nodes' 错误

### 错误信息
```
ERROR:app.api.workbench:[WF][271ba766-4ff6-40ff-ac19-f36a0f486b76] Workflow execution failed. 
Error: 'NoneType' object has no attribute 'nodes', 
Error type: AttributeError, 
Node results count: 5, 
Node statuses: {'app-2': 'completed', 'end-7': 'completed', 'start-3': 'completed', 'condition-4': 'completed', 'humanInTheLoop-5': 'completed'}, 
Has pending HumanInTheLoop nodes: False, 
Task status before update: running, 
Current node ID: None
```

### 根本原因

**代码位置**：`backend/app/api/v1/workbench.py:857-966`

当从Message Center提交审批后，代码会调用 `execute_workflow_task` 继续执行workflow：

```python
# steering_tasks.py:533-539
asyncio.create_task(
    workbench.execute_workflow_task(
        task_id=task_id_str,
        request=None,  # ❌ 这里传入了None
        session=session,
        user_id=current_user.id
    )
)
```

但是 `execute_workflow_task` 函数在多个地方直接访问 `request.nodes`、`request.edges` 等属性：

```python
# workbench.py:960-966
graph = execution_service.build_graph(
    request.nodes,        # ❌ request为None时会报错
    request.edges,        # ❌ request为None时会报错
    request.app_node_configs,  # ❌ request为None时会报错
    async_session,
    user_id
)
```

**问题**：当 `request=None` 时（从checkpoint继续执行），需要从数据库中的 `WorkflowExecutionTask` 读取 `nodes`、`edges` 等信息，而不是从 `request` 中读取。

### 需要修改的位置

1. **960-966行**：`build_graph` 调用
   ```python
   graph = execution_service.build_graph(
       request.nodes,  # ❌ 需要改为：task.nodes if request is None else request.nodes
       request.edges,  # ❌ 需要改为：task.edges if request is None else request.edges
       request.app_node_configs,  # ❌ 需要改为：task.app_node_configs if request is None else request.app_node_configs
       ...
   )
   ```

2. **972-986行**：`initial_state` 构建
   ```python
   initial_state = {
       "input_data": request.input_data or {},  # ❌ 需要改为：task.input_data if request is None else (request.input_data or {})
       "workflow_definition": {
           "nodes": request.nodes,  # ❌ 需要改为：task.nodes
           "edges": request.edges   # ❌ 需要改为：task.edges
       },
       ...
   }
   ```

3. **995行**：日志输出
   ```python
   f"[WF][{task_id}] Starting workflow execution with {len(request.nodes)} nodes."  # ❌ 需要改为：len(task.nodes)
   ```

### 解决方案

修改 `execute_workflow_task` 函数，当 `request=None` 时，从数据库读取workflow定义：

```python
async def execute_workflow_task(
    task_id: str,
    request: Optional[WorkflowExecuteRequest],  # 改为Optional
    session: Session,
    user_id: uuid.UUID
):
    async_session = Session(engine)
    
    try:
        # 1. 获取task（无论是新执行还是继续执行都需要）
        task = async_session.exec(select(WorkflowExecutionTask).where(...)).first()
        if not task:
            logger.error(f"Task {task_id} not found")
            return
        
        # 2. 如果request为None，从task中读取workflow定义
        if request is None:
            # 从checkpoint继续执行，使用task中的定义
            nodes = task.nodes
            edges = task.edges
            app_node_configs = task.app_node_configs or {}
            input_data = task.input_data or {}
        else:
            # 新执行，使用request中的定义
            nodes = request.nodes
            edges = request.edges
            app_node_configs = request.app_node_configs
            input_data = request.input_data or {}
        
        # 3. 后续代码使用nodes、edges、app_node_configs、input_data变量
        graph = execution_service.build_graph(
            nodes,
            edges,
            app_node_configs,
            async_session,
            user_id
        )
        
        initial_state = {
            "input_data": input_data,
            "workflow_definition": {
                "nodes": nodes,
                "edges": edges
            },
            ...
        }
        
        logger.info(f"[WF][{task_id}] Starting workflow execution with {len(nodes)} nodes.")
        ...
```

### 还需要检查的问题

从日志看，错误发生在继续执行workflow时。还需要确认：

1. **Thread ID和Checkpoint ID**：继续执行时，应该使用原有的`thread_id`和`checkpoint_id`，而不是创建新的
2. **State恢复**：应该从checkpoint恢复state，而不是创建新的initial_state

---

## 修复优先级

### 优先级1：问题2（'NoneType' object has no attribute 'nodes'）

**影响**：导致workflow无法继续执行，任务状态变为`failed`

**修复**：
1. 修改 `execute_workflow_task` 函数，支持 `request=None` 的情况
2. 从数据库读取workflow定义（nodes、edges、app_node_configs、input_data）
3. 使用原有的thread_id和checkpoint_id继续执行

### 优先级2：问题1（WebSocket通知未发送）

**影响**：前端无法实时更新节点状态（需要手动刷新）

**验证步骤**：
1. 先使用不过滤的日志命令检查是否有 `[Workflow Callback]` 相关日志
2. 如果有日志但WebSocket未发送，检查是否有warning日志
3. 如果确实未发送，检查WebSocket连接状态

**修复**（如果确实有问题）：
1. 确保WebSocket通知代码执行路径正确
2. 添加更详细的日志
3. 检查WebSocket连接管理













