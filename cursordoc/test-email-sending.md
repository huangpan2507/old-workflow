# 测试邮件发送功能

## 测试脚本位置

测试脚本已创建在：`backend/test_email_send.py`

## 运行测试

### 方法 1: 在容器内运行（推荐）

```bash
# 1. 复制测试脚本到容器
docker cp backend/test_email_send.py foundation-backend-container:/app/test_email_send.py

# 2. 在容器内运行测试
docker exec foundation-backend-container python /app/test_email_send.py
```

### 方法 2: 使用 docker-compose exec

```bash
docker-compose -f docker-compose.yml -f docker-compose.dev.yml exec backend python /app/test_email_send.py
```

### 方法 3: 进入容器交互式运行

```bash
# 进入容器
docker exec -it foundation-backend-container bash

# 在容器内运行
cd /app
python test_email_send.py
```

## 测试内容

测试脚本会执行以下测试：

1. **配置检查** - 验证 SMTP 配置是否正确
2. **直接邮件发送** - 通过 SMTP 直接发送测试邮件
3. **邮件队列测试** - 将邮件加入 Redis 队列

## 预期输出

如果一切正常，你应该看到：

```
============================================================
Email Notification System Test
============================================================

Test Email: qingsong.fan@tietoevry.com

============================================================
Email Configuration Test
============================================================

SMTP Configuration:
  SMTP_HOST: smtp.gmail.com
  SMTP_PORT: 587
  SMTP_TLS: True
  SMTP_SSL: False
  SMTP_USER: fanqingsong@gmail.com
  SMTP_PASSWORD: ************
  EMAILS_FROM_EMAIL: fanqingsong@gmail.com
  EMAILS_FROM_NAME: 

Emails Enabled: True
✅ Email configuration looks good

============================================================
Test 1: Direct Email Send (SMTP)
============================================================

Sending test email to: qingsong.fan@tietoevry.com
Please wait...
✅ Email sent successfully!
   Check your inbox at: qingsong.fan@tietoevry.com

============================================================
Test 2: Email Queue (Redis)
============================================================

✅ Redis is connected
Enqueuing test email to: qingsong.fan@tietoevry.com
✅ Email enqueued successfully!
   Email ID: <uuid>
   Email will be processed by background worker

Queue Statistics:
  Pending: 0
  Priority: 1
  Processing: 0
  Failed: 0

============================================================
Test Summary
============================================================

Direct Email Send: ✅ PASS
Email Queue:        ✅ PASS

✅ Direct email send test passed!
   Please check your inbox at: qingsong.fan@tietoevry.com
✅ Email queue test passed!
   Email has been added to queue and will be processed by background worker
   Check your inbox in a few seconds

============================================================
```

## 故障排查

### 问题 1: SMTP 认证失败

**错误信息**: `SMTP authentication failed`

**可能原因**:
- Gmail 密码错误
- 需要使用应用专用密码（App Password）

**解决方法**:
1. 登录 Gmail 账户
2. 启用两步验证
3. 生成应用专用密码
4. 在 `env.dev` 中使用应用专用密码

### 问题 2: 连接超时

**错误信息**: `SMTP connection error: Connection timeout`

**可能原因**:
- 网络问题
- SMTP 服务器地址错误
- 防火墙阻止连接

**解决方法**:
- 检查网络连接
- 验证 SMTP_HOST 配置
- 尝试使用不同的端口（587 或 465）

### 问题 3: Redis 未连接

**错误信息**: `Redis is not connected`

**解决方法**:
```bash
# 检查 Redis 容器状态
docker ps | grep redis

# 检查 Redis 连接
docker exec foundation-backend-container python -c "
from app.services.redis_service import redis_service
print('Redis connected:', redis_service.is_connected())
"
```

### 问题 4: 邮件队列未处理

**检查队列状态**:
```bash
# 检查队列长度
docker exec foundation-redis-container redis-cli -a ${REDIS_PASSWORD} LLEN email:queue:priority
docker exec foundation-redis-container redis-cli -a ${REDIS_PASSWORD} LLEN email:queue:pending

# 检查后台任务是否运行
docker exec foundation-backend-container ps aux | grep python
```

## 验证邮件接收

测试完成后：

1. **检查收件箱** - 登录 `qingsong.fan@tietoevry.com` 查看邮件
2. **检查垃圾邮件文件夹** - 测试邮件可能被标记为垃圾邮件
3. **检查发送时间** - 直接发送应该立即收到，队列发送可能需要几秒钟

## 手动测试邮件发送（简单版本）

如果测试脚本有问题，可以使用这个简单的 Python 命令：

```bash
docker exec foundation-backend-container python -c "
import sys
sys.path.insert(0, '/app')
from app.services.email_service import EmailService

email_service = EmailService()
result = email_service.send_email(
    to_email='qingsong.fan@tietoevry.com',
    subject='[Test] Email Test',
    html_content='<h1>Test Email</h1><p>This is a test email.</p>'
)
print('Success:', result['success'])
if not result['success']:
    print('Error:', result.get('error'))
"
```

## 下一步

如果测试成功：

1. ✅ 邮件配置正确
2. ✅ SMTP 连接正常
3. ✅ 邮件队列工作正常
4. ✅ 可以开始使用邮件通知功能

如果测试失败，请检查：
- SMTP 配置是否正确
- Gmail 应用密码是否正确
- Redis 是否正常运行
- 后台任务是否启动

