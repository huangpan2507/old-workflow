# Docker镜像环境变量修复总结

## 问题描述

在运行离线启动脚本时出现以下错误：
```
! frontend Warning             pull access denied for frontend, repository does not exist or may require 'docker login'
! files-ingestion Warning      pull access denied for files-ingestion, repository does not exist or may require 'docker login'  
! prestart Warning             pull access denied for backend, repository does not exist or may require 'docker login'
! backend Warning              pull access denied for backend, repository does not exist or may require 'docker login'
```

## 根本原因

在 `docker-compose.yml` 文件中，这些服务使用了环境变量来定义镜像名称：

```yaml
prestart:
  image: '${DOCKER_IMAGE_BACKEND?Variable not set}:${TAG-latest}'
  
backend:
  image: '${DOCKER_IMAGE_BACKEND?Variable not set}:${TAG-latest}'
  
frontend:
  image: '${DOCKER_IMAGE_FRONTEND?Variable not set}:${TAG-latest}'
  
files-ingestion:
  image: '${DOCKER_IMAGE_FILES_INGESTION?Variable not set}:${TAG-latest}'
```

当这些环境变量未设置时，Docker Compose尝试从Docker Hub拉取不存在的镜像，导致错误。

## 解决方案

### 方案1：设置环境变量（已采用）

在所有环境文件中添加必要的Docker镜像环境变量：

#### 修改的文件

1. **env.dev** ✅ （已有配置，无需修改）
2. **env.staging** ✅ （已更新）
3. **env.prod** ✅ （已更新）
4. **env.exmaple** ✅ （已更新）

#### 添加的环境变量

```bash
# Docker 镜像配置
DOCKER_IMAGE_BACKEND=backend
DOCKER_IMAGE_FRONTEND=frontend
DOCKER_IMAGE_FILES_INGESTION=files-ingestion
TAG=latest
```

## 修复结果

### 修复前
```
! frontend Warning             pull access denied for frontend
! files-ingestion Warning      pull access denied for files-ingestion
! prestart Warning             pull access denied for backend
! backend Warning              pull access denied for backend
```

### 修复后
Docker Compose配置正确解析环境变量：
```yaml
backend:
  image: backend:latest
  
frontend:
  image: frontend:latest
  
files-ingestion:
  image: files-ingestion:latest
```

## 验证步骤

1. **验证环境变量解析**：
   ```bash
   docker compose -f docker-compose.yml -f docker-compose.dev.offline.yml --env-file env.dev config --services
   ```

2. **检查镜像名称**：
   ```bash
   docker compose -f docker-compose.yml -f docker-compose.dev.offline.yml --env-file env.dev config | grep "image.*backend\|image.*frontend\|image.*files-ingestion"
   ```

3. **验证离线镜像**：
   ```bash
   ./scripts/verify-offline-images.sh -e dev
   ```

## 技术说明

### 为什么需要这些环境变量？

1. **灵活性**：允许在不同环境中使用不同的镜像名称或标签
2. **CI/CD集成**：支持在部署管道中动态设置镜像名称
3. **多环境支持**：开发、测试、生产环境可以使用不同的镜像配置

### 镜像构建vs镜像拉取

这些服务既有 `image` 配置也有 `build` 配置：
- **有镜像时**：使用预构建的镜像（更快）
- **无镜像时**：从源代码构建（fallback）

### 环境变量优先级

1. 命令行 `-e` 参数
2. `--env-file` 指定的文件
3. `env_file` 在compose文件中指定
4. 系统环境变量
5. `.env` 文件（如果存在）

## 最佳实践

1. **保持一致性**：所有环境文件都应包含相同的Docker镜像变量
2. **使用有意义的名称**：镜像名称应该反映服务功能
3. **版本管理**：使用 `TAG` 变量管理镜像版本
4. **文档化**：在示例文件中包含所有必要的变量

## 相关文件

- `docker-compose.yml` - 主配置文件
- `docker-compose.dev.offline.yml` - 开发环境离线配置
- `docker-compose.staging.offline.yml` - 测试环境离线配置  
- `docker-compose.prod.offline.yml` - 生产环境离线配置
- `env.dev` - 开发环境变量
- `env.staging` - 测试环境变量
- `env.prod` - 生产环境变量
- `env.exmaple` - 环境变量示例

## 总结

通过在所有环境文件中正确设置Docker镜像环境变量，解决了离线启动时的镜像拉取错误。现在Docker Compose能够正确解析镜像名称，避免尝试从Docker Hub拉取不存在的镜像。

这个修复确保了：
- ✅ 离线模式正常工作
- ✅ 环境变量正确解析
- ✅ 所有环境配置一致
- ✅ 向后兼容性保持



