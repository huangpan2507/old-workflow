# Workflow保存即发布方案（1.15交付版本）- 模板不可修改

## 一、需求变更说明

### 原计划
- **1.15号**：workflow保存 + 多实例化运行
- **1.25号**：workflow发布、审核

### 新计划（调整后）
- **1.15号**：workflow保存即发布 + 多实例化运行
  - ✅ 保存模板后，**直接发布**，其他用户立即可用
  - ❌ **不需要**审核流程（submit/approve）
  - ❌ **不需要**状态流转（draft → submitted → published）
  - ✅ **模板结构不可修改**（默认，UI不提供选项）

---

## 二、核心设计原则

### 2.1 模板结构不可修改

**设计原则**：
- ✅ **默认不可修改**：所有模板的结构（节点、连线）默认不可修改
- ✅ **UI不提供选项**：保存模板时，UI界面不显示"允许修改模板结构"的选项
- ✅ **数据库字段**：`is_editable = false`（固定值，不在UI中暴露）
- ✅ **使用模板时**：只能配置节点参数（文件、提示词等），不能修改结构

**实现方式**：
```python
# 保存模板时，固定设置
template = ApprovalWorkflowTemplate(
    ...
    is_editable=False,  # 固定为False，不在UI中显示选项
    ...
)
```

---

## 三、简化后的设计

### 3.1 状态管理简化

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
创建即发布，结构不可修改
```

**实现方式**：
- 保存模板时，直接设置 `is_active = true`
- 固定设置 `is_editable = false`（不在UI中显示）
- 不需要 `status` 字段（或固定为 "published"）

---

### 3.2 数据模型

**ApprovalWorkflowTemplate 字段**：
```python
class ApprovalWorkflowTemplate:
    id: int
    name: str
    description: str | None
    resource_type: str = "workbench_workflow"
    is_editable: bool = False  # 固定为False，不在UI中暴露
    
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

**关键点**：
- ✅ `is_editable` 字段存在，但**固定为False**
- ✅ **不在UI中显示**该选项
- ✅ 用户使用模板时，**只能配置节点参数**，不能修改结构

---

### 3.3 API设计

#### 保存模板API（简化）

```python
POST /api/v1/workbench/workflows/save-template
{
  "name": "简历审核流程",
  "description": "自动分析简历，判断是否符合要求",
  "nodes": [...],
  "edges": [...]
  // ❌ 不包含 is_editable 字段（后端固定为false）
}

Response:
{
  "id": 1,
  "name": "简历审核流程",
  "message": "模板已保存并发布，其他用户现在可以使用"
}
```

**后端实现**：
```python
@router.post("/workbench/workflows/save-template")
async def save_workflow_template(
    request: WorkflowTemplateSaveRequest,
    session: SessionDep,
    current_user: CurrentUser
):
    """
    保存Workflow模板（保存即发布，结构不可修改）
    """
    template = ApprovalWorkflowTemplate(
        name=request.name,
        description=request.description,
        resource_type="workbench_workflow",
        is_editable=False,  # 固定为False，不在UI中显示
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

**完整的API列表**：
```python
# 1. 保存模板（创建即发布，结构不可修改）
POST /api/v1/workbench/workflows/save-template
→ 创建模板，is_active=true，is_editable=false（固定）

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

**注意**：
- ❌ **不提供**编辑模板结构的API（因为is_editable=false）
- ✅ 可以删除模板（软删除）
- ✅ 可以查看模板详情

---

## 四、UI交互流程

### 4.1 保存模板流程（简化，无is_editable选项）

```
用户在Workbench中搭建Workflow
    ↓
点击"保存模板"按钮
    ↓
弹出保存对话框（简化版）：
  ┌────────────────────────────────────────┐
  │  保存Workflow模板          [×]          │
  ├────────────────────────────────────────┤
  │                                         │
  │  模板名称: [________________]           │
  │                                         │
  │  描述: [________________________________]│
  │       [________________________________]│
  │                                         │
  │  ⚠️ 提示：                               │
  │  • 保存后，所有用户都可以使用此模板     │
  │  • 模板结构（节点、连线）不可修改      │
  │  • 使用模板时，只能配置节点参数        │
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

**关键点**：
- ❌ **不显示**"允许修改模板结构"的复选框
- ✅ **显示提示**：说明模板结构不可修改
- ✅ 用户明确知道：保存后结构固定，只能配置参数

---

### 4.2 使用模板创建实例流程

```
用户进入模板库
    ↓
选择模板："简历审核流程"
    ↓
点击"使用模板"
    ↓
加载模板到画布（只读模式）：
  ┌─────────────────────────────────────────────────────────────┐
  │  ← 返回模板库    简历审核流程 (模板)          [执行] [取消] │
  ├─────────────────────────────────────────────────────────────┤
  │                                                              │
  │  ⚠️ 提示: 这是模板结构，不能修改。您可以配置节点参数。      │
  │                                                              │
  │  ┌─────────┐                                               │
  │  │  START  │                                               │
  │  └────┬────┘                                               │
  │       │                                                     │
  │       ▼                                                     │
  │  ┌─────────────────────┐                                   │
  │  │   App Node          │  ← 点击配置参数                   │
  │  │   简历分析          │                                   │
  │  │   [配置]            │                                   │
  │  └────┬────────────────┘                                   │
  │       │                                                     │
  │       ▼                                                     │
  │  ┌─────────────────────┐                                   │
  │  │ Condition Node      │                                   │
  │  │   score > 80        │                                   │
  │  └────┬────────────────┘                                   │
  │       │                                                     │
  │       ▼                                                     │
  │  ┌─────────────────────┐                                   │
  │  │ HITL Node           │  ← 点击配置参数                   │
  │  │   审批              │                                   │
  │  │   [配置]            │                                   │
  │  └────┬────────────────┘                                   │
  │       │                                                     │
  │       ▼                                                     │
  │  ┌─────────┐                                               │
  │  │   END   │                                               │
  │  └─────────┘                                               │
  │                                                              │
  │  🔒 只读模式: 不能拖拽、不能增删节点、不能修改连线          │
  │                                                              │
  └─────────────────────────────────────────────────────────────┘
    ↓
点击节点，配置参数：
  ┌─────────────────────────────────────────────────────────────┐
  │  配置节点: App Node (app-1)                      [× 关闭] │
  ├─────────────────────────────────────────────────────────────┤
  │                                                              │
  │  节点信息                                                     │
  │  ┌──────────────────────────────────────────────────────┐  │
  │  │ 节点类型: App Node                                    │  │
  │  │ 节点ID: app-1 (不可修改)                            │  │
  │  │ Application: 简历分析                               │  │
  │  └──────────────────────────────────────────────────────┘  │
  │                                                              │
  │  配置参数 (可修改)                                            │
  │  ┌──────────────────────────────────────────────────────┐  │
  │  │ 输入模式: ○ URL  ● 文件上传                          │  │
  │  │                                                      │  │
  │  │ 文件上传:                                            │  │
  │  │ ┌──────────────────────────────────────────────┐  │  │
  │  │ │  [选择文件] 或拖拽文件到此处                  │  │  │
  │  │ │  当前文件: zhangsan_resume.pdf                │  │  │
  │  │ └──────────────────────────────────────────────┘  │  │
  │  │                                                      │  │
  │  │ 提示词 (可选覆盖):                                  │  │
  │  │ ┌──────────────────────────────────────────────┐  │  │
  │  │ │ 重点分析工作经验和技能匹配度                  │  │  │
  │  │ │ (留空则使用模板默认提示词)                    │  │  │
  │  │ └──────────────────────────────────────────────┘  │  │
  │  └──────────────────────────────────────────────────────┘  │
  │                                                              │
  │  [保存配置]  [取消]                                          │
  │                                                              │
  └─────────────────────────────────────────────────────────────┘
    ↓
点击"执行"
    ↓
创建实例并执行
```

**关键点**：
- ✅ 画布**只读模式**：不能拖拽、不能增删节点、不能修改连线
- ✅ 可以**配置节点参数**：文件上传、提示词、审批人等
- ✅ 配置覆盖**不影响模板**：只影响当前实例

---

### 4.3 模板库界面

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
│  │ [使用模板] [删除]  ←─── 只有创建者看到删除按钮       │  │
│  │                        ❌ 不显示"编辑"按钮（结构不可修改）│
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
- ✅ 创建者可以看到"删除"按钮
- ❌ **不显示"编辑"按钮**（因为结构不可修改）
- ✅ 非创建者只能看到"使用模板"按钮

---

## 五、数据模型详细设计

### 5.1 ApprovalWorkflowTemplate

```python
class ApprovalWorkflowTemplate(SQLModel, table=True):
    """Workflow模板 - 保存即发布，结构不可修改"""
    __tablename__ = "approval_workflow_templates"
    
    id: int = Field(default=None, primary_key=True)
    
    # 模板信息
    name: str = Field(max_length=255, description="模板名称")
    description: str | None = Field(default=None, max_length=1000, description="模板描述")
    resource_type: str = Field(
        default="workbench_workflow",
        max_length=50,
        description="资源类型：workbench_workflow"
    )
    
    # 工作流结构定义
    workflow_nodes: Optional[List[Dict[str, Any]]] = Field(
        default=None,
        sa_column=Column(JSONB),
        description="节点定义：[{id, type, data, position}, ...]"
    )
    workflow_edges: Optional[List[Dict[str, Any]]] = Field(
        default=None,
        sa_column=Column(JSONB),
        description="边定义：[{source, target, ...}, ...]"
    )
    
    # 模板属性（固定值，不在UI中暴露）
    is_editable: bool = Field(
        default=False,
        description="是否允许修改结构（固定为False，不在UI中显示）"
    )
    is_active: bool = Field(
        default=True,
        description="是否激活（保存即发布，true表示已发布）"
    )
    
    # 元数据
    created_by: uuid.UUID | None = Field(default=None, foreign_key="users.id")
    create_time: datetime = Field(default_factory=get_current_time)
    update_time: datetime = Field(default_factory=get_current_time)
    yn: bool = Field(default=True, nullable=False)
    
    # 索引
    __table_args__ = (
        Index('ix_template_resource_type', 'resource_type'),
        Index('ix_template_active', 'is_active'),
        Index('ix_template_created_by', 'created_by'),
    )
```

### 5.2 WorkflowInstance

```python
class WorkflowInstance(SQLModel, table=True):
    """Workflow执行实例"""
    __tablename__ = "workflow_instances"
    
    id: int = Field(default=None, primary_key=True)
    template_id: int = Field(
        foreign_key="approval_workflow_templates.id",
        description="关联的模板ID"
    )
    
    # 执行状态
    status: str = Field(
        default="pending",
        description="执行状态: pending, running, completed, failed, paused"
    )
    current_node_id: Optional[str] = Field(
        default=None,
        description="当前执行节点ID"
    )
    
    # 执行结果
    execution_result: Optional[Dict[str, Any]] = Field(
        default=None,
        sa_column=Column(JSONB),
        description="最终执行结果和摘要"
    )
    
    # 元数据
    created_by: uuid.UUID | None = Field(default=None, foreign_key="users.id")
    create_time: datetime = Field(default_factory=get_current_time)
    update_time: datetime = Field(default_factory=get_current_time)
    
    # 索引
    __table_args__ = (
        Index('ix_workflow_instance_template', 'template_id'),
        Index('ix_workflow_instance_status', 'status'),
        Index('ix_workflow_instance_created_by', 'created_by'),
    )
```

### 5.3 WorkflowExecutionTask（扩展）

```python
class WorkflowExecutionTask(SQLModel, table=True):
    """Workflow执行任务（关联实例）"""
    __tablename__ = "workflow_execution_tasks"
    
    id: int = Field(default=None, primary_key=True)
    task_id: str = Field(unique=True, index=True, max_length=255)
    
    # 关联实例（新增）
    instance_id: Optional[int] = Field(
        default=None,
        foreign_key="workflow_instances.id",
        description="关联的实例ID（如果从模板创建）"
    )
    
    # 工作流定义（从模板加载 + 配置覆盖）
    nodes: List[Dict[str, Any]] = Field(sa_column=Column(JSONB))
    edges: List[Dict[str, Any]] = Field(sa_column=Column(JSONB))
    input_data: Dict[str, Any] = Field(sa_column=Column(JSONB))
    app_node_configs: Dict[str, Any] = Field(sa_column=Column(JSONB))
    
    # 执行状态（实时更新）
    status: str = Field(default="pending", max_length=50)
    current_node_id: Optional[str] = Field(default=None, max_length=255)
    node_results: Dict[str, Any] = Field(
        default_factory=dict,
        sa_column=Column(JSONB)
    )
    
    # LangGraph相关
    thread_id: Optional[str] = Field(default=None, index=True, max_length=255)
    checkpoint_id: Optional[str] = Field(default=None, max_length=255)
    
    # 元数据
    created_by: uuid.UUID | None = Field(default=None, foreign_key="users.id")
    create_time: datetime = Field(default_factory=get_current_time)
    update_time: datetime = Field(default_factory=get_current_time)
    finished_at: Optional[datetime] = Field(default=None)
    
    # 索引
    __table_args__ = (
        Index('ix_workflow_task_task_id', 'task_id'),
        Index('ix_workflow_task_instance', 'instance_id'),
        Index('ix_workflow_task_status', 'status'),
        Index('ix_workflow_task_thread_id', 'thread_id'),
        Index('ix_workflow_task_created_by', 'created_by'),
    )
```

---

## 六、前端实现细节

### 6.1 保存模板对话框（无is_editable选项）

```tsx
interface SaveTemplateDialogProps {
  isOpen: boolean
  onClose: () => void
  onSave: (data: { name: string; description: string }) => void
}

const SaveTemplateDialog: React.FC<SaveTemplateDialogProps> = ({
  isOpen,
  onClose,
  onSave
}) => {
  const [name, setName] = useState("")
  const [description, setDescription] = useState("")
  
  const handleSave = () => {
    if (!name.trim()) {
      toast({
        title: "错误",
        description: "请输入模板名称",
        status: "error"
      })
      return
    }
    
    onSave({
      name: name.trim(),
      description: description.trim()
      // ❌ 不包含 is_editable 字段
    })
  }
  
  return (
    <Modal isOpen={isOpen} onClose={onClose}>
      <ModalOverlay />
      <ModalContent>
        <ModalHeader>保存Workflow模板</ModalHeader>
        <ModalCloseButton />
        <ModalBody>
          <VStack spacing={4}>
            <FormControl isRequired>
              <FormLabel>模板名称</FormLabel>
              <Input
                value={name}
                onChange={(e) => setName(e.target.value)}
                placeholder="例如：简历审核流程"
              />
            </FormControl>
            
            <FormControl>
              <FormLabel>描述</FormLabel>
              <Textarea
                value={description}
                onChange={(e) => setDescription(e.target.value)}
                placeholder="描述此模板的用途和使用场景"
                rows={3}
              />
            </FormControl>
            
            {/* ⚠️ 提示信息 */}
            <Alert status="info">
              <AlertIcon />
              <VStack align="start" spacing={1}>
                <Text fontSize="sm" fontWeight="bold">
                  重要提示：
                </Text>
                <Text fontSize="sm">
                  • 保存后，所有用户都可以使用此模板
                </Text>
                <Text fontSize="sm">
                  • 模板结构（节点、连线）不可修改
                </Text>
                <Text fontSize="sm">
                  • 使用模板时，只能配置节点参数（文件、提示词等）
                </Text>
              </VStack>
            </Alert>
          </VStack>
        </ModalBody>
        <ModalFooter>
          <Button variant="ghost" mr={3} onClick={onClose}>
            取消
          </Button>
          <Button colorScheme="blue" onClick={handleSave}>
            保存并发布
          </Button>
        </ModalFooter>
      </ModalContent>
    </Modal>
  )
}
```

### 6.2 模板库列表（无编辑按钮）

```tsx
const TemplateCard: React.FC<{ template: WorkflowTemplate }> = ({
  template
}) => {
  const { currentUser } = useAuth()
  const isOwner = template.created_by === currentUser.id
  
  return (
    <Box borderWidth="1px" borderRadius="lg" p={4}>
      <VStack align="start" spacing={2}>
        <HStack justify="space-between" width="100%">
          <Heading size="md">{template.name}</Heading>
          <Badge colorScheme="green">已发布</Badge>
        </HStack>
        
        <Text color="gray.600">{template.description}</Text>
        
        <HStack spacing={2} fontSize="sm" color="gray.500">
          <Text>创建者: {template.creator_name}</Text>
          <Text>•</Text>
          <Text>使用次数: {template.usage_count}</Text>
        </HStack>
        
        <HStack spacing={2}>
          <Button
            colorScheme="blue"
            size="sm"
            onClick={() => handleUseTemplate(template.id)}
          >
            使用模板
          </Button>
          
          {/* 只有创建者可以看到删除按钮 */}
          {isOwner && (
            <Button
              colorScheme="red"
              variant="outline"
              size="sm"
              onClick={() => handleDeleteTemplate(template.id)}
            >
              删除
            </Button>
          )}
          
          {/* ❌ 不显示"编辑"按钮（结构不可修改） */}
        </HStack>
      </VStack>
    </Box>
  )
}
```

---

## 七、后端实现细节

### 7.1 保存模板API

```python
class WorkflowTemplateSaveRequest(BaseModel):
    """保存模板请求（不包含is_editable）"""
    name: str
    description: str | None = None
    nodes: List[Dict[str, Any]]
    edges: List[Dict[str, Any]]


@router.post("/workbench/workflows/save-template")
async def save_workflow_template(
    request: WorkflowTemplateSaveRequest,
    session: SessionDep,
    current_user: CurrentUser
):
    """
    保存Workflow模板（保存即发布，结构不可修改）
    
    - 保存后立即可用（is_active=True）
    - 结构不可修改（is_editable=False，固定值）
    - 所有用户可以看到并使用
    """
    # 验证节点和边
    if not request.nodes:
        raise HTTPException(400, "节点不能为空")
    if not request.edges:
        raise HTTPException(400, "边不能为空")
    
    # 创建模板
    template = ApprovalWorkflowTemplate(
        name=request.name,
        description=request.description,
        resource_type="workbench_workflow",
        workflow_nodes=request.nodes,
        workflow_edges=request.edges,
        is_editable=False,  # 固定为False，不在UI中显示
        is_active=True,    # 保存即发布
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

### 7.2 获取模板列表API

```python
@router.get("/workbench/workflow-templates")
async def get_workflow_templates(
    mine: bool = False,
    session: SessionDep,
    current_user: CurrentUser
):
    """
    获取Workflow模板列表
    
    - 只返回已发布的模板（is_active=True）
    - 支持筛选"我创建的"模板
    """
    query = select(ApprovalWorkflowTemplate).where(
        ApprovalWorkflowTemplate.resource_type == "workbench_workflow",
        ApprovalWorkflowTemplate.is_active == True,
        ApprovalWorkflowTemplate.yn == True
    )
    
    if mine:
        query = query.where(
            ApprovalWorkflowTemplate.created_by == current_user.id
        )
    
    templates = session.exec(query.order_by(
        ApprovalWorkflowTemplate.create_time.desc()
    )).all()
    
    # 统计使用次数
    result = []
    for template in templates:
        usage_count = session.exec(
            select(func.count(WorkflowInstance.id)).where(
                WorkflowInstance.template_id == template.id
            )
        ).one()
        
        result.append({
            "id": template.id,
            "name": template.name,
            "description": template.description,
            "created_by": template.created_by,
            "create_time": template.create_time,
            "usage_count": usage_count,
            "is_owner": template.created_by == current_user.id
        })
    
    return result
```

### 7.3 删除模板API

```python
@router.delete("/workbench/workflow-templates/{template_id}")
async def delete_workflow_template(
    template_id: int,
    session: SessionDep,
    current_user: CurrentUser
):
    """
    删除模板（软删除）
    
    - 只有创建者可以删除
    - 软删除（is_active=False）
    - 已创建的实例不受影响
    """
    template = session.get(ApprovalWorkflowTemplate, template_id)
    if not template or not template.yn:
        raise HTTPException(404, "模板不存在")
    
    # 检查权限
    if template.created_by != current_user.id:
        raise HTTPException(403, "只能删除自己创建的模板")
    
    # 软删除
    template.is_active = False
    template.update_time = datetime.now(timezone.utc)
    
    session.add(template)
    session.commit()
    
    return {"message": "模板已删除"}
```

---

## 八、需要确认的问题

### 1. 权限控制
**问题**：是否所有登录用户都可以创建模板并发布？
- **选项A**：所有登录用户都可以（推荐，适合demo）
- **选项B**：只有特定角色可以（如管理员）

**我的建议**：选项A，因为：
- 1.15是demo版本，需要快速上线
- 后续可以添加权限控制

### 2. 模板删除
**问题**：删除模板后，已创建的实例如何处理？
- **选项A**：软删除（is_active=false），实例仍然可以查看（推荐）
- **选项B**：硬删除，实例关联失效

**我的建议**：选项A（软删除），因为：
- 已创建的实例应该可以继续查看
- 只是新用户看不到这个模板

### 3. 模板可见性
**问题**：是否需要区分"我创建的"和"所有模板"？
- **选项A**：需要（推荐）
- **选项B**：不需要，所有模板混在一起

**我的建议**：选项A，因为：
- 用户可以快速找到自己创建的模板
- 可以管理自己的模板

### 4. 模板结构验证
**问题**：保存模板时，是否需要验证结构合法性？
- **选项A**：需要（推荐）
  - 检查是否有Start节点和End节点
  - 检查节点连接是否合法
  - 检查是否有孤立节点
- **选项B**：不需要，保存时不做验证

**我的建议**：选项A，因为：
- 防止保存无效的模板
- 提升用户体验

---

## 九、开发工作量评估

### 后端
- ✅ 保存模板API（固定is_editable=false）：**0.5天**
- ✅ 获取模板列表API：**0.5天**
- ✅ 删除模板API（软删除）：**0.5天**
- ✅ 从模板创建实例API：**1天**
- ✅ 获取实例列表API：**0.5天**
- ✅ 模板结构验证：**0.5天**
- ✅ 数据模型调整：**0.5天**
- **总计：4天**

### 前端
- ✅ 保存模板对话框（无is_editable选项）：**0.5天**
- ✅ 模板库页面（列表、筛选）：**1天**
- ✅ 从模板创建实例界面（只读画布）：**1.5天**
- ✅ 实例列表页面：**1天**
- **总计：4天**

### 测试
- ✅ 功能测试：**1天**
- ✅ 集成测试：**0.5天**
- **总计：1.5天**

**总体评估：9.5天**（可以压缩到8天）

---

## 十、风险与注意事项

### 风险
1. **质量风险**：没有审核流程，可能发布有问题的模板
   - **缓解**：添加模板验证（检查节点连接、必填字段等）
   - **后续**：1.25号可以添加审核流程

2. **用户误操作**：用户可能误删模板
   - **缓解**：删除前确认对话框
   - **后续**：可以添加回收站功能

### 注意事项
1. **向后兼容**：如果未来添加审核流程，需要兼容现有数据
2. **性能**：模板列表查询需要优化（如果模板很多）
3. **用户体验**：保存即发布需要明确提示用户
4. **结构不可修改**：UI中明确提示，避免用户困惑

---

## 十一、总结

### 核心设计
- ✅ **保存即发布**：`is_active = true`（保存后立即可用）
- ✅ **结构不可修改**：`is_editable = false`（固定值，不在UI中显示）
- ✅ **简化流程**：不需要审核，不需要状态流转
- ✅ **多实例化**：支持从模板创建多个实例

### 实现要点
1. 保存模板时，固定设置 `is_editable = False`
2. UI中不显示"允许修改模板结构"选项
3. 使用模板时，画布只读模式，只能配置节点参数
4. 所有用户可以看到所有已发布的模板
5. 只有创建者可以删除自己的模板

### 后续扩展（1.25号）
- 可以添加审核流程
- 可以添加模板版本管理
- 可以添加权限控制
- 可以添加模板编辑功能（如果is_editable=true）

---

## 十二、需要您确认

请确认以下问题：

1. **权限控制**：是否所有登录用户都可以创建模板？
2. **模板删除**：是否使用软删除（is_active=false）？
3. **模板可见性**：是否需要"我创建的"筛选功能？
4. **模板结构验证**：保存时是否需要验证结构合法性？

确认后，我将开始实现。



