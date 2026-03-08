# Department Head Approval Logic Update

## 问题描述

原先的 department head 审批人查找逻辑存在问题：
- 直接根据 `policy.responsible_function_role_id` 查找 `function_head`
- 在全局范围内查找用户，没有限制到创建者所在的 BU

## 新的查找逻辑

根据用户需求，现在的查找逻辑如下：

```
1. 根据当前策略创建人找到其所属的 BU
2. 在这个 BU 下找到跟 responsible_function_role_id 相同的部门
3. 找到这个部门的 head，这是最终的查找结果
```

## 代码变更

### 修改的文件

1. **backend/app/services/policy_approval_service.py**
   - 更新了 `_get_function_role_head` 方法
   - 原签名：`_get_function_role_head(function_role_id: int)`
   - 新签名：`_get_function_role_head(policy: Policy)`
   - 新的查找逻辑：
     - 获取创建者用户及其 `business_unit_id`
     - 在创建者的 BU 下查找 Function
     - 匹配条件：`Function.business_unit_id == creator.business_unit_id` 且 `Function.function_role_id == policy.responsible_function_role_id`
     - 返回该 Function 的 `function_head_id` 对应的用户

2. **backend/app/api/v1/policies.py**
   - 删除了 `_get_function_role_approver` 函数（不再需要）
   - 删除了在创建 policy 时手动设置 approver 的逻辑
   - 审批人现在由 `PolicyApprovalService` 统一查找

## 实现细节

```python
def _get_function_role_head(self, policy: Policy) -> Optional[User]:
    """
    Get the function role head based on the logic:
    1. Find the creator's BU
    2. In that BU, find the Function with matching function_role_id
    3. Return the function_head of that Function
    """
    if not policy.responsible_function_role_id or not policy.created_by:
        return None
    
    # Step 1: Get the creator user
    creator = self.session.get(User, policy.created_by)
    if not creator or not creator.business_unit_id:
        return None
    
    # Step 2: In the creator's BU, find Function with matching function_role_id
    function_query = select(Function).where(
        and_(
            Function.business_unit_id == creator.business_unit_id,
            Function.function_role_id == policy.responsible_function_role_id,
            Function.is_active == True
        )
    )
    function = self.session.exec(function_query).first()
    
    if not function or not function.function_head_id:
        return None
    
    # Step 3: Return the function_head user
    return self.session.get(User, function.function_head_id)
```

## 数据模型关系

```
Policy
  ├─ created_by -> User (创建者)
  └─ responsible_function_role_id -> FunctionRole (责任职能部门角色类型)

User
  └─ business_unit_id -> BusinessUnit (所属业务单元)

Function
  ├─ business_unit_id -> BusinessUnit (所属业务单元)
  ├─ function_role_id -> FunctionRole (职能部门角色类型)
  └─ function_head_id -> User (部门负责人)
```

## 查找流程示例

假设：
- Policy 创建者：张三（email: zhangsan@example.com）
- 张三所在 BU：XX学校（ID: 100）
- Policy 的 responsible_function_role_id：DCB DBA（ID: 50）

查找过程：
1. 根据 policy.created_by 获取张三用户
2. 获取张三的 business_unit_id = 100
3. 在 Function 表中查找：
   - `business_unit_id == 100`
   - `function_role_id == 50`
   - `is_active == True`
4. 找到对应的 Function，获取其 function_head_id
5. 返回该 function_head 用户作为审批人

## 影响范围

- 所有使用 `function_role_head` 节点类型的审批流程
- 审批人会从同一个 BU 内查找，而不是全局查找
- 如果创建者没有所属 BU，或者 BU 下没有对应的 Function，将无法找到审批人

