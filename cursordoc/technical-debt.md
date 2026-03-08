# 技术债记录 / Technical Debt

本文档记录项目中已知的技术债，便于后续集中处理与排期。

---

## 1. AsyncConnectionPool 生命周期管理（方案 2 & 3）

**状态**: 待处理  
**优先级**: 中  
**相关组件**: `backend/app/services/workflow/workflow_execution_service.py`、Workbench 工作流执行

### 1.1 现象

- **Deprecation 警告**（来自 `psycopg_pool/pool_async.py`）：
  - 在构造函数中打开 async pool（`AsyncConnectionPool(..., open=True)`）已被弃用，未来版本将不再支持。
  - 建议改为：使用 `await pool.open()`，或使用上下文管理器 `async with AsyncConnectionPool(...) as pool:`。

- **资源未关闭导致的 asyncio 警告**（工作流执行失败或异常退出时）：
  - `Task was destroyed but it is pending!`（`pool-1-worker-0/1/2` 等）。
  - 原因：工作流任务在 `build_graph` 阶段就失败（如语法错误）或执行中途异常时，`WorkflowExecutionService` 已创建并 `await initialize_checkpoint()`，但 `close_checkpoint()` 未被调用，连接池及其 worker 任务未被正确关闭。

### 1.2 当前实现要点

- **创建方式**：`workflow_execution_service.py` 中在 `initialize_checkpoint()` 内：
  - 使用 `AsyncConnectionPool(conninfo=..., open=True)` 创建池；
  - 随后再 `await self._connection_pool.open()`（与弃用建议一致，但构造函数仍使用 `open=True`，触发弃用警告）。
- **关闭方式**：提供 `close_checkpoint()`，在池非空时 `await self._connection_pool.close()` 并置空。
- **调用关系**：`close_checkpoint()` 未在工作流任务执行路径上被保证调用；`execute_workflow_task`（`backend/app/api/v1/workbench.py`）在创建 `WorkflowExecutionService`、调用 `initialize_checkpoint()` 和 `build_graph()` 后，若发生异常或提前返回，没有在 `finally` 或类似路径中调用 `close_checkpoint()`，导致池泄漏。

### 1.3 建议方案（后续实现）

**方案 2：消除弃用用法**

- 创建 `AsyncConnectionPool` 时不再在构造函数中打开（去掉 `open=True` 或使用默认 `open=False`）。
- 仅通过 `await self._connection_pool.open()` 打开池；或改为使用 `async with AsyncConnectionPool(...) as pool:`，由上下文管理器负责 open/close。

**方案 3：保证池生命周期与任务绑定**

- 在 `execute_workflow_task`（及所有创建 `WorkflowExecutionService` 并调用 `initialize_checkpoint()` 的路径）中：
  - 使用 `try/finally` 或 `async with`，确保在任务结束（成功、失败、取消）时调用 `await execution_service.close_checkpoint()`。
- 可选：将“创建池 + 使用 + 关闭”封装为 `async with`（例如在 service 或上层提供 `async with execution_service.checkpoint_session():`），从 API 层统一只使用该入口，避免漏关。

### 1.4 涉及文件与位置

- `backend/app/services/workflow/workflow_execution_service.py`  
  - 约 94–105 行：`AsyncConnectionPool` 创建与 `open()` 调用；  
  - 158–172 行：`close_checkpoint()`。  
- `backend/app/api/v1/workbench.py`  
  - `execute_workflow_task`：在创建 `WorkflowExecutionService` 并初始化 checkpoint 后，需在退出路径（含异常）中调用 `close_checkpoint()`。

### 1.5 验收标准（后续完成时）

- 运行时不再出现 “opening the async pool ... in the constructor is deprecated” 类警告。
- 工作流执行失败或异常退出时，不再出现 “Task was destroyed but it is pending” 的 pool worker 任务警告。
- 所有创建并打开 `AsyncConnectionPool` 的代码路径，均在退出时调用 `close()` 或通过上下文管理器关闭。

---

*文档维护：在还清某项技术债后，可将对应条目标记为“已解决”并注明完成方式与 PR/commit。*
