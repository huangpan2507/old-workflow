# Workflow三大使用场景通俗解释

## 场景概览

本文档用通俗易懂的方式解释三个使用场景，并说明如何通过Workflow的6大Node来实现。

---

## 场景1：审计发现问题 → 分配任务 → 更新信息 → 上级检查

### 📖 通俗解释

**就像公司内部的"问题处理流水线"**：

1. **发现问题**：审计部门发现"供应商A的资质证书即将过期"
2. **提出建议**：AI分析后建议"需要更新资质证书"
3. **管理层决策**：财务总监决定"这个任务需要处理"
4. **分配任务**：系统自动把任务分配给"采购经理"（todo list出现待办）
5. **执行任务**：采购经理联系供应商，更新了证书信息
6. **上级检查**：财务总监检查更新结果，写下评论"证书已更新，符合要求"
7. **完成**：整个流程结束

### 🔄 通过Workflow实现

```
┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
│  Start  │ --> │   App   │ --> │Condition│ --> │  HITL   │
│  Node   │     │  Node   │     │  Node   │     │  Node   │
│         │     │(AI分析) │     │(判断)   │     │(Assign) │
└─────────┘     └─────────┘     └─────────┘     └─────────┘
  接收审计报告     分析问题并        判断是否需要      分配任务给
                 提出建议          管理层决策        采购经理
                                                      │
                                                      ▼
┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
│  End    │ <-- │  Tool   │ <-- │  HITL    │ <-- │  HITL    │
│  Node   │     │  Node   │     │  Node    │     │  Node    │
│         │     │(发邮件) │     │(Request │     │(Assign)  │
└─────────┘     └─────────┘     │Update)  │     └─────────┘
  流程结束        通知相关人员     等待更新信息      任务完成
```

### 📋 详细步骤

#### Step 1: Start Node（起始节点）🟢
- **输入**：审计报告文件（PDF/Word）
- **作用**：接收审计报告，作为整个流程的起点
- **输出**：`{ "audit_report": "report.pdf", "issue_type": "certificate_expiry" }`

#### Step 2: App Node（AI分析节点）🔵
- **配置**：使用"审计问题分析"Application
- **作用**：AI分析审计报告，识别问题并提出建议
- **输出**：
  ```json
  {
    "issues_found": ["供应商A资质证书即将过期"],
    "recommendations": ["需要更新资质证书"],
    "severity": "high"
  }
  ```

#### Step 3: Condition Node（条件判断节点）🟠
- **条件**：`severity === "high" AND issues_found.length > 0`
- **作用**：判断问题是否严重，是否需要管理层决策
- **True路径**：需要管理层决策 → 进入HITL节点（管理层审批）
- **False路径**：问题不严重 → 直接结束

#### Step 4: HITL Node - Approval（管理层审批）🟡
- **任务类型**：`approval`（审批）
- **配置**：
  ```json
  {
    "taskType": "approval",
    "approvalConfig": {
      "approverJobTitleIds": [1],  // 财务总监的职位ID
      "title": "审计问题处理决策",
      "description": "发现供应商资质证书即将过期，需要决定如何处理"
    }
  }
  ```
- **作用**：暂停workflow，等待财务总监审批
- **结果**：财务总监审批通过，决定"分配给采购经理处理"

#### Step 5: HITL Node - Assign（分配任务）🟡
- **任务类型**：`assign`（分配）
- **配置**：
  ```json
  {
    "taskType": "assign",
    "assignConfig": {
      "assigneeJobTitleIds": [2],  // 采购经理的职位ID
      "title": "更新供应商资质证书",
      "description": "请联系供应商A，更新即将过期的资质证书",
      "priority": "high",
      "dueDate": "2024-01-20T10:00:00Z"
    }
  }
  ```
- **作用**：
  - 在消息中心创建"分配"任务
  - 采购经理的todo list出现待办事项
  - 等待采购经理完成任务
- **结果**：采购经理完成任务，标记为"已完成"

#### Step 6: HITL Node - Request for Update（请求更新信息）🟡
- **任务类型**：`request_for_update`（请求更新）
- **配置**：
  ```json
  {
    "taskType": "request_for_update",
    "updateConfig": {
      "recipientJobTitleIds": [2],  // 采购经理的职位ID
      "title": "请更新证书信息",
      "description": "请提交更新后的证书信息",
      "requiredFields": ["update_info", "certificate_file"]
    }
  }
  ```
- **作用**：
  - 采购经理收到"更新请求"消息
  - 采购经理填写更新信息并上传新证书
  - 系统收集更新信息
- **结果**：采购经理提交了更新信息

#### Step 7: HITL Node - Approval（上级检查）🟡
- **任务类型**：`approval`（审批）
- **配置**：
  ```json
  {
    "taskType": "approval",
    "approvalConfig": {
      "approverJobTitleIds": [1],  // 财务总监的职位ID
      "title": "检查证书更新结果",
      "description": "请检查采购经理提交的证书更新信息"
    }
  }
  ```
- **作用**：
  - 财务总监收到"审批"消息
  - 财务总监查看更新信息，写下评论"证书已更新，符合要求"
  - 财务总监审批通过
- **结果**：审批通过，workflow继续执行

#### Step 8: Tool Node（发送通知）⚙️
- **工具类型**：`email`（邮件）
- **配置**：
  ```json
  {
    "toolType": "email",
    "config": {
      "to": ["procurement@company.com", "finance@company.com"],
      "subject": "供应商资质证书已更新",
      "body": "供应商A的资质证书已成功更新，财务总监已审批通过。"
    }
  }
  ```
- **作用**：自动发送邮件通知相关人员

#### Step 9: End Node（结束节点）🔴
- **作用**：标记workflow执行完成
- **输出**：汇总整个流程的执行结果

### 💡 关键点

1. **HITL Node的三种任务类型**：
   - `approval`：需要审批决策（管理层决策、上级检查）
   - `assign`：分配任务给用户（todo list待办）
   - `request_for_update`：请求用户更新信息

2. **消息中心的作用**：
   - 所有HITL节点创建的任务都会在消息中心显示
   - 用户可以在消息中心处理任务（审批/更新/完成待办）

3. **异步执行**：
   - workflow执行到HITL节点时暂停
   - 等待用户处理任务
   - 用户处理完成后，workflow自动继续执行

---

## 场景2：用户聊天提出需求 → AI生成Workflow → 用户编辑保存

### 📖 通俗解释

**就像"AI工作流助手"**：

1. **用户提出需求**：用户在聊天窗口说"我需要一个供应商审核流程"
2. **AI理解需求**：AI理解用户想要一个审核供应商的workflow
3. **AI搜索市场**：AI去workflow市场搜索是否有现成的"供应商审核"模板
4. **AI生成Workflow**：
   - 如果找到：直接使用现有模板
   - 如果没找到：AI根据当前流程树上的App和workflow节点，自动生成一个新的workflow
5. **给用户链接**：AI生成workflow后，给用户一个链接"点击这里编辑workflow"
6. **用户编辑**：用户点击链接，进入workflow编辑界面，可以调整节点和配置
7. **用户保存**：用户满意后，点击"保存"，workflow保存为模板

### 🔄 通过Workflow实现

```
用户聊天窗口
    │
    ▼
┌─────────────┐
│  Chat Agent │  ← 这是聊天Agent，不是Workflow
│  (LangGraph)│     它理解用户需求，然后生成Workflow
└─────────────┘
    │
    ├─→ 搜索Workflow市场
    │   └─→ 找到模板？ → 使用模板
    │
    └─→ 没找到？ → 根据流程树生成新Workflow
        │
        ▼
┌─────────────────┐
│ 生成的Workflow   │
│ (包含6大Node)    │
└─────────────────┘
    │
    ▼
┌─────────────────┐
│ 给用户编辑链接   │
│ /workbench?workflow_id=123
└─────────────────┘
    │
    ▼
用户编辑界面（Workbench）
    │
    ├─→ 调整节点
    ├─→ 修改配置
    └─→ 保存为模板
```

### 📋 详细步骤

#### Step 1: 用户在聊天窗口提出需求

**用户输入**：
```
"我需要一个供应商审核流程，包括：
1. 分析供应商资料
2. 判断是否符合条件
3. 如果符合，需要财务总监审批
4. 审批通过后发送邮件通知"
```

#### Step 2: Chat Agent理解需求

**Chat Agent处理**：
- 使用LangGraph的Chat Agent
- 理解用户需求，提取关键信息：
  - 流程类型：供应商审核
  - 需要的步骤：分析 → 判断 → 审批 → 通知

#### Step 3: AI搜索Workflow市场

**搜索逻辑**：
```python
# 伪代码
workflow_templates = search_workflow_market("供应商审核")
if workflow_templates:
    # 找到现成模板
    selected_template = workflow_templates[0]
    workflow = load_template(selected_template.id)
else:
    # 没找到，根据流程树生成
    workflow = generate_workflow_from_process_tree(
        user_requirement,
        available_apps,  # 当前流程树上的App
        available_workflow_nodes  # 可用的workflow节点
    )
```

#### Step 4: AI生成Workflow结构

**生成的Workflow包含6大Node**：

```
Start Node
    │
    ▼
App Node (供应商资料分析)
    │
    ▼
Condition Node (判断是否符合条件)
    │
    ├─→ True → HITL Node (财务总监审批)
    │              │
    │              ▼
    │          Tool Node (发送邮件通知)
    │              │
    │              ▼
    └─→ False → End Node (结束)
```

**生成的Workflow配置**：
```json
{
  "nodes": [
    {
      "id": "start-1",
      "type": "start",
      "data": {}
    },
    {
      "id": "app-1",
      "type": "app",
      "data": {
        "appId": 123,  // 供应商分析App
        "prompt": "分析供应商资料，提取关键信息"
      }
    },
    {
      "id": "condition-1",
      "type": "condition",
      "data": {
        "condition": "supplier_score > 80 AND supplier_status === 'active'"
      }
    },
    {
      "id": "hitl-1",
      "type": "humanInTheLoop",
      "data": {
        "taskType": "approval",
        "approvalConfig": {
          "approverJobTitleIds": [1],  // 财务总监
          "title": "供应商审核审批"
        }
      }
    },
    {
      "id": "tool-1",
      "type": "tool",
      "data": {
        "toolType": "email",
        "config": {
          "to": ["procurement@company.com"],
          "subject": "供应商审核结果"
        }
      }
    },
    {
      "id": "end-1",
      "type": "end",
      "data": {}
    }
  ],
  "edges": [
    {"source": "start-1", "target": "app-1"},
    {"source": "app-1", "target": "condition-1"},
    {"source": "condition-1", "target": "hitl-1", "condition": "true"},
    {"source": "condition-1", "target": "end-1", "condition": "false"},
    {"source": "hitl-1", "target": "tool-1"},
    {"source": "tool-1", "target": "end-1"}
  ]
}
```

#### Step 5: AI返回编辑链接

**Chat Agent响应**：
```
我已经为您生成了一个"供应商审核"workflow！

这个workflow包含以下步骤：
1. 分析供应商资料（App Node）
2. 判断是否符合条件（Condition Node）
3. 财务总监审批（HITL Node）
4. 发送邮件通知（Tool Node）

[点击这里编辑workflow](/workbench?workflow_id=123&mode=edit)
```

#### Step 6: 用户编辑Workflow

**用户在Workbench编辑界面**：
- 可以调整节点顺序
- 可以修改节点配置（如更换App、修改审批人）
- 可以添加/删除节点
- 可以预览workflow结构

#### Step 7: 用户保存Workflow

**保存选项**：
- **保存为草稿**：只有自己能看到，可以继续编辑
- **提交审批**：提交给超级管理员审批
- **直接发布**：如果用户是超级管理员，可以直接发布

### 💡 关键点

1. **Chat Agent不是Workflow**：
   - Chat Agent是理解用户需求、生成Workflow的工具
   - 它本身不是workflow，而是workflow的"生成器"

2. **Workflow市场**：
   - 已发布的workflow模板库
   - AI可以搜索和复用现有模板

3. **从流程树生成**：
   - 如果市场没有，AI会根据：
     - 当前流程树上的App（可用的AI应用）
     - 可用的workflow节点类型
     - 用户需求描述
   - 自动生成workflow结构

4. **6大Node的作用**：
   - **Start Node**：workflow入口
   - **App Node**：使用流程树上的App进行分析
   - **Condition Node**：根据分析结果判断
   - **HITL Node**：需要人工审批
   - **Tool Node**：发送通知
   - **End Node**：workflow结束

---

## 场景3：消息中心每个消息作为独立Agent上下文，相同Workflow消息归集

### 📖 通俗解释

**就像"智能消息助手"**：

1. **每个消息都有AI助手**：
   - 用户在消息中心看到一条"供应商审核审批"消息
   - 点击消息，打开详情页
   - 详情页有一个"AI助手"标签页
   - 用户可以在AI助手中提问："这个供应商的评分是多少？"

2. **AI助手知道上下文**：
   - AI助手知道这条消息关联的workflow执行情况
   - AI助手知道这条消息的前置节点（App Node的分析结果）
   - AI助手可以回答关于这条消息的所有问题

3. **相同Workflow的消息归集**：
   - 同一个workflow执行过程中，可能产生多条消息
   - 例如：审批消息、更新请求消息、分配任务消息
   - 这些消息都归集到同一个AI助手上下文中
   - 用户可以在任何一条消息的AI助手中，看到整个workflow的执行情况

### 🔄 通过Workflow实现

```
Workflow执行过程
    │
    ├─→ HITL Node (审批) → 创建消息1
    │
    ├─→ HITL Node (分配) → 创建消息2
    │
    └─→ HITL Node (更新) → 创建消息3
            │
            ▼
┌─────────────────────┐
│  消息中心            │
│  - 消息1 (审批)      │
│  - 消息2 (分配)      │
│  - 消息3 (更新)      │
└─────────────────────┘
    │
    ├─→ 点击消息1
    │   └─→ AI助手上下文 = Workflow执行上下文
    │
    ├─→ 点击消息2
    │   └─→ AI助手上下文 = Workflow执行上下文（相同）
    │
    └─→ 点击消息3
        └─→ AI助手上下文 = Workflow执行上下文（相同）
```

### 📋 详细步骤

#### Step 1: Workflow执行过程中创建多条消息

**Workflow执行**：
```
Start Node
    │
    ▼
App Node (分析供应商)
    │
    ▼
HITL Node (审批) → 创建SteeringTask 1 (审批消息)
    │
    ▼
HITL Node (分配) → 创建SteeringTask 2 (分配消息)
    │
    ▼
HITL Node (更新) → 创建SteeringTask 3 (更新消息)
    │
    ▼
End Node
```

**每条消息都关联到同一个workflow**：
```json
{
  "steering_task_1": {
    "task_type": "request_for_approval",
    "workflow_task_id": 123,  // 同一个workflow
    "workflow_node_id": "hitl-1"
  },
  "steering_task_2": {
    "task_type": "assign",
    "workflow_task_id": 123,  // 同一个workflow
    "workflow_node_id": "hitl-2"
  },
  "steering_task_3": {
    "task_type": "request_for_update",
    "workflow_task_id": 123,  // 同一个workflow
    "workflow_node_id": "hitl-3"
  }
}
```

#### Step 2: 用户点击消息，打开详情页

**消息详情页结构**：
```
┌─────────────────────────────┐
│  任务详情                    │
├─────────────────────────────┤
│  [基本信息] [AI助手]        │  ← 两个标签页
├─────────────────────────────┤
│                              │
│  基本信息标签页：            │
│  - 任务标题                  │
│  - 任务描述                  │
│  - 任务状态                  │
│  - 审批/更新表单              │
│                              │
│  AI助手标签页：              │
│  - 聊天界面                  │
│  - 显示workflow执行上下文    │
│  - 可以提问关于workflow的问题│
│                              │
└─────────────────────────────┘
```

#### Step 3: AI助手加载Workflow上下文

**上下文加载逻辑**：
```python
# 伪代码
def load_agent_context(steering_task_id):
    steering_task = get_steering_task(steering_task_id)
    workflow_task_id = steering_task.workflow_task_id
    
    # 获取workflow执行上下文
    workflow_context = get_workflow_execution_context(workflow_task_id)
    
    # 获取所有相关消息（相同workflow的消息）
    related_messages = get_steering_tasks_by_workflow(workflow_task_id)
    
    # 构建AI助手上下文
    agent_context = {
        "workflow_id": workflow_task_id,
        "workflow_status": workflow_context.status,
        "node_results": workflow_context.node_results,  # 所有节点的执行结果
        "related_messages": related_messages,  # 相关消息列表
        "current_message": steering_task  # 当前消息
    }
    
    return agent_context
```

**上下文内容示例**：
```json
{
  "workflow_id": 123,
  "workflow_status": "running",
  "node_results": {
    "app-1": {
      "status": "completed",
      "output": {
        "supplier_name": "供应商A",
        "supplier_score": 85,
        "supplier_status": "active"
      }
    },
    "condition-1": {
      "status": "completed",
      "output": {
        "condition_result": true,
        "business_meaning": "供应商符合条件"
      }
    },
    "hitl-1": {
      "status": "pending",
      "output": {
        "waiting_for_human_input": true,
        "task_type": "approval"
      }
    }
  },
  "related_messages": [
    {
      "id": 1,
      "task_type": "request_for_approval",
      "title": "供应商审核审批"
    },
    {
      "id": 2,
      "task_type": "assign",
      "title": "分配任务给采购经理"
    }
  ],
  "current_message": {
    "id": 1,
    "task_type": "request_for_approval"
  }
}
```

#### Step 4: 用户在AI助手中提问

**用户提问示例**：
```
用户："这个供应商的评分是多少？"

AI助手（基于上下文回答）：
"根据workflow执行结果，供应商A的评分是85分。
这个评分来自App Node的分析结果。
当前workflow正在等待您的审批决策。"
```

**用户继续提问**：
```
用户："这个workflow还有哪些消息？"

AI助手：
"这个workflow共有3条消息：
1. 供应商审核审批（当前消息）- 待处理
2. 分配任务给采购经理 - 已完成
3. 请求更新证书信息 - 待处理

您可以在消息中心查看所有相关消息。"
```

#### Step 5: 相同Workflow的消息共享上下文

**归集逻辑**：
```python
# 伪代码
def get_agent_context_for_message(steering_task_id):
    steering_task = get_steering_task(steering_task_id)
    workflow_task_id = steering_task.workflow_task_id
    
    # 查找或创建agent上下文
    agent_context = find_or_create_agent_context(
        context_type="workflow",
        resource_id=workflow_task_id
    )
    
    # 所有相同workflow的消息都使用同一个agent上下文
    return agent_context
```

**效果**：
- 用户点击消息1的AI助手，看到workflow执行上下文
- 用户点击消息2的AI助手，看到**相同的**workflow执行上下文
- 用户点击消息3的AI助手，看到**相同的**workflow执行上下文
- 所有消息的AI助手都知道整个workflow的执行情况

### 💡 关键点

1. **每个消息都有独立的AI助手**：
   - 不是所有消息共享一个AI助手
   - 每个消息都有自己的AI助手实例
   - 但相同workflow的消息，AI助手知道相同的上下文

2. **上下文包含什么**：
   - Workflow执行状态
   - 所有节点的执行结果（App Node的分析结果、Condition Node的判断结果等）
   - 相关消息列表
   - 当前消息信息

3. **AI助手可以做什么**：
   - 回答关于workflow执行的问题
   - 解释节点执行结果
   - 查看相关消息
   - 提供workflow执行建议

4. **技术实现**：
   - 使用LangGraph的Chat Agent
   - 每个agent上下文有独立的`thread_id`
   - 相同workflow的消息，agent上下文共享workflow执行数据

---

## 总结

### 场景1：审计流程
- **核心**：使用HITL Node的三种任务类型（approval/assign/request_for_update）
- **流程**：发现问题 → AI分析 → 管理层决策 → 分配任务 → 更新信息 → 上级检查
- **6大Node**：Start → App → Condition → HITL(approval) → HITL(assign) → HITL(request_for_update) → HITL(approval) → Tool → End

### 场景2：AI生成Workflow
- **核心**：Chat Agent理解需求，搜索或生成Workflow，给用户编辑链接
- **流程**：用户聊天 → AI理解 → 搜索市场 → 生成Workflow → 用户编辑 → 保存
- **6大Node**：生成的Workflow包含完整的6大Node结构

### 场景3：消息中心Agent上下文
- **核心**：每个消息有独立的AI助手，相同workflow的消息共享上下文
- **流程**：Workflow执行 → 创建多条消息 → 用户点击消息 → AI助手加载上下文 → 用户提问
- **6大Node**：AI助手可以查看所有Node的执行结果，回答相关问题

---

## 常见问题

### Q1: 场景2中，Chat Agent生成的Workflow是立即执行的吗？
**A**: 不是。Chat Agent只是生成Workflow结构，用户需要：
1. 点击链接进入编辑界面
2. 调整和配置Workflow
3. 保存为模板
4. 然后才能执行

### Q2: 场景3中，如果用户在不同的消息中提问，AI助手会记住之前的对话吗？
**A**: 会。因为相同workflow的消息共享同一个agent上下文，所以：
- 用户在消息1的AI助手中提问，对话记录保存在agent上下文中
- 用户在消息2的AI助手中提问，可以看到之前在消息1中的对话记录

### Q3: 场景1中，如果采购经理没有完成任务怎么办？
**A**: 可以配置超时机制：
- 在HITL Node的assign配置中设置`dueDate`
- 如果超时，workflow可以：
  - 自动失败
  - 发送提醒
  - 重新分配任务

---

**文档版本**: v1.0  
**最后更新**: 2024-01-15  
**维护者**: Foundation Team




