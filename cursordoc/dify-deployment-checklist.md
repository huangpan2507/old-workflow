# Dify集成部署检查清单

## 🚀 部署前检查

### 1. 环境配置 ✅

- [ ] 在 `.env` 文件中添加所有必要的Dify配置项
- [ ] 获取并配置Dify API密钥 (`DIFY_API_KEY`)
- [ ] 配置Dify应用ID (`DIFY_POLICY_APP_ID`, `DIFY_ANALYSIS_APP_ID`, `DIFY_DEFAULT_APP_ID`)
- [ ] 设置API URL (`DIFY_API_URL`)
- [ ] 配置超时和重试参数

### 2. 数据库准备 ✅

- [ ] 创建 `dify_policy_results` 表
- [ ] 创建 `dify_policy_workflow_results` 表
- [ ] 创建 `policy_requirements` 表 (可选)
- [ ] 创建 `policy_documents` 表 (可选)
- [ ] 验证数据库连接权限

### 3. 依赖安装 ✅

- [ ] 安装 `httpx` (异步HTTP客户端)
- [ ] 安装 `aiohttp` (用于Airflow集成)
- [ ] 验证现有依赖兼容性

### 4. Dify平台准备 ✅

- [ ] 在Dify平台创建策略分析应用 (Agent类型)
- [ ] 在Dify平台创建策略生成应用 (Workflow类型)  
- [ ] 在Dify平台创建策略咨询应用 (Chatbot类型)
- [ ] 获取各应用的App ID
- [ ] 测试应用功能正常

## 🔧 部署步骤

### 1. 代码部署

```bash
# 确保所有新文件已添加到项目中
cd /home/huangpan/foundation

# 检查文件结构
ls -la backend/app/core/dify_config.py
ls -la backend/app/services/dify_policy_service.py
ls -la backend/app/api/routes/dify_policy_routes.py
ls -la airflow/dags/dify_policy_workflow.py
ls -la airflow/dags/agent_flows/dify_policy_integration.py
```

### 2. 环境变量配置

```bash
# 在 .env 文件中添加以下配置
cat >> .env << EOF

# =============================================================================
# Dify集成配置
# =============================================================================

# Dify平台基础配置
DIFY_API_URL=https://api.dify.ai/v1
DIFY_API_KEY=your-dify-api-key-here
DIFY_TIMEOUT=30
DIFY_RETRY_COUNT=3
DIFY_ENABLED=true

# Dify应用配置
DIFY_DEFAULT_APP_ID=app-xxxxxxxxxx
DIFY_POLICY_APP_ID=app-yyyyyyyyyy
DIFY_ANALYSIS_APP_ID=app-zzzzzzzzzz

# Dify功能开关
DIFY_ENABLE_STREAMING=true
DIFY_ENABLE_CONVERSATION=true
DIFY_AUTO_GENERATE_NAME=true

# Dify高级配置
DIFY_MAX_TOKENS=4000
DIFY_TEMPERATURE=0.7
DIFY_TOP_P=0.9

# Dify缓存配置
DIFY_CACHE_TTL=3600
DIFY_USE_CACHE=true
EOF
```

### 3. 数据库迁移

```sql
-- 连接到PostgreSQL数据库并执行以下SQL

-- Dify策略处理结果表
CREATE TABLE IF NOT EXISTS dify_policy_results (
    id SERIAL PRIMARY KEY,
    area VARCHAR(100) NOT NULL,
    task_type VARCHAR(50) NOT NULL,
    result_data JSONB NOT NULL,
    conversation_id VARCHAR(100),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    status VARCHAR(20) DEFAULT 'pending',
    INDEX idx_dify_results_area_type (area, task_type),
    INDEX idx_dify_results_created (created_at DESC)
);

-- Dify工作流结果表
CREATE TABLE IF NOT EXISTS dify_policy_workflow_results (
    id SERIAL PRIMARY KEY,
    area VARCHAR(100) NOT NULL,
    workflow_run_id VARCHAR(100) NOT NULL,
    analysis_result JSONB,
    recommendation_result JSONB,
    consolidated_result JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    status VARCHAR(20) DEFAULT 'processing',
    UNIQUE(area, workflow_run_id),
    INDEX idx_workflow_results_area (area),
    INDEX idx_workflow_results_status (status)
);

-- 策略需求表(可选)
CREATE TABLE IF NOT EXISTS policy_requirements (
    id SERIAL PRIMARY KEY,
    area VARCHAR(100) NOT NULL,
    requirement_text TEXT NOT NULL,
    priority INTEGER DEFAULT 1,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    INDEX idx_policy_req_area_active (area, is_active)
);

-- 策略文档表(可选)
CREATE TABLE IF NOT EXISTS policy_documents (
    id SERIAL PRIMARY KEY,
    area VARCHAR(100) NOT NULL,
    document_content TEXT NOT NULL,
    status VARCHAR(20) DEFAULT 'active',
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    INDEX idx_policy_docs_area_status (area, status)
);
```

### 4. 服务重启

```bash
# 重启后端服务
cd /home/huangpan/foundation/backend
uvicorn app.main:app --reload

# 重启Airflow服务 (如果使用Docker)
docker-compose restart airflow-webserver
docker-compose restart airflow-scheduler
```

## ✅ 部署后验证

### 1. 基础功能测试

```bash
# 1. 健康检查
curl -X GET "http://localhost:8000/api/v1/dify/health"

# 2. 应用列表
curl -X GET "http://localhost:8000/api/v1/dify/apps" \
  -H "Authorization: Bearer your-jwt-token"

# 3. 简单聊天测试
curl -X POST "http://localhost:8000/api/v1/dify/chat" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your-jwt-token" \
  -d '{
    "message": "Hello, this is a test message"
  }'
```

### 2. 策略功能测试

```bash
# 策略文档分析测试
curl -X POST "http://localhost:8000/api/v1/dify/analyze" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your-jwt-token" \
  -d '{
    "document_content": "This is a test policy document for analysis.",
    "analysis_type": "summary"
  }'

# 策略建议生成测试
curl -X POST "http://localhost:8000/api/v1/dify/recommend" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your-jwt-token" \
  -d '{
    "context": "Test policy context",
    "requirements": ["Test requirement 1", "Test requirement 2"],
    "policy_type": "general"
  }'
```

### 3. Airflow工作流测试

```bash
# 触发Dify策略工作流
curl -X POST "http://localhost:8080/api/v1/dags/dify_policy_workflow/dagRuns" \
  -H "Content-Type: application/json" \
  -H "Authorization: Basic $(echo -n 'airflow:airflow' | base64)" \
  -d '{
    "conf": {
      "area": "test_area",
      "document_content": "This is a test policy document for workflow testing."
    }
  }'

# 检查工作流状态
curl -X GET "http://localhost:8080/api/v1/dags/dify_policy_workflow/dagRuns" \
  -H "Authorization: Basic $(echo -n 'airflow:airflow' | base64)"
```

### 4. 数据库验证

```sql
-- 检查结果表是否有数据写入
SELECT COUNT(*) FROM dify_policy_results;
SELECT COUNT(*) FROM dify_policy_workflow_results;

-- 查看最新的处理结果
SELECT * FROM dify_policy_results ORDER BY created_at DESC LIMIT 5;
SELECT * FROM dify_policy_workflow_results ORDER BY created_at DESC LIMIT 5;
```

## 🔍 监控设置

### 1. 日志监控

```bash
# 设置日志轮转
sudo logrotate -d /etc/logrotate.d/foundation

# 监控关键日志
tail -f /home/huangpan/foundation/backend/logs/dify.log
tail -f /home/huangpan/foundation/airflow/logs/dags/dify_policy_workflow/
```

### 2. 性能监控

- [ ] 配置Dify API调用监控
- [ ] 设置响应时间告警
- [ ] 监控工作流执行状态
- [ ] 跟踪错误率和成功率

### 3. 资源监控

- [ ] 监控内存使用情况
- [ ] 跟踪数据库连接数
- [ ] 监控磁盘空间使用

## 🚨 常见问题排查

### 1. API连接问题

```bash
# 测试网络连接
curl -v https://api.dify.ai/v1/info

# 检查DNS解析
nslookup api.dify.ai

# 验证SSL证书
openssl s_client -connect api.dify.ai:443 -servername api.dify.ai
```

### 2. 认证问题

```bash
# 验证API密钥格式
echo $DIFY_API_KEY | wc -c  # 应该是合理长度

# 测试API密钥有效性
curl -H "Authorization: Bearer $DIFY_API_KEY" https://api.dify.ai/v1/info
```

### 3. 数据库问题

```sql
-- 检查表是否存在
\dt dify_*

-- 检查表结构
\d dify_policy_results

-- 检查权限
SELECT * FROM information_schema.table_privileges 
WHERE table_name LIKE 'dify_%';
```

## 📋 维护任务

### 日常维护

- [ ] 检查API调用配额使用情况
- [ ] 清理过期的缓存数据
- [ ] 监控数据库表大小增长
- [ ] 检查日志文件大小

### 定期维护

- [ ] 更新Dify API密钥 (每季度)
- [ ] 备份重要配置和数据 (每周)
- [ ] 检查依赖包更新 (每月)
- [ ] 性能优化评估 (每月)

### 紧急维护

- [ ] API服务中断处理流程
- [ ] 数据恢复程序
- [ ] 回滚方案准备
- [ ] 联系方式和升级路径

## 🎯 成功标准

部署成功的标志：

- [ ] 所有API端点正常响应
- [ ] Dify应用调用成功
- [ ] Airflow工作流正常执行
- [ ] 数据库正确存储结果
- [ ] 日志记录详细且准确
- [ ] 性能指标在预期范围内

---

**部署完成后，请将此清单存档并定期回顾更新。**
