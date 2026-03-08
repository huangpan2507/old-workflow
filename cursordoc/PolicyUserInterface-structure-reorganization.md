# PolicyUserInterface 文件结构重组总结

## 重组目标
按照 `PolicyUserInterfaceOptimized.tsx` 入口文件实现的功能（Quick Links 中的功能点），将相同功能的文件收纳到子文件夹中，提高代码组织性和可维护性。

## 重组后的文件结构

```
PolicyUserInterface/
├── PolicyUserInterfaceOptimized.tsx    # 主入口文件
├── components/                          # 功能组件目录
│   ├── ai/                            # AI 相关功能
│   │   ├── AskAIPanel.tsx             # AI 问答面板
│   │   └── CopilotSidebar.tsx         # AI 助手侧边栏
│   ├── approval/                      # 审批相关功能
│   │   ├── ApprovalNodeConfig.tsx      # 审批节点配置
│   │   ├── ApprovalWorkflowConfig.tsx  # 审批工作流配置
│   │   └── PolicyRevisionForm.tsx     # 策略修订表单
│   ├── display/                       # 数据显示组件
│   │   ├── MyApproval.tsx             # 我的审批
│   │   ├── MyPolicies.tsx             # 我的策略
│   │   └── PublishedPolicies.tsx      # 已发布策略
│   ├── modals/                        # 模态框组件
│   │   ├── FilePreviewModal.tsx       # 文件预览模态框
│   │   └── PolicyVersionHistory.tsx   # 策略版本历史
│   ├── navigation/                    # 导航组件
│   │   ├── QuickLinksPanel.tsx        # 快速链接面板
│   │   └── SearchBar.tsx              # 搜索栏
│   └── utils/                         # 工具和 hooks
│       └── hooks/
│           ├── useOnDemandData.ts     # 按需数据加载 hook
│           ├── usePolicies.ts         # 策略数据管理 hook
│           └── usePolicyActions.ts    # 策略操作 hook
└── drawer/                            # 抽屉组件
    ├── ApprovalDrawer.tsx             # 审批抽屉
    ├── DeviationDrawer.tsx            # 偏差请求抽屉
    ├── FileViewerDrawer.tsx           # 文件查看器抽屉
    └── MyPoliciesDrawer.tsx           # 我的策略抽屉
```

## Quick Links 功能映射

| Quick Link 功能 | 对应组件 | 文件位置 |
|----------------|----------|----------|
| **Explore** | PublishedPolicies | `components/display/PublishedPolicies.tsx` |
| **My Policies** | MyPolicies | `components/display/MyPolicies.tsx` |
| **Approval** | MyApproval | `components/display/MyApproval.tsx` |
| **Deviation** | DeviationDrawer | `drawer/DeviationDrawer.tsx` |
| **File Viewer** | FileViewerDrawer | `drawer/FileViewerDrawer.tsx` |
| **Workflow Config** | ApprovalWorkflowConfig | `components/approval/ApprovalWorkflowConfig.tsx` |
| **AI Assistant** | AskAIPanel + CopilotSidebar | `components/ai/` |

## 功能模块说明

### 1. AI 模块 (`components/ai/`)
- **AskAIPanel.tsx**: AI 问答面板，提供快速 AI 交互入口
- **CopilotSidebar.tsx**: AI 助手侧边栏，提供完整的 AI 对话功能

### 2. 审批模块 (`components/approval/`)
- **ApprovalNodeConfig.tsx**: 审批节点配置组件
- **ApprovalWorkflowConfig.tsx**: 审批工作流配置主组件
- **PolicyRevisionForm.tsx**: 策略修订表单

### 3. 显示模块 (`components/display/`)
- **MyApproval.tsx**: 我的审批列表显示
- **MyPolicies.tsx**: 我的策略列表显示
- **PublishedPolicies.tsx**: 已发布策略列表显示

### 4. 模态框模块 (`components/modals/`)
- **FilePreviewModal.tsx**: 文件预览模态框
- **PolicyVersionHistory.tsx**: 策略版本历史模态框

### 5. 导航模块 (`components/navigation/`)
- **QuickLinksPanel.tsx**: 快速链接面板，提供主要功能入口
- **SearchBar.tsx**: 搜索栏组件

### 6. 工具模块 (`components/utils/hooks/`)
- **useOnDemandData.ts**: 按需数据加载 hook
- **usePolicies.ts**: 策略数据管理 hook
- **usePolicyActions.ts**: 策略操作 hook

### 7. 抽屉模块 (`drawer/`)
- **ApprovalDrawer.tsx**: 审批相关抽屉组件
- **DeviationDrawer.tsx**: 偏差请求抽屉组件
- **FileViewerDrawer.tsx**: 文件查看器抽屉组件
- **MyPoliciesDrawer.tsx**: 我的策略抽屉组件

## 更新的导入路径

### 主入口文件更新
```typescript
// PolicyUserInterfaceOptimized.tsx
import AskAIPanel from "./components/ai/AskAIPanel"
import ApprovalWorkflowConfig from "./components/approval/ApprovalWorkflowConfig"
import CopilotSidebar from "./components/ai/CopilotSidebar"
import MyPolicies from "./components/display/MyPolicies"
import PolicyRevisionForm from "./components/approval/PolicyRevisionForm"
import FilePreviewModal from "./components/modals/FilePreviewModal"
import PublishedPolicies from "./components/display/PublishedPolicies"
import MyApproval from "./components/display/MyApproval"
import QuickLinksPanel from "./components/navigation/QuickLinksPanel"
import SearchBar from "./components/navigation/SearchBar"
import { useOnDemandData } from "./components/utils/hooks/useOnDemandData"
import { usePolicyActions } from "./components/utils/hooks/usePolicyActions"
```

### 其他文件更新
```typescript
// drawer/MyPoliciesDrawer.tsx
import MyPolicies from "../components/display/MyPolicies"

// drawer/ApprovalDrawer.tsx
import PublishedPolicies from "../components/display/PublishedPolicies"

// PolicyDetailModal.tsx
import PolicyVersionHistory from "../PolicyUserInterface/components/modals/PolicyVersionHistory"
```

## 重组优势

### 1. 功能模块化
- 相关功能的组件集中在一起，便于维护和开发
- 每个模块职责明确，符合单一职责原则

### 2. 代码组织性
- 文件结构清晰，易于理解和导航
- 新开发者可以快速找到相关功能的代码

### 3. 可维护性提升
- 修改特定功能时只需要关注对应模块
- 减少了文件查找时间，提高开发效率

### 4. 可扩展性
- 新功能可以独立添加到对应模块
- 模块间依赖关系清晰，便于重构

### 5. 团队协作
- 不同开发者可以并行开发不同模块
- 减少代码冲突和合并问题

## 注意事项

1. **导入路径**: 所有导入路径已更新，确保项目能正常运行
2. **功能完整性**: 重组过程中保持了所有功能的完整性
3. **向后兼容**: 主入口文件 `PolicyUserInterfaceOptimized.tsx` 保持不变
4. **类型安全**: 所有 TypeScript 类型定义保持完整

## 总结

通过这次文件结构重组，PolicyUserInterface 组件现在具有了更好的组织性和可维护性。每个功能模块都有明确的职责，文件结构清晰，便于团队协作和后续开发。
