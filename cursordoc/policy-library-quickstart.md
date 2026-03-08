# Policy Library - Quick Start Guide

## 概述

政策库管理系统是一个完整的企业政策管理解决方案，支持政策上传、在线审批、权限控制和智能检索等功能。

## 主要功能

### 1. 政策发布流程
- 用户上传政策文件并填写表单
- 系统根据负责部门自动分配审批人
- 审批人在线审批政策
- 审批通过后政策发布给相关人员

### 2. 权限控制
- 基于业务单元(BU)和部门的访问控制
- 只有适用的BU和部门的用户才能查看政策
- 支持多BU、多部门的灵活配置

### 3. 审批工作流
- 自动识别部门负责人作为审批人
- 支持审批、拒绝并添加评论
- 完整的审批历史记录

### 4. 搜索和检索
- 按政策名称、摘要搜索
- 按状态、部门过滤
- 快速定位需要的政策

## 快速开始

### 步骤 1: 设置数据库

```bash
# 进入后端目录
cd /home/song/workspace/xproject/foundation/backend

# 运行数据库迁移
alembic upgrade head
```

### 步骤 2: 配置环境变量

确保以下环境变量已正确配置：

```bash
# MinIO配置
MINIO_ENDPOINT=localhost:9000
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=minioadmin
MINIO_SECURE=false
POLICY_LIBRARY_BUCKET_NAME=policy-library

# 数据库配置
DATABASE_URL=postgresql://user:password@localhost/dbname
```

### 步骤 3: 初始化组织架构

在使用政策库之前，需要先设置组织架构：

1. **创建部门 (Functions)**
   - 财务部
   - IT部
   - 人力资源部
   - 等等...

2. **创建业务单元 (Business Units)**
   - 各个子公司或学校

3. **分配部门负责人**
   - 为每个部门设置部门主管 (employee_level = "function_head")

### 步骤 4: 上传第一个政策

1. 访问政策库页面: `/policy-library`
2. 点击 "Policies" 标签
3. 点击 "New Policy" 按钮
4. 填写表单：
   - **Revision Type**: 选择 "New" 或 "Revised"
   - **Policy Name**: 输入政策名称 (必填)
   - **Policy Summary**: 输入政策摘要
   - **Responsible Department**: 选择负责部门 (必填)
   - **Applicable Business Units**: 选择适用的BU (必填，可多选)
   - **Applicable Departments**: 选择适用的部门 (可选)
   - **Effective Date**: 选择生效日期 (必填)
   - **Valid Until**: 选择失效日期 (可选)
   - **Person-In-Charge**: 输入负责人姓名
   - **Attachment**: 上传政策文件
5. 点击 "Submit" 提交

### 步骤 5: 审批政策

作为审批人：

1. 访问政策库页面
2. 在 "Policies" 标签中查看待审批的政策（状态为 "Pending"）
3. 点击 "Review" 按钮
4. 查看政策详情
5. 下载并审阅政策文件
6. 输入审批意见（可选）
7. 点击 "Approve" 或 "Reject"

### 步骤 6: 查看和下载政策

作为普通用户：

1. 访问政策库页面
2. 使用搜索框查找政策
3. 使用状态过滤器筛选 "Approved" 政策
4. 点击 "View Details" 查看政策详情
5. 点击下载按钮下载政策文件

## 表单字段说明

### 必填字段

| 字段 | 说明 | 示例 |
|------|------|------|
| Revision or New | 是新政策还是修订政策 | "New" 或 "Revised" |
| Policy Name | 政策名称 | "IT Security Policy 2024" |
| Responsible Department | 负责部门 | "IT Department" |
| Applicable Business Units | 适用的业务单元 | 选择一个或多个BU |
| Effective Date | 生效日期 | "2024-01-01" |

### 可选字段

| 字段 | 说明 | 示例 |
|------|------|------|
| Policy Summary | 政策摘要 | "Updated security guidelines..." |
| Applicable Departments | 适用的部门 | 仅当政策只适用于特定部门时选择 |
| Valid Until | 失效日期 | "2025-12-31" |
| Person-In-Charge | 负责人 | "John Doe" |
| Attachment | 政策文件 | 上传PDF等文档 |

## 政策状态说明

| 状态 | 说明 | 可执行操作 |
|------|------|-----------|
| Draft | 草稿 | 编辑、提交审批、删除 |
| Pending | 待审批 | 审批、拒绝 (仅审批人) |
| Approved | 已批准 | 查看、下载 (所有适用用户) |
| Rejected | 已拒绝 | 查看拒绝原因、重新提交 |

## 权限说明

### 查看权限

用户可以查看以下政策：

1. **自己创建的政策** (任何状态)
2. **待自己审批的政策** (Pending状态)
3. **已批准且适用于自己BU/部门的政策** (Approved状态)
4. **超级管理员** 可以查看所有政策

### 审批权限

- 只有部门主管 (function_head) 可以审批该部门的政策
- 系统自动根据"负责部门"分配审批人
- 当前审批人可以批准或拒绝政策

### 编辑权限

- 政策创建者可以编辑自己的政策 (未批准状态)
- 超级管理员可以编辑任何政策
- 已批准的政策不能编辑

### 删除权限

- 政策创建者可以删除自己的政策
- 超级管理员可以删除任何政策
- 删除操作是软删除，不会真正删除数据

## 审批工作流

### 自动审批流程

```
1. 用户创建政策
   ↓
2. 系统根据"负责部门"查找部门主管
   ↓
3. 设置部门主管为当前审批人
   ↓
4. 创建审批流程记录
   ↓
5. 政策状态设为 "Pending"
   ↓
6. 通知审批人 (未来功能)
```

### 审批决策

**批准流程:**
```
1. 审批人查看政策
   ↓
2. 下载并审阅政策文件
   ↓
3. 输入审批意见
   ↓
4. 点击 "Approve"
   ↓
5. 政策状态变更为 "Approved"
   ↓
6. 政策对适用用户可见
```

**拒绝流程:**
```
1. 审批人查看政策
   ↓
2. 输入拒绝原因
   ↓
3. 点击 "Reject"
   ↓
4. 政策状态变更为 "Rejected"
   ↓
5. 创建者收到通知并可重新提交
```

## 常见问题

### Q1: 为什么我看不到某些政策？

**A:** 可能的原因：
- 政策尚未批准（只有创建者和审批人可以看到待审批的政策）
- 政策不适用于您的BU或部门
- 您没有查看该政策的权限

### Q2: 谁可以审批我的政策？

**A:** 审批人是根据"负责部门"自动确定的，通常是该部门的部门主管 (function_head level)

### Q3: 如何修改已提交的政策？

**A:** 
- 如果政策处于 "Pending" 状态，需要先联系审批人拒绝
- 政策被拒绝后，您可以修改并重新提交
- 已批准的政策不能修改，需要创建新的修订版本

### Q4: 政策文件格式有什么限制？

**A:**
- 文件大小限制: 100MB
- 支持的格式: PDF, Word, Excel, PPT等
- 建议使用PDF格式以确保兼容性

### Q5: 如何设置政策适用范围？

**A:**
- **Applicable Business Units**: 选择政策适用的所有BU
- **Applicable Departments**: 仅当政策只适用于特定部门时选择
- 如果政策适用于所有员工，选择所有BU

## 技术架构

### 前端组件

- `PolicyUploadForm` - 政策上传表单
- `PolicyList` - 政策列表和搜索
- `PolicyDetailModal` - 政策详情和审批
- `PolicyApprovals` - 待审批政策列表
- `PolicyFileManager` - 文件管理器

### 后端API

- `/api/v1/policies/` - 政策CRUD操作
- `/api/v1/policies/{id}/submit` - 提交审批
- `/api/v1/policies/{id}/approve` - 审批政策
- `/api/v1/policies/my-approvals/pending` - 待审批列表
- `/api/v1/policy-library/` - 文件存储操作

### 数据库表

- `policies` - 政策主表
- `policy_approval_flows` - 审批流程表
- `policy_access_logs` - 访问日志表

## 下一步

1. **集成通知系统**
   - 邮件通知待审批政策
   - 审批结果通知
   - 政策到期提醒

2. **AI问答功能**
   - 连接知识库
   - 自然语言查询政策
   - 智能推荐相关政策

3. **高级搜索**
   - 全文搜索
   - 多条件组合查询
   - 保存搜索条件

4. **版本管理**
   - 政策版本追踪
   - 版本对比
   - 回滚功能

5. **分析报告**
   - 访问统计
   - 审批效率分析
   - 合规性报告

## 支持

如有问题或建议，请联系：
- 技术支持邮箱: support@company.com
- API文档: http://localhost:8001/docs
- 用户手册: /cursordocs/policy-library-implementation.md

