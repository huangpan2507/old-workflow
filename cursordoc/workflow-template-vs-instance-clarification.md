# Workflow模板与实例的关系和区别详解

## 一、外在表现（UI/用户体验层面）

### 1.1 模板（Template）- 用户看到什么

**模板是什么**：
- 模板是**可复用的工作流结构定义**
- 类似于"蓝图"或"配方"
- 定义了"做什么"，但不包含"具体数据"

**用户在UI上看到**：

```
┌─────────────────────────────────────────────────────────────┐
│  Workflow模板库                                    [+ 新建模板] │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ 📋 简历审核流程                                        │  │
│  │                                                        │  │
│  │ 描述: 自动分析简历，判断是否符合要求，需要人工审批    │  │
│  │ 创建者: 小王                                           │  │
│  │ 创建时间: 2024-01-10                                  │  │
│  │ 使用次数: 15次                                        │  │
│  │                                                        │  │
│  │ [使用模板] [编辑] [删除]                              │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ 📋 合同审批流程                                        │  │
│  │                                                        │  │
│  │ 描述: 分析合同条款，提取关键信息，需要财务审批        │  │
│  │ 创建者: 小李                                           │  │
│  │ 创建时间: 2024-01-08                                  │  │
│  │ 使用次数: 8次                                         │  │
│  │                                                        │  │
│  │ [使用模板] [编辑] [删除]                              │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

**模板的特点**：
- ✅ **结构固定**：定义了节点类型、连接关系、默认配置
- ✅ **可复用**：可以被多次使用，创建多个实例
- ✅ **可编辑**：创建者可以修改模板（如果允许）
- ❌ **不包含具体数据**：没有上传的文件、没有执行结果

---

### 1.2 实例（Instance）- 用户看到什么

**实例是什么**：
- 实例是**基于模板创建的一次具体执行**
- 类似于"按照蓝图建造的房子"
- 包含了"具体的数据"和"执行结果"

**用户在UI上看到**：

```
┌─────────────────────────────────────────────────────────────┐
│  Workflow模板: 简历审核流程                                  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 实例 #1: 张三简历                    ✅ 已完成        │  │
│  │ 创建时间: 2024-01-15 10:30:00                         │  │
│  │ 执行时间: 8分钟                                        │  │
│  │                                                        │  │
│  │ 配置信息:                                             │  │
│  │ - 文件: zhangsan_resume.pdf                           │  │
│  │ - 提示词: 重点分析工作经验和技能匹配度                │  │
│  │                                                        │  │
│  │ 执行结果:                                             │  │
│  │ - App Node: 分析完成，评分85分                       │  │
│  │ - Condition Node: 通过 (score > 80)                  │  │
│  │ - HITL Node: 已审批通过（审批人：HR经理）            │  │
│  │                                                        │  │
│  │ [查看详情] [重新执行] [下载报告]                     │  │
│  ├───────────────────────────────────────────────────────┤  │
│  │ 实例 #2: 李四简历                    🟢 运行中         │  │
│  │ 创建时间: 2024-01-15 10:35:00                         │  │
│  │ 执行时间: 3分钟（进行中）                              │  │
│  │                                                        │  │
│  │ 配置信息:                                             │  │
│  │ - 文件: lisi_resume.pdf                               │  │
│  │ - 提示词: 使用模板默认提示词                           │  │
│  │                                                        │  │
│  │ 当前状态:                                             │  │
│  │ - 当前节点: App Node (简历分析)                       │  │
│  │ - 进度: 60%                                           │  │
│  │                                                        │  │
│  │ [查看详情] [停止]                                     │  │
│  ├───────────────────────────────────────────────────────┤  │
│  │ 实例 #3: 王五简历                    ⏸️ 等待审批       │  │
│  │ 创建时间: 2024-01-15 10:20:00                         │  │
│  │ 执行时间: 3分钟（已暂停）                              │  │
│  │                                                        │  │
│  │ 配置信息:                                             │  │
│  │ - 文件: wangwu_resume.pdf                             │  │
│  │ - 提示词: 使用模板默认提示词                           │  │
│  │                                                        │  │
│  │ 等待审批:                                             │  │
│  │ - 当前节点: HITL Node (人工审批)                      │  │
│  │ - 审批人: HR经理                                       │  │
│  │ - 等待时间: 15分钟                                    │  │
│  │                                                        │  │
│  │ [查看详情] [继续执行]                                 │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

**实例的特点**：
- ✅ **包含具体数据**：上传的文件、输入的参数、配置覆盖
- ✅ **有执行状态**：pending/running/completed/failed
- ✅ **有执行结果**：每个节点的输出、最终结果
- ✅ **独立存在**：即使模板被删除或修改，实例仍然存在
- ❌ **不可修改结构**：不能改变节点类型、连接关系

---

### 1.3 用户操作流程对比

#### 创建/编辑模板（开发者视角）

```
1. 进入Workbench
   ↓
2. 拖拽节点，搭建Workflow结构
   - Start Node
   - App Node（选择Application，设置默认提示词）
   - Condition Node（设置默认条件）
   - HITL Node（设置默认审批人）
   - End Node
   ↓
3. 连接节点，形成流程
   ↓
4. 点击"保存模板"
   - 输入模板名称："简历审核流程"
   - 输入描述："自动分析简历，判断是否符合要求"
   - 设置是否允许修改：false（默认）
   ↓
5. 模板保存成功
   - 模板出现在"模板库"中
   - 可以被其他用户使用
```

**关键点**：
- 模板只定义**结构**和**默认配置**
- 不包含**具体数据**（文件、参数等）

---

#### 使用模板创建实例（使用者视角）

```
1. 进入模板库，选择"简历审核流程"
   ↓
2. 点击"使用模板"
   - 系统加载模板结构到画布（只读模式）
   - 显示提示："这是模板结构，不能修改。您可以配置节点参数。"
   ↓
3. 配置节点参数（可选）
   - 点击 App Node
   - 上传文件：zhangsan_resume.pdf
   - 修改提示词："重点分析工作经验和技能匹配度"（可选）
   ↓
4. 点击"执行"
   - 系统创建实例
   - 合并模板结构 + 配置覆盖
   - 开始执行
   ↓
5. 查看执行结果
   - 实例出现在"我的实例"列表中
   - 可以查看每个节点的执行结果
   - 可以下载报告
```

**关键点**：
- 实例基于模板创建，但包含**具体数据**
- 可以**覆盖配置**，但不影响模板
- 每个实例**独立执行**，互不干扰

---

## 二、内在存储（数据结构层面）

### 2.1 模板存储结构

**数据库表**：`approval_workflow_templates`

```sql
CREATE TABLE approval_workflow_templates (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,                    -- 模板名称
    description TEXT,                              -- 模板描述
    resource_type VARCHAR(50) DEFAULT 'workbench_workflow',  -- 资源类型
    is_editable BOOLEAN DEFAULT FALSE,            -- 是否允许修改结构（新增）
    
    -- 核心：工作流结构定义
    workflow_nodes JSONB,                          -- 节点定义数组
    workflow_edges JSONB,                          -- 边定义数组
    
    -- 元数据
    created_by UUID REFERENCES users(id),
    create_time TIMESTAMP DEFAULT NOW(),
    update_time TIMESTAMP DEFAULT NOW(),
    is_active BOOLEAN DEFAULT TRUE,
    yn BOOLEAN DEFAULT TRUE
);
```

**workflow_nodes 示例数据**：
```json
[
  {
    "id": "start-1",
    "type": "start",
    "data": {},
    "position": {"x": 100, "y": 100}
  },
  {
    "id": "app-1",
    "type": "app",
    "data": {
      "appNodeId": 123,
      "executionConfig": {
        "params": {
          "requirement": "分析简历，提取关键信息"  // 默认提示词
        }
      }
    },
    "position": {"x": 300, "y": 100}
  },
  {
    "id": "condition-1",
    "type": "condition",
    "data": {
      "condition": "score > 80"  // 默认条件
    },
    "position": {"x": 500, "y": 100}
  },
  {
    "id": "hitl-1",
    "type": "humanInTheLoop",
    "data": {
      "approverJobTitleIds": [5],  // 默认审批人职位ID
      "approvalLogic": "any"
    },
    "position": {"x": 700, "y": 100}
  },
  {
    "id": "end-1",
    "type": "end",
    "data": {},
    "position": {"x": 900, "y": 100}
  }
]
```

**workflow_edges 示例数据**：
```json
[
  {
    "id": "e1",
    "source": "start-1",
    "target": "app-1",
    "sourceHandle": null,
    "targetHandle": null
  },
  {
    "id": "e2",
    "source": "app-1",
    "target": "condition-1",
    "sourceHandle": null,
    "targetHandle": null
  },
  {
    "id": "e3",
    "source": "condition-1",
    "target": "hitl-1",
    "sourceHandle": "true",
    "targetHandle": null,
    "condition": "true"
  },
  {
    "id": "e4",
    "source": "condition-1",
    "target": "end-1",
    "sourceHandle": "false",
    "targetHandle": null,
    "condition": "false"
  },
  {
    "id": "e5",
    "source": "hitl-1",
    "target": "end-1",
    "sourceHandle": null,
    "targetHandle": null
  }
]
```

**模板存储的关键点**：
- ✅ **只存储结构**：nodes和edges定义
- ✅ **存储默认配置**：节点内的默认参数（提示词、条件、审批人等）
- ❌ **不存储具体数据**：没有文件、没有执行结果
- ❌ **不存储执行状态**：模板本身不执行

---

### 2.2 实例存储结构

**数据库表1**：`workflow_instances`（实例元数据）

```sql
CREATE TABLE workflow_instances (
    id SERIAL PRIMARY KEY,
    template_id INTEGER REFERENCES approval_workflow_templates(id),  -- 关联模板
    
    -- 执行状态
    status VARCHAR(50) DEFAULT 'pending',  -- pending/running/completed/failed/paused
    current_node_id VARCHAR(255),          -- 当前执行节点ID
    
    -- 执行结果（最终结果）
    execution_result JSONB,                -- 最终执行结果和摘要
    
    -- 元数据
    created_by UUID REFERENCES users(id),
    create_time TIMESTAMP DEFAULT NOW(),
    update_time TIMESTAMP DEFAULT NOW()
);
```

**execution_result 示例数据**（执行完成后）：
```json
{
  "final_output": {
    "workflow_result": "success",
    "summary": {
      "total_nodes": 5,
      "completed_nodes": 5,
      "execution_time": "8m30s"
    }
  },
  "instance_config": {
    "app_node_configs": {
      "app-1": {
        "inputMode": "file",
        "dagRunId": "dag_run_123",  // 具体上传的文件
        "requirement": "重点分析工作经验和技能匹配度"  // 覆盖的提示词
      }
    },
    "input_data": {}
  }
}
```

**数据库表2**：`workflow_execution_tasks`（执行过程详情）

```sql
CREATE TABLE workflow_execution_tasks (
    id SERIAL PRIMARY KEY,
    task_id VARCHAR(255) UNIQUE NOT NULL,  -- UUID
    instance_id INTEGER REFERENCES workflow_instances(id),  -- 关联实例（新增）
    
    -- 工作流定义（从模板加载 + 配置覆盖）
    nodes JSONB,                            -- 节点定义（从模板加载）
    edges JSONB,                            -- 边定义（从模板加载）
    input_data JSONB,                       -- 初始输入数据
    app_node_configs JSONB,                -- 配置覆盖
    
    -- 执行状态（实时更新）
    status VARCHAR(50) DEFAULT 'pending',
    current_node_id VARCHAR(255),
    node_results JSONB,                     -- 每个节点的执行结果（实时更新）
    
    -- LangGraph相关
    thread_id VARCHAR(255),
    checkpoint_id VARCHAR(255),
    
    -- 元数据
    created_by UUID REFERENCES users(id),
    create_time TIMESTAMP DEFAULT NOW(),
    update_time TIMESTAMP DEFAULT NOW(),
    finished_at TIMESTAMP
);
```

**nodes 示例数据**（从模板加载，但可能被配置覆盖）：
```json
[
  {
    "id": "start-1",
    "type": "start",
    "data": {}
  },
  {
    "id": "app-1",
    "type": "app",
    "data": {
      "appNodeId": 123,
      "executionConfig": {
        "params": {
          "requirement": "重点分析工作经验和技能匹配度"  // 被覆盖的提示词
        }
      },
      // 配置覆盖后的数据
      "inputMode": "file",
      "dagRunId": "dag_run_123"  // 具体上传的文件
    }
  },
  // ... 其他节点
]
```

**app_node_configs 示例数据**（用户配置覆盖）：
```json
{
  "app-1": {
    "inputMode": "file",
    "dagRunId": "dag_run_123",  // 新上传的文件
    "requirement": "重点分析工作经验和技能匹配度"  // 覆盖的提示词
  }
}
```

**node_results 示例数据**（执行过程中实时更新）：
```json
{
  "start-1": {
    "status": "completed",
    "output": {
      "input_data": {}
    },
    "timestamp": "2024-01-15T10:30:00Z"
  },
  "app-1": {
    "status": "completed",
    "output": {
      "content": "AI分析结果：张三，5年工作经验，技能匹配度85分...",
      "structured_data": {
        "score": 85,
        "experience_years": 5
      }
    },
    "timestamp": "2024-01-15T10:35:00Z"
  },
  "condition-1": {
    "status": "completed",
    "output": {
      "condition_result": true,
      "business_meaning": "评分大于80分，符合条件"
    },
    "timestamp": "2024-01-15T10:36:00Z"
  },
  "hitl-1": {
    "status": "completed",
    "output": {
      "decision": "approve",
      "reviewer_info": {
        "reviewer_id": "user123",
        "reviewer_name": "HR经理",
        "review_time": "2024-01-15T10:40:00Z"
      }
    },
    "timestamp": "2024-01-15T10:40:00Z"
  },
  "end-1": {
    "status": "completed",
    "output": {
      "workflow_result": "success"
    },
    "timestamp": "2024-01-15T10:40:30Z"
  }
}
```

**实例存储的关键点**：
- ✅ **关联模板**：通过 `template_id` 关联到模板
- ✅ **存储配置覆盖**：用户上传的文件、修改的参数
- ✅ **存储执行状态**：当前状态、当前节点
- ✅ **存储执行结果**：每个节点的输出、最终结果
- ✅ **独立存在**：即使模板被删除，实例数据仍然存在

---

## 三、模板与实例的关系

### 3.1 关系图

```
┌─────────────────────────────────────────────────────────────┐
│  模板（Template）                                            │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ ApprovalWorkflowTemplate                             │  │
│  │ ├─ id: 1                                             │  │
│  │ ├─ name: "简历审核流程"                              │  │
│  │ ├─ workflow_nodes: [...]  ←─── 结构定义              │  │
│  │ ├─ workflow_edges: [...]  ←─── 连接定义              │  │
│  │ └─ is_editable: false                                │  │
│  └───────────────────────────────────────────────────────┘  │
│                          │                                    │
│                          │ template_id                        │
│                          │                                    │
│         ┌────────────────┼────────────────┐                 │
│         │                │                │                 │
│         ▼                ▼                ▼                 │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐             │
│  │ 实例 #1  │    │ 实例 #2  │    │ 实例 #3  │             │
│  │          │    │          │    │          │             │
│  │ 张三简历 │    │ 李四简历 │    │ 王五简历 │             │
│  │          │    │          │    │          │             │
│  │ 已完成   │    │ 运行中   │    │ 等待审批 │             │
│  └──────────┘    └──────────┘    └──────────┘             │
│         │                │                │                 │
│         └────────────────┼────────────────┘                 │
│                          │                                    │
│                          ▼                                    │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ WorkflowExecutionTask (执行过程)                      │  │
│  │ ├─ instance_id: 关联到实例                            │  │
│  │ ├─ nodes: 从模板加载 + 配置覆盖                       │  │
│  │ ├─ edges: 从模板加载                                  │  │
│  │ ├─ app_node_configs: 用户配置覆盖                     │  │
│  │ └─ node_results: 执行结果（实时更新）                 │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 数据流向

```
创建模板
    ↓
ApprovalWorkflowTemplate
├─ workflow_nodes: [节点定义]
└─ workflow_edges: [边定义]
    ↓
用户使用模板创建实例
    ↓
WorkflowInstance (实例元数据)
├─ template_id: 1  ←─── 关联模板
├─ status: "pending"
└─ execution_result: null
    ↓
WorkflowExecutionTask (执行过程)
├─ instance_id: 100  ←─── 关联实例
├─ nodes: 从模板加载 + 配置覆盖
├─ edges: 从模板加载
├─ app_node_configs: 用户配置
└─ node_results: {}  ←─── 实时更新
    ↓
执行完成
    ↓
WorkflowInstance
└─ execution_result: {最终结果}  ←─── 更新
    ↓
WorkflowExecutionTask
└─ node_results: {所有节点结果}  ←─── 更新
```

---

## 四、关键区别总结

### 4.1 存储内容对比

| 对比项 | 模板（Template） | 实例（Instance） |
|--------|-----------------|-----------------|
| **节点定义** | ✅ 存储（workflow_nodes） | ❌ 不存储（从模板加载） |
| **边定义** | ✅ 存储（workflow_edges） | ❌ 不存储（从模板加载） |
| **默认配置** | ✅ 存储（节点内的默认参数） | ❌ 不存储（从模板加载） |
| **配置覆盖** | ❌ 不存储 | ✅ 存储（app_node_configs） |
| **具体数据** | ❌ 不存储（文件、参数） | ✅ 存储（上传的文件、输入的参数） |
| **执行状态** | ❌ 不存储 | ✅ 存储（status, current_node_id） |
| **执行结果** | ❌ 不存储 | ✅ 存储（node_results, execution_result） |

### 4.2 生命周期对比

**模板生命周期**：
```
创建 → 保存 → 使用（创建实例）→ 编辑（可选）→ 删除
```

**实例生命周期**：
```
创建（基于模板）→ 执行 → 完成/失败 → 查看结果 → 保留（永久）
```

### 4.3 修改影响对比

**修改模板**：
- ✅ 影响：未来创建的实例会使用新模板
- ❌ 不影响：已创建的实例（实例不存储nodes/edges）

**修改实例**：
- ❌ 不能修改：节点结构、连接关系
- ✅ 可以修改：执行状态（暂停/继续）、查看结果

---

## 五、实际数据示例

### 场景：HR审核3份简历

**模板数据**（1条记录）：
```sql
-- approval_workflow_templates
id=1, name="简历审核流程", 
workflow_nodes='[{"id":"start-1","type":"start",...},{"id":"app-1","type":"app","data":{"appNodeId":123,"executionConfig":{"params":{"requirement":"分析简历，提取关键信息"}}},...},...]',
workflow_edges='[{"source":"start-1","target":"app-1",...},...]'
```

**实例数据**（3条记录）：
```sql
-- workflow_instances
id=100, template_id=1, status="completed", execution_result='{"final_output":{...}}'
id=101, template_id=1, status="running", execution_result=null
id=102, template_id=1, status="paused", execution_result=null
```

**执行任务数据**（3条记录）：
```sql
-- workflow_execution_tasks
id=200, instance_id=100, app_node_configs='{"app-1":{"dagRunId":"dag_run_123"}}', node_results='{"start-1":{...},"app-1":{...},...}'
id=201, instance_id=101, app_node_configs='{"app-1":{"dagRunId":"dag_run_456"}}', node_results='{"start-1":{...},"app-1":{...}}'
id=202, instance_id=102, app_node_configs='{"app-1":{"dagRunId":"dag_run_789"}}', node_results='{"start-1":{...}}'
```

**关键点**：
- 1个模板 → 3个实例
- 每个实例有不同的文件（dagRunId不同）
- 每个实例有独立的执行状态和结果
- 模板修改不影响这3个已创建的实例

---

## 六、总结

### 模板（Template）
- **本质**：可复用的结构定义
- **存储**：nodes、edges、默认配置
- **特点**：静态、可编辑、可复用
- **用途**：定义"做什么"

### 实例（Instance）
- **本质**：基于模板的一次具体执行
- **存储**：配置覆盖、执行状态、执行结果
- **特点**：动态、独立、不可修改结构
- **用途**：记录"做了什么"和"结果如何"

### 关系
- **1对多**：1个模板可以创建多个实例
- **继承**：实例继承模板的结构和默认配置
- **独立**：实例可以覆盖配置，执行结果独立存储
- **隔离**：模板修改不影响已创建的实例



