# AI Workbench - Debug Guide for Multiple Canvas Issue

## 已完成的优化

### ✅ 优化1：WorkbenchSidebar的Zustand订阅
**问题**: 订阅整个store导致任何store变化都触发WorkbenchSidebar重新渲染

**修复**:
```typescript
// 修复前
const { stickyNotes } = useWorkbenchStore()

// 修复后
const stickyNotes = useWorkbenchStore((state) => state.stickyNotes)
```

**效果**: WorkbenchSidebar只在`stickyNotes`变化时重新渲染，而不是每次store任何属性变化时都渲染。

---

### ✅ 优化2：WorkbenchSidebar的自定义比较函数
**问题**: 默认React.memo使用浅比较，props引用变化会导致重新渲染

**修复**:
```typescript
export default React.memo(WorkbenchSidebar, (prevProps, nextProps) => {
  // 只有这些props变化时才重新渲染
  return (
    prevProps.onCanvasSelect === nextProps.onCanvasSelect &&
    prevProps.w === nextProps.w
  )
})
```

**效果**: 只有在关键props真正变化时才重新渲染。

---

### ✅ 优化3：handleCanvasSelect已有useCallback
**检查结果**: `handleCanvasSelect`已经正确使用了`useCallback`，依赖数组为空，引用稳定。

```typescript
const handleCanvasSelect = React.useCallback((canvasId: string) => {
  setSelectedNoteId(canvasId)
}, [])
```

---

### ✅ 增强4：详细的调试日志系统

已在以下关键位置添加详细调试日志：

#### 1. CanvasDropZone组件
```typescript
🔵 CanvasDropZone component rendered - 组件渲染
🎯 Drop event triggered in CanvasDropZone - Drop事件触发
📍 Calculated drop position - 计算的drop位置
🔔 About to call onApplicationDrop callback - 即将调用回调
✓ onApplicationDrop callback completed - 回调完成
```

#### 2. handleApplicationDrop函数
```typescript
🔥 handleApplicationDrop called - 函数被调用
🚫 Duplicate drop call prevented - 重复调用被阻止
📝 About to call addNote with canvas - 即将调用addNote
✅ addNote called successfully - addNote调用成功
```

#### 3. workbenchStore.ts的addNote函数
```typescript
🟢 addNote function called in store - store中的addNote被调用
🆔 Generated note ID - 生成的note ID
📡 Adding note to Yjs document - 添加到Yjs文档
💾 Adding note to local state - 添加到本地状态
🏁 addNote completed, current notes count - 完成，当前notes数量
```

---

## 🔍 调试策略

### 步骤1：观察调试日志的完整链路

当你拖拽一个application到canvas时，控制台应该显示以下**完整且唯一**的日志链路：

```
1. 🚀 Drag started (ApplicationCard)
2. 🎯 Drop event triggered in CanvasDropZone
3. 📍 Calculated drop position
4. 🔔 About to call onApplicationDrop callback
5. ✓ onApplicationDrop callback completed
6. 🔥 handleApplicationDrop called
7. 📝 About to call addNote with canvas
8. 🟢 addNote function called in store (+ stackTrace)
9. 🆔 Generated note ID
10. 📡 Adding note to Yjs document (或 💾 Adding note to local state)
11. 🏁 addNote completed, current notes count
12. ✅ addNote called successfully
13. 🏁 Drag ended
```

### 步骤2：识别异常模式

#### 模式A：Drop事件多次触发
如果看到多个`🎯 Drop event triggered`，说明：
- `useDrop` hook被创建了多次
- CanvasDropZone组件可能在不必要的时候重新渲染
- **查看**: `🔵 CanvasDropZone component rendered`的次数

#### 模式B：addNote被多次调用
如果看到多个`🟢 addNote function called in store`，说明：
- `handleApplicationDrop`被多次调用，或
- `addNote`函数被其他地方调用
- **查看**: 每个`🟢`日志中的`stackTrace`，找出调用来源

#### 模式C：Yjs同步导致重复
如果看到：
```
🟢 addNote called 1 time
📡 Adding note to Yjs document
🏁 current notes count: 1
... (等待几秒)
🏁 current notes count: 2, 3, 4...
```
说明Yjs协同可能有问题，服务器返回了多个副本。

---

## 🎯 关键检查点

### 检查点1：CanvasDropZone渲染次数
**预期**: 初始渲染1次，之后只在关键props变化时渲染
**异常**: 每次拖拽都重新渲染
**解决**: 检查`onApplicationDrop`的引用是否稳定（已使用useCallback）

### 检查点2：Drop事件触发次数
**预期**: 每次拖拽触发1次
**异常**: 触发多次
**解决**: 
- 检查useDrop的依赖数组
- 检查CanvasDropZone是否被多次实例化

### 检查点3：addNote调用次数
**预期**: 每次drop调用1次
**异常**: 调用多次
**解决**:
- 查看stackTrace找出所有调用来源
- 检查是否有其他地方也在调用addNote

### 检查点4：防重复机制
**预期**: 短时间内重复drop被`lastDropCallRef`阻止
**异常**: 看到`🚫 Duplicate drop call prevented`但仍然生成多个画布
**解决**: 说明不是同一个drop调用的重复，而是多个独立的drop事件

---

## 🧪 测试步骤

### 测试1：单次拖拽
1. 刷新页面，清空控制台
2. 拖拽一个application到canvas
3. 观察控制台日志
4. **记录**:
   - `🎯 Drop event triggered`出现几次？
   - `🟢 addNote function called`出现几次？
   - SPACES区域显示几个canvas？

### 测试2：删除后再拖拽
1. 删除所有canvas
2. 清空控制台
3. 再次拖拽同一个application
4. **记录**:
   - 日志模式是否一致？
   - 生成的canvas数量是否增加？

### 测试3：不同application测试
1. 清空所有canvas和控制台
2. 拖拽不同的application（如果有多个）
3. **记录**:
   - 是否所有application都有同样的问题？
   - 某些application是否正常？

### 测试4：WebSocket连接测试
观察控制台是否有WebSocket错误：
```
❌ WebSocket connection to 'ws://localhost:8001/...' failed
```
如果有WebSocket错误，Yjs协同可能失败，addNote会使用本地状态。

---

## 🔧 可能的根本原因

### 原因1：Yjs Observer触发循环
**症状**: 
- `addNote`调用1次
- 但`stickyNotes`数量持续增加
- 看到多个`🏁 current notes count`日志

**原因**: Yjs的observer可能在监听到变化后触发额外的更新

**位置**: `workbenchStore.ts`的`initializeCollaboration`中的observer

**检查**:
```typescript
yNotes.observe(() => {
  const notesArray: StickyNote[] = []
  yNotes.forEach((note: any, id: string) => {
    notesArray.push({ ...note, id })
  })
  set({ stickyNotes: notesArray })  // 这里可能触发额外更新
})
```

### 原因2：useDrop工厂函数依赖变化
**症状**:
- 看到多次`🎯 Drop event triggered`
- 每次drop位置略有不同

**原因**: `useDrop`的依赖数组中的值频繁变化，导致hook重新创建

**检查**: CanvasDropZone的依赖
```typescript
[onApplicationDrop, stageRef, stagePos.x, stagePos.y, zoom]
```

### 原因3：React Strict Mode双重调用
**症状**:
- 开发环境下看到所有日志都是2的倍数
- 生产环境正常

**原因**: React 18的Strict Mode会在开发环境双重调用某些函数

**解决**: 这是正常行为，生产环境不会有此问题

### 原因4：事件冒泡或多个Drop Zone
**症状**:
- 看到多个不同的`🎯 Drop event triggered`日志
- stackTrace显示不同的调用栈

**原因**: 可能有多个drop zone实例，或者事件冒泡触发了多次

**检查**: 
- 搜索代码中是否有多个`<CanvasDropZone>`
- 检查是否有嵌套的drop zone

---

## 📊 预期vs实际日志对比

### ✅ 正常情况（预期）
```
🚀 Drag started: {applicationId: 15, ...}
🎯 Drop event triggered in CanvasDropZone: {applicationId: 15, ...}
📍 Calculated drop position: {x: 456, y: 789}
🔔 About to call onApplicationDrop callback
✓ onApplicationDrop callback completed
🔥 handleApplicationDrop called: {applicationId: 15, ...}
📝 About to call addNote with canvas
🟢 addNote function called in store
🆔 Generated note ID: note-1234567890-abc123 Name: App Canvas 1
📡 Adding note to Yjs document
🏁 addNote completed, current notes count: 1
✅ addNote called successfully
🏁 Drag ended: {applicationId: 15, ...}
```
**结果**: SPACES显示1个canvas ✅

### ❌ 异常情况1：Drop多次触发
```
🚀 Drag started: {applicationId: 15, ...}
🎯 Drop event triggered in CanvasDropZone: {applicationId: 15, ...}
🎯 Drop event triggered in CanvasDropZone: {applicationId: 15, ...}  ❌ 重复
🎯 Drop event triggered in CanvasDropZone: {applicationId: 15, ...}  ❌ 重复
...
```
**结果**: SPACES显示3个canvas ❌

### ❌ 异常情况2：addNote多次调用
```
🎯 Drop event triggered in CanvasDropZone: (1次) ✅
🔥 handleApplicationDrop called: (1次) ✅
🟢 addNote function called in store (1次) ✅
🆔 Generated note ID: note-xxx-1
🏁 current notes count: 1
(pause)
🟢 addNote function called in store (2次) ❌ 为什么？
🆔 Generated note ID: note-xxx-2
🏁 current notes count: 2
🟢 addNote function called in store (3次) ❌
🆔 Generated note ID: note-xxx-3
🏁 current notes count: 3
```
**结果**: SPACES显示3个canvas ❌
**查看**: 第2、3次的stackTrace

---

## 🚨 下一步行动

### 立即执行：
1. **清空浏览器控制台**
2. **拖拽一个application**
3. **复制完整的控制台日志**
4. **截图SPACES区域显示的canvas数量**
5. **特别关注**:
   - `🎯 Drop event triggered`出现几次
   - `🟢 addNote function called`出现几次
   - 每个`🟢`后面的stackTrace内容

### 分析日志：
- 如果`🎯`出现多次 → CanvasDropZone或useDrop有问题
- 如果`🟢`出现多次但`🎯`只1次 → handleApplicationDrop或Yjs有问题
- 如果日志都正常但SPACES显示多个 → Zustand状态管理或React渲染有问题

---

## 📝 文件修改记录

### WorkbenchSidebar.tsx
- ✅ 改用选择器订阅`stickyNotes`
- ✅ 添加自定义React.memo比较函数

### CanvasDropZone.tsx
- ✅ 添加详细调试日志
- ✅ 添加stackTrace追踪

### workbench.tsx
- ✅ 在handleApplicationDrop中添加日志

### workbenchStore.ts
- ✅ 在addNote函数中添加详细日志和stackTrace
- ✅ 追踪Yjs vs 本地状态的选择

---

**状态**: ✅ 调试系统已部署，等待测试结果

**建议**: 请执行上述测试步骤，并提供完整的控制台日志截图，这将帮助我们精确定位问题根源。




