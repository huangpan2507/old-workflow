# 用户认证安全策略测试指南

本文档提供了测试新增安全策略功能的详细步骤和方法。

## 测试环境准备

### 1. 确保服务运行
```bash
cd /home/song/workspace/xproject/foundation
docker compose ps  # 确认backend和db容器运行中
```

### 2. 获取API基础URL
- 后端API: `http://localhost:8001/api/v1`
- 文档: `http://localhost:8001/docs`

## 测试方法

### 方法1: 使用API文档（Swagger UI）测试

1. 打开浏览器访问 `http://localhost:8001/docs`
2. 使用Swagger UI的交互式界面测试各个端点

### 方法2: 使用curl命令测试

#### 2.1 测试密码强度验证

**测试弱密码（应该失败）**
```bash
# 测试密码太短
curl -X POST "http://localhost:8001/api/v1/users/signup" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test1@example.com",
    "password": "1234567",
    "full_name": "Test User 1"
  }'

# 测试密码复杂度不足（只有数字）
curl -X POST "http://localhost:8001/api/v1/users/signup" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test2@example.com",
    "password": "12345678",
    "full_name": "Test User 2"
  }'

# 测试密码复杂度不足（只有小写字母）
curl -X POST "http://localhost:8001/api/v1/users/signup" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test3@example.com",
    "password": "abcdefgh",
    "full_name": "Test User 3"
  }'
```

**测试强密码（应该成功）**
```bash
# 包含大写、小写、数字、特殊字符（4种类型）
curl -X POST "http://localhost:8001/api/v1/users/signup" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test4@example.com",
    "password": "Test123!@#",
    "full_name": "Test User 4"
  }'

# 包含大写、小写、数字（3种类型，应该成功）
curl -X POST "http://localhost:8001/api/v1/users/signup" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test5@example.com",
    "password": "Test12345",
    "full_name": "Test User 5"
  }'
```

#### 2.2 测试账户锁定机制

**步骤1: 创建测试用户（需要管理员先激活）**
```bash
# 先登录管理员账户获取token
TOKEN=$(curl -X POST "http://localhost:8001/api/v1/login/access-token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=admin@example.com&password=your_admin_password" | jq -r '.access_token')

# 创建测试用户
curl -X POST "http://localhost:8001/api/v1/users/" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "locktest@example.com",
    "password": "Test123!@#",
    "full_name": "Lock Test User",
    "is_active": true
  }'

# 激活用户
USER_ID=$(curl -X GET "http://localhost:8001/api/v1/users/" \
  -H "Authorization: Bearer $TOKEN" | jq -r '.data[] | select(.email=="locktest@example.com") | .id')

curl -X PATCH "http://localhost:8001/api/v1/users/$USER_ID/activate" \
  -H "Authorization: Bearer $TOKEN"
```

**步骤2: 测试登录失败次数累积**
```bash
# 尝试5次错误密码登录
for i in {1..5}; do
  echo "Attempt $i:"
  curl -X POST "http://localhost:8001/api/v1/login/access-token" \
    -H "Content-Type: application/x-www-form-urlencoded" \
    -d "username=locktest@example.com&password=wrongpassword"
  echo ""
done
```

**步骤3: 验证账户被锁定**
```bash
# 第6次尝试应该返回423错误（账户锁定）
curl -X POST "http://localhost:8001/api/v1/login/access-token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=locktest@example.com&password=Test123!@#"
```

**步骤4: 测试密码重置解锁账户**
```bash
# 请求密码重置
curl -X POST "http://localhost:8001/api/v1/password-recovery/locktest@example.com"

# 等待邮件中的token，然后重置密码
curl -X POST "http://localhost:8001/api/v1/reset-password/" \
  -H "Content-Type: application/json" \
  -d '{
    "token": "YOUR_RESET_TOKEN_FROM_EMAIL",
    "new_password": "NewTest123!@#"
  }'

# 尝试登录（应该成功，账户已解锁）
curl -X POST "http://localhost:8001/api/v1/login/access-token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=locktest@example.com&password=NewTest123!@#"
```

#### 2.3 测试密码重置频率限制

```bash
# 第一次请求密码重置（应该成功）
curl -X POST "http://localhost:8001/api/v1/password-recovery/locktest@example.com"

# 立即再次请求（应该返回429错误，频率限制）
curl -X POST "http://localhost:8001/api/v1/password-recovery/locktest@example.com"
```

#### 2.4 测试密码过期检查

**步骤1: 检查登录响应中的密码过期信息**
```bash
# 登录并查看响应
curl -X POST "http://localhost:8001/api/v1/login/access-token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=locktest@example.com&password=NewTest123!@#" | jq

# 响应应该包含 password_expires_in_days 字段
```

**步骤2: 手动设置密码过期时间（需要数据库访问）**
```bash
# 在数据库中设置密码修改时间为90天前
docker compose exec db psql -U postgres -d app -c "
UPDATE users 
SET password_changed_at = NOW() - INTERVAL '91 days' 
WHERE email = 'locktest@example.com';
"

# 再次登录，检查响应
curl -X POST "http://localhost:8001/api/v1/login/access-token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=locktest@example.com&password=NewTest123!@#" | jq
```

#### 2.5 测试密码更新功能

```bash
# 先登录获取token
TOKEN=$(curl -X POST "http://localhost:8001/api/v1/login/access-token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=locktest@example.com&password=NewTest123!@#" | jq -r '.access_token')

# 测试更新密码（弱密码应该失败）
curl -X PATCH "http://localhost:8001/api/v1/users/me/password" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "current_password": "NewTest123!@#",
    "new_password": "weak"
  }'

# 测试更新密码（强密码应该成功）
curl -X PATCH "http://localhost:8001/api/v1/users/me/password" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "current_password": "NewTest123!@#",
    "new_password": "Updated123!@#"
  }'
```

### 方法3: 使用Python测试脚本

创建一个测试脚本 `test_security_policies.py`:

```python
import requests
import time
from datetime import datetime, timedelta

BASE_URL = "http://localhost:8001/api/v1"

def test_password_strength():
    """测试密码强度验证"""
    print("\n=== 测试密码强度验证 ===")
    
    # 测试弱密码
    weak_passwords = [
        ("1234567", "太短"),
        ("12345678", "只有数字"),
        ("abcdefgh", "只有小写字母"),
        ("ABCDEFGH", "只有大写字母"),
    ]
    
    for password, reason in weak_passwords:
        response = requests.post(
            f"{BASE_URL}/users/signup",
            json={
                "email": f"test_{int(time.time())}@example.com",
                "password": password,
                "full_name": "Test User"
            }
        )
        print(f"密码 '{password}' ({reason}): {'失败 ✓' if response.status_code == 400 else '意外成功 ✗'}")
    
    # 测试强密码
    strong_password = "Test123!@#"
    response = requests.post(
        f"{BASE_URL}/users/signup",
        json={
            "email": f"test_{int(time.time())}@example.com",
            "password": strong_password,
            "full_name": "Test User"
        }
    )
    print(f"强密码 '{strong_password}': {'成功 ✓' if response.status_code == 200 else '失败 ✗'}")

def test_account_lockout(admin_token, test_email, test_password):
    """测试账户锁定机制"""
    print("\n=== 测试账户锁定机制 ===")
    
    # 尝试5次错误密码
    for i in range(1, 6):
        response = requests.post(
            f"{BASE_URL}/login/access-token",
            data={
                "username": test_email,
                "password": "wrongpassword"
            }
        )
        print(f"第{i}次错误登录: {response.status_code} - {response.json().get('detail', '')}")
    
    # 第6次应该被锁定
    response = requests.post(
        f"{BASE_URL}/login/access-token",
        data={
            "username": test_email,
            "password": test_password
        }
    )
    if response.status_code == 423:
        print(f"账户锁定测试成功 ✓ (状态码: 423)")
    else:
        print(f"账户锁定测试失败 ✗ (状态码: {response.status_code})")

def test_password_reset_rate_limit(test_email):
    """测试密码重置频率限制"""
    print("\n=== 测试密码重置频率限制 ===")
    
    # 第一次请求
    response1 = requests.post(f"{BASE_URL}/password-recovery/{test_email}")
    print(f"第一次请求: {response1.status_code}")
    
    # 立即第二次请求（应该被限制）
    response2 = requests.post(f"{BASE_URL}/password-recovery/{test_email}")
    if response2.status_code == 429:
        print(f"频率限制测试成功 ✓ (状态码: 429)")
    else:
        print(f"频率限制测试失败 ✗ (状态码: {response2.status_code})")

def test_password_expiry_warning(test_email, test_password):
    """测试密码过期警告"""
    print("\n=== 测试密码过期警告 ===")
    
    response = requests.post(
        f"{BASE_URL}/login/access-token",
        data={
            "username": test_email,
            "password": test_password
        }
    )
    
    if response.status_code == 200:
        token_data = response.json()
        expires_in_days = token_data.get("password_expires_in_days")
        if expires_in_days is not None:
            print(f"密码过期警告测试成功 ✓ (剩余天数: {expires_in_days})")
        else:
            print(f"密码过期警告测试失败 ✗ (响应中无password_expires_in_days字段)")
    else:
        print(f"登录失败: {response.status_code}")

if __name__ == "__main__":
    # 需要先获取管理员token和创建测试用户
    # 这里只是示例，实际使用时需要根据实际情况调整
    print("安全策略功能测试脚本")
    print("=" * 50)
    
    # 测试密码强度
    test_password_strength()
    
    # 其他测试需要先设置测试用户
    print("\n注意: 账户锁定、密码重置等测试需要先创建和激活测试用户")
```

### 方法4: 数据库验证

直接查询数据库验证字段是否正确设置：

```bash
# 查看用户的安全字段
docker compose exec db psql -U postgres -d app -c "
SELECT 
    email,
    password_changed_at,
    failed_login_attempts,
    locked_until,
    last_password_reset_request
FROM users 
WHERE email = 'your_test_email@example.com';
"

# 查看所有被锁定的账户
docker compose exec db psql -U postgres -d app -c "
SELECT 
    email,
    failed_login_attempts,
    locked_until,
    CASE 
        WHEN locked_until > NOW() THEN 'Locked'
        ELSE 'Not Locked'
    END as status
FROM users 
WHERE locked_until IS NOT NULL;
"
```

## 测试检查清单

### 密码强度验证
- [ ] 密码长度小于8字符应该被拒绝
- [ ] 密码只包含1种字符类型应该被拒绝
- [ ] 密码只包含2种字符类型应该被拒绝
- [ ] 密码包含3种字符类型应该被接受
- [ ] 密码包含4种字符类型应该被接受

### 账户锁定机制
- [ ] 5次登录失败后账户应该被锁定
- [ ] 锁定后尝试登录应该返回423错误
- [ ] 锁定信息应该包含剩余锁定时间
- [ ] 密码重置后账户应该自动解锁
- [ ] 密码更新后账户应该自动解锁

### 密码重置频率限制
- [ ] 1小时内第二次重置请求应该被拒绝（429错误）
- [ ] 1小时后可以再次请求重置

### 密码过期检查
- [ ] 登录响应应该包含 `password_expires_in_days` 字段
- [ ] 密码过期后应该显示负数天数

### 密码更新功能
- [ ] 更新密码时应该验证密码强度
- [ ] 更新密码后 `password_changed_at` 应该更新
- [ ] 更新密码后失败登录次数应该重置

## 注意事项

1. **测试用户创建**: 新注册的用户默认是未激活状态，需要管理员激活后才能登录
2. **邮件服务**: 密码重置功能需要邮件服务正常运行
3. **数据库时区**: 确保数据库和应用使用相同的时区设置
4. **清理测试数据**: 测试完成后记得清理测试用户数据

## 快速测试命令集合

```bash
# 1. 测试密码强度（弱密码）
curl -X POST "http://localhost:8001/api/v1/users/signup" \
  -H "Content-Type: application/json" \
  -d '{"email":"weak@test.com","password":"1234567","full_name":"Test"}'

# 2. 测试密码强度（强密码）
curl -X POST "http://localhost:8001/api/v1/users/signup" \
  -H "Content-Type: application/json" \
  -d '{"email":"strong@test.com","password":"Test123!@#","full_name":"Test"}'

# 3. 测试账户锁定（需要先创建并激活用户）
for i in {1..6}; do
  curl -X POST "http://localhost:8001/api/v1/login/access-token" \
    -H "Content-Type: application/x-www-form-urlencoded" \
    -d "username=test@example.com&password=wrong"
done

# 4. 测试密码重置频率限制
curl -X POST "http://localhost:8001/api/v1/password-recovery/test@example.com"
curl -X POST "http://localhost:8001/api/v1/password-recovery/test@example.com"  # 应该返回429
```

## 故障排查

如果测试失败，检查以下内容：

1. **容器状态**: `docker compose ps`
2. **日志**: `docker compose logs backend`
3. **数据库连接**: `docker compose exec backend python3 -c "from app.core.database import engine; print('DB OK' if engine else 'DB Error')"`
4. **迁移状态**: `docker compose exec backend alembic current`


