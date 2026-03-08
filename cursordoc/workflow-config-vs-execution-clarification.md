# Workflow配置阶段与执行阶段澄清

## 问题背景

用户在配置 App Node 时（选择 inputMode、上传文件），发现这些配置参数最终出现在 Start Node 的 output 中，产生了以下疑问：
1. 配置 App Node 时发生了什么？保存了什么？保存在哪里？
2. 为什么 App Node 的配置参数会出现在 Start Node 的 output 里？
3. 工作流上下游节点配置的逻辑是否正确？

---

## 核心澄清：配置阶段 vs 执行阶段

### 关键区别

| 阶段 | 配置阶段 | 执行阶段 |
|------|---------|---------|
| **时间点** | 用户在设计工作流时配置节点 | 用户点击"运行工作流"时 |
| **数据位置** | 节点的 `data.executionState` | `app_node_configs` 和 `input_data` |
| **存储位置** | 工作流定义（数据库中的 `workflow_templates` 或 `workflow_instances`） | `WorkflowExecutionTask` 表 |
| **数据用途** | 定义节点如何工作 | 实际执行时的输入参数 |

---

## 一、配置阶段：App Node 配置发生了什么？

### 1.1 用户操作流程

当用户在界面上配置 App Node 时（如图片所示）：

1. **选择 Input Mode**：用户选择 `'file'` 或 `'url'`
2. **上传文件**（如果是 file 模式）：用户选择文件，如 `供应商B基本信息.docx`
3. **填写 Requirement**（可选）：用户输入分析需求文本

### 1.2 前端处理

**文件位置**：`frontend/src/components/Workbench/AppNodeConfigPanel.tsx`

```typescript
// 用户操作会更新节点的 executionState
const handleInputModeChange = (mode: 'url' | 'file') => {
  updateState({ inputMode: mode })
  onUpdate(nodeId, {
    executionState: {
      ...state,
      inputMode: mode,
      url: mode === 'file' ? '' : state.url,
      files: mode === 'url' ? [] : state.files,
    },
  })
}

const handleFileUpload = (files: File[]) => {
  updateState({ files })
  onUpdate(nodeId, {
    executionState: {
      ...state,
      files,
    },
  })
}
```

### 1.3 保存的内容

**前端内存中的数据结构**（存储在节点的 `data.executionState` 中）：

```typescript
{
  executionState: {
    inputMode: 'file',  // 或 'url'
    files: [File对象],   // 文件对象列表（仅在前端内存中，不保存到数据库）
    requirement: '分析供应商信息',  // 用户输入的需求文本
    status: 'idle',     // 执行状态
    dagRunId: undefined,  // 此时还没有（执行时才会生成）
  }
}
```

**实际保存到数据库的内容**（验证结果）：

```json
{
  "executionState": {
    "status": "idle",
    "inputMode": "file",
    "requirement": ""
  }
}
```

**重要说明**：
- ✅ **保存**：`inputMode`、`requirement`、`status`
- ❌ **不保存**：`files`（文件对象列表）、`dagRunId`（执行时生成）、执行结果等

### 1.4 保存形式

- **前端**：TypeScript 对象，存储在 React 组件的 state 中
- **传输到后端**：JSON 格式
- **持久化**：作为工作流定义的一部分，保存在数据库的 `workflow_templates` 或 `workflow_instances` 表中

### 1.5 保存位置

**数据库表**：
- `workflow_templates`：工作流模板定义
- `workflow_instances`：工作流实例定义

**存储字段**：
- `workflow_nodes`：JSONB 数组，包含所有节点的定义
- 每个节点的 `data.executionState` 字段包含配置信息

**实际数据库验证结果**（基于 `workflow_templates` 表，id=8）：

```json
{
  "executionState": {
    "status": "idle",
    "inputMode": "file",
    "requirement": ""
  }
}
```

**关键发现**：
1. ✅ **保存的内容**：`inputMode`、`requirement`、`status`
2. ❌ **不保存的内容**：
   - `files`：文件对象列表**不会**保存到数据库
   - `dagRunId`：执行时才会生成，配置阶段不存在
   - `url`：URL 模式下的 URL 地址（如果存在，会保存）
3. **文件处理**：
   - 文件本身**不会**直接保存到数据库
   - 文件会先上传到对象存储（MinIO）或文件系统
   - 数据库中只保存配置元数据（`inputMode`、`requirement`），不保存文件引用
4. **执行痕迹清理**：
   - 根据配置保留策略（选项A），执行痕迹（如 `dagRunId`、`markdownResult`）会被清空
   - 只保留用户配置（`inputMode`、`requirement`）

---

## 二、执行阶段：数据如何流转？

### 2.1 执行流程概览

当用户点击"运行工作流"时：

```
1. 前端收集配置 → 2. 上传文件获取 dagRunId → 3. 构建 app_node_configs → 4. 调用后端API → 5. 后端创建初始状态 → 6. Start Node 执行
```

### 2.2 前端执行逻辑

**文件位置**：`frontend/src/components/Workbench/FlowWorkspace.tsx` - `handleRunWorkflow()` 方法

#### 步骤1：文件上传（如果是 file 模式）

```typescript
// 为每个 App Node（file 模式）上传文件
for (const node of nodes) {
  if (node.type !== 'application') continue
  const appData = node.data as ApplicationNodeData
  const executionState = appData.executionState
  
  if (executionState.inputMode === 'file') {
    // 上传文件到后端，获取 dagRunId
    const uploadResp = await ReportService.uploadMultipleFiles(
      appData.appNodeId || 0,
      executionState.files,
      executionState.requirement || ''
    )
    const dagRunId = uploadResp.dag_run_id
    fileDagRunMap[node.id] = dagRunId  // 保存 dagRunId
  }
}
```

#### 步骤2：收集 App Node 配置

```typescript
// 收集所有 App Node 的配置到 app_node_configs
const appNodeConfigs: Record<string, any> = {}
for (const node of nodes) {
  if (node.type === 'application') {
    const appData = node.data as ApplicationNodeData
    const executionState = appData.executionState
    
    if (executionState?.inputMode === 'url') {
      appNodeConfigs[node.id] = {
        inputMode: 'url',
        url: executionState.url,
        requirement: executionState.requirement || '',
        appNodeId: appData.appNodeId,
      }
    } else { // file
      const dagRunId = fileDagRunMap[node.id] || executionState?.dagRunId
      appNodeConfigs[node.id] = {
        inputMode: 'file',
        dagRunId,  // 从文件上传获取的 dagRunId
        requirement: executionState.requirement || '',
        appNodeId: appData.appNodeId,
      }
    }
  }
}
```

#### 步骤3：调用后端API

```typescript
// 调用后端执行API
const response = await fetch(`/api/v1/workbench/workflows/execute`, {
  method: 'POST',
  body: JSON.stringify({
    nodes: nodes,  // 节点定义
    edges: edges,  // 边定义
    input_data: {},  // 通常为空字典
    app_node_configs: appNodeConfigs,  // App Node 配置
  }),
})
```

### 2.3 后端处理逻辑

**文件位置**：`backend/app/api/v1/workbench.py` - `execute_workflow_task()` 方法

#### 步骤1：保存到数据库

```python
# 创建 WorkflowExecutionTask 记录
task = WorkflowExecutionTask(
    task_id=task_id,
    nodes=request.nodes,
    edges=request.edges,
    input_data=request.input_data,  # 通常是 {}
    app_node_configs=request.app_node_configs,  # 包含所有 App Node 的配置
    status="pending",
)
session.add(task)
session.commit()
```

#### 步骤2：创建初始状态

```python
# 创建初始工作流状态
initial_state = {
    "instance_id": 0,
    "workflow_task_id": task.id,
    "created_by_user_id": str(user_id),
    "input_data": input_data,  # 从 request.input_data 获取，通常是 {}
    "workflow_definition": {
        "nodes": nodes,
        "edges": edges
    },
    "node_results": {},
    "current_node_id": None,
    "structured_data": {},
    "execution_history": [],
    "errors": []
}
```

**关键发现**：
- `initial_state['input_data']` 来自 `request.input_data`，通常是空字典 `{}`
- `app_node_configs` 是**单独存储**的，**不会**自动合并到 `input_data` 中

#### 步骤3：Start Node 执行

**文件位置**：`backend/app/services/workflow/workflow_execution_service.py` - `_create_start_node()` 方法

```python
async def start_node(state: WorkflowState) -> WorkflowState:
    state['current_node_id'] = node['id']
    input_data = state.get('input_data', {})  # 从 state 中获取
    state['node_results'][node['id']] = {
        'status': 'completed',
        'output': input_data  # 直接输出 input_data
    }
    return state
```

**关键发现**：
- Start Node **只输出** `state['input_data']`
- 如果 `input_data` 是空字典，Start Node 的输出也应该是空字典
- **但是**，用户看到的 Start Node 输出包含了 `files`、`dagRunId`、`inputMode` 等字段

---

## 三、关键问题：为什么 App Node 配置会出现在 Start Node 输出中？

### 3.1 基于代码验证的事实

**代码验证结果**：

1. **前端代码**（`frontend/src/components/Workbench/FlowWorkspace.tsx:979`）：
   ```typescript
   body: JSON.stringify({
     nodes: nodes.map(n => ({ id: n.id, type: n.type, data: n.data })),
     edges: edges.map(e => ({...})),
     input_data: {},  // ✅ 明确设置为空字典
     app_node_configs: appNodeConfigs,  // ✅ 单独传递
   })
   ```
   **事实**：前端将 `input_data` 设置为空字典 `{}`，`app_node_configs` 单独传递。

2. **后端代码**（`backend/app/api/v1/workbench.py:908`）：
   ```python
   input_data = request.input_data or {}  # ✅ 直接使用请求中的 input_data
   ```
   **事实**：后端直接使用请求中的 `input_data`，不进行任何合并操作。

3. **初始状态创建**（`backend/app/api/v1/workbench.py:1087`）：
   ```python
   initial_state = {
       "input_data": input_data,  # ✅ 直接使用 input_data（空字典）
       # ... 其他字段
   }
   ```
   **事实**：初始状态中的 `input_data` 直接来自请求，不进行合并。

4. **Start Node 执行**（`backend/app/services/workflow/workflow_execution_service.py:519`）：
   ```python
   async def start_node(state: WorkflowState) -> WorkflowState:
       input_data = state.get('input_data', {})  # ✅ 从 state 获取
       state['node_results'][node['id']] = {
           'status': 'completed',
           'output': input_data  # ✅ 直接输出 input_data
       }
   ```
   **事实**：Start Node 直接输出 `state['input_data']`，不做任何处理。

5. **数据库验证**（`workflow_execution_tasks` 表）：
   ```json
   {
     "input_data": {},
     "app_node_configs": {
       "app-1": {
         "dagRunId": "manual__2026-01-22T07:24:53.319169+00:00",
         "appNodeId": 105,
         "inputMode": "file",
         "requirement": ""
       }
     }
   }
   ```
   **事实**：数据库中 `input_data` 是空字典 `{}`，`app_node_configs` 单独存储。

### 3.2 基于事实的结论

**确定的结论**：

1. ✅ **Start Node 的输出应该是空字典 `{}`**
   - 代码逻辑：Start Node 输出 `state['input_data']`
   - 数据库验证：`input_data = {}`
   - 因此：Start Node 输出 = `{}`

2. ✅ **`app_node_configs` 不会合并到 `input_data`**
   - 前端代码：`input_data: {}`，`app_node_configs` 单独传递
   - 后端代码：`input_data = request.input_data or {}`，不进行合并
   - 数据库验证：`input_data` 和 `app_node_configs` 分别存储

3. ✅ **实际数据流（基于代码和数据库验证）**：
   ```
   1. 前端收集 app_node_configs
      ↓
   2. 前端调用 API：input_data = {}, app_node_configs = {...}
      ↓
   3. 后端保存到数据库：input_data = {}, app_node_configs = {...}
      ↓
   4. 后端创建初始状态：state['input_data'] = {}
      ↓
   5. Start Node 执行：output = state['input_data'] = {}
      ↓
   6. App Node 执行：从 app_node_configs 读取配置（包含 dagRunId）
   ```

### 3.3 如果用户看到 Start Node 输出包含其他字段

**需要进一步调查的情况**：

如果用户实际看到的 Start Node 输出包含 `files`、`dagRunId`、`inputMode` 等字段，这**不符合代码逻辑**。需要调查：

1. **检查前端显示逻辑**：
   - 前端是否正确显示 Start Node 的输出？
   - 是否混淆了不同节点的输出（例如显示了 App Node 的配置）？

2. **检查实际执行日志**：
   - 查看后端日志，确认 Start Node 的实际输出
   - 查看数据库中的 `node_results`，确认 Start Node 的实际输出

3. **检查是否有其他代码路径**：
   - 是否有其他 API 端点会设置 `input_data`？
   - 是否有其他代码会修改 `input_data`？

**注意**：基于当前代码和数据库验证，Start Node 的输出应该是空字典 `{}`。如果实际看到其他内容，需要进一步调查，而不是基于推测。

---

## 四、工作流上下游节点配置逻辑澄清

### 4.1 设计理念

工作流的设计遵循以下原则：

1. **配置与执行分离**：
   - 配置阶段：定义节点如何工作（保存在节点定义中）
   - 执行阶段：提供实际输入数据（通过 `app_node_configs` 和 `input_data`）

2. **数据流向**：
   - Start Node → App Node → 其他节点
   - 每个节点从上游节点的输出中获取输入

3. **App Node 的特殊性**：
   - App Node 可以通过 `app_node_configs` 直接获取配置（file/url 模式）
   - 也可以从上游节点的输出中获取输入数据

### 4.2 基于代码验证的设计逻辑

**代码验证结果**：

1. **Start Node 的输出**：
   - 代码：`output: input_data`
   - 数据库验证：`input_data = {}`
   - **结论**：Start Node 输出空字典 `{}`

2. **App Node 的配置**：
   - 代码：从 `app_node_configs` 读取配置
   - 数据库验证：`app_node_configs` 包含 `dagRunId`、`inputMode`、`requirement` 等
   - **结论**：App Node 从 `app_node_configs` 读取配置，不依赖 Start Node 输出

3. **数据流向**：
   - Start Node → App Node：Start Node 输出空字典，App Node 不依赖此输出
   - App Node 执行：直接从 `app_node_configs` 读取配置（包含 `dagRunId`）

**设计逻辑**：
- ✅ **配置与执行分离**：配置保存在 `app_node_configs`，执行数据在 `input_data`
- ✅ **数据流向清晰**：Start Node 输出 `input_data`（空字典），App Node 从 `app_node_configs` 读取配置
- ✅ **逻辑正确**：基于代码和数据库验证，设计逻辑是正确的

---

## 五、基于事实的总结

### 5.1 代码和数据库验证的确定事实

1. **配置阶段**：
   - 保存内容：`inputMode`、`requirement`、`status`
   - 不保存：`files`、`dagRunId`（执行时生成）
   - 存储位置：`workflow_templates.workflow_nodes[].data.executionState`

2. **执行阶段**：
   - `input_data`：空字典 `{}`（代码和数据库验证确认）
   - `app_node_configs`：包含执行时的完整配置（`dagRunId`、`inputMode`、`requirement`、`appNodeId`）
   - 存储位置：`workflow_execution_tasks` 表

3. **Start Node 输出**：
   - 代码逻辑：`output = state['input_data']`
   - 数据库验证：`input_data = {}`
   - **确定结论**：Start Node 输出 = `{}`

4. **App Node 执行**：
   - 代码逻辑：从 `app_node_configs` 读取配置
   - 数据库验证：`app_node_configs` 包含完整配置
   - **确定结论**：App Node 从 `app_node_configs` 读取配置，不依赖 Start Node 输出

### 5.2 数据流（基于代码和数据库验证）

```
配置阶段：
  用户配置 App Node (inputMode, requirement)
    ↓
  保存到 workflow_templates.workflow_nodes[].data.executionState
    ↓
  只保存：inputMode, requirement, status
    ↓
  不保存：files, dagRunId

执行阶段：
  前端收集 app_node_configs
    ↓
  上传文件，获取 dagRunId
    ↓
  调用 API：input_data = {}, app_node_configs = {dagRunId, inputMode, requirement, appNodeId}
    ↓
  后端保存：input_data = {}, app_node_configs = {...}
    ↓
  创建初始状态：state['input_data'] = {}
    ↓
  Start Node 执行：output = state['input_data'] = {}
    ↓
  App Node 执行：从 app_node_configs 读取配置（包含 dagRunId）
```

### 5.3 需要进一步调查的情况

如果用户实际看到的 Start Node 输出包含 `files`、`dagRunId`、`inputMode` 等字段：

1. **检查前端显示逻辑**：
   - 查看前端代码中显示 Start Node 输出的逻辑
   - 确认是否混淆了不同节点的输出

2. **检查实际执行日志**：
   - 查看后端日志中 Start Node 的实际输出
   - 查看数据库中 `node_results` 的实际内容

3. **检查是否有其他代码路径**：
   - 搜索代码库中是否有其他修改 `input_data` 的逻辑
   - 检查是否有其他 API 端点会设置 `input_data`

---

## 六、总结

### 6.1 配置阶段（数据库验证结果）

**保存内容**：
- ✅ `inputMode`：用户选择的输入模式（'file' 或 'url'）
- ✅ `requirement`：用户输入的需求文本
- ✅ `status`：执行状态（'idle'）
- ❌ **不保存**：`files`（文件对象列表）、`dagRunId`（执行时生成）、执行结果等

**保存形式**：JSONB 格式（PostgreSQL）

**保存位置**：
- 表：`workflow_templates` 或 `workflow_instances`
- 字段：`workflow_nodes`（JSONB 数组）
- 路径：`workflow_nodes[].data.executionState`

**实际数据库示例**（`workflow_templates.id = 8`）：
```json
{
  "executionState": {
    "status": "idle",
    "inputMode": "file",
    "requirement": ""
  }
}
```

### 6.2 执行阶段（数据库验证结果）

**数据来源**：
- `app_node_configs`：App Node 的配置（包含 `dagRunId`、`inputMode`、`requirement`、`appNodeId`）
- `input_data`：工作流的初始输入数据（**验证结果：空字典 `{}`**）

**保存位置**：
- 表：`workflow_execution_tasks`
- 字段：`app_node_configs`（JSONB）、`input_data`（JSONB）

**实际数据库示例**（`workflow_execution_tasks.task_id = be3d292f-46bc-4308-a99d-58d55fe06e79`）：
```json
{
  "app_node_configs": {
    "app-1": {
      "dagRunId": "manual__2026-01-22T07:24:53.319169+00:00",
      "appNodeId": 105,
      "inputMode": "file",
      "requirement": ""
    }
  },
  "input_data": {}
}
```

**数据流转**：
- Start Node 输出 `input_data`（根据验证，通常是空字典 `{}`）
- App Node 从 `app_node_configs` 读取配置（包含 `dagRunId`）

### 6.3 关键澄清（基于数据库验证）

1. ✅ **配置阶段只保存元数据**：`inputMode`、`requirement`，不保存文件
2. ✅ **执行阶段 `input_data` 是空字典**：验证结果确认 `input_data = {}`
3. ✅ **Start Node 输出应该是空字典**：因为 `input_data` 是空字典
4. ✅ **App Node 配置通过 `app_node_configs` 传递**：包含执行时的完整配置（包括 `dagRunId`）
5. ⚠️ **如果用户看到 Start Node 输出包含 `files`、`dagRunId` 等**：这不符合代码逻辑，需要进一步调查前端显示逻辑或实际执行日志

### 6.4 设计合理性

- **逻辑正确**：工作流的设计是合理的，配置与执行阶段的数据分离清晰
- **数据库验证**：证实了配置阶段和执行阶段的数据存储符合设计预期
- **建议改进**：
  1. 前端显示逻辑：检查是否正确显示 Start Node 的输出
  2. 代码注释：添加注释说明 `input_data` 和 `app_node_configs` 的区别
  3. 文档更新：已根据数据库验证结果更新文档

---

## 七、相关代码位置

### 前端
- **配置保存**：`frontend/src/components/Workbench/AppNodeConfigPanel.tsx`
- **执行逻辑**：`frontend/src/components/Workbench/FlowWorkspace.tsx` - `handleRunWorkflow()`

### 后端
- **API 接收**：`backend/app/api/v1/workbench.py` - `execute_workflow()`
- **任务执行**：`backend/app/api/v1/workbench.py` - `execute_workflow_task()`
- **Start Node**：`backend/app/services/workflow/workflow_execution_service.py` - `_create_start_node()`
- **App Node**：`backend/app/services/workflow/workflow_execution_service.py` - `_create_app_node()`

---

## 八、实际代码验证结果

### 8.1 前端代码验证

**文件位置**：`frontend/src/components/Workbench/FlowWorkspace.tsx` - `handleRunWorkflow()` 方法

**验证结果**：
```typescript
// 前端调用 API 时
body: JSON.stringify({
  nodes: nodes.map(n => ({ id: n.id, type: n.type, data: n.data })),
  edges: edges.map(e => ({...})),
  input_data: {},  // ✅ 明确设置为空字典
  app_node_configs: appNodeConfigs,  // ✅ 单独传递
})
```

**结论**：前端**不会**将 `app_node_configs` 合并到 `input_data` 中。

### 8.2 后端代码验证

**文件位置**：`backend/app/api/v1/workbench.py` - `execute_workflow_task()` 方法

**验证结果**：
```python
# 创建初始状态时
initial_state = {
    "input_data": input_data,  # 直接使用 request.input_data，通常是 {}
    # ... 其他字段
}
```

**结论**：后端**不会**将 `app_node_configs` 合并到 `input_data` 中。

### 8.3 Start Node 代码验证

**文件位置**：`backend/app/services/workflow/workflow_execution_service.py` - `_create_start_node()` 方法

**验证结果**：
```python
async def start_node(state: WorkflowState) -> WorkflowState:
    input_data = state.get('input_data', {})  # 从 state 中获取
    state['node_results'][node['id']] = {
        'status': 'completed',
        'output': input_data  # 直接输出 input_data
    }
```

**结论**：Start Node **只输出** `state['input_data']`，不做任何处理。

### 8.4 关键发现

**如果 `input_data` 为空字典 `{}`，Start Node 的输出也应该是空字典 `{}`。**

**但是**，如果用户看到的 Start Node 输出包含了 `files`、`dagRunId`、`inputMode` 等字段，这**不符合代码逻辑**。

**基于代码和数据库验证的事实**：
- 代码逻辑：Start Node 输出 `state['input_data']`，而 `input_data = {}`
- 数据库验证：`input_data = {}`
- **确定结论**：Start Node 输出应该是空字典 `{}`

**如果实际看到其他内容，需要进一步调查**：
1. 检查前端显示逻辑：是否正确显示 Start Node 的输出？
2. 检查实际执行日志：查看后端日志和数据库中的实际输出
3. 检查是否有其他代码路径：是否有其他代码会修改 `input_data`？

### 8.5 数据库验证结果总结

**验证时间**：2026-01-22

**验证数据库**：
- 地址：localhost:5434
- 数据库：app
- 表：`workflow_templates`、`workflow_execution_tasks`

**验证结果**：

#### 1. 配置阶段保存的内容（`workflow_templates.workflow_nodes`）

**实际保存的数据**：
```json
{
  "executionState": {
    "status": "idle",
    "inputMode": "file",
    "requirement": ""
  }
}
```

**结论**：
- ✅ 保存了 `inputMode`（用户选择的输入模式）
- ✅ 保存了 `requirement`（用户输入的需求文本）
- ✅ 保存了 `status`（执行状态）
- ❌ **不保存** `files`（文件对象列表）
- ❌ **不保存** `dagRunId`（执行时生成）
- ❌ **不保存** `url`（URL 模式下的 URL，如果存在会保存）

#### 2. 执行阶段保存的内容（`workflow_execution_tasks`）

**实际保存的数据**（示例，task_id: be3d292f-46bc-4308-a99d-58d55fe06e79）：

```json
{
  "app_node_configs": {
    "app-1": {
      "dagRunId": "manual__2026-01-22T07:24:53.319169+00:00",
      "appNodeId": 105,
      "inputMode": "file",
      "requirement": ""
    }
  },
  "input_data": {}
}
```

**结论**：
- ✅ `app_node_configs` 包含执行时的配置（包括 `dagRunId`）
- ✅ `input_data` 确实是空字典 `{}`
- ✅ 这证实了 Start Node 的输出应该也是空字典（如果 `input_data` 为空）

#### 3. 关键发现

1. **配置与执行分离**：
   - 配置阶段：只保存用户配置（`inputMode`、`requirement`）
   - 执行阶段：`app_node_configs` 包含执行时的完整配置（包括 `dagRunId`）

2. **文件处理**：
   - 文件不会保存到数据库
   - 文件上传到对象存储后，生成 `dagRunId`
   - `dagRunId` 保存在 `app_node_configs` 中，不保存在 `input_data` 中

3. **Start Node 输出**：
   - 根据代码，Start Node 输出 `input_data`
   - 根据数据库验证，`input_data` 是空字典 `{}`
   - 因此，Start Node 的输出应该是空字典 `{}`
   - 如果用户看到 Start Node 输出包含 `files`、`dagRunId` 等，这不符合代码逻辑，需要进一步调查前端显示逻辑或实际执行日志

### 8.6 建议

1. **前端显示逻辑检查**：
   - 查看前端代码中显示 Start Node 输出的逻辑
   - 确认是否混淆了不同节点的输出（例如显示了 App Node 的配置）

2. **检查实际执行日志**：
   - 查看后端日志中 Start Node 的实际输出
   - 查看数据库中 `node_results` 的实际内容
   - 对比代码逻辑和实际执行结果

3. **文档更新**：
   - 已根据代码和数据库验证结果更新文档
   - 明确说明配置阶段和执行阶段保存的不同内容
   - 所有结论均基于代码和数据库验证，无推测内容

---

## 九、实际执行数据流转示例（基于日志和数据库验证）

### 9.1 执行路径

**实际执行路径**（基于日志验证）：
```
start-2 → app-1 → questionClassifier-3 → humanInTheLoop-6 (request_for_update) → end-7
```

**执行顺序**（基于日志）：
```
Execution order: ['start-2', 'app-1', 'questionClassifier-3', 'humanInTheLoop-4', 'humanInTheLoop-6', 'end-5', 'end-7']
实际执行路径: ['start-2', 'app-1', 'questionClassifier-3', 'humanInTheLoop-6', 'end-7']
```

### 9.2 各节点实际输入输出数据（基于数据库验证）

#### 9.2.1 Start Node (start-2)

**输入**：
```python
{
    # state['input_data']
    # 实际验证结果：{}（空字典）
}
```

**输出**（基于数据库验证）：
```python
{
    'status': 'completed',
    'output': {}  # 空字典
}
```

**日志验证**：
- `[WF][start] node_id=start-2 keys=[] len=0`

---

#### 9.2.2 App Node (app-1)

**输入**：
```python
{
    # 从 start-2 获取
    # 实际验证结果：{}（空字典）
    # 日志：prev=['start-2'] keys=[] len=0
}
```

**配置数据**（从 `app_node_configs` 获取）：
```python
{
    'dagRunId': 'manual__2026-01-22T09:29:20.874721+00:00',
    'inputMode': 'file',
    'requirement': '',
    'appNodeId': 105
}
```

**输出**（基于数据库验证）：
```python
{
    'status': 'completed',
    'output': {
        'conclusion_detail': 'Summary\n- Source materials: Three child-node outputs...',  # 6974字符
        'dag_run_id': 'manual__2026-01-22T09:29:20.874721+00:00',
        'structured_output': {},
        'upstream_context': {},
        'merged_context': {},
        '_metadata': {
            'node_id': 'app-1',
            'node_type': 'app',
            'executed_at': '2026-01-22T09:36:28.784361Z',
            'context_chain': ['start-2', 'app-1'],
            'schema_version': '3.0',
            'upstream_node_id': 'start-2'
        }
    }
}
```

**日志验证**：
- `[WF][app] node_id=app-1 prev=['start-2'] keys=[] len=0`
- `[WF][app] node_id=app-1 markdown_result extracted: type=str, length=6974`
- `[WF][app] node_id=app-1 final_output created: keys=['conclusion_detail', 'dag_run_id', 'structured_output', 'upstream_context', 'merged_context', '_metadata']`

---

#### 9.2.3 Question Classifier Node (questionClassifier-3)

**输入**：
```python
{
    # 从 app-1 获取完整 output
    # 提取逻辑：优先从 conclusion_detail 提取（6974字符）
    # 实际验证结果：从 app-1 的 conclusion_detail 提取了6974字符的文本
}
```

**配置数据**（从节点定义中获取）：
```python
{
    'classes': [
        {'id': 'class_1', 'name': 'supplier admission approval', 'description': '...'},
        {'id': 'class_2', 'name': 'supplier information update', 'description': '...'}
    ],
    'instruction': '如果输入同时包含批准相关词汇和缺失材料信息,优先选择 class_1...',
    'llmConfig': {'model': 'gpt5', 'temperature': 0}
}
```

**LLM 分类结果**（基于日志）：
```python
{
    'class_id': 'class_2',
    'reason': '结论为暂不授标/拒绝当前入围，明确指出因材料缺失需发RFI并补充并验证所需文件后才能重新审核或考虑有条件准入，属于要求补充材料后重审的情形。'
}
```

**输出**（基于数据库验证）：
```python
{
    'status': 'completed',
    'output': {
        'branch': 'class_2',
        'class_id': 'class_2',
        'class_name': 'supplier information update',
        'structured_output': {
            'selected_class': 'class_2',
            'selected_class_name': 'supplier information update',
            'classification_reason': '结论为暂不授标/拒绝当前入围，明确指出因材料缺失需发RFI并补充并验证所需文件后才能重新审核或考虑有条件准入，属于要求补充材料后重审的情形。',
            'available_classes': ['class_1', 'class_2']
        },
        'upstream_context': {},
        'merged_context': {
            'selected_class': 'class_2',
            'available_classes': ['class_1', 'class_2'],
            'selected_class_name': 'supplier information update',
            'classification_reason': '结论为暂不授标/拒绝当前入围，明确指出因材料缺失需发RFI并补充并验证所需文件后才能重新审核或考虑有条件准入，属于要求补充材料后重审的情形。'
        },
        '_metadata': {
            'node_id': 'questionClassifier-3',
            'node_type': 'questionClassifier',
            'executed_at': '2026-01-22T09:36:41.208102Z',
            'context_chain': ['start-2', 'app-1', 'questionClassifier-3'],
            'schema_version': '3.0',
            'upstream_node_id': 'app-1'
        }
    }
}
```

**日志验证**：
- `[WF][classifier] node_id=questionClassifier-3 Got upstream output from app-1`
- `[WF][classifier] node_id=questionClassifier-3 Input text length: 6974`
- `[WF][classifier] node_id=questionClassifier-3 LLM response: {'class_id': 'class_2', 'reason': '...'}`
- `[WF][classifier] node_id=questionClassifier-3 Classification result: class_2 (supplier information update)`
- `[WF][classifier] node_id=questionClassifier-3 Routing to branch=class_2`

---

#### 9.2.4 Human In The Loop Node (humanInTheLoop-6, taskType=request_for_update)

**输入**：
```python
{
    # 从 questionClassifier-3 获取
    # 提取逻辑：从 questionClassifier-3 的 merged_context 提取
    # 实际验证结果（基于日志和数据库）：
    {
        'branch': 'class_2',
        'class_id': 'class_2',
        'class_name': 'supplier information update',
        'structured_output': {
            'selected_class': 'class_2',
            'selected_class_name': 'supplier information update',
            'classification_reason': '结论为暂不授标/拒绝当前入围，明确指出因材料缺失需发RFI并补充并验证所需文件后才能重新审核或考虑有条件准入，属于要求补充材料后重审的情形。',
            'available_classes': ['class_1', 'class_2']
        },
        'upstream_context': {},
        'merged_context': {
            'selected_class': 'class_2',
            'available_classes': ['class_1', 'class_2'],
            'selected_class_name': 'supplier information update',
            'classification_reason': '...'
        },
        '_metadata': {
            'node_id': 'questionClassifier-3',
            'node_type': 'questionClassifier',
            'executed_at': '2026-01-22T09:36:41.208102Z',
            'context_chain': ['start-2', 'app-1', 'questionClassifier-3'],
            'schema_version': '3.0',
            'upstream_node_id': 'app-1'
        }
    }
}
```

**配置数据**（从节点定义中获取）：
```python
{
    'taskType': 'request_for_update',
    'updateConfig': {
        'updateJobTitleIds': [1],
        'title': 'supplier information update',
        'description': '22'
    },
    'timeout': 43200,  # 12小时
    'timeoutAction': 'fail'
}
```

**用户提交数据**（等待用户输入后）：
```python
{
    'decision': 'updated',
    'update_info': 'yy',
    'file_ingestion_record_ids': [318],
    'comments': 'yy'
}
```

**输出**（基于数据库验证）：
```python
{
    'status': 'completed',
    'output': {
        'human_decision': 'updated',
        'decision_reason': 'yy',
        'structured_output': {
            'task_type': 'request_for_update',
            'is_updated': True,
            'update_info': 'yy',
            'human_decision': 'updated',
            'reviewer_email': 'huangpan2507@126.com',
            'reviewer_job_title': 'Group Director of Operations',
            'rejection_reason': None,
            'decision_timestamp': '2026-01-22T09:38:15.673603+00:00',
            'file_ingestion_record_ids': [318]
        },
        'upstream_context': {
            # 包含 questionClassifier-3 的完整输出
            'branch': 'class_2',
            'class_id': 'class_2',
            'class_name': 'supplier information update',
            'structured_output': {...},
            'upstream_context': {},
            'merged_context': {...},
            '_metadata': {...}
        },
        'merged_context': {
            # 包含上游上下文 + 当前节点的输出
            'branch': 'class_2',
            'class_id': 'class_2',
            'class_name': 'supplier information update',
            'task_type': 'request_for_update',
            'human_decision': 'updated',
            'is_updated': True,
            'update_info': 'yy',
            'merged_context': {...},  # 嵌套的 merged_context
            'structured_output': {...},
            '_metadata': {...}
        },
        '_metadata': {
            'schema_version': '2.0',
            'node_id': 'humanInTheLoop-6',
            'node_type': 'humanInTheLoop',
            'task_type': 'request_for_update',
            'executed_at': '2026-01-22T09:36:41.238615+00:00',
            'upstream_node_id': 'questionClassifier-3',
            'context_chain': ['start-2', 'app-1', 'humanInTheLoop-6'],
            'decision_timestamp': '2026-01-22T09:38:15.673603+00:00',
            'reviewer_id': '72b2d35d-ccf3-4eda-843f-93a1ba0ae7f5',
            'reviewer_email': 'huangpan2507@126.com',
            'reviewer_job_title_id': 1,
            'reviewer_job_title_name': 'Group Director of Operations',
            'steering_task_id': 30,
            'steering_task_ids': [30],
            'update_job_title_ids': [1],
            'file_ingestion_record_ids': [318],
            'timeout': 43200,
            'expires_at': '2026-01-22T21:36:41.238615+00:00'
        }
    }
}
```

**日志验证**：
- `[WF][humanInTheLoop][REQUESTFORUPDATE] node_id=humanInTheLoop-6 Starting execution, task_type=request_for_update`
- `[WF][humanInTheLoop] node_id=humanInTheLoop-6 Got upstream context from questionClassifier-3 (type=questionClassifier): ['branch', 'class_id', 'class_name', 'structured_output', 'upstream_context', 'merged_context', '_metadata']`
- `[WF][humanInTheLoop][UPDATE] node_id=humanInTheLoop-6 Update config loaded: updateJobTitleIds=[1], title='supplier information update', description_length=2, timeout=43200`
- `[WF][humanInTheLoop][UPDATE] Created Update SteeringTask id=30 for user=huangpan2507@126.com, task_type=request_for_update`
- `[WF][humanInTheLoop][REQUESTFORUPDATE] node_id=humanInTheLoop-6 Human decision processed: updated`

---

#### 9.2.5 End Node (end-7)

**输入**：
```python
{
    # End Node 不读取输入数据
}
```

**输出**（基于数据库验证）：
```python
{
    'status': 'completed',
    'output': {
        'workflow_completed': True
    }
}
```

---

### 9.3 数据流转分析（基于实际执行）

#### 9.3.1 数据流转路径

```
1. Start Node (start-2)
   输入: state['input_data'] = {}
   输出: {}（空字典）
   
2. App Node (app-1)
   输入: 从 start-2 获取 {}（空字典）
   配置: 从 app_node_configs 获取 {dagRunId, inputMode, requirement, appNodeId}
   输出: {
       conclusion_detail: "..." (6974字符),
       dag_run_id: "manual__2026-01-22T09:29:20.874721+00:00",
       structured_output: {},
       upstream_context: {},
       merged_context: {},
       _metadata: {...}
   }
   
3. Question Classifier Node (questionClassifier-3)
   输入: 从 app-1 的 conclusion_detail 提取（6974字符）
   配置: 从节点定义获取 {classes, instruction, llmConfig}
   LLM分类: class_id='class_2', reason='...'
   输出: {
       branch: 'class_2',
       class_id: 'class_2',
       class_name: 'supplier information update',
       structured_output: {...},
       upstream_context: {},
       merged_context: {
           selected_class: 'class_2',
           selected_class_name: 'supplier information update',
           classification_reason: '...',
           available_classes: ['class_1', 'class_2']
       },
       _metadata: {...}
   }
   
4. Human In The Loop Node (humanInTheLoop-6, request_for_update)
   输入: 从 questionClassifier-3 的 merged_context 提取
   配置: 从节点定义获取 {taskType, updateConfig, timeout}
   用户提交: {decision: 'updated', update_info: 'yy', file_ingestion_record_ids: [318]}
   输出: {
       human_decision: 'updated',
       decision_reason: 'yy',
       structured_output: {
           task_type: 'request_for_update',
           is_updated: True,
           update_info: 'yy',
           file_ingestion_record_ids: [318],
           ...
       },
       upstream_context: {...},  # 包含 questionClassifier-3 的完整输出
       merged_context: {...},  # 包含上游上下文 + 当前节点输出
       _metadata: {...}
   }
   
5. End Node (end-7)
   输入: 不读取输入
   输出: {workflow_completed: True}
```

#### 9.3.2 关键发现

1. **Start Node 输出为空字典**：
   - 代码逻辑：`output = state['input_data']`
   - 数据库验证：`input_data = {}`
   - 实际输出：`{}`（空字典）

2. **App Node 使用 `app_node_configs` 而非输入数据**：
   - 输入：从 start-2 获取 `{}`（空字典）
   - 配置：从 `app_node_configs` 获取 `dagRunId`、`inputMode` 等
   - 输出：包含 `conclusion_detail`（6974字符）

3. **Question Classifier Node 从 `conclusion_detail` 提取输入**：
   - 输入提取：优先从 `output.conclusion_detail` 提取
   - 实际验证：提取了6974字符的文本
   - 输出：包含分类结果和 `merged_context`

4. **Human In The Loop Node 从 `merged_context` 提取输入**：
   - 输入提取：从上游节点的 `merged_context` 提取
   - 实际验证：从 questionClassifier-3 的 `merged_context` 提取了完整上下文
   - 输出：包含用户提交的数据和完整的上下文链

5. **上下文传递机制**：
   - App Node：`merged_context = upstream_context`（当前为空）
   - Question Classifier Node：`merged_context = upstream_context + structured_output`
   - Human In The Loop Node：`merged_context = upstream_context + current_output`

---

### 9.4 其他 Task Type 的输入结构

#### 9.4.1 Approval类型（taskType = 'approval'）

**输入结构**（基于代码验证）：
```python
{
    # 从上游节点的 merged_context 提取的上下文数据
    # 提取逻辑：使用 get_context_from_upstream_output() 统一提取逻辑
    # 对于 app/condition/humanInTheLoop 类型：从 output.merged_context 提取
    # 对于 questionClassifier 类型：使用整个 output
    # 对于 start 类型：使用整个 output
}
```

**配置数据来源**：
```python
{
    'taskType': 'approval',
    'approvalConfig': {
        'approverJobTitleIds': [1, 2, ...],  # 审批人职位ID列表（必需）
        'title': 'Approval Request',  # 审批任务标题
        'description': '...',  # 审批任务描述
        'approvalLogic': 'any' | 'all' | 'majority',  # 审批逻辑
        'tieBreaker': 'approve' | 'reject'  # 平局处理
    },
    'timeout': 43200,  # 超时时间（秒）
    'timeoutAction': 'fail'  # 超时动作
}
```

**用户提交数据**（等待用户输入后）：
```python
{
    'decision': 'approved' | 'rejected',
    'comments': '...',  # 审批意见
    'file_ingestion_record_ids': [...]  # 附件文件ID列表（可选）
}
```

---

#### 9.4.2 Inform类型（taskType = 'inform'）

**输入结构**（基于代码验证）：
```python
{
    # 从上游节点的 merged_context 提取的上下文数据
    # 提取逻辑：使用 get_context_from_upstream_output() 统一提取逻辑
    # 与 approval 类型相同
}
```

**配置数据来源**：
```python
{
    'taskType': 'inform',
    'informConfig': {
        'informJobTitleIds': [1, 2, ...],  # 被通知人职位ID列表（必需）
        'title': 'Inform Notification',  # 通知任务标题
        'description': '...'  # 通知任务描述
    }
    # 注意：inform 类型不需要 timeout（立即完成）
}
```

**用户提交数据**：
- Inform 类型不等待用户提交，节点执行后立即完成
- 不包含用户提交数据

---

#### 9.4.3 Assign类型（taskType = 'assign'）

**输入结构**（基于代码验证）：
```python
{
    # 从上游节点的 merged_context 提取的上下文数据
    # 提取逻辑：使用 get_context_from_upstream_output() 统一提取逻辑
    # 与 approval 类型相同
}
```

**配置数据来源**：
```python
{
    'taskType': 'assign',
    'assignConfig': {
        'assigneeEmail': 'user@example.com',  # 被分配人邮箱地址（必需）
        'title': 'Assign Task',  # 分配任务标题
        'description': '...'  # 分配任务描述
    },
    'timeout': 43200,  # 超时时间（秒）
    'timeoutAction': 'fail'  # 超时动作
}
```

**用户提交数据**（等待用户输入后）：
```python
{
    'decision': 'assigned' | 'completed',
    'assign_info': '...',  # 分配信息或处理结果内容
    'file_ingestion_record_ids': [...]  # 附件文件ID列表（可选）
}
```

---

### 9.5 数据流转关键点总结

1. **输入数据来源**：
   - Start Node：`state['input_data']`（通常为空字典）
   - App Node：从上游节点获取，或从 `state['input_data']` 获取
   - Question Classifier Node：从上游节点的 `conclusion_detail` 或 `merged_context` 提取
   - Human In The Loop Node：从上游节点的 `merged_context` 提取
   - End Node：不读取输入

2. **配置数据来源**：
   - App Node：从 `app_node_configs[node_id]` 获取
   - Question Classifier Node：从节点定义的 `classifierConfig` 获取
   - Human In The Loop Node：从节点定义的 `taskType` 和相关配置获取

3. **上下文传递机制**：
   - 使用 `get_context_from_upstream_output()` 统一提取逻辑
   - 对于 `app`、`condition`、`humanInTheLoop` 类型：从 `output.merged_context` 提取
   - 对于 `questionClassifier` 类型：使用整个 `output`
   - 对于 `start` 类型：使用整个 `output`

4. **实际验证结果**：
   - 所有结论均基于日志和数据库验证
   - Start Node 输出为空字典（已验证）
   - App Node 使用 `app_node_configs` 而非输入数据（已验证）
   - Question Classifier Node 从 `conclusion_detail` 提取输入（已验证）
   - Human In The Loop Node 从 `merged_context` 提取输入（已验证）
   - 说明 Start Node 的输出来源（`input_data`，通常是空字典）
