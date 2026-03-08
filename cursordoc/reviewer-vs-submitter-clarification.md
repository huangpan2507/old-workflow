# "Reviewed By" vs "Submitted By" 字段含义澄清

## 问题

在 `request_for_update` 任务类型中，UI 显示 "Reviewed By: Email: xxx@example.com"，但用户不理解：
- 这是给谁 review？
- 和 "Attached Files - Uploaded by" 有什么区别？

---

## 代码分析

### 后端代码（steering_tasks.py）

```python
# 在 _process_workflow_update_callback 函数中（第924-938行）
# 4. Build reviewer info
reviewer_info = {
    'reviewer_id': str(current_user.id),
    'reviewer_email': current_user.email,  # ⚠️ 这是提交更新的用户
    'reviewer_job_title_id': current_user.job_title_id,
    'reviewer_job_title_name': job_title.name,  # ⚠️ 这是提交更新的用户的职位
    'steering_task_id': task.id
}
```

**关键发现**：
- `reviewer_email` 实际上是 `current_user.email`（提交更新的用户）
- `reviewer_job_title` 实际上是 `current_user.job_title`（提交更新的用户的职位）

---

## 字段含义对比

### 在 `request_for_approval` 场景中：

```
Reviewed By: reviewer@example.com
- 含义：审核人（做出 approve/reject 决定的人）
- 来源：提交审批决定的用户
```

### 在 `request_for_update` 场景中：

```
Reviewed By: submitter@example.com  ⚠️ 字段名不准确！
- 实际含义：提交更新的人（提交 update_info 和文件的人）
- 来源：提交更新信息的用户
- 问题：字段名是 "reviewer"，但实际是 "submitter"
```

---

## 为什么会有这个混淆？

### 历史原因

1. **代码复用**：
   - `process_human_decision` 方法同时处理 `approval` 和 `request_for_update`
   - 为了保持输出结构一致，使用了相同的字段名 `reviewer_email`

2. **字段名不准确**：
   - 在 `request_for_update` 场景中，没有"审核"的概念
   - 用户是"提交更新"而不是"审核"
   - 但字段名仍然是 `reviewer_email`

---

## 实际含义

### `request_for_update` 场景中的字段：

| 字段名 | 实际含义 | 说明 |
|--------|---------|------|
| `reviewer_email` | 提交更新的用户的 email | ⚠️ 字段名不准确，应该是 `submitted_by_email` |
| `reviewer_job_title` | 提交更新的用户的职位 | ⚠️ 字段名不准确，应该是 `submitted_by_job_title` |
| `decision_timestamp` | 提交更新的时间 | ✅ 字段名准确 |

### 与 "Attached Files - Uploaded by" 的区别：

| 位置 | 字段 | 含义 |
|------|------|------|
| **Update Information** | `reviewer_email` | 提交更新信息的用户（在 Message Center 提交 update_info 的用户） |
| **Attached Files** | `uploaded_by` | 上传文件的用户（在 Message Center 上传文件的用户） |

**注意**：
- 在大多数情况下，这两个用户是**同一个人**（同一个用户在 Message Center 提交更新信息和上传文件）
- 但在某些场景下，可能不同（例如：用户A提交更新信息，用户B上传文件）

---

## UI 显示建议

### 方案1：修改显示标签（推荐，无需后端改动）

**当前显示**：
```
Reviewed By:
• Email: submitter@example.com
• Job Title: Operations Manager
• Decision Time: 2026/1/13 16:16:22
```

**建议显示**：
```
Submitted By:  (或 "Updated By:")
• Email: submitter@example.com
• Job Title: Operations Manager
• Submitted Time: 2026/1/13 16:16:22  (或 "Update Time:")
```

**优点**：
- 无需修改后端代码
- 前端只需修改显示标签
- 语义更准确

---

### 方案2：修改后端字段名（需要后端改动）

**修改**：
- `reviewer_email` → `submitted_by_email`
- `reviewer_job_title` → `submitted_by_job_title`
- `decision_timestamp` → `submitted_timestamp`

**优点**：
- 字段名更准确
- 代码语义更清晰

**缺点**：
- 需要修改后端代码
- 可能影响其他使用这些字段的地方
- 需要数据库迁移（如果存储了这些字段）

---

## 推荐方案

**推荐使用方案1（修改前端显示标签）**，因为：

1. **无需后端改动**：保持后端代码的兼容性
2. **快速实现**：只需修改前端 UI 文本
3. **语义准确**：用户看到的是准确的描述
4. **向后兼容**：不影响现有的数据结构

---

## 更新后的 UI 预览

### 修改前：
```
┌─────────────────────────────────────────────────────────┐
│  Update Information                    [UPDATED] (绿色) │
├─────────────────────────────────────────────────────────┤
│  Update Information Content...                          │
│                                                          │
│  ────────────────────────────────────────────────────  │
│                                                          │
│  Reviewed By:  ⚠️ 不准确                                │
│  • Email: submitter@example.com                         │
│  • Job Title: Operations Manager                        │
│  • Decision Time: 2026/1/13 16:16:22                  │
└─────────────────────────────────────────────────────────┘
```

### 修改后：
```
┌─────────────────────────────────────────────────────────┐
│  Update Information                    [UPDATED] (绿色) │
├─────────────────────────────────────────────────────────┤
│  Update Information Content...                          │
│                                                          │
│  ────────────────────────────────────────────────────  │
│                                                          │
│  Submitted By:  ✅ 准确                                 │
│  • Email: submitter@example.com                         │
│  • Job Title: Operations Manager                        │
│  • Submitted Time: 2026/1/13 16:16:22                  │
└─────────────────────────────────────────────────────────┘
```

---

## 总结

1. **字段含义**：
   - `reviewer_email` 在 `request_for_update` 中实际是"提交更新的用户"
   - 不是"审核人"，而是"提交人"

2. **与 "Uploaded by" 的关系**：
   - 通常是同一个人（同一个用户在 Message Center 提交更新和上传文件）
   - 但概念上不同：一个是提交更新信息，一个是上传文件

3. **建议修改**：
   - 前端显示标签改为 "Submitted By" 或 "Updated By"
   - 时间标签改为 "Submitted Time" 或 "Update Time"
   - 无需修改后端字段名（保持兼容性）
