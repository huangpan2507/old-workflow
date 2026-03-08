# Workflow 链式调用设计方案

## 概述

本文档设计支持**任意节点类型**的链式调用方案，包括：
- App Node → App Node
- App Node → Question Classifier Node → Human In The Loop Node → App Node
- Human In The Loop Node (request_for_update) → Human In The Loop Node (approval)
- 以及其他任意组合

---

## 一、链式调用场景分析

### 1.1 场景1: App Node → App Node

**业务场景**: 
- App Node 1 分析文件，生成分析报告
- App Node 2 基于 App Node 1 的报告进行进一步分析

**数据流**:
```
App Node 1
  output.data.conclusion_detail: "分析报告（Markdown）"
    ↓
App Node 2
  input.data: {conclusion_detail: "分析报告（Markdown）"}
  → 转换为 file_content 格式
  → 与文件内容（如果有）合并
  → 发送给大模型分析
```

---

### 1.2 场景2: App Node → Question Classifier Node → Human In The Loop Node → App Node

**业务场景**:
- App Node 1 分析文件，生成分析报告
- Question Classifier Node 对报告进行分类（如：approval / update）
- Human In The Loop Node 进行人工审批或更新
- App Node 2 基于审批结果进行进一步分析

**数据流**:
```
App Node 1
  output.data.conclusion_detail: "分析报告（Markdown）"
    ↓
Question Classifier Node
  input.data: {conclusion_detail: "分析报告（Markdown）"}
  output.data: {branch: "class_1", class_id: "class_1", class_name: "supplier admission approval"}
    ↓
Human In The Loop Node (approval)
  input.data: {branch: "class_1", class_id: "class_1", class_name: "supplier admission approval"}
  output.data: {human_decision: "approved", decision_reason: "符合要求"}
    ↓
App Node 2
  input.data: {human_decision: "approved", decision_reason: "符合要求"}
  → 转换为 file_content 格式
  → 与文件内容（如果有）合并
  → 发送给大模型分析
```

---

### 1.3 场景3: Human In The Loop Node (request_for_update) → Human In The Loop Node (approval)

**业务场景**:
- Human In The Loop Node 1 (request_for_update): 让人上传文件或填写信息
- Human In The Loop Node 2 (approval): 另一个人查看并审核之前上传的文件是否符合要求

**重要**: request_for_update 节点**不一定需要上传文件**，有两种情况：
1. **情况1**: 上传了文件 + 填写了 comments
2. **情况2**: 只填写了 comments，没有上传文件

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
    - 如果有 file_ingestion_record_ids: 前端显示文件列表（支持预览）
    - 如果没有 file_ingestion_record_ids: 只显示 update_info
  → 审批人做出决策
  output.data: {
    human_decision: "approved",
    decision_reason: "文件符合要求"  // 或 "说明信息充分"
  }
```

**文件内容获取方式**:
- **只传递文件ID**（不传递文件内容）
- 前端按需获取文件内容（调用 `GET /file-ingestion/records/{id}/extracted-text`）
- 文件预览功能由团队另一个人实现，我们只需要支持传递 `file_ingestion_record_ids`

**详细说明**: 请参考 `workflow-hitl-chain-call-clarification.md`

---

## 二、通用链式调用设计原则

### 2.1 核心原则

1. **统一的数据提取接口**
   - 所有节点都从上游节点的 `output.data` 提取数据
   - 使用统一的提取函数：`get_context_from_upstream_output()`

2. **数据格式转换**
   - 不同节点类型需要不同的数据格式
   - 提供统一的转换函数，将上游数据转换为当前节点需要的格式

3. **数据合并策略**
   - 如果当前节点需要文件内容，将上游数据与文件内容合并
   - 如果当前节点不需要文件内容，直接使用上游数据

4. **向后兼容**
   - 保持现有功能不变
   - 新增链式调用功能作为增强

---

## 三、节点类型与数据需求

### 3.1 App Node

**需要的数据**:
- 文件内容（file_content 格式）
- 或上游节点的文本输出（conclusion_detail）

**数据转换**:
```python
# 从上游节点提取数据
upstream_data = get_upstream_output_data()

# 提取 conclusion_detail（如果有）
conclusion_detail = upstream_data.get('conclusion_detail', '')

# 转换为 file_content 格式
if conclusion_detail:
    workflow_file = {
        "file_name": "workflow_upstream_output.md",
        "file_content": f"[WORKFLOW UPSTREAM OUTPUT]\n\n{conclusion_detail}"
    }
    
    # 与文件内容合并（如果有）
    if file_content:
        files = json.loads(file_content)
        files.insert(0, workflow_file)
        file_content = json.dumps(files, ensure_ascii=False)
    else:
        file_content = json.dumps([workflow_file], ensure_ascii=False)
```

---

### 3.2 Question Classifier Node

**需要的数据**:
- 文本内容（conclusion_detail 或 merged_context）

**数据转换**:
```python
# 从上游节点提取数据
upstream_data = get_upstream_output_data()

# 提取 conclusion_detail
conclusion_detail = upstream_data.get('conclusion_detail', '')

# 如果没有 conclusion_detail，尝试从 merged_context 提取
if not conclusion_detail:
    merged_context = upstream_data.get('merged_context', {})
    if isinstance(merged_context, dict):
        # 转换为文本
        conclusion_detail = convert_dict_to_text(merged_context)
```

---

### 3.3 Human In The Loop Node

**需要的数据**:
- 根据 task_type 不同，需要不同的数据：
  - **approval**: 需要上下文信息（用于审批决策）
  - **request_for_update**: 需要上下文信息（用于更新任务）
  - **inform**: 需要上下文信息（用于通知）
  - **assign**: 需要上下文信息（用于分配任务）

**数据转换**:
```python
# 从上游节点提取数据
upstream_data = get_upstream_output_data()

# 根据 task_type 提取不同的数据
if task_type == 'approval':
    # 提取审批所需的上下文
    context = extract_approval_context(upstream_data)
elif task_type == 'request_for_update':
    # 提取更新任务所需的上下文
    context = extract_update_context(upstream_data)
    # 如果有 file_ingestion_record_ids，也需要提取
    file_ids = upstream_data.get('file_ingestion_record_ids', [])
elif task_type == 'assign':
    # 提取分配任务所需的上下文
    context = extract_assign_context(upstream_data)
```

---

## 四、统一的数据提取和转换服务

### 4.1 设计思路

创建一个统一的服务，用于：
1. 从上游节点提取数据
2. 根据当前节点类型转换数据格式
3. 合并数据（如果需要）

### 4.2 服务接口设计

```python
class ChainCallDataService:
    """链式调用数据服务"""
    
    @staticmethod
    def extract_upstream_data(
        state: WorkflowState,
        current_node_id: str,
        current_node_type: str
    ) -> Dict[str, Any]:
        """
        从上游节点提取数据
        
        Args:
            state: Workflow 状态
            current_node_id: 当前节点ID
            current_node_type: 当前节点类型
        
        Returns:
            提取的上游数据
        """
        # 1. 找到上游节点
        upstream_node_id = find_upstream_node(state, current_node_id)
        if not upstream_node_id:
            return {}
        
        # 2. 获取上游节点输出
        upstream_result = state.get('node_results', {}).get(upstream_node_id, {})
        upstream_output = upstream_result.get('output', {})
        
        # 3. 提取数据（使用统一的提取函数）
        from app.services.workflow.node_utils import get_context_from_upstream_output
        upstream_node_type = get_node_type_from_result(upstream_result)
        
        context = get_context_from_upstream_output(
            upstream_node_type,
            upstream_output,
            current_node_id
        )
        
        return context
    
    @staticmethod
    def convert_for_node_type(
        upstream_data: Dict[str, Any],
        target_node_type: str,
        node_config: Optional[Dict[str, Any]] = None
    ) -> Dict[str, Any]:
        """
        将上游数据转换为目标节点类型需要的格式
        
        Args:
            upstream_data: 上游数据
            target_node_type: 目标节点类型
            node_config: 节点配置（可选）
        
        Returns:
            转换后的数据
        """
        if target_node_type == 'app':
            return ChainCallDataService._convert_for_app_node(upstream_data, node_config)
        elif target_node_type == 'questionClassifier':
            return ChainCallDataService._convert_for_classifier_node(upstream_data)
        elif target_node_type == 'humanInTheLoop':
            return ChainCallDataService._convert_for_hitl_node(upstream_data, node_config)
        else:
            return upstream_data
    
    @staticmethod
    def _convert_for_app_node(
        upstream_data: Dict[str, Any],
        node_config: Optional[Dict[str, Any]] = None
    ) -> Dict[str, Any]:
        """
        转换为 App Node 需要的格式（file_content）
        """
        import json
        
        # 提取 conclusion_detail
        conclusion_detail = upstream_data.get('conclusion_detail', '')
        
        # 如果没有 conclusion_detail，尝试从其他字段提取
        if not conclusion_detail:
            # 尝试从 merged_context 提取
            merged_context = upstream_data.get('merged_context', {})
            if isinstance(merged_context, dict):
                conclusion_detail = convert_dict_to_markdown(merged_context)
            else:
                # 尝试将整个 upstream_data 转换为文本
                conclusion_detail = json.dumps(upstream_data, ensure_ascii=False, indent=2)
        
        # 转换为 file_content 格式
        if conclusion_detail:
            workflow_file = {
                "file_name": "workflow_upstream_output.md",
                "file_content": f"[WORKFLOW UPSTREAM OUTPUT]\n\n{conclusion_detail}"
            }
            
            # 如果有文件内容，合并
            file_content = node_config.get('file_content') if node_config else None
            if file_content:
                try:
                    files = json.loads(file_content) if isinstance(file_content, str) else file_content
                    if isinstance(files, list):
                        files.insert(0, workflow_file)
                        return {"file_content": json.dumps(files, ensure_ascii=False)}
                except:
                    pass
            
            return {"file_content": json.dumps([workflow_file], ensure_ascii=False)}
        
        return {}
    
    @staticmethod
    def _convert_for_classifier_node(upstream_data: Dict[str, Any]) -> Dict[str, Any]:
        """
        转换为 Question Classifier Node 需要的格式
        """
        conclusion_detail = upstream_data.get('conclusion_detail', '')
        if conclusion_detail:
            return {"conclusion_detail": conclusion_detail}
        return {}
    
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
        
        if task_type == 'request_for_update':
            # request_for_update 需要提取文件ID（如果有）
            file_ids = upstream_data.get('file_ingestion_record_ids', [])
            context = upstream_data.copy()
            if file_ids:
                context['file_ingestion_record_ids'] = file_ids
            return context
        elif task_type == 'approval':
            # approval 需要提取审批上下文
            context = extract_approval_context(upstream_data)
            
            # 如果上游节点是 request_for_update，传递 file_ingestion_record_ids
            # 注意: 不在这里获取文件内容，只传递文件ID
            # 文件内容由前端按需获取（调用 API: GET /file-ingestion/records/{id}/extracted-text）
            file_ids = upstream_data.get('file_ingestion_record_ids', [])
            if file_ids:
                context['file_ingestion_record_ids'] = file_ids
                # 不获取文件内容，前端按需获取（性能更好，不阻塞 Workflow）
            
            return context
        else:
            # 其他类型，直接返回上游数据
            return upstream_data
    
    @staticmethod
    def _get_file_contents_from_ids(
        file_ids: List[int],
        session,
        extract_text: bool = True
    ) -> List[Dict[str, Any]]:
        """
        从 file_ingestion_record_ids 获取文件内容
        
        Args:
            file_ids: 文件记录ID列表
            session: 数据库会话
            extract_text: 是否提取文本内容（默认 True，用于 App Node 分析）
        
        Returns:
            文件内容列表
        """
        from app.models import FileIngestionRecord
        from app.services.minio_service import minio_service
        
        file_contents = []
        for file_id in file_ids:
            try:
                # 查询文件记录
                file_record = session.get(FileIngestionRecord, file_id)
                if not file_record:
                    continue
                
                file_info = {
                    "file_id": file_id,
                    "file_name": file_record.file_name,
                    "file_path": file_record.file_path,
                    "file_hash": file_record.file_hash
                }
                
                # 如果需要提取文本内容（用于 App Node 分析）
                if extract_text:
                    # 从已提取的文本获取（不需要重新提取）
                    if file_record.extracted_text_storage_path:
                        try:
                            # 从 MinIO 获取已提取的文本
                            text_content = minio_service.get_file(
                                file_record.extracted_text_storage_path
                            )
                            file_info["file_content"] = text_content.decode('utf-8') if isinstance(text_content, bytes) else text_content
                        except Exception as e:
                            logger.warning(f"Failed to get extracted text for file_id={file_id}: {e}")
                            file_info["file_content"] = None
                    else:
                        # 如果没有提取的文本，返回 None（文件可能还在处理中）
                        file_info["file_content"] = None
                
                file_contents.append(file_info)
            except Exception as e:
                logger.warning(f"Failed to get file content for file_id={file_id}: {e}")
                continue
        
        return file_contents
```

---

## 五、实现方案

### 5.1 App Node 链式调用实现

**位置**: `backend/app/services/workflow/workflow_execution_service.py:_create_app_node()`

**修改**:
```python
# 在触发 DAG 前，检查是否有上游数据
if previous_nodes:
    source_node_id = previous_nodes[0]
    source_result = state.get('node_results', {}).get(source_node_id, {})
    source_output = source_result.get('output', {})
    
    # 提取上游数据
    from app.services.workflow.chain_call_data_service import ChainCallDataService
    upstream_data = ChainCallDataService.extract_upstream_data(state, node['id'], 'app')
    
    # 转换为 App Node 需要的格式
    converted_data = ChainCallDataService.convert_for_node_type(
        upstream_data,
        'app',
        app_config
    )
    
    # 如果有转换后的 file_content，合并到 app_config
    if converted_data.get('file_content'):
        # 如果 app_config 中已有 file_content，合并
        if app_config and app_config.get('file_content'):
            existing_files = json.loads(app_config['file_content'])
            new_files = json.loads(converted_data['file_content'])
            combined_files = new_files + existing_files
            app_config['file_content'] = json.dumps(combined_files, ensure_ascii=False)
        else:
            # 如果没有，直接使用转换后的 file_content
            if not app_config:
                app_config = {}
            app_config['file_content'] = converted_data['file_content']
        
        # 更新 input_data（用于追溯）
        input_data = converted_data.copy()
```

---

### 5.2 Question Classifier Node 链式调用实现

**位置**: `backend/app/services/workflow/nodes/question_classifier_node.py:execute()`

**修改**:
```python
# 在提取输入文本时，支持从上游节点的 conclusion_detail 提取
upstream_info = self._get_upstream_info(state)
upstream_output = upstream_info.get('output', {})

# 使用统一的数据提取服务
from app.services.workflow.chain_call_data_service import ChainCallDataService
upstream_data = ChainCallDataService.extract_upstream_data(state, self.node_id, 'questionClassifier')

# 转换为 Question Classifier Node 需要的格式
converted_data = ChainCallDataService.convert_for_node_type(upstream_data, 'questionClassifier')

# 提取 conclusion_detail
conclusion_detail = converted_data.get('conclusion_detail', '')
if conclusion_detail:
    input_text = conclusion_detail
    input_data = {'conclusion_detail': conclusion_detail}
else:
    # Fallback: 使用原有逻辑
    input_text = self._extract_input_text(upstream_output)
    input_data = {}
```

---

### 5.3 Human In The Loop Node 链式调用实现

**位置**: `backend/app/services/workflow/nodes/human_in_the_loop_node.py:execute()`

**修改**:
```python
# 在提取上游上下文时，使用统一的数据提取服务
from app.services.workflow.chain_call_data_service import ChainCallDataService
upstream_data = ChainCallDataService.extract_upstream_data(state, self.node_id, 'humanInTheLoop')

# 转换为 Human In The Loop Node 需要的格式
converted_data = ChainCallDataService.convert_for_node_type(
    upstream_data,
    'humanInTheLoop',
    {'taskType': self.task_type}
)

# 根据 task_type 处理不同的数据
if self.task_type == 'request_for_update':
    # 提取文件ID（如果有）
    file_ids = converted_data.get('file_ingestion_record_ids', [])
    if file_ids:
        # 保存到 input_data 中，供后续使用
        input_data['file_ingestion_record_ids'] = file_ids
    
    # 提取上下文用于更新任务
    context = converted_data
elif self.task_type == 'approval':
    # 提取审批上下文
    context = extract_approval_context(converted_data)
else:
    # 其他类型，直接使用转换后的数据
    context = converted_data

# 更新 input_data（用于追溯）
input_data = context.copy() if isinstance(context, dict) else {}
```

---

## 六、数据流示例

### 6.1 场景1: App Node → App Node

```
App Node 1 执行:
  - 分析文件，生成 conclusion_detail
  - 存储到 output.data.conclusion_detail

App Node 2 执行:
  - 提取 App Node 1 的 conclusion_detail
  - 转换为 file_content 格式:
    [
      {
        "file_name": "workflow_upstream_output.md",
        "file_content": "[WORKFLOW UPSTREAM OUTPUT]\n\n{conclusion_detail}"
      }
    ]
  - 如果有文件，合并到文件列表
  - 发送给 Airflow DAG 分析
```

### 6.2 场景2: App Node → Question Classifier Node → Human In The Loop Node → App Node

```
App Node 1:
  output.data.conclusion_detail: "分析报告"

Question Classifier Node:
  input.data: {conclusion_detail: "分析报告"}
  output.data: {branch: "class_1", class_id: "class_1", class_name: "supplier admission approval"}

Human In The Loop Node (approval):
  input.data: {branch: "class_1", class_id: "class_1", class_name: "supplier admission approval"}
  output.data: {human_decision: "approved", decision_reason: "符合要求"}

App Node 2:
  input.data: {human_decision: "approved", decision_reason: "符合要求"}
  → 转换为 file_content 格式:
    [
      {
        "file_name": "workflow_upstream_output.md",
        "file_content": "[WORKFLOW UPSTREAM OUTPUT]\n\nhuman_decision: approved\ndecision_reason: 符合要求"
      }
    ]
  → 发送给 Airflow DAG 分析
```

### 6.3 场景3: Human In The Loop Node (request_for_update) → Human In The Loop Node (approval)

**情况1: 有文件**
```
Human In The Loop Node 1 (request_for_update)
  output.data: {
    human_decision: "updated",
    decision_reason: "已上传文件",
    update_info: "用户填写的更新信息",
    file_ingestion_record_ids: [123, 456]
  }

Human In The Loop Node 2 (approval)
  input.data: {
    human_decision: "updated",
    decision_reason: "已上传文件",
    update_info: "用户填写的更新信息",
    file_ingestion_record_ids: [123, 456]
  }
  → 前端显示文件列表（支持预览）
  → 前端显示 update_info
  → 审批人查看文件（按需加载）
  → 审批人做出决策
  output.data: {
    human_decision: "approved",
    decision_reason: "文件符合要求"
  }
```

**情况2: 无文件，只有 comments**
```
Human In The Loop Node 1 (request_for_update)
  output.data: {
    human_decision: "updated",
    decision_reason: "用户填写的说明信息",
    update_info: "用户填写的更新信息",
    file_ingestion_record_ids: []  // 空列表
  }

Human In The Loop Node 2 (approval)
  input.data: {
    human_decision: "updated",
    decision_reason: "用户填写的说明信息",
    update_info: "用户填写的更新信息",
    file_ingestion_record_ids: []  // 空列表
  }
  → 前端只显示 update_info（无文件列表）
  → 审批人查看说明信息
  → 审批人做出决策
  output.data: {
    human_decision: "approved",
    decision_reason: "说明信息充分"
  }
```

---

## 七、实现步骤

### 7.1 第一阶段：创建统一的数据服务

1. 创建 `ChainCallDataService` 类
2. 实现 `extract_upstream_data()` 方法
3. 实现 `convert_for_node_type()` 方法
4. 实现各节点类型的转换方法

### 7.2 第二阶段：修改各节点实现

1. **App Node**: 
   - 在触发 DAG 前，检查上游数据
   - 转换为 file_content 格式并合并

2. **Question Classifier Node**:
   - 使用统一服务提取上游数据
   - 提取 conclusion_detail

3. **Human In The Loop Node**:
   - 根据 task_type 提取不同的数据
   - 处理 file_ingestion_record_ids（request_for_update → approval）

### 7.3 第三阶段：测试和验证

1. 测试 App Node → App Node
2. 测试 App Node → Question Classifier Node → Human In The Loop Node → App Node
3. 测试 Human In The Loop Node (request_for_update) → Human In The Loop Node (approval)
4. 验证向后兼容性

---

## 八、关键设计决策

### 8.1 数据格式转换

**决策**: 使用统一的服务进行数据格式转换

**理由**:
- 不同节点类型需要不同的数据格式
- 统一服务便于维护和扩展
- 支持向后兼容

### 8.2 数据合并策略

**决策**: 上游数据优先，文件内容次之

**理由**:
- 上游数据是 workflow 的核心数据流
- 文件内容可以作为补充
- 合并时，上游数据放在文件列表前面

### 8.3 向后兼容

**决策**: 保持现有功能不变，新增链式调用功能

**理由**:
- 不影响现有 workflow
- 逐步迁移到新功能
- 降低风险

---

## 九、关键设计决策

### 9.1 App Node 的 app_node_id 来源

**已确认**:
- `app_node_id` 是从 `node.data.appNodeId` 获取的
- 这个值是在**创建节点时就已经确定的**（从 `applicationData?.id` 获取）
- **不需要用户在 UI 中选择**，光盘（disk）是写死的、固定的

### 9.2 Human In The Loop Node 的文件处理

**方案**:
- `request_for_update` 节点上传的文件，通过 `file_ingestion_record_ids` 传递给下游节点
- `approval` 节点从 `file_ingestion_record_ids` 获取文件内容
- 使用 `ChainCallDataService._get_file_contents_from_ids()` 方法获取文件内容
- 文件内容添加到上下文中，供审批人查看

### 9.3 数据格式统一

**决策**:
- 所有节点都使用统一的 `input.data` 格式（用于追溯）
- 但每个节点类型可以根据需要提取不同的字段
- 使用统一的数据提取和转换服务，确保数据格式一致

---

## 十、实现优先级

### 10.1 第一阶段：基础链式调用支持

1. ✅ 创建 `ChainCallDataService` 统一服务
2. ✅ 实现 App Node 链式调用（App Node → App Node）
3. ✅ 实现 Question Classifier Node 数据提取修复

### 10.2 第二阶段：复杂链式调用支持

1. ⏳ 实现 Human In The Loop Node 链式调用
   - request_for_update → approval 的文件处理
   - 从 file_ingestion_record_ids 获取文件内容
2. ⏳ 实现混合链式调用（App Node → Question Classifier Node → Human In The Loop Node → App Node）

### 10.3 第三阶段：优化和测试

1. ⏳ 性能优化
2. ⏳ 完整测试覆盖
3. ⏳ 文档更新

---

## 十一、文件内容获取方式说明

### 11.1 两种方式对比

**方式1: 后端实时提取文本并传递**
- ❌ **不推荐**: 会阻塞 Workflow 执行，性能差
- ❌ 如果文件很大，提取文本需要时间
- ❌ 用户体验差

**方式2: 只传递文件ID，前端按需获取**（推荐）
- ✅ **推荐**: 不阻塞 Workflow 执行，性能好
- ✅ 文件上传时已经提取了文本（存储在 `extracted_text_storage_path`）
- ✅ 前端按需获取，用户体验好

**详细说明**: 请参考 `workflow-hitl-chain-call-clarification.md`

---

## 十二、待确认问题

1. **数据大小限制**:
   - 如果上游节点的 conclusion_detail 很大（如 10MB），如何传递给下游节点？
   - 是否需要限制数据大小或使用流式传输？

2. **错误处理**:
   - 如果上游节点执行失败，下游节点如何处理？
   - 是否需要支持部分数据传递？

3. **文件预览功能**:
   - 文件预览功能由团队另一个人实现
   - 我们只需要支持传递 `file_ingestion_record_ids` 即可
