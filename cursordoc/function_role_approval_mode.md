# 职能驱动审批模式 (Function Role-Driven Approval Mode)

## 概述

职能驱动审批模式是一种基于职能角色的政策审批工作流，它允许系统根据政策的性质和涉及的职能部门自动确定审批人。这种模式特别适用于需要多个职能部门参与审批的复杂政策。

## 功能特性

### 1. 职能角色支持
- 支持23种标准职能角色（IT、HR、Finance、Legal等）
- 每个职能角色可以配置不同的审批级别
- 支持职能角色范围的灵活配置

### 2. 审批级别配置
- **function_head**: 职能部门负责人审批
- **function_member**: 职能部门成员审批  
- **all_members**: 职能部门所有成员都可以审批

### 3. 范围配置
- **global**: 全局范围
- **group**: 特定组范围
- **business_unit**: 特定业务单元范围
- **function**: 特定职能部门范围

## 数据模型

### PolicyApprovalNodeFunctionRole
```python
class PolicyApprovalNodeFunctionRole(SQLModel, table=True):
    """职能角色审批节点配置"""
    id: int = Field(primary_key=True)
    node_id: int = Field(foreign_key="policy_approval_node_definitions.id")
    function_role_id: int = Field(foreign_key="function_roles.id")
    
    # 配置参数
    is_primary: bool = Field(default=False)  # 是否主要审批人
    priority: int = Field(default=0)  # 审批优先级
    scope_type: str | None = Field(default=None)  # 范围类型
    scope_config: str | None = Field(default=None)  # 范围配置JSON
    approval_level: str = Field(default="function_head")  # 审批级别
    level_config: str | None = Field(default=None)  # 级别配置JSON
    
    # 时间控制
    effective_from: datetime | None = Field(default=None)
    effective_until: datetime | None = Field(default=None)
    
    # 状态管理
    is_active: bool = Field(default=True)
```

## API接口

### 1. 添加职能角色到审批节点
```http
POST /api/v1/approval-nodes/nodes/{node_id}/function-roles
Content-Type: application/json

{
    "function_role_id": 1,
    "is_primary": true,
    "priority": 10,
    "scope_type": "global",
    "approval_level": "function_head"
}
```

### 2. 获取节点的职能角色配置
```http
GET /api/v1/approval-nodes/nodes/{node_id}/function-roles
```

### 3. 更新职能角色配置
```http
PUT /api/v1/approval-nodes/nodes/{node_id}/function-roles/{function_role_id}
Content-Type: application/json

{
    "priority": 15,
    "approval_level": "function_member"
}
```

### 4. 删除职能角色配置
```http
DELETE /api/v1/approval-nodes/nodes/{node_id}/function-roles/{function_role_id}
```

### 5. 获取所有可用职能角色
```http
GET /api/v1/approval-nodes/function-roles
```

## 使用示例

### 创建职能驱动审批模板

1. **创建审批模板**
```python
template = PolicyApprovalTemplate(
    name="IT Policy Approval Workflow",
    description="IT相关政策审批流程",
    workflow_type="function_role_driven",
    is_active=True
)
```

2. **创建审批节点**
```python
node = PolicyApprovalNodeDefinition(
    template_id=template.id,
    step_order=1,
    node_name="IT Function Head Approval",
    node_type="function_roles",
    is_required=True,
    approval_logic="any"
)
```

3. **配置职能角色**
```python
function_role_assignment = PolicyApprovalNodeFunctionRole(
    node_id=node.id,
    function_role_id=it_function_role.id,
    is_primary=True,
    priority=10,
    scope_type="global",
    approval_level="function_head"
)
```

### 审批流程示例

假设有一个IT安全政策需要审批：

1. **政策提交**: 用户提交IT安全政策
2. **自动识别**: 系统识别政策涉及IT职能
3. **查找审批人**: 系统查找IT职能部门的负责人
4. **分配审批**: 将审批任务分配给IT职能负责人
5. **审批完成**: IT职能负责人审批后，进入下一阶段

## 配置说明

### 范围配置示例

```json
{
    "scope_type": "function",
    "scope_config": {
        "function_ids": [1, 2, 3],
        "business_unit_id": 1
    }
}
```

### 级别配置示例

```json
{
    "approval_level": "function_member",
    "level_config": {
        "employee_levels": ["senior", "manager"],
        "exclude_levels": ["intern"]
    }
}
```

## 优势

1. **自动化**: 根据政策性质自动确定审批人
2. **灵活性**: 支持多种审批级别和范围配置
3. **可扩展**: 易于添加新的职能角色和配置
4. **可追溯**: 完整的审批历史记录
5. **权限控制**: 基于职能角色的细粒度权限管理

## 注意事项

1. 确保职能角色数据已正确初始化
2. 审批节点必须设置为`node_type="function_roles"`
3. 职能角色配置必须与实际的职能部门结构匹配
4. 建议在生产环境中测试审批流程的完整性

## 相关文件

- 模型定义: `backend/app/models.py`
- 服务逻辑: `backend/app/services/policy_approval_service.py`
- API接口: `backend/app/api/v1/approval_nodes.py`
- 数据库迁移: `backend/app/alembic/versions/g1234567890b_add_function_role_approval_support.py`
- 工作模板: `backend/app/core/database/default_data/policy_approval_templates.py`
