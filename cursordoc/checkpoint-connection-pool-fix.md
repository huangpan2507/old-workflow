# Checkpoint 连接池配置修复

## 修复日期
2024-12-19

## 问题描述

### 原始问题
1. **连接池配置不可用**：`PoolConfig` 导入失败，导致使用默认连接池配置
2. **连接超时**：默认配置在长时间运行的工作流中导致连接被关闭
3. **错误日志**：
   ```
   [WF] PoolConfig not available, using default connection pool settings. 
   This may cause connection timeout issues during long-running workflows.
   ```
4. **实际影响**：在执行过程中出现 `OperationalError: the connection is closed`

## 修复方案（最终版本）

### 重要发现

经过深入调查，发现：
1. **`AsyncPostgresSaver.from_conn_string()` 不支持 `pool_config` 参数**
2. **`psycopg_pool` 没有 `PoolConfig` 类**，而是使用 `AsyncConnectionPool` 类
3. **`AsyncPostgresSaver` 可以直接接受 `AsyncConnectionPool` 实例作为构造参数**

### 最终修复方案

使用 `AsyncConnectionPool` 创建连接池，然后直接传递给 `AsyncPostgresSaver` 构造函数。

### 1. 改进导入策略

**修复前**：
```python
try:
    from psycopg_pool import PoolConfig
    POOL_CONFIG_AVAILABLE = True
except ImportError:
    try:
        from langgraph.checkpoint.postgres.aio import PoolConfig
        POOL_CONFIG_AVAILABLE = True
    except ImportError:
        PoolConfig = None
        POOL_CONFIG_AVAILABLE = False
```

**修复后**：
```python
# Import AsyncConnectionPool for connection pool configuration
# AsyncPostgresSaver can accept AsyncConnectionPool directly as constructor parameter
POOL_AVAILABLE = False
AsyncConnectionPool = None

try:
    from psycopg_pool import AsyncConnectionPool
    POOL_AVAILABLE = True
    logger.info("[WF] Successfully imported AsyncConnectionPool from psycopg_pool")
except ImportError:
    logger.warning(
        "[WF] AsyncConnectionPool not available. Connection pool configuration will be skipped. "
        "This may cause connection timeout issues during long-running workflows. "
        "Please install psycopg-pool: pip install psycopg-pool>=3.1.0"
    )
```

**改进点**：
- 使用 `AsyncConnectionPool` 而不是不存在的 `PoolConfig`
- 添加了详细的日志记录，便于诊断导入问题
- 提供了明确的安装指导

### 2. 增强连接池初始化逻辑

**修复前**：
```python
pool_config = None
if POOL_CONFIG_AVAILABLE and PoolConfig:
    pool_config = PoolConfig(...)
    checkpointer_cm = AsyncPostgresSaver.from_conn_string(
        db_url,
        pool_config=pool_config  # ❌ 不支持此参数
    )
else:
    checkpointer_cm = AsyncPostgresSaver.from_conn_string(db_url)
```

**修复后**：
```python
# Strategy 1: Use AsyncConnectionPool if available (preferred method)
if POOL_AVAILABLE and AsyncConnectionPool is not None:
    try:
        # Create connection pool with optimized settings for long-running workflows
        self._connection_pool = AsyncConnectionPool(
            conninfo=normalized_db_url,
            min_size=2,          # Minimum connections in pool
            max_size=10,         # Maximum connections in pool
            max_lifetime=3600,   # Connection max lifetime: 1 hour (3600 seconds)
            max_idle=600,       # Connection max idle time: 10 minutes (600 seconds)
            open=True            # Open connections immediately
        )
        
        # Open the connection pool
        await self._connection_pool.open()
        logger.info("[WF] AsyncConnectionPool opened successfully")
        
        # Initialize AsyncPostgresSaver with the connection pool
        # AsyncPostgresSaver can accept AsyncConnectionPool directly as constructor parameter
        self.checkpoint = AsyncPostgresSaver(self._connection_pool)
        await self.checkpoint.setup()  # Create necessary tables
        
        logger.info(
            "[WF] ✅ Checkpoint initialized successfully with AsyncConnectionPool. "
            "Connection pool configuration applied to prevent timeout during long-running workflows."
        )
    except Exception as pool_error:
        logger.error(
            f"[WF] Failed to create AsyncConnectionPool: {pool_error}. "
            "Falling back to default connection pool settings.",
            exc_info=True
        )
        # Fall back to default method
        checkpointer_cm = AsyncPostgresSaver.from_conn_string(normalized_db_url)
        self.checkpoint = await checkpointer_cm.__aenter__()
        await self.checkpoint.setup()
else:
    # Fall back to default (no pool configuration)
    logger.warning(
        "[WF] AsyncConnectionPool not available. "
        "Using default connection pool settings. "
        "This may cause connection timeout issues during long-running workflows. "
        "Please install psycopg-pool: pip install psycopg-pool>=3.1.0"
    )
    checkpointer_cm = AsyncPostgresSaver.from_conn_string(normalized_db_url)
    self.checkpoint = await checkpointer_cm.__aenter__()
    await self.checkpoint.setup()
```

**改进点**：
1. **使用正确的 API**：直接使用 `AsyncConnectionPool` 创建连接池，然后传递给 `AsyncPostgresSaver` 构造函数
2. **连接池生命周期管理**：在 `__init__` 中存储连接池引用，提供 `close_checkpoint()` 方法用于清理
3. **错误处理**：添加了完整的错误处理，确保即使连接池创建失败也能回退到默认配置
4. **数据库 URL 规范化**：自动处理 `postgresql+psycopg2://` 前缀
5. **详细日志**：记录每个步骤的成功或失败，便于调试

### 3. 连接池参数配置

**配置参数**：
- `min_size=2`：最小连接池大小，确保始终有可用连接
- `max_size=10`：最大连接池大小，支持并发工作流执行
- `max_lifetime=3600`：连接最大生命周期（1小时），确保连接定期刷新
- `max_idle=600`：连接最大空闲时间（10分钟），在长时间运行的工作流中保持连接活跃
- `open=True`：立即打开连接，避免延迟

## 修复效果

### 预期效果
1. **连接池正确配置**：如果 `psycopg-pool` 已安装，连接池将使用上述参数配置
2. **连接保持活跃**：在长时间运行的工作流中，连接不会因超时而关闭
3. **使用正确的 API**：直接使用 `AsyncConnectionPool` 和 `AsyncPostgresSaver` 构造函数，而不是不支持的 `pool_config` 参数
4. **错误恢复**：即使配置失败，也能回退到默认配置，不会导致工作流执行失败
5. **详细日志**：便于诊断和调试连接池问题
6. **资源管理**：提供 `close_checkpoint()` 方法用于清理连接池资源

### 验证方法
1. **检查日志**：查看是否有 `[WF] Successfully initialized AsyncPostgresSaver with PoolConfig` 日志
2. **测试长时间运行的工作流**：执行包含多个 App Node 的长时间工作流，确认连接不会断开
3. **监控连接状态**：观察是否还有 `OperationalError: the connection is closed` 错误

## 依赖要求

### 必需依赖
- `psycopg-pool>=3.1.0`：提供 `PoolConfig` 类

### 已包含在项目依赖中
根据 `backend/pyproject.toml`：
```toml
"psycopg-pool>=3.1.0",  # Connection pool with PoolConfig class (required for AsyncPostgresSaver)
```

### 安装命令
```bash
pip install psycopg-pool>=3.1.0
```

或使用项目依赖管理：
```bash
cd backend
uv sync  # 如果使用 uv
# 或
pip install -r requirements.txt  # 如果使用 requirements.txt
```

## 相关文件

- `backend/app/services/workflow/workflow_execution_service.py`：主要修复文件
- `backend/pyproject.toml`：依赖声明
- `cursordocs/checkpoint-analysis-and-risks.md`：问题分析和风险评估

## 后续优化建议

1. **监控连接池状态**：添加连接池使用情况的监控和告警
2. **动态配置**：根据工作流负载动态调整连接池大小
3. **连接池健康检查**：定期检查连接池健康状态，自动恢复失效连接
4. **性能测试**：进行长时间运行的工作流压力测试，验证连接池配置的有效性

## 注意事项

1. **依赖安装**：确保 `psycopg-pool>=3.1.0` 已正确安装
2. **数据库 URL 格式**：代码会自动处理 `postgresql+psycopg2://` 前缀，但建议使用标准的 `postgresql://` 格式
3. **回退机制**：即使连接池配置失败，代码也会回退到默认配置，不会导致工作流执行失败
4. **日志监控**：定期检查日志，确认连接池配置是否成功应用



