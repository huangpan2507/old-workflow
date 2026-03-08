# Workflow 文件上传与解析时序详解

## 问题

**用户疑问**：为什么点击 "Run Workflow" 后，workflow 日志立即就打印到了 `_poll_dag_status`？文件上传后不是应该还有文件解析过程吗？

## 答案

**文件解析在 workflow 执行之前就已经完成了！**

## 完整流程时序图

```
┌─────────────────────────────────────────────────────────────────┐
│ 阶段1: 文件上传与解析（在 Run Workflow 之前）                    │
└─────────────────────────────────────────────────────────────────┘

用户操作:
  1. 配置 App Node，选择 "File Upload" 模式
  2. 选择本地文件并上传
  3. 点击上传按钮

前端处理 (FlowWorkspace.tsx:764-777):
  ↓
  ReportService.uploadFilesExtractOnly()
  ↓
  调用后端 API: POST /api/v1/report/{app_node_id}/upload-files-extract-only

后端处理 (report.py:531-764):
  ↓
  1. 接收文件内容
  2. 创建 FileIngestionRecord (status="pending")
  3. 缓存文件到 Redis
  4. 调用 document_parser 服务提取文本 ⭐ 关键步骤
  5. 保存提取的文本到 MinIO
  6. 保存原始文件到 MinIO (可选)
  7. 更新 FileIngestionRecord (status="extracted")
  8. 返回 file_ingestion_record_ids

前端接收:
  ↓
  保存 file_ingestion_record_ids 到节点配置
  节点状态更新为 "extracted" ✅

┌─────────────────────────────────────────────────────────────────┐
│ 阶段2: Workflow 执行（点击 Run Workflow 之后）                  │
└─────────────────────────────────────────────────────────────────┘

用户操作:
  点击 "Run Workflow" 按钮

前端处理:
  ↓
  发送 workflow 执行请求，包含 app_node_configs (含 fileIngestionRecordIds)

后端处理 (workflow_execution_service.py):
  ↓
  App Node 执行:
    1. 从 app_node_configs 获取 fileIngestionRecordIds
    2. 从 FileIngestionRecord 读取已提取的文本 (从 MinIO) ⚡ 快速
    3. 直接触发 DAG (使用已提取的文本)
    4. 立即开始 _poll_dag_status ⚡ 快速
```

## 详细步骤说明

### 阶段1: 文件上传与解析（Run Workflow 之前）

#### 1.1 前端上传文件

**位置**: `frontend/src/components/Workbench/FlowWorkspace.tsx:764-777`

```typescript
// 用户选择文件后，点击上传
const uploadResp = await ReportService.uploadFilesExtractOnly(
  appData.appNodeId || 0,
  executionState.files,  // 本地文件列表
  executionState.requirement || ''
)

// 返回 file_ingestion_record_ids
const fileIngestionRecordIds = uploadResp.file_ingestion_record_ids
fileIngestionRecordIdsMap[node.id] = fileIngestionRecordIds

// 节点状态更新为 'extracted'
status: 'extracted'  // ✅ 文件已提取完成
```

#### 1.2 后端处理文件提取

**位置**: `backend/app/api/v1/report.py:531-764`

**API**: `POST /api/v1/report/{app_node_id}/upload-files-extract-only`

**处理流程**:

```python
# 1. 接收文件
file_content = file.file.read()

# 2. 创建 FileIngestionRecord
file_record = FileIngestionRecord(
    file_name=file.filename,
    status="pending",
    ...
)

# 3. 缓存到 Redis
cache_id = redis_service.cache_file_content(file_content, file.filename)

# 4. ⭐ 调用 document_parser 服务提取文本
extraction_result = await document_parser_adapter.extract_text_from_file(
    cache_id, file.filename
)
extracted_text = extraction_result.get("extracted_text", "")

# 5. 保存提取的文本到 MinIO
extracted_text_path = file_retention_service.save_extracted_text(
    extracted_text=extracted_text,
    filename=file.filename,
    record_id=file_record.id
)
file_record.extracted_text_storage_path = extracted_text_path

# 6. 保存原始文件到 MinIO (可选)
storage_path = file_retention_service.save_file_for_retention(...)
file_record.file_storage_path = storage_path

# 7. 更新状态为 "extracted"
file_record.status = "extracted"  # ✅ 提取完成

# 8. 返回 file_ingestion_record_ids
return {
    "file_ingestion_record_ids": [file_record.id, ...]
}
```

**日志输出** (从你的日志可以看到):

```
INFO:app.api.v1.report:[WF][ExtractOnly] Caching file content to Redis: 供应商C基本信息.docx (record_id: 342)
INFO:app.api.v1.report:[WF][ExtractOnly] Calling document_parser service: 供应商C基本信息.docx
INFO:app.api.v1.report:[WF][ExtractOnly] Text extraction successful: 供应商C基本信息.docx, length=158
INFO:app.api.v1.report:[WF][ExtractOnly] Extracted text saved to MinIO: file-retention/342/供应商C基本信息.txt
INFO:app.api.v1.report:[WF][ExtractOnly] Original file saved to MinIO: file-retention/342/0aed096b_供应商C基本信息.docx
INFO:app.api.v1.report:[WF][ExtractOnly] Completed: app_node_id=105, successful=1, failed=0, file_ingestion_record_ids=[342]
```

**关键点**:
- ✅ 文件解析在 workflow 执行之前完成
- ✅ 提取的文本已保存到 MinIO
- ✅ FileIngestionRecord 状态为 "extracted"
- ✅ 返回 `file_ingestion_record_ids=[342]`

---

### 阶段2: Workflow 执行（Run Workflow 之后）

#### 2.1 前端发送执行请求

**位置**: `frontend/src/components/Workbench/FlowWorkspace.tsx:handleRunWorkflow()`

```typescript
// 构建 app_node_configs，包含 fileIngestionRecordIds
const appNodeConfigs = {
  'app-1': {
    fileIngestionRecordIds: [342],  // 从阶段1获取
    inputMode: 'file',
    requirement: '...',
    appNodeId: 105
  }
}

// 发送执行请求
await executeWorkflow({
  nodes: workflowNodes,
  edges: workflowEdges,
  app_node_configs: appNodeConfigs,  // 包含 fileIngestionRecordIds
  input_data: {}
})
```

#### 2.2 后端 App Node 执行

**位置**: `backend/app/services/workflow/workflow_execution_service.py:654-774`

**处理流程**:

```python
# 1. 从 app_node_configs 获取 fileIngestionRecordIds
file_ingestion_record_ids = local_app_config.get('fileIngestionRecordIds', [])
# file_ingestion_record_ids = [342]

# 2. ⚡ 从 FileIngestionRecord 读取已提取的文本 (从 MinIO)
# 注意：文本已经在阶段1提取完成，这里只是读取
node_files = ChainCallDataService._get_file_contents_from_ids(
    file_ingestion_record_ids, session, extract_text=True
)
# 从 MinIO 读取: file-retention/342/供应商C基本信息.txt
# 内容: 已提取的文本 (158 字符)

# 3. 构建 file_content JSON
all_file_contents.append({
    "file_name": "供应商C基本信息.docx",
    "file_content": "已提取的文本内容..."  # 从 MinIO 读取
})

# 4. ⚡ 直接触发 DAG (使用已提取的文本)
merged_file_content = json.dumps(all_file_contents, ensure_ascii=False)
dag_run_id = await trigger_agent_analysis_unified(
    app_node_id,
    file_content=merged_file_content,  # 已提取的文本
    user_id=user_id,
    requirement=requirement
)

# 5. ⚡ 立即开始轮询 DAG 状态
logger.info(f"Starting _poll_dag_status, dag_run_id={dag_run_id}")
result = await self._poll_dag_status(app_node_id, dag_run_id, session, max_wait_time=600)
```

**日志输出** (从你的日志可以看到):

```
INFO:app.services.workflow.workflow_execution_service:[WF][app] node_id=app-1 Getting file content from FileIngestionRecord IDs: [342]
INFO:app.services.workflow.chain_call_data_service:[WF][ChainCall] Successfully got file content for file_id=342, bucket=file-retention, path=file-retention/342/供应商C基本信息.txt, length=158
INFO:app.services.workflow.workflow_execution_service:[WF][app] node_id=app-1 Added 1 node files from FileIngestionRecord to merge list
INFO:app.services.workflow.workflow_execution_service:[WF][app] node_id=app-1 File mode with merged file_content: triggered DAG with 1 files
INFO:app.services.workflow.workflow_execution_service:[WF][app] node_id=app-1 Starting _poll_dag_status, dag_run_id=manual__2026-01-26T05:41:13.176288+00:00, app_node_id=105
```

**关键点**:
- ⚡ 文件内容已提取，只需从 MinIO 读取（快速）
- ⚡ 不需要再次调用 document_parser 服务
- ⚡ 直接触发 DAG，立即开始轮询

---

## 为什么这么快？

### 时间线对比

#### 旧设计（已废弃）:
```
上传文件 → 创建 FileIngestionRecord → 触发 DAG → DAG 内部提取文本 → 分析
         [慢]                      [慢]         [慢]              [慢]
```

#### 新设计（当前实现）:
```
阶段1 (上传时): 上传文件 → 提取文本 → 保存到 MinIO → 返回 IDs
               [慢]      [慢]      [快]            [快]
               
阶段2 (执行时): 读取已提取文本 → 触发 DAG → 分析
               [快]            [快]      [慢]
```

### 性能优势

1. **文件解析提前完成**
   - 在用户点击 "Run Workflow" 之前就完成
   - 用户等待时间分散（上传时等待，而不是执行时等待）

2. **Workflow 执行更快**
   - 只需从 MinIO 读取已提取的文本（毫秒级）
   - 不需要等待 document_parser 服务响应（秒级）
   - 直接触发 DAG，立即开始分析

3. **更好的用户体验**
   - 上传时显示 "Uploading and extracting files..."
   - 执行时显示 "Running workflow..."（更快）

---

## 数据流图

```
┌─────────────────────────────────────────────────────────────┐
│ 阶段1: 文件上传与解析                                        │
└─────────────────────────────────────────────────────────────┘

本地文件 (供应商C基本信息.docx)
    ↓
前端上传 (uploadFilesExtractOnly)
    ↓
后端接收 (report.py:upload_files_extract_only)
    ↓
创建 FileIngestionRecord (id=342, status="pending")
    ↓
缓存到 Redis
    ↓
调用 document_parser 服务 ⭐ 提取文本
    ↓
提取成功 (158 字符)
    ↓
保存到 MinIO: file-retention/342/供应商C基本信息.txt
    ↓
更新 FileIngestionRecord (status="extracted")
    ↓
返回 file_ingestion_record_ids=[342]
    ↓
前端保存到节点配置

┌─────────────────────────────────────────────────────────────┐
│ 阶段2: Workflow 执行                                         │
└─────────────────────────────────────────────────────────────┘

点击 "Run Workflow"
    ↓
发送执行请求 (包含 fileIngestionRecordIds=[342])
    ↓
App Node 执行
    ↓
从 FileIngestionRecord 读取已提取文本 ⚡ (从 MinIO)
    ↓
构建 file_content JSON
    ↓
触发 DAG (使用已提取的文本)
    ↓
立即开始 _poll_dag_status ⚡
    ↓
等待 DAG 分析完成
```

---

## 关键代码位置

### 文件上传与解析

1. **前端**: `frontend/src/components/Workbench/FlowWorkspace.tsx:764-777`
   - `ReportService.uploadFilesExtractOnly()`

2. **后端 API**: `backend/app/api/v1/report.py:531-764`
   - `upload_files_extract_only()` 方法
   - 调用 `document_parser` 服务提取文本
   - 保存到 MinIO

### Workflow 执行

1. **App Node 执行**: `backend/app/services/workflow/workflow_execution_service.py:654-774`
   - 从 `fileIngestionRecordIds` 读取已提取文本
   - 使用 `ChainCallDataService._get_file_contents_from_ids()`

2. **文件内容获取**: `backend/app/services/workflow/chain_call_data_service.py:207-275`
   - `_get_file_contents_from_ids()` 方法
   - 从 MinIO 读取已提取的文本

---

## 总结

### 回答你的问题

**Q: 为什么这么快就到了 `_poll_dag_status`？**

**A**: 因为文件解析在 workflow 执行之前就已经完成了！

1. **上传阶段**（点击上传按钮时）:
   - 文件上传到服务器
   - 立即调用 `document_parser` 服务提取文本
   - 提取的文本保存到 MinIO
   - 创建 `FileIngestionRecord`，状态为 "extracted"
   - 返回 `file_ingestion_record_ids`

2. **执行阶段**（点击 Run Workflow 时）:
   - App Node 从 `FileIngestionRecord` 读取已提取的文本（从 MinIO）
   - 直接触发 DAG（使用已提取的文本）
   - 立即开始 `_poll_dag_status`（不需要等待文件解析）

**Q: 此时用的是我从本地电脑上传的文件吗？**

**A**: 是的，但使用的是**已提取的文本版本**，不是原始文件。

- 原始文件: 保存在 MinIO (`file-retention/342/0aed096b_供应商C基本信息.docx`)
- 提取的文本: 保存在 MinIO (`file-retention/342/供应商C基本信息.txt`)
- App Node 使用: **提取的文本**（158 字符）

**Q: 本地上传后，不是应该还有 files ingestion 解析文件过程吗？**

**A**: 有，但这个过程在**上传阶段**就完成了，不是在 workflow 执行阶段。

- ✅ 上传时: 文件解析已完成（`status="extracted"`）
- ✅ 执行时: 直接使用已提取的文本（快速）

---

## 设计优势

1. **性能优化**: 文件解析提前完成，workflow 执行更快
2. **用户体验**: 上传时等待解析，执行时快速响应
3. **资源利用**: 文件解析可以并行处理多个文件
4. **错误处理**: 上传时发现文件解析失败，可以立即提示用户

---

## 相关文档

- `workflow-app-node-complete-process.md` - App Node 完整流程
- `workflow-chain-call-design.md` - 链式调用设计
- `workflow-accumulated-context-feature.md` - 累计上下文功能
