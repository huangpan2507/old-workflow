# Workflow保存即发布方案（1.15交付版本）

## 一、需求变更说明

### 原计划
- **1.15号**：workflow保存 + 多实例化运行
- **1.25号**：workflow发布、审核流程

### 新计划（调整后）
- **1.15号**：workflow保存即发布 + 多实例化运行
  - ✅ 保存模板后，**直接发布**，其他用户立即可用
  - ❌ **不需要**审核流程（submit/approve）
  - ❌ **不需要**状态流转（draft → submitted → published）

---

## 二、简化后的设计

### 2.1 状态管理简化

**原设计**（类似Application）：
```
draft → submitted → published
  ↓        ↓           ↓
创建    提交审核    审批通过
```

**新设计**（简化）：
```
保存 → published（立即可用）
  ↓
创建即发布
```

**实现方式**：
- 保存模板时，直接设置 `is_active = true`
- 不需要 `status` 字段（或固定为 "published"）
- 不需要 `submitted_at`、`published_by` 等字段

---

### 2.2 数据模型调整

**ApprovalWorkflowTemplate 字段**（后端内部字段，UI不暴露结构开关）：
```python
class ApprovalWorkflowTemplate:
    id: int
    name: str
    description: str | None
    resource_type: str = "workbench_workflow"
    is_editable: bool = False  # 是否允许修改结构（当前固定为False，不在UI中暴露）
    
    # 工作流结构
    workflow_nodes: JSONB
    workflow_edges: JSONB
    
    # 简化：不需要status字段，直接用is_active
    is_active: bool = True  # 保存即激活（发布）
    
    # 元数据
    created_by: uuid.UUID
    create_time: datetime
    update_time: datetime
    yn: bool = True
```

**关键变化**：
- ❌ 删除：`status` 字段（draft/submitted/published）
- ❌ 删除：`submitted_at`、`published_by`、`rejected_reason` 等审核相关字段
- ✅ 保留：`is_active`（用于软删除，true表示可用）
- ✅ 新增：`is_editable`（是否允许修改结构）

---

### 2.3 API设计简化

#### 原设计（需要审核）
```python
# 1. 保存模板（draft状态）
POST /api/v1/workbench/workflows/save-template
→ status = "draft"

# 2. 提交审核
POST /api/v1/workbench/workflows/{template_id}/submit
→ status = "submitted"

# 3. 管理员审批
POST /api/v1/workbench/workflows/{template_id}/approve
→ status = "published"
```

#### 新设计（保存即发布，前端不暴露结构相关选项）
```python
# 1. 保存模板（直接发布）
POST /api/v1/workbench/workflows/save-template
{
  "name": "简历审核流程",
  "description": "...",
  "nodes": [...],
  "edges": [...]
}
→ is_active = true  # 立即可用
→ 其他用户可以看到并使用
```

**API列表**（简化后）：
```python
# 1. 保存模板（创建即发布）
POST /api/v1/workbench/workflows/save-template
→ 创建模板，is_active=true，立即可用

# 2. 获取模板列表
GET /api/v1/workbench/workflow-templates
→ 查询 is_active=true 的模板

# 3. 获取我的模板
GET /api/v1/workbench/workflow-templates?mine=true
→ 查询 created_by=current_user 的模板

# 4. 删除模板（软删除）
DELETE /api/v1/workbench/workflow-templates/{template_id}
→ is_active = false

# 5. 从模板创建实例
POST /api/v1/workbench/workflows/create-instance
{
  "template_id": 1,
  "app_node_configs": {...},
  "input_data": {}
}

# 6. 获取实例列表
GET /api/v1/workbench/workflow-instances?template_id=1
```

---

### 2.4 用户权限简化

**原设计**（需要审核）：
- 创建者：可以创建、编辑draft模板、提交审核
- 管理员：可以审批、拒绝
- 所有用户：只能使用published模板

**新设计**（保存即发布）：
- **创建者**：
  - ✅ 可以创建模板（保存即发布）
  - ✅ 可以编辑自己的模板（如果is_editable=true）
  - ✅ 可以删除自己的模板（软删除）
  - ✅ 可以查看自己创建的所有模板
- **所有用户**：
  - ✅ 可以看到所有已发布的模板（is_active=true）
  - ✅ 可以使用任何模板创建实例
  - ❌ 不能编辑别人的模板

**权限检查**：
```python
# 创建模板：所有登录用户都可以
if not current_user:
    raise HTTPException(403, "需要登录")

# 编辑模板：只有创建者可以
if template.created_by != current_user.id:
    raise HTTPException(403, "只能编辑自己创建的模板")

# 删除模板：只有创建者可以
if template.created_by != current_user.id:
    raise HTTPException(403, "只能删除自己创建的模板")
```

---

## 三、UI交互流程

### 3.1 保存模板流程（简化）

```
用户在Workbench中搭建Workflow
    ↓
点击"保存模板"按钮
    ↓
弹出保存对话框：
  ┌────────────────────────────────────────┐
  │  保存Workflow模板          [×]          │
  ├────────────────────────────────────────┤
  │                                         │
  │  模板名称: [________________]           │
  │                                         │
  │  描述: [________________________________]│
  │       [________________________________]│
  │                                         │
  │  [取消]  [保存并发布]                   │
  └────────────────────────────────────────┘
    ↓
保存成功
    ↓
显示提示："模板已保存并发布，其他用户现在可以使用"
    ↓
模板出现在"模板库"中（所有用户可见）
```

### 3.2 模板库界面（简化）

```
┌─────────────────────────────────────────────────────────────┐
│  Workflow模板库                                    [+ 新建模板] │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  筛选: [全部] [我创建的]                                     │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ 📋 简历审核流程                                        │  │
│  │                                                        │  │
│  │ 描述: 自动分析简历，判断是否符合要求，需要人工审批    │  │
│  │ 创建者: 小王                                           │  │
│  │ 创建时间: 2024-01-15 10:30:00                         │  │
│  │ 使用次数: 15次                                        │  │
│  │                                                        │  │
│  │ [使用模板] [编辑] [删除]  ←─── 只有创建者看到编辑/删除│  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ 📋 合同审批流程                                        │  │
│  │                                                        │  │
│  │ 描述: 分析合同条款，提取关键信息，需要财务审批        │  │
│  │ 创建者: 小李                                           │  │
│  │ 创建时间: 2024-01-15 09:00:00                         │  │
│  │ 使用次数: 8次                                         │  │
│  │                                                        │  │
│  │ [使用模板]  ←─── 非创建者只能看到"使用模板"           │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

**关键点**：
- ✅ 所有用户可以看到所有模板（is_active=true）
- ✅ 创建者可以看到"编辑"和"删除"按钮
- ✅ 非创建者只能看到"使用模板"按钮
- ❌ 不需要"提交审核"、"审批"等按钮

---

## 四、与Application的对比

### Application的发布流程（参考）

```
创建Application
    ↓
status = "draft"
    ↓
提交审核
    ↓
status = "submitted"
    ↓
管理员审批
    ↓
status = "published"  ←─── 其他用户可以使用
```

### Workflow的发布流程（简化后）

```
保存Workflow模板
    ↓
is_active = true  ←─── 立即可用，其他用户可以使用
```

**区别**：
- Application：需要审核，确保质量
- Workflow：保存即发布，快速上线（适合demo场景）

---

## 五、实现要点

### 5.1 后端实现

**保存模板API**：
```python
@router.post("/workbench/workflows/save-template")
async def save_workflow_template(
    request: WorkflowTemplateSaveRequest,
    session: SessionDep,
    current_user: CurrentUser
):
    """
    保存Workflow模板（保存即发布）
    """
    template = ApprovalWorkflowTemplate(
        name=request.name,
        description=request.description,
        resource_type="workbench_workflow",
        is_editable=request.is_editable,
        workflow_nodes=request.nodes,
        workflow_edges=request.edges,
        is_active=True,  # 保存即激活（发布）
        created_by=current_user.id,
        create_time=datetime.now(timezone.utc),
        update_time=datetime.now(timezone.utc)
    )
    
    session.add(template)
    session.commit()
    session.refresh(template)
    
    return {
        "id": template.id,
        "name": template.name,
        "message": "模板已保存并发布，其他用户现在可以使用"
    }
```

**获取模板列表API**：
```python
@router.get("/workbench/workflow-templates")
async def get_workflow_templates(
    mine: bool = False,  # 是否只查询我创建的
    session: SessionDep,
    current_user: CurrentUser
):
    """
    获取Workflow模板列表
    """
    query = select(ApprovalWorkflowTemplate).where(
        ApprovalWorkflowTemplate.resource_type == "workbench_workflow",
        ApprovalWorkflowTemplate.is_active == True,  # 只查询已发布的
        ApprovalWorkflowTemplate.yn == True
    )
    
    if mine:
        query = query.where(
            ApprovalWorkflowTemplate.created_by == current_user.id
        )
    
    templates = session.exec(query).all()
    return templates
```

### 5.2 前端实现

**保存模板按钮**：
```tsx
// 在Workbench顶部工具栏
<Button
  onClick={handleSaveTemplate}
  leftIcon={<FiSave />}
>
  保存模板
</Button>

const handleSaveTemplate = async () => {
  // 弹出保存对话框
  const result = await openSaveTemplateDialog({
    name: "",
    description: "",
    is_editable: false
  })
  
  if (result) {
    // 调用API保存
    const response = await api.post("/workbench/workflows/save-template", {
      name: result.name,
      description: result.description,
      nodes: nodes,
      edges: edges,
      is_editable: result.is_editable
    })
    
    toast({
      title: "保存成功",
      description: "模板已保存并发布，其他用户现在可以使用",
      status: "success"
    })
  }
}
```

---

## 六、需要确认的问题

### 1. 权限控制
**问题**：是否所有用户都可以创建模板并发布？
- **选项A**：所有登录用户都可以（推荐，适合demo）
- **选项B**：只有特定角色可以（如管理员）

**我的建议**：选项A，因为：
- 1.15是demo版本，需要快速上线
- 后续可以添加权限控制

### 2. 模板编辑
**问题**：创建者是否可以随时编辑已发布的模板？
- **选项A**：可以编辑（如果is_editable=true）
- **选项B**：不允许编辑已发布的模板（防止影响已创建的实例）

**我的建议**：选项A，但需要提示：
- "编辑模板不会影响已创建的实例"
- 或者：编辑后创建新版本，保留旧版本

### 3. 模板删除
**问题**：删除模板后，已创建的实例如何处理？
- **选项A**：软删除（is_active=false），实例仍然可以查看
- **选项B**：硬删除，实例关联失效

**我的建议**：选项A（软删除），因为：
- 已创建的实例应该可以继续查看
- 只是新用户看不到这个模板

### 4. 模板可见性
**问题**：是否需要区分"我创建的"和"所有模板"？
- **选项A**：需要（推荐）
- **选项B**：不需要，所有模板混在一起

**我的建议**：选项A，因为：
- 用户可以快速找到自己创建的模板
- 可以管理自己的模板

---

## 七、开发工作量评估

### 后端
- ✅ 保存模板API（简化，不需要审核流程）：**0.5天**
- ✅ 获取模板列表API：**0.5天**
- ✅ 从模板创建实例API：**1天**
- ✅ 获取实例列表API：**0.5天**
- ✅ 数据模型调整（删除status字段等）：**0.5天**
- **总计：3天**

### 前端
- ✅ 保存模板对话框：**0.5天**
- ✅ 模板库页面（列表、筛选）：**1天**
- ✅ 从模板创建实例界面：**1.5天**
- ✅ 实例列表页面：**1天**
- **总计：4天**

### 测试
- ✅ 功能测试：**1天**
- ✅ 集成测试：**0.5天**
- **总计：1.5天**

**总体评估：8.5天**（可以压缩到7天）

---

## 八、风险与注意事项

### 风险
1. **质量风险**：没有审核流程，可能发布有问题的模板
   - **缓解**：添加模板验证（检查节点连接、必填字段等）
   - **后续**：1.25号可以添加审核流程

2. **数据一致性**：编辑模板可能影响已创建的实例
   - **缓解**：提示用户"编辑不会影响已创建的实例"
   - **后续**：可以添加模板版本管理

### 注意事项
1. **向后兼容**：如果未来添加审核流程，需要兼容现有数据
2. **性能**：模板列表查询需要优化（如果模板很多）
3. **用户体验**：保存即发布需要明确提示用户

---

## 九、总结

### 核心变化
- ❌ **删除**：审核流程（submit/approve）
- ❌ **删除**：状态流转（draft → submitted → published）
- ✅ **简化**：保存即发布（is_active=true）
- ✅ **保留**：多实例化运行功能

### 实现要点
1. 保存模板时，直接设置 `is_active = true`
2. 所有用户可以看到所有已发布的模板
3. 只有创建者可以编辑/删除自己的模板
4. 从模板创建实例，支持配置覆盖

### 后续扩展（1.25号）
- 可以添加审核流程
- 可以添加模板版本管理
- 可以添加权限控制

---

## 十、需要您确认

请确认以下问题：

1. **权限控制**：是否所有登录用户都可以创建模板？
2. **模板编辑**：创建者是否可以编辑已发布的模板？
3. **模板删除**：是否使用软删除（is_active=false）？
4. **模板可见性**：是否需要"我创建的"筛选功能？

确认后，我将开始实现。

