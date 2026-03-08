# 数据库迁移完成说明

## 迁移目标

删除旧的 `policy_approval_*` 表，使用新的通用 `approval_workflow_*` 表。

## 迁移文件

**Revision:** `j1234567890e_refactor_approval_workflow_to_generic.py`

## 迁移步骤

### 1. 删除的旧表

在 Alembic migration 的 `upgrade()` 函数中，按依赖关系依次删除：

```python
# Drop policy_approval_flows (depends on steps and instances)
op.drop_table('policy_approval_flows')

# Drop policy_approval_steps (depends on instances and node_definitions)
op.drop_table('policy_approval_steps')

# Drop policy_approval_instances (depends on templates and policies)
op.drop_table('policy_approval_instances')

# Drop policy_approval_node_roles (depends on node_definitions)
op.drop_table('policy_approval_node_roles')

# Drop policy_approval_node_users (depends on node_definitions)
op.drop_table('policy_approval_node_users')

# Drop policy_approval_node_definitions (depends on templates)
op.drop_table('policy_approval_node_definitions')

# Drop policy_approval_templates
op.drop_table('policy_approval_templates')
```

### 2. 创建的新表

在同一个 revision 中创建：

- `approval_workflow_templates` - 通用审批流模板
- `approval_workflow_node_definitions` - 节点定义
- `approval_workflow_node_users` - 节点用户关联
- `approval_workflow_node_roles` - 节点角色关联
- `approval_workflow_instances` - 通用审批流实例
- `approval_workflow_steps` - 审批步骤
- `approval_workflow_flows` - 审批流程

### 3. 数据迁移

**没有进行数据迁移**

原因：
- 旧数据是本地开发测试数据
- 用户明确表示不需要保留旧数据
- 新系统使用全新的默认模板

### 4. Rollback 支持

在 `downgrade()` 函数中：
- 删除所有新的 `approval_workflow_*` 表
- 重新创建旧的 `policy_approval_*` 表（简化版本）

## 当前状态

- ✅ 当前数据库版本: `j1234567890e`
- ✅ 旧表数量: 0（已全部删除）
- ✅ 新表数量: 7（全部创建成功）
- ✅ 默认模板: 已创建（Standard Policy Approval Workflow）

## 删除的相关文件

1. **旧服务代码**
   - `backend/app/services/policy_approval_service.py`

2. **旧 API 端点**
   - `backend/app/api/v1/policy_approval_templates.py`
   - `backend/app/api/v1/approval_nodes.py`

3. **旧模型定义**（从 `models.py` 中删除）
   - `PolicyApprovalTemplate`
   - `PolicyApprovalNodeDefinition`
   - `PolicyApprovalNodeUser`
   - `PolicyApprovalNodeRole`
   - `PolicyApprovalInstance`
   - `PolicyApprovalStep`
   - `PolicyApprovalFlow`

4. **旧初始化文件**
   - `backend/app/core/database/default_data/policy_approval_templates.py`

## 保留的内容

1. **新模型和 Schema**
   - 通用 `ApprovalWorkflow*` 模型
   - 对应的 Pydantic Schema

2. **新服务层**
   - `ApprovalWorkflowService` - 通用服务
   - `PolicyApprovalAdapter` - Policy 适配器

3. **新 API 端点**
   - `policies.py` - 使用新的适配器
   - `policy_revisions.py` - 使用新的适配器

4. **新模板初始化**
   - `backend/app/core/database/default_data/approval_workflow_templates.py`

## 验证

数据库迁移已通过 Alembic 完成，所有旧表已删除，新表已创建，系统正常工作。

