# merged_context 字段澄清

## 一、merged_context 是什么？

**`merged_context` 不是所有节点的标准字段**，它只在以下场景中使用：

### 1. App Node (v2.0 结构，向后兼容)

在 App Node 的 v2.0 结构中，`merged_context` 是一个临时字段，用于合并上游节点的上下文和当前节点的输出：

```python
# v2.0 结构（旧结构，向后兼容）
{
    'structured_output': {
        # 9个固定字段
        'node_type': 'app',
        'app_node_id': 105,
        'conclusion_detail': '...',
        # ...
    },
    'upstream_context': {
        # 上游节点的上下文
    },
    'merged_context': {
        # 浅合并：upstream_context + structured_output
        # structured_output 的字段优先（冲突时覆盖 upstream_context）
    }
}
```

**计算方式**:
```python
merged_context = {}
if upstream_context:
    merged_context.update(upstream_context)
merged_context.update(structured_output)  # structured_output 优先
```

---

### 2. 数据提取逻辑（向后兼容）

`get_context_from_upstream_output()` 函数会检查 `merged_context`（用于向后兼容）：

```python
# 新结构 (v3.0) 优先
if 'data' in upstream_output:
    return upstream_output.get('data', {})

# 旧结构 (v2.0) 向后兼容
if upstream_node_type in ('app', 'condition', 'humanInTheLoop'):
    merged_context = upstream_output.get('merged_context')
    if merged_context:
        return merged_context
```

---

### 3. Question Classifier Node（向后兼容）

在提取输入文本时，会尝试从 `merged_context` 提取（向后兼容）：

```python
# 优先：从新结构提取
conclusion_detail = upstream_output.get('data', {}).get('conclusion_detail', '')

# 备选：从旧结构提取（向后兼容）
merged_context = upstream_output.get('merged_context', {})
if merged_context:
    # 转换为字符串
    return convert_to_string(merged_context)
```

---

## 二、新结构 (v3.0) 不再使用 merged_context

### 新结构设计原则

1. **数据分离**: 每个节点只存储自己产生的数据
2. **不包含上游数据副本**: 不存储 `upstream_context` 和 `merged_context`
3. **统一接口**: 使用 `output.data` 和 `output.structured_output`

### 新结构示例

```python
# v3.0 结构（新结构）
{
    'status': 'completed',
    'input': {
        'data': {},
        'source_type': 'upstream',
        'source_node_id': 'app-1'
    },
    'operation': {
        'type': 'app_execution',
        'config': {...}
    },
    'output': {
        'data': {
            # 仅包含当前节点产生的数据
            'conclusion_detail': '...',
            'dag_run_id': '...'
        },
        'structured_output': {
            # 结构化业务数据（用于 view output 展示）
            'node_type': 'app',
            'app_node_id': 105,
            # ...
        }
    },
    'timeline': {
        'executed_at': '...',
        'execution_order': 2
    }
}
```

**注意**: 新结构不包含 `merged_context` 和 `upstream_context` 字段。

---

## 三、向后兼容机制

### 数据提取逻辑

`get_context_from_upstream_output()` 函数支持新旧两种结构：

```python
def get_context_from_upstream_output(
    upstream_node_type: Optional[str], 
    upstream_output: Dict[str, Any],
    node_id: str = None
) -> Dict[str, Any]:
    """
    提取上游节点的上下文数据
    
    优先级：
    1. 新结构 (v3.0): output.data
    2. 旧结构 (v2.0): output.merged_context（向后兼容）
    3. 其他旧结构: 根据节点类型提取
    """
    # 1. 检查新结构
    if 'data' in upstream_output:
        return upstream_output.get('data', {})
    
    # 2. 检查旧结构（向后兼容）
    if upstream_node_type in ('app', 'condition', 'humanInTheLoop'):
        merged_context = upstream_output.get('merged_context')
        if merged_context:
            return merged_context
    
    # 3. 其他情况
    return {}
```

---

## 四、总结

### merged_context 的使用场景

1. ✅ **App Node (v2.0 结构)**: 用于合并上游上下文和当前输出（向后兼容）
2. ✅ **数据提取逻辑**: 用于向后兼容旧结构的数据提取
3. ✅ **Question Classifier Node**: 用于向后兼容旧结构的输入文本提取

### merged_context 不是标准字段

1. ❌ **新结构 (v3.0)**: 不再使用 `merged_context`
2. ❌ **所有节点**: `merged_context` 不是所有节点的标准字段
3. ❌ **数据存储**: 新结构不存储 `merged_context`

### 推荐做法

1. ✅ **使用新结构**: 优先使用 `output.data` 和 `output.structured_output`
2. ✅ **向后兼容**: 代码支持从旧结构提取数据（自动兼容）
3. ✅ **数据分离**: 每个节点只存储自己产生的数据

---

## 五、相关文档

- `workflow-nodes-complete-output-structure.md`: 节点完整输出结构文档
- `workflow-chain-call-design.md`: 链式调用设计文档
