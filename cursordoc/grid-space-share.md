# Grid Space Share Capability

## Overview

This document explains the new grid space sharing workflow that enables owners to share entire Canvas workspaces with specific users in read-only or editable modes. The share flow integrates with the existing resource/permission model and the generic approval workflow engine.

## Data Model Updates

- Added table `grid_space_shares` to persist share requests, status, approval metadata, and captured resource snapshots (apps/disks referenced on the canvas).
- `grid_space` table now has:
  - `parent_space_id`: links editable copies to their original canvas.
  - `share_source_id`: stores the originating share record for duplicated canvases.
- `backend/app/models.py` includes:
  - SQLModel `GridSpaceShare` plus DTOs `GridSpaceShareCreateRequest` / `GridSpaceShareRead`.
  - `GridSpace` schema fields `parent_space_id` / `share_source_id`.
- `Permission.scope_type='user'` is treated as a direct user allocation in `AdvancedPermissionChecker`.

## Service Layer

- New `GridSpaceShareService` (`backend/app/services/grid_space_share_service.py`):
  - Collects grid dependencies (apps/disks) from `node_list`.
  - Detects missing permissions per target user/action; triggers approval only if needed.
  - Creates approval workflows with `resource_type="grid_space_share"` when missing dependencies exist.
  - Activates shares by granting `Permission` rows (`scope_type='user'`) for grid/app/disk resources and, for edit mode, cloning the grid into a new record owned by the recipient.
  - Exposes helper `ensure_resource_registered` so new canvases immediately get a `Resource` row.
  - Handles approval callbacks through `activate_share` / `mark_share_rejected`.

## API Changes

- `POST /grid-space/{id}/share` (owner only): create share request with body `GridSpaceShareCreateRequest`.
- `GET /grid-space/{id}/share` (owner only): list share records with status indicators (e.g., Pending, Active) for UI badges.
- `GET /grid-space/{id}`: now allows access if requester is listed on an active share (read-only mode) in addition to the owner.
- Grid creation endpoint now registers a `grid_space` resource automatically.

## Approval Integration

- `ApprovalWorkflowService.process_approval_decision` now monitors `resource_type == "grid_space_share"`:
  - On completion → invokes `GridSpaceShareService.activate_share`.
  - On failure → marks the share as rejected with context for UI messaging.

## Frontend Considerations

- Share entry点：Canvas 右上角新增 Share 按钮，弹出 `GridSpaceShareModal`，支持搜索用户、选择 Read/Edit 模式并填写 Description，调用 `POST /grid-space/{id}/share`。可编辑模式在审批通过后会提示“Editable copy”。
- Share 交互优化：画布功能按钮（Edit）与 Share 垂直堆叠，Share 按钮位于 Edit 下方，使用 `blue.500` 背景与白色 `FiShare2` 图标，并在 hover 时统一显示 “Share this canvas” 文案（即使按钮禁用也保持一致提示）。
- 列表状态：`AIGridSidebar` 在每个 owned 空间卡片上展示分享状态徽标（Pending/Shared），状态数据来自 `GET /grid-space/share-status`。
- Shared with me：侧边栏新增 `SHARED WITH ME` 分区，调用 `GET /grid-space/shared-with-me` 展示分享给当前用户的画布（含待审批/已分享两种状态）；只读分享只能查看，可编辑分享直接打开复制出来的 grid space。
- Editable shares open the cloned grid (`shared_space_id`)，read-only shares keep the original canvas ID but UI 强制只读，禁止进入 Edit Mode。

Refer to `backend/app/alembic/versions/u20251121_add_grid_space_share.py` for the exact migration.

## Share Management Enhancements (2025-11)

### API Surface

- `PATCH /grid-space/share/{share_id}`：支持权限级别切换（read↔edit）、分享对象纠错、备注更新。升级为 edit 会重新触发审批；降级为 read 会清理可编辑副本并重新授予只读权限。
- `POST /grid-space/share/{share_id}/revoke`：撤销单条分享，清理 `shared_space_id` 指向的副本，并通过 `_revoke_share_permissions` 失效所有临时 `Permission`。
- `POST /grid-space/share/bulk-update` / `bulk-revoke`：批量执行上述操作，接口返回更新后的 `GridSpaceShareRead`。
- `GET /grid-space/{id}/share` 现返回 `shared_by_user`、`shared_to_user` 元信息，便于前端展示。

### Service Logic

- `GridSpaceShareService.update_share` 新增：
  - `_prepare_share_snapshot`、`_cleanup_shared_space`、`_revoke_share_permissions` 等辅助函数，确保降级即刻回收编辑权限、删除副本，并记录 `audit_trail`。
  - 审批模板 ID 持久化在 `share_metadata.approval_template_id`，以支持重复触发审批。
- `revoke_share` 把分享状态置为 `revoked`，写入原因与审计记录，方便合规追踪。

### Frontend – Share Management View

- `GridSpaceShareModal` 扩展为“分享管理”页面：
  - 顶部保留创建入口，下方加入分享列表、批量操作（设为 Read Only、设为 Editable、批量撤销）、实时刷新。
  - 每条记录显示接收人、分享人、当前权限、审批状态、最近审计日志；提供权限切换、指派纠错、撤销按钮。
  - 支持批量多选，通过 `GridSpaceService.bulkUpdateShares` / `bulkRevokeShares` 直接调用后端。
  - 更换接收人时内嵌用户搜索列表，提交后自动刷新 `shared-with-me` 与分享状态。

### Audit & Visibility

- `share_metadata.audit_trail` 记录 `{timestamp, actor, action, details}`，UI 展示最近一次变更。
- 当正在查看的共享副本被降级时，前端根据 `share.shared_space_id` 清空情况将用户重定向回原画布的只读模式，保证错误对象立即失去访问权限。

