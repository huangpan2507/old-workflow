# Policy Approval Workflow Configuration Implementation Summary

## 功能概述

成功实现了政策审批流程的可配置功能，支持管理员定义多级审批节点。当前实现了两级审批流程：
1. **第一级审批**：政策提交人所在部门的 function head
2. **第二级审批**：指定的政策库管理员（policy_admin 角色）

## 实现的功能

### 1. 数据库模型
- **PolicyApprovalTemplate**: 审批流程模板配置表
- **PolicyApprovalNodeDefinition**: 审批节点定义表
- 新增了政策库管理员角色（policy_admin）

### 2. 审批流程服务
- **PolicyApprovalService**: 核心审批流程服务
  - `get_working_approval_template()`: 获取工作审批模板
  - `create_approval_nodes_for_policy()`: 根据模板创建审批节点
  - `process_approval_decision()`: 处理审批决策和流转
  - `get_policy_approval_history()`: 获取审批历史

### 3. API 接口
- **审批模板管理 API** (`/api/v1/policy-approval-templates/`)
  - 创建、查询、更新、删除审批模板
  - 管理审批节点定义
  - 设置默认模板

- **更新的政策 API** (`/api/v1/policies/`)
  - `submit_policy_for_approval()`: 提交时自动创建审批流程
  - `approve_policy()`: 处理审批决策，自动流转
  - `get_approval_history()`: 显示完整审批链

### 4. 初始数据
- 创建了"Policy Library Administrator"角色
- 创建了默认的两级审批流程模板
- 配置了两个审批节点：
  - 节点1：部门负责人审批（department_head）
  - 节点2：政策库管理员审批（role_based）

## 审批流程逻辑

```
1. 用户提交政策
   ↓
2. 系统查找默认审批流程模板
   ↓
3. 创建所有审批节点（状态：pending）
   ↓
4. 第一个节点状态设为 active，设置当前审批人
   ↓
5. 当前审批人审批后：
   - 如果批准：流转到下一节点
   - 如果拒绝：整个流程结束，政策状态=rejected
   ↓
6. 所有节点都批准后：政策状态=approved
```

## 审批人查找逻辑

### 部门负责人查找
- 通过 `User.function_id` = 提交人的部门ID
- `User.employee_level` = "function_head"

### 政策库管理员查找
- 通过角色关联查询
- 查找拥有 "policy_admin" 角色的用户

## 数据库迁移

- 创建了 Alembic 迁移脚本：`b72017308d83_add_policy_approval_workflow_.py`
- 成功应用了数据库迁移
- 创建了相关索引以优化查询性能

## 文件结构

```
backend/app/
├── models.py                                    # 新增审批流程模型
├── services/
│   └── policy_approval_service.py              # 审批流程服务
├── api/v1/
│   ├── policy_approval_templates.py            # 审批模板管理API
│   └── policies.py                             # 更新的政策API
└── core/database/default_data/
    └── policy_approval_templates.py            # 初始数据脚本
```

## 使用方式

### 1. 提交政策审批
```bash
POST /api/v1/policies/{policy_id}/submit
```

### 2. 审批政策
```bash
POST /api/v1/policies/{policy_id}/approve
{
  "decision": "approve",  # 或 "reject"
  "comments": "审批意见"
}
```

### 3. 查看审批历史
```bash
GET /api/v1/policies/{policy_id}/approval-history
```

### 4. 管理审批模板（仅超级管理员）
```bash
GET /api/v1/policy-approval-templates/
POST /api/v1/policy-approval-templates/
PUT /api/v1/policy-approval-templates/{id}
DELETE /api/v1/policy-approval-templates/{id}
```

## 扩展性

该实现为未来的扩展预留了空间：
- 支持更多审批节点类型（specific_user, role_based）
- 支持条件审批（conditions 字段）
- 支持委托审批（can_delegate 字段）
- 支持自动审批（auto_approve 字段）

## 测试建议

1. 创建政策并提交审批
2. 验证第一级审批人（部门负责人）能收到审批任务
3. 验证审批后自动流转到第二级审批人
4. 验证政策库管理员能完成最终审批
5. 测试拒绝流程
6. 验证审批历史记录

## 注意事项

- 只有超级管理员可以管理审批模板
- 审批流程一旦开始，不能修改
- 所有审批操作都有完整的审计日志
- 支持政策修订时的原政策替代逻辑
