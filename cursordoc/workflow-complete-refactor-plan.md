# Workflow 彻底重构方案：删除所有向后兼容代码

## 一、前提条件

✅ **项目未上线，未交付给客户**
- 没有生产环境的旧数据需要保护
- 可以接受破坏性改动
- 可以彻底重构，删除所有向后兼容代码
- 不需要考虑历史数据的兼容性

---

## 二、重构目标

### 2.1 统一数据结构

**目标**: 所有节点统一使用 v3.0 结构

```python
# 统一的新结构 (v3.0)
{
    'status': 'completed' | 'pending' | 'failed',
    'input': {
        'data': dict,  # 实际输入数据
        'source_node_id': str | None,
        'source_type': str,  # 'upstream' | 'workflow_input' | 'config'
    },
    'operation': {
        'type': str,
        'description': str,
        'config': dict,
        'duration_ms': int | None,
    },
    'output': {
        'data': dict,  # 节点产生的实际数据
        'structured_output': dict,  # 结构化业务数据
    },
    'timeline': {
        'executed_at': str,
        'execution_order': int,
    }
}
```

### 2.2 删除的字段

- ❌ `merged_context` - 完全删除
- ❌ `upstream_context` - 完全删除
- ❌ `_metadata` - 完全删除（或保留但简化）

---

## 三、需要修改的代码位置

### 3.1 后端代码

#### 1. `backend/app/services/workflow/node_utils.py`

**需要修改**:
- `get_context_from_upstream_output()` - 删除旧结构处理逻辑
- `get_metadata_from_upstream_output()` - 删除或简化

**当前代码**:
```python
def get_context_from_upstream_output(...):
    # 检查新结构
    if 'data' in upstream_output:
        return upstream_output.get('data', {})
    
    # ❌ 删除：旧结构处理（向后兼容）
    if upstream_node_type in ('app', 'condition', 'humanInTheLoop'):
        merged_context = upstream_output.get('merged_context')
        if merged_context:
            return merged_context
    ...
```

**修改后**:
```python
def get_context_from_upstream_output(...):
    # 只支持新结构 (v3.0)
    if not upstream_output or not isinstance(upstream_output, dict):
        return {}
    
    # 从 output.data 提取
    output_data = upstream_output.get('data', {})
    if isinstance(output_data, dict) and output_data:
        logger.info(f"{log_prefix} Extracted context from {upstream_node_type} Node output.data (v3.0)")
        return output_data
    
    logger.warning(f"{log_prefix} {upstream_node_type} Node has no valid output.data, returning empty context")
    return {}
```

---

#### 2. `backend/app/services/workflow/workflow_execution_service.py`

**需要修改**:
- `_extract_structured_output()` - 删除 `merged_context` 和 `upstream_context` 生成代码

**当前代码**:
```python
# ❌ 删除：merged_context 生成
merged_context = {}
if upstream_context:
    merged_context.update(upstream_context)
merged_context.update(structured_output)

final_result = {
    'structured_output': structured_output,
    'upstream_context': upstream_context if upstream_context else {},  # ❌ 删除
    'merged_context': merged_context,  # ❌ 删除
    '_metadata': {
        'schema_version': '2.0',  # ❌ 修改为 '3.0'
        ...
    }
}
```

**修改后**:
```python
# 只返回 structured_output（用于 App Node 的特殊场景）
# 注意：这个方法可能不再需要，因为 App Node 已经使用新结构
final_result = {
    'structured_output': structured_output,
    # ❌ 删除：upstream_context, merged_context, _metadata
}
```

**注意**: 检查 `_extract_structured_output()` 的返回值是否被使用。如果 App Node 已经使用新结构，这个方法可能不再需要。

---

#### 3. `backend/app/services/workflow/nodes/question_classifier_node.py`

**需要修改**:
- `_extract_input_text()` - 删除旧结构处理逻辑

**当前代码**:
```python
def _extract_input_text(self, upstream_output: Dict[str, Any]) -> str:
    # 优先：从新结构提取
    if isinstance(upstream_output, dict) and 'data' in upstream_output:
        conclusion_detail = upstream_output.get('data', {}).get('conclusion_detail', '')
    else:
        conclusion_detail = upstream_output.get('conclusion_detail', '')
    
    if conclusion_detail:
        return conclusion_detail
    
    # ❌ 删除：旧结构处理（向后兼容）
    merged_context = upstream_output.get('merged_context', {})
    if merged_context:
        # Convert to readable string
        ...
```

**修改后**:
```python
def _extract_input_text(self, upstream_output: Dict[str, Any]) -> str:
    # 只支持新结构 (v3.0)
    if isinstance(upstream_output, dict) and 'data' in upstream_output:
        conclusion_detail = upstream_output.get('data', {}).get('conclusion_detail', '')
    else:
        conclusion_detail = upstream_output.get('conclusion_detail', '')
    
    if conclusion_detail and isinstance(conclusion_detail, str) and conclusion_detail.strip():
        return conclusion_detail
    
    # Fallback: convert entire output to JSON string
    try:
        return json.dumps(upstream_output, ensure_ascii=False, indent=2)
    except Exception:
        return str(upstream_output)
```

---

#### 4. `backend/app/services/workflow/nodes/human_in_the_loop_node.py`

**需要检查**:
- `_get_upstream_context()` - 确认是否使用 `get_context_from_upstream_output()`
- 如果使用，不需要修改（因为 `get_context_from_upstream_output()` 已经修改）

---

### 3.2 前端代码

#### 1. `frontend/src/components/Workbench/FlowWorkspace.tsx`

**需要修改**:
- 删除旧结构的处理逻辑

**当前代码**:
```typescript
// ❌ 删除：旧结构处理
if (nodeResult.output) {
    // New structure: prioritize structured_output, then data, then fallback to output
    outputs = nodeResult.output.structured_output || nodeResult.output.data || nodeResult.output || {}
} else {
    // Old structure: use output directly
    outputs = nodeResult.output || {}
}
```

**修改后**:
```typescript
// 只支持新结构 (v3.0)
if (nodeResult.output) {
    outputs = nodeResult.output.structured_output || nodeResult.output.data || {}
} else {
    outputs = {}
}
```

---

#### 2. `frontend/src/components/Workbench/NodeOutputPanel.tsx`

**需要修改**:
- 删除旧结构的处理逻辑

**当前代码**:
```typescript
// ❌ 删除：旧结构支持
const newStructure = nodeData && (
    nodeData.input || nodeData.operation || nodeData.output || nodeData.timeline
)

const finalOutputs = newStructure 
    ? (nodeData.output?.structured_output || nodeData.output?.data || outputs || {})
    : (outputs || {})
```

**修改后**:
```typescript
// 只支持新结构 (v3.0)
const finalOutputs = nodeData?.output?.structured_output || nodeData?.output?.data || outputs || {}
const finalInputs = nodeData?.input?.data || inputs || {}
```

---

## 四、实施步骤

### 步骤1: 检查代码依赖

1. **检查 `_extract_structured_output()` 的返回值是否被使用**:
   ```bash
   grep -r "_extract_structured_output" backend/app/services/workflow/
   ```

2. **检查所有使用 `merged_context` 的地方**:
   ```bash
   grep -r "merged_context" backend/app/services/workflow/
   grep -r "merged_context" frontend/src/
   ```

3. **检查所有使用 `upstream_context` 的地方**:
   ```bash
   grep -r "upstream_context" backend/app/services/workflow/
   grep -r "upstream_context" frontend/src/
   ```

---

### 步骤2: 修改后端代码

1. **修改 `node_utils.py`**:
   - 删除 `get_context_from_upstream_output()` 中的旧结构处理
   - 简化 `get_metadata_from_upstream_output()`（如果需要）

2. **修改 `workflow_execution_service.py`**:
   - 删除 `_extract_structured_output()` 中的 `merged_context` 和 `upstream_context` 生成
   - 检查返回值是否被使用，如果不需要则删除整个方法

3. **修改 `question_classifier_node.py`**:
   - 删除 `_extract_input_text()` 中的旧结构处理

4. **检查其他节点**:
   - 确认所有节点都使用新结构

---

### 步骤3: 修改前端代码

1. **修改 `FlowWorkspace.tsx`**:
   - 删除旧结构的处理逻辑

2. **修改 `NodeOutputPanel.tsx`**:
   - 删除旧结构的处理逻辑

3. **检查其他组件**:
   - 确认所有组件都使用新结构

---

### 步骤4: 测试验证

1. **单元测试**:
   - 测试所有节点的输出结构
   - 测试数据提取逻辑

2. **集成测试**:
   - 测试完整的 workflow 执行流程
   - 测试节点之间的数据传递

3. **前端测试**:
   - 测试节点输出显示
   - 测试数据格式解析

---

### 步骤5: 更新文档

1. **更新 `workflow-nodes-complete-output-structure.md`**:
   - 删除所有关于 `merged_context` 和 `upstream_context` 的说明
   - 统一使用 v3.0 结构

2. **更新其他相关文档**:
   - 删除向后兼容的说明
   - 统一使用新结构

---

## 五、预期效果

### 5.1 代码简化

- ✅ 删除所有向后兼容代码
- ✅ 统一使用新结构（v3.0）
- ✅ 代码更清晰，易于维护

### 5.2 性能提升

- ✅ 减少不必要的检查逻辑
- ✅ 减少数据转换开销

### 5.3 维护性提升

- ✅ 没有历史包袱
- ✅ 代码结构清晰
- ✅ 易于理解和维护

---

## 六、风险评估

### 6.1 低风险

- ✅ 项目未上线，没有生产数据
- ✅ 可以接受破坏性改动
- ✅ 可以彻底重构

### 6.2 需要注意

- ⚠️ 确保所有节点都使用新结构
- ⚠️ 确保前端正确解析新结构
- ⚠️ 充分测试所有功能

---

## 七、总结

### 7.1 重构原则

1. **彻底删除向后兼容代码**
2. **统一使用新结构（v3.0）**
3. **简化代码逻辑**
4. **提高代码质量**

### 7.2 关键步骤

1. ✅ 检查代码依赖
2. ✅ 修改后端代码
3. ✅ 修改前端代码
4. ✅ 测试验证
5. ✅ 更新文档

### 7.3 预期收益

- ✅ 代码更清晰
- ✅ 易于维护
- ✅ 没有历史包袱
- ✅ 统一的数据结构

---

## 八、下一步行动

1. **立即开始**: 检查代码依赖，确认需要修改的位置
2. **逐步实施**: 按照步骤修改代码
3. **充分测试**: 确保所有功能正常
4. **更新文档**: 同步更新相关文档

**建议**: 可以立即开始彻底重构，删除所有向后兼容代码。
