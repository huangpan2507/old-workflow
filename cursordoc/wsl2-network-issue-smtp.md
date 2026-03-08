# WSL2 网络问题导致 SMTP 连接失败

## 问题描述

在 WSL2 环境中，Docker 容器无法访问外部 SMTP 服务器（如 smtp.163.com），错误信息：
```
OSError: [Errno 101] Network is unreachable
```

## 问题原因

这是 WSL2 的网络路由问题，不是 Docker 配置问题。WSL2 使用虚拟网络适配器，有时候会出现网络路由异常。

## 解决方案

### 方案 1: 重启 WSL2 网络（推荐）

在 Windows PowerShell（管理员权限）中执行：

```powershell
# 重启 WSL2
wsl --shutdown

# 等待几秒后，重新启动 WSL2
wsl
```

### 方案 2: 重置 WSL2 网络

在 Windows PowerShell（管理员权限）中执行：

```powershell
# 重置 WSL2 网络
netsh winsock reset
netsh int ip reset

# 重启计算机（或重启 WSL2）
wsl --shutdown
```

### 方案 3: 检查 WSL2 DNS 配置

在 WSL2 中检查 DNS 配置：

```bash
# 检查 DNS 配置
cat /etc/resolv.conf

# 如果 DNS 有问题，可以手动设置
sudo bash -c 'echo "nameserver 8.8.8.8" > /etc/resolv.conf'
sudo bash -c 'echo "nameserver 8.8.4.4" >> /etc/resolv.conf'
```

### 方案 4: 使用 host 网络模式（仅用于开发环境）

修改 `docker-compose.dev.yml`，为 backend 服务添加网络模式：

```yaml
backend:
  network_mode: "host"
```

**注意**：host 模式会直接使用主机网络，可能存在安全风险，仅建议在开发环境使用。

### 方案 5: 配置代理（如果有代理）

如果使用代理，可以在 Docker Compose 中配置：

```yaml
backend:
  environment:
    HTTP_PROXY: "http://proxy.example.com:8080"
    HTTPS_PROXY: "http://proxy.example.com:8080"
    NO_PROXY: "localhost,127.0.0.1,db,mongo,redis"
```

## 验证网络连接

修复后，验证网络连接：

```bash
# 在容器内测试连接
docker exec foundation-backend-container python -c "import socket; socket.create_connection(('smtp.163.com', 465), timeout=5); print('Connection successful!')"

# 在主机上测试连接
python3 -c "import socket; socket.create_connection(('smtp.163.com', 465), timeout=5); print('Host can connect')"
```

## 临时解决方案

如果上述方案都无法解决，可以考虑：

1. **使用本地邮件测试服务**：在开发环境中使用 mailcatcher（已经在 docker-compose.dev.yml 中配置）
2. **使用其他 SMTP 服务**：尝试使用其他 SMTP 服务器（如 Gmail、QQ 邮箱等）
3. **在 Windows 主机上运行后端**：如果 WSL2 网络问题持续存在，可以考虑在 Windows 主机上直接运行后端服务

## 相关文件

- `docker-compose.yml` - Docker Compose 基础配置
- `docker-compose.dev.yml` - 开发环境配置
- `env.dev` - 开发环境变量配置



