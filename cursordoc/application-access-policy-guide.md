# Application Access Policy 配置指南

## 问题描述

当使用 Client Credentials Flow 访问 Microsoft Graph API 时，可能会遇到以下错误：

```
403 ErrorAccessDenied
Access to OData is disabled: [RAOP] : Blocked by tenant configured AppOnly AccessPolicy settings.
```

这表示租户配置了应用访问策略（Application Access Policy），阻止了应用使用应用级权限访问 Graph API。

## 官方文档链接

### 1. Application Access Policy（旧版功能）

**文档链接**: https://learn.microsoft.com/zh-cn/exchange/permissions-exo/application-access-policies

**说明**: 
- 这是 Exchange Online 中的应用程序访问策略文档
- 注意：此功能已被基于角色的访问控制（RBAC）取代
- 但仍可用于配置应用访问限制

### 2. Exchange 应用程序的基于角色的访问控制（推荐）

**文档链接**: https://learn.microsoft.com/zh-cn/exchange/permissions-exo/role-based-access-control

**说明**:
- Microsoft 推荐使用 RBAC 来管理应用程序权限
- 这是 Application Access Policy 的现代替代方案

### 3. Microsoft Graph 最佳实践

**文档链接**: https://learn.microsoft.com/zh-cn/graph/best-practices-concept

**说明**:
- 包含权限和授权的最佳实践
- 帮助理解如何安全地访问 Graph API

### 4. 配置应用程序对联机会议或虚拟事件的访问

**文档链接**: https://learn.microsoft.com/zh-cn/graph/cloud-communication-online-meeting-application-access-policy

**说明**:
- 展示了如何配置应用程序访问策略的示例
- 虽然针对联机会议，但配置方法类似

## 解决方案

### 方案1: 使用 Exchange Online PowerShell 配置 Application Access Policy

如果租户已经配置了 Application Access Policy 来限制访问，需要：

1. **检查现有策略**:
   ```powershell
   Get-ApplicationAccessPolicy
   ```

2. **创建允许策略**:
   ```powershell
   # 允许应用访问特定邮箱组
   New-ApplicationAccessPolicy -AppId "<Your-App-ID>" -PolicyScopeGroupId "<Group-Name>" -AccessRight RestrictAccess -Description "Allow app to access specific mailboxes"
   ```

3. **或者删除限制策略**（如果策略过于严格）:
   ```powershell
   Remove-ApplicationAccessPolicy -Identity "<Policy-Identity>"
   ```

### 方案2: 使用 Exchange RBAC（推荐）

1. **创建应用角色分配策略**:
   ```powershell
   New-ApplicationAccessPolicy -AppId "<Your-App-ID>" -PolicyScopeGroupId "<Group-Name>" -AccessRight RestrictAccess
   ```

2. **验证策略**:
   ```powershell
   Get-ApplicationAccessPolicy | Where-Object {$_.AppId -eq "<Your-App-ID>"}
   ```

### 方案3: 联系租户管理员

如果无法直接修改策略，需要联系租户管理员：

1. **说明需求**:
   - 应用需要使用 Mail.Send 应用权限
   - 需要允许应用使用 Client Credentials Flow 访问 Graph API

2. **提供信息**:
   - 应用 ID (Client ID): `ca9147f1-7b3c-4639-95dd-76298c795469`
   - 需要访问的邮箱: `admin@internal.devops.local`
   - 需要的权限: `Mail.Send` (Microsoft Graph 应用权限)

3. **请求操作**:
   - 检查并修改 Application Access Policy
   - 或使用 Exchange RBAC 配置应用权限
   - 确保应用可以访问 Graph API

## 注意事项

1. **Application Access Policy 是旧功能**:
   - Microsoft 建议迁移到基于角色的访问控制（RBAC）
   - 但旧策略仍可能影响应用访问

2. **租户级别策略**:
   - 这些策略是租户级别的配置
   - 需要 Exchange 管理员权限才能修改

3. **策略优先级**:
   - 如果存在限制策略，会阻止应用访问
   - 需要创建允许策略或删除限制策略

## 替代方案

如果无法修改租户策略，可以考虑：

1. **使用 SMTP OAuth**:
   - 不受 Application Access Policy 限制
   - 但需要 Exchange 服务主体注册

2. **使用 Authorization Code Flow**:
   - 使用用户委托权限
   - 不受 AppOnly AccessPolicy 限制

## 参考资源

- [Application Access Policies (Legacy)](https://learn.microsoft.com/en-us/exchange/permissions-exo/application-access-policies)
- [Exchange Application Role-Based Access Control](https://learn.microsoft.com/en-us/exchange/permissions-exo/role-based-access-control)
- [Microsoft Graph Best Practices](https://learn.microsoft.com/en-us/graph/best-practices-concept)
- [Control Access to EWS in Exchange](https://learn.microsoft.com/en-us/exchange/client-developer/exchange-web-services/how-to-control-access-to-ews-in-exchange)


