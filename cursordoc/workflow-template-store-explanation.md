# Workflow Template 完整使用流程详解

## 目录

1. [什么是 Store？](#一什么是-store)
2. [完整使用流程](#二完整使用流程)
   - [场景1：保存 Workflow 为 Template](#场景1保存-workflow-为-template)
   - [场景2：从 Marketplace 加载 Template](#场景2从-marketplace-加载-template)
   - [场景3：配置并运行 Workflow Template](#场景3配置并运行-workflow-template)
   - [场景4：清空 Workflow Template](#场景4清空-workflow-template)
3. [执行结果存储位置](#三执行结果存储位置)
4. [数据模型关系](#四数据模型关系)

---

## 一、什么是 Store？

**Store（状态存储）** = 全局状态管理容器

- **作用**：在多个组件之间共享数据
- **类比**：就像应用的"内存数据库"，所有组件都可以访问和修改
- **技术实现**：使用 Zustand 状态管理库

### Store 中的 workflowTemplate 数据结构

```typescript
// frontend/src/stores/workbenchStore.ts

workflowTemplate: {
  templateId: number | null      // 模板ID，例如：8
  appId: number | null            // 应用ID，例如：111
  name: string | null             // 模板名称，例如："011201120112"
  description: string | null      // 模板描述
  nodes: any[] | null             // 节点数据数组（6个节点）
  edges: any[] | null             // 连线数据数组（5条连线）
} | null                          // 未加载模板时为 null
```

### Store 操作函数

```typescript
// 加载模板到 Store
loadWorkflowTemplate(template) {
  set({ workflowTemplate: template })
}

// 清空 Store 中的模板数据
clearWorkflowTemplate() {
  set({ workflowTemplate: null })  // ⬅️ 将 workflowTemplate 设置为 null
}
```

---

## 二、完整使用流程

### 场景1：保存 Workflow 为 Template

```
┌─────────────────────────────────────────────────────────────┐
│ 步骤1：用户在 Workbench 中搭建 Workflow                     │
│                                                             │
│ 画布状态：                                                  │
│ - 6 个节点（Start, App, Condition, HumanInTheLoop, End...）│
│ - 5 条连线                                                  │
│ - 节点已配置（App Node 已上传文件，HumanInTheLoop 已设置）  │
└───────────────────┬─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────────────┐
│ 步骤2：用户点击 "Save Template" 按钮                        │
│ - 填写模板名称："011201120112"                              │
│ - 填写描述（可选）                                           │
└───────────────────┬─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────────────┐
│ 步骤3：前端清理运行时数据                                   │
│ cleanNodeExecutionState() 执行：                            │
│ - 保留：requirement, inputMode（用户配置）                  │
│ - 清空：files, url, dagRunId, status（运行时数据）          │
│ - 清空：outputs, error（执行结果）                          │
└───────────────────┬─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────────────┐
│ 步骤4：调用后端 API                                         │
│ POST /api/v1/workbench/workflows/save-template             │
│ Body: {                                                     │
│   name: "011201120112",                                     │
│   description: "...",                                       │
│   nodes: [...],  // 已清理运行时数据                         │
│   edges: [...]                                              │
│ }                                                           │
└───────────────────┬─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────────────┐
│ 步骤5：后端保存到数据库                                     │
│                                                             │
│ 数据库表：approval_workflow_templates                       │
│ ┌─────────────────────────────────────────────────────┐   │
│ │ id: 8                                               │   │
│ │ name: "011201120112"                                │   │
│ │ workflow_nodes: [...]  // 节点结构（无运行时数据）   │   │
│ │ workflow_edges: [...]  // 连线结构                  │   │
│ │ is_active: true                                     │   │
│ │ created_by: admin_user_id                           │   │
│ └─────────────────────────────────────────────────────┘   │
│                                                             │
│ 数据库表：app_node                                          │
│ ┌─────────────────────────────────────────────────────┐   │
│ │ id: 111                                             │   │
│ │ app_id: uuid("71dbadfe-...")                        │   │
│ │ name: "011201120112"                                │   │
│ │ app_type: "workflow"                                │   │
│ │ workflow_template_id: 8  ⬅️ 关联到 template         │   │
│ │ status: "published"                                 │   │
│ │ category: "Workflow"                                │   │
│ └─────────────────────────────────────────────────────┘   │
│                                                             │
│ 数据库表：resources + permissions                          │
│ - 创建 Resource: "app_111"                                 │
│ - 创建 Permission: 全局 read 权限（所有用户可访问）         │
└───────────────────┬─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────────────┐
│ 步骤6：Template 出现在 Application Marketplace             │
│                                                             │
│ Application Marketplace UI:                                │
│ ┌─────────────────────────────────────────────────────┐   │
│ │ Featured                                            │   │
│ │ ┌──────────────┐                                    │   │
│ │ │ WORKFLOW     │                                    │   │
│ │ │ 011201120112 │  ⬅️ 新保存的模板显示在这里        │   │
│ │ │ by admin@... │                                    │   │
│ │ └──────────────┘                                    │   │
│ └─────────────────────────────────────────────────────┘   │
│                                                             │
│ ✅ Template 已保存并发布，所有用户都可以在 Marketplace 看到 │
└─────────────────────────────────────────────────────────────┘
```

### 场景2：从 Marketplace 加载 Template

```
┌─────────────────────────────────────────────────────────────┐
│ 步骤1：用户在 Application Marketplace 点击 Template         │
│                                                             │
│ 用户操作：                                                  │
│ - 点击 "WORKFLOW 011201120112" 卡片                        │
└───────────────────┬─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────────────┐
│ 步骤2：WorkbenchSidebar 调用后端 API                       │
│ GET /api/v1/workbench/workflows/from-app/{app_id}          │
│ app_id = 111                                                │
└───────────────────┬─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────────────┐
│ 步骤3：后端查询数据库并返回 Template 数据                   │
│                                                             │
│ 后端逻辑：                                                  │
│ 1. 通过 app_id 查找 app_node (id=111)                      │
│ 2. 通过 workflow_template_id (8) 查找 template             │
│ 3. 返回 template 数据：                                     │
│    {                                                        │
│      template_id: 8,                                        │
│      app_id: 111,                                           │
│      name: "011201120112",                                  │
│      nodes: [...],  // 6 个节点（无运行时数据）              │
│      edges: [...]   // 5 条连线                             │
│    }                                                        │
└───────────────────┬─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────────────┐
│ 步骤4：调用 loadWorkflowTemplate() 存入 Store              │
│                                                             │
│ useWorkbenchStore.getState().loadWorkflowTemplate({         │
│   templateId: 8,                                            │
│   appId: 111,                                               │
│   name: "011201120112",                                     │
│   nodes: [...],                                             │
│   edges: [...]                                              │
│ })                                                          │
│                                                             │
│ Store 状态：                                                │
│ workflowTemplate = {                                        │
│   templateId: 8,                                            │
│   appId: 111,                                               │
│   name: "011201120112",                                     │
│   nodes: [6个节点],                                         │
│   edges: [5条连线]                                          │
│ }                                                           │
└───────────────────┬─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────────────┐
│ 步骤5：FlowWorkspace 组件监听 Store 变化                    │
│                                                             │
│ const workflowTemplate = useWorkbenchStore(                 │
│   (state) => state.workflowTemplate                        │
│ )                                                           │
│                                                             │
│ useEffect(() => {                                           │
│   if (workflowTemplate && workflowTemplate.nodes) {         │
│     // Store 中有数据，自动加载到画布                       │
│   }                                                         │
│ }, [workflowTemplate])                                     │
└───────────────────┬─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────────────┐
│ 步骤6：自动加载 Template 到画布                             │
│                                                             │
│ FlowWorkspace 执行：                                        │
│ 1. setNodes(templateNodes)   // 加载 6 个节点到画布         │
│ 2. setEdges(templateEdges)   // 加载 5 条连线到画布         │
│ 3. setIsTemplateMode(true)   // 进入模板模式                │
│ 4. setTemplateAppId(111)     // 设置模板应用ID              │
│ 5. 设置节点为只读（不可拖拽、不可删除）                      │
└───────────────────┬─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────────────┐
│ 步骤7：用户看到 Template 显示在画布上                       │
│                                                             │
│ 画布状态：                                                  │
│ - 6 个节点显示（只读，不可拖拽/删除）                        │
│ - 5 条连线显示（只读，不可删除）                             │
│ - 所有节点的 executionState 为初始状态（无文件、无运行数据）│
│ - "Clear Workflow" 按钮可用（我们刚实现的功能）             │
│                                                             │
│ ✅ Template 加载完成，用户可以配置节点参数                  │
└─────────────────────────────────────────────────────────────┘
```

### 场景3：配置并运行 Workflow Template

```
┌─────────────────────────────────────────────────────────────┐
│ 步骤1：用户配置 App Node                                    │
│                                                             │
│ 用户操作：                                                  │
│ 1. 点击 App Node，打开配置面板                              │
│ 2. 选择 Input Mode: File Upload                            │
│ 3. 上传文件（例如：供应商资料.pdf）                          │
│ 4. 设置 Requirement（提示词）                               │
│                                                             │
│ 配置数据存储在：                                            │
│ - nodes[app-1].data.executionState = {                     │
│     inputMode: 'file',                                      │
│     files: [File对象],                                      │
│     requirement: '...',                                     │
│     status: 'idle'                                          │
│   }                                                         │
└───────────────────┬─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────────────┐
│ 步骤2：用户点击 "Run Workflow" 按钮                         │
└───────────────────┬─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────────────┐
│ 步骤3：前端预处理文件上传                                   │
│                                                             │
│ handleRunWorkflow() 执行：                                  │
│ 1. 检查所有 App Node 的配置                                 │
│ 2. 对于 file mode，先上传文件获取 dag_run_id                │
│ 3. 收集所有 app_node_configs：                              │
│    {                                                        │
│      "app-1": {                                             │
│        inputMode: "file",                                   │
│        dagRunId: "manual__2026-01-12T...",                  │
│        requirement: "...",                                   │
│        appNodeId: 105                                       │
│      }                                                      │
│    }                                                        │
└───────────────────┬─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────────────┐
│ 步骤4：调用后端 API 创建 Instance                           │
│                                                             │
│ POST /api/v1/workbench/workflows/create-instance           │
│ Body: {                                                     │
│   template_id: 8,                                           │
│   app_id: 111,                                              │
│   input_data: {},                                           │
│   app_node_configs: {                                       │
│     "app-1": { ... }                                        │
│   }                                                         │
│ }                                                           │
└───────────────────┬─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────────────┐
│ 步骤5：后端创建 WorkflowInstance                            │
│                                                             │
│ 数据库表：workflow_instances                                │
│ ┌─────────────────────────────────────────────────────┐   │
│ │ id: 2                                               │   │
│ │ template_id: 8  ⬅️ 关联到 template                  │   │
│ │ app_id: 111       ⬅️ 关联到 marketplace app         │   │
│ │ status: "pending"                                   │   │
│ │ created_by: user_id                                 │   │
│ │ execution_result: null  ⬅️ 执行完成后更新           │   │
│ └─────────────────────────────────────────────────────┘   │
│                                                             │
│ ✅ WorkflowInstance 记录创建完成                            │
└───────────────────┬─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────────────┐
│ 步骤6：后端创建 WorkflowExecutionTask                       │
│                                                             │
│ 数据库表：workflow_execution_tasks                          │
│ ┌─────────────────────────────────────────────────────┐   │
│ │ id: 16                                              │   │
│ │ task_id: "f9e61bc6-..."  ⬅️ UUID                    │   │
│ │ instance_id: 2  ⬅️ 关联到 WorkflowInstance          │   │
│ │ nodes: [...]      // 从 template 加载的节点结构      │   │
│ │ edges: [...]      // 从 template 加载的连线结构      │   │
│ │ app_node_configs: {...}  // 用户配置的覆盖          │   │
│ │ status: "pending"                                    │   │
│ │ node_results: {}   ⬅️ 执行过程中实时更新             │   │
│ │ created_by: user_id                                  │   │
│ └─────────────────────────────────────────────────────┘   │
│                                                             │
│ ✅ WorkflowExecutionTask 记录创建完成                       │
└───────────────────┬─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────────────┐
│ 步骤7：后端启动异步执行                                     │
│                                                             │
│ execute_workflow_task() 异步执行：                          │
│ 1. 构建 LangGraph StateGraph                                │
│ 2. 执行 workflow（节点依次执行）                            │
│ 3. 实时更新 WorkflowExecutionTask.node_results              │
│ 4. 通过 WebSocket 推送执行进度到前端                        │
└───────────────────┬─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────────────┐
│ 步骤8：执行过程中实时更新数据库                             │
│                                                             │
│ WorkflowExecutionTask.node_results 实时更新：               │
│ {                                                           │
│   "start-2": {                                              │
│     status: "completed",                                    │
│     output: {...}                                           │
│   },                                                        │
│   "app-1": {                                                │
│     status: "completed",                                    │
│     output: {                                               │
│       conclusion_detail: "...",                             │
│       dag_run_id: "...",                                    │
│       structured_output: {...}                              │
│     }                                                       │
│   },                                                        │
│   "condition-3": { ... },                                   │
│   "humanInTheLoop-4": { ... },                              │
│   "end-5": { ... }                                          │
│ }                                                           │
│                                                             │
│ ✅ 每个节点执行完成后立即保存结果到数据库                    │
└───────────────────┬─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────────────┐
│ 步骤9：执行完成后更新最终结果                               │
│                                                             │
│ 数据库表：workflow_execution_tasks                          │
│ ┌─────────────────────────────────────────────────────┐   │
│ │ id: 16                                              │   │
│ │ status: "completed"  ⬅️ 更新为已完成                │   │
│ │ node_results: { ... }  ⬅️ 所有节点的执行结果         │   │
│ │ update_time: 2026-01-12T...                         │   │
│ └─────────────────────────────────────────────────────┘   │
│                                                             │
│ 数据库表：workflow_instances                                │
│ ┌─────────────────────────────────────────────────────┐   │
│ │ id: 2                                               │   │
│ │ status: "completed"  ⬅️ 更新为已完成                │   │
│ │ execution_result: {  ⬅️ 最终执行结果                │   │
│ │   final_output: {...},                               │   │
│ │   summary: {                                         │   │
│ │     total_nodes: 6,                                  │   │
│ │     completed_nodes: 5,                              │   │
│ │     execution_time: "..."                            │   │
│ │   }                                                  │   │
│ │ }                                                    │   │
│ │ update_time: 2026-01-12T...                         │   │
│ └─────────────────────────────────────────────────────┘   │
│                                                             │
│ ✅ 执行结果已保存到数据库                                    │
└───────────────────┬─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────────────┐
│ 步骤10：前端显示执行结果                                    │
│                                                             │
│ 用户界面：                                                  │
│ 1. 画布上节点显示执行状态（completed, failed等）            │
│ 2. 可以通过 WebSocket 实时查看执行进度                      │
│ 3. 可以查看每个节点的输出结果                               │
│ 4. URL 中包含 task_id，可以刷新页面后查看结果               │
│                                                             │
│ ✅ Workflow 执行完成，结果已保存                            │
└─────────────────────────────────────────────────────────────┘
```

### 场景4：清空 Workflow Template

```
┌─────────────────────────────────────────────────────────────┐
│ 步骤1：用户在 Template Mode 下点击 "Clear Workflow" 按钮   │
└───────────────────┬─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────────────┐
│ 步骤2：显示确认对话框                                       │
│                                                             │
│ window.confirm("确认清空工作流？")                          │
└───────────────────┬─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────────────┐
│ 步骤3：用户点击确认                                         │
└───────────────────┬─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────────────┐
│ 步骤4：clearCanvas() 函数执行                               │
│                                                             │
│ clearCanvas() {                                             │
│   // a. 清空画布显示                                         │
│   setNodes([])              // 画布节点数组 = []            │
│   setEdges([])              // 画布连线数组 = []            │
│                                                             │
│   // b. 清空模板相关状态                                     │
│   if (isTemplateMode) {                                     │
│     setIsTemplateMode(false)  // 退出模板模式               │
│     setTemplateAppId(null)     // 清空模板应用ID            │
│     clearWorkflowTemplate()    // ⬅️ 清空 Store             │
│   }                                                         │
│ }                                                           │
└───────────────────┬─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────────────┐
│ 步骤5：clearWorkflowTemplate() 执行                         │
│                                                             │
│ clearWorkflowTemplate() {                                   │
│   set({ workflowTemplate: null })                           │
│ }                                                           │
│                                                             │
│ Store 状态变为：                                            │
│ workflowTemplate = null  ⬅️ 数据被清空                      │
└───────────────────┬─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────────────┐
│ 步骤6：FlowWorkspace 的 useEffect 检测到变化                │
│                                                             │
│ useEffect(() => {                                           │
│   if (!workflowTemplate) {                                  │
│     // Store 中的 workflowTemplate 变为 null                │
│     setIsTemplateMode(false)                                │
│     setTemplateAppId(null)                                  │
│   }                                                         │
│ }, [workflowTemplate])                                     │
└───────────────────┬─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────────────┐
│ 步骤7：用户界面变为空白画布                                 │
│                                                             │
│ 画布状态：                                                  │
│ - 画布上没有任何节点和连线                                   │
│ - 退出模板模式（可以添加新节点）                             │
│ - Store 中的 workflowTemplate = null                        │
│                                                             │
│ ✅ 清空完成，用户可以重新加载其他模板或创建新工作流          │
└─────────────────────────────────────────────────────────────┘
```

---

## 三、执行结果存储位置

### 3.1 执行结果保存在哪里？

**运行 workflow template 后，执行结果保存在数据库的两个表中：**

#### 1. `workflow_execution_tasks` 表（详细节点结果）

```sql
-- 表：workflow_execution_tasks
SELECT 
  id,                    -- 16
  task_id,               -- "f9e61bc6-..."
  instance_id,           -- 2 (关联到 WorkflowInstance)
  status,                -- "completed"
  node_results,          -- JSONB: 每个节点的详细执行结果
  created_by,            -- 执行者用户ID
  create_time,           -- 创建时间
  update_time            -- 更新时间
FROM workflow_execution_tasks
WHERE task_id = 'f9e61bc6-...';
```

**node_results 字段结构**：
```json
{
  "start-2": {
    "status": "completed",
    "output": {...}
  },
  "app-1": {
    "status": "completed",
    "output": {
      "conclusion_detail": "...",
      "dag_run_id": "manual__...",
      "structured_output": {...}
    }
  },
  "condition-3": {
    "status": "completed",
    "output": {
      "selected_branch": "True",
      "condition_result": true,
      ...
    }
  },
  "humanInTheLoop-4": {
    "status": "completed",
    "output": {
      "decision": "approved",
      ...
    }
  },
  "end-5": {
    "status": "completed",
    "output": {...}
  }
}
```

#### 2. `workflow_instances` 表（最终执行结果）

```sql
-- 表：workflow_instances
SELECT 
  id,                    -- 2
  template_id,           -- 8 (关联到 ApprovalWorkflowTemplate)
  app_id,                -- 111 (关联到 AppNode)
  status,                -- "completed"
  execution_result,      -- JSONB: 最终执行结果汇总
  created_by,            -- 执行者用户ID
  create_time,           -- 创建时间
  update_time            -- 更新时间
FROM workflow_instances
WHERE id = 2;
```

**execution_result 字段结构**：
```json
{
  "final_output": {
    "end-5": {
      "status": "completed",
      "output": {...}
    }
  },
  "summary": {
    "total_nodes": 6,
    "completed_nodes": 5,
    "execution_time": "0:03:15"
  }
}
```

### 3.2 如何查看执行结果？

#### 方式1：通过前端 UI 查看（实时）

- **执行过程中**：通过 WebSocket 实时推送，画布上节点状态实时更新
- **执行完成后**：可以点击节点查看详细输出结果
- **URL 中包含 task_id**：`/workbench?task_id=f9e61bc6-...`，刷新页面后可查看结果

#### 方式2：通过 API 查询

```bash
# 查询任务详情（包含所有节点结果）
GET /api/v1/workbench/workflow-tasks/{task_id}

# 查询实例列表
GET /api/v1/workbench/workflow-instances?template_id=8
```

#### 方式3：直接查询数据库

```sql
-- 查看某个任务的所有节点结果
SELECT node_results 
FROM workflow_execution_tasks 
WHERE task_id = 'f9e61bc6-...';

-- 查看某个模板的所有执行实例
SELECT id, status, execution_result, create_time
FROM workflow_instances 
WHERE template_id = 8
ORDER BY create_time DESC;
```

### 3.3 执行结果的持久性

✅ **执行结果会被永久保存**：

- `workflow_execution_tasks` 记录：**永久保存**，不会被自动删除
- `workflow_instances` 记录：**永久保存**，不会被自动删除
- 可以通过 `task_id` 或 `instance_id` 随时查询历史执行结果

---

## 四、数据模型关系

### 4.1 数据库表关系图

```
┌─────────────────────────────────────┐
│  approval_workflow_templates        │  Template 定义（1个）
│  ─────────────────────────────────  │
│  id: 8                              │
│  name: "011201120112"               │
│  workflow_nodes: [...]              │
│  workflow_edges: [...]              │
│  is_active: true                    │
└──────────────┬──────────────────────┘
               │
               │ workflow_template_id
               │
┌──────────────▼──────────────────────┐
│  app_node                           │  Marketplace 入口（1个）
│  ─────────────────────────────────  │
│  id: 111                            │
│  app_type: "workflow"               │
│  workflow_template_id: 8  ⬅️        │
│  status: "published"                │
└──────────────┬──────────────────────┘
               │
               │ 用户点击 Marketplace
               │
┌──────────────▼──────────────────────┐
│  workflow_instances                 │  执行实例（多个）
│  ─────────────────────────────────  │
│  id: 2, 3, 4, ...                   │
│  template_id: 8  ⬅️                  │
│  app_id: 111     ⬅️                  │
│  status: "completed"                │
│  execution_result: {...}            │
└──────────────┬──────────────────────┘
               │
               │ instance_id
               │
┌──────────────▼──────────────────────┐
│  workflow_execution_tasks           │  执行任务（1:1）
│  ─────────────────────────────────  │
│  id: 16                             │
│  task_id: "f9e61bc6-..."            │
│  instance_id: 2  ⬅️                  │
│  node_results: {...}                │
│  status: "completed"                │
└─────────────────────────────────────┘
```

### 4.2 数据流转说明

1. **Template 创建**：
   - `approval_workflow_templates` (1条记录)
   - `app_node` (1条记录，关联到 template)

2. **Template 执行**（每次运行都会创建新记录）：
   - `workflow_instances` (每次运行创建1条记录)
   - `workflow_execution_tasks` (每次运行创建1条记录，关联到 instance)

3. **结果查询**：
   - 通过 `template_id` 查询所有执行实例
   - 通过 `instance_id` 查询执行任务和详细结果
   - 通过 `task_id` 查询任务详情

---

## 五、总结

### 关键概念对比

| 概念 | 说明 | 存储位置 | 数量关系 |
|------|------|----------|----------|
| **Template** | 工作流模板定义 | `approval_workflow_templates` | 1个模板 |
| **AppNode** | Marketplace 显示入口 | `app_node` | 1个入口（关联1个模板）|
| **Instance** | 模板的执行实例 | `workflow_instances` | 1个模板 → 多个实例 |
| **Task** | 具体的执行任务 | `workflow_execution_tasks` | 1个实例 → 1个任务 |

### 数据存储位置总结

| 数据 | 存储位置 | 是否永久保存 |
|------|----------|-------------|
| Template 定义 | `approval_workflow_templates` 表 | ✅ 是 |
| Marketplace 入口 | `app_node` 表 | ✅ 是 |
| 执行实例记录 | `workflow_instances` 表 | ✅ 是 |
| 详细节点结果 | `workflow_execution_tasks.node_results` | ✅ 是 |
| 最终执行结果 | `workflow_instances.execution_result` | ✅ 是 |
| Store 中的模板数据 | 前端内存（临时） | ❌ 否（刷新页面会丢失）|

### 重要提示

1. **Template 是永久保存的**：一旦保存，就会一直存在于数据库
2. **执行结果是永久保存的**：每次运行的结果都会永久保存在数据库中
3. **Store 是临时存储**：前端 Store 中的 workflowTemplate 只是临时数据，用于控制 UI 状态
4. **清空 Store ≠ 删除 Template**：清空 Store 只是退出模板模式，不会删除数据库中的 Template
