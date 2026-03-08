# Workflow 累计上下文收集功能

## 概述

累计上下文收集功能允许 workflow 节点按执行顺序收集所有上游节点的输出数据，提供完整的执行历史上下文。这对于需要查看完整执行链的节点（如 HITL approval 节点）非常有用。

## 使用场景

### 典型场景

```
start-2 → app-1 → questionClassifier-3 → humanInTheLoop-4 (request_for_update) → humanInTheLoop-5 (approval) → end-6
```

**问题**：`humanInTheLoop-5` (approval) 节点需要查看：
- `app-1` 的分析结果 (`conclusion_detail`)
- `questionClassifier-3` 的分类结果 (`branch`, `class_id`, `class_name`)
- `humanInTheLoop-4` 的更新信息 (`human_decision`, `file_ingestion_record_ids`)

**解决方案**：使用累计上下文收集功能，按执行顺序收集所有上游节点的输出数据。

## 实现位置

### 核心服务

**文件**: `backend/app/services/workflow/chain_call_data_service.py`

**方法**: `ChainCallDataService.extract_accumulated_upstream_data()`

```python
@staticmethod
def extract_accumulated_upstream_data(
    state: Dict[str, Any],
    current_node_id: str
) -> Dict[str, Any]:
    """
    Extract accumulated data from ALL upstream nodes in execution order
    
    Returns:
        Dict with structure:
        {
            'accumulated_context': {
                'node_id_1': {
                    'node_type': str,
                    'execution_order': int,
                    'output_data': dict,  # output.data
                    'structured_output': dict,  # output.structured_output
                    'executed_at': str
                },
                ...
            },
            'execution_order': [node_id_1, node_id_2, ...],
            'summary': {...}
        }
    """
```

### 集成点

**文件**: `backend/app/services/workflow/nodes/human_in_the_loop_node.py`

**位置**: `HumanInTheLoopNodeExecutor.execute()` 方法（approval 类型）

```python
# 3.5. Extract accumulated upstream context (for approval nodes that need full context)
accumulated_context = {}
try:
    from app.services.workflow.chain_call_data_service import ChainCallDataService
    accumulated_result = ChainCallDataService.extract_accumulated_upstream_data(
        state, self.node_id
    )
    accumulated_context = accumulated_result.get('accumulated_context', {})
except Exception as e:
    # Fallback to direct upstream only
    pass
```

## 数据结构

### 返回结构

```python
{
    'accumulated_context': {
        'start-2': {
            'node_type': 'start',
            'execution_order': 1,
            'output_data': {
                # output.data from start-2
            },
            'structured_output': {
                # output.structured_output from start-2
            },
            'executed_at': '2026-01-26T02:37:08.358047+00:00'
        },
        'app-1': {
            'node_type': 'app',
            'execution_order': 2,
            'output_data': {
                'conclusion_detail': '# 分析报告\n\n...',
                'dag_run_id': 'manual_2026-01-26T02:37:08.358047+00:00'
            },
            'structured_output': {
                'node_type': 'app',
                'app_node_id': 105,
                'conclusion_length': 8323,
                'dag_run_id': 'manual_2026-01-26T02:37:08.358047+00:00',
                'execution_status': 'success',
                'has_conclusion': True
            },
            'executed_at': '2026-01-26T02:37:10.123456+00:00'
        },
        'questionClassifier-3': {
            'node_type': 'questionClassifier',
            'execution_order': 3,
            'output_data': {
                'branch': 'class_1',
                'class_id': 'class_1',
                'class_name': 'supplier admission approval'
            },
            'structured_output': {
                'node_type': 'questionClassifier',
                'selected_class': 'class_1',
                'selected_class_name': 'supplier admission approval',
                ...
            },
            'executed_at': '2026-01-26T02:37:12.789012+00:00'
        },
        'humanInTheLoop-4': {
            'node_type': 'humanInTheLoop',
            'execution_order': 4,
            'output_data': {
                'human_decision': 'updated',
                'decision_reason': '需要补充供应商资质文件',
                'update_info': '需要补充供应商资质文件',
                'file_ingestion_record_ids': [123, 124, 125]
            },
            'structured_output': {
                'task_type': 'request_for_update',
                'human_decision': 'updated',
                'is_updated': True,
                'update_info': '需要补充供应商资质文件',
                'file_ingestion_record_ids': [123, 124, 125],
                'reviewer_email': 'reviewer@example.com',
                'decision_timestamp': '2026-01-26T02:40:00.000000+00:00'
            },
            'executed_at': '2026-01-26T02:38:00.000000+00:00'
        }
    },
    'execution_order': ['start-2', 'app-1', 'questionClassifier-3', 'humanInTheLoop-4'],
    'summary': {
        'total_nodes': 4,
        'app_nodes': ['app-1'],
        'question_classifier_nodes': ['questionClassifier-3'],
        'human_in_the_loop_nodes': ['humanInTheLoop-4'],
        'start_nodes': ['start-2']
    }
}
```

### 在 HITL Approval 节点中的使用

**输入数据增强** (`input.data`):

```python
{
    'accumulated_context_summary': {
        'total_upstream_nodes': 4,
        'node_ids': ['start-2', 'app-1', 'questionClassifier-3', 'humanInTheLoop-4']
    },
    # 直接上游节点的上下文（向后兼容）
    'human_decision': 'updated',
    'file_ingestion_record_ids': [123, 124, 125]
}
```

**传递给 SteeringTask 的上下文**:

```python
{
    'direct_upstream': {
        # 直接上游节点 humanInTheLoop-4 的输出数据
        'human_decision': 'updated',
        'file_ingestion_record_ids': [123, 124, 125]
    },
    'accumulated_context': {
        # 所有上游节点的完整输出数据（按执行顺序）
        'start-2': {...},
        'app-1': {...},
        'questionClassifier-3': {...},
        'humanInTheLoop-4': {...}
    }
}
```

## 实现细节

### 递归查找算法

使用深度优先搜索 (DFS) 递归查找所有上游节点：

```python
@staticmethod
def _find_all_upstream_nodes(
    current_node_id: str,
    edges: List[Dict[str, Any]],
    visited: set
) -> set:
    """
    Recursively find all upstream nodes using DFS
    """
    if current_node_id in visited:
        # Cycle detected, return empty set
        return set()
    
    visited.add(current_node_id)
    upstream_nodes = set()
    
    # Find direct upstream nodes
    for edge in edges:
        if edge.get('target') == current_node_id:
            source_node_id = edge.get('source')
            if source_node_id and source_node_id not in visited:
                upstream_nodes.add(source_node_id)
                # Recursively find upstream of upstream
                upstream_nodes.update(
                    ChainCallDataService._find_all_upstream_nodes(
                        source_node_id, edges, visited.copy()
                    )
                )
    
    return upstream_nodes
```

### 执行顺序排序

使用 `timeline.execution_order` 或 `execution_history` 排序：

```python
# Get execution order from timeline or execution_history
timeline = node_result.get('timeline', {})
execution_order = timeline.get('execution_order')
if execution_order is None:
    # Fallback: use position in execution_history
    try:
        execution_order = execution_history.index(node_id) + 1
    except ValueError:
        execution_order = 999999  # Put at end if not found

# Sort by execution order
sorted_node_ids = sorted(
    accumulated_context.keys(),
    key=lambda nid: accumulated_context[nid]['execution_order']
)
```

### 数据收集

只收集已完成节点的数据：

```python
for node_id in upstream_node_ids:
    node_result = node_results.get(node_id)
    if not node_result or node_result.get('status') != 'completed':
        logger.debug(f"Skipping node {node_id} (not completed)")
        continue
    
    # Extract output data
    output = node_result.get('output', {})
    output_data = output.get('data', {}) if isinstance(output, dict) else {}
    structured_output = output.get('structured_output', {}) if isinstance(output, dict) else {}
    
    accumulated_context[node_id] = {
        'node_type': node_type,
        'execution_order': execution_order,
        'output_data': output_data,
        'structured_output': structured_output,
        'executed_at': timeline.get('executed_at', '')
    }
```

## 使用示例

### 在 HITL Approval 节点中使用

```python
# 在 execute() 方法中
if self.task_type == 'approval':
    # Extract accumulated upstream context
    from app.services.workflow.chain_call_data_service import ChainCallDataService
    accumulated_result = ChainCallDataService.extract_accumulated_upstream_data(
        state, self.node_id
    )
    accumulated_context = accumulated_result.get('accumulated_context', {})
    
    # Merge with direct upstream context
    enhanced_upstream_context = {
        'direct_upstream': upstream_context,
        'accumulated_context': accumulated_context
    }
    
    # Pass to SteeringTask
    steering_task_ids, approver_info_list = await self._create_steering_tasks(
        session=session,
        approver_job_title_ids=approver_job_title_ids,
        workflow_task_id=workflow_task_id,
        approval_title=approval_title,
        approval_description=approval_description,
        upstream_context=enhanced_upstream_context,  # Enhanced context
        created_by_user_id=created_by_user_id
    )
```

### 访问累计上下文

```python
# 在 SteeringTask 或前端代码中
accumulated_context = enhanced_context.get('accumulated_context', {})

# 访问特定节点的数据
app_node_data = accumulated_context.get('app-1', {})
conclusion_detail = app_node_data.get('output_data', {}).get('conclusion_detail', '')

# 按执行顺序遍历
execution_order = enhanced_context.get('execution_order', [])
for node_id in execution_order:
    node_data = accumulated_context.get(node_id, {})
    print(f"Node {node_id}: {node_data.get('node_type')}")
```

## 性能考虑

### 优化策略

1. **只收集已完成节点**: 跳过 `pending` 或 `failed` 状态的节点
2. **避免循环依赖**: DFS 算法检测并避免循环
3. **按需使用**: 只在需要完整上下文的节点（如 approval）中使用

### 内存占用

- 每个节点的数据包括 `output.data` 和 `output.structured_output`
- 对于长 workflow（20+ 节点），累计上下文可能较大
- 建议：只在必要时使用，避免在简单节点中使用

## 向后兼容

### 保持直接上游访问

累计上下文功能**不破坏**现有的直接上游节点访问：

```python
# 仍然可以使用直接上游上下文
upstream_context = self._get_upstream_context(state)

# 同时可以使用累计上下文
accumulated_result = ChainCallDataService.extract_accumulated_upstream_data(state, self.node_id)
```

### 增强的上下文结构

传递给 SteeringTask 的上下文包含两部分：

```python
{
    'direct_upstream': {...},      # 向后兼容：直接上游节点数据
    'accumulated_context': {...}    # 新增：所有上游节点数据
}
```

## 测试建议

### 单元测试

1. **测试递归查找**:
   - 简单链式结构（A → B → C）
   - 分支结构（A → B, A → C → D）
   - 循环检测（A → B → A）

2. **测试执行顺序排序**:
   - 验证节点按 `execution_order` 正确排序
   - 验证 `execution_history` fallback 逻辑

3. **测试数据收集**:
   - 验证只收集 `completed` 状态的节点
   - 验证 `output.data` 和 `output.structured_output` 都包含

### 集成测试

1. **测试 HITL Approval 节点**:
   - 验证累计上下文正确传递给 SteeringTask
   - 验证前端可以访问所有上游节点数据

2. **测试性能**:
   - 测试长 workflow（20+ 节点）的性能
   - 验证内存占用在可接受范围内

## 更新记录

- **2026-01-26**: 初始实现
  - 添加 `extract_accumulated_upstream_data()` 方法
  - 在 HITL approval 节点中集成累计上下文收集
  - 支持按执行顺序收集所有上游节点数据

---

## 相关文档

- `workflow-nodes-complete-output-structure.md` - 节点输出结构说明
- `workflow-nodes-input-output-analysis.md` - 节点输入输出分析
- `workflow-chain-call-design.md` - 链式调用设计
