# Airflow 集成文档

本文档描述了后端与 Apache Airflow 的集成实现，用于文件上传后自动触发数据处理工作流。

## 🏗️ 架构概述

```
用户上传文件 → FastAPI 后端 → 保存文件记录 → 触发 Airflow DAG → 数据处理工作流
```

## 📋 功能特性

### ✅ 已实现功能

1. **文件上传触发 DAG**
   - 支持 Word (.doc, .docx) 和 PDF (.pdf) 文件
   - 自动计算文件哈希值，防止重复上传
   - 上传成功后自动触发 Airflow DAG 运行

2. **DAG 状态管理**
   - 实时获取 DAG 运行状态
   - 支持 DAG 重试功能
   - 错误处理和状态更新

3. **API 端点**
   - `POST /api/v1/file-ingestion/upload` - 文件上传
   - `GET /api/v1/file-ingestion/dag-status/{record_id}` - 获取 DAG 状态
   - `POST /api/v1/file-ingestion/retry-dag/{record_id}` - 重试 DAG 运行

## 🔧 配置说明

### 环境变量配置

在 `.env` 文件中添加以下配置：

```bash
# Airflow 配置
AIRFLOW_API_URL=http://airflow-webserver:8080
AIRFLOW_USERNAME=airflow
AIRFLOW_PASSWORD=airflow
AIRFLOW_DAG_ID=file_ingest_base_agent
```

**禁用 Airflow（未部署 Airflow 时）**：若仅运行主 docker-compose 而未启动 Airflow 栈，可将 `AIRFLOW_API_URL` 设为空以禁用维度分析/审批打标等依赖 Airflow 的逻辑，避免 `airflow-webserver` 解析失败和连接错误，审批流程仍会正常完成：

```bash
AIRFLOW_API_URL=
```

### 依赖安装

确保已安装 `requests` 依赖：

```bash
# 使用 uv (推荐)
uv add requests

# 或使用 pip
pip install requests
```

## 🚀 使用流程

### 1. 启动 Airflow 服务

```bash
# 进入 airflow 目录
cd ../airflow

# 启动 Airflow 服务（EIM 版本）
./bin/start-dev-eim.sh
# 或使用通用版本
# ./bin/start-dev-generic.sh

# 检查服务状态
./bin/status.sh
```

### 2. 测试连接

```bash
# 进入 backend 目录
cd ../backend

# 运行连接测试
python scripts/test_airflow_connection.py
```

### 3. 文件上传流程

1. **用户上传文件** → 前端调用 `/api/v1/file-ingestion/upload`
2. **后端处理** → 保存文件记录，触发 Airflow DAG
3. **状态更新** → 文件记录状态更新为 "running"
4. **监控状态** → 前端可以查询 DAG 运行状态

## 📊 API 接口详情

### 文件上传

```http
POST /api/v1/file-ingestion/upload
Content-Type: multipart/form-data

file: [文件内容]
```

**响应示例：**
```json
{
  "id": 1,
  "file_name": "document.pdf",
  "file_hash": "abc123...",
  "source": "web",
  "uploaded_by": "user@example.com",
  "uploaded_at": "2024-01-01T12:00:00Z",
  "dag_run_id": "file_upload_1_1234567890",
  "status": "running",
  "status_updated_at": "2024-01-01T12:00:01Z",
  "error_message": null
}
```

### 获取 DAG 状态

```http
GET /api/v1/file-ingestion/dag-status/{record_id}
```

**响应示例：**
```json
{
  "record_id": 1,
  "dag_run_id": "file_upload_1_1234567890",
  "dag_status": {
    "dag_run_id": "file_upload_1_1234567890",
    "dag_id": "file_ingest_base_agent",
    "state": "running",
    "start_date": "2024-01-01T12:00:01Z",
    "end_date": null
  },
  "record_status": "running",
  "last_updated": "2024-01-01T12:00:01Z"
}
```

### 重试 DAG 运行

```http
POST /api/v1/file-ingestion/retry-dag/{record_id}
```

**响应示例：**
```json
{
  "message": "DAG 重试成功",
  "dag_run_id": "file_upload_1_1234567891",
  "status": "running"
}
```

## 🔍 错误处理

### 常见错误及解决方案

1. **Airflow 服务不可用**
   ```
   错误: Airflow service is not healthy
   解决: 检查 Airflow 服务是否启动，网络连接是否正常
   ```

2. **DAG 触发失败**
   ```
   错误: Failed to trigger DAG run
   解决: 检查 DAG ID 是否正确，认证信息是否有效
   ```

3. **文件重复上传**
   ```
   错误: 该文件已经上传过，请勿重复上传
   解决: 检查文件哈希值，或使用不同的文件
   ```

### 状态说明

- **pending**: 文件已上传，等待处理
- **running**: DAG 正在运行中
- **completed**: 处理完成
- **failed**: 处理失败

## 🧪 测试

### 运行测试脚本

```bash
# 测试 Airflow 连接
python scripts/test_airflow_connection.py
```

### 手动测试

1. **启动服务**
   ```bash
   # 启动后端服务
   uvicorn app.main:app --reload --host 0.0.0.0 --port 8001
   
   # 启动 Airflow（EIM 版本）
   cd ../airflow && ./bin/start-dev-eim.sh
   # 或使用通用版本
   # cd ../airflow && ./bin/start-dev-generic.sh
   ```

2. **上传文件测试**
   - 访问 http://localhost:5173/data-ingestor/upload
   - 上传一个 Word 或 PDF 文件
   - 检查后端日志和 Airflow Web UI

3. **查看处理状态**
   - 访问 http://localhost:5173/data-ingestor/dashboard
   - 查看文件处理状态

## 🔧 故障排除

### 日志查看

```bash
# 后端日志
docker logs backend-1

# Airflow 日志
cd ../airflow
docker logs airflow-webserver-1
docker logs airflow-scheduler-1
```

### 网络连接测试

```bash
# 测试 Airflow API 连接
curl -u airflow:airflow http://localhost:9090/health
```

### 配置验证

```bash
# 检查环境变量
echo $AIRFLOW_API_URL
echo $AIRFLOW_USERNAME
echo $AIRFLOW_PASSWORD
```

## 📝 注意事项

1. **文件大小限制**: 建议文件大小不超过 50MB
2. **文件类型**: 仅支持 .doc, .docx, .pdf 格式
3. **网络要求**: 后端需要能够访问 Airflow Web Server
4. **认证配置**: 确保 Airflow 用户名和密码正确
5. **DAG 配置**: 确保 `file_ingest_base_agent` DAG 已部署到 Airflow

## 🔄 更新日志

- **v1.0.0**: 初始实现，支持基本的文件上传和 DAG 触发
- **v1.1.0**: 添加 DAG 状态查询和重试功能
- **v1.2.0**: 改进错误处理和日志记录 