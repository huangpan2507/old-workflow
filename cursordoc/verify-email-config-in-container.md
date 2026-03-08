# 验证容器内邮件配置

## 快速检查命令

### 方法 1: 检查环境变量

```bash
# 检查所有 SMTP 和 EMAIL 相关的环境变量
docker exec foundation-backend-container env | grep -E "SMTP|EMAIL" | sort
```

### 方法 2: 使用 Python 检查配置

```bash
docker exec foundation-backend-container python -c "
import sys
sys.path.insert(0, '/app')
from app.core.config import settings

print('=== SMTP Configuration ===')
print(f'SMTP_HOST: {settings.SMTP_HOST}')
print(f'SMTP_PORT: {settings.SMTP_PORT}')
print(f'SMTP_TLS: {settings.SMTP_TLS}')
print(f'SMTP_SSL: {settings.SMTP_SSL}')
print(f'SMTP_USER: {settings.SMTP_USER}')
print(f'SMTP_PASSWORD: {\"*\" * len(settings.SMTP_PASSWORD) if settings.SMTP_PASSWORD else \"Not set\"}')

print()
print('=== Email Sender ===')
print(f'EMAILS_FROM_EMAIL: {settings.EMAILS_FROM_EMAIL}')
print(f'EMAILS_FROM_NAME: {settings.EMAILS_FROM_NAME}')

print()
print('=== Email Queue ===')
print(f'EMAIL_QUEUE_BATCH_SIZE: {settings.EMAIL_QUEUE_BATCH_SIZE}')
print(f'EMAIL_QUEUE_PROCESS_INTERVAL: {settings.EMAIL_QUEUE_PROCESS_INTERVAL}')
print(f'EMAIL_MAX_RETRY_COUNT: {settings.EMAIL_MAX_RETRY_COUNT}')

print()
print('=== Status ===')
print(f'Emails Enabled: {settings.emails_enabled}')
"
```

### 方法 3: 使用检查脚本

```bash
# 运行检查脚本
bash check_email_env.sh

# 或者直接复制脚本到容器运行
docker cp backend/check_email_config.py foundation-backend-container:/tmp/
docker exec foundation-backend-container python /tmp/check_email_config.py
```

## 预期结果

如果配置正确，你应该看到：

```
=== SMTP Configuration ===
SMTP_HOST: smtp.gmail.com
SMTP_PORT: 587
SMTP_TLS: True
SMTP_SSL: False
SMTP_USER: fanqingsong@gmail.com
SMTP_PASSWORD: ************

=== Email Sender ===
EMAILS_FROM_EMAIL: fanqingsong@gmail.com
EMAILS_FROM_NAME: 

=== Email Queue ===
EMAIL_QUEUE_BATCH_SIZE: 10
EMAIL_QUEUE_PROCESS_INTERVAL: 5.0
EMAIL_MAX_RETRY_COUNT: 6

=== Status ===
Emails Enabled: True
```

## 如果配置不正确

### 检查容器名称

```bash
# 列出所有运行中的容器
docker ps --format "table {{.Names}}\t{{.Status}}"

# 查找 backend 容器
docker ps | grep backend
```

### 检查环境文件是否加载

```bash
# 检查容器使用的环境文件
docker inspect foundation-backend-container | grep -A 10 "EnvFile"
```

### 重启容器

```bash
# 重启容器使配置生效
docker-compose -f docker-compose.yml -f docker-compose.dev.yml restart backend

# 或者重新创建容器
docker-compose -f docker-compose.yml -f docker-compose.dev.yml up -d --force-recreate backend
```

## 验证配置是否生效

配置生效后，你可以：

1. **创建测试审批请求** - 创建一个 Policy Revision 或 Deviation
2. **检查日志** - 查看是否有邮件发送相关的日志
3. **检查 Redis 队列** - 查看邮件是否被加入队列

```bash
# 检查 Redis 队列
docker exec foundation-redis-container redis-cli -a ${REDIS_PASSWORD} LLEN email:queue:pending
docker exec foundation-redis-container redis-cli -a ${REDIS_PASSWORD} LLEN email:queue:priority

# 查看队列统计
docker exec foundation-redis-container redis-cli -a ${REDIS_PASSWORD} HGETALL email:queue:stats
```

## 常见问题

### 问题 1: 环境变量为空

**原因**: `env.dev` 文件没有被正确加载

**解决**: 检查 `docker-compose.dev.yml` 中的 `env_file` 配置

### 问题 2: 配置被覆盖

**原因**: `docker-compose.dev.yml` 中的 `environment` 覆盖了 `env_file`

**解决**: 确保已删除或注释掉 `environment` 中的 SMTP 配置

### 问题 3: 容器名称不对

**原因**: 容器名称可能不同

**解决**: 使用 `docker ps` 查找正确的容器名称

