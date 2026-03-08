# 前端职能驱动审批模式更新

## 更新内容

已成功将前端审批节点配置页面中的"Organizational Structure"替换为"Function Role"模式。

### 主要修改

1. **节点类型选项更新**
   - 将下拉菜单中的"Organizational Structure"替换为"Function Role"
   - 选项值从`organizational_structure`改为`function_roles`

2. **接口定义扩展**
   - 添加了`FunctionRole`接口定义
   - 添加了`ApprovalNodeFunctionRole`接口定义
   - 更新了`ApprovalNodeConfig`接口，添加`function_roles`字段

3. **状态管理更新**
   - 添加了`functionRoles`状态用于存储可用的职能角色
   - 更新了配置状态初始化，包含`function_roles`字段

4. **数据加载功能**
   - 添加了`loadFunctionRoles`函数，从API加载职能角色数据
   - 更新了`loadBaseData`函数，包含职能角色数据加载

5. **职能角色管理函数**
   - `addFunctionRole`: 添加职能角色到审批节点
   - `removeFunctionRole`: 从审批节点移除职能角色
   - `updateFunctionRole`: 更新职能角色配置

6. **UI界面更新**
   - 替换了原有的组织结构配置界面
   - 新的职能角色配置界面包含：
     - 职能角色列表显示
     - "Add Function Head"和"Add Function Member"按钮
     - 已选职能角色的管理界面
     - 审批级别选择（Function Head、Function Member、All Members）
     - 优先级和主要审批人设置

### 界面特性

- **颜色主题**: 使用蓝色主题（blue.600）区分职能角色配置
- **审批级别**: 支持三种审批级别选择
- **优先级管理**: 支持设置审批优先级
- **主要审批人**: 支持设置主要审批人标记
- **动态配置**: 支持实时添加、删除和更新职能角色配置

### API集成

- 使用`/approval-nodes/function-roles`端点获取可用职能角色
- 与后端职能驱动审批模式API完全集成
- 支持完整的CRUD操作

### 使用方式

1. 在审批节点配置中选择"Function Role"节点类型
2. 从可用职能角色列表中选择需要的职能角色
3. 选择审批级别（Function Head、Function Member或All Members）
4. 设置优先级和主要审批人标记
5. 保存配置

这个更新使得前端界面与后端的职能驱动审批模式完全匹配，用户可以通过直观的界面配置基于职能角色的审批流程。
