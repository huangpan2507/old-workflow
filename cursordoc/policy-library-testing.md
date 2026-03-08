# 政策库页面数据加载优化 - 测试验证

## 测试步骤

### 1. 启动应用
```bash
cd /home/song/workspace/xproject/foundation
npm run dev
```

### 2. 打开浏览器开发者工具
1. 按 F12 打开开发者工具
2. 切换到 "Network" 标签页
3. 确保 "Preserve log" 选项已勾选

### 3. 访问政策库页面
1. 在浏览器中访问政策库页面
2. 观察 Network 标签页中的请求

### 4. 验证初始加载优化
**预期结果**：
- 初始加载时只应该有少量请求（function roles、business units等元数据）
- 不应该有大量的政策数据请求

### 5. 测试按需加载
依次点击不同的 Quick Links，观察 Network 请求：

#### 5.1 点击 "Explore" 按钮
**预期结果**：
- 应该看到新的 API 请求（如 `/api/v1/policies/?status=approved`）
- 请求应该包含 `status=approved` 参数

#### 5.2 点击 "My Policies" 按钮
**预期结果**：
- 应该看到新的 API 请求（如 `/api/v1/policies/`）
- 前端会过滤出当前用户创建的政策

#### 5.3 点击 "My Approval" 按钮
**预期结果**：
- 应该看到新的 API 请求（如 `/api/v1/policies/my-pending-approvals`）

### 6. 验证缓存机制
1. 再次点击相同的 Quick Link
2. **预期结果**：不应该有新的网络请求（因为数据已缓存）

### 7. 验证数据实时性
1. 在另一个浏览器标签页中修改政策数据
2. 回到政策库页面，点击对应的 Quick Link
3. **预期结果**：应该看到新的网络请求，获取最新数据

## 成功标准

✅ **初始加载优化**：页面进入时只加载必要数据，不加载所有政策数据

✅ **按需加载**：用户点击 Quick Link 时才加载对应数据

✅ **网络请求可见**：在 Network 标签页中能看到对应的 API 请求

✅ **缓存机制**：重复点击相同 Quick Link 不会重复请求

✅ **数据实时性**：每次切换都能获取最新数据

## 故障排除

### 如果看不到网络请求
1. 检查 Network 标签页是否已打开
2. 确保 "Preserve log" 选项已勾选
3. 刷新页面重新测试

### 如果看到错误
1. 检查浏览器控制台是否有错误信息
2. 检查后端服务是否正常运行
3. 检查 API 端点是否正确

### 如果数据不更新
1. 检查缓存机制是否正常工作
2. 检查 API 响应是否正确
3. 检查前端数据过滤逻辑

## 技术细节

### API 端点
- 已发布政策：`GET /api/v1/policies/?status=approved`
- 我的政策：`GET /api/v1/policies/`（前端过滤）
- 待审批：`GET /api/v1/policies/my-pending-approvals`

### 缓存策略
- 每个数据源独立缓存
- 避免重复加载相同数据
- 支持手动刷新获取最新数据
