# Workflow Enhancements Summary

## 修改日期
2025-12-10

## 修改内容

### 1. 高优先级：统一前端 API 调用路径

**问题**：
- 前端混用了两个不同的 API 接口：
  - `fetchTaskDetail` 使用 `/workflow-tasks/{task_id}` (line 573)
  - 恢复任务状态使用 `/workflows/tasks/{task_id}` (line 1053)
- 导致部分请求返回 404 错误

**修复**：
- 统一使用 `/workflow-tasks/{task_id}` 接口（信息更完整，包括 `app_node_configs`）
- 修改位置：`frontend/src/components/Workbench/FlowWorkspace.tsx:1053`

**修改前**：
```typescript
const response = await fetch(`${API_BASE_URL}/workbench/workflows/tasks/${taskId}`, {
```

**修改后**：
```typescript
const response = await fetch(`${API_BASE_URL}/workbench/workflow-tasks/${taskId}`, {
```

### 2. 中优先级：增强错误处理

**问题**：
- 任务在初始化阶段失败（数据库连接错误）时，任务状态立即被标记为 "failed"
- 但 DAG 可能仍在运行（因为 DAG 是在 App Node 执行时触发的，不依赖工作流执行服务）
- 前端无法区分"任务执行失败"和"DAG 运行中"的情况

**修复**：
- 在 `fetchTaskDetail` 中增加 `checkDagStatus` 参数
- 当任务状态为 "failed" 时，检查 `app_node_configs` 中是否有 `dagRunId`
- 如果有，说明 DAG 可能仍在运行，将任务状态临时更新为 "running" 以允许继续轮询

**修改位置**：`frontend/src/components/Workbench/FlowWorkspace.tsx:570-614`

**关键逻辑**：
```typescript
// Enhanced error handling: If task failed but DAG might still be running
if (taskStatus === 'failed' && checkDagStatus && appNodeConfigs) {
  // Check if any App Node has a dag_run_id (DAG might still be running)
  const hasRunningDag = Object.values(appNodeConfigs).some((config: any) => {
    return config.dagRunId && config.inputMode === 'file'
  })
  
  if (hasRunningDag) {
    // Task failed but DAG might still be running - try to check DAG status
    console.log('[Workflow] Task failed but DAG might still be running, checking DAG status...')
    
    // Update task status to 'running' temporarily to allow polling
    setWorkflowTasks(prev => {
      const next = new Map(prev)
      next.set(taskId, { status: 'running', createdAt: prev.get(taskId)?.createdAt || Date.now() })
      return next
    })
  }
}
```

### 3. 低优先级：添加前端轮询机制

**问题**：
- 前端主要依赖 WebSocket 获取实时更新
- 如果 WebSocket 连接断开或消息丢失，前端无法获取最新状态
- 在 DAG 运行期间（约 4 分钟），前端没有主动轮询机制

**修复**：
- 添加轮询机制，在任务运行期间每 5 秒轮询一次任务状态
- 作为 WebSocket 的补充和降级方案
- 轮询条件：任务状态为 "running" 或 "pending"

**修改位置**：`frontend/src/components/Workbench/FlowWorkspace.tsx:839-850`

**关键逻辑**：
```typescript
// Polling mechanism: Poll task status every 5 seconds when task is running
useEffect(() => {
  if (!activeTaskId) return
  
  const task = workflowTasks.get(activeTaskId)
  const isRunning = task?.status === 'running' || task?.status === 'pending'
  
  if (!isRunning) return
  
  // Poll every 5 seconds
  const pollInterval = setInterval(() => {
    fetchTaskDetail(activeTaskId, true)
  }, 5000)
  
  return () => {
    clearInterval(pollInterval)
  }
}, [activeTaskId, workflowTasks, fetchTaskDetail])
```

## 技术细节

### API 接口说明

**`/workflow-tasks/{task_id}`** (推荐使用)
- 返回完整的任务信息，包括：
  - `task_id`, `status`, `nodes`, `edges`, `input_data`
  - `app_node_configs` (包含 `dagRunId` 等信息)
  - `node_results`, `current_node_id`, `error`
  - `thread_id`, `checkpoint_id`
  - `create_time`, `update_time`, `finished_at`

**`/workflows/tasks/{task_id}`** (保留用于兼容)
- 返回基本信息，不包含 `app_node_configs`
- 函数已重命名为 `get_workflow_task_status` 以避免路由冲突

### 轮询机制

- **轮询间隔**：5 秒
- **轮询条件**：任务状态为 "running" 或 "pending"
- **停止条件**：任务状态变为 "completed" 或 "failed"（且没有运行中的 DAG）
- **与 WebSocket 的关系**：轮询作为 WebSocket 的补充，两者同时工作

### 错误处理增强

- **任务失败但 DAG 运行中**：
  - 检测到 `app_node_configs` 中有 `dagRunId` 且 `inputMode === 'file'`
  - 将任务状态临时更新为 "running" 以继续轮询
  - 后端会在 DAG 完成后更新任务状态

## 测试建议

1. **API 路径统一测试**：
   - 启动工作流后，检查 Network 面板，确认所有请求都使用 `/workflow-tasks/` 接口
   - 不应再出现 404 错误

2. **错误处理测试**：
   - 模拟任务初始化失败（但 DAG 仍在运行）的场景
   - 验证前端是否能正确识别并继续轮询

3. **轮询机制测试**：
   - 启动工作流后，检查 Network 面板，确认每 5 秒有一次轮询请求
   - 断开 WebSocket 连接，验证轮询机制仍能正常工作

4. **DAG 长时间运行测试**：
   - 启动一个需要 4+ 分钟的工作流
   - 验证前端在整个运行期间都能正确获取状态更新

## 相关文件

- `frontend/src/components/Workbench/FlowWorkspace.tsx` - 主要修改文件
- `backend/app/api/v1/workbench.py` - 后端 API 接口（之前已修复数据库连接问题）

## 注意事项

1. 轮询机制会增加服务器负载，但 5 秒间隔是合理的平衡
2. 如果任务数量很多，建议考虑限制同时轮询的任务数量
3. 轮询机制会在任务完成后自动停止，无需手动清理



