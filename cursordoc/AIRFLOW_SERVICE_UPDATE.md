# AirflowService 更新文档

## 概述

本次更新将 `airflow_service.py` 中的 `dag_run_conf` 构建逻辑与 `utils.py` 中的实现保持一致，确保两个模块使用相同的文件处理和元数据提取逻辑。

## 修改内容

### 1. 新增导入模块

```python
import io
import os
import tempfile
from datetime import datetime
from docx import Document
import pdfplumber
```

### 2. 新增方法

#### `_get_file_metadata(content: bytes, filename: str, file_content: str = "") -> str`

从 `utils.py` 移植的文件元数据提取方法，用于获取文件的修改时间信息。

**功能：**
- 对于 Word 文档 (.docx)：尝试从文档属性中提取修改时间或创建时间
- 对于 PDF 文件：尝试从 PDF 元数据中提取修改时间或创建时间
- 如果无法从文件内容获取，则创建临时文件获取文件系统时间
- 如果所有方法都失败，返回当前日期

#### `_extract_text_content(content: bytes, filename: str) -> str`

从 `utils.py` 移植的文本内容提取方法，用于从文件中提取文本内容。

**功能：**
- 对于 Word 文档 (.docx)：提取所有段落的文本
- 对于 PDF 文件：提取所有页面的文本
- 对于其他文件类型：使用 UTF-8 解码
- 如果解析失败，使用 UTF-8 解码作为后备方案

### 3. 修改 `trigger_dag_run` 方法

#### 修改前：
```python
dag_run_conf = {
    "file_ingestion_record_id": file_ingestion_record_id,
    "file_name": file_name,
    "file_content": file_content.decode('utf-8', errors='ignore')
}
```

#### 修改后：
```python
# 提取文本内容
text = self._extract_text_content(file_content, file_name)

# 获取文件元数据
file_updated_at = self._get_file_metadata(file_content, file_name, text)

# 构建 DAG 运行配置，与 utils.py 保持一致
dag_run_conf = {
    "file_ingestion_record_id": file_ingestion_record_id,  # 新增参数，需要保留
    "file_name": file_name,
    "file_content": text,  # 使用解析后的文本内容
    "file_updated_at": file_updated_at  # 新增参数
}
```

## 关键改进

### 1. 文件内容处理一致性

- **之前**：直接使用 `file_content.decode('utf-8', errors='ignore')`，可能无法正确解析 Word 和 PDF 文件
- **现在**：使用专门的文本提取方法，能够正确解析 Word 和 PDF 文件的文本内容

### 2. 文件元数据提取

- **新增**：`file_updated_at` 参数，提供文件的修改时间信息
- **来源**：优先从文件内容中提取，其次从文件系统获取，最后使用当前时间

### 3. 参数结构一致性

确保 `dag_run_conf` 的结构与 `utils.py` 中的实现保持一致：

| 参数 | 来源 | 说明 |
|------|------|------|
| `file_ingestion_record_id` | 新增 | 文件记录ID，用于关联数据库记录 |
| `file_name` | 保持 | 文件名 |
| `file_content` | 改进 | 解析后的文本内容，而非原始字节 |
| `file_updated_at` | 新增 | 文件更新时间 |

## 依赖要求

确保 `pyproject.toml` 中包含以下依赖：

```toml
dependencies = [
    "python-docx",  # Word 文档处理
    "pdfplumber",   # PDF 文档处理
    # ... 其他依赖
]
```

## 测试

使用提供的测试脚本验证修改：

```bash
cd backend
python scripts/test_airflow_service_update.py
```

## 兼容性

- ✅ 保持向后兼容性
- ✅ 保留 `file_ingestion_record_id` 参数
- ✅ 与 `utils.py` 中的实现保持一致
- ✅ 支持相同的文件类型（.docx, .pdf）

## 注意事项

1. **性能影响**：新增的文本提取和元数据提取可能增加处理时间
2. **错误处理**：所有新增方法都包含完善的错误处理机制
3. **日志记录**：保留了原有的日志记录功能
4. **依赖管理**：确保所有必要的依赖都已正确安装 