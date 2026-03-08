# Workflow节点输入输出分析

## 概述

本文档分析了workflow中各个节点的输入输出数据格式和流转逻辑。基于代码实现分析，详细说明了每个节点如何处理输入数据并产生输出数据。

## 节点列表

1. **Start Node** - 工作流起始节点
2. **App Node** - 应用节点（执行AI分析任务）
3. **Data Transform Node** - 数据转换节点（从文本提取结构化数据）
4. **Condition Node** - 条件判断节点（路由工作流）
5. **Human in the Loop Node** - 人工确认节点
6. **Tool Node** - 工具节点（邮件和爬虫功能）
7. **Approval Node** - 审批节点
8. **End Node** - 工作流结束节点

---

## 1. Start Node（起始节点）

### 输入
- **数据来源**: 从 `state['input_data']` 字段获取
- **数据来源说明**: 
  - 当用户执行workflow时，通过API `POST /workbench/workflow-tasks/{task_id}/execute` 传入的 `input_data` 参数
  - 该数据会被放入workflow的初始状态 `initial_state['input_data']` 中
  - Start Node从 `state.get('input_data', {})` 读取这个数据
- **数据格式**: 任意字典类型 (`Dict[str, Any]`)，例如：
  ```python
  {
      "url": "https://example.com/document.pdf",
      "requirement": "分析文档内容",
      # ... 其他任意字段
  }
  ```
- **数据示例**: 
  ```python
  # 用户调用API时传入
  {
      "input_data": {
          "document_url": "https://example.com/doc.pdf",
          "analysis_type": "supplier_check"
      }
  }
  ```

### 输出
- **输出位置**: `state['node_results'][node_id]`
- **输出格式**: 
```python
{
    'status': 'completed',
    'output': {
        # 直接输出接收到的input_data的所有字段
        "document_url": "https://example.com/doc.pdf",
        "analysis_type": "supplier_check",
        # ... 其他input_data中的字段
    }
}
```
- **字段说明**:
  - `status`: 固定为 `'completed'`，表示节点执行成功
  - `output`: 直接输出接收到的 `input_data` 的所有内容，**不做任何处理或转换**
- **输出用途**: 下游节点（如App Node）可以通过 `state['node_results'][start_node_id]['output']` 获取这些数据

### 代码位置
- 实现: `backend/app/services/workflow/workflow_execution_service.py:443-460`
- 关键代码:
  ```python
  input_data = state.get('input_data', {})
  state['node_results'][node['id']] = {
      'status': 'completed',
      'output': input_data  # 直接赋值，不做任何处理
  }
  ```

---

## 2. App Node（应用节点）

### 输入
- **数据来源路径**:
  1. **优先**: 从 `state['node_results'][previous_node_id]['output']` 获取（如果存在前一个节点）
     - 通过 `self.find_previous_nodes(node_id, edges)` 找到前一个节点ID
     - 从 `state['node_results'][source_node_id]['output']` 读取数据
  2. **备选**: 从 `state['input_data']` 获取（如果没有前一个节点）
- **数据格式**: 任意字典类型 (`Dict[str, Any]`)
- **数据示例**:
  ```python
  # 从前一个Start Node获取
  {
      "document_url": "https://example.com/doc.pdf",
      "analysis_type": "supplier_check"
  }
  ```
- **特殊说明**: 
  - 如果App Node配置了 `inputMode='file'` 或 `inputMode='url'`，则**不需要**从上游节点获取输入数据
  - 输入数据会被保存在输出的 `input_received` 字段中，供后续节点使用

### 输出
- **输出位置**: `state['node_results'][app_node_id]`
- **输出格式**:
```python
{
    'status': 'completed' | 'failed',
    'output': {
        'conclusion_detail': str,        # Markdown格式的分析结果（必需）
        'dag_run_id': str,              # Airflow DAG运行ID（必需）
        'input_received': dict,         # 接收到的输入数据（必需）
        'text': str,                    # 文本输出（同conclusion_detail，必需）
        'reasoning_content': str,       # 推理内容（同conclusion_detail，必需）
        'structured_output': dict       # 可选：自动提取的结构化数据（如果提取成功）
    }
}
```
- **字段详细说明**:
  - `conclusion_detail`: App Node执行后生成的Markdown格式报告内容，从Airflow DAG的 `result_data.markdown_conclusion` 字段获取
  - `dag_run_id`: Airflow DAG运行ID，用于追踪任务执行状态
  - `input_received`: **保存接收到的输入数据**，便于后续节点（如Data Transform Node）使用
  - `text`: 与 `conclusion_detail` 相同，用于兼容性
  - `reasoning_content`: 与 `conclusion_detail` 相同，用于兼容性
  - `structured_output`: **可选字段**，自动从 `conclusion_detail` 中提取的结构化数据（JSON格式），基于自动生成的JSON Schema
- **输出示例**:
  ```python
  {
      'status': 'completed',
      'output': {
          'conclusion_detail': '# 分析报告\n\n根据文档分析...',
          'dag_run_id': 'dag_run_12345',
          'input_received': {
              'document_url': 'https://example.com/doc.pdf'
          },
          'text': '# 分析报告\n\n根据文档分析...',
          'reasoning_content': '# 分析报告\n\n根据文档分析...',
          'structured_output': {
              'has_qualified_suppliers': True,
              'supplier_count': 3,
              'qualified_option_names': ['供应商A', '供应商B', '供应商C']
          }
      }
  }
  ```

### 代码位置
- 实现: `backend/app/services/workflow/workflow_execution_service.py:462-706`
- 关键代码:
  ```python
  # 获取输入
  previous_nodes = self.find_previous_nodes(node['id'], edges)
  input_data = state.get('input_data', {})
  if previous_nodes:
      source_node_id = previous_nodes[0]
      source_result = state.get('node_results', {}).get(source_node_id, {})
      input_data = source_result.get('output', input_data)
  
  # 输出
  state['node_results'][node['id']] = {
      'status': 'completed',
      'output': {
          'conclusion_detail': markdown_result,
          'dag_run_id': dag_run_id,
          'input_received': input_data,
          'text': markdown_result,
          'reasoning_content': markdown_result,
          'structured_output': structured_output  # 如果提取成功
      }
  }
  ```

---

## 3. Data Transform Node（数据转换节点）

### 输入
- **数据来源路径**:
  1. **优先**: 从上游节点的 `state['node_results'][source_node_id]['output']` 获取
  2. **备选**: 从 `state['input_data']` 获取（如果 `sourceType='workflow_input'`）
- **数据源配置** (`dataSource`):
  - `sourceType`: `'previous_node'` | `'workflow_input'` | `'specific_node'`
  - `sourceNodeId`: 可选，指定具体节点ID（当 `sourceType='specific_node'` 时必需）
- **文本提取逻辑**（从上游节点的output中提取文本）:
  1. **优先**: 从 `output.conclusion_detail` 字段提取（App Node总是包含此字段）
  2. **备选1**: 从 `output.markdown_conclusion` 字段提取
  3. **备选2**: 从 `output.result` 字段提取
  4. **最后**: 将整个 `output` 字典转换为JSON字符串
- **输入数据示例**:
  ```python
  # 从App Node的output获取
  {
      'conclusion_detail': '# 分析报告\n\n根据文档分析，发现3个合格供应商...',
      'dag_run_id': 'dag_run_12345',
      'input_received': {...}
  }
  # Data Transform Node会提取 conclusion_detail 字段的文本内容
  ```

### 输出
- **输出位置**: `state['node_results'][data_transform_node_id]`
- **输出格式**:
```python
{
    'status': 'completed' | 'failed',
    'output': {
        # 直接输出结构化数据字典，不包含'outputs'包装
        'field1': value1,
        'field2': value2,
        # ... 根据extractionRules提取的所有字段
    },
    'attempt': int  # 重试次数（仅在completed时存在）
}
```
- **字段说明**:
  - `output`: **直接是结构化数据字典**，根据配置的 `extractionRules` 从输入文本中提取
  - 每个字段根据 `extractionRules` 中定义的 `fieldName`、`dataType`、`extractionPrompt` 进行提取
  - 支持的数据类型: `string`, `number`, `boolean`, `array`, `object`
  - 输出格式: `json` (默认) | `xml` | `csv`（但实际存储都是字典格式）
- **输出示例**:
  ```python
  {
      'status': 'completed',
      'output': {
          'has_qualified_suppliers': True,
          'supplier_count': 3,
          'qualified_option_names': ['供应商A', '供应商B', '供应商C'],
          'is_request_approved': False,
          'approval_reasons': ['价格过高', '资质不全']
      },
      'attempt': 1
  }
  ```
- **重要说明**: 
  - Data Transform Node的 `output` **直接是结构化数据**，不是 `{'outputs': {...}}` 格式
  - 但Condition Node在读取时，会检查是否有 `outputs` 字段，如果没有则使用整个 `output`

### 代码位置
- 实现: `backend/app/services/workflow/nodes/data_transform_node.py`
- 关键代码:
  ```python
  # 获取输入文本
  input_text = self._get_input_text(state)  # 从上游节点的output.conclusion_detail提取
  
  # 输出
  state['node_results'][node_id] = {
      'status': 'completed',
      'output': structured_data,  # 直接是字典，不是{'outputs': structured_data}
      'attempt': attempt + 1
  }
  ```

---

## 4. Condition Node（条件判断节点）

### 输入
- **数据来源路径**: 从上游节点的 `state['node_results'][upstream_node_id]['output']` 获取
- **数据提取逻辑**:
  1. **优先**: 从 `output.outputs` 字段获取结构化数据（如果存在）
  2. **备选**: 使用整个 `output` 作为上下文（如果没有 `outputs` 字段）
- **输入数据示例**:
  ```python
  # 从Data Transform Node的output获取
  {
      'has_qualified_suppliers': True,
      'supplier_count': 3,
      'qualified_option_names': ['供应商A', '供应商B', '供应商C']
  }
  # 或者从App Node的structured_output获取
  {
      'is_request_approved': False,
      'approval_reasons': ['价格过高']
  }
  ```
- **说明**: Condition Node会使用这个结构化数据来评估条件表达式

### 输出
- **输出位置**: `state['node_results'][condition_node_id]`
- **输出格式**:
```python
{
    'status': 'completed' | 'failed',
    'output': {
        'condition_result': bool,           # 条件表达式求值结果（必需）
        'branch': 'True' | 'False',          # 路由分支（必需）
        'branch_label': str,                 # 分支标签（用户自定义，必需）
        'condition_expression': str,         # 条件表达式原文（必需）
        'evaluated_context': dict            # 评估上下文（必需，用于下游Human in the Loop Node）
    }
}
```
- **字段详细说明**:
  - `condition_result`: 根据 `conditionExpression` 和输入数据求值得到的布尔结果
  - `branch`: 根据 `condition_result` 确定的路由分支（'True' 或 'False'），用于工作流路由
  - `branch_label`: 分支标签（用户自定义），例如 "通过" / "拒绝"
  - `condition_expression`: 条件表达式原文，例如 `"has_qualified_suppliers && supplier_count > 0"`
  - `evaluated_context`: **从输入数据中提取的相关字段**，用于下游Human in the Loop Node
    - 提取逻辑：
      1. 如果输入数据中存在 `evaluated_context` 字段，直接使用
      2. 否则，从输入数据中提取以下字段组装：
         - `is_request_approved`: 审批判断场景
         - `approval_reasons`: 审批原因列表
         - `required_actions`: 必需的操作
         - `qualified_option_names`: 供应商选择场景（优先级最高）
- **输出示例**:
  ```python
  {
      'status': 'completed',
      'output': {
          'condition_result': True,
          'branch': 'True',
          'branch_label': '有合格供应商',
          'condition_expression': 'has_qualified_suppliers && supplier_count > 0',
          'evaluated_context': {
              'qualified_option_names': ['供应商A', '供应商B', '供应商C'],
              'supplier_count': 3
          }
      }
  }
  ```

### 条件表达式支持
- 支持JavaScript风格的表达式: `===`, `!==`, `==`, `!=`, `>`, `<`, `>=`, `<=`
- 逻辑运算符: `&&`, `||`, `!`
- 数组操作: `.length`, `.includes()`
- 示例: `"has_qualified_suppliers && supplier_count > 0"`

### 代码位置
- 实现: `backend/app/services/workflow/nodes/condition_node.py`
- 关键代码:
  ```python
  # 获取输入
  upstream_output = self._get_upstream_output(state)
  # 优先从 output.outputs 获取，否则使用整个 output
  
  # 评估条件
  condition_result = self._evaluate_condition(condition_expression, upstream_output)
  
  # 提取evaluated_context
  evaluated_context = self._extract_evaluated_context(upstream_output)
  
  # 输出
  state['node_results'][node_id] = {
      'status': 'completed',
      'output': {
          'condition_result': condition_result,
          'branch': 'True' if condition_result else 'False',
          'branch_label': ...,
          'condition_expression': ...,
          'evaluated_context': evaluated_context
      }
  }
  ```

---

## 5. Human in the Loop Node（人工确认节点）

### 输入
- **数据来源路径**: 从上游Condition Node的 `state['node_results'][condition_node_id]['output']['evaluated_context']` 获取
- **数据格式**: 字典类型 (`Dict[str, Any]`)
- **必需字段**（至少需要以下之一）:
  - `qualified_option_names`: 供应商选择场景（优先级最高）
  - `is_request_approved`: 审批判断场景
- **输入数据示例**:
  ```python
  # 供应商选择场景
  {
      'qualified_option_names': ['供应商A', '供应商B', '供应商C'],
      'supplier_count': 3
  }
  
  # 审批判断场景
  {
      'is_request_approved': False,
      'approval_reasons': ['价格过高', '资质不全'],
      'required_actions': '需要重新报价'
  }
  ```
- **场景类型检测**:
  - `supplier_selection`: 如果存在 `qualified_option_names` 字段（优先级最高）
  - `approval_judgment`: 如果存在 `is_request_approved` 字段

### 输出（初始状态 - pending）
- **输出位置**: `state['node_results'][human_in_the_loop_node_id]`
- **输出格式**:
```python
{
    'status': 'pending',
    'output': {
        'waiting_for_human_input': True,    # 固定为True（必需）
        'scenario_type': 'supplier_selection' | 'approval_judgment',  # 场景类型（必需）
        'evaluated_context': dict,         # 从Condition Node获取的上下文（必需）
        'timeout': int,                    # 超时时间（秒，可选）
        'created_at': str                  # ISO格式时间戳（必需）
    }
}
```
- **说明**: 节点执行后会进入 `pending` 状态，等待人工输入

### 输出（完成状态 - completed）
- **输出位置**: `state['node_results'][human_in_the_loop_node_id]`（通过 `process_human_decision` 方法更新）

#### 供应商选择场景 (`supplier_selection`)
```python
{
    'status': 'completed',
    'output': {
        'decision_type': 'supplier_selection',  # 固定值（必需）
        'status': 'completed',                  # 固定值（必需）
        'human_decision': 'approved' | 'rejected',  # 人工决策（必需）
        'selected_suppliers': list,            # 选中的供应商列表（必需）
        'all_selected': bool,                  # 是否全选（必需）
        'rejection_reason': str | None,        # 拒绝原因（如果rejected）
        'decision_timestamp': str,             # ISO格式时间戳（必需）
        'reviewer_id': str,                    # 审核人ID（可选）
        'context_snapshot': {                  # 上下文快照（必需）
            'qualified_option_names': list,
            'available_options_count': int
        }
    }
}
```

#### 审批判断场景 (`approval_judgment`)
```python
{
    'status': 'completed',
    'output': {
        'decision_type': 'approval_judgment',  # 固定值（必需）
        'status': 'completed',                  # 固定值（必需）
        'human_decision': 'approve' | 'reject', # 人工决策（必需）
        'approval_reason': str | None,          # 批准原因（如果approve）
        'rejection_reason': str | None,         # 拒绝原因（如果reject）
        'decision_timestamp': str,             # ISO格式时间戳（必需）
        'reviewer_id': str,                    # 审核人ID（可选）
        'context_snapshot': {                  # 上下文快照（必需）
            'is_request_approved': bool,
            'approval_reasons': list,
            'required_actions': str
        }
    }
}
```

### 输出示例
- **供应商选择场景（approved）**:
  ```python
  {
      'status': 'completed',
      'output': {
          'decision_type': 'supplier_selection',
          'status': 'completed',
          'human_decision': 'approved',
          'selected_suppliers': [
              {'name': '供应商A'},
              {'name': '供应商B'}
          ],
          'all_selected': False,
          'rejection_reason': None,
          'decision_timestamp': '2024-01-01T12:00:00Z',
          'reviewer_id': 'user_123',
          'context_snapshot': {
              'qualified_option_names': ['供应商A', '供应商B', '供应商C'],
              'available_options_count': 3
          }
      }
  }
  ```

- **审批判断场景（reject）**:
  ```python
  {
      'status': 'completed',
      'output': {
          'decision_type': 'approval_judgment',
          'status': 'completed',
          'human_decision': 'reject',
          'approval_reason': None,
          'rejection_reason': '价格超出预算',
          'decision_timestamp': '2024-01-01T12:00:00Z',
          'reviewer_id': 'user_123',
          'context_snapshot': {
              'is_request_approved': False,
              'approval_reasons': ['价格过高'],
              'required_actions': '需要重新报价'
          }
      }
  }
  ```

### 代码位置
- 实现: `backend/app/services/workflow/nodes/human_in_the_loop_node.py`
- 关键代码:
  ```python
  # 获取输入
  evaluated_context = self._get_evaluated_context(state)
  # 从 state['node_results'][upstream_node_id]['output']['evaluated_context'] 获取
  
  # 初始输出（pending状态）
  state['node_results'][node_id] = {
      'status': 'pending',
      'output': {
          'waiting_for_human_input': True,
          'scenario_type': scenario_type,
          'evaluated_context': evaluated_context,
          'timeout': self.timeout,
          'created_at': datetime.now(timezone.utc).isoformat()
      }
  }
  
  # 完成输出（通过process_human_decision方法更新）
  state['node_results'][node_id] = {
      'status': 'completed',
      'output': {
          'decision_type': ...,
          'human_decision': ...,
          # ... 其他字段
      }
  }
  ```

---

## 6. Tool Node（工具节点）

### 输入
- **来源**: 前一个节点的 `output` 字段（如果存在）
- **格式**: 任意字典类型
- **说明**: 目前实现为占位符，具体输入格式待实现

### 输出
- **格式**:
```python
{
    'status': 'completed',
    'output': {
        'tool_result': 'Placeholder'  # 占位符，待实现
    }
}
```
- **工具类型**:
  - `email`: 邮件发送功能
  - `web_scraper`: 网页爬虫功能
  - `api_call`: API调用
  - `database_query`: 数据库查询

### 配置
- `toolConfig.toolType`: 工具类型
- `emailConfig`: 邮件配置（收件人、主题、模板等）
- `webScraperConfig`: 爬虫配置（URL、选择器等）

### 代码位置
- 实现: `backend/app/services/workflow/workflow_execution_service.py:1693-1703`
- **状态**: ⚠️ 待实现（目前为占位符）

---

## 7. Approval Node（审批节点）

### 输入
- **来源**: 前一个节点的 `output` 字段（如果存在）
- **格式**: 任意字典类型
- **说明**: 目前实现为占位符，具体输入格式待实现

### 输出
- **格式**:
```python
{
    'status': 'pending',
    'output': {
        'waiting_for_approval': True  # 占位符，待实现
    }
}
```

### 配置
- `approvalConfig.assigneeConfig`: 审批人配置
  - `assigneeType`: `'job_title'` | `'role'` | `'individual'` | `'functional_director'`
  - `jobTitleIds`: 职位ID列表
  - `roleIds`: 角色ID列表
  - `userIds`: 用户ID列表
- `approvalConfig.approvalLogic`: `'any'` | `'all'`（任一审批/全部审批）
- `approvalConfig.minApprovers`: 最小审批人数

### 代码位置
- 实现: `backend/app/services/workflow/workflow_execution_service.py:1681-1691`
- **状态**: ⚠️ 待实现（目前为占位符）

---

## 8. End Node（结束节点）

### 输入
- **数据来源**: End Node本身**不主动读取**任何输入数据
- **数据访问说明**: 
  - End Node执行时，可以通过 `state['node_results']` 访问**所有已执行节点**的输出数据
  - 但End Node的代码实现中**没有读取**这些数据，只是标记工作流完成
- **可访问的数据**: 
  - `state['node_results']`: 包含所有已执行节点的结果
  - `state['execution_history']`: 节点执行历史列表
  - `state['structured_data']`: 结构化数据（如果有Data Transform Node）
  - `state['errors']`: 错误信息列表
- **说明**: End Node的作用是**标记工作流执行完成**，而不是处理数据

### 输出
- **输出位置**: `state['node_results'][node_id]`
- **输出格式**:
```python
{
    'status': 'completed',
    'output': {
        'workflow_completed': True
    }
}
```
- **字段说明**:
  - `status`: 固定为 `'completed'`，表示节点执行成功
  - `output.workflow_completed`: 固定为 `True`，标识整个工作流已完成
- **输出用途**: 
  - 用于标识工作流执行完成
  - 前端可以通过检查End Node的状态来判断workflow是否执行完毕
  - 不包含任何业务数据，仅作为完成标记

### 代码位置
- 实现: `backend/app/services/workflow/workflow_execution_service.py:1705-1714`
- 关键代码:
  ```python
  async def end_node(state: WorkflowState) -> WorkflowState:
      state['current_node_id'] = node['id']
      state['node_results'][node['id']] = {
          'status': 'completed',
          'output': {'workflow_completed': True}  # 固定输出，不读取任何输入
      }
      return state
  ```

---

## 数据流转示例

### 典型流程1: App → Data Transform → Condition → Human in the Loop

```
Start Node
  ↓ (input_data)
App Node
  ↓ output: {conclusion_detail: "...", structured_output: {...}}
Data Transform Node
  ↓ output: {outputs: {field1: value1, field2: value2}}
Condition Node
  ↓ output: {condition_result: true, evaluated_context: {...}}
Human in the Loop Node
  ↓ output: {decision_type: "...", human_decision: "..."}
End Node
```

### 典型流程2: App → Condition（直接使用structured_output）

```
Start Node
  ↓ (input_data)
App Node
  ↓ output: {conclusion_detail: "...", structured_output: {...}}
Condition Node
  ↓ (直接使用App Node的structured_output作为输入)
  ↓ output: {condition_result: true, evaluated_context: {...}}
End Node
```

---

## 注意事项

1. **Data Transform Node已废弃**: 新工作流应使用App Node的 `structured_output` 功能，而不是单独的Data Transform Node
2. **Tool Node和Approval Node**: 目前实现为占位符，具体功能待开发
3. **输入数据优先级**: 大多数节点优先使用前一个节点的输出，如果没有前一个节点则使用workflow的 `input_data`
4. **错误处理**: 所有节点在失败时都会设置 `status='failed'` 并在 `output` 中包含 `error` 字段
5. **执行历史**: 所有节点执行后都会更新 `execution_history`，用于追踪节点执行顺序

---

## 待确认问题

1. **Tool Node的输入输出格式**: 需要确认邮件和爬虫功能的具体输入输出格式
2. **Approval Node的输入输出格式**: 需要确认审批流程的具体输入输出格式
3. **Human in the Loop Node的输出使用**: 下游节点如何使用人工决策的结果？
4. **Condition Node的evaluated_context提取逻辑**: 是否需要支持更多场景类型？

---

## 更新记录

- 2024-XX-XX: 初始版本，基于代码分析整理

