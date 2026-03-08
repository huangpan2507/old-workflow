# 文件保留清理策略文档

## 概述

本文档详细说明系统中文件保留（Retain File）的清理策略、机制和配置。系统通过两种机制自动管理保留文件的清理，确保磁盘空间得到有效利用。

## 清理机制

系统提供两种文件清理机制：

### 1. 定期清理过期文件（定时任务）

**执行时间**：每天凌晨 2:00 自动执行

**清理范围**：仅清理已过期的保留文件

**实现位置**：
- 任务定义：`backend/app/services/background_tasks.py:164-173`
- 任务执行：`backend/app/services/background_tasks.py:475-613`

**清理逻辑**：
- 查询所有 `retention_expires_at <= now()` 的保留文件
- 删除 MinIO 中的文件对象
- 更新数据库记录：`is_retained = False`, `file_storage_path = None`, `retention_expires_at = None`
- 记录清理统计信息到任务执行日志

**特点**：
- 仅清理已过期文件，不会清理未过期文件
- 执行后检查磁盘空间状态，如果仍超过警告阈值会记录告警日志
- 任务执行结果会记录到后台任务执行日志中

### 2. 磁盘空间清理（按需触发）

**触发条件**：磁盘使用率超过 `DISK_SPACE_WARNING_THRESHOLD`（默认 80%）

**清理目标**：释放空间至 `DISK_SPACE_RECOVERY_TARGET`（默认 50%）

**实现位置**：
- 空间计算：`backend/app/services/disk_space_service.py:127-147`
- 文件选择：`backend/app/services/file_retention_service.py:262-306`
- 清理执行：`backend/app/services/file_retention_service.py:369-554`

**清理逻辑**：
1. 计算需要释放的空间：
   ```python
   target_used = total * (recovery_target / 100)
   space_to_free = max(0, used - target_used)
   ```
2. 按优先级选择文件进行删除
3. 删除 MinIO 中的文件对象
4. 更新数据库记录
5. 返回清理统计信息

**特点**：
- 会清理未过期文件（如果已过期文件不足以释放目标空间）
- 优先清理已过期文件，然后清理最旧的文件
- 可通过 API 手动触发（仅超级用户）

## 清理优先级

文件清理按照以下优先级顺序进行：

### 排序规则

在 `get_oldest_retention_files()` 方法中，文件按以下规则排序：

```262:285:backend/app/services/file_retention_service.py
    def get_oldest_retention_files(
        self, 
        db: Session, 
        target_space_bytes: int
    ) -> List[FileIngestionRecord]:
        """
        Get list of oldest retention files that need to be deleted to free target space
        
        Args:
            db: Database session
            target_space_bytes: Target space to free in bytes
            
        Returns:
            List of FileIngestionRecord objects, ordered by retention_expires_at (oldest first)
        """
        try:
            # Get all retained files, ordered by expiration date (oldest first)
            query = select(FileIngestionRecord).where(
                FileIngestionRecord.is_retained == True,
                FileIngestionRecord.file_storage_path.isnot(None)
            ).order_by(
                FileIngestionRecord.retention_expires_at.asc().nullsfirst(),
                FileIngestionRecord.uploaded_at.asc()
            )
```

**优先级顺序**：

1. **已过期文件优先**
   - 条件：`retention_expires_at <= now()`
   - 排序：按 `retention_expires_at` 升序（最早过期的先删除）

2. **无过期时间的文件**
   - 条件：`retention_expires_at IS NULL`
   - 排序：按 `uploaded_at` 升序（最早上传的先删除）

3. **未过期文件**
   - 条件：`retention_expires_at > now()`
   - 排序：按 `retention_expires_at` 升序（最早过期的先删除）

**选择逻辑**：
- 按上述顺序遍历所有保留文件
- 累加文件大小，直到 `total_space >= target_space_bytes`
- 返回选中的文件列表

## 配置参数

### 环境变量配置

在 `env.dev` 文件中配置以下参数：

```76:81:env.dev
# 磁盘空间管理配置
MINIO_DATA_VOLUME_PATH=/data/minio
DISK_SPACE_WARNING_THRESHOLD=80.0
DISK_SPACE_RECOVERY_TARGET=50.0
FILE_RETENTION_DEFAULT_DAYS=30
FILE_RETENTION_BUCKET_NAME=file-retention
```

### 参数说明

| 参数 | 说明 | 默认值 | 单位 |
|------|------|--------|------|
| `MINIO_DATA_VOLUME_PATH` | MinIO 数据卷路径 | `/data/minio` | - |
| `DISK_SPACE_WARNING_THRESHOLD` | 磁盘使用率警告阈值 | `80.0` | 百分比 |
| `DISK_SPACE_RECOVERY_TARGET` | 磁盘空间恢复目标 | `50.0` | 百分比 |
| `FILE_RETENTION_DEFAULT_DAYS` | 默认文件保留期限 | `30` | 天 |
| `FILE_RETENTION_BUCKET_NAME` | 文件保留存储桶名称 | `file-retention` | - |

### 配置示例

**示例场景**：
- 当前磁盘使用率：85%（超过 80% 警告阈值）
- 总磁盘空间：1000 GB
- 已使用空间：850 GB
- 恢复目标：50%

**计算过程**：
- 目标使用空间：1000 GB × 50% = 500 GB
- 需要释放空间：850 GB - 500 GB = **350 GB**

系统会按优先级删除保留文件，直到释放至少 350 GB 空间。

## 清理条件

文件必须满足以下所有条件才会被清理：

1. ✅ `is_retained == True` - 文件标记为保留状态
2. ✅ `file_storage_path IS NOT NULL` - 文件有存储路径
3. ✅ 文件在 MinIO 存储桶中存在 - 文件实际存在于 `file-retention` 桶中

**注意**：
- 如果文件标记为保留但 `file_storage_path` 为 NULL，该文件不会被清理
- 如果文件在数据库中标记为保留，但在 MinIO 中不存在，清理时会跳过并记录错误

## API 接口

### 1. 查询保留状态

**接口**：`GET /api/v1/file-ingestion/retention-status`

**权限**：所有登录用户

**返回数据**：
```json
{
  "retained_count": 100,              // 保留文件总数
  "retained_total_size": 1073741824,  // 保留文件总大小（字节）
  "retained_with_path_count": 95,     // 有存储路径的保留文件数
  "retained_with_path_size": 1024000000, // 有存储路径的文件总大小
  "retained_without_path_count": 5,   // 无存储路径的保留文件数
  "soon_expire_count": 10,            // 7天内即将到期的文件数
  "expired_count": 5                  // 已过期的文件数
}
```

### 2. 执行磁盘空间清理

**接口**：`POST /api/v1/file-ingestion/cleanup-old-files`

**权限**：仅超级用户（审计人员）

**功能**：清理旧文件，使磁盘空间恢复到目标值（50%）

**返回数据**：
```json
{
  "message": "清理完成，已释放 350.00 GB",
  "deleted_count": 25,                // 删除的文件数
  "freed_space": 375809638400,        // 释放的空间（字节）
  "freed_space_formatted": "350.00 GB", // 格式化后的空间大小
  "target_space": 375809638400,       // 目标释放空间（字节）
  "target_space_formatted": "350.00 GB",
  "errors": []                         // 错误列表（如果有）
}
```

### 3. 查询磁盘状态

**接口**：`GET /api/v1/file-ingestion/disk-status`

**权限**：所有登录用户

**返回数据**：
```json
{
  "total": 1073741824000,      // 总空间（字节）
  "used": 912680550400,        // 已使用空间（字节）
  "free": 161061273600,        // 可用空间（字节）
  "percentage": 85.0,          // 使用率（百分比）
  "warning_threshold": 80.0,   // 警告阈值
  "recovery_target": 50.0,     // 恢复目标
  "is_critical": true,         // 是否超过警告阈值
  "space_to_free": 375809638400 // 需要释放的空间（字节）
}
```

### 4. 保留诊断

**接口**：`GET /api/v1/file-ingestion/retention-diagnosis`

**权限**：仅超级用户

**功能**：诊断保留文件状态，包括数据库和 MinIO 的一致性检查

## 定时任务

### 任务配置

**任务ID**：`cleanup_expired_retention_files`

**任务名称**：`Cleanup Expired Retention Files`

**执行时间**：每天凌晨 2:00

**实现位置**：`backend/app/services/background_tasks.py:164-173`

```python
# 添加文件保留清理任务（每天凌晨 2 点）
scheduler.add_job(
    func=self._cleanup_expired_retention_files_task,
    trigger="cron",
    hour=2,
    minute=0,
    id="cleanup_expired_retention_files",
    name="Cleanup Expired Retention Files",
    replace_existing=True
)
```

### 任务执行流程

1. 获取所有已过期的保留文件
2. 删除 MinIO 中的文件对象
3. 更新数据库记录
4. 检查磁盘空间状态
5. 记录任务执行日志（包括成功/失败状态、执行时间、清理统计等）

### 任务日志

任务执行结果会记录到 `background_task_execution_logs` 表中，包括：
- 任务开始时间、完成时间、执行时长
- 删除的文件数、释放的空间大小
- 执行状态（running/success/failed）
- 错误信息（如果有）

## 清理流程详解

### 定期清理流程

```
定时任务触发（每天 2:00）
    ↓
查询已过期文件（retention_expires_at <= now()）
    ↓
遍历文件列表
    ↓
删除 MinIO 文件对象
    ↓
更新数据库记录
    ↓
检查磁盘空间状态
    ↓
记录任务执行日志
```

### 磁盘空间清理流程

```
检查磁盘使用率
    ↓
是否超过警告阈值（80%）？
    ├─ 否 → 无需清理
    └─ 是 → 计算需要释放的空间
            ↓
        按优先级查询保留文件
            ↓
        选择文件直到达到目标空间
            ↓
        删除 MinIO 文件对象
            ↓
        更新数据库记录
            ↓
        返回清理统计信息
```

## 常见问题

### 1. 为什么清理返回 0 字节？

**可能原因**：
- 没有保留文件（`is_retained = true` 的记录为 0）
- 保留文件没有存储路径（`file_storage_path IS NULL`）
- 所有保留文件已被清理
- 文件大小为 0 或 NULL

**排查方法**：
```sql
-- 检查保留文件数量
SELECT COUNT(*) FROM file_ingestion_records WHERE is_retained = true;

-- 检查有存储路径的保留文件
SELECT COUNT(*) FROM file_ingestion_records 
WHERE is_retained = true AND file_storage_path IS NOT NULL;

-- 检查有实际大小的保留文件
SELECT COUNT(*), SUM(file_size) FROM file_ingestion_records 
WHERE is_retained = true AND file_storage_path IS NOT NULL AND file_size > 0;
```

### 2. 清理后空间仍不足怎么办？

**可能原因**：
- 保留文件总大小不足以达到目标空间
- 部分文件删除失败（检查 `errors` 数组）
- MinIO 连接问题

**解决方案**：
- 检查清理返回的 `errors` 数组
- 验证 MinIO 连接状态
- 考虑调整 `DISK_SPACE_RECOVERY_TARGET` 参数
- 手动检查并清理 MinIO 中的孤立文件

### 3. 如何查看定时任务执行情况？

**方法**：
- 通过后台任务管理界面查看任务执行日志
- 查询数据库：`SELECT * FROM background_task_execution_logs WHERE task_id = 'cleanup_expired_retention_files' ORDER BY started_at DESC;`
- 查看应用日志中的任务执行记录

### 4. 文件保留期限如何设置？

**默认保留期限**：通过 `FILE_RETENTION_DEFAULT_DAYS` 配置（默认 30 天）

**单个文件保留期限**：上传文件时可通过 `retention_period_days` 参数指定，如果不指定则使用默认值

**计算过期时间**：
```python
retention_expires_at = uploaded_at + timedelta(days=retention_period_days)
```

## 数据库查询示例

### 查看保留文件统计

```sql
-- 保留文件总数和总大小
SELECT 
    COUNT(*) as total_count,
    SUM(file_size) as total_size,
    SUM(file_size) / 1024.0 / 1024.0 / 1024.0 as total_size_gb
FROM file_ingestion_records 
WHERE is_retained = true;
```

### 查看即将到期的文件

```sql
-- 7天内即将到期的文件
SELECT 
    id,
    file_name,
    file_size,
    uploaded_at,
    retention_expires_at,
    DATEDIFF(retention_expires_at, NOW()) as days_until_expiry
FROM file_ingestion_records 
WHERE is_retained = true 
    AND retention_expires_at IS NOT NULL
    AND retention_expires_at <= DATE_ADD(NOW(), INTERVAL 7 DAY)
    AND retention_expires_at > NOW()
ORDER BY retention_expires_at ASC;
```

### 查看已过期的文件

```sql
-- 已过期的保留文件
SELECT 
    id,
    file_name,
    file_size,
    uploaded_at,
    retention_expires_at
FROM file_ingestion_records 
WHERE is_retained = true 
    AND retention_expires_at IS NOT NULL
    AND retention_expires_at <= NOW()
ORDER BY retention_expires_at ASC;
```

### 查看最旧的保留文件（可能被优先清理）

```sql
-- 最旧的保留文件（按清理优先级排序）
SELECT 
    id,
    file_name,
    file_size,
    uploaded_at,
    retention_expires_at,
    file_storage_path
FROM file_ingestion_records 
WHERE is_retained = true 
    AND file_storage_path IS NOT NULL
ORDER BY 
    retention_expires_at ASC NULLS FIRST,
    uploaded_at ASC
LIMIT 20;
```

## 相关文件

- 清理逻辑实现：`backend/app/services/file_retention_service.py`
- 磁盘空间服务：`backend/app/services/disk_space_service.py`
- 定时任务配置：`backend/app/services/background_tasks.py`
- API 接口：`backend/app/api/v1/file_ingestion.py`
- 配置文件：`env.dev`, `env.staging`, `env.prod`

## 更新记录

- 2025-01-XX：创建文档，记录文件保留清理策略

