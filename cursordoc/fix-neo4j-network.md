# 修复 Neo4j 容器网络连接问题

## 问题描述
在生产环境中，Neo4j 容器没有连接到 `shared_external_network` 网络，导致 Airflow DAGs 无法访问 Neo4j 服务。

## 问题原因

### 1. 网络配置问题
Docker Compose 在修改网络配置后，需要重新创建容器才能应用新的网络配置。仅仅重启容器是不够的，因为容器的网络配置是在创建时确定的。

### 2. 环境变量配置错误（已修复）
Neo4j 容器会将所有以 `NEO4J_` 开头的环境变量自动转换为配置项。如果环境变量文件（如 `env.prod`）中包含 `NEO4J_URI`，Neo4j 会尝试将其解析为配置项 `URI`，但这不是有效的 Neo4j 配置项，导致容器启动失败并报错：
```
Failed to read config: Unrecognized setting. No declared setting with name: URI.
```

**解决方案**：在 `docker-compose.prod.yml` 和 `docker-compose.staging.yml` 中，不再使用 `env_file` 加载所有环境变量，而是明确指定 Neo4j 需要的环境变量，避免 `NEO4J_URI` 等变量被误解析。

## 解决方案

### 方法 1：强制重新创建容器（推荐）

在生产环境服务器上执行：

```bash
cd ~/deployment/foundation

# 停止并删除 Neo4j 容器（数据会保留在 volumes 中）
sudo docker compose -f docker-compose.yml -f docker-compose.prod.yml stop neo4j
sudo docker compose -f docker-compose.yml -f docker-compose.prod.yml rm -f neo4j

# 重新创建并启动 Neo4j 容器
sudo docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d neo4j
```

### 方法 2：使用 force-recreate 参数

```bash
cd ~/deployment/foundation

# 强制重新创建 Neo4j 容器
sudo docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --force-recreate neo4j
```

## 验证修复

执行以下命令验证 Neo4j 容器是否已连接到共享网络：

```bash
# 检查网络中的容器
sudo docker network inspect shared_external_network | grep neo4j

# 或者查看完整的网络信息
sudo docker network inspect shared_external_network

# 检查 Neo4j 容器的网络配置
sudo docker inspect foundation-neo4j-prod-container | grep -A 10 Networks
```

如果修复成功，应该能看到 `foundation-neo4j-prod-container` 在 `shared_external_network` 网络中。

## 注意事项

1. **数据安全**：重新创建容器不会影响数据，因为数据存储在 Docker volumes 中（`neo4j-data-prod`、`neo4j-logs-prod` 等）。

2. **服务中断**：重新创建容器会导致 Neo4j 服务短暂中断（通常几秒钟），建议在低峰期执行。

3. **连接验证**：修复后，可以通过以下方式验证 Airflow 是否能访问 Neo4j：
   ```bash
   # 在 Airflow worker 容器中测试连接
   sudo docker exec -it airflow-airflow-worker-1 ping -c 3 neo4j
   ```

## 相关配置

配置文件已更新：
- `docker-compose.prod.yml` - 添加了 Neo4j 的 networks 配置，并修复了环境变量问题
- `docker-compose.staging.yml` - 添加了 Neo4j 的 networks 配置，并修复了环境变量问题

配置内容：
```yaml
neo4j:
  networks:
    - shared_external_network
    - default
  # 不使用 env_file，避免 NEO4J_URI 等变量被误解析为配置项
  # 明确指定 Neo4j 需要的环境变量
  environment:
    - ENVIRONMENT=production
    - NEO4J_AUTH=${NEO4J_USER:-neo4j}/${NEO4J_PASSWORD:-changethis}
    - NEO4J_PLUGINS=["apoc"]
    - NEO4J_dbms_security_procedures_unrestricted=apoc.*
    - NEO4J_dbms_security_procedures_allowlist=apoc.*
```

**重要说明**：
- `NEO4J_URI` 环境变量不应该传递给 Neo4j 容器，它会被误解析为配置项
- `NEO4J_URI` 只用于应用程序（如 Airflow DAGs、后端服务）连接 Neo4j，不用于 Neo4j 容器自身的配置
- Neo4j 容器的认证信息通过 `NEO4J_AUTH` 环境变量传递，格式为 `username/password`

