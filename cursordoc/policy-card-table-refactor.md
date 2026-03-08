# Policy Card to Table Refactor

## Overview
根据客户反馈，对政策卡片显示进行了重构，主要显示summary信息，并改为表格形式展示。

## Changes Made

### 1. PolicyCard Component (`PolicyCard.tsx`)
- **简化卡片内容**：移除了版本号、部门、标签、所有者等冗余信息
- **突出summary显示**：将summary显示行数从3行增加到6行，占据更多空间
- **调整卡片高度**：从280px减少到200px，更加紧凑
- **保留核心功能**：保留了查看、编辑、修订和下载等操作按钮

### 2. PolicyTable Component (`PolicyTable.tsx`)
- **优化表格结构**：将"Revision Type"列替换为"Summary"列
- **增强summary显示**：summary列显示最多3行，最大宽度300px
- **支持published模式**：添加了对"published"模式的支持
- **改进空状态消息**：为published模式添加了专门的空状态提示

### 3. PolicyUserInterface Component (`PolicyUserInterface.tsx`)
- **替换卡片为表格**：将Published Policies模式从卡片布局改为表格布局
- **统一显示方式**：默认模式和published模式都使用表格显示
- **移除未使用导入**：清理了PolicyCard的导入，因为不再使用

## Benefits
1. **信息聚焦**：主要显示summary，减少信息冗余
2. **空间利用**：表格形式更好地利用屏幕空间
3. **一致性**：所有政策显示都使用统一的表格格式
4. **可读性**：summary信息更加突出和易读

## Technical Details
- 保持了所有原有的功能（查看、下载、编辑等）
- 表格支持hover效果和响应式设计
- 空状态处理更加完善
- 代码结构更加清晰和可维护
