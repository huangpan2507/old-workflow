# Policy Approval Workflow Instance Implementation

## 概述

按照方案二（新增审批流实例表）完成了政策审批流模版支持多节点审批的需求。新的设计提供了更灵活和可扩展的审批流管理。

## 新的数据模型

### 1. PolicyApprovalInstance（审批流实例表）
记录政策使用的审批流实例，包含：
- `policy_id`: 关联的政策ID
- `template_id`: 使用的审批流模版ID
- `template_name`: 模版名称（冗余存储，便于查询）
- `workflow_type`: 工作流类型
- `status`: 实例状态（active, completed, cancelled, failed）
- `current_step`: 当前审批步骤
- `total_steps`: 总步骤数
- `completed_steps`: 已完成步骤数
- `approved_steps`: 已批准步骤数
- `rejected_steps`: 已拒绝步骤数
- `current_approver_id`: 当前审批人ID

### 2. PolicyApprovalStep（审批步骤表）
记录每个审批步骤的实例，包含：
- `instance_id`: 关联的审批流实例ID
- `node_definition_id`: 关联的节点定义ID
- `step_order`: 步骤顺序
- `step_name`: 步骤名称
- `step_type`: 步骤类型
- `is_required`: 是否必需
- `approval_logic`: 审批逻辑（any, all, majority）
- `min_approvers`: 最少审批人数
- `status`: 步骤状态
- `total_approvers`: 总审批人数
- `approved_count`: 已批准人数
- `rejected_count`: 已拒绝人数

### 3. PolicyApprovalFlow（审批流执行记录表）
记录具体的审批执行记录，包含：
- `instance_id`: 关联的审批流实例ID
- `step_id`: 关联的审批步骤ID
- `approver_id`: 审批人ID
- `approver_department_id`: 审批人部门ID
- `status`: 审批状态
- `decision`: 审批决定
- `comments`: 审批意见

## 新的服务方法

### PolicyApprovalService 更新

1. **create_approval_instance_for_policy()**: 为政策创建审批流实例
2. **process_approval_decision()**: 处理审批决定，支持多节点审批逻辑
3. **get_policy_approval_instance()**: 获取政策的审批流实例
4. **get_approval_template_for_policy()**: 获取政策使用的审批流模版
5. **_check_step_completion()**: 检查步骤是否完成
6. **_get_next_step()**: 获取下一个步骤
7. **_get_next_approver()**: 获取下一个审批人

## 新的 API 接口

### 1. 政策提交审批
```
POST /policies/{policy_id}/submit
```
返回：
- `approval_instance_id`: 审批流实例ID
- `template_name`: 使用的模版名称
- `total_steps`: 总步骤数
- `current_approver_id`: 当前审批人ID

### 2. 获取审批历史
```
GET /policies/{policy_id}/approval-history
```
返回：
- `template_name`: 使用的模版名称
- `workflow_status`: 工作流状态
- `current_step`: 当前步骤
- `total_steps`: 总步骤数
- `steps`: 步骤列表（包含每个步骤的审批人信息）

### 3. 获取审批流模版
```
GET /policies/{policy_id}/approval-template
```
返回：
- `template`: 完整的模版信息
- `nodes`: 模版节点定义列表

## 多节点审批支持

### 审批逻辑支持
1. **any**: 任意一个审批人批准即可
2. **all**: 所有审批人都必须批准
3. **majority**: 多数审批人批准即可

### 节点类型支持
1. **individual_users**: 指定具体用户
2. **roles**: 基于角色的审批
3. **function_role_head**: 职能负责人审批

## 数据库迁移

创建了迁移文件 `h1234567890c_add_policy_approval_instance_tables.py`，包含：
- 创建 `policy_approval_instances` 表
- 创建 `policy_approval_steps` 表
- 重构 `policy_approval_flows` 表
- 添加相应的索引和外键约束

## 使用示例

### 创建审批流实例
```python
approval_service = PolicyApprovalService(session)
instance = approval_service.create_approval_instance_for_policy(policy_id)
```

### 处理审批决定
```python
result = approval_service.process_approval_decision(
    policy_id=policy_id,
    approver_id=str(user_id),
    decision="approved",
    comments="审批通过"
)
```

### 获取审批模版
```python
template = approval_service.get_approval_template_for_policy(policy_id)
```

## 优势

1. **可追溯性**: 可以准确知道每个政策使用的审批流模版
2. **多节点支持**: 支持复杂的多步骤审批流程
3. **灵活配置**: 支持不同的审批逻辑和节点类型
4. **状态跟踪**: 详细跟踪每个步骤和审批人的状态
5. **扩展性**: 易于添加新的审批逻辑和节点类型

## 兼容性

- 保持了现有 API 接口的兼容性
- 新的数据结构不影响现有的审批流程
- 支持从旧数据结构迁移到新结构

这个实现完全满足了你的需求：支持多节点审批、可以追溯到使用的审批流模版、支持复杂的审批逻辑，并且为未来的扩展提供了良好的基础。
