# Remove Status Column from Published Policies Table

## Overview
根据用户反馈，从Published Policies表格中删除了Status列，因为该表格中显示的所有政策都是已经发布过的政策，Status列是冗余的。

## Changes Made

### 1. PolicyTable Component Updates
- **条件渲染Status列**：只在非"published"模式下显示Status列
- **动态列数调整**：根据activeTab调整表格的列数
- **空状态colSpan更新**：根据列数动态调整空状态行的colSpan

### 2. Technical Implementation

#### Header Row
```typescript
<Tr>
  <Th>Policy Name</Th>
  <Th>Summary</Th>
  {activeTab !== "published" && <Th>Status</Th>}
  <Th>Effective Date</Th>
  <Th>Created</Th>
  <Th textAlign="right">Actions</Th>
</Tr>
```

#### Data Row
```typescript
<Td>
  <Text fontWeight="medium">
    {policy.policy_name}
  </Text>
</Td>
<Td>
  <Text fontSize="sm" color="gray.600" noOfLines={3} maxW="300px">
    {policy.policy_summary || "No summary available"}
  </Text>
</Td>
{activeTab !== "published" && <Td>{getStatusBadge(policy.approval_status)}</Td>}
<Td>{formatDate(policy.effective_date)}</Td>
<Td>{formatDate(policy.create_time)}</Td>
```

#### Empty State
```typescript
<Td colSpan={activeTab === "published" ? 5 : 6}>
  <Text color="gray.500" textAlign="center" py={8}>
    {/* Empty state message */}
  </Text>
</Td>
```

## Benefits
1. **减少冗余信息**：Published Policies表格不再显示不必要的Status列
2. **更清晰的界面**：表格更加简洁，专注于重要信息
3. **保持一致性**：其他模式（my-policies, requests）仍然显示Status列
4. **动态适应**：根据不同的activeTab自动调整表格结构

## Affected Modes
- **Published Policies**：不显示Status列（5列）
- **My Policies**：显示Status列（6列）
- **Requests/Approval**：显示Status列（6列）

## Result
现在Published Policies表格只显示以下列：
1. Policy Name
2. Summary
3. Effective Date
4. Created
5. Actions

Status列已被移除，使表格更加简洁和专注于已发布政策的核心信息。
