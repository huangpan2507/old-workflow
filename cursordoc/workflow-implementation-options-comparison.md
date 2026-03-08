# Workflow多实例实现方案对比

## 一、实例存储方案对比

### 方案A：使用 WorkflowInstance（推荐）

**数据模型特点**：
- 专门为模板实例设计
- 已有 `template_id` 外键关联
- 字段简洁，专注实例追踪
- 设计初衷就是"模板的实例"

**字段对比**：
```
WorkflowInstance:
├─ template_id (外键) ✅ 天然关联模板
├─ status (执行状态)
├─ current_node_id
├─ execution_result (JSONB)
├─ celery_task_id (可选)
└─ created_by, create_time

WorkflowExecutionTask:
├─ task_id (UUID)
├─ nodes (完整定义) ⚠️ 冗余存储
├─ edges (完整定义) ⚠️ 冗余存储
├─ input_data
├─ app_node_configs
├─ status
├─ node_results (实时更新)
├─ thread_id, checkpoint_id
└─ created_by, create_time
```

**数据流对比**：

#### 方案A流程（WorkflowInstance）：
```
用户操作: 从模板创建实例
    ↓
1. 选择模板 (template_id=1)
2. 配置覆盖 (app_node_configs)
    ↓
3. 创建 WorkflowInstance
   - template_id = 1
   - status = "pending"
   - execution_result = null
    ↓
4. 创建 WorkflowExecutionTask (执行引擎需要)
   - 关联 instance_id (新增字段)
   - nodes/edges 从模板加载
   - app_node_configs 合并
    ↓
5. 执行完成后
   - WorkflowInstance.execution_result 更新
   - WorkflowInstance.status = "completed"
```

**UI展示效果**：
```
┌─────────────────────────────────────────────────────────┐
│  Workflow模板: 简历审核流程                              │
│  ┌───────────────────────────────────────────────────┐  │
│  │ 实例1: 张三简历                    🟢 运行中        │  │
│  │ 创建时间: 10:30:00  执行时间: 5分钟                │  │
│  │ [查看详情] [停止]                                  │  │
│  ├───────────────────────────────────────────────────┤  │
│  │ 实例2: 李四简历                    ✅ 已完成        │  │
│  │ 创建时间: 10:25:00  执行时间: 8分钟                │  │
│  │ [查看详情] [重新执行]                              │  │
│  ├───────────────────────────────────────────────────┤  │
│  │ 实例3: 王五简历                    ⏸️ 等待审批      │  │
│  │ 创建时间: 10:20:00  执行时间: 3分钟                │  │
│  │ [查看详情] [继续执行]                              │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

**优点**：
- ✅ 语义清晰：Instance就是"模板的实例"
- ✅ 数据不冗余：nodes/edges在模板中，实例只存结果
- ✅ 查询简单：`WHERE template_id = ?` 即可
- ✅ 扩展性好：未来可以添加实例级别的配置

**缺点**：
- ⚠️ 需要同时维护两个表（Instance + ExecutionTask）
- ⚠️ 需要新增关联字段（ExecutionTask.instance_id）

---

#### 方案B流程（WorkflowExecutionTask扩展）：
```
用户操作: 从模板创建实例
    ↓
1. 选择模板 (template_id=1)
2. 配置覆盖 (app_node_configs)
    ↓
3. 创建 WorkflowExecutionTask
   - template_id = 1 (新增字段)
   - nodes/edges 从模板加载并完整存储 ⚠️
   - app_node_configs 合并
   - status = "pending"
    ↓
4. 执行完成后
   - node_results 更新
   - status = "completed"
```

**UI展示效果**：
```
┌─────────────────────────────────────────────────────────┐
│  Workflow模板: 简历审核流程                              │
│  ┌───────────────────────────────────────────────────┐  │
│  │ 执行记录1: task_abc123              🟢 运行中      │  │
│  │ 创建时间: 10:30:00  执行时间: 5分钟                │  │
│  │ [查看详情] [停止]                                  │  │
│  ├───────────────────────────────────────────────────┤  │
│  │ 执行记录2: task_def456              ✅ 已完成      │  │
│  │ 创建时间: 10:25:00  执行时间: 8分钟                │  │
│  │ [查看详情] [重新执行]                              │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

**优点**：
- ✅ 改动最小：只需添加 `template_id` 字段
- ✅ 单一数据源：所有执行记录在一个表
- ✅ 兼容现有代码：执行逻辑无需大改

**缺点**：
- ⚠️ 数据冗余：每个实例都完整存储nodes/edges
- ⚠️ 语义模糊：Task既用于临时执行，又用于模板实例
- ⚠️ 查询复杂：需要区分"临时执行"和"模板实例"
- ⚠️ 模板修改影响：如果模板修改，已存储的nodes/edges会过时

---

### 推荐方案：方案A（WorkflowInstance）

**理由**：
1. **数据一致性**：模板修改不影响已创建实例（实例不存nodes/edges）
2. **语义清晰**：Instance专门表示"模板的实例"，Task表示"执行任务"
3. **扩展性**：未来可以在Instance层面添加更多元数据
4. **符合设计原则**：模板定义结构，实例记录执行

**实现要点**：
- WorkflowInstance 存储实例元数据和最终结果
- WorkflowExecutionTask 存储执行过程（可关联instance_id）
- 查询时：`SELECT * FROM workflow_instances WHERE template_id = ?`

---

## 二、前端交互方案对比

### 方案A：Workbench内直接保存（推荐）

**交互流程**：
```
用户进入Workbench
    ↓
拖拽节点，搭建Workflow
    ↓
点击"保存模板"按钮（顶部工具栏）
    ↓
弹出保存对话框：
  - 模板名称
  - 描述
  - 是否允许修改（默认false）
    ↓
保存成功，显示提示
    ↓
继续编辑或执行
```

**UI预览**：
```
┌─────────────────────────────────────────────────────────────┐
│  Workbench                                    [保存模板] [执行] │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────┐                                               │
│  │  START  │                                               │
│  └────┬────┘                                               │
│       │                                                     │
│       ▼                                                     │
│  ┌─────────────────────┐                                   │
│  │   App Node          │                                   │
│  │   简历分析          │                                   │
│  └────┬────────────────┘                                   │
│       │                                                     │
│       ▼                                                     │
│  ┌─────────────────────┐                                   │
│  │ Condition Node      │                                   │
│  └─────────────────────┘                                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘

点击"保存模板"后：
┌─────────────────────────────────────────────────────────────┐
│  保存Workflow模板                              [×]          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  模板名称: [________________]                                │
│                                                              │
│  描述: [________________________________]                    │
│       [________________________________]                    │
│                                                              │
│  ☑ 允许修改模板结构（节点、连线）                           │
│     (取消勾选后，使用模板时只能配置节点参数)                 │
│                                                              │
│  [取消]  [保存]                                             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**优点**：
- ✅ **操作流畅**：搭建完直接保存，无需切换页面
- ✅ **上下文清晰**：保存时能看到完整的workflow结构
- ✅ **符合用户习惯**：类似"保存文件"的操作
- ✅ **开发简单**：只需添加一个保存按钮和对话框

**缺点**：
- ⚠️ Workbench界面可能略显拥挤（需要加按钮）
- ⚠️ 模板管理功能较弱（需要单独页面查看所有模板）

---

### 方案B：独立模板库页面

**交互流程**：
```
用户进入Workbench
    ↓
拖拽节点，搭建Workflow
    ↓
点击"保存模板"按钮
    ↓
跳转到"模板库"页面
    ↓
在模板库页面填写信息并保存
    ↓
可以选择：
  - 返回Workbench继续编辑
  - 在模板库中管理模板
  - 使用模板创建实例
```

**UI预览**：
```
┌─────────────────────────────────────────────────────────────┐
│  Workbench                                    [保存模板] [执行] │
├─────────────────────────────────────────────────────────────┤
│  (当前workflow内容)                                          │
└─────────────────────────────────────────────────────────────┘

点击"保存模板"后跳转：
┌─────────────────────────────────────────────────────────────┐
│  ← 返回Workbench    Workflow模板库        [+ 新建模板]       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────┐  ┌──────────────────┐              │
│  │ 📋 简历审核流程   │  │ 📋 合同审批流程   │              │
│  │                  │  │                  │              │
│  │ 状态: Draft      │  │ 状态: Draft      │              │
│  │ 创建者: 我        │  │ 创建者: 我        │              │
│  │ 创建时间: 1.15   │  │ 创建时间: 1.14   │              │
│  │                  │  │                  │              │
│  │ [使用模板] [编辑] │  │ [使用模板] [编辑] │              │
│  └──────────────────┘  └──────────────────┘              │
│                                                              │
│  新建模板表单：                                              │
│  ┌──────────────────────────────────────────────────┐    │
│  │ 模板名称: [________________]                      │    │
│  │ 描述: [________________________________]          │    │
│  │ ☑ 允许修改模板结构                                │    │
│  │ [取消] [保存]                                     │    │
│  └──────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**优点**：
- ✅ **功能完整**：可以集中管理所有模板
- ✅ **界面清晰**：Workbench专注编辑，模板库专注管理
- ✅ **扩展性好**：未来可以添加模板分类、搜索等功能

**缺点**：
- ⚠️ **操作步骤多**：需要跳转页面，打断工作流
- ⚠️ **开发工作量大**：需要新建页面、路由、状态管理
- ⚠️ **用户体验略差**：保存模板需要离开当前上下文

---

### 推荐方案：方案A（Workbench内直接保存）+ 轻量级模板列表

**理由**：
1. **1.15交付版本**：功能简单，不需要复杂的模板管理
2. **用户体验优先**：保存操作应该流畅，不打断工作
3. **渐进式开发**：先实现保存功能，后续可以添加模板库页面

**实现方案**：
- Workbench顶部添加"保存模板"按钮
- 点击后弹出对话框，填写信息保存
- 添加一个简单的"我的模板"侧边栏或下拉菜单
- 可以从模板列表中选择并加载到Workbench

**后续扩展**：
- 如果用户反馈需要更强大的模板管理，再添加独立页面

---

## 三、资源类型方案对比

### 当前ApprovalWorkflowTemplate的resource_type用途

**现有设计**：
- `resource_type` 用于区分不同的业务场景
- 例如：`policy_revision`（政策修订）、`policy_deviation`（政策偏离）、`contract`（合同）
- 每个resource_type可以有默认模板（`is_default = true`）

**现有使用场景**：
```
ApprovalWorkflowTemplate (审批工作流模板)
├─ resource_type: "policy_revision"
│   └─ 用于政策修订的审批流程
├─ resource_type: "policy_deviation"  
│   └─ 用于政策偏离的审批流程
└─ resource_type: "contract"
    └─ 用于合同审批流程
```

---

### 方案A：使用通用资源类型 "workbench_workflow"

**设计**：
```python
ApprovalWorkflowTemplate
├─ resource_type: "workbench_workflow"  # 新增
│   └─ 用于Workbench创建的通用工作流
├─ resource_type: "policy_revision"      # 现有
├─ resource_type: "policy_deviation"     # 现有
└─ resource_type: "contract"             # 现有
```

**优点**：
- ✅ **简单直接**：所有Workbench模板统一类型
- ✅ **查询简单**：`WHERE resource_type = 'workbench_workflow'`
- ✅ **不影响现有功能**：审批流程和Workbench流程分离
- ✅ **开发快速**：无需修改现有代码逻辑

**缺点**：
- ⚠️ **语义不够细分**：如果未来Workbench有不同类型工作流，需要扩展
- ⚠️ **资源类型字段意义减弱**：对于Workbench来说，resource_type意义不大

**数据示例**：
```sql
-- Workbench模板
INSERT INTO approval_workflow_templates 
(name, resource_type, workflow_nodes, workflow_edges)
VALUES 
('简历审核流程', 'workbench_workflow', '[...]', '[...]');

-- 审批流程模板（现有）
INSERT INTO approval_workflow_templates 
(name, resource_type, ...)
VALUES 
('政策修订审批', 'policy_revision', ...);
```

---

### 方案B：不指定resource_type，设为NULL或空字符串

**设计**：
```python
ApprovalWorkflowTemplate
├─ resource_type: "policy_revision"      # 审批流程
├─ resource_type: "policy_deviation"    # 审批流程
├─ resource_type: "contract"             # 审批流程
└─ resource_type: NULL                  # Workbench工作流（不关联具体资源）
```

**优点**：
- ✅ **语义清晰**：NULL表示"不关联特定资源类型"
- ✅ **查询灵活**：`WHERE resource_type IS NULL` 查询Workbench模板
- ✅ **未来扩展**：如果Workbench需要细分，可以添加新类型

**缺点**：
- ⚠️ **需要处理NULL**：查询时需要特殊处理
- ⚠️ **可能违反业务逻辑**：如果系统要求resource_type必填，需要修改

**数据示例**：
```sql
-- Workbench模板
INSERT INTO approval_workflow_templates 
(name, resource_type, workflow_nodes, workflow_edges)
VALUES 
('简历审核流程', NULL, '[...]', '[...]');
```

---

### 方案C：创建新的WorkflowTemplate表（不推荐）

**设计**：
```python
# 新建表
WorkflowTemplate (专门用于Workbench)
├─ id
├─ name
├─ workflow_nodes
├─ workflow_edges
└─ ...

# 保留现有表
ApprovalWorkflowTemplate (专门用于审批流程)
```

**优点**：
- ✅ **完全分离**：Workbench和审批流程完全独立
- ✅ **表结构清晰**：每个表职责单一

**缺点**：
- ❌ **代码重复**：两个表结构几乎相同，需要维护两套代码
- ❌ **查询复杂**：如果需要统一查询，需要UNION
- ❌ **开发工作量大**：需要新建表、迁移数据、修改所有相关代码

---

### 推荐方案：方案A（使用 "workbench_workflow"）

**理由**：
1. **复用现有基础设施**：ApprovalWorkflowTemplate已经支持JSONB的nodes/edges
2. **开发效率高**：无需新建表，只需添加新的resource_type值
3. **查询简单**：`WHERE resource_type = 'workbench_workflow'` 即可
4. **未来扩展**：如果Workbench需要细分类型，可以扩展为：
   - `workbench_workflow_document`（文档处理）
   - `workbench_workflow_analysis`（数据分析）
   - 等等

**实现要点**：
- 创建模板时，固定设置 `resource_type = 'workbench_workflow'`
- 查询模板时，过滤 `resource_type = 'workbench_workflow'`
- 不影响现有的审批流程模板（resource_type不同）

---

## 四、综合推荐方案

### 最终方案组合

1. **实例存储**：使用 `WorkflowInstance` + `WorkflowExecutionTask`（关联）
2. **前端交互**：Workbench内直接保存 + 简单模板列表
3. **资源类型**：使用 `resource_type = 'workbench_workflow'`

### 数据模型设计

```python
# 模板表（复用现有）
ApprovalWorkflowTemplate
├─ id
├─ name: "简历审核流程"
├─ resource_type: "workbench_workflow"  # 新增
├─ workflow_nodes: JSONB  # 节点定义
├─ workflow_edges: JSONB  # 边定义
├─ is_editable: bool = False  # 是否允许修改（新增）
└─ created_by, create_time

# 实例表（使用现有）
WorkflowInstance
├─ id
├─ template_id: FK → ApprovalWorkflowTemplate
├─ status: "pending" | "running" | "completed" | "failed"
├─ current_node_id
├─ execution_result: JSONB  # 最终结果
└─ created_by, create_time

# 执行任务表（扩展）
WorkflowExecutionTask
├─ id
├─ task_id: UUID
├─ instance_id: FK → WorkflowInstance  # 新增关联
├─ nodes: JSONB  # 从模板加载
├─ edges: JSONB  # 从模板加载
├─ app_node_configs: JSONB  # 配置覆盖
├─ status, node_results, thread_id
└─ created_by, create_time
```

### 核心API设计

```python
# 1. 保存模板
POST /api/v1/workbench/workflows/save-template
{
  "name": "简历审核流程",
  "description": "...",
  "nodes": [...],
  "edges": [...],
  "is_editable": false
}

# 2. 获取模板列表
GET /api/v1/workbench/workflow-templates?resource_type=workbench_workflow

# 3. 从模板创建实例
POST /api/v1/workbench/workflows/create-instance
{
  "template_id": 1,
  "app_node_configs": {
    "app-1": {
      "inputMode": "file",
      "dagRunId": "dag_run_123"
    }
  },
  "input_data": {}
}

# 4. 执行实例（复用现有，添加instance_id关联）
POST /api/v1/workbench/workflows/execute
{
  "instance_id": 1,  # 新增：如果提供，表示从模板实例执行
  "nodes": [...],    # 如果提供instance_id，这些从模板加载
  "edges": [...],
  "app_node_configs": {...}
}
```

---

## 五、决策建议

### 如果您选择推荐方案：

**开发工作量**：
- 后端：2-3天（API开发 + 数据模型调整）
- 前端：2-3天（保存对话框 + 模板列表 + 实例创建界面）
- 测试：1天
- **总计：5-7天**

**风险**：
- ✅ 低风险：复用现有模型，改动小
- ✅ 向后兼容：不影响现有功能

**后续扩展**：
- 可以轻松添加模板库页面
- 可以添加模板分类、搜索等功能
- 可以支持模板版本管理

---

## 六、需要您确认的问题

1. **实例存储方案**：是否同意使用 `WorkflowInstance` + `WorkflowExecutionTask` 关联？
2. **前端交互**：是否同意在Workbench内直接保存，后续再考虑模板库页面？
3. **资源类型**：是否同意使用 `resource_type = 'workbench_workflow'`？
4. **模板可修改字段**：`is_editable` 字段名是否合适？还是用 `allow_structure_edit`？

请确认后，我将开始实现。



