# SharePoint Site ID 和 Drive ID 手动获取指南

本文档提供手动获取 SharePoint Site ID 和 Drive ID 的方法，适用于无法通过 API 自动获取或需要手动验证的场景。

## 前提条件

- 能够访问目标 SharePoint 站点
- 具有站点访问权限
- 浏览器（推荐 Chrome 或 Edge）

## 方法一：通过 SharePoint 网站直接获取

### 步骤 1：获取 Site ID

1. **访问 SharePoint 站点**
   - 在浏览器中打开目标 SharePoint 站点
   - 例如：`https://dcigroupadmin.sharepoint.com/teams/EiMStaffPortal/`

2. **访问 Site ID API 端点**
   - 在浏览器地址栏的 URL 后面添加：`/_api/site/id`
   - 完整 URL 示例：
     ```
     https://dcigroupadmin.sharepoint.com/teams/EiMStaffPortal/_api/site/id
     ```

3. **获取 Site ID**
   - 页面会显示 JSON 格式的响应
   - 找到 `Id` 字段，这就是 `site_id`
   - 示例响应：
     ```json
     {
       "d": {
         "Id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
       }
     }
     ```
   - `site_id` = `a1b2c3d4-e5f6-7890-abcd-ef1234567890`

### 步骤 2：获取 Drive ID

#### 方法 2.1：获取默认文档库（推荐）

1. **访问默认文档库 API 端点**
   - 在浏览器地址栏输入：
     ```
     https://dcigroupadmin.sharepoint.com/teams/EiMStaffPortal/_api/web/defaultdocumentlibrary
     ```
   - 或者使用 Site ID：
     ```
     https://dcigroupadmin.sharepoint.com/teams/EiMStaffPortal/_api/web/lists/getbytitle('Documents')
     ```

2. **获取 Drive ID**
   - 页面会显示 JSON 格式的响应
   - 找到 `Id` 字段，这就是 `drive_id`
   - 示例响应：
     ```json
     {
       "d": {
         "Id": "b2c3d4e5-f6a7-8901-bcde-f12345678901"
       }
     }
     ```

#### 方法 2.2：获取所有文档库

1. **访问文档库列表 API 端点**
   - 在浏览器地址栏输入：
     ```
     https://dcigroupadmin.sharepoint.com/teams/EiMStaffPortal/_api/web/lists?$filter=BaseTemplate eq 101
     ```
   - 这会返回所有文档库（BaseTemplate = 101 表示文档库）

2. **查找目标文档库**
   - 页面会显示所有文档库的列表
   - 找到目标文档库（例如：`docshare`）
   - 记录该文档库的 `Id` 字段

## 方法二：通过 Microsoft Graph Explorer 获取

### 步骤 1：访问 Graph Explorer

1. **打开 Graph Explorer**
   - 访问：https://developer.microsoft.com/graph/graph-explorer
   - 使用 Microsoft 账户登录

2. **设置权限**
   - 点击左侧的 "Modify permissions" 按钮
   - 确保已授予以下权限：
     - `Sites.Read.All`（读取站点信息）
     - `Files.Read.All`（读取文件信息）

### 步骤 2：获取 Site ID

1. **执行查询**
   - 在查询框中输入：
     ```
     GET https://graph.microsoft.com/v1.0/sites/dcigroupadmin.sharepoint.com:/teams/EiMStaffPortal
     ```
   - 或者使用完整 URL：
     ```
     GET https://graph.microsoft.com/v1.0/sites/root:/sites/yoursite
     ```

2. **获取 Site ID**
   - 点击 "Run query" 按钮
   - 在响应 JSON 中找到 `id` 字段
   - 示例响应：
     ```json
     {
       "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
       "displayName": "EiM Staff Portal",
       "webUrl": "https://dcigroupadmin.sharepoint.com/teams/EiMStaffPortal"
     }
     ```
   - `site_id` = `a1b2c3d4-e5f6-7890-abcd-ef1234567890`

### 步骤 3：获取 Drive ID

#### 方法 3.1：获取默认文档库

1. **执行查询**
   - 在查询框中输入（使用上一步获取的 site_id）：
     ```
     GET https://graph.microsoft.com/v1.0/sites/{site-id}/drive
     ```
   - 例如：
     ```
     GET https://graph.microsoft.com/v1.0/sites/a1b2c3d4-e5f6-7890-abcd-ef1234567890/drive
     ```

2. **获取 Drive ID**
   - 点击 "Run query" 按钮
   - 在响应 JSON 中找到 `id` 字段
   - 示例响应：
     ```json
     {
       "id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
       "name": "Documents",
       "driveType": "documentLibrary"
     }
     ```
   - `drive_id` = `b2c3d4e5-f6a7-8901-bcde-f12345678901`

#### 方法 3.2：获取所有文档库

1. **执行查询**
   - 在查询框中输入：
     ```
     GET https://graph.microsoft.com/v1.0/sites/{site-id}/drives
     ```
   - 例如：
     ```
     GET https://graph.microsoft.com/v1.0/sites/a1b2c3d4-e5f6-7890-abcd-ef1234567890/drives
     ```

2. **查找目标文档库**
   - 点击 "Run query" 按钮
   - 响应会返回所有文档库的列表
   - 找到目标文档库（例如：`docshare`）
   - 记录该文档库的 `id` 字段

## 方法三：通过浏览器开发者工具获取

### 步骤 1：打开开发者工具

1. **访问 SharePoint 站点**
   - 在浏览器中打开目标 SharePoint 站点

2. **打开开发者工具**
   - 按 `F12` 或右键点击页面选择 "检查"
   - 切换到 "Network"（网络）标签页

### 步骤 2：获取 Site ID

1. **触发网络请求**
   - 刷新页面或导航到站点
   - 在 Network 标签页中查找包含 `/_api/site` 的请求

2. **查看响应**
   - 点击该请求
   - 在 "Response" 标签页中查看 JSON 响应
   - 找到 `Id` 字段

### 步骤 3：获取 Drive ID

1. **访问文档库**
   - 在 SharePoint 站点中导航到目标文档库
   - 例如：点击 "docshare" 文档库

2. **查看网络请求**
   - 在 Network 标签页中查找包含 `/_api/web/lists` 或 `/_api/drive` 的请求
   - 查看响应中的 `Id` 字段

## 针对 EiMStaffPortal 站点的具体示例

### 站点信息
- **站点 URL**: `https://dcigroupadmin.sharepoint.com/teams/EiMStaffPortal/`
- **目标文档库**: `docshare`

### 获取 Site ID

**方法 1：通过 SharePoint API**
```
https://dcigroupadmin.sharepoint.com/teams/EiMStaffPortal/_api/site/id
```

**方法 2：通过 Graph Explorer**
```
GET https://graph.microsoft.com/v1.0/sites/dcigroupadmin.sharepoint.com:/teams/EiMStaffPortal
```

### 获取 Drive ID（docshare 文档库）

**方法 1：通过 SharePoint API**
```
https://dcigroupadmin.sharepoint.com/teams/EiMStaffPortal/_api/web/lists/getbytitle('docshare')
```

**方法 2：通过 Graph Explorer（需要先获取 site_id）**
```
GET https://graph.microsoft.com/v1.0/sites/{site-id}/drives
```
然后在返回的列表中找到 `name` 为 `docshare` 的文档库，记录其 `id`。

## 验证获取的 ID

获取到 ID 后，可以使用以下方式验证：

### 验证 Site ID
```
GET https://graph.microsoft.com/v1.0/sites/{site-id}
```

### 验证 Drive ID
```
GET https://graph.microsoft.com/v1.0/sites/{site-id}/drives/{drive-id}
```

## 常见问题

### Q1: 访问 `/_api/site/id` 时显示 403 错误
**A**: 确保已登录并具有站点访问权限。如果使用匿名访问，可能需要先登录。

### Q2: Graph Explorer 中显示权限不足
**A**: 点击 "Modify permissions" 并授予 `Sites.Read.All` 权限，然后重新登录。

### Q3: 如何区分不同的文档库？
**A**: 在返回的文档库列表中，查看 `name` 字段来识别目标文档库。

### Q4: Site ID 和 Drive ID 的格式是什么？
**A**: 两者都是 GUID 格式，例如：`a1b2c3d4-e5f6-7890-abcd-ef1234567890`

## 注意事项

1. **权限要求**：
   - 通过 SharePoint API 访问需要站点访问权限
   - 通过 Graph Explorer 需要相应的 Microsoft Graph API 权限

2. **ID 的唯一性**：
   - Site ID 在整个 SharePoint 租户中是唯一的
   - Drive ID 在站点内是唯一的

3. **ID 的稳定性**：
   - Site ID 和 Drive ID 通常不会改变
   - 但如果站点被删除并重建，ID 会发生变化

4. **文档库名称**：
   - 文档库名称（如 `docshare`）可能包含空格或特殊字符
   - 在 API 调用中需要使用正确的编码

## 相关资源

- [Microsoft Graph API - Site Resource](https://learn.microsoft.com/en-us/graph/api/resources/site)
- [Microsoft Graph API - Drive Resource](https://learn.microsoft.com/en-us/graph/api/resources/drive)
- [Microsoft Graph Explorer](https://developer.microsoft.com/graph/graph-explorer)
- [SharePoint REST API 参考](https://learn.microsoft.com/en-us/sharepoint/dev/sp-add-ins/get-to-know-the-sharepoint-rest-service)




