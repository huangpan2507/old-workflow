# Approval Workflow API 迁移修复

## 问题描述

前端访问 Approval Workflow 页面时报 404 错误：
```
GET /api/v1/policy-approval-templates/ 
Status: 404 Not Found
```

原因：在清理旧代码时，`policy_approval_templates.py` 和 `approval_nodes.py` 被删除，但前端仍在使用这些旧的 API 端点。

## 解决方案

创建新的通用审批流模板 API 端点，替代旧的政策审批模板端点。

### 1. 创建新的 API 文件

**文件：** `backend/app/api/v1/approval_workflow_templates.py`

**端点：**
- `GET /api/v1/approval-workflow-templates/` - 列出所有模板
- `GET /api/v1/approval-workflow-templates/{template_id}` - 获取单个模板
- `GET /api/v1/approval-workflow-templates/{template_id}/nodes` - 获取模板的节点定义
- `GET /api/v1/approval-workflow-templates/default/{resource_type}` - 获取默认模板

### 2. 注册新路由

在 `backend/app/api/v1/__init__.py` 中：
```python
from .approval_workflow_templates import router as approval_workflow_templates_router

api_router.include_router(
    approval_workflow_templates_router, 
    prefix="/approval-workflow-templates", 
    tags=["approval-workflow-templates"]
)
```

### 3. API 差异

**旧 API (已删除):**
- `/api/v1/policy-approval-templates/` - 政策审批模板
- 仅支持 `policy` 资源类型

**新 API:**
- `/api/v1/approval-workflow-templates/` - 通用审批流模板
- 支持多种资源类型：`policy`, `contract`, `leave_request` 等
- 可以使用 `?resource_type=policy` 过滤

### 4. 前端适配建议

前端需要更新 API 调用：

**旧方式：**
```typescript
GET /api/v1/policy-approval-templates/
```

**新方式（获取政策审批模板）：**
```typescript
// 方式1：获取所有政策审批模板
GET /api/v1/approval-workflow-templates/?resource_type=policy

// 方式2：获取默认模板
GET /api/v1/approval-workflow-templates/default/policy

// 方式3：获取所有模板
GET /api/v1/approval-workflow-templates/
```

## 数据兼容性

- 新 API 返回的数据结构与旧 API 类似
- 增加 `resource_type` 字段
- 节点数据结构保持一致
- 角色关联信息包含在响应中

## 测试状态

- ✅ API 端点已创建
- ✅ 路由已注册
- ✅ 无 lint 错误
- ⏳ 待前端适配新 API

## 下一步

1. 更新前端代码以使用新的 API 端点
2. 测试 Approval Workflow 页面功能
3. 验证模板创建、编辑、删除等功能

