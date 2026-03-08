# Function Roles 审批节点类型移除总结

## 概述
根据用户要求，完全移除了 Function Roles 审批节点类型，简化审批流程配置。

## 移除内容

### 1. 前端组件
- **ApprovalNodeConfig.tsx**: 移除了 Function Roles 配置界面
  - 移除了 `function_roles` 字段
  - 移除了 Function Roles 相关的状态管理函数
  - 移除了 Function Roles 配置表单

- **ApprovalWorkflowConfig.tsx**: 移除了 Function Roles 显示逻辑
  - 移除了 `function_roles` 字段
  - 移除了 Function Roles 标签显示

### 2. 后端模型
- **models.py**: 完全移除了 Function Roles 审批节点相关模型
  - 移除了 `PolicyApprovalNodeFunctionRole` SQLModel 表定义
  - 移除了所有相关的 Pydantic Schema:
    - `PolicyApprovalNodeFunctionRoleBase`
    - `PolicyApprovalNodeFunctionRoleCreate`
    - `PolicyApprovalNodeFunctionRoleUpdate`
    - `PolicyApprovalNodeFunctionRoleRead`
    - `PolicyApprovalNodeFunctionRoleUpdateCreate`
  - 从 `PolicyApprovalNodeDefinition` 相关 Schema 中移除了 `function_roles` 字段

### 3. API 接口
- **approval_nodes.py**: 移除了所有 Function Roles 管理端点
  - 移除了 `add_node_function_role` 端点
  - 移除了 `get_node_function_roles` 端点
  - 移除了 `update_node_function_role` 端点
  - 移除了 `remove_node_function_role` 端点
  - 移除了 `get_function_roles` 端点

- **policy_approval_templates.py**: 移除了 Function Roles 相关逻辑
  - 移除了节点创建时的 Function Roles 分配逻辑
  - 移除了节点查询时的 Function Roles 加载逻辑
  - 移除了节点更新时的 Function Roles 处理逻辑

### 4. 服务层
- **policy_approval_service.py**: 移除了 Function Roles 审批逻辑
  - 移除了 `_find_approver_by_function_role_head` 方法
  - 移除了 `function_role_head` 节点类型的处理逻辑

### 5. 数据库
- **表删除**: 直接删除了 `policy_approval_node_function_roles` 表
- **数据迁移**: 将现有的 `function_role_head` 类型节点转换为 `individual_users` 类型
- **迁移文件**: 更新了相关迁移文件的注释

### 6. 默认数据
- **policy_approval_templates.py**: 移除了 Function Roles 分配逻辑
  - 移除了 Function Roles 审批节点的创建逻辑
  - 简化了模板创建流程

## 影响范围

### 审批节点类型
现在支持的审批节点类型：
- `individual_users`: 个人用户审批
- `roles`: 角色审批
- `function_role_head`: 职能负责人审批（保留，但不再使用 Function Roles 表）

### 数据一致性
- 所有现有的 `function_role_head` 类型节点已转换为 `individual_users` 类型
- 相关的 Function Roles 分配数据已清理
- 数据库表结构已更新

## 验证结果

### 后端服务
- ✅ 后端服务正常启动
- ✅ 所有 API 端点正常工作
- ✅ 数据库连接正常
- ✅ 没有导入错误

### 前端界面
- ✅ 审批节点配置界面正常
- ✅ 不再显示 Function Roles 选项
- ✅ 审批流程显示正常

## 总结
Function Roles 审批节点类型已完全移除，系统现在只支持个人用户和角色两种审批方式，简化了审批流程的配置和管理。所有相关的前端界面、后端逻辑、数据库结构和 API 接口都已相应更新。
