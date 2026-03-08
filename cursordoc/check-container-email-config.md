# 检查容器内邮件配置

## 问题发现

在 `docker-compose.dev.yml` 中，`environment` 部分会**覆盖** `env.dev` 文件中的配置：

```yaml
backend:
  env_file:
    - env.dev
  environment:
    SMTP_HOST: "mailcatcher"      # ⚠️ 这会覆盖 env.dev 中的配置
    SMTP_PORT: "1025"
    SMTP_TLS: "false"
    EMAILS_FROM_EMAIL: "noreply@example.com"
```

## 检查方法

### 方法 1: 检查环境变量

```bash
# 检查所有 SMTP 相关环境变量
docker exec foundation-backend-container env | grep SMTP

# 检查所有 EMAIL 相关环境变量
docker exec foundation-backend-container env | grep EMAIL

# 检查特定变量
docker exec foundation-backend-container printenv SMTP_HOST
docker exec foundation-backend-container printenv SMTP_PORT
docker exec foundation-backend-container printenv EMAIL_QUEUE_BATCH_SIZE
```

### 方法 2: 使用 Python 检查配置

```bash
# 复制检查脚本到容器
docker cp backend/check_email_config.py foundation-backend-container:/app/

# 在容器内运行
docker exec foundation-backend-container python /app/check_email_config.py
```

### 方法 3: 直接使用 Python 命令

```bash
docker exec foundation-backend-container python -c "
from app.core.config import settings
print('SMTP_HOST:', settings.SMTP_HOST)
print('SMTP_PORT:', settings.SMTP_PORT)
print('SMTP_TLS:', settings.SMTP_TLS)
print('SMTP_SSL:', settings.SMTP_SSL)
print('SMTP_USER:', settings.SMTP_USER)
print('EMAILS_FROM_EMAIL:', settings.EMAILS_FROM_EMAIL)
print('EMAIL_QUEUE_BATCH_SIZE:', settings.EMAIL_QUEUE_BATCH_SIZE)
print('EMAIL_QUEUE_PROCESS_INTERVAL:', settings.EMAIL_QUEUE_PROCESS_INTERVAL)
print('EMAIL_MAX_RETRY_COUNT:', settings.EMAIL_MAX_RETRY_COUNT)
"
```

## 当前配置问题

根据 `docker-compose.dev.yml`，当前容器内的配置是：

```bash
SMTP_HOST=mailcatcher          # 来自 docker-compose.dev.yml (覆盖了 env.dev)
SMTP_PORT=1025                # 来自 docker-compose.dev.yml (覆盖了 env.dev)
SMTP_TLS=false                # 来自 docker-compose.dev.yml (覆盖了 env.dev)
EMAILS_FROM_EMAIL=noreply@example.com  # 来自 docker-compose.dev.yml
```

而 `env.dev` 中的配置：
```bash
SMTP_HOST=smtp.gmail.com       # 被覆盖 ❌
SMTP_PORT=465                  # 被覆盖 ❌
SMTP_TLS=True                  # 被覆盖 ❌
SMTP_USER=fanqingsong@gmail.com
SMTP_PASSWORD=...
EMAILS_FROM_EMAIL=fanqingsong@gmail.com  # 被覆盖 ❌
```

## 解决方案

### 方案 1: 修改 docker-compose.dev.yml（推荐）

如果要使用 `env.dev` 中的 Gmail 配置，需要**删除或注释掉** `docker-compose.dev.yml` 中的 `environment` 覆盖：

```yaml
backend:
  env_file:
    - env.dev
  # environment:  # 注释掉这部分，让 env.dev 的配置生效
  #   SMTP_HOST: "mailcatcher"
  #   SMTP_PORT: "1025"
  #   SMTP_TLS: "false"
  #   EMAILS_FROM_EMAIL: "noreply@example.com"
```

### 方案 2: 更新 docker-compose.dev.yml 中的配置

如果要在 docker-compose 中直接配置，更新为：

```yaml
backend:
  env_file:
    - env.dev
  environment:
    SMTP_HOST: "smtp.gmail.com"
    SMTP_PORT: "465"
    SMTP_TLS: "True"
    SMTP_SSL: "False"
    SMTP_USER: "fanqingsong@gmail.com"
    SMTP_PASSWORD: "wkab culv iiza yjqt"
    EMAILS_FROM_EMAIL: "fanqingsong@gmail.com"
    EMAILS_FROM_NAME: ""
    EMAIL_QUEUE_BATCH_SIZE: "10"
    EMAIL_QUEUE_PROCESS_INTERVAL: "5.0"
    EMAIL_MAX_RETRY_COUNT: "6"
```

### 方案 3: 使用环境变量文件优先级

Docker Compose 的环境变量优先级：
1. `environment` (最高优先级，会覆盖)
2. `env_file` (较低优先级)

所以 `docker-compose.dev.yml` 中的 `environment` 会覆盖 `env.dev` 文件。

## 验证配置生效

修改后，重启容器并验证：

```bash
# 重启容器
docker-compose -f docker-compose.yml -f docker-compose.dev.yml restart backend

# 检查配置
docker exec foundation-backend-container python -c "
from app.core.config import settings
print('✅ SMTP_HOST:', settings.SMTP_HOST)
print('✅ SMTP_PORT:', settings.SMTP_PORT)
print('✅ SMTP_TLS:', settings.SMTP_TLS)
print('✅ SMTP_SSL:', settings.SMTP_SSL)
print('✅ Emails Enabled:', settings.emails_enabled)
"
```

## 注意事项

1. **Gmail 配置问题**: `env.dev` 中 `SMTP_PORT=465` 但 `SMTP_TLS=True`，这是不匹配的
   - 端口 465 应该使用 `SMTP_SSL=True`
   - 或者改为端口 587 使用 `SMTP_TLS=True`

2. **推荐 Gmail 配置**:
   ```bash
   SMTP_HOST=smtp.gmail.com
   SMTP_PORT=587          # 使用 587 而不是 465
   SMTP_TLS=True
   SMTP_SSL=False
   ```

3. **密码安全**: 不要在配置文件中硬编码密码，使用环境变量或密钥管理服务

