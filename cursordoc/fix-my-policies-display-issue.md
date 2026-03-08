# Fix My Policies Display Issue

## 问题描述

在 My Policies 页面，用户无法看到自己创建的所有 policies。

**原因**：后端 API 的权限过滤逻辑会过滤掉 `applicable_function_role_ids` 不包含用户 function_id 的 policies，即使用户是这些 policies 的创建者。

## 解决方案

修改 `backend/app/api/v1/policies.py` 中的权限过滤逻辑：

**修改前**：只显示 applicable 到用户 BU/function 的 policies
**修改后**：显示用户创建的 policies **OR** applicable 到用户 BU/function 的 policies

### 代码变更

```python
# Apply permission filtering
if not current_user.is_superuser:
    # Filter by applicable BUs and departments, OR policies created by current user
    bu_filter = None
    dept_filter = None
    function_role_filter = None
    my_policies_filter = Policy.created_by == str(current_user.id)  # 新增：用户自己创建的
    
    if current_user.business_unit_id:
        bu_filter = Policy.applicable_business_units.like(f'%{current_user.business_unit_id}%')
    
    if current_user.function_id:
        function_role_filter = Policy.applicable_function_role_ids.like(f'%{current_user.function_id}%')
    
    # Combine filters: user's own policies OR applicable to user's BU/function
    filter_conditions = [my_policies_filter]  # 总是包含用户自己创建的
    if bu_filter is not None:
        filter_conditions.append(bu_filter)
    if function_role_filter is not None:
        filter_conditions.append(function_role_filter)
    
    if len(filter_conditions) > 1:
        query = query.where(or_(*filter_conditions))  # 使用 OR 组合条件
    elif len(filter_conditions) == 1:
        query = query.where(filter_conditions[0])
```

## 逻辑说明

现在 API 返回的 policies 满足以下任一条件：
1. **用户创建的** (`Policy.created_by == current_user.id`)
2. **Applicable 到用户的 BU** (`applicable_business_units` 包含用户 BU)
3. **Applicable 到用户的 Function** (`applicable_function_role_ids` 包含用户 function)

这样可以确保：
- 用户在 My Policies 页面总是能看到自己创建的所有 policies
- 即使用户的 policies 在创建时没有正确设置 `applicable_function_role_ids`
- 也不影响其他权限逻辑

