# Policy Lifecycle with PolicyRevision

本文档描述了一个政策的完整生命周期，以及如何使用 `PolicyRevision` 机制进行政策更新。

## 核心设计理念

- **单一 Policy 实例**：每个政策在 `policies` 表中只有一个记录，代表当前版本
- **PolicyRevision 表**：所有变更请求（新建、更新、废弃）都通过 `PolicyRevision` 表管理
- **PolicyHistory 表**：保存历史版本快照，用于版本追踪和审计

## 政策生命周期阶段

### 1. 创建阶段 (Creation)

**流程：**
```
创建 Policy → 创建 PolicyRevision (revision_type="new") → 提交审批
```

**详细步骤：**

1. **创建 Policy 记录**
   - 调用 `POST /api/v1/policies/`
   - 创建 `Policy` 记录，状态为 `draft`，版本号为 `1`
   - 上传文件到 MinIO（如果提供）

2. **自动创建初始 PolicyRevision**
   - 系统自动创建 `PolicyRevision`，`revision_type="new"`
   - `approval_status="draft"`
   - 包含完整的政策信息

3. **提交审批**
   - 调用 `POST /api/v1/policy-revisions/{policy_id}/revisions/{revision_id}/submit`
   - 创建审批工作流实例
   - `PolicyRevision.approval_status` 变为 `"pending"`
   - `PolicyRevision.approval_instance_id` 和 `current_approver_id` 被设置

**数据状态：**
- `Policy.status = "draft"`
- `Policy.version_number = 1`
- `PolicyRevision.revision_type = "new"`
- `PolicyRevision.approval_status = "pending"`

---

### 2. 审批阶段 (Approval)

**流程：**
```
审批人审批 → 审批通过/拒绝 → 如果通过，应用修订到 Policy
```

**详细步骤：**

1. **审批决策**
   - 调用 `POST /api/v1/policy-revisions/revisions/{revision_id}/approve`
   - 审批人做出 `approve` 或 `reject` 决策

2. **审批通过 (Approved)**
   - 如果 `revision_type="new"`：
     - 应用修订到 `Policy`
     - `Policy.status` 变为 `"active"`
     - `Policy.version_number` 递增
   - 如果 `revision_type="update"`：
     - 保存当前 `Policy` 状态到 `PolicyHistory`
     - 应用修订中的变更到 `Policy`（只更新非 None 字段）
     - `Policy.version_number` 递增
   - 如果 `revision_type="obsolete"`：
     - 保存当前 `Policy` 状态到 `PolicyHistory`
     - `Policy.status` 变为 `"obsoleted"`
     - `Policy.version_number` 递增
   - `PolicyRevision.approval_status` 变为 `"approved"`
   - 记录 `approved_at` 和 `approved_by`

3. **审批拒绝 (Rejected)**
   - `PolicyRevision.approval_status` 变为 `"rejected"`
   - 记录 `rejected_at`、`rejected_by` 和 `rejection_reason`
   - `Policy` 保持不变

**数据状态（审批通过后）：**
- `Policy.status = "active"` (首次审批) 或保持当前状态（更新）
- `Policy.version_number = 2, 3, 4...` (递增)
- `PolicyRevision.approval_status = "approved"`
- `PolicyHistory` 中保存了旧版本快照（如果是更新或废弃）

---

### 3. 更新阶段 (Update)

**流程：**
```
创建 PolicyRevision (revision_type="update") → 提交审批 → 审批通过 → 应用变更
```

**详细步骤：**

1. **创建更新修订**
   - 调用 `POST /api/v1/policy-revisions/{policy_id}/revisions`
   - 设置 `revision_type="update"`
   - 只提供需要变更的字段（其他字段为 `None`）
   - 可以上传新文件替换旧文件
   - `PolicyRevision.approval_status = "draft"`

2. **提交审批**
   - 调用 `POST /api/v1/policy-revisions/{policy_id}/revisions/{revision_id}/submit`
   - 创建审批工作流实例
   - `PolicyRevision.approval_status = "pending"`

3. **审批通过**
   - 调用 `POST /api/v1/policy-revisions/revisions/{revision_id}/approve`
   - `PolicyRevisionService.apply_revision_to_policy()` 执行：
     - **保存历史**：将当前 `Policy` 状态保存到 `PolicyHistory`
     - **应用变更**：只更新 `PolicyRevision` 中非 `None` 的字段
     - **版本递增**：`Policy.version_number += 1`
     - **更新状态**：`PolicyRevision.approval_status = "approved"`

**关键特性：**
- **部分更新**：只更新提供的字段，未提供的字段保持原值
- **历史保留**：旧版本保存在 `PolicyHistory` 中
- **版本追踪**：通过 `version_number` 追踪版本变化

**示例：**
```python
# 只更新政策名称和摘要
revision = PolicyRevision(
    policy_id=1,
    revision_type="update",
    revision_reason="更新政策描述",
    policy_name="新政策名称",  # 只更新这个
    policy_summary="新摘要",    # 只更新这个
    # 其他字段为 None，不会更新
)
```

---

### 4. 废弃阶段 (Obsoletion)

**流程：**
```
创建 PolicyRevision (revision_type="obsolete") → 提交审批 → 审批通过 → 标记废弃
```

**详细步骤：**

1. **创建废弃修订**
   - 调用 `POST /api/v1/policies/{policy_id}/obsolete` 或
   - 调用 `POST /api/v1/policy-revisions/{policy_id}/revisions` (revision_type="obsolete")
   - 必须提供 `obsoletion_reason`
   - `PolicyRevision.approval_status = "draft"`

2. **提交审批**
   - 创建审批工作流实例
   - `PolicyRevision.approval_status = "pending"`

3. **审批通过**
   - 保存当前 `Policy` 状态到 `PolicyHistory`
   - `Policy.status` 变为 `"obsoleted"`
   - `Policy.version_number` 递增
   - 废弃信息存储在 `PolicyRevision.obsoletion_reason` 中

**数据状态：**
- `Policy.status = "obsoleted"`
- `PolicyRevision.revision_type = "obsolete"`
- `PolicyRevision.obsoletion_reason` 包含废弃原因
- `PolicyHistory` 中保存了废弃前的最后版本

---

## PolicyRevision 字段说明

### 核心字段

| 字段 | 说明 | 示例值 |
|------|------|--------|
| `revision_type` | 修订类型 | `"new"`, `"update"`, `"obsolete"` |
| `revision_reason` | 修订原因 | `"更新政策内容"` |
| `approval_status` | 审批状态 | `"draft"`, `"pending"`, `"approved"`, `"rejected"` |
| `approval_instance_id` | 审批工作流实例 ID | `123` |
| `current_approver_id` | 当前审批人 ID | `uuid` |

### 变更字段（部分更新）

这些字段在 `PolicyRevision` 中为可选，只有非 `None` 的字段才会应用到 `Policy`：

- `policy_name`
- `policy_summary`
- `responsible_function_role_id`
- `applicability_type_business_unit`
- `applicable_business_units`
- `effective_date`
- `valid_until`
- `file_object_name` (新文件)
- `obsoletion_reason` (仅废弃时)

---

## 数据表关系

```
┌─────────────┐
│   Policy    │  ← 当前版本（单一实例）
│             │
│ id          │
│ status      │  ← "draft", "active", "obsoleted"
│ version     │  ← 当前版本号
└──────┬──────┘
       │
       │ 1:N
       │
       ▼
┌──────────────────┐
│ PolicyRevision   │  ← 所有变更请求
│                  │
│ policy_id        │
│ revision_type    │  ← "new", "update", "obsolete"
│ approval_status  │  ← "draft", "pending", "approved", "rejected"
│ approval_instance_id │
│ current_approver_id │
└──────┬───────────┘
       │
       │ 1:N
       │
       ▼
┌──────────────────┐
│ PolicyHistory    │  ← 历史版本快照
│                  │
│ policy_id        │
│ version_number    │  ← 历史版本号
│ revision_id      │  ← 关联的修订 ID
└──────────────────┘
```

---

## 关键 API 端点

### 创建政策
```
POST /api/v1/policies/
→ 创建 Policy + PolicyRevision (revision_type="new")
```

### 创建修订
```
POST /api/v1/policy-revisions/{policy_id}/revisions
→ 创建 PolicyRevision (revision_type="update" 或 "obsolete")
```

### 提交审批
```
POST /api/v1/policy-revisions/{policy_id}/revisions/{revision_id}/submit
→ 创建审批工作流，状态变为 "pending"
```

### 审批决策
```
POST /api/v1/policy-revisions/revisions/{revision_id}/approve
→ 审批通过/拒绝，如果通过则应用变更
```

### 查看修订列表
```
GET /api/v1/policy-revisions/{policy_id}/revisions
→ 获取某个政策的所有修订请求
```

### 查看版本历史
```
GET /api/v1/policies/{policy_id}/version-history
→ 获取 PolicyHistory 中的历史版本
```

---

## 优势

1. **单一数据源**：`Policy` 表始终代表当前版本，避免数据不一致
2. **完整审计**：所有变更都通过 `PolicyRevision` 记录，可追溯
3. **版本管理**：`PolicyHistory` 保存历史快照，支持版本对比
4. **灵活更新**：支持部分字段更新，不需要提供完整数据
5. **并发安全**：多个修订请求可以同时存在，通过审批流程控制
6. **审批追踪**：每个修订都有独立的审批流程和状态

---

## 状态流转图

```
创建 Policy
    ↓
创建 PolicyRevision (new, draft)
    ↓
提交审批 → PolicyRevision (new, pending)
    ↓
审批通过 → Policy (active, v1) + PolicyRevision (approved)
    ↓
创建 PolicyRevision (update, draft)
    ↓
提交审批 → PolicyRevision (update, pending)
    ↓
审批通过 → PolicyHistory (v1) + Policy (active, v2) + PolicyRevision (approved)
    ↓
创建 PolicyRevision (obsolete, draft)
    ↓
提交审批 → PolicyRevision (obsolete, pending)
    ↓
审批通过 → PolicyHistory (v2) + Policy (obsoleted, v3) + PolicyRevision (approved)
```

---

## 注意事项

1. **废弃政策不能更新**：`status="obsoleted"` 的政策不能创建新的修订
2. **部分更新**：只提供需要变更的字段，未提供的字段保持原值
3. **版本递增**：每次审批通过后，`version_number` 自动递增
4. **历史保存**：更新和废弃时会自动保存当前版本到 `PolicyHistory`
5. **审批必需**：所有变更都必须经过审批流程才能生效

