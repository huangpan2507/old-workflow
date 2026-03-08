# Human In The Loop Node 链式调用澄清

## 一、request_for_update → approval 场景分析

### 1.1 两种情况的详细说明

#### 情况1: 上传了文件

**用户操作**:
- 用户在消息中心收到 `request_for_update` 任务
- 用户上传了文件（附件）
- 用户填写了 comments（可选）
- 用户提交

**数据存储**:
```python
output.data = {
    'human_decision': 'updated',
    'decision_reason': '已上传文件',
    'update_info': '用户填写的更新信息',
    'file_ingestion_record_ids': [123, 456]  # 上传的文件ID列表
}
```

---

#### 情况2: 只填写了 comments，没有上传文件

**用户操作**:
- 用户在消息中心收到 `request_for_update` 任务
- 用户**没有上传文件**
- 用户只填写了 comments（说明信息）
- 用户提交

**数据存储**:
```python
output.data = {
    'human_decision': 'updated',
    'decision_reason': '用户填写的说明信息',
    'update_info': '用户填写的更新信息',
    'file_ingestion_record_ids': []  # 空列表，没有文件
}
```

---

### 1.2 下游 approval 节点的处理

**场景**: Human In The Loop Node (request_for_update) → Human In The Loop Node (approval)

**审批人的操作**:
1. 在消息中心收到 `approval` 任务
2. **如果有文件**（`file_ingestion_record_ids` 不为空）:
   - 可以预览之前上传的文件（这个功能是团队另一个人做的）
   - 查看 comments
   - 做出审批决策
3. **如果没有文件**（`file_ingestion_record_ids` 为空）:
   - 只需要查看 comments
   - 做出审批决策

**数据流**:
```
Human In The Loop Node 1 (request_for_update)
  output.data: {
    human_decision: "updated",
    decision_reason: "用户填写的说明信息",
    update_info: "用户填写的更新信息",
    file_ingestion_record_ids: [123, 456]  // 可能为空 []
  }
    ↓
Human In The Loop Node 2 (approval)
  input.data: {
    human_decision: "updated",
    decision_reason: "用户填写的说明信息",
    update_info: "用户填写的更新信息",
    file_ingestion_record_ids: [123, 456]  // 可能为空 []
  }
  → 判断是否有文件
    - 如果有 file_ingestion_record_ids: 展示文件预览（前端功能）
    - 如果没有: 只展示 comments
  → 审批人做出决策
  output.data: {
    human_decision: "approved",
    decision_reason: "文件符合要求"  // 或 "说明信息充分"
  }
```

---

## 二、文件内容获取方式详解

### 2.1 什么是"实时提取文本"？

**实时提取文本**指的是：
- 在链式调用时，**后端实时从文件提取文本内容**
- 如果文件还没有提取文本，需要调用文本提取服务（如 RAGFlow）
- 提取后的文本内容**立即**传递给下游节点

**示例**:
```python
# 方式1: 实时提取文本
file_ids = [123, 456]
for file_id in file_ids:
    # 1. 从 MinIO 获取文件
    file_content = minio_service.get_file(file_record.file_path)
    # 2. 实时提取文本（如果还没有提取）
    if not file_record.extracted_text_storage_path:
        text_content = await extract_text_from_file(file_content, file_record.file_name)  # 耗时操作
    else:
        text_content = minio_service.get_file(file_record.extracted_text_storage_path)
    # 3. 立即传递给下游节点
    downstream_data['file_contents'].append(text_content)
```

---

### 2.2 两种方式对比

#### 方式1: 后端实时提取文本并传递

**流程**:
```
后端获取 file_ingestion_record_ids
  ↓
后端从 MinIO 获取文件
  ↓
后端提取文本内容（如果还没有提取）
  ↓
后端将文本内容传递给下游节点
  ↓
下游节点接收完整的文本内容
```

**优点**:
- ✅ 下游节点可以直接使用文本内容（如 App Node 可以直接分析）
- ✅ 不需要前端额外请求
- ✅ 数据流清晰，所有数据在后端处理

**缺点**:
- ❌ **性能问题**: 如果文件很大，提取文本需要时间（可能几秒到几分钟）
- ❌ **阻塞问题**: 链式调用会被阻塞，等待文本提取完成
- ❌ **资源消耗**: 后端需要处理大量文件内容
- ❌ **用户体验差**: Workflow 执行变慢，用户需要等待

**适用场景**:
- 文件已经提取过文本（存储在 `extracted_text_storage_path`）
- 文件很小，提取文本很快
- 下游节点（如 App Node）需要文本内容进行分析

---

#### 方式2: 只传递文件ID，前端按需获取

**流程**:
```
后端获取 file_ingestion_record_ids
  ↓
后端只传递 file_ingestion_record_ids 给下游节点
  ↓
下游节点（approval）接收 file_ingestion_record_ids
  ↓
前端展示任务时，按需获取文件内容
  ↓
用户点击预览时，前端调用 API 获取文件内容
```

**优点**:
- ✅ **性能好**: 不阻塞 Workflow 执行
- ✅ **用户体验好**: Workflow 快速完成，文件按需加载
- ✅ **资源节省**: 后端不需要处理大量文件内容
- ✅ **灵活性高**: 用户可以选择是否查看文件

**缺点**:
- ❌ 下游节点（如 App Node）无法直接使用文件内容
- ❌ 需要前端额外请求（但这是按需的，不影响性能）

**适用场景**:
- Human In The Loop Node (approval): 用户按需查看文件
- 文件预览功能（团队另一个人做的功能）
- 不需要立即分析文件内容的场景

---

### 2.3 混合方案（推荐）

**根据下游节点类型选择不同的方式**:

1. **下游是 App Node**:
   - 需要文件内容进行分析
   - 使用方式1: 后端实时获取文件内容（从 `extracted_text_storage_path`）
   - 但**不重新提取**，只使用已提取的文本

2. **下游是 Human In The Loop Node (approval)**:
   - 只需要文件预览（用户查看）
   - 使用方式2: 只传递 `file_ingestion_record_ids`
   - 前端按需获取文件内容

---

## 三、详细流程图

### 3.1 request_for_update → approval 完整流程

```mermaid
graph TB
    Start([用户收到 request_for_update 任务]) --> Action{用户操作}
    
    Action -->|上传文件 + 填写 comments| Upload[上传文件<br/>填写 comments]
    Action -->|只填写 comments| Comments[只填写 comments<br/>不上传文件]
    
    Upload --> Submit1[提交]
    Comments --> Submit2[提交]
    
    Submit1 --> Store1[存储数据<br/>file_ingestion_record_ids: [123, 456]<br/>update_info: '说明信息']
    Submit2 --> Store2[存储数据<br/>file_ingestion_record_ids: []<br/>update_info: '说明信息']
    
    Store1 --> Downstream1[下游 approval 节点]
    Store2 --> Downstream2[下游 approval 节点]
    
    Downstream1 --> Check1{检查 file_ingestion_record_ids}
    Downstream2 --> Check2{检查 file_ingestion_record_ids}
    
    Check1 -->|有文件| ShowFiles1[展示文件预览<br/>+ comments]
    Check1 -->|无文件| ShowComments1[只展示 comments]
    
    Check2 -->|有文件| ShowFiles2[展示文件预览<br/>+ comments]
    Check2 -->|无文件| ShowComments2[只展示 comments]
    
    ShowFiles1 --> Decision1[审批人做出决策]
    ShowComments1 --> Decision1
    ShowFiles2 --> Decision2[审批人做出决策]
    ShowComments2 --> Decision2
    
    Decision1 --> End([完成])
    Decision2 --> End
    
    style Start fill:#e1f5ff
    style End fill:#c8e6c9
    style Upload fill:#fff3e0
    style Comments fill:#fff3e0
    style ShowFiles1 fill:#e8f5e9
    style ShowFiles2 fill:#e8f5e9
    style ShowComments1 fill:#f3e5f5
    style ShowComments2 fill:#f3e5f5
```

### 3.2 文件内容获取方式对比流程图

```mermaid
graph TB
    subgraph Way1[方式1: 后端实时提取文本]
        A1[后端获取 file_ingestion_record_ids] --> B1[从 MinIO 获取文件]
        B1 --> C1{文件是否已提取文本?}
        C1 -->|是| D1[从 MinIO 获取已提取的文本]
        C1 -->|否| E1[实时提取文本<br/>调用 RAGFlow/解析器<br/>⏱️ 耗时操作]
        D1 --> F1[传递给下游节点]
        E1 --> F1
        F1 --> G1[下游节点接收完整文本]
    end
    
    subgraph Way2[方式2: 只传递文件ID]
        A2[后端获取 file_ingestion_record_ids] --> B2[只传递 file_ingestion_record_ids]
        B2 --> C2[下游节点接收 file_ingestion_record_ids]
        C2 --> D2[前端展示任务]
        D2 --> E2{用户是否点击预览?}
        E2 -->|是| F2[前端调用 API<br/>GET /file-ingestion/records/{id}/extracted-text]
        E2 -->|否| G2[不加载文件内容]
        F2 --> H2[展示文件内容]
    end
    
    style E1 fill:#ffcdd2
    style F1 fill:#c8e6c9
    style G1 fill:#c8e6c9
    style B2 fill:#c8e6c9
    style C2 fill:#c8e6c9
    style F2 fill:#fff3e0
    style H2 fill:#e8f5e9
```

---

## 四、UI 预览图说明

### 4.1 request_for_update 任务界面

#### 情况1: 上传了文件

```
┌─────────────────────────────────────────┐
│  Request For Update 任务                 │
├─────────────────────────────────────────┤
│                                         │
│  任务标题: 供应商信息更新                │
│                                         │
│  更新说明:                              │
│  ┌───────────────────────────────────┐ │
│  │ 请上传最新的供应商资质文件...     │ │
│  └───────────────────────────────────┘ │
│                                         │
│  上传文件:                              │
│  ┌───────────────────────────────────┐ │
│  │ 📄 supplier_certificate.pdf        │ │
│  │ 📄 business_license.pdf            │ │
│  └───────────────────────────────────┘ │
│                                         │
│  补充说明（可选）:                      │
│  ┌───────────────────────────────────┐ │
│  │ 已上传最新文件，请审核...         │ │
│  └───────────────────────────────────┘ │
│                                         │
│  [提交]  [取消]                         │
└─────────────────────────────────────────┘
```

#### 情况2: 只填写了 comments

```
┌─────────────────────────────────────────┐
│  Request For Update 任务                 │
├─────────────────────────────────────────┤
│                                         │
│  任务标题: 供应商信息更新                │
│                                         │
│  更新说明:                              │
│  ┌───────────────────────────────────┐ │
│  │ 请上传最新的供应商资质文件...     │ │
│  └───────────────────────────────────┘ │
│                                         │
│  上传文件:                              │
│  ┌───────────────────────────────────┐ │
│  │ [点击上传文件]                     │ │
│  └───────────────────────────────────┘ │
│                                         │
│  补充说明（可选）:                      │
│  ┌───────────────────────────────────┐ │
│  │ 文件正在准备中，稍后上传...       │ │
│  └───────────────────────────────────┘ │
│                                         │
│  [提交]  [取消]                         │
└─────────────────────────────────────────┘
```

---

### 4.2 approval 任务界面

#### 情况1: 有文件需要预览

```
┌─────────────────────────────────────────┐
│  Approval 任务                           │
├─────────────────────────────────────────┤
│                                         │
│  任务标题: 审核供应商信息更新            │
│                                         │
│  上游更新信息:                          │
│  ┌───────────────────────────────────┐ │
│  │ 已上传最新文件，请审核...         │ │
│  └───────────────────────────────────┘ │
│                                         │
│  上传的文件:                            │
│  ┌───────────────────────────────────┐ │
│  │ 📄 supplier_certificate.pdf        │ │
│  │    [预览] [下载]                   │ │
│  │ 📄 business_license.pdf            │ │
│  │    [预览] [下载]                   │ │
│  └───────────────────────────────────┘ │
│                                         │
│  审批决策:                              │
│  ○ 批准  ● 拒绝                         │
│                                         │
│  审批意见:                              │
│  ┌───────────────────────────────────┐ │
│  │ 文件符合要求，批准通过...         │ │
│  └───────────────────────────────────┘ │
│                                         │
│  [提交]  [取消]                         │
└─────────────────────────────────────────┘
```

#### 情况2: 没有文件，只有 comments

```
┌─────────────────────────────────────────┐
│  Approval 任务                           │
├─────────────────────────────────────────┤
│                                         │
│  任务标题: 审核供应商信息更新            │
│                                         │
│  上游更新信息:                          │
│  ┌───────────────────────────────────┐ │
│  │ 文件正在准备中，稍后上传...       │ │
│  └───────────────────────────────────┘ │
│                                         │
│  上传的文件:                            │
│  ┌───────────────────────────────────┐ │
│  │ （无文件上传）                     │ │
│  └───────────────────────────────────┘ │
│                                         │
│  审批决策:                              │
│  ○ 批准  ● 拒绝                         │
│                                         │
│  审批意见:                              │
│  ┌───────────────────────────────────┐ │
│  │ 说明信息充分，批准通过...         │ │
│  └───────────────────────────────────┘ │
│                                         │
│  [提交]  [取消]                         │
└─────────────────────────────────────────┘
```

---

## 五、文件内容获取方式详细说明

### 5.1 当前实现方式

**从代码分析**:
- 文件上传时，文本内容已经提取并存储在 MinIO 中
- 存储路径: `extracted_text_storage_path`
- API 端点: `GET /file-ingestion/records/{record_id}/extracted-text`
- 前端已经有获取文件内容的逻辑（`NodeOutputPanel.tsx`, `TaskDetailModal.tsx`）

**结论**: 
- **不需要"实时提取文本"**
- 文件上传时已经提取了文本
- 只需要从 MinIO 获取已提取的文本即可

---

### 5.2 两种方式详细对比

#### 方式1: 后端获取文件内容并传递给下游节点

**完整流程**:
```
后端获取 file_ingestion_record_ids: [123, 456]
  ↓
后端查询 FileIngestionRecord 表
  ↓
后端从 MinIO 获取已提取的文本（extracted_text_storage_path）
  ↓
后端将文本内容添加到 input.data 或 context
  ↓
下游节点接收完整的文本内容
```

**性能分析**:
- ⏱️ **时间**: 每个文件需要从 MinIO 读取（约 100-500ms）
- 💾 **内存**: 需要加载所有文件内容到内存
- 🔄 **阻塞**: Workflow 执行会被阻塞，等待文件内容加载

**适用场景**:
- ✅ 下游是 App Node（需要文本内容进行分析）
- ✅ 文件数量少（1-3个文件）
- ✅ 文件已经提取过文本（不需要重新提取）

---

#### 方式2: 只传递文件ID，前端按需获取

**完整流程**:
```
后端获取 file_ingestion_record_ids: [123, 456]
  ↓
后端只传递 file_ingestion_record_ids 给下游节点
  ↓
下游节点接收 file_ingestion_record_ids
  ↓
前端展示任务时，显示文件列表（但不加载内容）
  ↓
用户点击"预览"按钮
  ↓
前端调用 API: GET /file-ingestion/records/{id}/extracted-text
  ↓
后端从 MinIO 获取已提取的文本
  ↓
前端展示文件内容
```

**性能分析**:
- ⚡ **时间**: Workflow 执行不阻塞，文件按需加载
- 💾 **内存**: 只加载用户查看的文件
- 🔄 **非阻塞**: Workflow 快速完成，用户体验好

**适用场景**:
- ✅ 下游是 Human In The Loop Node (approval)（用户按需查看）
- ✅ 文件数量多（多个文件）
- ✅ 用户可能不需要查看所有文件

---

### 5.3 混合方案（推荐）

**根据下游节点类型选择**:

```python
# 判断下游节点类型
downstream_node_type = get_downstream_node_type()

if downstream_node_type == 'app':
    # App Node 需要文件内容进行分析
    # 方式1: 后端获取文件内容
    file_contents = get_file_contents_from_ids(file_ids, session)
    # 转换为 file_content 格式
    file_content = convert_to_file_content_format(file_contents)
    # 传递给 App Node
    app_config['file_content'] = file_content
elif downstream_node_type == 'humanInTheLoop':
    # Human In The Loop Node 只需要文件ID
    # 方式2: 只传递 file_ingestion_record_ids
    input_data['file_ingestion_record_ids'] = file_ids
    # 前端按需获取文件内容
else:
    # 其他节点类型，根据需求选择
    pass
```

---

## 六、设计调整建议

### 6.1 request_for_update → approval 场景

**调整**:
1. **支持两种情况**:
   - 有文件: `file_ingestion_record_ids` 不为空
   - 无文件: `file_ingestion_record_ids` 为空，只有 `update_info`

2. **数据传递**:
   - 只传递 `file_ingestion_record_ids`（不传递文件内容）
   - 前端按需获取文件内容（已有实现）

3. **前端展示**:
   - 如果有 `file_ingestion_record_ids`: 显示文件列表，支持预览（团队另一个人做的功能）
   - 如果没有 `file_ingestion_record_ids`: 只显示 `update_info`（comments）

---

### 6.2 文件内容获取方式

**推荐方案**: **混合方案**

1. **Human In The Loop Node (approval)**:
   - 只传递 `file_ingestion_record_ids`
   - 前端按需获取文件内容
   - 不阻塞 Workflow 执行

2. **App Node**:
   - 如果上游有文件，获取文件内容
   - 转换为 `file_content` 格式
   - 与文件内容合并，发送给 Airflow DAG

---

## 七、实现建议

### 7.1 修改 ChainCallDataService

```python
@staticmethod
def _convert_for_hitl_node(
    upstream_data: Dict[str, Any],
    node_config: Optional[Dict[str, Any]] = None,
    session=None
) -> Dict[str, Any]:
    """
    转换为 Human In The Loop Node 需要的格式
    """
    task_type = node_config.get('taskType') if node_config else None
    
    # 提取基础数据
    context = {
        'human_decision': upstream_data.get('human_decision'),
        'decision_reason': upstream_data.get('decision_reason'),
        'update_info': upstream_data.get('update_info'),
    }
    
    # 提取 file_ingestion_record_ids（如果有）
    file_ids = upstream_data.get('file_ingestion_record_ids', [])
    if file_ids:
        context['file_ingestion_record_ids'] = file_ids
        # 注意: 不在这里获取文件内容，只传递文件ID
        # 文件内容由前端按需获取
    
    if task_type == 'approval':
        # approval 节点需要上游的更新信息
        # 如果有文件，前端会显示文件列表（支持预览）
        # 如果没有文件，只显示 update_info
        return context
    else:
        return context
```

---

### 7.2 修改 App Node 链式调用

```python
# 在 App Node 执行时
if previous_nodes:
    upstream_data = ChainCallDataService.extract_upstream_data(state, node['id'], 'app')
    
    # 检查是否有 file_ingestion_record_ids（来自 request_for_update）
    file_ids = upstream_data.get('file_ingestion_record_ids', [])
    if file_ids:
        # 获取文件内容（从已提取的文本）
        file_contents = ChainCallDataService._get_file_contents_from_ids(file_ids, session)
        # 转换为 file_content 格式
        workflow_files = [
            {
                "file_name": f["file_name"],
                "file_content": f["file_content"] or f"[File ID: {f['file_id']}]"
            }
            for f in file_contents
        ]
        
        # 与现有文件内容合并
        if app_config and app_config.get('file_content'):
            existing_files = json.loads(app_config['file_content'])
            combined_files = workflow_files + existing_files
            app_config['file_content'] = json.dumps(combined_files, ensure_ascii=False)
        else:
            if not app_config:
                app_config = {}
            app_config['file_content'] = json.dumps(workflow_files, ensure_ascii=False)
```

---

## 八、总结

### 8.1 request_for_update → approval 场景

**支持两种情况**:
1. ✅ 有文件: 传递 `file_ingestion_record_ids`，前端支持预览
2. ✅ 无文件: 只传递 `update_info`，前端只显示说明信息

**数据传递**:
- 只传递 `file_ingestion_record_ids`（不传递文件内容）
- 前端按需获取文件内容（已有实现）

---

### 8.2 文件内容获取方式

**推荐**: **混合方案**

1. **Human In The Loop Node (approval)**:
   - 方式2: 只传递文件ID，前端按需获取
   - 优点: 不阻塞，用户体验好

2. **App Node**:
   - 方式1: 后端获取文件内容（从已提取的文本）
   - 优点: 可以直接分析文件内容

**关键点**:
- ✅ 文件上传时已经提取了文本，不需要"实时提取"
- ✅ 只需要从 MinIO 获取已提取的文本（`extracted_text_storage_path`）
- ✅ 性能好，不阻塞 Workflow 执行
