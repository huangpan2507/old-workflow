# Workflow模板保存与多实例运行实现计划

## 一、核心约束明确

### 模板使用的基本原则

**✅ 可以修改的内容**：
- Node节点内的配置参数（如提示词、要求等）
- App Node的文件上传（重新上传不同的文件）
- App Node的URL输入（修改URL地址）
- 输入数据（input_data）

**❌ 不能修改的内容**：
- 节点数量（不能增删节点）
- 节点之间的连线（edges）
- 节点类型（不能改变节点类型）
- 流程结构（不能改变workflow的执行流程）

### 设计理念

**模板 = 结构固定 + 配置可覆盖**

- **结构固定**：nodes和edges在模板中定义，使用模板时不能修改
- **配置可覆盖**：节点内的配置参数可以通过`app_node_configs`覆盖

---

## 二、依赖关系结论（更新）

### 核心结论：**部分依赖，可以分阶段实现**

**必须依赖**：
- ✅ **Workflow模板保存功能**（1.15前必须完成）
  - 多实例运行需要基于模板创建实例
  - 必须先有模板保存机制（nodes、edges、配置持久化）
  - **关键约束**：模板保存的是结构（nodes + edges），配置可以覆盖

**可选依赖**：
- ⚠️ **Workflow审批发布功能**（1.25完成，不影响1.15交付）
  - 个人使用场景：不需要（创建者可以使用自己的draft模板）
  - 团队共享场景：需要（需要published模板才能被其他用户使用）

---

## 三、权限控制策略（已确认）

### Draft模板权限
- ✅ 创建者只能看到自己创建的draft模板
- ✅ 创建者可以使用自己的draft模板创建实例
- ✅ 其他用户看不到draft模板

### Published模板权限
- ✅ 所有用户都可以看到published模板
- ✅ 所有用户都可以使用published模板创建实例

---

## 四、使用场景说明

### 典型场景：HR批量审核简历

**步骤1：创建并保存模板**
```
开发者操作：
1. 在Workbench搭建workflow：
   Start → App Node（简历分析）→ Condition Node → HITL Node → End
2. 配置App Node：
   - 选择"简历分析"Application
   - 设置默认提示词："分析简历，提取关键信息"
   - 设置输入模式：File Upload（但此时不上传文件）
3. 保存为模板："简历审核流程"（draft状态）
```

**步骤2：使用模板创建多个实例**
```
使用者操作（每天审核10份简历）：

实例1：审核张三的简历
- 选择模板："简历审核流程"
- 系统加载模板的nodes和edges（结构固定，不能修改）
- 配置覆盖：
  - App Node：上传文件 zhangsan_resume.pdf
  - 可选：修改提示词
- 点击"执行"

实例2：审核李四的简历
- 选择模板："简历审核流程"
- 系统加载模板的nodes和edges（结构固定，不能修改）
- 配置覆盖：
  - App Node：上传文件 lisi_resume.pdf
- 点击"执行"

...（实例3-10类似）
```

**关键点**：
- ✅ 10个实例使用相同的nodes和edges结构
- ✅ 每个实例使用不同的文件（通过app_node_configs覆盖）
- ✅ 不能修改节点数量、连线、流程结构
- ✅ 可以修改App Node的配置参数（文件、提示词等）

详细使用场景请参考：[workflow-template-usage-scenarios.md](workflow-template-usage-scenarios.md)

---

## 五、向后兼容性分析

### 问题：如果1.15先实现基于draft的多实例，1.25加入审批后是否需要迁移？

**答案：不需要迁移**

**原因分析**：

1. **数据模型兼容**
   ```
   模板状态设计：
   - draft（草稿）：创建者私有，1.15支持
   - submitted（已提交）：1.25新增，不影响draft
   - published（已发布）：1.25新增，不影响draft
   ```

2. **功能兼容**
   - 1.15实现：支持基于draft模板创建实例（结构固定，配置可覆盖）
   - 1.25新增：支持基于published模板创建实例（结构固定，配置可覆盖）
   - **两者可以并存**，通过权限控制区分

3. **API兼容**
   ```python
   # 1.15实现（支持draft）
   POST /api/v1/workflows/templates/{template_id}/instances
   {
     "app_node_configs": {
       "app-1": {
         "inputMode": "file",
         "dagRunId": "dag_run_123"  # 新上传的文件
       }
     }
   }
   # 权限检查：template.created_by == current_user.id
   
   # 1.25增强（同时支持published）
   # API不变，只是权限检查增强：
   #   - draft: template.created_by == current_user.id
   #   - published: 所有用户都可以
   ```

4. **用户体验兼容**
   - 1.15：创建者使用自己的draft模板创建实例（个人场景）
   - 1.25：所有用户使用published模板创建实例（团队场景）
   - **两种场景可以同时存在**，互不干扰

**结论**：设计得当，无需数据迁移，向后兼容。

---

## 六、功能完整性分析

### 问题1：1.15交付的多实例功能是否满足当前业务需求？

**答案：部分满足，取决于使用场景**

**场景分析**：

| 使用场景 | 1.15功能是否满足 | 说明 |
|---------|-----------------|------|
| **个人重复使用** | ✅ 完全满足 | 创建者保存draft模板，多次创建实例处理不同数据 |
| **个人批量处理** | ✅ 完全满足 | 使用同一个draft模板，创建多个实例并行处理 |
| **团队共享使用** | ❌ 不满足 | 需要等待1.25的审批发布功能，让模板变成published状态 |
| **跨部门协作** | ❌ 不满足 | 需要等待1.25的审批发布功能 |

**示例场景**：

**场景A：个人使用（1.15满足）**
```
HR小王创建了"简历审核流程"workflow
→ 保存为draft模板（1.15功能）
→ 每天审核10份简历，使用同一个模板创建10个实例（1.15功能）
→ 每个实例上传不同的简历文件
→ ✅ 完全满足需求
```

**场景B：团队共享（需要1.25）**
```
销售团队创建了"合同审批流程"workflow
→ 保存为draft模板（1.15功能）
→ 提交审批（1.25功能）
→ 审批通过，变成published模板（1.25功能）
→ 所有销售团队成员都可以使用这个模板（1.25功能）
→ ❌ 1.15无法满足，需要等待1.25
```

### 问题2：是否需要等待审批发布功能才能正式使用？

**答案：不需要等待，可以分场景使用**

**使用策略**：

1. **1.15交付后（个人场景）**
   - ✅ 创建者可以立即使用自己的draft模板创建多个实例
   - ✅ 适合个人重复使用、批量处理场景
   - ✅ 功能完整，可以正式使用

2. **1.25交付后（团队场景）**
   - ✅ 团队共享场景可以使用published模板
   - ✅ 跨部门协作场景可以使用published模板
   - ✅ 企业级管控场景可以使用审批发布功能

**结论**：1.15的功能已经可以满足个人使用场景，不需要等待1.25。团队共享场景需要等待1.25。

---

## 七、实现路径设计

### 阶段1：1.15交付（多实例运行基础能力）

**功能清单**：

1. ✅ **Workflow模板保存（draft状态）**
   - 保存nodes、edges到数据库（结构固定）
   - 保存默认配置（可选，用于参考）
   - 支持模板命名、描述
   - 状态：draft（默认）

2. ✅ **权限控制（draft模板）**
   - 创建者只能看到自己创建的draft模板
   - 创建者可以使用自己的draft模板创建实例
   - 其他用户看不到draft模板

3. ✅ **多实例运行（基于draft模板，结构固定，配置可覆盖）**
   - 从draft模板加载nodes和edges（结构固定，不能修改）
   - 支持配置覆盖（app_node_configs）
   - 支持文件上传覆盖（fileDagRunMap）
   - 每个实例独立追踪（task_id、thread_id）

**技术实现要点**：

```python
# 1. 模板保存API
POST /api/v1/workflows/templates
{
  "name": "简历审核流程",
  "description": "自动审核简历",
  "nodes": [...],  # 结构定义
  "edges": [...],  # 连线定义
  "status": "draft"  # 默认draft
}

# 2. 模板列表API（权限过滤）
GET /api/v1/workflows/templates
# 返回：当前用户创建的draft模板 + 所有published模板（1.25后）

# 3. 创建实例API（结构固定，配置可覆盖）
POST /api/v1/workflows/templates/{template_id}/instances
{
  "app_node_configs": {
    "app-1": {
      "inputMode": "file",
      "dagRunId": "dag_run_123",  # 新上传的文件
      "requirement": "重点分析工作经验和技能匹配度"  # 覆盖默认提示词
    }
  },
  "input_data": {
    "candidate_name": "张三"
  }
}
# 权限检查：
#   - draft模板：template.created_by == current_user.id
#   - published模板：所有用户都可以（1.25后）

# 4. 执行逻辑
def create_instance_from_template(template_id, app_node_configs, input_data):
    # 1. 加载模板（结构固定）
    template = get_template(template_id)
    nodes = template.nodes  # 不能修改
    edges = template.edges   # 不能修改
    
    # 2. 合并配置覆盖（不改变结构）
    for node in nodes:
        if node['id'] in app_node_configs:
            # 覆盖节点配置，但不改变节点结构
            node_config = app_node_configs[node['id']]
            # 合并到node的data中
            node['data'].update(node_config)
    
    # 3. 创建执行实例
    task_id = create_workflow_task(nodes, edges, input_data)
    return task_id
```

### 阶段2：1.25交付（企业级管控）

**功能清单**：

1. ✅ **Workflow审批发布流程**
   - 状态流转：draft → submitted → published
   - 提交审批：创建者提交draft模板审批
   - 审批流程：超级管理员审批/拒绝
   - 权限控制：published模板所有用户可见

2. ✅ **权限控制增强（published模板）**
   - 所有用户都可以看到published模板
   - 所有用户都可以使用published模板创建实例（结构固定，配置可覆盖）

3. ✅ **多实例运行增强（同时支持published模板）**
   - 保持对draft模板的支持（向后兼容）
   - 新增对published模板的支持
   - 通过权限控制区分访问权限

**技术实现要点**：

```python
# 提交审批API
POST /api/v1/workflows/templates/{template_id}/submit
# 状态：draft → submitted

# 审批API（超级管理员）
POST /api/v1/workflows/templates/{template_id}/approve
POST /api/v1/workflows/templates/{template_id}/reject
# 状态：submitted → published / rejected

# 模板列表API（权限过滤增强）
GET /api/v1/workflows/templates
# 返回：
#   - 当前用户创建的draft模板
#   - 所有published模板（新增）
```

---

## 八、时间安排建议

### 1.15交付（8天：1月7日-1月15日）

```
Day 1-3: Workflow模板保存功能
├─ 数据模型设计（WorkflowTemplate表）
│  ├─ nodes: JSONB（结构定义，不能修改）
│  ├─ edges: JSONB（连线定义，不能修改）
│  └─ default_configs: JSONB（默认配置，可选）
├─ 保存API实现
├─ 加载API实现
└─ 权限控制（draft模板仅创建者可见）

Day 4-6: 多实例运行功能（结构固定，配置可覆盖）
├─ 基于模板创建实例API
│  ├─ 加载模板nodes和edges（结构固定）
│  ├─ 配置覆盖机制（app_node_configs）
│  └─ 文件上传覆盖（fileDagRunMap）
├─ 实例独立追踪（task_id、thread_id）
└─ UI交互（只读模式显示模板结构）

Day 7-8: 测试和优化
├─ 单元测试
├─ 集成测试
└─ 性能优化
```

### 1.25交付（10天：1月16日-1月25日）

```
Day 1-3: Workflow审批发布流程
├─ 状态流转实现（draft → submitted → published）
├─ 提交审批API
├─ 审批API（超级管理员）
└─ 拒绝和重新提交机制

Day 4-5: 权限控制增强
├─ Published模板权限控制
├─ 模板列表API权限过滤增强
└─ 多实例运行权限检查增强

Day 6-8: 其他功能（Condition Node增强等）
Day 9-10: 测试和优化
```

---

## 九、关键设计决策

### 1. 模板状态设计

```python
class WorkflowTemplateStatus(str, Enum):
    DRAFT = "draft"          # 草稿：创建者私有
    SUBMITTED = "submitted"  # 已提交：等待审批
    PUBLISHED = "published" # 已发布：所有用户可用
    REJECTED = "rejected"   # 已拒绝：创建者可重新提交
```

### 2. 模板数据结构

```python
class WorkflowTemplate(SQLModel, table=True):
    id: int
    name: str
    description: Optional[str]
    status: str  # draft/submitted/published/rejected
    
    # 结构定义（固定，不能修改）
    nodes: List[Dict[str, Any]]  # JSONB
    edges: List[Dict[str, Any]]  # JSONB
    
    # 默认配置（可选，用于参考）
    default_configs: Optional[Dict[str, Any]]  # JSONB
    
    created_by: uuid.UUID
    create_time: datetime
    update_time: datetime
```

### 3. 权限控制设计

```python
def can_access_template(user: User, template: WorkflowTemplate) -> bool:
    """检查用户是否可以访问模板"""
    # 超级管理员可以访问所有模板
    if user.is_superuser:
        return True
    
    # Published模板所有用户都可以访问
    if template.status == "published":
        return True
    
    # Draft模板只有创建者可以访问
    if template.status == "draft":
        return template.created_by == user.id
    
    # Submitted/Rejected模板只有创建者和审批人可以访问
    if template.status in ["submitted", "rejected"]:
        return template.created_by == user.id or user.is_superuser
    
    return False
```

### 4. 多实例运行权限检查

```python
def can_create_instance(user: User, template: WorkflowTemplate) -> bool:
    """检查用户是否可以基于模板创建实例"""
    # 检查模板访问权限
    if not can_access_template(user, template):
        return False
    
    # Draft模板只有创建者可以创建实例
    if template.status == "draft":
        return template.created_by == user.id
    
    # Published模板所有用户都可以创建实例
    if template.status == "published":
        return True
    
    return False
```

### 5. 配置覆盖机制

```python
def merge_template_with_configs(
    template: WorkflowTemplate,
    app_node_configs: Dict[str, Any]
) -> Tuple[List[Dict], List[Dict]]:
    """
    合并模板结构和配置覆盖
    
    关键约束：
    - nodes和edges结构固定，不能修改
    - 只能覆盖节点内的配置参数
    """
    # 1. 复制模板结构（不能修改）
    nodes = copy.deepcopy(template.nodes)
    edges = copy.deepcopy(template.edges)
    
    # 2. 合并配置覆盖（不改变结构）
    for node in nodes:
        node_id = node.get('id')
        if node_id in app_node_configs:
            # 覆盖节点配置，但不改变节点结构
            node_config = app_node_configs[node_id]
            # 合并到node的data中
            if 'data' not in node:
                node['data'] = {}
            node['data'].update(node_config)
    
    return nodes, edges
```

---

## 十、UI交互设计

### 场景：使用模板创建实例

**界面流程**：

1. **选择模板**
   - 显示模板列表（draft模板：仅创建者可见；published模板：所有用户可见）
   - 点击"使用模板"按钮

2. **加载模板结构**
   - 显示workflow画布，加载模板的nodes和edges
   - **关键**：画布处于"只读模式"
     - ❌ 不能拖拽节点
     - ❌ 不能增删节点
     - ❌ 不能修改连线
     - ✅ 可以点击节点配置参数
   - 显示提示："这是模板结构，不能修改。您可以配置节点参数。"

3. **配置节点参数**
   - 点击App Node，打开配置面板
   - **可以修改**：
     - 文件上传（重新上传文件）
     - 提示词（修改requirement）
     - URL输入（如果支持）
   - **不能修改**：
     - 节点类型
     - 节点ID
     - 节点位置（可选，不影响功能）

4. **执行实例**
   - 点击"执行"按钮
   - 系统创建新实例，使用模板的nodes和edges，但使用新的配置

---

## 十一、风险评估

### 低风险 ✅
- **向后兼容**：设计得当，无需数据迁移
- **功能完整性**：个人使用场景完全满足
- **技术实现**：基于现有架构，技术风险低
- **约束明确**：结构固定、配置可覆盖的设计清晰

### 中风险 ⚠️
- **时间紧迫**：1.15交付时间紧张，需要合理安排
- **团队共享场景**：需要等待1.25才能满足
- **UI交互**：只读模式的设计需要清晰的用户引导

### 建议
- ✅ 1.15先交付个人使用场景，满足大部分需求
- ✅ 1.25补充团队共享场景，完善企业级功能
- ✅ 设计时考虑向后兼容，避免后续重构
- ✅ UI设计时明确区分"结构固定"和"配置可覆盖"

---

## 十二、总结

### 依赖关系结论
- ✅ **必须依赖**：Workflow模板保存功能（1.15前必须完成）
- ⚠️ **可选依赖**：Workflow审批发布功能（1.25完成，不影响1.15交付）

### 核心约束
- ✅ **结构固定**：nodes和edges在模板中定义，使用模板时不能修改
- ✅ **配置可覆盖**：节点内的配置参数可以通过app_node_configs覆盖

### 向后兼容性
- ✅ **无需迁移**：设计得当，draft和published模板可以并存

### 功能完整性
- ✅ **个人场景**：1.15功能完全满足，可以正式使用
- ⚠️ **团队场景**：需要等待1.25的审批发布功能

### 实现建议
- ✅ 1.15先实现draft模板的多实例运行（个人场景，结构固定，配置可覆盖）
- ✅ 1.25补充published模板的多实例运行（团队场景，结构固定，配置可覆盖）
- ✅ 设计时考虑权限控制和向后兼容
- ✅ UI设计时明确区分"结构固定"和"配置可覆盖"



