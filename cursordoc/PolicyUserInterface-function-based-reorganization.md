# PolicyUserInterface 按功能重组文件结构

## 重组目标
按照 Quick Links 中的功能点，为每个功能创建独立的文件夹，确保一个功能一个文件夹的原则。

## Quick Links 功能分析

根据 `QuickLinksPanel.tsx` 的分析，Quick Links 包含以下功能：

1. **Explore** - 探索已发布策略
2. **My Policies** - 我的策略管理
3. **My Approval** - 我的审批管理
4. **Deviation** - 偏差请求
5. **File Viewer** - 文件查看器（仅超级用户）
6. **Approval Workflow** - 审批工作流配置（仅超级用户）
7. **AI Assistant** - AI 助手功能

## 重组后的文件结构

```
PolicyUserInterface/
├── PolicyUserInterfaceOptimized.tsx    # 主入口文件
├── explore/                            # 探索功能
│   └── PublishedPolicies.tsx          # 已发布策略列表
├── my-policies/                        # 我的策略功能
│   ├── MyPolicies.tsx                 # 我的策略列表
│   └── MyPoliciesDrawer.tsx           # 我的策略抽屉
├── my-approval/                        # 我的审批功能
│   ├── MyApproval.tsx                  # 我的审批列表
│   └── ApprovalDrawer.tsx              # 审批抽屉
├── deviation/                          # 偏差请求功能
│   └── DeviationDrawer.tsx             # 偏差请求抽屉
├── file-viewer/                        # 文件查看器功能
│   └── FileViewerDrawer.tsx            # 文件查看器抽屉
├── workflow-config/                    # 审批工作流配置功能
│   ├── ApprovalWorkflowConfig.tsx      # 审批工作流配置
│   └── ApprovalNodeConfig.tsx          # 审批节点配置
├── ai-assistant/                       # AI 助手功能
│   ├── AskAIPanel.tsx                  # AI 问答面板
│   └── CopilotSidebar.tsx              # AI 助手侧边栏
└── shared/                             # 共享组件和工具
    ├── QuickLinksPanel.tsx             # 快速链接面板
    ├── SearchBar.tsx                    # 搜索栏
    ├── FilePreviewModal.tsx            # 文件预览模态框
    ├── PolicyVersionHistory.tsx        # 策略版本历史
    ├── PolicyRevisionForm.tsx          # 策略修订表单
    └── hooks/                          # 共享 hooks
        ├── useOnDemandData.ts          # 按需数据加载
        ├── usePolicies.ts              # 策略数据管理
        └── usePolicyActions.ts         # 策略操作
```

## 功能文件夹详细说明

### 1. explore/ - 探索功能
- **PublishedPolicies.tsx**: 显示已发布的策略列表，支持筛选和搜索

### 2. my-policies/ - 我的策略功能
- **MyPolicies.tsx**: 显示用户创建的策略列表
- **MyPoliciesDrawer.tsx**: 我的策略抽屉组件

### 3. my-approval/ - 我的审批功能
- **MyApproval.tsx**: 显示需要用户审批的策略列表
- **ApprovalDrawer.tsx**: 审批抽屉组件

### 4. deviation/ - 偏差请求功能
- **DeviationDrawer.tsx**: 偏差请求表单抽屉

### 5. file-viewer/ - 文件查看器功能
- **FileViewerDrawer.tsx**: 文件管理查看器抽屉

### 6. workflow-config/ - 审批工作流配置功能
- **ApprovalWorkflowConfig.tsx**: 审批工作流配置主组件
- **ApprovalNodeConfig.tsx**: 审批节点配置组件

### 7. ai-assistant/ - AI 助手功能
- **AskAIPanel.tsx**: AI 问答面板
- **CopilotSidebar.tsx**: AI 助手侧边栏

### 8. shared/ - 共享组件和工具
- **QuickLinksPanel.tsx**: 快速链接导航面板
- **SearchBar.tsx**: 搜索栏组件
- **FilePreviewModal.tsx**: 文件预览模态框
- **PolicyVersionHistory.tsx**: 策略版本历史
- **PolicyRevisionForm.tsx**: 策略修订表单
- **hooks/**: 共享的自定义 hooks

## 更新的导入路径

### 主入口文件更新
```typescript
// PolicyUserInterfaceOptimized.tsx
import AskAIPanel from "./ai-assistant/AskAIPanel"
import ApprovalWorkflowConfig from "./workflow-config/ApprovalWorkflowConfig"
import CopilotSidebar from "./ai-assistant/CopilotSidebar"
import MyPolicies from "./my-policies/MyPolicies"
import PolicyRevisionForm from "./shared/PolicyRevisionForm"
import FilePreviewModal from "./shared/FilePreviewModal"
import PublishedPolicies from "./explore/PublishedPolicies"
import MyApproval from "./my-approval/MyApproval"
import QuickLinksPanel from "./shared/QuickLinksPanel"
import SearchBar from "./shared/SearchBar"
import DeviationDrawer from "./deviation/DeviationDrawer"
import FileViewerDrawer from "./file-viewer/FileViewerDrawer"
import { useOnDemandData } from "./shared/hooks/useOnDemandData"
import { usePolicyActions } from "./shared/hooks/usePolicyActions"
```

### 其他文件更新
```typescript
// my-policies/MyPoliciesDrawer.tsx
import MyPolicies from "./MyPolicies"

// my-approval/ApprovalDrawer.tsx
import PublishedPolicies from "../explore/PublishedPolicies"

// PolicyDetailModal.tsx
import PolicyVersionHistory from "../PolicyUserInterface/shared/PolicyVersionHistory"
```

## 重组优势

### 1. 功能模块化
- 每个 Quick Links 功能都有独立的文件夹
- 相关功能的组件集中在一起
- 符合"一个功能一个文件夹"的原则

### 2. 清晰的职责分离
- **功能文件夹**: 包含特定功能的组件
- **共享文件夹**: 包含跨功能使用的组件和工具
- **主入口文件**: 负责组合各个功能模块

### 3. 易于维护和扩展
- 修改特定功能时只需要关注对应文件夹
- 新功能可以独立添加到新的功能文件夹
- 共享组件统一管理，避免重复

### 4. 团队协作友好
- 不同开发者可以并行开发不同功能文件夹
- 减少代码冲突和合并问题
- 文件结构直观，新成员容易理解

### 5. 符合用户界面逻辑
- 文件结构与用户界面功能一一对应
- 开发者可以快速找到对应功能的代码
- 功能边界清晰，便于测试和调试

## 功能映射表

| Quick Links 按钮 | 功能文件夹 | 主要组件 | 说明 |
|-----------------|------------|----------|------|
| **Explore** | `explore/` | PublishedPolicies.tsx | 探索已发布策略 |
| **My Policies** | `my-policies/` | MyPolicies.tsx | 我的策略管理 |
| **My Approval** | `my-approval/` | MyApproval.tsx | 我的审批管理 |
| **Deviation** | `deviation/` | DeviationDrawer.tsx | 偏差请求 |
| **File Viewer** | `file-viewer/` | FileViewerDrawer.tsx | 文件查看器 |
| **Approval Workflow** | `workflow-config/` | ApprovalWorkflowConfig.tsx | 审批工作流配置 |
| **AI Assistant** | `ai-assistant/` | AskAIPanel.tsx + CopilotSidebar.tsx | AI 助手功能 |

## 注意事项

1. **导入路径**: 所有导入路径已更新，确保项目能正常运行
2. **功能完整性**: 重组过程中保持了所有功能的完整性
3. **向后兼容**: 主入口文件 `PolicyUserInterfaceOptimized.tsx` 保持不变
4. **类型安全**: 所有 TypeScript 类型定义保持完整
5. **共享组件**: 跨功能使用的组件统一放在 `shared/` 文件夹中

## 总结

通过这次按功能重组，PolicyUserInterface 现在具有了更加清晰和直观的文件结构。每个 Quick Links 功能都有独立的文件夹，符合"一个功能一个文件夹"的原则，大大提高了代码的组织性和可维护性。
