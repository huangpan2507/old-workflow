# EIM版本和通用版本部署指南

## 概述

本项目支持两种部署版本：
- **EIM版本**: 面向EIM集团客户的版本
- **通用版本**: 面向其他公司客户的版本

两种版本使用同一套代码，通过环境变量配置区分版本类型和品牌信息。

## 环境文件结构

### 开发环境
- `env.dev.eim` - 开发环境 EIM版本配置
- `env.dev.generic` - 开发环境 通用版本配置

### 测试环境
- `env.staging.eim` - 测试环境 EIM版本配置
- `env.staging.generic` - 测试环境 通用版本配置

### 生产环境
- `env.prod.eim` - 生产环境 EIM版本配置
- `env.prod.generic` - 生产环境 通用版本配置

## 版本配置说明

每个版本特定的环境文件都包含以下配置项：

```bash
# 版本类型配置: eim (EIM版本) 或 generic (通用版本)
DEPLOYMENT_VERSION=eim  # 或 generic
# 项目显示名称（用于UI显示）
PROJECT_DISPLAY_NAME=EIM  # 或 X
```

## 启动脚本

### 开发环境

```bash
# 启动开发环境 - EIM版本
./bin/start-dev-eim.sh

# 启动开发环境 - 通用版本
./bin/start-dev-generic.sh
```

### 测试环境

```bash
# 启动测试环境 - EIM版本
./bin/start-staging-eim.sh

# 启动测试环境 - 通用版本
./bin/start-staging-generic.sh
```

### 生产环境

```bash
# 启动生产环境 - EIM版本
./bin/start-prod-eim.sh

# 启动生产环境 - 通用版本
./bin/start-prod-generic.sh
```

## 版本差异

### EIM版本
- `DEPLOYMENT_VERSION=eim`
- `PROJECT_DISPLAY_NAME=EIM`
- UI显示EIM品牌色 (#9e1422)
- UI显示EIM相关品牌信息

### 通用版本
- `DEPLOYMENT_VERSION=generic`
- `PROJECT_DISPLAY_NAME=X`
- UI显示通用品牌色 (#2563eb)
- UI显示通用品牌信息

## 注意事项

1. **数据通用性**: 所有数据表都是通用的，可以同时包含EIM和非EIM的数据，不进行数据隔离。

2. **版本切换**: 切换版本时，只需使用对应的启动脚本即可，系统会自动读取相应的环境配置文件。

3. **配置管理**: 每个版本的环境文件可以独立配置其他差异化的设置（如域名、数据库等）。

4. **向后兼容**: 默认版本为EIM版本，确保现有部署不受影响。

## 验证版本配置

启动后，可以通过以下方式验证版本配置：

1. **API验证**: 访问 `/api/v1/system/deployment-version` 端点查看版本信息
2. **UI验证**: 查看前端页面标题和品牌色是否符合版本配置
3. **日志验证**: 查看启动日志中的版本信息输出

## 常见问题

### Q: 如何在同一台服务器上同时运行两个版本？
A: 需要修改docker-compose配置，使用不同的端口和容器名称。建议在不同服务器或使用不同的docker-compose文件。

### Q: 版本切换后需要重新初始化数据库吗？
A: 不需要。版本配置只影响UI显示和功能开关，不影响数据库结构。

### Q: 如何修改版本配置？
A: 直接编辑对应的环境文件（如 `env.dev.eim`），修改 `DEPLOYMENT_VERSION` 和 `PROJECT_DISPLAY_NAME` 配置项，然后重启服务。

