# 用户认证方案总结

## 概述

本项目采用 **JWT (JSON Web Token) + OAuth2 Password Flow** 的认证方案，结合 bcrypt 密码加密，实现了完整的用户认证和授权机制。

## 认证架构

### 1. 认证流程

#### 1.1 用户登录流程

```
用户提交凭证 → 后端验证 → 生成JWT Token → 前端存储Token → 后续请求携带Token
```

**关键代码位置：**
- 登录端点：`backend/app/api/v1/login.py`
- 认证逻辑：`backend/app/crud.py` 中的 `authenticate()` 函数
- Token生成：`backend/app/core/security.py` 中的 `create_access_token()` 函数

#### 1.2 Token验证流程

```
请求携带Token → OAuth2PasswordBearer提取Token → JWT解码验证 → 查询用户信息 → 返回当前用户
```

**关键代码位置：**
- Token验证：`backend/app/api/deps.py` 中的 `get_current_user()` 函数
- OAuth2配置：`backend/app/api/deps.py` 中的 `reusable_oauth2`

## 技术实现细节

### 2. 密码安全

#### 2.1 密码加密
- **算法**：bcrypt
- **实现**：使用 `passlib` 库的 `CryptContext`
- **配置**：`backend/app/core/security.py`

```python
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
```

#### 2.2 密码操作
- **哈希生成**：`get_password_hash(password: str) -> str`
- **密码验证**：`verify_password(plain_password: str, hashed_password: str) -> bool`
- **使用场景**：
  - 用户注册时：`backend/app/crud.py` 的 `create_user()` 函数
  - 用户登录时：`backend/app/crud.py` 的 `authenticate()` 函数
  - 密码修改时：`backend/app/api/v1/users.py` 的 `update_password_me()` 函数

### 3. JWT Token机制

#### 3.1 Token生成
- **算法**：HS256
- **密钥**：`settings.SECRET_KEY`（从环境变量读取，默认生成32位随机字符串）
- **Token内容**：
  - `sub`：用户ID（UUID格式）
  - `exp`：过期时间（UTC时间戳）
- **过期时间**：8天（60分钟 × 24小时 × 8天）
- **配置位置**：`backend/app/core/config.py`

```python
ACCESS_TOKEN_EXPIRE_MINUTES: int = 60 * 24 * 8
```

#### 3.2 Token验证
- **提取方式**：通过 `OAuth2PasswordBearer` 从请求头的 `Authorization: Bearer <token>` 中提取
- **验证步骤**：
  1. 解码JWT Token
  2. 验证Token签名和过期时间
  3. 从Token中提取用户ID
  4. 查询数据库获取用户信息
  5. 检查用户是否激活
  6. 返回用户对象

**关键代码：**
```python
def get_current_user(session: SessionDep, token: TokenDep) -> User:
    # JWT解码和验证
    payload = jwt.decode(token, settings.SECRET_KEY, algorithms=[security.ALGORITHM])
    token_data = TokenPayload(**payload)
    
    # 从数据库获取用户
    user_id = uuid.UUID(token_data.sub)
    user = session.get(User, user_id)
    
    # 验证用户状态
    if not user or not user.is_active:
        raise HTTPException(...)
    
    return user
```

### 4. 用户注册

#### 4.1 公开注册
- **端点**：`POST /api/v1/users/signup`
- **特点**：
  - 新用户默认为**未激活状态**（`is_active = False`）
  - 需要管理员激活后才能登录
  - 自动分配 "General" 角色
- **实现位置**：`backend/app/api/v1/users.py` 的 `register_user()` 函数

#### 4.2 管理员创建用户
- **端点**：`POST /api/v1/users/`
- **权限**：需要超级用户权限
- **特点**：
  - 可以设置用户为激活状态
  - 可以设置用户为超级用户
  - 可选择发送欢迎邮件

### 5. 权限控制

#### 5.1 用户状态检查
- **激活状态**：`user.is_active`
  - 未激活用户无法登录
  - 登录时会检查：`backend/app/api/v1/login.py` 的 `login_access_token()` 函数

#### 5.2 超级用户权限
- **检查函数**：`get_current_active_superuser()`
- **使用场景**：
  - 用户管理（创建、删除、激活/停用用户）
  - 系统配置管理
  - 后台任务管理
  - 邮件审计日志查看
  - 等等

**关键代码：**
```python
def get_current_active_superuser(current_user: CurrentUser) -> User:
    if not current_user.is_superuser:
        raise HTTPException(status_code=403, detail="The user doesn't have enough privileges")
    return current_user
```

## 前端实现

### 6. Token管理

#### 6.1 Token存储
- **存储位置**：`localStorage`（键名：`access_token`）
- **实现位置**：`frontend/src/hooks/useAuth.ts`

#### 6.2 Token使用
- **请求头添加**：所有API请求自动在 `Authorization` 头中添加 `Bearer <token>`
- **实现位置**：`frontend/src/utils/api.ts` 的 `apiRequest()` 函数

```typescript
const token = localStorage.getItem("access_token")
const authHeader = token ? { Authorization: `Bearer ${token}` } : {}
```

#### 6.3 Token过期处理
- **过期检查**：前端定期检查Token是否过期（每分钟检查一次）
- **过期处理**：
  - 自动清除本地存储的Token
  - 清除所有缓存数据
  - 重定向到登录页面
- **实现位置**：`frontend/src/hooks/useAuth.ts`

```typescript
// 定期检查token过期
useEffect(() => {
  const checkTokenExpiry = () => {
    if (localStorage.getItem("access_token") && isTokenExpired()) {
      clearAuth()
      navigate({ to: "/login" })
    }
  }
  const interval = setInterval(checkTokenExpiry, 60000)
  return () => clearInterval(interval)
}, [navigate])
```

### 7. 认证状态管理

#### 7.1 登录状态检查
- **函数**：`isLoggedIn()`
- **逻辑**：
  1. 检查localStorage中是否存在token
  2. 检查token是否过期
  3. 如果过期则清除认证数据

#### 7.2 用户信息获取
- **Hook**：`useAuth()`
- **功能**：
  - 获取当前登录用户信息
  - 处理登录/注册/登出操作
  - 监听认证错误（如401未授权）
- **实现位置**：`frontend/src/hooks/useAuth.ts`

## API端点

### 8. 认证相关端点

#### 8.1 登录
- **端点**：`POST /api/v1/login/access-token`
- **请求格式**：OAuth2 Password Request Form
  - `username`：用户邮箱
  - `password`：用户密码
  - `grant_type`：固定为 "password"
- **响应**：`{ "access_token": "<jwt_token>" }`

#### 8.2 Token测试
- **端点**：`POST /api/v1/login/test-token`
- **功能**：验证当前Token是否有效，返回当前用户信息
- **权限**：需要有效Token

#### 8.3 用户注册
- **端点**：`POST /api/v1/users/signup`
- **请求体**：`UserRegister` 模型
- **响应**：创建的用户信息（未激活状态）

#### 8.4 获取当前用户
- **端点**：`GET /api/v1/users/me`
- **权限**：需要有效Token
- **响应**：当前登录用户的详细信息

#### 8.5 修改密码
- **端点**：`PATCH /api/v1/users/me/password`
- **权限**：需要有效Token
- **请求体**：
  - `current_password`：当前密码
  - `new_password`：新密码（8-40字符）
- **验证**：需要验证当前密码正确性，新密码不能与当前密码相同

## 安全特性

### 9. 安全措施

#### 9.1 密码安全
- ✅ 使用bcrypt进行密码哈希（不可逆加密）
- ✅ 密码长度限制：8-40字符
- ✅ 修改密码时需要验证当前密码
- ✅ 新密码不能与当前密码相同

#### 9.2 Token安全
- ✅ JWT Token包含过期时间
- ✅ Token使用HS256算法签名
- ✅ 密钥从环境变量读取，不在代码中硬编码
- ✅ 前端定期检查Token过期并自动登出

#### 9.3 用户状态管理
- ✅ 新注册用户默认为未激活状态，需要管理员激活
- ✅ 未激活用户无法登录
- ✅ 已停用用户无法登录
- ✅ Token验证时会检查用户状态

#### 9.4 权限控制
- ✅ 基于用户角色的权限控制
- ✅ 超级用户权限检查
- ✅ 用户激活/停用需要管理员权限

## 配置项

### 10. 环境变量配置

#### 10.1 认证相关配置
- `SECRET_KEY`：JWT签名密钥（必需）
- `ACCESS_TOKEN_EXPIRE_MINUTES`：Token过期时间（分钟），默认11520（8天）
- `API_V1_STR`：API版本前缀，默认 `/api/v1`

#### 10.2 邮件相关配置（用于密码重置）
- `SMTP_HOST`：SMTP服务器地址
- `SMTP_PORT`：SMTP端口
- `SMTP_USER`：SMTP用户名
- `SMTP_PASSWORD`：SMTP密码
- `EMAILS_FROM_EMAIL`：发件人邮箱
- `EMAILS_FROM_NAME`：发件人名称
- `EMAIL_RESET_TOKEN_EXPIRE_HOURS`：密码重置Token过期时间（小时），默认48小时

## 依赖关系

### 11. 关键依赖

#### 11.1 后端依赖
- `PyJWT`：JWT Token生成和验证
- `passlib[bcrypt]`：密码哈希和验证
- `fastapi`：Web框架
- `fastapi.security.OAuth2PasswordBearer`：OAuth2认证

#### 11.2 前端依赖
- `@tanstack/react-query`：数据获取和状态管理
- `localStorage`：Token存储

## 总结

### 12. 认证方案特点

1. **安全性**：
   - 密码使用bcrypt加密存储
   - JWT Token包含过期时间
   - 用户状态检查（激活/未激活）

2. **易用性**：
   - 前端自动处理Token过期
   - 统一的认证错误处理
   - 清晰的权限控制

3. **可扩展性**：
   - 基于角色的权限系统
   - 支持用户激活/停用
   - 支持密码重置（通过邮件）

4. **标准化**：
   - 遵循OAuth2 Password Flow标准
   - 使用JWT标准格式
   - RESTful API设计

## 相关文件清单

### 后端文件
- `backend/app/api/v1/login.py` - 登录端点
- `backend/app/api/v1/users.py` - 用户管理端点
- `backend/app/api/deps.py` - 认证依赖注入
- `backend/app/core/security.py` - 安全工具函数
- `backend/app/core/config.py` - 配置管理
- `backend/app/crud.py` - 数据库操作（包含认证逻辑）

### 前端文件
- `frontend/src/hooks/useAuth.ts` - 认证Hook
- `frontend/src/utils/api.ts` - API请求工具（包含Token处理）
- `frontend/src/utils/auth.ts` - 认证工具函数
- `frontend/src/routes/login.tsx` - 登录页面











