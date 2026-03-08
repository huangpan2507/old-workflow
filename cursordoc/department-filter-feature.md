# Department Filter Feature Implementation

## Overview
为Published Policies添加了按部门过滤的功能，用户可以通过部门选择的tab list灵活选择要查看的部门政策。

## Features Implemented

### 1. Department Filter State Management
- 添加了`selectedDepartments`状态来管理选中的部门ID列表
- 支持多选部门过滤
- 状态与UI组件同步更新

### 2. Department Filter UI Components
- **部门选择按钮**：每个部门显示为一个可点击的按钮
- **选中状态**：选中的部门按钮显示为蓝色实心样式，未选中的为灰色边框样式
- **部门计数**：选中的部门按钮显示该部门下的政策数量
- **批量操作**：提供"Select All"和"Clear All"按钮
- **过滤状态提示**：显示当前选中部门下的政策总数

### 3. Enhanced Policy Filtering Logic
- 更新了`getCurrentPolicies`函数，支持部门过滤参数
- 在`usePolicies` hook中添加了部门过滤逻辑
- 过滤基于`responsible_department_id`字段

### 4. User Experience Improvements
- **实时更新**：选择部门后立即更新政策列表
- **视觉反馈**：选中状态清晰可见
- **计数显示**：每个部门显示对应的政策数量
- **状态提示**：显示当前过滤结果的政策总数

## Technical Implementation

### State Management
```typescript
const [selectedDepartments, setSelectedDepartments] = useState<string[]>([])
```

### Filter Functions
```typescript
function handleDepartmentToggle(departmentId: string) {
  setSelectedDepartments(prev => {
    if (prev.includes(departmentId)) {
      return prev.filter(id => id !== departmentId)
    } else {
      return [...prev, departmentId]
    }
  })
}
```

### Enhanced getCurrentPolicies Function
```typescript
function getCurrentPolicies(activeTab: string, statusFilter: string, departmentFilter?: string[]) {
  // ... existing logic ...
  
  // Apply department filter if provided
  if (departmentFilter && departmentFilter.length > 0) {
    filteredPolicies = filteredPolicies.filter(policy => 
      policy.responsible_department_id && 
      departmentFilter.includes(policy.responsible_department_id.toString())
    )
  }
  
  return filteredPolicies
}
```

## UI Layout
部门过滤组件位于：
1. Published Policies标题下方
2. 政策表格上方
3. 包含部门选择按钮和批量操作按钮
4. 显示过滤状态和计数信息

## Benefits
1. **灵活过滤**：用户可以自由选择要查看的部门
2. **多选支持**：可以同时选择多个部门
3. **实时反馈**：选择后立即看到结果
4. **计数显示**：清楚了解每个部门的政策数量
5. **批量操作**：快速选择或清除所有部门
6. **状态提示**：明确显示当前过滤结果

## Usage
1. 用户进入Published Policies页面
2. 在部门过滤区域选择感兴趣的部门
3. 系统实时更新政策列表
4. 可以随时添加或移除部门选择
5. 使用"Select All"或"Clear All"进行批量操作
