# Excel解析器输出格式修复说明

## 问题分析

用户反馈Excel解析器的输出格式仍然显示旧的格式，包含行号前缀和表头信息。经过排查，发现问题出现在以下位置：

## 根本原因

系统使用的是`files_ingestion/standalone_document_parser/parsers/ragflow_parsers.py`中的`RAGFlowExcelParser`，而不是我们之前修改的`files_ingestion/deepdoc/parser/excel_parser.py`中的版本。

## 修改位置

### 主要修改文件
**`files_ingestion/standalone_document_parser/parsers/ragflow_parsers.py`**

这个文件中的`RAGFlowExcelParser`类被`DocumentProcessor`使用，当`use_ragflow_parsers=True`时会调用这个解析器。

### 修改内容

1. **替换了`df.to_string(index=False)`方法**
   - **之前**: 使用pandas的`to_string()`方法，会生成包含行号的输出
   - **现在**: 使用自定义的`_dataframe_to_text()`方法

2. **新增了`_dataframe_to_text()`方法**
   - 实现了新的输出格式
   - 去除了行号前缀
   - 去除了表头信息
   - 空值统一显示为`[空]`
   - 使用`|`分隔符

## 修改后的代码

```python
def __call__(self, file_data: BytesIO) -> List[Tuple[str, str]]:
    """解析Excel文件"""
    try:
        import pandas as pd
        
        df = pd.read_excel(file_data, sheet_name=None)
        chunks = []
        
        for sheet_name, sheet_df in df.items():
            # 转换为文本，使用新的格式
            text = self._dataframe_to_text(sheet_df)
            if text.strip():  # 只有当有内容时才添加
                chunks.append((text, f"工作表: {sheet_name}"))
        
        return chunks
    except Exception as e:
        logger.error(f"Excel解析失败: {e}")
        return []

def _dataframe_to_text(self, df) -> str:
    """将DataFrame转换为文本，使用新的格式"""
    if df.empty:
        return ""
    
    lines = []
    
    # 处理数据行，过滤空行
    for _, row in df.iterrows():
        # 检查该行是否有实际内容
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

## 验证步骤

### 1. 确认修改已保存
检查文件`files_ingestion/standalone_document_parser/parsers/ragflow_parsers.py`中的`RAGFlowExcelParser`类是否已更新。

### 2. 重启服务
由于修改了Python代码，需要重启files_ingestion服务：

```bash
# 如果使用Docker Compose
docker-compose restart files-ingestion

# 或者重启整个项目
docker-compose down
docker-compose up -d
```

### 3. 测试API接口
使用以下命令测试Excel解析：

```bash
curl -X POST "http://localhost:8800/extract_text" \
  -H 'accept: application/json' \
  -H 'Content-Type: multipart/form-data' \
  -F 'file=@EIM_case_simple_5.xlsx;type=application/vnd.openxmlformats-officedocument.spreadsheetml.sheet' \
  -F 'enable_ocr=true'
```

### 4. 检查输出格式
新的输出格式应该是：

```
=== 工作表: V1.3 ===
Y | [空] | EiM云server上部署+调试 | 5 | All | [空] | 1.0 | [空]
Y | [空] | 部署A公司的云化软件， 将AI 功能移植到新的云化的版本上 | [空] | [空] | [空] | 3.0 | 3~15 (depends on…)
[空] | y | 文件管理服务集成到EIM解决方案中（上传文件，dify可以访问） | 2 | qingsong | [空] | 1.0 | [空]
```

而不是之前的：

```
=== 工作表: V1.3 ===
第1行 - Seq: Y | Unnamed: 1: y | To Do: EiM云server上部署+调试 | Effort estimation ( ManDay): 5 | Owner: All | Status: [空] | Priority: 1.0 | Deadline/comments: [空]
第2行 - Seq: Y | Unnamed: 1: [空] | To Do: 部署A公司的云化软件， 将AI 功能移植到新的云化的版本上 | Effort estimation ( ManDay): [空] | Owner: [空] | Status: [空] | Priority: 3.0 | Deadline/comments: 3~15 (depends on…)
```

## 故障排除

### 如果修改仍未生效

1. **检查文件路径**: 确认修改的是正确的文件
2. **检查服务重启**: 确认服务已经重启
3. **检查缓存**: 某些情况下可能需要清除Python缓存
4. **检查日志**: 查看服务日志是否有错误信息

### 清除Python缓存

```bash
# 在files_ingestion目录下
find . -name "*.pyc" -delete
find . -name "__pycache__" -type d -exec rm -rf {} +
```

## 总结

Excel解析器输出格式问题已经修复，主要修改了`files_ingestion/standalone_document_parser/parsers/ragflow_parsers.py`文件中的`RAGFlowExcelParser`类。

修改完成后，需要重启files_ingestion服务，然后测试API接口验证新的输出格式是否符合预期。

