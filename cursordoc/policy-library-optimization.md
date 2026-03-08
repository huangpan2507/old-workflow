# 政策库页面数据加载优化

## 问题描述

原始实现中，政策库页面在进入时会一次性加载所有数据（前三个quick links的数据），然后用户切换quick links时没有新的网络请求，只是显示已缓存的数据。这导致：

1. **初始加载慢**：页面进入时需要加载大量数据
2. **缺乏实时性**：用户无法感知数据的最新状态
3. **用户体验差**：无法体现数据加载的实时性

## 解决方案

### 1. 创建按需数据加载Hook

创建了 `useOnDemandData` hook，提供以下功能：

- **分离的数据状态**：`publishedPolicies`、`myPolicies`、`pendingApprovals`
- **独立的加载状态**：每个数据源都有独立的loading状态
- **按需加载函数**：`loadPublishedPolicies`、`loadMyPolicies`、`loadPendingApprovals`
- **智能缓存**：避免重复加载已存在的数据

### 2. 优化组件架构

创建了 `PolicyUserInterfaceOptimized` 组件，主要改进：

- **移除一次性加载**：组件挂载时只加载必要的元数据（function roles、business units）
- **按需数据加载**：用户点击quick link时才加载对应数据
- **实时网络请求**：每次切换都会触发新的API请求（如果数据未缓存）

### 3. 数据加载时机

| Quick Link | 触发时机 | 加载的数据 |
|------------|----------|------------|
| **Explore** | 点击Explore按钮或搜索框获得焦点 | 已发布政策（status=approved） |
| **My Policies** | 点击My Policies按钮 | 当前用户创建的政策 |
| **My Approval** | 点击My Approval按钮 | 待审批政策 |

### 4. 技术实现细节

#### useOnDemandData Hook

```typescript
interface UseOnDemandDataReturn {
  // Published policies data
  publishedPolicies: Policy[]
  publishedLoading: boolean
  loadPublishedPolicies: () => Promise<void>
  
  // My policies data
  myPolicies: Policy[]
  myPoliciesLoading: boolean
  loadMyPolicies: () => Promise<void>
  
  // Pending approvals data
  pendingApprovals: Policy[]
  pendingApprovalsLoading: boolean
  loadPendingApprovals: () => Promise<void>
  
  // Combined loading state
  isLoading: boolean
}
```

#### 智能缓存机制

```typescript
const loadPublishedPolicies = useCallback(async () => {
  if (publishedPolicies.length > 0) {
    console.log("Published policies already loaded, skipping")
    return
  }
  // 执行API请求...
}, [publishedPolicies.length, toast])
```

#### 按需加载触发

```typescript
function handleExploreClick() {
  setActiveContent("published")
  loadPublishedPolicies() // 只在点击时加载
}

function handleMyPoliciesClick() {
  setActiveContent("my-policies")
  loadMyPolicies() // 只在点击时加载
}

function handleApprovalClick() {
  setActiveContent("approval")
  loadPendingApprovals() // 只在点击时加载
}
```

## 优化效果

### 1. 性能提升
- **初始加载时间减少**：页面进入时只加载必要数据
- **内存使用优化**：按需加载减少内存占用
- **网络请求优化**：避免不必要的API调用

### 2. 用户体验改善
- **实时数据感知**：用户切换quick link时能看到网络请求
- **加载状态反馈**：每个数据源都有独立的loading状态
- **响应速度提升**：页面初始加载更快

### 3. 开发体验提升
- **代码结构清晰**：数据加载逻辑分离
- **易于维护**：每个数据源独立管理
- **易于扩展**：新增quick link时只需添加对应的加载函数

## 使用方式

1. **替换组件**：将 `PolicyUserInterface` 替换为 `PolicyUserInterfaceOptimized`
2. **更新路由**：修改路由文件使用新的组件
3. **测试验证**：在浏览器开发者工具的Network标签中验证按需加载效果

## 验证方法

1. 打开浏览器开发者工具的Network标签
2. 进入政策库页面
3. 观察初始加载的请求数量（应该只有function roles和business units）
4. 依次点击不同的quick links
5. 验证每次点击都会触发对应的API请求（如果数据未缓存）

## 注意事项

1. **API兼容性**：确保后端API支持按状态过滤政策
2. **错误处理**：每个数据加载函数都有完整的错误处理
3. **缓存策略**：避免重复加载相同数据，但确保数据实时性
4. **用户体验**：在数据加载期间显示适当的loading状态
