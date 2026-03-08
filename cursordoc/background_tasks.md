# 后台任务系统

## 概述

后台任务系统使用 APScheduler 库实现定时任务管理，主要用于自动同步文件处理状态，确保系统状态的实时性和准确性。

## 功能特性

### 1. 文件状态自动同步
- **频率**: 每30秒自动检查一次运行中的文件状态
- **功能**: 从 Airflow 获取 DAG 运行状态，更新本地数据库中的文件状态
- **状态映射**:
  - `success` → `completed`
  - `failed/error` → `failed`
  - `running/queued` → `running`

### 2. 任务管理API
- `GET /api/v1/background-tasks/status` - 获取后台任务状态
- `POST /api/v1/background-tasks/start` - 启动后台任务
- `POST /api/v1/background-tasks/stop` - 停止后台任务
- `POST /api/v1/background-tasks/restart` - 重启后台任务

## 配置选项

在 `.env` 文件中可以配置以下选项：

```env
# 启用/禁用后台任务
BACKGROUND_TASKS_ENABLED=true

# 文件状态同步间隔（秒）
BACKGROUND_TASKS_SYNC_INTERVAL_SECONDS=30
```

## 使用方法

### 自动启动
后台任务会在应用启动时自动启动，无需手动干预。

### 手动控制
可以通过API接口手动控制后台任务：

```bash
# 查看任务状态
curl -X GET "http://localhost:8001/api/v1/background-tasks/status" \
  -H "Authorization: Bearer YOUR_TOKEN"

# 启动任务
curl -X POST "http://localhost:8001/api/v1/background-tasks/start" \
  -H "Authorization: Bearer YOUR_TOKEN"

# 停止任务
curl -X POST "http://localhost:8001/api/v1/background-tasks/stop" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

### 测试脚本
运行测试脚本验证后台任务功能：

```bash
cd backend
python scripts/test_background_tasks.py
```

## 架构设计

### 核心组件

1. **BackgroundTaskService** (`app/services/background_tasks.py`)
   - 管理 APScheduler 调度器
   - 提供任务启动/停止接口
   - 实现文件状态同步逻辑

2. **API路由** (`app/api/routes/background_tasks.py`)
   - 提供任务管理REST API
   - 支持任务状态查询和控制

3. **应用集成** (`app/main.py`)
   - 应用启动时自动启动后台任务
   - 应用关闭时自动停止后台任务

### 数据流

```
[定时任务] → [查询运行中文件] → [调用Airflow API] → [更新数据库状态]
     ↓
[日志记录] ← [状态变更通知] ← [数据库提交]
```

## 优势

1. **可靠性**: 不依赖浏览器轮询，确保状态同步的可靠性
2. **实时性**: 30秒间隔的自动同步，保证状态更新的及时性
3. **可扩展性**: 基于APScheduler的架构，易于添加新的定时任务
4. **可配置性**: 支持通过环境变量配置任务参数
5. **监控性**: 提供完整的API接口用于任务监控和管理

## 注意事项

1. **数据库连接**: 每个任务执行时都会创建新的数据库连接，确保连接池的合理配置
2. **错误处理**: 单个文件同步失败不会影响其他文件的同步
3. **日志记录**: 所有操作都有详细的日志记录，便于问题排查
4. **资源管理**: 任务执行完成后会自动释放数据库连接

## 故障排除

### 常见问题

1. **任务未启动**
   - 检查 `BACKGROUND_TASKS_ENABLED` 配置
   - 查看应用启动日志

2. **同步失败**
   - 检查 Airflow 服务连接
   - 查看任务执行日志

3. **数据库连接问题**
   - 检查数据库连接配置
   - 确认连接池设置

### 日志位置
后台任务的日志会输出到应用的标准日志中，可以通过日志级别控制输出详细程度。 