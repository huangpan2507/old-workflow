# 实现工作台可连接app节点

## 项目概述

本项目实现了在Workbench工作台中，从AGENTS区域拖拽应用程序到中央画布，显示为可连接的app节点。该节点支持输入输出配置，可以与其他节点连线，并能够执行应用程序逻辑（文件上传或URL处理），最终显示markdown格式的结果。

## 需求演进

### 初始需求
1. 从AGENTS区域拖拽application到中央区域
2. 拖拽后只显示一个app node
3. app node需要明确显示输入和输出
4. 输入参考Standard导航栏的app输入（Requirement可选文本、Input Mode URL/File Upload、URL或文件上传）
5. 输出为markdown格式的结果
6. app node可以与其他节点连线

### 需求细化
1. app node需要直接嵌入Standard模式下`ReportDiskStandardView`的UI功能
2. 在app node上可以填写：
   - Requirement (Optional)
   - Input Mode (URL Input / File Upload)
   - URL (string) 或直接文件上传
3. 点击Run/Process Files按钮后，处理几分钟后显示markdown格式结果
4. 复用Standard模式下app的逻辑

### 最终方案
- **节点UI设计**：节点显示基本信息，右侧面板显示详细表单（类似NodeConfigPanel）
- **结果展示方式**：在右侧配置面板中显示markdown结果
- **输入端口连接**：支持通过Handle连接其他节点
- **执行时机**：手动执行（点击Run按钮）
- **代码组织**：创建独立的hook和配置面板组件

## 技术实现

### 1. 核心组件

#### 1.1 ApplicationNode (`/frontend/src/components/Workbench/FlowNodes/ApplicationNode.tsx`)

**功能**：
- 显示app节点的可视化表示
- 提供连接端口（输入/输出）
- 支持节点操作（配置、重命名、固定、删除）

**关键特性**：
- **紧凑布局**：采用水平布局，最小宽度120px，最大宽度200px
- **简化显示**：只显示应用图标、名称和执行状态，移除了冗余信息（Description、Owner、Area、Category、Disks等）
- **连接端口**：
  - 输入端口（左侧）：`id="input"`，统一接收requirement和input_data
  - 输出端口（右侧）：`id="markdown_output"`，输出markdown结果
- **拖拽优化**：
  - 拖拽阈值从3px提高到5px，提升拖拽检测准确性
  - 通过`onDisableDrag`回调立即禁用节点拖拽
  - 使用`data-dragging-from-button`属性标记拖拽来源
  - 在`onNodeDragStart`中检测并取消节点拖拽，允许连接创建

**数据结构**：
```typescript
export interface ApplicationNodeData {
  id: string
  name: string
  appNodeId?: number  // 引用app_node表（执行必需）
  executionState?: {
    requirement?: string
    inputMode?: 'url' | 'file'
    url?: string
    files?: File[]
    status: 'idle' | 'running' | 'completed' | 'failed' | 'partial_success'
    dagRunId?: string
    markdownResult?: string
    error?: string
  }
  inputs?: {
    requirement?: string
    inputData?: any
  }
  outputs?: {
    markdownOutput?: string
  }
  onConfigure?: (id: string) => void
  onDisableDrag?: (nodeId: string) => void
  // ... 其他字段
}
```

#### 1.2 AppNodeConfigPanel (`/frontend/src/components/Workbench/AppNodeConfigPanel.tsx`)

**功能**：
- 提供app节点的配置界面（右侧面板）
- 支持Requirement输入、Input Mode选择、URL输入或文件上传
- 执行应用程序并显示结果

**UI组件**：
- Requirement文本输入框（可选）
- Input Mode选择器（URL / File Upload）
- URL输入框（URL模式）
- 文件上传区域（File Upload模式）
- Run/Process Files按钮
- 执行状态显示（idle/running/completed/failed/partial_success）
- Markdown结果展示区域

**关键逻辑**：
- 使用`useAppExecution` hook管理执行状态
- 同步执行状态到节点数据
- 支持文件多选和移除
- 显示执行进度和结果

#### 1.3 useAppExecution Hook (`/frontend/src/hooks/useAppExecution.ts`)

**功能**：
- 封装应用程序执行的业务逻辑
- 管理执行状态（idle/running/completed/failed/partial_success）
- 处理文件上传和URL处理
- DAG状态轮询和结果获取
- 结果缓存管理

**核心方法**：
- `processFiles()`: 处理文件上传
- `processUrl()`: 处理URL输入
- `getDAGStatus()`: 获取DAG运行状态
- `getDAGResult()`: 获取DAG执行结果
- `startPolling()`: 开始轮询DAG状态
- `updateState()`: 更新执行状态
- `resetState()`: 重置执行状态

**状态管理**：
```typescript
export interface AppExecutionState {
  requirement: string
  inputMode: 'url' | 'file'
  url: string
  files: File[]
  status: 'idle' | 'running' | 'completed' | 'failed' | 'partial_success'
  dagRunId?: string
  markdownResult?: string
  error?: string
  progress?: number
}
```

**缓存机制**：
- 使用localStorage缓存执行结果
- 缓存键格式：`app_execution_${appNodeId}_${dagRunId}`
- 缓存有效期：24小时

#### 1.4 FlowWorkspace集成 (`/frontend/src/components/Workbench/FlowWorkspace.tsx`)

**关键修改**：

1. **状态管理**：
```typescript
const [selectedAppNodeId, setSelectedAppNodeId] = useState<string | null>(null)
const [isAppNodePanelOpen, setIsAppNodePanelOpen] = useState(false)
```

2. **节点创建**：
```typescript
const addApplicationNode = useCallback((applicationData?: any, position?: { x: number; y: number }) => {
  const newNodeId = `app-${nodeIdCounter}`
  // 创建application类型节点，初始化executionState
  const newNode: Node = {
    id: newNodeId,
    type: 'application',
    data: {
      // ... 节点数据
      executionState: {
        requirement: '',
        inputMode: 'url',
        url: '',
        files: [],
        status: 'idle',
      },
      inputs: {},
      outputs: {},
    },
  }
  // ...
}, [])
```

3. **配置处理**：
```typescript
const handleNodeConfigure = useCallback((nodeId: string) => {
  const node = nodes.find(n => n.id === nodeId)
  if (node && node.type === 'application') {
    setSelectedAppNodeId(nodeId)
    setIsAppNodePanelOpen(true)
  } else {
    // 其他节点使用NodeConfigPanel
    setSelectedNodeId(nodeId)
    setIsNodePanelOpen(true)
  }
}, [nodes])
```

4. **拖拽处理**：
```typescript
const onNodeDragStart = useCallback((event: React.MouseEvent, node: any) => {
  // 检测是否从+按钮拖拽
  // 如果是，禁用节点拖拽，允许连接创建
  // ...
}, [setNodes])
```

5. **面板渲染**：
```typescript
<AppNodeConfigPanel
  nodeId={selectedAppNodeId}
  nodeData={selectedAppNodeId ? (nodes.find(n => n.id === selectedAppNodeId)?.data as ApplicationNodeData) || null : null}
  isOpen={isAppNodePanelOpen}
  onClose={() => {
    setIsAppNodePanelOpen(false)
    setSelectedAppNodeId(null)
  }}
  onUpdate={(nodeId, updates) => {
    setNodes((nds) =>
      nds.map((node) =>
        node.id === nodeId ? { ...node, data: { ...node.data, ...updates } } : node
      )
    )
  }}
/>
```

### 2. 后端集成

#### 2.1 ReportService (`/frontend/src/services/reportService.ts`)

**API方法**：
- `uploadFiles(appNodeId, files, requirement?)`: 上传文件处理
- `processUrl(appNodeId, url, requirement?)`: 处理URL
- `getDAGStatus(dagRunId)`: 获取DAG状态
- `getDAGResult(dagRunId)`: 获取DAG结果

**接口规范**：
- 文件上传：`POST /api/reports/upload-files/{app_node_id}`
- URL处理：`POST /api/reports/process-url/{app_node_id}`
- DAG状态：`GET /api/reports/dag-status/{dag_run_id}`
- DAG结果：`GET /api/reports/dag-result/{dag_run_id}`

### 3. UI优化

#### 3.1 节点样式优化

**修改前**：
- 尺寸：`minW="300px"`, `maxW="400px"`
- Padding：`p={4}`
- 边框：`border="2px solid"`
- 字体：`fontSize="lg"`（节点名称）
- 显示内容：包含Description、Owner、Area、Category、Disks等冗余信息

**修改后**：
- 尺寸：`minW="120px"`, `maxW="200px"`
- Padding：`p={0.5}`, `pl={2}`, `pr={2}`
- 边框：`border="1px solid"`
- 字体：`fontSize="8px"`（节点名称和图标）
- 布局：水平布局（HStack），与其他节点一致
- 显示内容：仅显示图标、名称、执行状态（如有）

#### 3.2 拖拽连接优化

**问题**：
- 从+按钮拖拽时，经常移动整个节点而不是创建连接
- 拖拽检测不够准确

**解决方案**：
1. 提高拖拽阈值：从3px改为5px
2. 立即禁用节点拖拽：在`handlePlusPointerDown`中调用`onDisableDrag`回调
3. 标记拖拽来源：使用`data-dragging-from-button`属性
4. 在`onNodeDragStart`中检测并取消节点拖拽

## 文件结构

```
frontend/src/
├── components/Workbench/
│   ├── FlowWorkspace.tsx          # 主工作区组件（集成app节点）
│   ├── FlowNodes/
│   │   └── ApplicationNode.tsx    # App节点组件
│   ├── AppNodeConfigPanel.tsx     # App节点配置面板
│   └── NodeConfigPanel.tsx        # 通用节点配置面板
├── hooks/
│   └── useAppExecution.ts         # App执行逻辑hook
└── services/
    └── reportService.ts           # 报告服务API
```

## 关键问题与解决方案

### 问题1：`Cannot access 'handleNodeConfigure' before initialization`

**原因**：`addApplicationNode`在`useCallback`依赖数组中引用了`handleNodeConfigure`，但`handleNodeConfigure`定义在其后，导致初始化时引用错误。

**解决方案**：将`handleNodeConfigure`的定义移到`addApplicationNode`之前。

### 问题2：`selectedAppNodeId is not defined`

**原因**：状态变量`selectedAppNodeId`和`isAppNodePanelOpen`的`useState`定义缺失或位置不正确。

**解决方案**：在`FlowWorkspace.tsx`中添加这两个状态变量的定义。

### 问题3：拖拽连接体验不佳

**原因**：
- 拖拽阈值太小（3px），容易误触发节点拖拽
- 节点容器较大，拖拽时容易移动整个节点

**解决方案**：
- 提高拖拽阈值到5px
- 立即禁用节点拖拽（通过`onDisableDrag`回调）
- 在`onNodeDragStart`中检测并取消节点拖拽

### 问题4：节点显示冗余内容

**原因**：节点显示了过多不必要的信息（Description、Owner、Area、Category、Disks等），导致节点过大，与其他节点不一致。

**解决方案**：
- 移除冗余信息
- 采用紧凑的水平布局
- 缩小字体和图标尺寸
- 统一节点样式

## 使用流程

1. **创建App节点**：
   - 从AGENTS区域拖拽application到中央画布
   - 系统自动创建application类型节点

2. **配置App节点**：
   - 右键点击节点或点击菜单中的"Configure"
   - 打开右侧配置面板（AppNodeConfigPanel）

3. **输入配置**：
   - 填写Requirement（可选）
   - 选择Input Mode（URL或File Upload）
   - 输入URL或上传文件

4. **执行应用**：
   - 点击"Run"或"Process Files"按钮
   - 系统开始处理，状态显示为"running"
   - 轮询DAG状态直到完成

5. **查看结果**：
   - 执行完成后，markdown结果显示在配置面板下方
   - 结果自动保存到节点的`outputs.markdownOutput`

6. **连接节点**：
   - 悬停节点显示+按钮
   - 从+按钮拖拽到其他节点创建连接
   - 输入端口接收其他节点的输出
   - 输出端口连接到其他节点的输入

## 技术栈

- **React**: UI框架
- **React Flow**: 节点编辑器库
- **Chakra UI**: UI组件库
- **React Dnd**: 拖拽功能
- **React Markdown**: Markdown渲染
- **TypeScript**: 类型安全

## 后续优化建议

1. **性能优化**：
   - 优化大文件上传体验
   - 添加上传进度显示
   - 优化DAG状态轮询频率

2. **功能增强**：
   - 支持从其他节点接收requirement和input_data
   - 支持结果预览和导出
   - 支持执行历史记录

3. **用户体验**：
   - 添加执行时间估算
   - 优化错误提示信息
   - 支持批量文件处理

4. **代码优化**：
   - 提取通用连接逻辑
   - 统一节点样式系统
   - 优化状态管理

## 总结

本项目成功实现了工作台中app节点的连接和执行功能，通过模块化的设计（独立的hook和配置面板），实现了代码复用和功能解耦。节点采用紧凑布局，与其他节点保持一致，提升了整体用户体验。拖拽连接功能经过优化，解决了误触发节点拖拽的问题。





