# Workflow XCom 和 App Node Output 问题分析报告

## 问题概述

### 问题1: JSON parsing failed
**现象**：日志显示 `JSON parsing failed: Expecting property name enclosed in double quotes: line 1 column 2 (char 1), attempting eval for Python dict format...`

**影响**：虽然后端通过 `eval` 成功解析，但前端 App Node 的 output 面板显示为空（`outputs: {}`）

### 问题2: Failed to save pending state
**现象**：日志显示 `[humanInTheLoop] node_id=humanInTheLoop-6 ❌ Failed to save pending state to database: name 'pending_output' is not defined`

**影响**：`humanInTheLoop-6` 节点无法保存 pending 状态到数据库

---

## 根本原因分析

### 问题1: JSON parsing failed

#### 数据流路径
1. **Airflow DAG** (`agent_file_ingest_unified.py:stage_1_base_agents`)
   - 函数直接返回 Python 字典：`return base_agents_result`
   - Airflow 自动将返回值序列化到 XCom 的 `return_value` key

2. **Airflow XCom 序列化**
   - Airflow 在序列化 Python 字典时，可能使用 `str(dict)` 而不是 `json.dumps()`
   - 这导致 XCom 中存储的是 Python 字典的字符串表示（如 `"{'success': True, 'markdown_conclusion': '...'}"`）
   - 这种格式不完全符合 JSON 规范（单引号、`True`/`False` 而非 `true`/`false`）

3. **后端解析** (`backend/app/api/v1/report.py:_process_standard_dag_status`)
   - 先尝试 `json.loads()` 解析 → 失败（因为格式不是标准 JSON）
   - 然后使用 `eval()` 解析 → 成功（能处理 Python 字典格式）
   - 解析后的数据正确传递到 App Node

4. **App Node 处理** (`backend/app/services/workflow/workflow_execution_service.py:app_node`)
   - 从 `result.get('result_data', {}).get('markdown_conclusion', '')` 提取数据
   - 构建 `node_data` 并写入 `state['node_results']`
   - 结构：`node_data.output.data.conclusion_detail` 和 `node_data.output.structured_output`

5. **前端提取** (`frontend/src/components/Workbench/FlowWorkspace.tsx`)
   - 从 `nodeResult.output` 获取数据
   - 但 App Node 的新结构是 `nodeResult.output.structured_output` 或 `nodeResult.output.data`
   - `NodeOutputPanel` 会检查新结构，但可能没有正确提取

#### 架构缺陷
- **数据契约不明确**：XCom 数据格式不统一（Python dict vs JSON）
- **前端数据提取逻辑不完整**：没有正确处理新结构的数据

#### 发生环节
- **后端**：`backend/app/api/v1/report.py` → `_process_standard_dag_status` → XCom 解析
- **阶段**：DAG 完成后，从 Airflow XCom 获取 `stage_1_base_agents` 的 `return_value`
- **影响**：虽然后端解析成功，但前端可能未正确显示数据

### 问题2: pending_output 未定义

#### 代码位置
- **文件**：`backend/app/services/workflow/nodes/human_in_the_loop_node.py`
- **函数**：`_execute_request_for_update`
- **行号**：1443 和 1466

#### 问题代码
```python
# 第 1443 行
node_results[self.node_id] = {
    'status': 'pending',
    'output': pending_output  # ❌ pending_output 未定义
}

# 第 1466 行
await websocket_manager.broadcast_to_room(room_id, {
    "type": "node_finished",
    "task_id": task_id,
    "node_id": self.node_id,
    "status": "pending",
    "outputs": pending_output  # ❌ pending_output 未定义
})
```

#### 正确代码
应该使用 `pending_node_data.get('output', {})` 或 `pending_node_data['output']`

#### 发生环节
- **后端**：`backend/app/services/workflow/nodes/human_in_the_loop_node.py` → `_execute_request_for_update`
- **阶段**：保存 pending 状态到数据库时
- **影响**：无法保存 pending 状态，导致：
  - 数据库缺少节点状态
  - 前端无法获取 pending 状态
  - 可能影响后续流程恢复

---

## 修复方案

### 修复1: pending_output 未定义问题 ✅

**已修复**：将 `pending_output` 替换为 `pending_node_data.get('output', {})`

**修改位置**：
- `backend/app/services/workflow/nodes/human_in_the_loop_node.py:1443`
- `backend/app/services/workflow/nodes/human_in_the_loop_node.py:1466`

**修改内容**：
```python
# 修改前
node_results[self.node_id] = {
    'status': 'pending',
    'output': pending_output  # ❌
}

# 修改后
node_results[self.node_id] = pending_node_data  # ✅ 保存完整结构
```

### 修复2: XCom 数据格式问题（待实施）

#### 方案A: 规范化 XCom 写入（推荐）
**位置**：`airflow/dags/agent_file_ingest_unified.py:stage_1_base_agents`

**修改**：在返回前将 Python 字典转换为 JSON 字符串
```python
import json

# 修改前
return base_agents_result

# 修改后
# 将 Python 字典转换为 JSON 字符串，确保 XCom 存储标准 JSON
return json.dumps(base_agents_result, ensure_ascii=False)
```

**注意**：如果使用 `json.dumps()`，后端解析逻辑也需要相应调整（直接使用 `json.loads()` 而不是 `eval`）

#### 方案B: 改进后端解析（临时方案）
**位置**：`backend/app/api/v1/report.py:_process_standard_dag_status`

**修改**：优先使用 `ast.literal_eval`（更安全）而不是 `eval`
```python
import ast

# 修改前
actual_data = eval(cleaned_value)

# 修改后
try:
    actual_data = ast.literal_eval(value_field)
except (ValueError, SyntaxError):
    # Fallback to eval only if ast.literal_eval fails
    actual_data = eval(cleaned_value)
```

### 修复3: 前端数据提取逻辑（待实施）

**位置**：`frontend/src/components/Workbench/FlowWorkspace.tsx`

**问题**：前端从 `nodeResult.output` 获取数据，但 App Node 的新结构是：
- `nodeResult.output.data.conclusion_detail`
- `nodeResult.output.structured_output`

**修改**：确保前端正确提取新结构的数据
```typescript
// 在 FlowWorkspace.tsx 中，提取 outputs 时：
let outputs = nodeData.outputs || {}
if ((!outputs || Object.keys(outputs).length === 0) && nodeResults[selectedOutputNodeId]) {
  const nodeResult = nodeResults[selectedOutputNodeId]
  
  // 检查新结构
  if (nodeResult.output) {
    // 新结构：优先使用 structured_output，其次使用 data
    outputs = nodeResult.output.structured_output || nodeResult.output.data || nodeResult.output
  } else {
    // 旧结构：直接使用 output
    outputs = nodeResult.output || {}
  }
}
```

**注意**：`NodeOutputPanel` 已经支持新结构（通过 `nodeData` prop），但需要确保 `outputs` prop 也正确传递

---

## 验证步骤

### 验证修复1: pending_output
1. 运行 workflow，触发 `request_for_update` 类型的 HumanInTheLoop 节点
2. 检查日志，确认没有 `pending_output is not defined` 错误
3. 检查数据库，确认 `node_results` 中保存了完整的 pending 状态

### 验证修复2: XCom 数据格式
1. 运行 workflow，触发 App Node 执行
2. 检查日志，确认没有 `JSON parsing failed` 警告（如果使用方案A）
3. 检查前端，确认 App Node output 面板正确显示数据

### 验证修复3: 前端数据提取
1. 运行 workflow，触发 App Node 执行
2. 打开 App Node 的 output 面板
3. 确认 `outputs` 不为空，且包含 `conclusion_detail` 或 `structured_output`

---

## 建议的修复优先级

1. **立即修复**：pending_output 未定义问题 ✅（已完成）
2. **短期修复**：前端数据提取逻辑（确保前端能正确显示数据）
3. **长期优化**：规范化 XCom 写入格式（统一使用 JSON 格式）

---

## 相关文件

- `backend/app/services/workflow/nodes/human_in_the_loop_node.py` - HumanInTheLoop 节点实现
- `backend/app/api/v1/report.py` - XCom 数据解析
- `backend/app/services/workflow/workflow_execution_service.py` - App Node 执行逻辑
- `airflow/dags/agent_file_ingest_unified.py` - Airflow DAG 定义
- `airflow/dags/agent_flows/app_hierarchy_processor.py` - App 层级处理逻辑
- `frontend/src/components/Workbench/FlowWorkspace.tsx` - 前端 workflow 界面
- `frontend/src/components/Workbench/NodeOutputPanel.tsx` - 前端节点输出面板
- `frontend/src/utils/workflowResultFormatter.ts` - 前端数据格式化工具
