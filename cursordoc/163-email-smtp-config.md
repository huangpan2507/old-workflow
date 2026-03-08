# 163 邮箱 SMTP 配置说明

## 163 邮箱 SMTP 配置要求

163 邮箱的 SMTP 服务器配置：

### 推荐配置（SSL）

```bash
SMTP_HOST=smtp.163.com
SMTP_PORT=465
SMTP_TLS=False
SMTP_SSL=True
SMTP_USER=your-email@163.com
SMTP_PASSWORD=your-authorization-code  # 注意：需要使用授权码，不是登录密码
```

### 备用配置（TLS）

如果 465 端口不可用，可以尝试：

```bash
SMTP_HOST=smtp.163.com
SMTP_PORT=25
SMTP_TLS=True
SMTP_SSL=False
```

或者：

```bash
SMTP_HOST=smtp.163.com
SMTP_PORT=587
SMTP_TLS=True
SMTP_SSL=False
```

## 重要注意事项

### 1. 授权码 vs 登录密码

163 邮箱需要使用**授权码**而不是登录密码！

**获取授权码步骤**：
1. 登录 163 邮箱网页版
2. 进入"设置" → "POP3/SMTP/IMAP"
3. 开启 SMTP 服务
4. 生成授权码（16位字符）
5. 使用授权码作为 `SMTP_PASSWORD`

### 2. 端口和加密方式

- **端口 465**: 必须使用 `SMTP_SSL=True`，`SMTP_TLS=False`
- **端口 587**: 必须使用 `SMTP_TLS=True`，`SMTP_SSL=False`
- **端口 25**: 通常使用 `SMTP_TLS=True`，`SMTP_SSL=False`

### 3. 常见错误

#### 错误：Connection unexpectedly closed

**原因**：
- 端口和加密方式不匹配
- 使用了登录密码而不是授权码
- SMTP 服务未开启

**解决方法**：
1. 确保端口 465 使用 SSL，端口 587 使用 TLS
2. 使用授权码而不是登录密码
3. 在 163 邮箱设置中开启 SMTP 服务

#### 错误：SMTP authentication failed

**原因**：
- 用户名或密码错误
- 使用了登录密码而不是授权码

**解决方法**：
- 使用授权码作为密码
- 确保用户名是完整的邮箱地址

## 测试配置

使用以下命令测试配置：

```bash
docker exec foundation-backend-container python /app/test_email_send.py
```

## 其他邮箱服务商配置参考

### Gmail
```bash
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_TLS=True
SMTP_SSL=False
# 需要使用应用专用密码
```

### QQ 邮箱
```bash
SMTP_HOST=smtp.qq.com
SMTP_PORT=465
SMTP_TLS=False
SMTP_SSL=True
# 需要使用授权码
```

### Outlook/Office365
```bash
SMTP_HOST=smtp.office365.com
SMTP_PORT=587
SMTP_TLS=True
SMTP_SSL=False
```

