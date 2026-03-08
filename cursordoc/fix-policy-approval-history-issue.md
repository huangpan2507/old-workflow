# Fix Policy Approval History Issue

## 问题描述

新建的 policy 在 Approval History 中没有显示审批人数据，显示 "No approval history available"。

## 根本原因

1. **条件检查错误**：在 `backend/app/api/v1/policies.py` 第 123 行，创建审批流程的条件包含了 `policy.current_approver_id` 的检查，但此时该字段为 None，导致审批流程不会被创建。

2. **Node Type 不匹配**：初始化脚本使用的 `node_type` 是 `"department_head"`，但服务中的查找逻辑检查的是 `"function_role_head"`，导致无法找到审批人。

3. **初始化脚本使用了不存在的字段**：使用了 `approver_type`、`approver_user_id`、`approver_role_code` 等字段，但这些字段在 `PolicyApprovalNodeDefinition` 模型中不存在。

## 解决方案

### 1. 修复 Policy 创建逻辑

**文件**：`backend/app/api/v1/policies.py`

**修改**：移除对 `policy.current_approver_id` 的检查

```python
# 修改前
if approval_status != "draft" and policy.current_approver_id:

# 修改后
if approval_status != "draft":
```

### 2. 修复初始化脚本

**文件**：`backend/app/core/database/default_data/policy_approval_templates.py`

**修改**：
- 将第一个节点的 `node_type` 从 `"department_head"` 改为 `"function_role_head"`
- 将第二个节点的 `node_type` 从 `"role_based"` 改为 `"roles"`
- 移除不存在的字段
- 为 roles 类型的节点添加 `PolicyApprovalNodeRole` 关联

### 3. 更新现有数据库

对于现有数据库中已存在的审批节点，需要手动更新其 `node_type`：

```sql
-- 更新 Department Head 节点的类型
UPDATE policy_approval_node_definitions 
SET node_type = 'function_role_head'
WHERE node_name = 'Department Head Approval' 
  AND node_type = 'department_head';

-- 更新 Policy Library Administrator 节点的类型
UPDATE policy_approval_node_definitions 
SET node_type = 'roles'
WHERE node_name = 'Policy Library Administrator Approval' 
  AND node_type = 'role_based';
```

## 测试步骤

1. 确保数据库中有默认的审批模板
2. 创建一个新的 policy（非 draft 状态）
3. 检查 policy 的 approval_status 是否为 "pending"
4. 查看 Approval History 是否显示审批人
5. 确认审批流程按照正确的逻辑查找审批人

## 查找逻辑验证

审批人查找逻辑（`function_role_head` 类型）：
1. 获取 policy 创建者
2. 找到创建者所在的 Business Unit
3. 在该 BU 下查找 Function，匹配 `function_role_id == policy.responsible_function_role_id`
4. 返回该 Function 的 `function_head_id` 对应的用户

## 相关文件

- `backend/app/api/v1/policies.py` - Policy 创建逻辑
- `backend/app/services/policy_approval_service.py` - 审批流程服务
- `backend/app/core/database/default_data/policy_approval_templates.py` - 审批模板初始化

