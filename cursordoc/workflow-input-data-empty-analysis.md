# Workflow input.data 为空问题分析报告

## 问题描述

用户反馈：所有节点的 `input.data` 都是空对象 `{}`，这与设计文档是否一致？

**实际观察到的 Raw Data**:
1. **App Node**: `input.data: {}`, `source_node_id: "start-2"`
2. **Question Classifier Node**: `input.data: {}`, `source_node_id: "app-1"`
3. **Human In The Loop Node**: `input.data: {}`, `source_node_id: "questionClassifier-3"`

## 设计文档预期

根据 `workflow-nodes-complete-output-structure.md`:

### App Node (line 155)
```python
'input': {
    'data': dict,  # 实际输入数据(通常为空,App Node主要使用config)
    'source_node_id': str | None,
    'source_type': str,  # 'config' (App Node主要使用app_node_configs)
}
```
**说明**: App Node 的 `input.data` 通常为空是**符合设计**的，因为 App Node 主要使用 `app_node_configs`，而不是上游数据。

### Question Classifier Node (line 173-178)
```python
# Extract input data from upstream (only what we need - conclusion_detail)
input_data = {}
if upstream_output:
    conclusion_detail = upstream_output.get('conclusion_detail', '')
    if conclusion_detail:
        input_data = {'conclusion_detail': conclusion_detail}
```
**预期**: Question Classifier Node 应该从 App Node 的 `output.data.conclusion_detail` 提取数据，存储在 `input.data` 中。

### Human In The Loop Node (line 223-224)
```python
# Extract input data from upstream (only what we need)
input_data = {}
```
**预期**: Human In The Loop Node 的 `input.data` 应该包含从上游节点提取的上下文数据。

---

## 问题根源分析

### 问题1: Question Classifier Node 无法获取 conclusion_detail

**代码位置**: `backend/app/services/workflow/nodes/question_classifier_node.py:173-178`

**当前代码**:
```python
# Extract input data from upstream (only what we need - conclusion_detail)
input_data = {}
if upstream_output:
    conclusion_detail = upstream_output.get('conclusion_detail', '')
    if conclusion_detail:
        input_data = {'conclusion_detail': conclusion_detail}
```

**问题**:
- `upstream_output` 是从 `_get_upstream_info()` 获取的 `output` 字段
- App Node 的新结构中，`conclusion_detail` 在 `output.data.conclusion_detail`
- 但代码从 `upstream_output.get('conclusion_detail', '')` 获取（旧结构路径）
- 导致获取失败，`input_data` 保持为空

**正确的获取路径**:
```python
# 应该从新结构获取
conclusion_detail = upstream_output.get('data', {}).get('conclusion_detail', '')
```

**验证**:
- 从日志看：`[WF][classifier] node_id=questionClassifier-3 Got upstream output from app-1`
- 从 Raw Data 看：App Node 的 `output.data.conclusion_detail` 存在（长度 8323）
- 但 Question Classifier Node 的 `input.data` 为空

---

### 问题2: Human In The Loop Node 没有提取上游数据到 input.data

**代码位置**: `backend/app/services/workflow/nodes/human_in_the_loop_node.py:223-224`

**当前代码**:
```python
# Extract input data from upstream (only what we need)
input_data = {}
source_type = 'upstream' if upstream_node_id else 'workflow_input'
```

**问题**:
- 代码直接设置 `input_data = {}`，没有从上游提取任何数据
- 虽然通过 `_get_upstream_context()` 获取了上下文（用于创建 SteeringTask），但不存储在 `input.data` 中
- 导致 `input.data` 为空

**设计文档预期** (line 264):
```python
'input': {
    'data': dict,  # 实际输入数据(从上游节点提取的字段)
    'source_node_id': str | None,
    'source_type': 'upstream'
}
```

**应该的行为**:
- 从上游节点的 `output.data` 提取数据（使用 `get_context_from_upstream_output()`）
- 存储在 `input.data` 中，用于追溯和调试

**验证**:
- 从日志看：`[WF][humanInTheLoop] node_id=humanInTheLoop-4 Got upstream context from questionClassifier-3 (type=questionClassifier): ['branch', 'class_id', 'class_name']`
- 说明上游上下文是存在的，但没有存储在 `input.data` 中

---

### 问题3: App Node 从 Start Node 获取空数据（符合设计）

**代码位置**: `backend/app/services/workflow/workflow_execution_service.py:609-622`

**当前代码**:
```python
if previous_nodes:
    source_node_id = previous_nodes[0]
    source_result = state.get('node_results', {}).get(source_node_id, {})
    source_output = source_result.get('output', {})
    if isinstance(source_output, dict):
        source_data = source_output.get('data', {})
        input_data = source_data.copy() if source_data else {}
```

**分析**:
- Start Node 的 `output` 直接是 `input_data`（不是新结构 `output.data`）
- 从日志看：`[WF][start] node_id=start-2 keys=[] len=0`，说明 Start Node 的 `input_data` 是空的
- 因此 App Node 的 `input.data` 为空是**符合设计**的

---

## 修复方案

### 修复1: Question Classifier Node - 从新结构获取 conclusion_detail

**位置**: `backend/app/services/workflow/nodes/question_classifier_node.py:173-178`

**修改**:
```python
# Extract input data from upstream (only what we need - conclusion_detail)
input_data = {}
if upstream_output:
    # Support both new structure (output.data) and old structure (backward compatibility)
    if isinstance(upstream_output, dict) and 'data' in upstream_output:
        # New structure: extract from output.data
        conclusion_detail = upstream_output.get('data', {}).get('conclusion_detail', '')
    else:
        # Old structure: direct access
        conclusion_detail = upstream_output.get('conclusion_detail', '')
    
    if conclusion_detail:
        input_data = {'conclusion_detail': conclusion_detail}
```

---

### 修复2: Human In The Loop Node - 提取上游数据到 input.data

**位置**: `backend/app/services/workflow/nodes/human_in_the_loop_node.py:223-224`

**修改**:
```python
# Extract input data from upstream (only what we need)
input_data = {}
source_type = 'upstream' if upstream_node_id else 'workflow_input'

# Get upstream context and store in input.data for traceability
if upstream_node_id:
    upstream_context = self._get_upstream_context(state)
    if upstream_context and isinstance(upstream_context, dict):
        # Store upstream context in input.data (for traceability and debugging)
        input_data = upstream_context.copy()
```

**注意**: 
- 使用 `_get_upstream_context()` 获取的上下文（已经存在）
- 存储在 `input.data` 中，用于追溯和调试
- 不影响现有功能（SteeringTask 创建逻辑不变）

---

## 设计文档一致性检查

### 符合设计的部分
1. ✅ **App Node**: `input.data` 为空符合设计（主要使用 `app_node_configs`）
2. ✅ **Start Node**: `output` 为空符合设计（工作流初始输入为空）

### 不符合设计的部分
1. ❌ **Question Classifier Node**: 应该从 App Node 的 `output.data.conclusion_detail` 提取，但代码从旧结构获取
2. ❌ **Human In The Loop Node**: 应该从上游节点的 `output.data` 提取数据，但代码直接设置为空

---

## 影响分析

### 当前影响
1. **调试困难**: 无法从 `input.data` 追溯节点接收的实际输入数据
2. **数据追溯不完整**: 无法完整追踪数据流（从上游到下游）
3. **不符合设计文档**: 与设计文档的预期不一致

### 修复后的影响
1. ✅ **数据追溯完整**: 可以从 `input.data` 看到节点接收的实际输入
2. ✅ **符合设计文档**: 与设计文档的预期一致
3. ✅ **向后兼容**: 支持新旧两种结构

---

## 建议

1. **立即修复**: Question Classifier Node 和 Human In The Loop Node 的 `input.data` 提取逻辑
2. **验证**: 修复后运行 workflow，验证 `input.data` 是否正确填充
3. **文档更新**: 确认修复后与设计文档一致
