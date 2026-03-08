# AsyncIOScheduler 介绍

## 概述

`AsyncIOScheduler` 是 APScheduler (Advanced Python Scheduler) 库中专门为异步 Python 应用设计的调度器。它基于 Python 的 `asyncio` 框架，允许在异步环境中调度和执行任务。

## 什么是 APScheduler？

APScheduler 是一个功能强大的 Python 任务调度库，支持多种调度器后端：

- **BlockingScheduler**: 阻塞式调度器，适用于单线程应用
- **BackgroundScheduler**: 后台调度器，使用独立线程
- **AsyncIOScheduler**: 异步调度器，适用于 asyncio 应用
- **GeventScheduler**: 适用于 Gevent 应用
- **TornadoScheduler**: 适用于 Tornado 应用
- **TwistedScheduler**: 适用于 Twisted 应用

## AsyncIOScheduler 的特点

### 1. 异步执行
- 基于 `asyncio` 事件循环
- 不会阻塞主线程
- 适合 FastAPI、aiohttp 等异步框架

### 2. 多种触发器类型
- **IntervalTrigger**: 间隔触发（如每 5 秒执行一次）
- **CronTrigger**: Cron 表达式触发（如每天凌晨 2 点）
- **DateTrigger**: 指定日期时间触发（如 2025-01-21 10:00:00）

### 3. 任务管理
- 添加、删除、暂停、恢复任务
- 任务持久化（可选）
- 任务执行历史记录

## 在我们的项目中的使用

### 项目位置

`backend/app/services/background_tasks.py`

### 使用场景

我们使用 `AsyncIOScheduler` 来管理后台定时任务：

1. **文件状态同步任务** - 定期从 Airflow 同步文件处理状态
2. **邮件队列处理任务** - 定期处理 Redis 中的邮件队列

### 代码示例

```python
from apscheduler.schedulers.asyncio import AsyncIOScheduler
from apscheduler.triggers.interval import IntervalTrigger

class BackgroundTaskService:
    def __init__(self):
        self.scheduler: Optional[AsyncIOScheduler] = None
        self._is_running = False
    
    def start(self):
        """启动调度器"""
        if self._is_running:
            return
        
        # 创建 AsyncIOScheduler 实例
        self.scheduler = AsyncIOScheduler()
        
        # 添加文件状态同步任务
        self.scheduler.add_job(
            func=self._sync_file_status_task,
            trigger=IntervalTrigger(seconds=30),
            id="sync_file_status",
            name="Sync File Status",
            replace_existing=True,
            max_instances=1
        )
        
        # 添加邮件队列处理任务
        self.scheduler.add_job(
            func=self._process_email_queue_task,
            trigger=IntervalTrigger(seconds=5.0),
            id="process_email_queue",
            name="Process Email Queue",
            replace_existing=True,
            max_instances=1
        )
        
        # 启动调度器
        self.scheduler.start()
        self._is_running = True
    
    def stop(self):
        """停止调度器"""
        if self.scheduler and self._is_running:
            self.scheduler.shutdown(wait=True)
            self._is_running = False
```

## 核心概念

### 1. 调度器 (Scheduler)

调度器是任务管理的核心，负责：
- 管理任务列表
- 触发任务执行
- 处理任务异常

```python
scheduler = AsyncIOScheduler()
```

### 2. 任务 (Job)

任务是要执行的函数或方法：

```python
def my_task():
    print("Task executed")

scheduler.add_job(
    func=my_task,
    trigger=IntervalTrigger(seconds=10)
)
```

### 3. 触发器 (Trigger)

触发器决定任务何时执行：

#### IntervalTrigger - 间隔触发

```python
from apscheduler.triggers.interval import IntervalTrigger

# 每 30 秒执行一次
trigger = IntervalTrigger(seconds=30)

# 每 5 分钟执行一次
trigger = IntervalTrigger(minutes=5)

# 每小时执行一次
trigger = IntervalTrigger(hours=1)
```

#### CronTrigger - Cron 表达式

```python
from apscheduler.triggers.cron import CronTrigger

# 每天凌晨 2 点执行
trigger = CronTrigger(hour=2, minute=0)

# 每周一上午 9 点执行
trigger = CronTrigger(day_of_week='mon', hour=9, minute=0)

# 每月 1 号执行
trigger = CronTrigger(day=1, hour=0, minute=0)
```

#### DateTrigger - 指定时间

```python
from apscheduler.triggers.date import DateTrigger
from datetime import datetime

# 在指定时间执行一次
trigger = DateTrigger(run_date=datetime(2025, 1, 21, 10, 0, 0))
```

### 4. 执行器 (Executor)

执行器负责实际执行任务。`AsyncIOScheduler` 默认使用 `asyncio` 执行器。

## 常用方法

### 添加任务

```python
scheduler.add_job(
    func=my_function,           # 要执行的函数
    trigger=IntervalTrigger(seconds=10),  # 触发器
    id="my_job_id",             # 任务 ID（唯一标识）
    name="My Job",              # 任务名称
    replace_existing=True,      # 如果 ID 已存在则替换
    max_instances=1,            # 最大并发实例数
    coalesce=True,              # 如果任务积压，是否合并执行
    misfire_grace_time=30       # 错过执行时间的宽限时间（秒）
)
```

### 删除任务

```python
# 通过 ID 删除
scheduler.remove_job("my_job_id")

# 删除所有任务
scheduler.remove_all_jobs()
```

### 暂停/恢复任务

```python
# 暂停任务
scheduler.pause_job("my_job_id")

# 恢复任务
scheduler.resume_job("my_job_id")

# 暂停所有任务
scheduler.pause()

# 恢复所有任务
scheduler.resume()
```

### 获取任务信息

```python
# 获取所有任务
jobs = scheduler.get_jobs()

# 获取特定任务
job = scheduler.get_job("my_job_id")

# 打印任务信息
for job in jobs:
    print(f"ID: {job.id}, Name: {job.name}, Next Run: {job.next_run_time}")
```

### 修改任务

```python
# 修改任务的触发器
scheduler.reschedule_job("my_job_id", trigger=IntervalTrigger(minutes=5))

# 修改任务的执行函数
scheduler.modify_job("my_job_id", func=new_function)
```

## 与 FastAPI 集成

### 在应用启动时启动调度器

```python
from fastapi import FastAPI
from app.services.background_tasks import background_task_service

app = FastAPI()

@app.on_event("startup")
async def startup_event():
    """应用启动时启动后台任务"""
    background_task_service.start()

@app.on_event("shutdown")
async def shutdown_event():
    """应用关闭时停止后台任务"""
    background_task_service.stop()
```

## 最佳实践

### 1. 任务函数设计

```python
# ✅ 好的做法：任务函数应该是独立的、幂等的
def sync_file_status_task():
    """同步文件状态"""
    try:
        # 执行任务逻辑
        pass
    except Exception as e:
        logger.error(f"Task failed: {e}")

# ❌ 不好的做法：任务函数依赖全局状态
global_state = {}

def bad_task():
    # 依赖全局状态，可能导致问题
    global_state['value'] = 1
```

### 2. 错误处理

```python
def my_task():
    try:
        # 任务逻辑
        pass
    except Exception as e:
        logger.error(f"Task error: {e}")
        # 不要抛出异常，避免影响调度器
```

### 3. 资源管理

```python
def database_task():
    db = get_db()
    try:
        # 使用数据库
        pass
    finally:
        db.close()  # 确保资源释放
```

### 4. 并发控制

```python
# 使用 max_instances 防止任务重叠执行
scheduler.add_job(
    func=long_running_task,
    trigger=IntervalTrigger(seconds=60),
    max_instances=1  # 确保同一时间只有一个实例运行
)
```

### 5. 任务持久化（可选）

如果需要任务持久化，可以使用数据库存储：

```python
from apscheduler.jobstores.sqlalchemy import SQLAlchemyJobStore

jobstores = {
    'default': SQLAlchemyJobStore(url='postgresql://...')
}

scheduler = AsyncIOScheduler(jobstores=jobstores)
```

## 常见问题

### Q1: AsyncIOScheduler 和 BackgroundScheduler 的区别？

**AsyncIOScheduler**:
- 基于 asyncio 事件循环
- 适合异步应用（FastAPI、aiohttp）
- 任务可以是异步函数

**BackgroundScheduler**:
- 使用独立线程
- 适合同步应用
- 任务必须是同步函数

### Q2: 如何执行异步任务？

```python
async def async_task():
    await some_async_operation()

# AsyncIOScheduler 可以直接执行异步函数
scheduler.add_job(
    func=async_task,
    trigger=IntervalTrigger(seconds=10)
)
```

### Q3: 如何传递参数给任务？

```python
def task_with_args(arg1, arg2):
    print(f"Args: {arg1}, {arg2}")

scheduler.add_job(
    func=task_with_args,
    args=['value1', 'value2'],
    trigger=IntervalTrigger(seconds=10)
)
```

### Q4: 如何动态添加/删除任务？

```python
# 动态添加
scheduler.add_job(
    func=dynamic_task,
    trigger=IntervalTrigger(seconds=30),
    id="dynamic_job"
)

# 动态删除
scheduler.remove_job("dynamic_job")
```

### Q5: 任务执行时间过长怎么办？

```python
# 使用 max_instances=1 防止任务重叠
# 如果任务执行时间超过间隔时间，会跳过下一次执行
scheduler.add_job(
    func=long_task,
    trigger=IntervalTrigger(seconds=60),
    max_instances=1,
    coalesce=True  # 如果积压，只执行一次
)
```

## 性能考虑

1. **任务执行时间**: 确保任务执行时间短于触发间隔
2. **并发控制**: 使用 `max_instances` 限制并发
3. **资源清理**: 任务完成后及时释放资源
4. **错误处理**: 避免任务异常影响调度器

## 监控和调试

### 查看任务状态

```python
# 获取所有任务
jobs = scheduler.get_jobs()
for job in jobs:
    print(f"""
    ID: {job.id}
    Name: {job.name}
    Next Run: {job.next_run_time}
    Last Run: {job.next_run_time  # 需要配置 jobstore 才能获取
    """)
```

### 日志配置

```python
import logging

logging.basicConfig()
logging.getLogger('apscheduler').setLevel(logging.DEBUG)
```

## 总结

`AsyncIOScheduler` 是一个强大的异步任务调度工具，特别适合 FastAPI 等异步框架。在我们的项目中，它用于：

1. **文件状态同步** - 每 30 秒同步一次 Airflow 状态
2. **邮件队列处理** - 每 5 秒处理一次邮件队列

通过合理使用 `AsyncIOScheduler`，我们可以实现可靠的后台任务调度，而不会阻塞主应用。

## 参考资源

- [APScheduler 官方文档](https://apscheduler.readthedocs.io/)
- [AsyncIOScheduler API 文档](https://apscheduler.readthedocs.io/en/stable/modules/schedulers/asyncio.html)
- [触发器文档](https://apscheduler.readthedocs.io/en/stable/modules/triggers/index.html)

