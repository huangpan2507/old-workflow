# Foundation项目Dify集成指南

## 概述

本文档介绍如何在Foundation项目的策略库(Policy Library)中集成Dify平台，将Dify应用封装为服务，实现AI驱动的策略分析、建议生成和智能咨询功能。

## 🎯 集成目标

- **策略文档分析**: 使用Dify Agent应用自动分析策略文档，提取关键信息和风险点
- **策略建议生成**: 通过Dify Workflow应用根据上下文生成策略建议和改进方案
- **智能策略咨询**: 提供基于Dify Chatbot的策略咨询服务
- **工作流集成**: 将Dify应用无缝集成到Airflow工作流中

## 🏗️ 架构设计

### 整体架构

```
Foundation Policy Library + Dify Integration
├── API Layer
│   ├── /api/v1/dify/analyze - 策略文档分析
│   ├── /api/v1/dify/recommend - 策略建议生成
│   ├── /api/v1/dify/chat - 策略咨询聊天
│   └── /api/v1/dify/health - 服务健康检查
├── Service Layer
│   ├── DifyPolicyService - 核心服务类
│   ├── DifyConfig - 配置管理
│   └── DifyAppRegistry - 应用注册表
├── Airflow Integration
│   ├── dify_policy_workflow.py - 主工作流DAG
│   └── dify_policy_integration.py - 集成模块
└── Database Layer
    ├── dify_policy_results - 处理结果表
    └── dify_policy_workflow_results - 工作流结果表
```

### 核心组件

1. **配置管理** (`backend/app/core/dify_config.py`)
   - Dify平台连接配置
   - 应用ID管理
   - 功能开关控制

2. **服务层** (`backend/app/services/dify_policy_service.py`)
   - HTTP客户端封装
   - API调用逻辑
   - 错误处理和重试

3. **API路由** (`backend/app/api/routes/dify_policy_routes.py`)
   - RESTful API端点
   - 请求/响应模型
   - 权限控制

4. **Airflow集成** (`airflow/dags/`)
   - 工作流定义
   - 任务编排
   - 结果持久化

## 🚀 快速开始

### 1. 环境配置

在 `.env` 文件中添加Dify配置：

```bash
# Dify平台基础配置
DIFY_API_URL=https://api.dify.ai/v1
DIFY_API_KEY=your-dify-api-key-here
DIFY_TIMEOUT=30
DIFY_RETRY_COUNT=3
DIFY_ENABLED=true

# Dify应用配置
DIFY_DEFAULT_APP_ID=app-xxxxxxxxxx          # 默认聊天应用
DIFY_POLICY_APP_ID=app-yyyyyyyyyy           # 策略分析应用(Agent)
DIFY_ANALYSIS_APP_ID=app-zzzzzzzzzz         # 策略生成应用(Workflow)

# 功能开关
DIFY_ENABLE_STREAMING=true
DIFY_ENABLE_CONVERSATION=true
DIFY_AUTO_GENERATE_NAME=true

# 高级配置
DIFY_MAX_TOKENS=4000
DIFY_TEMPERATURE=0.7
DIFY_TOP_P=0.9

# 缓存配置
DIFY_CACHE_TTL=3600
DIFY_USE_CACHE=true
```

### 2. 数据库表创建

创建必要的数据库表：

```sql
-- Dify策略处理结果表
CREATE TABLE IF NOT EXISTS dify_policy_results (
    id SERIAL PRIMARY KEY,
    area VARCHAR(100) NOT NULL,
    task_type VARCHAR(50) NOT NULL,
    result_data JSONB NOT NULL,
    conversation_id VARCHAR(100),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    status VARCHAR(20) DEFAULT 'pending'
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
    UNIQUE(area, workflow_run_id)
);

-- 策略需求表(可选)
CREATE TABLE IF NOT EXISTS policy_requirements (
    id SERIAL PRIMARY KEY,
    area VARCHAR(100) NOT NULL,
    requirement_text TEXT NOT NULL,
    priority INTEGER DEFAULT 1,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 策略文档表(可选)
CREATE TABLE IF NOT EXISTS policy_documents (
    id SERIAL PRIMARY KEY,
    area VARCHAR(100) NOT NULL,
    document_content TEXT NOT NULL,
    status VARCHAR(20) DEFAULT 'active',
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

### 3. 启动服务

重启Foundation后端服务以加载新的配置和路由：

```bash
cd /home/huangpan/foundation/backend
uvicorn app.main:app --reload
```

## 📚 API使用指南

### 健康检查

```bash
curl -X GET "http://localhost:8000/api/v1/dify/health" \
  -H "Authorization: Bearer your-jwt-token"
```

### 策略文档分析

```bash
curl -X POST "http://localhost:8000/api/v1/dify/analyze" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your-jwt-token" \
  -d '{
    "document_content": "Your policy document content here...",
    "analysis_type": "comprehensive"
  }'
```

### 策略建议生成

```bash
curl -X POST "http://localhost:8000/api/v1/dify/recommend" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your-jwt-token" \
  -d '{
    "context": "Policy context information...",
    "requirements": [
      "Ensure compliance with regulations",
      "Improve operational efficiency"
    ],
    "policy_type": "compliance"
  }'
```

### 策略咨询聊天

```bash
curl -X POST "http://localhost:8000/api/v1/dify/chat" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your-jwt-token" \
  -d '{
    "message": "What are the key compliance requirements for data privacy policies?",
    "conversation_id": "optional-conversation-id"
  }'
```

### 流式聊天

```bash
curl -X POST "http://localhost:8000/api/v1/dify/chat/stream" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your-jwt-token" \
  -d '{
    "message": "Explain the risk assessment process for new policies"
  }'
```

## 🔧 Airflow工作流使用

### 触发工作流

通过Airflow API触发Dify策略工作流：

```bash
curl -X POST "http://localhost:8080/api/v1/dags/dify_policy_workflow/dagRuns" \
  -H "Content-Type: application/json" \
  -H "Authorization: Basic $(echo -n 'airflow:airflow' | base64)" \
  -d '{
    "conf": {
      "area": "data_privacy",
      "document_content": "Your policy document content here..."
    }
  }'
```

### 工作流步骤

1. **文档内容提取**: 从参数或数据库获取策略文档
2. **配置验证**: 验证Dify服务配置和连接
3. **策略分析**: 使用Dify Agent分析文档
4. **建议生成**: 使用Dify Workflow生成策略建议
5. **结果整合**: 将分析和建议结果保存到数据库

## 🛠️ 开发指南

### 添加新的Dify应用

1. 在Dify平台创建新应用并获取App ID
2. 在 `DifyAppRegistry` 中注册应用：

```python
dify_app_registry.register_app("new_policy_app", {
    "app_id": "app-new-id",
    "type": DifyAppType.AGENT,
    "description": "New policy application",
    "max_tokens": 4000,
    "temperature": 0.5,
})
```

3. 在 `DifyPolicyService` 中添加相应的方法
4. 创建对应的API端点

### 扩展工作流

1. 在 `dify_policy_integration.py` 中添加新的任务函数
2. 在 `dify_policy_workflow.py` 中定义新的任务节点
3. 配置任务依赖关系

### 自定义分析类型

支持的分析类型：
- `comprehensive`: 全面分析
- `summary`: 摘要分析
- `compliance`: 合规性分析
- `risk`: 风险分析

可在 `DifyPolicyService.analyze_policy_document()` 方法中添加新的分析类型。

## 🔍 监控和日志

### 日志查看

```bash
# 查看后端日志
tail -f /home/huangpan/foundation/backend/logs/app.log

# 查看Airflow日志
tail -f /home/huangpan/foundation/airflow/logs/dags/dify_policy_workflow/
```

### 监控指标

- Dify API调用成功率
- 响应时间监控
- 工作流执行状态
- 错误率统计

### 健康检查端点

定期检查服务健康状态：
- `/api/v1/dify/health` - Dify服务健康检查
- `/api/v1/dify/apps` - 可用应用列表

## 🚨 故障排除

### 常见问题

1. **API密钥无效**
   - 检查 `DIFY_API_KEY` 环境变量
   - 验证Dify平台中的API密钥是否正确

2. **应用ID配置错误**
   - 确认 `DIFY_POLICY_APP_ID` 等应用ID配置
   - 验证应用在Dify平台中是否存在且已发布

3. **网络连接问题**
   - 检查 `DIFY_API_URL` 配置
   - 验证网络连接和防火墙设置

4. **超时问题**
   - 调整 `DIFY_TIMEOUT` 配置
   - 检查文档内容大小是否超限

### 错误代码

- `500`: 内部服务器错误
- `401`: 认证失败
- `404`: 应用或资源未找到
- `429`: 请求频率限制

## 📈 性能优化

### 缓存策略

- 启用Redis缓存 (`DIFY_USE_CACHE=true`)
- 调整缓存TTL (`DIFY_CACHE_TTL`)
- 使用会话ID维持上下文

### 并发控制

- 配置HTTP连接池大小
- 设置合理的超时时间
- 实施请求重试机制

### 资源限制

- 控制最大token数量 (`DIFY_MAX_TOKENS`)
- 限制并发请求数量
- 监控内存使用情况

## 🔐 安全考虑

### API密钥管理

- 使用环境变量存储敏感信息
- 定期轮换API密钥
- 限制API密钥权限范围

### 数据保护

- 敏感数据加密存储
- 日志脱敏处理
- 访问权限控制

### 网络安全

- 使用HTTPS协议
- 配置防火墙规则
- 实施IP白名单

## 🔄 版本更新

### 更新步骤

1. 备份现有配置
2. 更新代码和配置
3. 运行数据库迁移
4. 重启服务
5. 验证功能正常

### 兼容性

- 向后兼容现有API
- 平滑升级策略
- 版本回退方案

## 📞 支持和反馈

如有问题或建议，请通过以下方式联系：

- 项目Issues: GitHub Issues
- 技术支持: 项目维护团队
- 文档更新: 提交PR到项目仓库

---

*本文档会持续更新，请关注最新版本。*
