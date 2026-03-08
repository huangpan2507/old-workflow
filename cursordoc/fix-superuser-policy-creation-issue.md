# Fix Superuser Policy Creation Issue

## 问题描述

使用超级管理员 (admin) 账户登录时，无法创建新的 policy，因为：
1. 超级管理员账户不属于任何业务单元 (BU)
2. 根据新的审批逻辑，需要先找到创建者所在的 BU，然后在 BU 下查找对应的 Function
3. 没有 BU 就无法找到 Function，因此无法找到审批人
4. 导致创建 policy 失败，错误消息："No approvers found for step department head"

## 解决方案

### 1. 前端 UI 优化

修改 `MyPolicies.tsx` 组件，添加以下功能：

1. **检查当前用户是否是超级管理员且没有 BU**
   ```typescript
   const isSuperUserWithoutBU = useMemo(() => {
     if (!currentUser) return false
     return currentUser.is_superuser && !currentUser.business_unit_id
   }, [currentUser])
   ```

2. **显示警告信息**
   - 当检测到超级管理员且没有 BU 时，显示黄色警告横幅
   - 提示："当前是超级管理员登录，无法新建政策，因为其不属于任何业务单元 (BU)。请使用普通用户账户创建政策。"

3. **禁用 Publish New Policy 按钮**
   - `isDisabled={isSuperUserWithoutBU}`
   - 灰化按钮样式
   - 添加 `cursor: not-allowed` 样式

### 2. 代码修改

**文件**: `frontend/src/components/PolicyUserInterface/my-policies/MyPolicies.tsx`

1. 添加 imports:
   ```typescript
   import type { ExtendedUserPublic } from "@/types/user"
   import { WarningIcon } from "@chakra-ui/icons"
   import { Alert, AlertDescription, AlertIcon, ... } from "@chakra-ui/react"
   ```

2. 添加 props:
   ```typescript
   interface MyPoliciesProps {
     // ... existing props
     currentUser?: ExtendedUserPublic | null
   }
   ```

3. 添加检查逻辑和 UI 元素

**文件**: `frontend/src/components/PolicyUserInterface/PolicyUserInterfaceOptimized.tsx`

在 MyPolicies 组件调用处添加 `currentUser={user}` prop

## 用户指导

### 对于超级管理员

如果您需要使用超级管理员账户创建 policy，有以下选项：

1. **为超级管理员分配一个 BU**
   - 通过用户管理界面，将超级管理员分配到某个业务单元
   - 然后就可以正常创建 policy

2. **使用普通用户账户**
   - 超级管理员可以创建具有 BU 的普通用户
   - 使用普通用户账户创建 policy

### 设计说明

这个限制是**有意设计的**：
- Policy 的审批流程依赖于组织结构
- 需要创建者所属的 BU 来确定审批人
- 超级管理员通常用于系统管理，不需要创建日常 policy
- 如果需要创建 policy，应该使用有组织归属的普通用户账户

## 测试

1. 使用超级管理员账户登录
2. 导航到 "My Policies" 页面
3. 确认显示警告消息
4. 确认 "Publish New Policy" 按钮被灰化且禁用
5. 点击按钮，确认无法打开创建 policy 的对话框

