# Actions Column Functionality Update for Published Policies

## Overview
根据客户需求，对Explore模式（Published Policies）下的Actions列功能进行了调整：
1. 移除Delete功能（因为只是查看模式）
2. 眼睛按钮改为预览文件内容
3. 将显示政策详情的功能移到三个点菜单中

## Changes Made

### 1. PolicyTable Component Updates

#### Interface Changes
```typescript
interface PolicyTableProps {
  // ... existing props
  onPreviewFile?: (policy: Policy) => void
}
```

#### Actions Column Logic
- **Published Mode (activeTab === "published")**:
  - 眼睛按钮：预览文件内容（仅当有文件时显示）
  - 三个点菜单：只显示"View Policy Details"
  - 移除：Delete功能

- **Other Modes (my-policies, requests)**:
  - 眼睛按钮：查看政策详情
  - 三个点菜单：显示所有管理选项（Submit for Approval, Create Revision, Delete）

#### Code Implementation
```typescript
{activeTab === "published" ? (
  // Published mode: Preview file content
  onPreviewFile && policy.file_object_name && (
    <IconButton
      aria-label="Preview File"
      icon={<ViewIcon />}
      size="sm"
      onClick={() => onPreviewFile(policy)}
    />
  )
) : (
  // Other modes: View policy details
  <IconButton
    aria-label="View Details"
    icon={<ViewIcon />}
    size="sm"
    onClick={() => onViewDetails(policy)}
  />
)}
```

### 2. PolicyUserInterface Component Updates

#### New Handler Function
```typescript
function handlePreviewFile(policy: Policy) {
  // TODO: Implement file preview functionality
  console.log("Preview file for policy:", policy.policy_name)
  // For now, we can open the file viewer drawer
  setActiveContent("file-viewer")
}
```

#### Updated PolicyTable Calls
```typescript
<PolicyTable
  policies={getCurrentPolicies("published", "", selectedDepartments)}
  loading={loading}
  activeTab="published"
  statusFilter=""
  onDownload={handleDownload}
  onViewDetails={handleViewDetails}
  onDeletePolicy={handleDeletePolicyWithCallback}
  onSubmitForApproval={handleSubmitForApprovalWithCallback}
  onPreviewFile={handlePreviewFile}  // New prop
/>
```

## Functionality Summary

### Published Policies Mode (Explore)
- **眼睛按钮**：预览文件内容（仅当政策有文件时显示）
- **下载按钮**：下载政策文件
- **三个点菜单**：
  - View Policy Details（查看政策详情）

### Other Modes (My Policies, Requests)
- **眼睛按钮**：查看政策详情
- **下载按钮**：下载政策文件
- **三个点菜单**：
  - Submit for Approval（草稿状态时）
  - Create Revision（已批准状态时）
  - Delete（删除政策）

## Benefits
1. **角色分离**：Published模式专注于查看，其他模式专注于管理
2. **功能清晰**：眼睛按钮在不同模式下有不同的功能
3. **用户友好**：Published模式下移除了不必要的Delete功能
4. **一致性**：政策详情功能统一放在三个点菜单中

## Future Implementation
- `handlePreviewFile`函数目前是占位符实现
- 需要实现实际的文件预览功能
- 可以考虑集成文件查看器或PDF预览组件
