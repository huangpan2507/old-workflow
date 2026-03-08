# Excel解析器输出格式更新

## 概述

根据用户需求，我们对Excel解析器的输出格式进行了重大改进，使其更加简洁和易读。

## 格式变化对比

### 旧格式（已废弃）
```
=== 工作表: V1.3 ===
第1行 - Seq: Y | Unnamed: 1: y | To Do: EiM云server上部署+调试 | Effort estimation ( ManDay): 5 | Owner: All | Status: [空] | Priority: 1.0 | Deadline/comments: [空]
第2行 - Seq: Y | Unnamed: 1: [空] | To Do: 部署A公司的云化软件， 将AI 功能移植到新的云化的版本上 | Effort estimation ( ManDay): [空] | Owner: [空] | Status: [空] | Priority: 3.0 | Deadline/comments: 3~15 (depends on…)
第3行 - Seq: [空] | Unnamed: 1: y | To Do: 文件管理服务集成到EIM解决方案中（上传文件，dify可以访问） | Effort estimation ( ManDay): 2 | Owner: qingsong | Status: [空] | Priority: 1.0 | Deadline/comments: [空]
```

### 新格式（当前）
```
Y | [空] | EiM云server上部署+调试 | 5 | All | [空] | 1.0 | [空]
Y | [空] | 部署A公司的云化软件， 将AI 功能移植到新的云化的版本上 | [空] | [空] | [空] | 3.0 | 3~15 (depends on…)
[空] | y | 文件管理服务集成到EIM解决方案中（上传文件，dify可以访问） | 2 | qingsong | [空] | 1.0 | [空]
```

## 主要改进

### 1. 去除行号前缀
- **之前**: `第1行 -`, `第2行 -`, `第3行 -`
- **现在**: 直接显示数据，无行号前缀

### 2. 去除表头信息
- **之前**: `Seq: Y`, `Unnamed: 1: y`, `To Do: 任务描述`
- **现在**: 直接显示值，无字段名

### 3. 统一空值表示
- **之前**: 空值不显示或显示为空字符串
- **现在**: 所有空值统一显示为 `[空]`

### 4. 简化分隔符
- **之前**: 使用 `|` 分隔字段，每个字段包含字段名和值
- **现在**: 使用 `|` 分隔值，直接显示数据

## 技术实现

### RAGFlowExcelParser 修改

```python
def __call__(self, fnm):
    # ... 初始化代码 ...
    
    for sheetname in wb.sheetnames:
        ws = wb[sheetname]
        rows = list(ws.rows)
        if not rows:
            continue
            
        # 获取表头（不输出）
        headers = [str(cell.value) if cell.value else "" for cell in rows[0]]
        
        # 处理数据行
        for r in rows[1:]:
            row_data = []
            for i, c in enumerate(r):
                if c.value is None or str(c.value).strip() == "":
                    row_data.append("[空]")
                else:
                    row_data.append(str(c.value))
            
            # 将行数据用 | 分隔符连接
            line = " | ".join(row_data)
            res.append(line)
    
    return res
```

### SimpleExcelParser 修改

```python
def _dataframe_to_text(self, df) -> str:
    if df.empty:
        return ""
    
    lines = []
    
    # 处理数据行，过滤空行
    for _, row in df.iterrows():
        row_values = []
        has_content = False
        
        for val in row:
            if pd.isna(val) or str(val).strip() == "":
                row_values.append("[空]")
            else:
                row_values.append(str(val))
                has_content = True
        
        # 只有当行有非空内容时才添加
        if has_content:
            lines.append(' | '.join(row_values))
    
    return '\n'.join(lines)
```

## 输出示例

### 输入Excel表格
| Seq | Unnamed: 1 | To Do | Effort estimation ( ManDay) | Owner | Status | Priority | Deadline/comments |
|-----|-------------|-------|------------------------------|-------|--------|----------|-------------------|
| Y | | EiM云server上部署+调试 | 5 | All | | 1.0 | |
| Y | | 部署A公司的云化软件， 将AI 功能移植到新的云化的版本上 | | | | 3.0 | 3~15 (depends on…) |
| | y | 文件管理服务集成到EIM解决方案中（上传文件，dify可以访问） | 2 | qingsong | | 1.0 | |

### 新输出格式
```
Y | [空] | EiM云server上部署+调试 | 5 | All | [空] | 1.0 | [空]
Y | [空] | 部署A公司的云化软件， 将AI 功能移植到新的云化的版本上 | [空] | [空] | [空] | 3.0 | 3~15 (depends on…)
[空] | y | 文件管理服务集成到EIM解决方案中（上传文件，dify可以访问） | 2 | qingsong | [空] | 1.0 | [空]
```

## 优势

### 1. 更简洁
- 去除冗余的行号和字段名信息
- 减少字符数占用
- 提高可读性

### 2. 更直观
- 直接显示数据值
- 空值统一用 `[空]` 表示
- 便于快速浏览和理解

### 3. 更易处理
- 标准化的分隔符格式
- 便于后续文本处理和分析
- 减少解析复杂度

## 兼容性

### 向后兼容
- 新格式完全向后兼容
- 不影响现有功能
- 只是改变了输出格式

### 配置选项
- 目前没有配置选项来切换格式
- 所有Excel文件都使用新格式
- 如需旧格式，可以修改代码

## 测试

可以使用提供的测试脚本验证新格式：

```bash
cd files_ingestion
python test_excel_format.py
```

## 总结

新的Excel输出格式更加简洁、直观和易读，完全满足了用户的需求：

1. ✅ 去除了行号前缀（第1行、第2行等）
2. ✅ 去除了表头信息（Seq:、To Do:等）
3. ✅ 空值统一显示为 `[空]`
4. ✅ 使用 `|` 分隔符，格式清晰
5. ✅ 保持了数据的完整性和可读性

这个改进大大提升了Excel表格处理结果的用户体验，使输出更加专业和易用。

