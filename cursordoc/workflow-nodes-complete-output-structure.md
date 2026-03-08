# Workflow节点完整输出数据结构

## 概述

本文档详细描述了workflow系统中各个节点的完整输出数据结构。所有节点的输出都存储在 `state['node_results'][node_id]` 中，采用新的统一结构：

```python
{
    'status': 'completed' | 'pending' | 'failed',
    'input': {
        'data': dict,  # 实际输入数据(仅包含节点需要的输入字段)
        'source_node_id': str | None,  # 数据来源节点ID(用于追溯)
        'source_type': str,  # 'upstream' | 'workflow_input' | 'config'
    },
    'operation': {
        'type': str,  # 操作类型(如'app_execution', 'classification', 'approval')
        'description': str,  # 操作描述(人类可读)
        'config': dict,  # 操作配置(如appNodeId, taskType等)
        'duration_ms': int | None,  # 执行耗时(毫秒)
    },
    'output': {
        'data': dict,  # 该节点产生的实际输出数据
        'structured_output': dict,  # 结构化业务数据(用于view output展示)
    },
    'timeline': {
        'executed_at': str,  # 执行时间(ISO格式)
        'execution_order': int,  # 执行顺序(从1开始)
    }
}
```

## 重要变更说明

### 数据分离原则

- **数据分离**: 每个节点只存储自己产生的数据，不包含上游节点的数据副本
- **按时间顺序存储**: 节点数据按执行时间顺序存储在 `node_results` 中
- **统一接口**: 提供统一的查询接口，支持按时间线追溯

### 统一数据结构 (v3.0)

**所有节点统一使用 v3.0 结构**:
- ✅ 使用 `output.data` 存储节点产生的实际数据
- ✅ 使用 `output.structured_output` 存储结构化业务数据
- ❌ **不再使用** `merged_context` 和 `upstream_context`
- ❌ **不再使用** `_metadata`（元数据存储在 `timeline` 中）

### 链式调用支持

**混合方案**:
- **Human In The Loop Node**: 只传递 `file_ingestion_record_ids`（前端按需获取文件内容）
- **App Node**: 获取文件内容（从已提取的文本，用于分析）
- **Question Classifier Node**: 从 `output.data.conclusion_detail` 提取输入文本

---

## 1. Start Node（起始节点）

### 输入位置
- `state['input_data']`

### 完整输入结构
```python
{
    # 工作流的初始输入数据
    # 通常为空字典 {}（基于代码和数据库验证）
    # 如果用户在执行workflow时传入了input_data，则包含这些字段
}
```

### 输入数据来源
- **数据来源**：从 `state.get('input_data', {})` 获取
- **实际验证结果**（基于数据库验证）：
  - 数据库中 `input_data = {}`（空字典）
  - 日志显示：`keys=[] len=0`（空字典）
- **代码逻辑**（`backend/app/services/workflow/workflow_execution_service.py:514`）：
  ```python
  input_data = state.get('input_data', {})  # 从 state 中获取
  ```

### 字段说明
- **`input_data`**: 工作流的初始输入数据
  - 通常为空字典 `{}`（基于代码和数据库验证）
  - 如果用户在执行workflow时传入了 `input_data`，则包含这些字段
  - 不包含 `app_node_configs` 的内容（`app_node_configs` 单独存储）

### 输出位置
- `state['node_results'][start_node_id]`

### 完整输出结构
```python
{
    'status': 'completed',  # 固定为 'completed'
    'input': {
        'data': {},  # 固定为空字典（Start Node 没有输入）
        'source_type': 'workflow_input'  # 数据来源类型
    },
    'operation': {
        'type': 'start',  # 操作类型
        'description': 'Workflow started',  # 操作描述
        'config': {}  # 操作配置（空字典）
    },
    'output': {
        'data': {},  # 从 input_data 复制，通常为空字典
        # 注意：不包含下游 App Node 的配置信息（如 files, dagRunId, inputMode, url 等）
        # 这些配置信息存储在 app_node_configs 中，不在 Start Node 的输出中
        'structured_output': {
            'node_type': 'start',
            'data': {}  # 从 input_data 复制
        }
    },
    'timeline': {
        'executed_at': str,  # 执行时间(ISO格式)
        'execution_order': int  # 执行顺序(从1开始)
    }
}
```

### 字段说明
- **`status`**: 固定为 `'completed'`，表示节点执行成功
- **`input.data`**: 固定为空字典 `{}`（Start Node 没有输入）
- **`input.source_type`**: 固定为 `'workflow_input'`，表示数据来源是工作流输入
- **`output.data`**: 从 `state.get('input_data', {})` 复制，通常为空字典 `{}`
  - **重要**: 不包含下游 App Node 的配置信息（如 `files`, `dagRunId`, `inputMode`, `url` 等）
  - 这些配置信息存储在 `app_node_configs[node_id]` 中，不在 Start Node 的输出中
  - **实际验证结果**：基于数据库验证，`output.data = {}`（空字典）
- **`output.structured_output`**: 结构化业务数据，用于 view output 展示
  - `node_type`: 固定为 `'start'`
  - `data`: 从 `input_data` 复制
- **`operation.type`**: 固定为 `'start'`
- **`operation.description`**: 固定为 `'Workflow started'`
- **`timeline.execution_order`**: 执行顺序，从 1 开始递增

### 代码位置
- `backend/app/services/workflow/workflow_execution_service.py` - `_create_start_node()` 方法（约第510-525行）

---

## 2. App Node（应用节点）

### 输入位置
- 从上游节点的输出获取，或从 `state['input_data']` 获取

### 完整输入结构
```python
{
    # 从上游节点（如 Start Node）获取的输入数据
    # 基于代码验证：如果存在上游节点，从 upstream_node_id 的 output 获取
    # 如果不存在上游节点，从 state['input_data'] 获取
    # 实际验证结果（基于日志）：从 start-2 获取，keys=[] len=0（空字典）
}
```

### 输入数据来源
- **数据来源路径**（`backend/app/services/workflow/workflow_execution_service.py:534-543`）：
  1. **优先**：从上游节点的 `state['node_results'][upstream_node_id]['output']` 获取
     - 通过 `self.find_previous_nodes(node_id, edges)` 找到上游节点ID
     - 从 `state['node_results'][source_node_id]['output']` 读取数据
  2. **备选**：从 `state['input_data']` 获取（如果没有上游节点）
- **实际验证结果**（基于日志）：
  - 日志：`prev=['start-2'] keys=[] len=0`
  - 从 start-2 获取的输入为空字典 `{}`
- **配置数据来源**：
  - 从 `app_node_configs[node_id]` 获取执行配置
  - 包含：`dagRunId`、`inputMode`、`requirement`、`appNodeId`
  - **注意**：`app_node_configs` 不合并到输入数据中，单独使用

### 字段说明
- **上游节点输出**：如果存在上游节点，使用上游节点的 `output` 作为输入
- **`app_node_configs`**：单独从 `app_node_configs[node_id]` 获取配置
  - `dagRunId`：文件上传后生成的 DAG 运行ID（file 模式）
  - `url`：URL 地址（url 模式）
  - `inputMode`：输入模式（'file' 或 'url'）
  - `requirement`：分析需求文本
  - `appNodeId`：App Node ID

### 输出位置
- `state['node_results'][app_node_id]`

### 完整输出结构
```python
{
    'status': 'completed' | 'failed',
    'input': {
        'data': dict,  # 实际输入数据(通常为空,App Node主要使用config)
        'source_node_id': str | None,  # 数据来源节点ID
        'source_type': str,  # 'config' (App Node主要使用app_node_configs)
    },
    'operation': {
        'type': 'app_execution',
        'description': str,  # 如 'Execute app analysis (app_node_id=105)'
        'config': {
            'appNodeId': int,
            'inputMode': str,  # 'file' | 'url'
            'requirement': str,
        },
        'duration_ms': int | None,  # 执行耗时(毫秒)
    },
    'output': {
        'data': {
            # 核心业务数据(仅App Node产生的数据)
            'conclusion_detail': str,  # Markdown格式的分析报告内容
            'dag_run_id': str,  # Airflow DAG运行ID
        },
        'structured_output': {
            'node_type': 'app',
            'app_node_id': int,
            'conclusion_length': int,  # 结论长度
            'dag_run_id': str,
            'execution_status': str,  # 'success' | 'failed'
            'has_conclusion': bool,
        }
    },
    'timeline': {
        'executed_at': str,  # 执行时间(ISO格式)
        'execution_order': int,  # 执行顺序(从1开始)
    }
}
```

### 字段详细说明
- **`output.data.conclusion_detail`**: App Node执行后生成的Markdown格式报告内容
  - 从Airflow DAG的 `result_data.markdown_conclusion` 字段获取
  - 用于后续节点（如Question Classifier Node）的输入
- **`output.data.dag_run_id`**: Airflow DAG运行ID，用于追踪任务执行状态
- **`output.structured_output`**: 结构化业务数据，用于view output展示
  - 包含 `app_node_id`, `conclusion_length`, `dag_run_id`, `execution_status`, `has_conclusion` 等字段
- **`input.data`**: 节点接收的输入数据（通常为空，App Node主要使用 `app_node_configs`）
- **`input.source_type`**: 输入来源类型，App Node通常为 `'config'`（使用 `app_node_configs`）
- **`operation.config`**: 操作配置，包含 `appNodeId`, `inputMode`, `requirement` 等
- **`timeline.execution_order`**: 执行顺序，从1开始递增

### 代码位置
- `backend/app/services/workflow/workflow_execution_service.py` - `_create_app_node()` 方法（约第529-706行）

---

## 3. Human in the Loop Node（人工介入节点）

### 输入位置
- 从上游节点的输出获取

### 完整输入结构（所有taskType通用）

#### 输入数据来源
- **数据来源路径**（`backend/app/services/workflow/nodes/human_in_the_loop_node.py:391-429`）：
  1. 通过 `_get_upstream_context()` 方法获取上游节点上下文
  2. 从 `state['node_results'][upstream_node_id]['output']` 获取完整输出
  3. 通过 `get_context_from_upstream_output()` 统一提取逻辑提取上下文
- **提取逻辑**（`backend/app/services/workflow/node_utils.py:90-120`）：
  - **新结构 (v3.0)**：从 `output.data` 提取（所有节点类型）
  - 使用 `get_context_from_upstream_output()` 统一提取逻辑
- **实际验证结果**（基于日志，taskType=request_for_update）：
  - 日志：`Got upstream context from questionClassifier-3 (type=questionClassifier): ['branch', 'class_id', 'class_name']`
  - 从 questionClassifier-3 的 `output.data` 提取上下文（新结构 v3.0）

#### 配置数据来源
- **节点配置**（从节点定义中获取）：
  - `taskType`：任务类型（'approval'、'inform'、'request_for_update'、'assign'）
  - `approvalConfig`：审批配置（当 taskType='approval' 时）
  - `informConfig`：通知配置（当 taskType='inform' 时）
  - `updateConfig`：更新配置（当 taskType='request_for_update' 时）
  - `assignConfig`：分配配置（当 taskType='assign' 时）
  - `timeout`：超时时间（秒）
  - `timeoutAction`：超时动作（当前仅支持 'fail'）

#### 输入数据结构（基于代码验证）
```python
{
    # 从上游节点的 output.data 提取的上下文数据（新结构 v3.0）
    # 基于代码验证：使用 get_context_from_upstream_output() 统一提取逻辑
    # 对于 questionClassifier 类型：从 output.data 提取（包含 branch, class_id, class_name）
    # 对于 app 类型：从 output.data 提取（通常包含 conclusion_detail, dag_run_id）
    # 对于 humanInTheLoop 类型：从 output.data 提取（包含 human_decision, file_ingestion_record_ids 等）
}
```

### 输出位置
- `state['node_results'][human_node_id]`

### 任务类型说明
Human in the Loop Node根据 `taskType` 配置不同，会产生不同的输出结构：
- **`approval`**: 审批任务（需要审批人决策）
- **`inform`**: 通知任务（仅通知，不等待决策）
- **`request_for_update`**: 更新请求任务（需要提交更新信息）
- **`assign`**: 分配任务（分配给指定人员处理）

### 通用输出结构（所有taskType）
```python
{
    'status': 'completed' | 'pending' | 'failed',
    'input': {
        'data': dict,  # 实际输入数据(从上游节点提取的字段)
        'source_node_id': str | None,  # 数据来源节点ID
        'source_type': 'upstream'
    },
    'operation': {
        'type': str,  # 'approval' | 'inform' | 'request_for_update' | 'assign'
        'description': str,  # 操作描述
        'config': dict,  # 操作配置(包含taskType,相关配置等)
        'duration_ms': int | None,  # 执行耗时(毫秒)
    },
    'output': {
        'data': {
            # Human in the Loop Node产生的数据
            'human_decision': str,
            'decision_reason': str,
            # ... 其他taskType特定字段
        },
        'structured_output': {
            # 结构化业务数据(用于view output展示)
            'task_type': str,
            # ... taskType特定的structured_output字段
        }
    },
    'timeline': {
        'executed_at': str,  # 执行时间(ISO格式)
        'execution_order': int,  # 执行顺序(从1开始)
    }
}
```

---

### 3.1 Approval类型（taskType = 'approval'）

#### 完整输入结构
```python
{
    # 从上游节点的 output.data 提取的上下文数据（新结构 v3.0）
    # 基于代码验证：使用 get_context_from_upstream_output() 统一提取逻辑
    # 提取逻辑（backend/app/services/workflow/node_utils.py:90-120）：
    #   从 output.data 提取（新结构 v3.0，所有节点类型）
    # 实际验证结果（基于代码）：从上游节点的 output.data 提取（新结构 v3.0）
    # 包含上游节点的业务数据（不包含 merged_context 和 upstream_context）
}
```

#### 配置数据来源
- **`approvalConfig`**（从节点定义中获取）：
  - `approverJobTitleIds`：审批人职位ID列表（必需）
  - `title`：审批任务标题
  - `description`：审批任务描述
  - `approvalLogic`：审批逻辑（'any'、'all'、'majority'）
  - `tieBreaker`：平局处理（'approve'、'reject'）
- **`timeout`**：超时时间（秒，0表示无超时）
- **`timeoutAction`**：超时动作（当前仅支持 'fail'）

#### 用户提交数据（等待用户输入）
- **`decision_data`**（用户通过 Message Center 提交）：
  - `decision`：决策类型（'approved' 或 'rejected'）
  - `comments`：审批意见（字符串）
  - `file_ingestion_record_ids`：附件文件ID列表（可选）

#### 完整输出结构
```python
{
    'status': 'completed' | 'pending' | 'failed',
    'output': {
        # Human in the Loop Node特定字段
        'human_decision': str,  # 'approved' | 'rejected' | 'timeout'
        'decision_reason': str,  # 决策原因/评论
        
        # 统一字段结构
        'structured_output': {
            'task_type': 'approval',
            'human_decision': str,  # 'approved' | 'rejected' | 'timeout'
            'is_approved': bool,  # True表示批准，False表示拒绝或超时
            'approval_reason': str | None,  # 批准原因（仅在批准时存在）
            'rejection_reason': str | None,  # 拒绝原因（仅在拒绝时存在）
            'reviewer_email': str | None,  # 审批人邮箱
            'reviewer_job_title': str | None,  # 审批人职位
            'decision_timestamp': str,  # 决策时间（ISO格式，UTC时区）
        },
        
        # 上下文管理
        # 注意：新结构 (v3.0) 不再包含 upstream_context 和 merged_context 字段
        
        # 审批跟踪（仅当approvalLogic为'all'或'majority'时存在）
        'approval_tracking': {
            'total_approvers': int,  # 总审批人数
            'total_approved': int,  # 已批准人数
            'total_rejected': int,  # 已拒绝人数
            'total_pending': int,  # 待处理人数
            'approval_logic': str,  # 'any' | 'all' | 'majority'
            'tie_breaker': str,  # 'approve' | 'reject'
            'approvers': [
                {
                    'steering_task_id': int,
                    'user_id': str,
                    'user_email': str,
                    'job_title_id': int | None,
                    'job_title_name': str,
                    'status': str,  # 'pending' | 'approved' | 'rejected'
                    'comments': str | None,
                    'decision_time': str | None,  # ISO格式
                },
                # ... 更多审批人
            ]
        },
        
        # 元数据
        '_metadata': {
            'schema_version': '2.0',
            'node_id': str,
            'node_type': 'humanInTheLoop',
            'task_type': 'approval',
            'executed_at': str,  # ISO格式，UTC时区
            'upstream_node_id': str,
            'context_chain': list,
            'decision_timestamp': str,  # ISO格式，UTC时区
            'reviewer_id': str,
            'reviewer_email': str,
            'reviewer_job_title_id': int,
            'reviewer_job_title_name': str,
            'steering_task_id': int,  # SteeringTask ID
        }
    }
}
```

#### 状态说明
- **`pending`**: 等待审批人决策
- **`completed`**: 审批完成（已批准或已拒绝）
- **`failed`**: 审批失败（超时或其他错误）

---

### 3.2 Inform类型（taskType = 'inform'）

#### 完整输入结构
```python
{
    # 从上游节点的 merged_context 提取的上下文数据
    # 基于代码验证：使用 get_context_from_upstream_output() 统一提取逻辑
    # 实际验证结果（基于代码）：从上游节点的 merged_context 提取
    # 包含上游节点的所有上下文信息
}
```

#### 配置数据来源
- **`informConfig`**（从节点定义中获取）：
  - `informJobTitleIds`：被通知人职位ID列表（必需）
  - `title`：通知任务标题
  - `description`：通知任务描述
- **注意**：inform 类型不需要 timeout（立即完成，不等待用户输入）

#### 用户提交数据
- Inform 类型不等待用户提交，节点执行后立即完成
- 不包含用户提交数据

#### 完整输出结构
```python
{
    'status': 'completed',  # 固定为 'completed'（inform不等待决策）
    'input': {
        'data': dict,  # 从上游节点提取的输入数据
        'source_node_id': str | None,  # 上游节点ID
        'source_type': 'upstream'
    },
    'operation': {
        'type': 'inform',
        'description': str,  # 如 'Send inform notification to N users'
        'config': {
            'taskType': 'inform',
            'informJobTitleIds': list[int],
            'title': str,
        },
        'duration_ms': None,  # Inform立即完成
    },
    'output': {
        'data': {
            'human_decision': 'informed',  # 固定为 'informed'
            'decision_reason': None,  # inform类型无决策原因
        },
        'structured_output': {
            'task_type': 'inform',
            'human_decision': 'informed',
            'is_approved': None,  # inform类型不适用
            'notification_sent': bool,  # 通知是否成功发送
            'notification_status': str,  # 'sent' | 'failed'
            'notified_users': [
                {
                    'user_id': str,
                    'user_email': str,
                    'job_title_id': int | None,
                    'job_title_name': str,
                    'steering_task_id': int,
                    'is_read': bool,  # 是否已读
                    'read_at': str | None,  # 阅读时间（ISO格式）
                },
                # ... 更多被通知用户
            ],
            'notification_timestamp': str,  # 通知发送时间（ISO格式，UTC时区）
            'total_notified': int,  # 总通知人数
            'total_read': int,  # 已读人数
        }
    },
    'timeline': {
        'executed_at': str,  # 执行时间(ISO格式)
        'execution_order': int,  # 执行顺序(从1开始)
    }
}
```

#### 特点说明
- **立即完成**: inform类型不等待用户决策，节点执行后立即完成
- **无决策数据**: `decision_reason` 为 `None`，`is_approved` 为 `None`
- **通知跟踪**: 包含被通知用户的列表和阅读状态

---

### 3.3 Request for Update类型（taskType = 'request_for_update'）

#### 完整输入结构
```python
{
    # 从上游节点的 output.data 提取的上下文数据（新结构 v3.0）
    # 基于代码验证：使用 get_context_from_upstream_output() 统一提取逻辑
    # 实际验证结果（基于日志和数据库）：
    #   从 questionClassifier-3 的 output.data 提取
    #   包含：branch, class_id, class_name
    #   实际数据示例（基于数据库验证）：
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
        # 注意：新结构 (v3.0) 不再包含 upstream_context, merged_context, _metadata
    }
}
```

#### 配置数据来源
- **`updateConfig`**（从节点定义中获取）：
  - `updateJobTitleIds`：可提交更新的人员职位ID列表（必需）
  - `title`：更新任务标题
  - `description`：更新任务描述
- **`timeout`**：超时时间（秒，0表示无超时）
  - **实际验证结果**（基于日志）：`43200`（12小时）
- **`timeoutAction`**：超时动作（当前仅支持 'fail'）

#### 用户提交数据（等待用户输入）
- **`decision_data`**（用户通过 Message Center 提交）：
  - `decision`：决策类型（'updated' 或 'not_updated'）
  - `update_info`：更新信息内容（字符串）
  - `file_ingestion_record_ids`：附件文件ID列表
  - `comments`：评论（可选）
- **实际验证结果**（基于数据库）：
  - `decision`: `"updated"`
  - `update_info`: `"yy"`
  - `file_ingestion_record_ids`: `[318]`

#### 完整输出结构
```python
{
    'status': 'completed' | 'pending' | 'failed',
    'input': {
        'data': dict,  # 从上游节点提取的输入数据
        'source_node_id': str | None,  # 上游节点ID
        'source_type': 'upstream'
    },
    'operation': {
        'type': 'request_for_update',
        'description': str,  # 如 'Request for update (waiting for user submission)'
        'config': {
            'taskType': 'request_for_update',
            'updateJobTitleIds': list[int],
            'title': str,
            'timeout': int,
            'expires_at': str | None,  # ISO格式
        },
        'duration_ms': int | None,
    },
    'output': {
        'data': {
            'human_decision': str,  # 'updated' | 'not_updated' | 'timeout'
            'decision_reason': str,  # 更新信息或评论
            'update_info': str,  # 更新信息内容
            'file_ingestion_record_ids': list,  # 附件文件ID列表
        },
        'structured_output': {
            'task_type': 'request_for_update',
            'human_decision': str,  # 'updated' | 'not_updated' | 'timeout'
            'is_updated': bool,  # True表示已更新，False表示未更新或超时
            'update_info': str,  # 更新信息内容
            'file_ingestion_record_ids': list,  # 附件文件ID列表
            'rejection_reason': None,  # request_for_update类型不适用
            'reviewer_email': str | None,  # 提交更新的人员邮箱
            'reviewer_job_title': str | None,  # 提交更新的人员职位
            'decision_timestamp': str,  # 决策时间（ISO格式，UTC时区）
        }
    },
    'timeline': {
        'executed_at': str,  # 执行时间(ISO格式)
        'execution_order': int,  # 执行顺序(从1开始)
    }
}
```

#### 特点说明
- **更新信息**: `update_info` 字段包含用户提交的更新内容
- **附件支持**: `file_ingestion_record_ids` 包含上传的附件文件ID列表
- **状态判断**: `is_updated` 表示是否已提交更新（`decision == 'updated'`）
- **实际验证结果**（基于数据库）：
  - `human_decision`: `"updated"`
  - `update_info`: `"yy"`
  - `file_ingestion_record_ids`: `[318]`
  - `is_updated`: `True`

---

### 3.4 Assign类型（taskType = 'assign'）

#### 完整输入结构
```python
{
    # 从上游节点的 merged_context 提取的上下文数据
    # 基于代码验证：使用 get_context_from_upstream_output() 统一提取逻辑
    # 实际验证结果（基于代码）：从上游节点的 merged_context 提取
    # 包含上游节点的所有上下文信息
}
```

#### 配置数据来源
- **`assignConfig`**（从节点定义中获取）：
  - `assigneeEmail`：被分配人邮箱地址（必需）
  - `title`：分配任务标题
  - `description`：分配任务描述
- **`timeout`**：超时时间（秒，0表示无超时）
- **`timeoutAction`**：超时动作（当前仅支持 'fail'）

#### 用户提交数据（等待用户输入）
- **`decision_data`**（用户通过 Message Center 提交）：
  - `decision`：决策类型（'assigned' 或 'completed'）
  - `assign_info`：分配信息或处理结果内容（字符串）
  - `file_ingestion_record_ids`：附件文件ID列表
  - `comments`：评论（可选）

#### 完整输出结构
```python
{
    'status': 'completed' | 'pending' | 'failed',
    'input': {
        'data': dict,  # 从上游节点提取的输入数据
        'source_node_id': str | None,  # 上游节点ID
        'source_type': 'upstream'
    },
    'operation': {
        'type': 'assign',
        'description': str,  # 如 'Assign task to user@example.com (waiting for submission)'
        'config': {
            'taskType': 'assign',
            'assigneeEmail': str,
            'title': str,
            'timeout': int,
            'expires_at': str | None,  # ISO格式
            'steering_task_id': int,
        },
        'duration_ms': int | None,
    },
    'output': {
        'data': {
            'human_decision': str,  # 'assigned' | 'completed' | 'timeout'
            'decision_reason': str,  # 分配信息或处理结果
            'assign_info': str,  # 分配信息或处理结果内容
            'file_ingestion_record_ids': list,  # 附件文件ID列表
        },
        'structured_output': {
            'task_type': 'assign',
            'human_decision': str,  # 'assigned' | 'completed' | 'timeout'
            'is_assigned': bool,  # True表示已分配/已完成，False表示超时
            'assign_info': str,  # 分配信息或处理结果内容
            'file_ingestion_record_ids': list,  # 附件文件ID列表
            'assignee_email': str | None,  # 被分配人邮箱
            'assignee_job_title': str | None,  # 被分配人职位
            'decision_timestamp': str,  # 决策时间（ISO格式，UTC时区）
        }
    },
    'timeline': {
        'executed_at': str,  # 执行时间(ISO格式)
        'execution_order': int,  # 执行顺序(从1开始)
    }
}
```

#### 特点说明
- **分配信息**: `assign_info` 字段包含被分配人提交的处理结果
- **附件支持**: `file_ingestion_record_ids` 包含上传的附件文件ID列表
- **状态判断**: `is_assigned` 表示是否已分配/已完成（`decision == 'assigned'`）

---

### 代码位置
- `backend/app/services/workflow/nodes/human_in_the_loop_node.py`
  - Approval: `process_human_decision()` 方法（约第847-894行）
  - Inform: `_execute_inform()` 方法（约第994-1113行）
  - Request for Update: `process_human_decision()` 方法（约第756-801行）
  - Assign: `process_human_decision()` 方法（约第802-846行）

---

## 4. Question Classifier Node（问题分类节点）

### 输入位置
- 从上游节点的输出获取

### 完整输入结构
```python
{
    # 从上游节点（如 App Node）获取的完整输出
    # 基于代码验证：从 upstream_node_id 的 output 获取
    # 提取逻辑（backend/app/services/workflow/nodes/question_classifier_node.py:231-265）：
    #   1. 优先：从 output.data.conclusion_detail 提取（新结构 v3.0）
    #   2. 备选：从 output.conclusion_detail 提取（旧结构，向后兼容）
    #   3. 备选：从 output.merged_context 转换为字符串（旧结构 v2.0，向后兼容）
    #   4. 最后：将整个 output 转换为 JSON 字符串
}
```

### 输入数据来源
- **数据来源路径**（`backend/app/services/workflow/nodes/question_classifier_node.py:369-423`）：
  1. 通过 `_get_upstream_info()` 方法获取上游节点信息
  2. 从 `state['node_results'][upstream_node_id]['output']` 获取完整输出
  3. 通过 `_extract_input_text()` 方法提取输入文本
- **提取优先级**（`backend/app/services/workflow/nodes/question_classifier_node.py:231-250`）：
  1. **优先**：`output.data.conclusion_detail`（新结构 v3.0）
  2. **备选**：`output.conclusion_detail`（直接访问，fallback）
  3. **最后**：整个 `output` 转换为 JSON 字符串
- **实际验证结果**（基于日志）：
  - 日志：`Got upstream output from app-1`
  - 日志：`Input text length: 6974`
  - 从 app-1 的 `output.data.conclusion_detail` 提取了6974字符的文本（新结构）

### 配置数据来源
- **节点配置**（从节点定义中获取）：
  - `classifierConfig.classes`：分类类别列表
  - `classifierConfig.instruction`：额外的分类指导
  - `classifierConfig.llmConfig`：LLM配置（model, temperature）

### 字段说明
- **`conclusion_detail`**：从上游 App Node 的 `output.conclusion_detail` 提取
  - 这是 LLM 分类的主要输入文本
  - **实际验证结果**：6974字符的markdown内容
- **Fallback**: 如果 `conclusion_detail` 不存在，将整个 `output` 转换为 JSON 字符串
- **配置字段**：
  - `classes`：分类类别列表，每个类别包含 `id`、`name`、`description`
  - `instruction`：额外的分类指导文本
  - `llmConfig`：LLM配置（model、temperature）

### 输出位置
- `state['node_results'][classifier_node_id]`

### 完整输出结构
```python
{
    'status': 'completed' | 'failed',
    'input': {
        'data': {
            'conclusion_detail': str,  # 从上游App Node提取的conclusion_detail
        },
        'source_node_id': str,  # 上游节点ID(通常是App Node)
        'source_type': 'upstream'
    },
    'operation': {
        'type': 'classification',
        'description': str,  # 如 'Classify question type (selected: supplier information update)'
        'config': {
            'classes_count': int,
            'instruction': str,
        },
        'duration_ms': int | None,  # 执行耗时(毫秒)
    },
    'output': {
        'data': {
            # LangGraph路由字段（必需）
            'branch': str,  # 选中的class_id，用于LangGraph路由
            'class_id': str,  # 选中的分类ID（与branch相同）
            'class_name': str,  # 选中的分类名称（人类可读）
        },
        'structured_output': {
            'node_type': 'questionClassifier',
            'selected_class': str,  # 选中的分类ID
            'selected_class_name': str,  # 选中的分类名称
            'classification_reason': str,  # LLM分类的原因说明
            'available_classes': list,  # 所有可用的分类ID列表
        }
    },
    'timeline': {
        'executed_at': str,  # 执行时间(ISO格式)
        'execution_order': int,  # 执行顺序(从1开始)
    }
}
```

### 字段详细说明
- **`branch`**: LangGraph路由使用的分支标识，值为选中的 `class_id`
  - 用于确定工作流执行的下一个分支
  - 必须与routing_map中的key匹配
  - **实际验证结果**（基于数据库）：`"class_2"`
- **`class_id`**: 选中的分类ID（与 `branch` 相同）
  - **实际验证结果**（基于数据库）：`"class_2"`
- **`class_name`**: 选中的分类名称（人类可读）
  - **实际验证结果**（基于数据库）：`"supplier information update"`
- **`structured_output.selected_class`**: 选中的分类ID
  - **实际验证结果**（基于数据库）：`"class_2"`
- **`structured_output.classification_reason`**: LLM返回的分类原因说明
  - **实际验证结果**（基于数据库）：`"结论为暂不授标/拒绝当前入围，明确指出因材料缺失需发RFI并补充并验证所需文件后才能重新审核或考虑有条件准入，属于要求补充材料后重审的情形。"`
- **`structured_output.available_classes`**: 所有可用的分类ID列表
  - **实际验证结果**（基于数据库）：`["class_1", "class_2"]`

**注意**: 新结构 (v3.0) 不再包含 `upstream_context` 和 `merged_context` 字段。这些字段只在旧结构 (v2.0) 中使用（向后兼容）。

### 特点说明
- **单选择路由**: LLM只返回一个分类结果，工作流只执行一个分支
- **路由功能**: `branch` 字段用于LangGraph的条件路由
- **上下文传递**: `output.data` 包含分类结果，供下游节点使用

### 代码位置
- `backend/app/services/workflow/nodes/question_classifier_node.py` - `execute()` 方法（约第76-182行）

---

## 5. End Node（结束节点）

### 输入位置
- 从上游节点的输出获取（可选，End Node 不读取输入）

### 完整输入结构
```python
{
    # End Node 不读取输入数据
    # 基于代码验证：End Node 不执行任何输入处理
    # 只输出完成标记
}
```

### 输入数据说明
- **不读取输入**: End Node 不读取任何输入数据
- **代码逻辑**（`backend/app/services/workflow/workflow_execution_service.py:1926-1935`）：
  - End Node 只输出完成标记，不处理输入数据
  - 不访问 `state['input_data']` 或上游节点的输出

### 输出位置
- `state['node_results'][end_node_id]`

### 完整输出结构
```python
{
    'status': 'completed',  # 固定为 'completed'
    'input': {
        'data': dict,  # 从上游节点提取的输入数据(通常为空)
        'source_node_id': str | None,  # 上游节点ID
        'source_type': 'upstream'
    },
    'operation': {
        'type': 'end',
        'description': 'Workflow completed',
        'config': {}
    },
    'output': {
        'data': {
            'workflow_completed': True  # 固定为 True，标识工作流执行完成
        },
        'structured_output': {
            'node_type': 'end',
            'workflow_completed': bool,
            'total_nodes': int,  # 总节点数
            'execution_time': str,  # 执行总时长(格式: HH:MM:SS)
        }
    },
    'timeline': {
        'executed_at': str,  # 执行时间(ISO格式)
        'execution_order': int,  # 执行顺序(从1开始)
    }
}
```

### 字段说明
- **`status`**: 固定为 `'completed'`，表示节点执行成功
- **`output.workflow_completed`**: 固定为 `True`，标识整个工作流已完成
- **不包含业务数据**: End Node只作为完成标记，不包含任何业务数据
- **实际验证结果**（基于数据库）：`{"workflow_completed": true}`

### 重要说明
- **完成标记**: End Node仅用于标识工作流执行完成
- **业务数据位置**: 实际的业务结果应从前一个业务节点（如App Node、Human in the Loop Node等）的输出中获取
- **不读取输入**: End Node不读取任何输入数据，只输出完成标记

### 代码位置
- `backend/app/services/workflow/workflow_execution_service.py` - `_create_end_node()` 方法（约第1926-1935行）

---

## 数据流转示例

### 示例1: Start → App → Question Classifier → Human (Request for Update) → End（基于实际执行）

**实际执行路径**（基于日志验证）：
```
start-2 → app-1 → questionClassifier-3 → humanInTheLoop-6 (request_for_update) → end-7
```

**数据流转**（基于数据库验证）：
```
Start Node (start-2)
  输入: state['input_data'] = {}
  输出: {}（空字典）
  
App Node (app-1)
  输入: 从 start-2 获取 {}（空字典）
  配置: 从 app_node_configs 获取 {dagRunId: 'manual__2026-01-22T09:29:20.874721+00:00', inputMode: 'file', requirement: '', appNodeId: 105}
  输出: {
    data: {
      conclusion_detail: "..." (6974字符),
      dag_run_id: "manual__2026-01-22T09:29:20.874721+00:00"
    },
    structured_output: {
      node_type: 'app',
      app_node_id: 105,
      ...
    }
    # 注意：新结构 (v3.0) 不再包含 upstream_context 和 merged_context
  }
  
Question Classifier Node (questionClassifier-3)
  输入: 从 app-1 的 output.data.conclusion_detail 提取（6974字符，新结构 v3.0）
  配置: 从节点定义获取 {classes: [...], instruction: '...', llmConfig: {...}}
  LLM分类: class_id='class_2', reason='...'
  输出: {
    data: {
      branch: 'class_2',
      class_id: 'class_2',
      class_name: 'supplier information update'
    },
    structured_output: {
      node_type: 'questionClassifier',
      selected_class: 'class_2',
      selected_class_name: 'supplier information update',
      classification_reason: '...',
      available_classes: ['class_1', 'class_2']
    }
    # 注意：新结构 (v3.0) 不再包含 upstream_context 和 merged_context
  }
  
Human In The Loop Node (humanInTheLoop-6, request_for_update)
  输入: 从 questionClassifier-3 的 output.data 提取（新结构 v3.0）
  配置: 从节点定义获取 {taskType: 'request_for_update', updateConfig: {...}, timeout: 43200}
  用户提交: {decision: 'updated', update_info: 'yy', file_ingestion_record_ids: [318]}
  输出: {
    data: {
      human_decision: 'updated',
      decision_reason: 'yy',
      update_info: 'yy',
      file_ingestion_record_ids: [318]
    },
    structured_output: {
      task_type: 'request_for_update',
      is_updated: True,
      update_info: 'yy',
      file_ingestion_record_ids: [318],
      ...
    }
    # 注意：新结构 (v3.0) 不再包含 upstream_context 和 merged_context
  }
  
End Node (end-7)
  输入: 不读取输入
  输出: {workflow_completed: True}
```

**关键发现**（基于实际验证）：
1. Start Node 输出为空字典 `{}`（已验证）
2. App Node 使用 `app_node_configs` 而非输入数据（已验证）
3. Question Classifier Node 从 `conclusion_detail` 提取输入（已验证）
4. Human In The Loop Node 从 `output.data` 提取输入（新结构 v3.0，已验证）
5. 上下文通过 `output.data` 传递，形成完整的上下文链（新结构 v3.0）

**注意**: 上述示例中的 `upstream_context` 和 `merged_context` 是旧结构 (v2.0) 的字段。新结构 (v3.0) 不再包含这些字段。

---

### 示例2: Start → App → Question Classifier → Human (Approval) → End

```
Start Node
  输入: state['input_data'] = {}
  输出: {}（空字典）
  
App Node
  输入: 从 start-2 获取 {}（空字典）
  配置: 从 app_node_configs 获取 {dagRunId, inputMode, requirement, appNodeId}
  输出: {
    data: {
      conclusion_detail: '...',
      dag_run_id: '...'
    },
    structured_output: {
      node_type: 'app',
      app_node_id: 105,
      ...
    }
    # 注意：新结构 (v3.0) 不再包含 upstream_context 和 merged_context
  }
  
Question Classifier Node
  输入: 从 app-1 的 output.data.conclusion_detail 提取（新结构 v3.0）
  配置: 从节点定义获取 {classes, instruction, llmConfig}
  输出: {
    data: {
      branch: 'class_1',
      class_id: 'class_1',
      class_name: '...'
    },
    structured_output: {
      node_type: 'questionClassifier',
      selected_class: 'class_1',
      ...
    }
    # 注意：新结构 (v3.0) 不再包含 upstream_context 和 merged_context
  }
  
Human in the Loop Node (Approval)
  输入: 从 questionClassifier-3 的 output.data 提取（新结构 v3.0）
  配置: 从节点定义获取 {taskType: 'approval', approvalConfig: {...}, timeout: 43200}
  用户提交: {decision: 'approved', comments: '...', file_ingestion_record_ids: [...]}
  输出: {
    data: {
      human_decision: 'approved',
      decision_reason: '...'
    },
    structured_output: {
      task_type: 'approval',
      is_approved: True,
      ...
    }
    # 注意：新结构 (v3.0) 不再包含 upstream_context, merged_context, _metadata
  }
  
End Node
  输入: 不读取输入
  输出: {workflow_completed: True}
```

---

### 示例3: Start → App → Human (Inform) → End

```
Start Node
  输入: state['input_data'] = {}
  输出: {}（空字典）
  
App Node
  输入: 从 start-2 获取 {}（空字典）
  配置: 从 app_node_configs 获取 {dagRunId, inputMode, requirement, appNodeId}
  输出: {
    data: {
      conclusion_detail: '...',
      dag_run_id: '...'
    },
    structured_output: {
      node_type: 'app',
      ...
    }
    # 注意：新结构 (v3.0) 不再包含 upstream_context, merged_context, _metadata
  }
  
Human in the Loop Node (Inform)
  输入: 从 app-1 的 output.data 提取（新结构 v3.0）
  配置: 从节点定义获取 {taskType: 'inform', informConfig: {...}}
  输出: {
    data: {
      human_decision: 'informed',
      decision_reason: '...'
    },
    structured_output: {
      task_type: 'inform',
      notification_sent: True,
      ...
    }
    # 注意：新结构 (v3.0) 不再包含 upstream_context 和 merged_context
  }
  # 注意：inform 类型立即完成，不等待用户输入
  
End Node
  输入: 不读取输入
  输出: {workflow_completed: True}
```

---

## 通用字段说明

### status字段
- **`completed`**: 节点执行成功完成
- **`pending`**: 节点等待中（如Human in the Loop Node等待审批）
- **`failed`**: 节点执行失败

### 上下文字段
**注意**: 新结构 (v3.0) 不再包含 `upstream_context` 和 `merged_context` 字段。这些字段只在旧结构 (v2.0) 中使用（向后兼容）。

### 元数据字段
- **`_metadata`**: 执行元数据，包含节点信息、执行时间、上下文链等
  - `schema_version`: 输出结构版本
  - `node_id`: 节点ID
  - `node_type`: 节点类型
  - `executed_at`: 执行时间（ISO格式，UTC时区）
  - `upstream_node_id`: 上游节点ID
  - `context_chain`: 执行历史链

---

## 注意事项

1. **数据存储位置**: 所有节点的输出都存储在 `state['node_results'][node_id]` 中
2. **数据库持久化**: 节点结果会持久化到 `WorkflowExecutionTask.node_results` 字段
3. **数据分离**: 每个节点只存储自己产生的数据（新结构 v3.0 不包含 `upstream_context` 和 `merged_context`）
4. **上游数据获取**: 如需上游数据，通过查询接口从其他节点获取（`GET /api/v1/workbench/workflow-tasks/{task_id}/nodes/{node_id}`）
5. **时间线追溯**: 通过时间线接口获取完整执行流程（`GET /api/v1/workbench/workflow-tasks/{task_id}/timeline`）
6. **End Node限制**: End Node不包含业务数据，只作为完成标记
7. **Human in the Loop Node状态**: 根据taskType不同，状态不同
   - `approval`、`request_for_update`、`assign`：可能处于 `pending` 状态等待用户决策
   - `inform`：固定为 `completed`（立即完成）
8. **统一结构**: 所有节点统一使用新结构 (v3.0)，不再支持旧结构

---

## 相关文档

- `cursordocs/workflow-start-end-node-output-structure.md` - Start和End节点详细说明
- `cursordocs/workflow-nodes-input-output-analysis.md` - 节点输入输出分析
- `cursordocs/question-classifier-node-clarification.md` - Question Classifier节点说明
- `cursordocs/human-in-the-loop-assign-task-type-implementation-plan.md` - Human in the Loop Node Assign类型实现
