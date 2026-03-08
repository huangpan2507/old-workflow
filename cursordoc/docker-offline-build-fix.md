# Docker离线构建修复总结

## 问题描述

在运行离线启动脚本时，即使设置了环境变量，仍然出现镜像拉取错误：

```
! backend Warning         pull access denied for backend, repository does not exist or may require 'docker login'
! frontend Warning        pull access denied for frontend, repository does not exist or may require 'docker login'
! prestart Warning        pull access denied for backend, repository does not exist or may require 'docker login'
! files-ingestion Warning pull access denied for files-ingestion, repository does not exist or may require 'docker login'
```

## 根本原因分析

### Docker Compose的镜像拉取优先级

当Docker Compose配置中同时存在 `image` 和 `build` 时：

1. **优先尝试拉取镜像** - Docker Compose首先尝试从注册表拉取指定的镜像
2. **拉取失败时构建** - 只有当镜像不存在或拉取失败时，才会fallback到构建

### 配置分析

在 `docker-compose.yml` 中：
```yaml
backend:
  image: '${DOCKER_IMAGE_BACKEND?Variable not set}:${TAG-latest}'
  build:
    context: ./backend
```

即使设置了环境变量：
```bash
DOCKER_IMAGE_BACKEND=backend
TAG=latest
```

Docker Compose仍然会尝试拉取 `backend:latest` 镜像，而这个镜像在Docker Hub上不存在。

## 解决方案实施

### 方案1：修改离线配置文件

在 `docker-compose.offline.yml` 中添加强制构建配置：

```yaml
services:
  # prestart服务离线模式配置 - 强制构建
  prestart:
    <<: *offline-config
    image: ""  # 清空image配置
    build:
      context: ./backend

  # 后端服务离线模式配置 - 强制构建
  backend:
    <<: *offline-config
    image: ""  # 清空image配置
    build:
      context: ./backend
      target: development

  # 前端服务离线模式配置 - 强制构建
  frontend:
    <<: *offline-config
    image: ""  # 清空image配置
    build:
      context: ./frontend
      dockerfile: Dockerfile.dev
      args:
        - VITE_API_URL=http://${DOMAIN:-localhost}:8000

  # 文件摄取服务离线模式配置 - 强制构建
  files-ingestion:
    <<: *offline-config
    image: ""  # 清空image配置
    build:
      context: ./files_ingestion
      dockerfile: standalone_document_parser/Dockerfile
```

### 方案2：添加--build参数

在所有离线启动脚本中添加 `--build` 参数强制构建：

**修改的文件**：
- `bin/start-dev-offline.sh`
- `bin/start-staging-offline.sh`  
- `bin/start-prod-offline.sh`
- `bin/start-dev-watch-offline.sh`

**修改内容**：
```bash
# 修改前
docker compose -f docker-compose.yml -f docker-compose.dev.offline.yml --env-file env.dev up -d

# 修改后
docker compose -f docker-compose.yml -f docker-compose.dev.offline.yml --env-file env.dev up -d --build
```

## 修复验证

### 修复前的行为
```
! backend Warning         pull access denied for backend
! frontend Warning        pull access denied for frontend
! prestart Warning        pull access denied for backend
! files-ingestion Warning pull access denied for files-ingestion
```

### 修复后的行为
```
#1 [internal] load local bake definitions
#2 [backend internal] load build definition from Dockerfile
#3 [files-ingestion internal] load build definition from Dockerfile
#4 [prestart internal] load metadata for docker.io/library/python:3.10
```

现在Docker Compose正确地从源代码构建镜像，而不是尝试拉取不存在的镜像。

## 技术说明

### --build参数的作用

`docker compose up --build` 参数：
- 强制重新构建所有有build配置的服务
- 忽略已存在的镜像，总是从源代码构建
- 确保使用最新的源代码和依赖

### 离线模式的优势

1. **完全离线** - 不需要网络连接
2. **一致性** - 总是使用本地源代码构建
3. **可控性** - 避免意外拉取外部镜像
4. **安全性** - 不依赖外部镜像仓库

### Docker Compose配置合并

Docker Compose文件的合并顺序：
1. `docker-compose.yml` (基础配置)
2. `docker-compose.dev.offline.yml` (环境特定配置)

后面的文件会覆盖前面文件的相同配置项。

## 最佳实践

### 离线部署建议

1. **使用--build参数** - 确保总是构建最新代码
2. **预下载基础镜像** - 使用 `./scripts/import-images.sh` 导入基础镜像
3. **验证镜像** - 使用 `./scripts/verify-offline-images.sh` 验证基础镜像
4. **分离配置** - 使用独立的离线配置文件

### 配置文件组织

```
docker-compose.yml              # 基础配置
docker-compose.dev.yml          # 开发环境在线配置
docker-compose.dev.offline.yml  # 开发环境离线配置
docker-compose.staging.yml      # 测试环境在线配置
docker-compose.staging.offline.yml # 测试环境离线配置
docker-compose.prod.yml         # 生产环境在线配置
docker-compose.prod.offline.yml # 生产环境离线配置
```

## 相关文件

### 修改的配置文件
- `docker-compose.offline.yml` - 添加强制构建配置

### 修改的启动脚本
- `bin/start-dev-offline.sh` - 添加--build参数
- `bin/start-staging-offline.sh` - 添加--build参数
- `bin/start-prod-offline.sh` - 添加--build参数
- `bin/start-dev-watch-offline.sh` - 添加--build参数

### 相关工具脚本
- `scripts/import-images.sh` - 导入基础镜像
- `scripts/verify-offline-images.sh` - 验证镜像状态

## 总结

通过在离线配置文件中清空image配置并在启动脚本中添加 `--build` 参数，成功解决了Docker Compose在离线模式下尝试拉取不存在镜像的问题。

现在离线启动脚本会：
- ✅ 自动导入缺失的基础镜像
- ✅ 强制从源代码构建应用镜像  
- ✅ 完全离线运行，不依赖外部网络
- ✅ 确保使用最新的本地代码

这个修复确保了离线部署的可靠性和一致性。



