# XCom调试日志查看指南

本文档说明如何查看App Node执行过程中的XCom解析调试日志。

## 日志标记

所有XCom相关的调试日志都包含以下标记：
- `[WF][XCom]` - Workflow执行过程中的XCom相关日志
- `[AirflowService]` - AirflowService中的XCom API调用日志

## 查看日志命令

### 1. 查看所有XCom相关日志（推荐）

```bash
docker logs foundation-backend-container 2>&1 | grep -E "\[WF\]\[XCom\]|\[AirflowService\]"
```

### 2. 查看特定DAG运行ID的XCom日志

```bash
docker logs foundation-backend-container 2>&1 | grep -E "\[WF\]\[XCom\]|\[AirflowService\].*manual__2025-12-15T07:16:19"
```

将 `manual__2025-12-15T07:16:19` 替换为实际的 `dag_run_id`。

### 3. 查看完整的Workflow执行日志（包含XCom）

```bash
docker logs foundation-backend-container 2>&1 | grep -E "\[WF\]\[.*\]"
```

### 4. 实时跟踪XCom日志

```bash
docker logs -f foundation-backend-container 2>&1 | grep -E "\[WF\]\[XCom\]|\[AirflowService\]"
```

### 5. 查看XCom解析错误日志

```bash
docker logs foundation-backend-container 2>&1 | grep -E "\[WF\]\[XCom\].*⚠️|\[WF\]\[XCom\].*❌|\[AirflowService\].*❌"
```

### 6. 查看最近的XCom日志（最后100行）

```bash
docker logs --tail 100 foundation-backend-container 2>&1 | grep -E "\[WF\]\[XCom\]|\[AirflowService\]"
```

### 7. 查看特定时间段的日志

```bash
docker logs --since "2025-12-15T07:20:00" --until "2025-12-15T07:25:00" foundation-backend-container 2>&1 | grep -E "\[WF\]\[XCom\]|\[AirflowService\]"
```

## 日志级别说明

### INFO级别日志（正常流程）
- ✅ 成功检索XCom数据
- 开始XCom检索
- XCom数据解析成功
- 找到markdown_conclusion

### WARNING级别日志（需要注意）
- ⚠️ 没有找到return_value条目
- ⚠️ 没有找到markdown_conclusion
- ⚠️ XCom数据解析失败但DAG已完成

### ERROR级别日志（错误）
- ❌ XCom API调用失败
- ❌ XCom数据解析异常
- ❌ 网络请求失败

## 关键日志点

### 1. XCom检索开始
```
[WF][XCom] Starting XCom retrieval for dag_run_id=xxx, task_id=stage_1_base_agents, dag_id=xxx
```

### 2. XCom API调用
```
[AirflowService] Requesting XCom data: dag_id=xxx, dag_run_id=xxx, task_id=xxx
[AirflowService] XCom API response: status_code=200, ...
```

### 3. XCom数据解析
```
[WF][XCom] XCom return_value retrieved: type=dict, is_none=False, is_dict=True
[WF][XCom] Parsed data is dict with keys: ['success', 'markdown_conclusion', ...]
```

### 4. 成功找到markdown_conclusion
```
[WF][XCom] ✅ Found markdown_conclusion: length=xxx, mode=xxx, success=True
```

### 5. 常见问题日志

**问题1：没有找到return_value**
```
[WF][XCom] ⚠️ No return_value entry found in XCom entries. Available keys: [...]
```

**问题2：XCom值解析失败**
```
[WF][XCom] ❌ Exception processing XCom data: ...
```

**问题3：API调用失败**
```
[AirflowService] ❌ Failed to get XCom data: status_code=404, ...
```

## 调试流程建议

1. **运行workflow后，立即查看日志**
   ```bash
   docker logs --tail 200 -f foundation-backend-container 2>&1 | grep -E "\[WF\]\[XCom\]|\[AirflowService\]"
   ```

2. **如果发现问题，查看完整的错误上下文**
   ```bash
   docker logs foundation-backend-container 2>&1 | grep -A 10 -B 5 "\[WF\]\[XCom\].*❌"
   ```

3. **检查XCom数据获取是否成功**
   ```bash
   docker logs foundation-backend-container 2>&1 | grep "\[AirflowService\].*Successfully retrieved XCom"
   ```

4. **检查解析流程**
   ```bash
   docker logs foundation-backend-container 2>&1 | grep -E "\[WF\]\[XCom\].*Parsed|\[WF\]\[XCom\].*Found markdown"
   ```

## 日志保存到文件

如果需要将日志保存到文件进行分析：

```bash
# 保存最近的XCom日志
docker logs foundation-backend-container 2>&1 | grep -E "\[WF\]\[XCom\]|\[AirflowService\]" > xcom_logs_$(date +%Y%m%d_%H%M%S).txt

# 保存特定DAG运行的日志
docker logs foundation-backend-container 2>&1 | grep -E "\[WF\]\[XCom\]|\[AirflowService\].*manual__2025-12-15T07:16:19" > xcom_logs_specific_run.txt
```

## 注意事项

1. 所有时间戳使用UTC时间
2. 日志可能包含敏感数据，分享前请检查
3. 如果容器名称不同，请替换 `foundation-backend-container` 为实际的容器名称
4. 某些日志可能很长，建议使用 `less` 或分页查看










