# Workflow Debugging Guide

## 1. API 调用方式

### 查看任务详情

**错误方式**（会返回 404）：
```
http://localhost:5173/api/v1/workbench/workflow-tasks/{task_id}
```
原因：5173 是前端端口，不是后端 API 端口。

**正确方式**：

1. **通过浏览器 Network 面板查看**（推荐）：
   - 打开浏览器开发者工具（F12）
   - 切换到 Network 标签
   - 运行工作流后，查找请求：`GET /api/v1/workbench/workflow-tasks/{task_id}`
   - 点击查看 Response，可以看到完整的任务详情

2. **直接访问后端 API**（需要知道后端端口，通常是 8001）：
   ```
   http://localhost:8001/api/v1/workbench/workflow-tasks/{task_id}
   ```
   注意：需要携带 Authorization header（Bearer token）

3. **通过前端代码调用**（已实现）：
   - 前端代码中已有 `fetchTaskDetail` 函数
   - 会自动调用正确的 API 并处理认证

## 2. 查看后端日志

### 方法 1：使用 Docker 日志（推荐）

```bash
# 查看后端容器日志（实时）
docker logs -f foundation-backend-1

# 过滤工作流相关日志
docker logs -f foundation-backend-1 2>&1 | grep -i "workflow\|task_id\|node_id\|dag_run"

# 查看最近 100 行日志
docker logs --tail 100 foundation-backend-1

# 查看特定时间段的日志（需要知道容器名称）
docker logs --since "2025-12-10T09:00:00" --until "2025-12-10T10:00:00" foundation-backend-1
```

### 方法 2：进入容器查看

```bash
# 进入后端容器
docker exec -it foundation-backend-1 /bin/bash

# 查看应用日志文件（如果配置了文件日志）
tail -f /path/to/logfile.log
```

### 方法 3：过滤特定任务 ID 的日志

```bash
# 替换 {task_id} 为实际的任务 ID
docker logs -f foundation-backend-1 2>&1 | grep "{task_id}"
```

### 方法 4：查看 Airflow 日志

```bash
# 查看 Airflow 容器日志
docker logs -f foundation-airflow-1

# 过滤特定 DAG 运行
docker logs -f foundation-airflow-1 2>&1 | grep "agent_file_ingest_unified"
```

## 3. 关键日志标识

工作流执行过程中，后端会输出以下关键日志：

- `[WF][app]` - App Node 执行日志
- `[WF][dataTransform]` - Data Transform Node 执行日志
- `Workflow execution started` - 工作流开始执行
- `Workflow execution completed` - 工作流执行完成
- `Error executing workflow task` - 工作流执行错误

## 4. 前端调试信息

### 查看 dag_run_id

1. **在 App Node 配置面板**：
   - 上传文件成功后，会在文件列表下方显示绿色的 "Files Uploaded Successfully" 卡片
   - 卡片中会显示 DAG Run ID

2. **在浏览器控制台**：
   - 打开开发者工具（F12）
   - 切换到 Console 标签
   - 查找日志：`[Workflow] App Node ... file upload successful`
   - 日志中会包含 `dagRunId` 字段

3. **在 Toast 提示中**：
   - 文件上传成功后会显示绿色提示
   - 提示中会显示 DAG Run ID

### 查看 task_id

1. **在右上角 Recent Tasks 面板**：
   - 运行工作流后，会显示任务 ID（前 6 位）

2. **在浏览器控制台**：
   - 查找日志：`[Workflow] Workflow execution started`
   - 日志中会包含 `taskId` 字段

3. **在 URL 参数中**：
   - 运行工作流后，URL 会包含 `?task_id=xxx` 参数

## 5. 常见问题排查

### 问题：App Node 输出为空

**排查步骤**：
1. 检查 dag_run_id 是否正确获取：
   - 查看浏览器控制台日志
   - 检查 App Node 配置面板是否显示 DAG Run ID
2. 检查后端日志：
   ```bash
   docker logs -f foundation-backend-1 2>&1 | grep "dag_run_id\|_poll_dag_status"
   ```
3. 检查 Airflow DAG 状态：
   - 访问 Airflow UI：http://localhost:9090
   - 查看 DAG `agent_file_ingest_unified` 的运行状态
4. 检查任务详情 API 返回：
   - 通过浏览器 Network 面板查看 `GET /api/v1/workbench/workflow-tasks/{task_id}` 的响应
   - 检查 `node_results` 字段是否包含 App Node 的输出

### 问题：Recent Tasks 显示 FAILED

**排查步骤**：
1. 查看任务详情中的 `error` 字段：
   ```bash
   # 通过浏览器 Network 面板查看任务详情响应
   # 或直接访问后端 API（需要 token）
   ```
2. 查看后端错误日志：
   ```bash
   docker logs -f foundation-backend-1 2>&1 | grep "Error executing workflow task"
   ```
3. 检查 WebSocket 消息：
   - 打开浏览器控制台
   - 查找 WebSocket 相关的错误消息

### 问题：dag_run_id 未显示

**排查步骤**：
1. 检查文件上传是否成功：
   - 查看浏览器控制台是否有上传错误
   - 检查 Toast 提示是否显示上传成功
2. 检查 appNodeId 是否正确：
   - 在浏览器控制台查看日志中的 `appNodeId` 字段
   - 确认是否与 Airflow DAG 配置中的 `app_node_id` 一致
3. 检查上传 API 响应：
   - 在浏览器 Network 面板查看 `POST /api/v1/report/{disk_id}/upload-files` 的响应
   - 确认响应中是否包含 `dag_run_id` 字段

## 6. 日志查看命令汇总

```bash
# 查看所有工作流相关日志
docker logs -f foundation-backend-1 2>&1 | grep -E "workflow|task_id|node_id|dag_run|WF\["

# 查看特定任务 ID 的日志（替换 {task_id}）
docker logs -f foundation-backend-1 2>&1 | grep "{task_id}"

# 查看 App Node 执行日志
docker logs -f foundation-backend-1 2>&1 | grep "\[WF\]\[app\]"

# 查看 Data Transform Node 执行日志
docker logs -f foundation-backend-1 2>&1 | grep "\[WF\]\[dataTransform\]"

# 查看错误日志
docker logs -f foundation-backend-1 2>&1 | grep -i "error\|exception\|failed"

# 查看最近 200 行日志
docker logs --tail 200 foundation-backend-1
```

## 7. 容器名称确认

如果容器名称不是 `foundation-backend-1`，请先确认容器名称：

```bash
# 列出所有运行中的容器
docker ps

# 查找后端容器
docker ps | grep backend

# 查找 Airflow 容器
docker ps | grep airflow
```

然后替换上述命令中的容器名称。



