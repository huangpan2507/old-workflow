# Job Title审批支持实现文档

## 概述

本次实现为审批工作流系统添加了基于Job Title（职务）的审批人和确认人分配功能。支持通过Job Title来分配审批节点和人工确认节点的操作人员，即使该用户是普通员工，只要拥有指定的Job Title，都可以进行审批或确认操作。

## 实现内容

### 1. 数据模型扩展

#### 1.1 ApprovalWorkflowNodeDefinition
- **扩展内容**：在`node_type`字段中添加了`"job_titles"`选项
- **位置**：`backend/app/models.py` 第2241-2243行

#### 1.2 ApprovalWorkflowNodeJobTitle（新建）
- **表名**：`approval_workflow_node_job_titles`
- **功能**：存储审批节点与Job Title的关联关系
- **主要字段**：
  - `node_id`: 审批节点ID
  - `job_title_id`: Job Title ID
  - `is_primary`: 是否主审批人
  - `priority`: 优先级
  - `scope_type`: 范围类型（global, group, business_unit, function）
  - `scope_config`: 范围配置（JSON格式）
  - `effective_from/until`: 生效时间范围
  - `is_active`: 是否激活
- **位置**：`backend/app/models.py` 第2351-2389行

### 2. 服务层扩展

#### 2.1 ApprovalWorkflowService
- **新增方法**：`_find_approvers_by_job_titles()`
  - 根据节点配置的Job Title查找所有拥有该Job Title的活跃用户
  - 支持scope范围过滤
- **扩展方法**：`_find_approvers_for_node()`
  - 添加了对`node_type == "job_titles"`的处理分支
- **新增方法**：`_check_user_scope_for_job_title()`
  - 检查用户是否符合Job Title分配的scope配置
- **位置**：`backend/app/services/approval_workflow_service.py`

#### 2.2 HumanInTheLoopService（新建）
- **功能**：处理人工确认节点的审批人查找
- **主要方法**：
  - `get_confirmation_assignee()`: 获取确认人列表
  - `_find_approvers_by_job_titles()`: 根据Job Title查找确认人
  - `_check_user_scope()`: 检查用户scope
- **位置**：`backend/app/services/human_in_the_loop_service.py`

### 3. API端点扩展

#### 3.1 GET /api/v1/approval-workflow-templates/{template_id}/nodes
- **扩展内容**：返回结果中新增`job_titles`字段
- **返回格式**：
```json
{
  "id": 1,
  "node_type": "job_titles",
  "job_titles": [
    {
      "id": 1,
      "job_title_id": 5,
      "name": "采购经理",
      "code": "PURCHASE_MANAGER",
      "level": 3,
      "is_primary": true,
      "priority": 1,
      "scope_type": "business_unit",
      "scope_config": "{\"business_unit_ids\": [1, 2]}"
    }
  ]
}
```

#### 3.2 POST /api/v1/approval-workflow-templates/{template_id}/nodes
- **扩展内容**：支持在创建节点时配置Job Title
- **请求格式**：
```json
{
  "node_name": "采购审批",
  "node_type": "job_titles",
  "job_titles": [
    {
      "job_title_id": 5,
      "is_primary": true,
      "priority": 1,
      "scope_type": "business_unit",
      "scope_config": "{\"business_unit_ids\": [1, 2]}"
    }
  ]
}
```

#### 3.3 PUT /api/v1/approval-workflow-templates/{template_id}/nodes/{node_id}
- **扩展内容**：支持更新节点的Job Title配置
- **请求格式**：同POST，包含`job_titles`字段

### 4. 数据库迁移

#### 4.1 迁移文件
- **文件**：`backend/app/alembic/versions/k20250131_add_job_title_approval_support.py`
- **功能**：
  - 创建`approval_workflow_node_job_titles`表
  - 创建相关索引
- **执行命令**：
```bash
cd /home/huangpan/foundation/backend
alembic upgrade head
```

## 使用示例

### 1. 创建基于Job Title的审批节点

```python
# 通过API创建审批节点
POST /api/v1/approval-workflow-templates/1/nodes
{
  "node_name": "Functional Director审批",
  "node_type": "job_titles",
  "step_order": 1,
  "is_required": true,
  "approval_logic": "any",
  "job_titles": [
    {
      "job_title_id": 10,  # Functional Director的Job Title ID
      "is_primary": true,
      "priority": 1,
      "scope_type": "business_unit",
      "scope_config": "{\"business_unit_ids\": [1, 2, 3]}"
    }
  ]
}
```

### 2. 在审批服务中使用

```python
from app.services.approval_workflow_service import ApprovalWorkflowService

service = ApprovalWorkflowService(session)

# 创建审批实例时，系统会自动根据节点类型查找审批人
instance = service.create_approval_instance(
    resource_type='policy_revision',
    resource_id=123,
    template_id=1
)

# 如果节点类型是job_titles，系统会：
# 1. 查找节点配置的所有Job Title
# 2. 查找所有拥有这些Job Title的活跃用户
# 3. 根据scope配置过滤用户
# 4. 创建审批流程
```

### 3. 在人工确认节点中使用

```python
from app.services.human_in_the_loop_service import HumanInTheLoopService

service = HumanInTheLoopService(session)

# 获取确认人
approvers = service.get_confirmation_assignee(
    workflow_instance=workflow_instance,
    confirmation_type='supplier_selection',
    node_config={
        'node_type': 'job_titles',
        'node_id': node_id,  # 或者直接提供job_title_ids
        'job_title_ids': [10, 11],  # 可选：直接指定Job Title IDs
        'scope_type': 'business_unit',
        'scope_config': '{"business_unit_ids": [1, 2]}'
    }
)
```

## 配置说明

### Scope配置格式

#### business_unit范围
```json
{
  "business_unit_ids": [1, 2, 3]
}
```

#### function范围
```json
{
  "function_ids": [10, 20, 30]
}
```

#### group范围
```json
{
  "group_ids": [1, 2]
}
```

#### global范围
- 不需要scope_config，或设置为null
- 所有拥有该Job Title的用户都可以审批

## 注意事项

1. **JobTitle表无需修改**：现有的`job_titles`表结构已足够，无需新增字段
2. **用户关联**：用户通过`users.job_title_id`字段关联到Job Title
3. **权限控制**：即使普通员工（employee_level="staff"），只要拥有指定的Job Title，都可以进行审批
4. **Scope过滤**：如果配置了scope，只有符合scope条件的用户才会被选中
5. **优先级**：支持设置`is_primary`和`priority`，用于多审批人场景

## 测试建议

1. **创建测试数据**：
   - 在`job_titles`表中创建测试Job Title
   - 在`users`表中创建测试用户，关联Job Title

2. **测试场景**：
   - 创建基于Job Title的审批节点
   - 验证审批人查找逻辑
   - 测试scope过滤功能
   - 验证人工确认节点的审批人分配

3. **API测试**：
   ```bash
   # 创建节点
   curl -X POST http://localhost:8001/api/v1/approval-workflow-templates/1/nodes \
     -H "Authorization: Bearer <token>" \
     -H "Content-Type: application/json" \
     -d '{
       "node_name": "测试Job Title审批",
       "node_type": "job_titles",
       "job_titles": [{"job_title_id": 1}]
     }'
   ```

## 相关文件

- `backend/app/models.py` - 数据模型定义
- `backend/app/services/approval_workflow_service.py` - 审批服务
- `backend/app/services/human_in_the_loop_service.py` - 人工确认服务
- `backend/app/api/v1/approval_workflow_templates.py` - API端点
- `backend/app/alembic/versions/k20250131_add_job_title_approval_support.py` - 数据库迁移

## 后续工作

1. 执行数据库迁移：`alembic upgrade head`
2. 测试API端点功能
3. 在前端UI中添加Job Title选择组件
4. 更新API文档










