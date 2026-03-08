# 部署配置说明

## 概述

本文档说明如何在生产环境中正确配置后台任务系统和相关依赖。

## 环境变量配置

### 必需配置

以下配置在生产环境中是必需的：

```env
# 数据库配置
POSTGRES_SERVER=your_postgres_host
POSTGRES_PORT=5432
POSTGRES_USER=your_postgres_user
POSTGRES_PASSWORD=your_secure_password
POSTGRES_DB=your_database_name

# 应用配置
PROJECT_NAME=Foundation
SECRET_KEY=your_secure_secret_key
FIRST_SUPERUSER=admin@example.com
FIRST_SUPERUSER_PASSWORD=your_secure_password

# 环境设置
ENVIRONMENT=production
```

### 可选配置

以下配置是可选的，根据实际需求设置：

```env
# 后台任务配置（可选，有默认值）
BACKGROUND_TASKS_ENABLED=true
BACKGROUND_TASKS_SYNC_INTERVAL_SECONDS=30

# Azure OpenAI 配置（可选，用于NL2SQL功能）
AZURE_OPENAI_API_KEY=your_azure_openai_api_key
AZURE_OPENAI_ENDPOINT=https://your-resource.openai.azure.com/
AZURE_OPENAI_API_VERSION=2024-08-01-preview
AZURE_OPENAI_DEPLOYMENT=gpt-4o-mini

# Airflow 配置（可选，有默认值）
AIRFLOW_API_URL=http://airflow-webserver:8080
AIRFLOW_USERNAME=airflow
AIRFLOW_PASSWORD=airflow
AIRFLOW_DAG_ID=file_ingest_base_agent

# CORS 配置
BACKEND_CORS_ORIGINS=["http://localhost:3000", "https://yourdomain.com"]

# 邮件配置（可选）
SMTP_HOST=your_smtp_host
SMTP_PORT=587
SMTP_USER=your_smtp_user
SMTP_PASSWORD=your_smtp_password
EMAILS_FROM_EMAIL=noreply@yourdomain.com
EMAILS_FROM_NAME=Foundation

# Sentry 配置（可选）
SENTRY_DSN=https://your-sentry-dsn
```

## 部署场景

### 1. 基础部署（无 Azure OpenAI）

如果不需要 NL2SQL 功能，可以不配置 Azure OpenAI 相关设置：

```env
# 基础必需配置
POSTGRES_SERVER=your_postgres_host
POSTGRES_USER=your_postgres_user
POSTGRES_PASSWORD=your_secure_password
POSTGRES_DB=your_database_name
PROJECT_NAME=Foundation
SECRET_KEY=your_secure_secret_key
FIRST_SUPERUSER=admin@example.com
FIRST_SUPERUSER_PASSWORD=your_secure_password
ENVIRONMENT=production

# 后台任务配置
BACKGROUND_TASKS_ENABLED=true
BACKGROUND_TASKS_SYNC_INTERVAL_SECONDS=30
```

### 2. 完整功能部署

如果需要所有功能，包括 NL2SQL：

```env
# 基础配置
POSTGRES_SERVER=your_postgres_host
POSTGRES_USER=your_postgres_user
POSTGRES_PASSWORD=your_secure_password
POSTGRES_DB=your_database_name
PROJECT_NAME=Foundation
SECRET_KEY=your_secure_secret_key
FIRST_SUPERUSER=admin@example.com
FIRST_SUPERUSER_PASSWORD=your_secure_password
ENVIRONMENT=production

# 后台任务配置
BACKGROUND_TASKS_ENABLED=true
BACKGROUND_TASKS_SYNC_INTERVAL_SECONDS=30

# Azure OpenAI 配置
AZURE_OPENAI_API_KEY=your_azure_openai_api_key
AZURE_OPENAI_ENDPOINT=https://your-resource.openai.azure.com/
AZURE_OPENAI_API_VERSION=2024-08-01-preview
AZURE_OPENAI_DEPLOYMENT=gpt-4o-mini

# Airflow 配置
AIRFLOW_API_URL=http://airflow-webserver:8080
AIRFLOW_USERNAME=airflow
AIRFLOW_PASSWORD=airflow
AIRFLOW_DAG_ID=file_ingest_base_agent
```

## Docker 部署

### Docker Compose 配置

在 `docker-compose.yml` 中设置环境变量：

```yaml
version: '3.8'
services:
  backend:
    build: ./backend
    environment:
      # 基础配置
      - POSTGRES_SERVER=postgres
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=your_secure_password
      - POSTGRES_DB=foundation
      - PROJECT_NAME=Foundation
      - SECRET_KEY=your_secure_secret_key
      - FIRST_SUPERUSER=admin@example.com
      - FIRST_SUPERUSER_PASSWORD=your_secure_password
      - ENVIRONMENT=production
      
      # 后台任务配置
      - BACKGROUND_TASKS_ENABLED=true
      - BACKGROUND_TASKS_SYNC_INTERVAL_SECONDS=30
      
      # Azure OpenAI 配置（可选）
      - AZURE_OPENAI_API_KEY=${AZURE_OPENAI_API_KEY}
      - AZURE_OPENAI_ENDPOINT=${AZURE_OPENAI_ENDPOINT}
      
      # Airflow 配置
      - AIRFLOW_API_URL=http://airflow-webserver:8080
      - AIRFLOW_USERNAME=airflow
      - AIRFLOW_PASSWORD=airflow
      - AIRFLOW_DAG_ID=file_ingest_base_agent
    depends_on:
      - postgres
      - airflow-webserver
```

### 环境变量文件

创建 `.env` 文件：

```env
# 数据库配置
POSTGRES_PASSWORD=your_secure_password

# 应用配置
SECRET_KEY=your_secure_secret_key
FIRST_SUPERUSER_PASSWORD=your_secure_password

# Azure OpenAI 配置（可选）
AZURE_OPENAI_API_KEY=your_azure_openai_api_key
AZURE_OPENAI_ENDPOINT=https://your-resource.openai.azure.com/
```

## 功能验证

### 1. 后台任务状态检查

启动应用后，可以通过API检查后台任务状态：

```bash
curl -X GET "http://localhost:8001/api/v1/background-tasks/status"
```

预期响应：
```json
{
  "status": "running",
  "message": "Background task scheduler is running",
  "jobs": [
    {
      "id": "sync_file_status",
      "name": "Sync File Status",
      "next_run_time": "2024-01-01T12:00:30+00:00",
      "trigger": "interval[0:00:30]"
    }
  ]
}
```

### 2. 文件状态同步测试

上传一个文件并检查状态同步是否正常工作：

```bash
# 上传文件
curl -X POST "http://localhost:8001/api/v1/file-ingestion/upload" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -F "file=@test.pdf"

# 检查文件状态
curl -X GET "http://localhost:8001/api/v1/file-ingestion/files" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

### 3. NL2SQL 功能测试（如果配置了 Azure OpenAI）

```bash
curl -X POST "http://localhost:8001/api/v1/nl2sql/query" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"query": "查询所有用户"}'
```

## 故障排除

### 1. 配置验证错误

如果遇到配置验证错误，检查：

- 所有必需的环境变量是否已设置
- 环境变量值是否正确（特别是数据库连接信息）
- 密码和密钥是否足够安全（不是默认值）

### 2. 后台任务未启动

检查日志中的错误信息：

```bash
docker compose logs backend
```

常见问题：
- 数据库连接失败
- Airflow 服务不可达
- 配置参数错误

### 3. Azure OpenAI 配置问题

如果 NL2SQL 功能不可用，检查：

- Azure OpenAI API 密钥是否正确
- 端点 URL 是否正确
- 部署名称是否存在
- API 版本是否支持

### 4. 数据库连接问题

确保：
- PostgreSQL 服务正在运行
- 数据库凭据正确
- 网络连接正常
- 数据库已创建

## 安全建议

1. **密码安全**：使用强密码，避免默认值
2. **密钥管理**：使用环境变量或密钥管理服务存储敏感信息
3. **网络安全**：在生产环境中使用 HTTPS
4. **访问控制**：限制数据库和 API 的访问权限
5. **日志管理**：配置适当的日志级别和轮转策略

## 性能优化

1. **后台任务间隔**：根据实际需求调整同步间隔
2. **数据库连接池**：配置适当的连接池大小
3. **缓存策略**：考虑添加 Redis 缓存
4. **监控告警**：设置任务执行监控和告警 