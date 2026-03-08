# 数据库同步脚本改进说明

## 问题描述

在执行 `./bin/sync-db-from-remote-to-local.sh` 脚本时，遇到以下问题：

1. **Collation Version 警告**：PostgreSQL 输出 collation version 不匹配警告
2. **脚本卡住**：警告信息输出到 stderr，导致脚本看起来卡住
3. **无进度反馈**：无法判断脚本是否正在运行
4. **执行速度慢**：309MB 数据库导出需要很长时间，没有使用压缩和并行导出

## 改进内容

### 1. 错误处理优化

**改进前**：
```bash
set -e  # 任何错误立即退出
```

**改进后**：
```bash
set -o pipefail  # 更灵活的错误处理，避免因警告而退出
```

### 2. pg_dump 命令改进

**改进前**：
```bash
docker exec -i "$DB_CONTAINER_NAME" \
    pg_dump "postgresql://..." \
    > "$TEMP_DIR/remote_dump.sql"
```

**改进后（第一次）**：
```bash
# 将 stderr 重定向到日志文件，stdout 重定向到 SQL 文件
# 添加 --no-owner --no-privileges 避免权限相关警告
docker exec -i "$DB_CONTAINER_NAME" \
    pg_dump \
    --no-owner \
    --no-privileges \
    --verbose \
    "postgresql://..." \
    > "$DUMP_OUTPUT" 2> "$DUMP_LOG" || DUMP_EXIT_CODE=$?
```

**改进后（第二次 - 性能优化）**：
```bash
# 使用自定义格式 + 压缩（注意：custom 格式不支持并行导出）
docker exec -i "$DB_CONTAINER_NAME" \
    pg_dump \
    --format=custom \
    --compress=9 \
    --no-owner \
    --no-privileges \
    --verbose \
    "postgresql://..." \
    > "$DUMP_OUTPUT" 2> "$DUMP_LOG" || DUMP_EXIT_CODE=$?
```

**关键改进点**：
- ✅ 将警告信息（stderr）重定向到日志文件，不影响主流程
- ✅ 添加 `--no-owner --no-privileges` 参数，减少权限相关警告
- ✅ 使用 `--verbose` 模式，便于调试
- ✅ 捕获退出码，进行错误处理
- ✅ **使用 `--format=custom` 支持压缩**
- ✅ **使用 `--compress=9` 最大压缩比**
- ⚠️ **注意：custom 格式不支持并行导出（`--jobs`），但支持并行恢复**

**格式选择说明**：
- `--format=custom`：单个文件，支持压缩，支持并行恢复，但不支持并行导出
- `--format=directory`：目录格式，支持压缩，支持并行导出和并行恢复
- 当前选择 custom 格式是为了简化文件管理（单个文件），虽然导出稍慢，但恢复时可以并行

### 3. 进度提示功能

**新增功能**：
- 在导出和导入过程中显示进度点（每2秒一个点）
- 完成后显示文件大小和警告统计

```bash
# 在后台显示进度提示
(
    while true; do
        echo -n "."
        sleep 2
    done
) &
PROGRESS_PID=$!
```

### 4. 警告信息处理

**新增功能**：
- 自动检测警告数量
- 显示主要警告信息（前5行）
- 保存完整警告日志到文件
- 警告不影响脚本执行

```bash
if [ -s "$DUMP_LOG" ]; then
    WARNING_COUNT=$(grep -c "WARNING" "$DUMP_LOG" 2>/dev/null || echo "0")
    if [ "$WARNING_COUNT" -gt 0 ]; then
        log_warning "导出过程中发现 $WARNING_COUNT 个警告（已记录到日志）"
        # 显示前几行警告信息
        head -n 5 "$DUMP_LOG" | sed 's/^/  /'
    fi
fi
```

### 5. 导入过程改进

**改进内容**：
- 添加进度提示
- 保存导入错误日志
- 改进错误处理

### 6. 日志文件管理

**改进内容**：
- 警告日志保存到临时目录
- 脚本结束时提示日志位置
- 可选择保留日志文件供后续分析

## 使用效果

### 改进前
```
ℹ️  导出远端数据库...
WARNING:  database "app" has a collation version mismatch
DETAIL:  The database was created using collation version 2.36, but the operating system provides version 2.41.
HINT:  Rebuild all objects in this database that use the default collation and run ALTER DATABASE app REFRESH COLLATION VERSION, or build PostgreSQL with the right library version.
[脚本看起来卡住]
```

### 改进后
```
ℹ️  导出远端数据库...
ℹ️  这可能需要几分钟时间，请耐心等待...
ℹ️  正在导出数据（警告信息将保存到日志文件）...
........
✅ 数据库导出完成，文件大小: 125M
⚠️  导出过程中发现 3 个警告（已记录到日志）
ℹ️  警告日志位置: /tmp/tmp.xxx/dump_warnings.log

主要警告信息：
  WARNING:  database "app" has a collation version mismatch
  DETAIL:  The database was created using collation version 2.36, but the operating system provides version 2.41.
  HINT:  Rebuild all objects in this database that use the default collation and run ALTER DATABASE app REFRESH COLLATION VERSION, or build PostgreSQL with the right library version.
  ... (更多警告信息请查看日志文件)
```

## 关于 Collation Version 警告

### 警告原因
- 远端数据库使用 collation version 2.36 创建
- 本地 PostgreSQL 提供 collation version 2.41
- 这是版本不匹配警告，**不影响数据导出和导入**

### 解决方案（可选）

如果需要消除警告，可以在远端数据库执行：

```sql
ALTER DATABASE app REFRESH COLLATION VERSION;
```

**注意**：这需要数据库管理员权限，且需要重建使用默认 collation 的对象。

### 建议
- ✅ **不需要降级操作系统版本**
- ✅ 警告不影响数据同步，可以忽略
- ✅ 如果警告过多，可以考虑在服务器端刷新 collation version

## 技术细节

### 错误处理策略
- 使用 `set -o pipefail` 而不是 `set -e`，允许更灵活的错误处理
- 捕获命令退出码，区分警告和错误
- 警告不影响脚本执行，只有真正的错误才会退出

### 日志管理
- 警告日志：`$TEMP_DIR/dump_warnings.log`
- 导入错误日志：`$TEMP_DIR/import_errors.log`
- 日志文件在脚本结束时提示位置，用户可选择保留

### 进度显示
- 使用后台进程显示进度点
- 每2秒输出一个点，让用户知道脚本正在运行
- 完成后自动清理进度进程

## 测试建议

1. **正常情况测试**：执行脚本，验证能正常同步
2. **警告处理测试**：确认警告信息被正确记录和显示
3. **错误处理测试**：模拟网络中断等错误，验证错误处理逻辑
4. **大数据库测试**：使用大型数据库测试进度显示功能

## 相关文件

- 脚本位置：`/home/huangpan/foundation/bin/sync-db-from-remote-to-local.sh`
- 本文档：`/home/huangpan/foundation/cursordocs/db-sync-script-improvement.md`

## 性能优化（第二次改进）

### 问题
- 309MB 数据库导出需要很长时间
- 未使用压缩，导出文件更大
- 未使用并行导出，无法充分利用多核 CPU
- 进度显示不够准确

### 优化方案

#### 1. 使用自定义格式 + 压缩
**改进前**：
```bash
pg_dump --no-owner --no-privileges --verbose "postgresql://..." > dump.sql
```

**改进后**：
```bash
pg_dump \
    --format=custom \
    --compress=9 \
    --jobs=4 \
    --no-owner \
    --no-privileges \
    --verbose \
    "postgresql://..." > dump.backup
```

**优势**：
- ✅ 自定义格式支持压缩，减少网络传输时间
- ✅ 压缩级别 9（最大压缩比）
- ✅ 并行导出（4个并行任务），充分利用多核 CPU
- ✅ 文件更小，传输更快

#### 2. 使用 pg_restore 并行导入
**改进前**：
```bash
psql -f dump.sql
```

**改进后**：
```bash
pg_restore \
    --jobs=4 \
    --no-owner \
    --no-privileges \
    --verbose \
    --host=localhost \
    --port=5434 \
    --username=postgres \
    --dbname=app \
    dump.backup
```

**优势**：
- ✅ 并行导入（4个并行任务），速度更快
- ✅ 支持自定义格式文件
- ✅ 更好的错误处理

#### 3. 改进进度显示
**新增功能**：
- 基于文件大小的进度显示
- 每3秒显示当前文件大小（MB）
- 更准确的进度反馈

```bash
# 显示格式：[50MB][100MB][150MB]...
```

### 性能提升预期

对于 309MB 数据库：
- **导出时间**：预计减少 30-50%（压缩，但无并行导出）
- **文件大小**：预计减少 60-80%（压缩）
- **导入时间**：预计减少 40-60%（并行导入，custom 格式支持并行恢复）

### 技术细节

1. **自定义格式（--format=custom）**：
   - 支持压缩
   - ⚠️ **不支持并行导出**（只支持并行恢复）
   - 文件格式为二进制，不是纯文本 SQL
   - 输出为单个文件，便于管理

2. **压缩级别（--compress=9）**：
   - 0-9，9 为最大压缩比
   - 压缩时间稍长，但文件更小
   - 适合网络传输
   - 显著减少文件大小（60-80%）

3. **并行恢复（--jobs=4）**：
   - custom 格式支持并行恢复
   - 使用 4 个并行任务
   - 适合多表数据库（59 个表）
   - 充分利用多核 CPU

4. **文件扩展名**：
   - 从 `.sql` 改为 `.backup`
   - 表示自定义格式文件

## 更新日期

- 第一次改进：2024-12-14（处理警告和进度显示）
- 第二次改进：2024-12-14（性能优化：压缩，custom 格式）
- 第三次修正：2024-12-14（移除并行导出，custom 格式不支持并行导出）












