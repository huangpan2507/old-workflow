# Policy Approval Data Analysis Report

## 问题描述

政策提交审核时报错："No approvers found for step Department Head Approval"

## 数据层面诊断结果

### 1. 政策数据检查

通过诊断脚本检查，发现数据库中：

**测试政策 (ID: 1, 名称: test)**
- ✅ Responsible Function Role ID: 11 (Information Technology)
- ✅ 审批状态: draft
- ✅ 创建者存在

### 2. FunctionRole 检查

- ✅ FunctionRole ID 11 存在
- ✅ 名称: Information Technology
- ✅ Code: IT

### 3. Group 检查

- ✅ Group (eim_global) 存在
- ✅ ID: 1
- ✅ Code: eim_global

### 4. OrganizationFunctionHead 检查

**查询条件：**
- group_id = 1
- function_role_id = 11
- is_active = True
- effective_date <= today 或为 null
- expiry_date >= today 或为 null

**查询结果：**
- ✅ 找到 1 条活跃记录
- ✅ ID: 5
- ✅ User: group.function.head.it@eim.global
- ✅ User ID: 0357d393-8eec-4ce5-961c-14ba37aa69af
- ✅ User Active: True
- ✅ Effective Date: 2025-11-04 (有效)
- ✅ Expiry Date: None (无过期)

### 5. 审批流程模板检查

- ✅ 默认审批流程模板存在
- ✅ Step 1: Department Head Approval (group_function_role_head)
- ✅ Step 2: Policy Library Administrator Approval (roles)

## 结论

**从数据层面看，所有必需的数据都存在且有效！**

1. ✅ Policy 有 responsible_function_role_id
2. ✅ FunctionRole 存在
3. ✅ Group 存在
4. ✅ OrganizationFunctionHead 存在、活跃、日期有效
5. ✅ 审批人用户存在且活跃

## 可能的原因

### 1. 后端代码未重启

**最可能的原因**：代码修改后，后端服务没有重启，仍然在使用旧的代码逻辑。

**解决方案**：
```bash
# 重启后端容器
docker restart foundation-backend-container

# 或者如果使用 docker-compose
docker-compose restart backend
```

### 2. 错误消息被截断

从浏览器开发者工具看到的错误消息是：
```json
{
  "detail": "Failed to submit policy for approval: No approvers found for step Department He"
}
```

注意 "Department He" 被截断了，说明可能是：
- HTTP 响应大小限制
- 前端错误消息显示限制
- 数据库查询结果被截断

### 3. 代码逻辑问题

虽然诊断脚本显示数据存在，但实际代码执行时可能：
- 使用了不同的数据库连接
- 事务隔离级别导致数据不可见
- 缓存问题

## 建议的修复步骤

### 步骤 1: 确认后端代码已更新

检查后端容器中的代码是否是最新的：

```bash
# 检查文件修改时间
docker exec foundation-backend-container ls -la /app/app/services/approval_workflow_service.py

# 检查关键代码行
docker exec foundation-backend-container grep -n "_find_policy_group_function_role_head" /app/app/services/approval_workflow_service.py
```

### 步骤 2: 重启后端服务

```bash
docker restart foundation-backend-container
```

### 步骤 3: 检查日志

查看后端日志，确认错误详情：

```bash
docker logs foundation-backend-container --tail 100 | grep -i "approver\|error"
```

### 步骤 4: 验证修复

重新提交政策，检查错误消息是否包含详细信息。

## 代码改进

已完成的代码改进：

1. **改进错误消息** (`_find_policy_group_function_role_head`)：
   - 提供详细的错误诊断信息
   - 检查非活跃、已过期、未来生效的记录
   - 包含政策名称和职能部门名称

2. **确保异常传播** (`create_approval_instance`)：
   - 使用 `raise ValueError(str(e)) from e` 确保原始错误消息被保留

3. **添加日志记录**：
   - 记录详细错误信息以便调试

## 测试验证

使用诊断脚本验证数据：

```bash
docker exec foundation-backend-container python /tmp/check_policy_approval_data.py
```

## 下一步

如果重启后端后问题仍然存在，需要：

1. 检查实际的 SQL 查询是否执行成功
2. 检查是否有其他代码路径导致错误
3. 检查事务和数据库连接状态
4. 添加更详细的日志记录

