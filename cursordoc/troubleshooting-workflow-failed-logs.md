# 排查Workflow Failed问题的Docker日志命令

## 后端容器名称

根据 `docker-compose.yml`，后端容器名称为：
- **开发环境**: `foundation-backend-container`
- **生产环境**: `foundation-backend-prod-container`
- **Staging环境**: `foundation-backend-staging-container`（如果存在）

---

## 基本日志查看命令

### 1. 查看实时日志（推荐）

```bash
# 开发环境
docker logs -f foundation-backend-container

# 生产环境
docker logs -f foundation-backend-prod-container
```

### 2. 查看最近100行日志

```bash
# 开发环境
docker logs --tail 100 foundation-backend-container

# 生产环境
docker logs --tail 100 foundation-backend-prod-container
```

### 3. 查看指定时间范围的日志

```bash
# 查看最近1小时的日志
docker logs --since 1h foundation-backend-container

# 查看指定时间之后的日志（ISO 8601格式）
docker logs --since "2025-01-01T10:00:00" foundation-backend-container
```

---

## 过滤Workflow Failed错误日志

### 1. 查看所有Workflow执行失败的日志

```bash
# 实时查看失败日志
docker logs -f foundation-backend-container 2>&1 | grep -i "Workflow execution failed"

# 查看历史失败日志（最近500行）
docker logs --tail 500 foundation-backend-container 2>&1 | grep -i "Workflow execution failed"
```

### 2. 查看详细的错误信息（包含节点状态）

```bash
# 查看包含详细节点状态信息的失败日志
docker logs --tail 1000 foundation-backend-container 2>&1 | grep -A 5 "\[WF\]\[.*\] Workflow execution failed"
```

### 3. 查看特定Task ID的日志

```bash
# 替换 YOUR_TASK_ID 为实际的task_id（例如：526090c7-1556-44c2-96e6-598d15ab8d6e）
TASK_ID="YOUR_TASK_ID"
docker logs --tail 1000 foundation-backend-container 2>&1 | grep "\[WF\]\[${TASK_ID}\]"
```

---

## 高级日志分析命令

### 1. 查看Workflow失败的完整上下文（包含前后日志）

```bash
# 查看失败前后各20行日志
docker logs --tail 500 foundation-backend-container 2>&1 | grep -B 20 -A 20 "Workflow execution failed"
```

### 2. 统计Workflow失败次数

```bash
# 统计最近1000行日志中的失败次数
docker logs --tail 1000 foundation-backend-container 2>&1 | grep -c "Workflow execution failed"
```

### 3. 查看包含pending HumanInTheLoop节点的失败日志

```bash
# 查看是否有pending的HumanInTheLoop节点的情况
docker logs --tail 1000 foundation-backend-container 2>&1 | grep -B 10 -A 10 "Has pending HumanInTheLoop nodes: true"
```

### 4. 查看特定错误类型的日志

```bash
# 查看数据库连接错误
docker logs --tail 1000 foundation-backend-container 2>&1 | grep -i "database connection error"

# 查看超时错误
docker logs --tail 1000 foundation-backend-container 2>&1 | grep -i "timeout"

# 查看节点执行错误
docker logs --tail 1000 foundation-backend-container 2>&1 | grep -i "node.*failed"
```

---

## 提取完整错误信息（包含节点状态）

### 使用脚本提取详细信息

创建一个临时脚本 `extract_workflow_errors.sh`：

```bash
#!/bin/bash
# 提取Workflow失败的完整信息

TASK_ID="${1:-}"  # 可选：指定task_id

if [ -z "$TASK_ID" ]; then
    # 如果没有指定task_id，显示所有失败的workflow
    echo "=== All Workflow Failures ==="
    docker logs --tail 2000 foundation-backend-container 2>&1 | \
        grep -B 5 -A 15 "\[WF\]\[.*\] Workflow execution failed"
else
    # 显示特定task_id的完整日志
    echo "=== Workflow Logs for Task ID: $TASK_ID ==="
    docker logs --tail 2000 foundation-backend-container 2>&1 | \
        grep "\[WF\]\[${TASK_ID}\]"
fi
```

使用方法：
```bash
# 赋予执行权限
chmod +x extract_workflow_errors.sh

# 查看所有失败
./extract_workflow_errors.sh

# 查看特定task_id的日志
./extract_workflow_errors.sh "526090c7-1556-44c2-96e6-598d15ab8d6e"
```

---

## 日志输出格式说明

根据我们添加的详细日志代码，失败日志会包含以下信息：

```
[WF][{task_id}] Workflow execution failed. 
Error: {error_message}, 
Error type: {exception_type}, 
Node results count: {count}, 
Node statuses: {node_id: status, ...}, 
Has pending HumanInTheLoop nodes: {true/false}, 
Task status before update: {status}, 
Current node ID: {node_id}
```

示例输出：
```
[WF][526090c7-1556-44c2-96e6-598d15ab8d6e] Workflow execution failed. 
Error: Connection timeout, 
Error type: TimeoutError, 
Node results count: 5, 
Node statuses: {'start': 'completed', 'app-1': 'completed', 'condition': 'completed', 'human-1': 'completed', 'end': 'pending'}, 
Has pending HumanInTheLoop nodes: false, 
Task status before update: running, 
Current node ID: end
```

---

## 快速诊断命令（一键执行）

```bash
# 组合命令：查看最近失败的workflow及其详细信息
echo "=== Recent Workflow Failures ===" && \
docker logs --tail 2000 foundation-backend-container 2>&1 | \
grep -B 3 -A 15 "\[WF\]\[.*\] Workflow execution failed" | \
tail -50
```

---

## 导出日志到文件

```bash
# 导出最近1000行日志到文件
docker logs --tail 1000 foundation-backend-container > workflow_logs.txt 2>&1

# 导出包含失败信息的日志到文件
docker logs --tail 2000 foundation-backend-container 2>&1 | \
grep -B 5 -A 15 "Workflow execution failed" > workflow_failures.txt
```

---

## 注意事项

1. **日志轮转**: Docker日志有大小限制（默认10MB，最多5个文件），旧的日志会被删除
2. **性能影响**: 使用 `-f` 实时查看日志会持续占用资源，查看完成后建议按 `Ctrl+C` 退出
3. **日志级别**: 确保后端日志级别设置为 `INFO` 或 `DEBUG` 才能看到详细的错误信息
4. **权限**: 确保有执行docker命令的权限

---

## 参考：检查日志级别的命令

```bash
# 检查容器环境变量中的日志级别
docker exec foundation-backend-container env | grep -i log

# 或者在容器内查看日志配置
docker exec foundation-backend-container cat /app/app/core/logging_config.py | grep -i level
```













