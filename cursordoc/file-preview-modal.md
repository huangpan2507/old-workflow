# File Preview Modal Implementation

## Overview
实现了文件预览弹窗功能，使用`react-doc-viewer`库来预览政策文件，替代了原来跳转到File Viewer界面的行为。

## Implementation Details

### 1. Library Installation
```bash
npm install react-doc-viewer
```
- 安装了`react-doc-viewer`库用于文档预览
- 支持多种文件格式（PDF, DOC, DOCX, PPT, PPTX, XLS, XLSX等）

### 2. FilePreviewModal Component
创建了新的文件预览弹窗组件：

#### Features
- **大尺寸弹窗**：使用`size="6xl"`和`maxW="90vw" maxH="90vh"`
- **文档预览**：使用`react-doc-viewer`进行文件预览
- **加载状态**：显示加载动画和提示
- **错误处理**：当没有文件时显示友好提示
- **响应式设计**：适配不同屏幕尺寸

#### Props Interface
```typescript
interface FilePreviewModalProps {
  isOpen: boolean
  onClose: () => void
  policy: {
    policy_name: string
    file_object_name?: string
    file_url?: string
  } | null
}
```

#### File URL Handling
```typescript
const getFileUrl = () => {
  if (policy.file_url) {
    return policy.file_url
  }
  if (policy.file_object_name) {
    return `/api/files/${policy.file_object_name}`
  }
  return null
}
```

### 3. PolicyUserInterface Integration

#### State Management
```typescript
const [selectedPolicyForPreview, setSelectedPolicyForPreview] = useState<Policy | null>(null)

// File preview modal
const {
  isOpen: isPreviewOpen,
  onOpen: onPreviewOpen,
  onClose: onPreviewClose,
} = useDisclosure()
```

#### Event Handlers
```typescript
function handlePreviewFile(policy: Policy) {
  setSelectedPolicyForPreview(policy)
  onPreviewOpen()
}

function handlePreviewClose() {
  onPreviewClose()
  setSelectedPolicyForPreview(null)
}
```

#### Modal Integration
```typescript
<FilePreviewModal
  isOpen={isPreviewOpen}
  onClose={handlePreviewClose}
  policy={selectedPolicyForPreview}
/>
```

## User Experience Improvements

### Before
- 点击眼睛按钮跳转到File Viewer界面
- 需要离开当前页面
- 用户体验不够流畅

### After
- 点击眼睛按钮弹出文件预览窗口
- 在当前页面内预览文件
- 可以快速关闭预览，回到政策列表
- 支持多种文件格式的预览

## Technical Features

### Document Viewer Configuration
```typescript
<DocViewer
  documents={[
    {
      uri: fileUrl,
      fileName: policy.file_object_name || policy.policy_name,
    }
  ]}
  pluginRenderers={DocViewerRenderers}
  config={{
    header: {
      disableHeader: true,
    },
    loadingRenderer: {
      showLoading: true,
      loadingComponent: <LoadingComponent />,
    },
  }}
  style={{
    height: '100%',
    width: '100%',
  }}
/>
```

### Supported File Types
- PDF documents
- Microsoft Word (DOC, DOCX)
- Microsoft PowerPoint (PPT, PPTX)
- Microsoft Excel (XLS, XLSX)
- Text files
- Images (PNG, JPG, GIF)

## Benefits
1. **无缝体验**：无需离开当前页面即可预览文件
2. **多格式支持**：支持常见的办公文档格式
3. **响应式设计**：适配不同设备和屏幕尺寸
4. **加载状态**：提供友好的加载提示
5. **错误处理**：优雅处理无文件的情况
6. **易于维护**：组件化设计，便于后续扩展

## Future Enhancements
- 添加文件下载功能到预览窗口
- 支持文件注释和标记
- 添加全屏预览模式
- 支持文件搜索功能
- 添加打印功能
