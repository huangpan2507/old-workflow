# Foundation 项目故障排除指南

## 目录
- [实际遇到的问题及解决方案](#实际遇到的问题及解决方案)
- [常见错误代码及解决方案](#常见错误代码及解决方案)

---

## 实际遇到的问题及解决方案

### 1. RAGFlow OCR模块导入失败问题

**问题描述**：在运行 dev 模式下的 foundation 项目时，前端界面上传 .jpeg 图片后，`foundation-files-ingestion-1` 容器日志显示各种模块导入失败错误。

**问题演进过程**：

#### 第一阶段：deepdoc 模块缺失
**错误现象**：
```
RAGFlow OCR模块导入失败: No module named 'deepdoc'
```

**根本原因**：Dockerfile 中的 `COPY . .` 指令只会复制 Dockerfile 所在目录（`files_ingestion/standalone_document_parser/`）下的文件，但 `deepdoc` 模块位于上级目录 `files_ingestion/deepdoc/`，因此容器内无法找到该模块。

**解决方案**：
1. **修改 Dockerfile**：调整 build context 和 COPY 指令
2. **修改 docker-compose.yml**：更新 build context 路径

**具体修改**：

**文件**：`files_ingestion/standalone_document_parser/Dockerfile`
```dockerfile
# 修改前：
# COPY . .

# 修改后：
# 复制deepdoc目录（RAGFlow核心模块）
COPY deepdoc/ ./deepdoc/

# 复制standalone_document_parser目录下的所有文件
COPY standalone_document_parser/ .
```

**文件**：`docker-compose.yml`
```yaml
# 修改前：
build:
  context: ./files_ingestion/standalone_document_parser
  dockerfile: Dockerfile

# 修改后：
build:
  context: ./files_ingestion
  dockerfile: standalone_document_parser/Dockerfile
```

#### 第二阶段：api 模块缺失
**错误现象**：
```
RAGFlow OCR模块导入失败: No module named 'api'
```

**根本原因**：`deepdoc` 模块依赖 `api` 模块，但 Dockerfile 中没有复制 `api/` 目录。

**解决方案**：在 Dockerfile 中添加 api 目录的复制

**具体修改**：
```dockerfile
# 复制deepdoc目录（RAGFlow核心模块）
COPY deepdoc/ ./deepdoc/

# 复制api目录（RAGFlow工具模块）
COPY api/ ./api/

# 复制standalone_document_parser目录下的所有文件
COPY standalone_document_parser/ .
```

#### 第三阶段：rag 模块缺失
**错误现象**：
```
RAGFlow OCR模块导入失败: No module named 'rag'
```

**根本原因**：`deepdoc` 模块还依赖 `rag` 模块，但 Dockerfile 中没有复制 `rag/` 目录。

**解决方案**：在 Dockerfile 中添加 rag 目录的复制

**具体修改**：
```dockerfile
# 复制deepdoc目录（RAGFlow核心模块）
COPY deepdoc/ ./deepdoc/

# 复制api目录（RAGFlow工具模块）
COPY api/ ./api/

# 复制rag目录（RAGFlow核心功能模块）
COPY rag/ ./rag/

# 复制standalone_document_parser目录下的所有文件
COPY standalone_document_parser/ .
```

#### 第四阶段：Python 依赖包缺失
**错误现象**：
```
RAGFlow OCR模块导入失败: No module named 'shapely'
```

**根本原因**：缺少 Python 依赖包，`shapely` 和 `pyclipper` 等包没有在 `requirements_cpu.txt` 中定义。

**解决方案**：在依赖文件中添加缺失的包

**具体修改**：

**文件**：`files_ingestion/standalone_document_parser/requirements_cpu.txt`
```txt
# OCR和图像处理（CPU版本）
pytesseract>=0.3.10        # OCR文字识别
opencv-python-headless>=4.8.0  # 图像处理（无GUI，CPU版本）
onnxruntime>=1.15.0        # ONNX模型推理（CPU版本，自动选择）
shapely>=2.0.0             # 几何计算（OCR后处理使用）
pyclipper>=1.3.0           # 多边形裁剪和偏移（OCR后处理使用）
```

#### 第五阶段：staging 模式配置同步
**问题描述**：dev 模式和 staging 模式的 `files-ingestion` 服务配置不一致。

**根本原因**：只修改了 `docker-compose.yml`，没有同步修改 `docker-compose.staging.yml`。

**解决方案**：同步修改 staging 配置文件

**具体修改**：

**文件**：`docker-compose.staging.yml`
```yaml
# 修改前：
build:
  context: ./files_ingestion/standalone_document_parser
  dockerfile: Dockerfile

# 修改后：
build:
  context: ./files_ingestion
  dockerfile: standalone_document_parser/Dockerfile
```

**验证步骤**：

1. **重新构建镜像**：
```bash
docker-compose build files-ingestion
```

2. **重启服务**：
```bash
docker-compose restart files-ingestion
```

3. **检查容器日志**：
```bash
docker logs foundation-files-ingestion-1 --tail=20
```

4. **测试 OCR 功能**：上传 .jpeg 图片，确认无模块导入错误

**技术要点**：

- **Docker build context**：决定了 COPY 指令的相对路径起点
- **Python 模块导入路径**：需要与实际文件结构保持一致
- **容器化部署**：需要确保所有依赖模块都被正确复制到容器内
- **配置同步**：dev 模式和 staging 模式应保持配置一致性
- **依赖管理**：Python 依赖包管理同样重要，需要确保所有必需的包都被安装

**预防措施**：

1. **模块依赖分析**：在移植 RAGFlow 模块时，先分析完整的依赖关系
2. **Docker 配置检查**：确保 build context 和 COPY 指令正确配置
3. **依赖包管理**：定期检查并更新 Python 依赖包列表
4. **配置同步**：修改配置后，确保所有环境配置文件都同步更新

**相关文件**：

- `files_ingestion/standalone_document_parser/Dockerfile`
- `files_ingestion/standalone_document_parser/requirements_cpu.txt`
- `docker-compose.yml` (files-ingestion 服务配置)
- `docker-compose.staging.yml` (files-ingestion 服务配置)
- `files_ingestion/standalone_document_parser/vision/ocr.py`
- `files_ingestion/standalone_document_parser/vision/postprocess.py`
- `files_ingestion/deepdoc/` 目录
- `files_ingestion/api/` 目录
- `files_ingestion/rag/` 目录

### 2. Excel解析器修改未生效问题

**问题描述**：修改Excel解析器代码后，重新构建并启动容器，使用files_ingestion FastAPI接口测试，输出格式仍然保持旧格式，修改未生效。

**问题现象**：
- 修改了`RAGFlowExcelParser._dataframe_to_text`方法
- 重新构建并启动容器：`./bin/stop-dev.sh  && ./bin/start-dev.sh`
- API测试结果仍然是旧格式：`第1行 - Seq: Y | Unnamed: 1: y | To Do: ...`
- 期望的新格式：`Y | y | EiM云server上部署+调试 | 5 | All | [空] | 1.0 | [空]`

**根本原因分析**：

#### 第一阶段：修改了错误的文件
- **错误假设**：认为修改`files_ingestion/deepdoc/parser/excel_parser.py`就能生效
- **实际情况**：Docker环境中实际使用的是`files_ingestion/standalone_document_parser/parsers/ragflow_parsers.py`

#### 第二阶段：pandas导入错误
- **错误现象**：日志显示`name 'pd' is not defined`
- **根本原因**：`_dataframe_to_text`方法中pandas没有正确导入
- **影响**：`__call__`方法异常，系统回退到HTML格式

#### 第三阶段：数据处理流程问题
- **错误现象**：`__call__`方法工作正常，但API输出仍是旧格式
- **根本原因**：`extract_text_only`方法中的表格处理逻辑会重新格式化数据
- **具体问题**：`_convert_html_table_to_structured_text`方法会添加"第X行 - "前缀

#### 第四阶段：表头处理问题
- **错误现象**：第一行表头显示`Unnamed: X`而不是`[空]`
- **根本原因**：pandas自动生成的列名没有被正确处理

**解决方案**：

#### 1. 修复pandas导入问题
**文件**：`files_ingestion/standalone_document_parser/parsers/ragflow_parsers.py`

```python
def _dataframe_to_text(self, df) -> str:
    """将DataFrame转换为文本，使用新的格式"""
    try:
        import pandas as pd  # 在方法内部导入pandas
        
        if df.empty:
            return ""
        
        lines = []
        
        # 首先添加表头行，处理空值
        headers = []
        for col in df.columns:
            if pd.isna(col) or str(col).strip() == "" or str(col).startswith('Unnamed:'):
                headers.append("[空]")
            else:
                headers.append(str(col))
        
        header_line = ' | '.join(headers)
        lines.append(header_line)
        
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
            
            if has_content:
                lines.append(' | '.join(row_values))
        
        return '\n'.join(lines)
    except Exception as e:
        logger.error(f"DataFrame转文本失败: {e}")
        return ""
```

#### 2. 修复数据处理流程
**文件**：`files_ingestion/standalone_document_parser/document_processor.py`

```python
def extract_text_only(self, file_path: Union[str, bytes], **kwargs) -> str:
    """仅提取文档的纯文本内容"""
    result = self.process_document(file_path, **kwargs)
    
    text_content = []
    
    # 提取文本分块
    if 'text_chunks' in result:
        text_content.extend(result['text_chunks'])
    
    # 处理表格内容
    if 'tables' in result:
        for table in result['tables']:
            if isinstance(table, str):
                # 检查是否已经是新格式（不包含"第X行 - "前缀）
                if not table.startswith('第') or '行 - ' not in table:
                    # 直接使用新格式的数据，不重新格式化
                    text_content.append(table)
                else:
                    # 如果是旧格式，使用结构化表格转换
                    structured_text = self._convert_html_table_to_structured_text(table)
                    if structured_text:
                        text_content.append(structured_text)
    
    # 返回完整的文档字符串
    full_text = '\n'.join(text_content)
    import re
    full_text = re.sub(r'\n\s*\n', '\n\n', full_text)
    full_text = re.sub(r'[ \t]+', ' ', full_text)
    
    return full_text.strip()
```

#### 3. 修复调用链优先级
**文件**：`files_ingestion/standalone_document_parser/document_processor.py`

```python
def _process_excel(self, filename: str, binary_data: bytes, **kwargs) -> Dict[str, Any]:
    """处理Excel文件"""
    parser = self.parsers['excel']
    
    logger.info(f"开始处理Excel文件: {filename}")
    logger.info(f"使用的解析器类型: {type(parser).__name__}")
    
    try:
        # 优先使用我们修改的__call__方法，获取新的格式
        logger.info("尝试使用__call__方法解析Excel...")
        sections = parser(BytesIO(binary_data))
        tables = [section[0] for section in sections if section[0].strip()]
        
        logger.info(f"__call__方法解析结果: {len(tables)} 个表格")
        if tables:
            logger.info(f"第一个表格内容预览: {tables[0][:200]}...")
        
        # 如果新格式没有内容，再尝试HTML格式作为备用
        if not tables:
            logger.info("新格式无内容，尝试HTML格式...")
            html_tables = parser.html(BytesIO(binary_data))
            tables = html_tables if isinstance(html_tables, list) else [html_tables]
            
    except Exception as e:
        logger.warning(f"Excel解析失败，错误详情: {e}")
        # 降级到HTML格式
        try:
            html_tables = parser.html(BytesIO(binary_data))
            tables = html_tables if isinstance(html_tables, list) else [html_tables]
        except Exception as e2:
            logger.error(f"Excel HTML解析也失败: {e2}")
            tables = []
    
    return {
        'type': 'excel',
        'text_chunks': [],
        'tables': tables,
        'images': 0,
        'total_chunks': len(tables)
    }
```

**验证步骤**：

1. **检查日志确认解析器类型**：
```bash
docker logs foundation-files-ingestion-1 | grep "Excel解析器类型"
# 应该显示：RAGFlowExcelParser
```

2. **检查__call__方法是否被调用**：
```bash
docker logs foundation-files-ingestion-1 | grep "__call__方法解析结果"
# 应该显示：__call__方法解析结果: X 个表格
```

3. **检查输出格式**：
```bash
# API测试应该返回新格式：
# [空] | y | EiM云server上部署+调试 | 5 | All | [空] | 1.0 | [空]
# 而不是旧格式：
# 第1行 - Seq: Y | Unnamed: 1: y | To Do: EiM云server上部署+调试...
```

**关键调试技巧**：

1. **确认修改的文件路径**：确保修改的是Docker容器中实际使用的文件
2. **检查导入错误**：pandas等依赖库的导入问题会导致方法异常
3. **验证调用链**：确保修改的方法在正确的调用链中被执行
4. **添加详细日志**：在关键位置添加日志，追踪执行流程
5. **检查数据重新格式化**：确认没有其他方法重新格式化数据

**预防措施**：

1. **使用绝对路径**：在Docker环境中使用绝对路径避免路径问题
2. **依赖管理**：确保所有依赖库在容器中正确安装
3. **测试验证**：每次修改后都要进行完整的端到端测试
4. **日志监控**：实时监控容器日志，及时发现问题

### 2. 前端上传Excel文件显示"不支持的文件类型"

**问题描述**：在前端界面上传Excel文件时，显示"不支持的文件类型"错误，但实际上files_ingestion服务支持22种文件格式。

**根本原因**：前端和后端的文件类型验证列表不一致，都只支持3种格式（.doc, .docx, .pdf），而files_ingestion服务支持22种格式。

**解决方案**：

#### 修改文件：`backend/app/api/routes/file_ingestion.py`
```python
# 修改前：只支持3种格式
# if not file.filename.lower().endswith(('.doc', '.docx', '.pdf')):

# 修改后：支持22种格式
supported_extensions = (
    '.txt', '.json', '.html', '.htm', '.md', '.markdown',  # 文本格式
    '.xlsx', '.xls', '.csv',  # Excel格式
    '.docx', '.doc',  # Word格式
    '.pptx', '.ppt',  # PowerPoint格式
    '.pdf',  # PDF格式
    '.png', '.jpg', '.jpeg', '.bmp', '.tiff', '.tif', '.gif', '.webp'  # 图片格式
)
if not file.filename.lower().endswith(supported_extensions):
    raise HTTPException(
        status_code=400,
        detail=f"不支持的文件格式: {file.filename}。支持的文件类型包括：Word(.doc, .docx), PDF(.pdf), Excel(.xlsx, .xls, .csv), PowerPoint(.pptx, .ppt), 图片(.png, .jpg, .jpeg, .bmp, .tiff, .tif, .gif, .webp), 文本(.txt, .json, .html, .htm, .md, .markdown)"
    )
```

#### 修改文件：`frontend/src/routes/_layout/data-ingestor.upload.tsx`
```typescript
// 修改前：只支持3种格式
// const allowedTypes = ['.doc', '.docx', '.pdf']

// 修改后：支持22种格式
const allowedTypes = [
  '.txt', '.json', '.html', '.htm', '.md', '.markdown',
  '.xlsx', '.xls', '.csv',
  '.docx', '.doc',
  '.pptx', '.ppt',
  '.pdf',
  '.png', '.jpg', '.jpeg', '.bmp', '.tiff', '.tif', '.gif', '.webp'
];

// 更新accept属性
<Input
  type="file"
  accept=".txt,.json,.html,.htm,.md,.markdown,.xlsx,.xls,.csv,.docx,.doc,.pptx,.ppt,.pdf,.png,.jpg,.jpeg,.bmp,.tiff,.tif,.gif,.webp"
  onChange={handleFileChange}
  multiple
/>

// 更新显示文本
<Text fontSize="sm" color="gray.600">
  支持的文件类型：Word(.doc, .docx), PDF(.pdf), Excel(.xlsx, .xls, .csv), PowerPoint(.pptx, .ppt), 图片(.png, .jpg, .jpeg, .bmp, .tiff, .tif, .gif, .webp), 文本(.txt, .json, .html, .htm, .md, .markdown)
</Text>
```

**重启服务**：
```bash
docker-compose restart frontend backend
```

### 2. 后端调用旧文档处理逻辑而不是RAGFlow

**问题描述**：后端仍然调用file_extraction_service（旧逻辑）而不是files_ingestion的RAGFlow处理逻辑。

**根本原因**：backend/app/api/routes/file_ingestion.py中导入的是旧服务。

**解决方案**：

#### 修改文件：`backend/app/api/routes/file_ingestion.py`
```python
# 修改前：导入旧服务
# from app.services.file_extraction_service import file_extraction_service

# 修改后：导入RAGFlow适配器
from app.services.ragflow_adapter import RAGFlowAdapter

# 在upload_file函数中修改调用
# 修改前：
# extraction_result = await file_extraction_service.extract_text_from_file(cache_id, file.filename)

# 修改后：
ragflow_adapter = RAGFlowAdapter()
if not await ragflow_adapter.check_service_health():
    logger.warning("RAGFlow service is not healthy, but continuing with file upload")
extraction_result = await ragflow_adapter.extract_text_from_file(cache_id, file.filename)
```

#### 修改文件：`backend/app/services/ragflow_adapter.py`
```python
# 修改前：使用settings
# from app.core.config import settings

# 修改后：使用ragflow_settings
from app.core.ragflow_config import ragflow_settings

def __init__(self):
    self.ragflow_service = RAGFlowDocumentService(
        enable_ocr=ragflow_settings.RAGFLOW_ENABLE_OCR,
        use_ragflow_parsers=ragflow_settings.RAGFLOW_USE_NATIVE_PARSERS
    )
    self.base_url = ragflow_settings.RAGFLOW_HTTP_URL
    self.timeout = ragflow_settings.RAGFLOW_HTTP_TIMEOUT
```

**重启服务**：
```bash
docker-compose restart backend
```

### 3. RAGFlow服务模块导入错误

**问题描述**：后端日志显示`No module named 'standalone_document_parser'`错误。

**根本原因**：ragflow_document_service.py尝试直接导入模块，但在容器化环境中应该使用HTTP调用。

**解决方案**：

#### 修改文件：`backend/app/services/ragflow_document_service.py`
```python
# 删除直接导入代码
# 删除这些行：
# files_ingestion_path = os.path.join(os.path.dirname(__file__), '..', '..', '..', 'files_ingestion')
# if os.path.exists(files_ingestion_path):
#     sys.path.insert(0, files_ingestion_path)

def _init_ragflow_processor(self):
    """初始化RAGFlow处理器"""
    try:
        import httpx
        
        # 使用HTTP调用files_ingestion服务
        self.files_ingestion_url = "http://files-ingestion:8800"
        
        # 测试连接
        with httpx.Client(timeout=5) as client:
            response = client.get(f"{self.files_ingestion_url}/health")
            if response.status_code == 200:
                self.processor = "http"  # 标记为使用HTTP调用
                logger.info("RAGFlow文档处理器初始化成功（HTTP模式）")
            else:
                logger.warning(f"Files ingestion服务不可用: {response.status_code}")
                self.processor = None
                
    except Exception as e:
        logger.warning(f"RAGFlow处理器初始化失败: {e}")
        self.processor = None

def _extract_with_ragflow(self, content: bytes, filename: str) -> str:
    """使用RAGFlow处理器提取文本"""
    try:
        import httpx
        
        # 通过HTTP调用files_ingestion服务
        with httpx.Client(timeout=60) as client:
            response = client.post(
                f"{self.files_ingestion_url}/extract_text",
                data={"enable_ocr": self.enable_ocr},
                files={"file": (filename, content, "application/octet-stream")},
                headers={"User-Agent": "Foundation-RAGFlow/1.0"}
            )
            
            if response.status_code == 200:
                result = response.json()
                text = result.get("text", "")
                logger.info(f"RAGFlow解析成功: {filename}, 文本长度: {len(text)}")
                return text
            else:
                logger.error(f"Files ingestion服务返回错误: {response.status_code}, {response.text}")
                raise Exception(f"Files ingestion服务错误: {response.status_code}")
                
    except Exception as e:
        logger.error(f"RAGFlow解析失败 {filename}: {e}")
        raise e

def is_available(self) -> bool:
    """检查RAGFlow服务是否可用"""
    return self.processor == "http"
```

**重启服务**：
```bash
docker-compose restart backend
```

### 4. PydanticImportError错误

**问题描述**：后端启动时显示`BaseSettings has been moved to the pydantic-settings package`错误。

**根本原因**：Pydantic v2将BaseSettings移到了单独的包中。

**解决方案**：

#### 修改文件：`backend/app/core/ragflow_config.py`
```python
# 修改前：
# from pydantic import BaseSettings

# 修改后：
from pydantic_settings import BaseSettings
```

**验证依赖**：确保`backend/pyproject.toml`包含：
```toml
pydantic-settings = "==2.2.1"
```

**重启服务**：
```bash
docker-compose restart backend
```

### 5. 数据库连接错误

**问题描述**：后端启动时显示`psycopg2.OperationalError: connection to server failed`和`FATAL: password authentication failed`错误。

**根本原因**：数据库服务未完全启动或密码配置问题。

**解决方案**：

#### 检查数据库状态：
```bash
# 查看数据库容器状态
docker ps | grep db

# 检查数据库健康状态
docker exec foundation-db-1 pg_isready -U postgres

# 测试数据库连接
docker exec foundation-backend-1 psql -h db -U postgres -d app -c "SELECT 1;"
```

#### 检查环境变量：
```bash
# 检查.env文件中的数据库配置
cat .env | grep -E "POSTGRES_PASSWORD|POSTGRES_USER"

# 检查数据库容器内的环境变量
docker exec foundation-db-1 env | grep POSTGRES
```

#### 重启数据库服务：
```bash
# 重启数据库
docker-compose restart db

# 等待数据库完全启动
sleep 10

# 重启后端服务
docker-compose restart backend
```

### 6. 数据库表结构错误

**问题描述**：后端启动时显示`null value in column "create_time" violates not-null constraint`错误。

**根本原因**：Role和Permission模型中的时间字段没有正确的默认值。

**解决方案**：

#### 修改文件：`backend/app/models.py`
```python
# 为Role模型添加默认值
class Role(SQLModel, table=True):
    __tablename__ = "roles"
    
    id: Optional[int] = Field(default=None, primary_key=True)
    name: str = Field(unique=True, index=True)
    description: Optional[str] = Field(default=None)
    # 修改这些字段的默认值
    create_time: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))
    update_time: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))

# 为Permission模型添加默认值
class Permission(SQLModel, table=True):
    __tablename__ = "permissions"
    
    id: Optional[int] = Field(default=None, primary_key=True)
    name: str = Field(unique=True, index=True)
    description: Optional[str] = Field(default=None)
    # 修改这些字段的默认值
    create_time: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))
    update_time: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))
```

#### 重新创建数据库表：
```bash
# 进入数据库容器
docker exec -it foundation-db-1 psql -U postgres -d app

# 删除现有表
DROP TABLE IF EXISTS roles, permissions CASCADE;

# 退出数据库
\q

# 重启后端服务以重新创建表
docker-compose restart backend
```

---

## 常见错误代码及解决方案

### 1. PydanticImportError
**错误**：`BaseSettings has been moved to the pydantic-settings package`

**解决**：修改导入语句
```python
from pydantic_settings import BaseSettings  # 正确
# from pydantic import BaseSettings  # 错误
```

### 2. 模块导入错误
**错误**：`No module named 'standalone_document_parser'`

**解决**：使用HTTP调用而不是直接导入模块

### 3. 数据库约束错误
**错误**：`null value in column "create_time" violates not-null constraint`

**解决**：为时间字段添加正确的默认值

---

## 预防措施

### 1. 定期检查服务状态
```bash
# 创建健康检查脚本
#!/bin/bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

### 2. 监控日志
```bash
# 设置日志监控
docker-compose logs -f --tail=100
```

### 3. 备份重要数据
```bash
# 备份数据库
docker exec foundation-db-1 pg_dump -U postgres app > backup.sql
```

---

## 联系支持

如果以上方法都无法解决问题，请：

1. 收集完整的错误日志
2. 记录问题发生的具体步骤
3. 提供环境信息（操作系统、Docker版本等）
4. 联系技术支持团队

---

## Docker离线部署相关问题

### 总结：最简化的离线部署方案 🎯

**经过实践验证，无需创建额外的offline脚本，直接使用原有的启动脚本即可实现离线部署！**

**核心发现**：
- Docker Compose会优先使用本地已存在的镜像
- 只要提前下载并导入所有镜像，原有的`start-dev.sh`、`start-staging.sh`、`start-prod.sh`脚本就能在离线环境正常工作
- 不需要修改任何docker-compose配置文件
- 不需要创建额外的offline启动脚本

**三步实现离线部署**：

1. **有网络环境预构建**：
```bash
./scripts/predownload-images.sh        # 下载所有基础镜像
./scripts/build-files-ingestion.sh     # 预构建files-ingestion镜像
docker save $(docker images --format "{{.Repository}}:{{.Tag}}") -o docker-images/all-images.tar
```

2. **传输到离线环境并导入**：
```bash
./scripts/import-images.sh --no-md5    # 导入所有镜像
```

3. **使用原有脚本启动**：
```bash
./bin/start-dev.sh        # 开发环境
./bin/start-staging.sh    # 测试环境  
./bin/start-prod.sh       # 生产环境
```

**优势总结**：
- ✅ 配置零改动
- ✅ 脚本零增加
- ✅ 维护零负担
- ✅ 在线/离线环境无缝切换

---

### 7. 离线环境Docker镜像构建失败问题

**问题描述**：在离线环境下启动项目时，files-ingestion服务构建失败，出现多个错误。

**问题演进过程**：

#### 第一阶段：Docker镜像pull访问被拒绝

**错误现象**：
```bash
WARN: pull access denied for backend, repository does not exist
WARN: pull access denied for frontend, repository does not exist
WARN: pull access denied for files-ingestion, repository does not exist
```

**根本原因**：
1. `docker-compose.yml`中同时定义了`image`和`build`字段
2. Docker Compose优先尝试拉取镜像，而不是本地构建
3. 环境变量`DOCKER_IMAGE_BACKEND`等设置为空字符串导致镜像名称无效

**实际解决方案**（推荐）⭐：

**无需创建额外的offline脚本，直接使用原有的start-dev.sh和start-staging.sh脚本**

这个方案的核心思路是：
1. 保持原有的`docker-compose.yml`配置不变（同时定义`image`和`build`）
2. 所有基础镜像（postgres、redis、nginx等）提前下载到本地
3. 自定义镜像（backend、frontend、files-ingestion）通过`--build`参数强制本地构建
4. Docker Compose会优先使用本地已存在的镜像，无需拉取

**关键点**：
- ✅ **不需要修改docker-compose配置文件** - 保持原有配置
- ✅ **不需要创建新的启动脚本** - 继续使用`./bin/start-dev.sh`和`./bin/start-staging.sh`
- ✅ **Docker Compose的智能机制** - 当本地镜像存在时，`--build`只会构建自定义服务，不会拉取基础镜像
- ✅ **简化维护** - 不需要维护额外的offline配置和脚本

**实施步骤**：

1. **预下载所有基础镜像**（在有网络的环境）：
```bash
# 方法1：使用预下载脚本
./scripts/predownload-images.sh

# 方法2：手动拉取基础镜像
docker pull postgres:17
docker pull redis:7-alpine
docker pull neo4j:5.22.0
docker pull nginx:latest
docker pull node:20
docker pull python:3.10-slim-bookworm
docker pull adminer
docker pull minio/minio:latest
docker pull mongo:latest
docker pull mongo-express:latest
docker pull filebrowser/filebrowser:v2.32.0
```

2. **预构建files-ingestion镜像**（需要系统依赖的服务）：
```bash
# 构建并导出files-ingestion镜像
./scripts/build-files-ingestion.sh
```

3. **导出所有镜像为tar文件**（可选，用于传输到离线环境）：
```bash
# 导出所有镜像
docker save $(docker images --format "{{.Repository}}:{{.Tag}}") -o docker-images/all-images.tar

# 或者单独导出每个镜像
cd docker-images
docker save postgres:17 -o postgres_17.tar
docker save redis:7-alpine -o redis_7_alpine.tar
# ... 其他镜像
```

4. **在离线环境导入镜像**：
```bash
# 使用导入脚本（支持增量导入）
./scripts/import-images.sh --no-md5

# 或者手动导入
docker load -i docker-images/all-images.tar
```

5. **使用原有脚本启动服务**：
```bash
# 开发环境
./bin/start-dev.sh

# 或者测试环境
./bin/start-staging.sh

# 或者生产环境
./bin/start-prod.sh
```

**工作原理解析**：

当执行`docker compose up -d --build`时，Docker Compose的行为：
1. 检查`docker-compose.yml`中定义的每个服务
2. 对于有`image`和`build`字段的服务：
   - 如果指定了`--build`参数，执行本地构建
   - 构建时需要的基础镜像（如`FROM python:3.10-slim-bookworm`）会先检查本地是否存在
   - **本地存在则直接使用，不存在才尝试拉取**
3. 对于只有`image`字段的服务（如postgres、redis）：
   - 检查本地是否存在该镜像
   - **本地存在则直接使用，不存在才尝试拉取**

**优势**：
- ✅ 配置简单，无需额外的offline配置文件
- ✅ 脚本简洁，继续使用原有的启动脚本
- ✅ 维护方便，开发、测试、生产环境使用统一的配置
- ✅ 灵活性高，可以在有网络和无网络环境无缝切换

**替代方案**（创建专用offline脚本，不推荐）：

如果你需要更严格的离线控制，也可以创建专用的offline脚本，但会增加维护成本：

**文件**：`docker-compose.dev.offline-only.yml`
```yaml
# 完全重新定义服务，移除image字段，只保留build配置
services:
  prestart:
    container_name: foundation-prestart-offline-container
    build:
      context: ./backend
    # 注意：没有image字段
    volumes:
      - ./backend:/app
    # ... 其他配置
  
  backend:
    container_name: foundation-backend-offline-container
    build:
      context: ./backend
      target: development
    # 注意：没有image字段
    # ... 其他配置
  
  frontend:
    container_name: foundation-frontend-offline-container
    build:
      context: ./frontend
      dockerfile: Dockerfile.dev
    # 注意：没有image字段
    # ... 其他配置
  
  files-ingestion:
    container_name: foundation-files-ingestion-offline-container
    image: foundation-files-ingestion:latest  # 使用预构建镜像
    # 注意：离线环境使用预构建镜像，不需要build
    # ... 其他配置
```

**文件**：`bin/start-dev-offline.sh`
```bash
# 使用独立的离线配置文件
docker compose \
  -f docker-compose.yml \
  -f docker-compose.dev.offline-only.yml \
  --env-file env.dev \
  up -d
```

**为什么不推荐这个方案**：
- ❌ 需要维护额外的配置文件（dev/staging/prod各一个）
- ❌ 需要维护额外的启动脚本
- ❌ 配置同步复杂，容易出现不一致
- ❌ 实际上并无额外优势，因为Docker Compose本身就会优先使用本地镜像

#### 第二阶段：Debian基础镜像版本不匹配

**错误现象**：
```bash
Err:12 http://deb.debian.org/debian trixie InRelease
  Unable to connect to deb.debian.org:http:
W: Failed to fetch http://deb.debian.org/debian/dists/trixie/InRelease
```

**根本原因**：
1. 使用`python:3.10-slim`作为基础镜像，实际上是Debian Trixie版本
2. Dockerfile中配置了Debian Bookworm的清华源
3. 版本不匹配导致apt尝试访问官方Trixie源
4. 离线环境无法访问外部网络，apt-get update失败

**解决方案**：明确指定基础镜像版本

**具体修改**：

**文件**：`files_ingestion/standalone_document_parser/Dockerfile`
```dockerfile
# 修改前：版本不明确
FROM python:3.10-slim

# 修改后：明确使用Bookworm版本
FROM python:3.10-slim-bookworm
```

**技术细节**：
- `python:3.10-slim`：跟随最新的Debian版本（可能是Trixie）
- `python:3.10-slim-bookworm`：明确使用Debian 12 Bookworm版本
- 必须确保基础镜像版本与apt源配置一致

#### 第三阶段：apt源配置不彻底

**错误现象**：
```bash
# 虽然配置了清华源，但仍然尝试访问官方源
30.77 Ign:12 http://deb.debian.org/debian trixie InRelease
30.77 Ign:13 http://deb.debian.org/debian trixie-updates InRelease
```

**根本原因**：
1. 基础镜像可能在`/etc/apt/sources.list.d/`目录有额外的源配置
2. 仅覆盖`/etc/apt/sources.list`不够，需要清理所有源配置
3. 旧的apt缓存可能导致问题

**解决方案**：完全清理并重新配置apt源

**具体修改**：

**文件**：`files_ingestion/standalone_document_parser/Dockerfile`
```dockerfile
# 完全清理apt源配置
RUN rm -rf /etc/apt/sources.list /etc/apt/sources.list.d/* && \
    echo "deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm main contrib non-free non-free-firmware" > /etc/apt/sources.list && \
    echo "deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm-updates main contrib non-free non-free-firmware" >> /etc/apt/sources.list && \
    echo "deb https://mirrors.tuna.tsinghua.edu.cn/debian-security/ bookworm-security main contrib non-free non-free-firmware" >> /etc/apt/sources.list && \
    apt-get clean && \
    apt-get update && apt-get install -y --no-install-recommends \
    # ... 包列表
    && rm -rf /var/lib/apt/lists/* \
    && rm -rf /tmp/* \
    && rm -rf /var/tmp/*
```

**关键点**：
- 删除`/etc/apt/sources.list`和`/etc/apt/sources.list.d/*`所有文件
- 添加`apt-get clean`清理旧缓存
- 使用`--no-install-recommends`减少依赖包
- 安装后彻底清理缓存和临时文件

#### 第四阶段：Debian包依赖冲突

**错误现象**：
```bash
The following packages have unmet dependencies:
 build-essential : Depends: libc6-dev but it is not installable or libc-dev
 perl : Depends: perl-base (= 5.36.0-7+deb12u3) but 5.40.1-6 is to be installed
E: Error, pkgProblemResolver::Resolve generated breaks
```

**根本原因**：
1. 使用的包名在Bookworm中已过时或不可用
2. `libgl1-mesa-glx`在较新的Debian版本中已被替换
3. `libxrender-dev`是开发包，应使用运行时包`libxrender1`
4. 中文OCR语言包在离线环境中可能导致额外的依赖问题

**解决方案**：更新包名并简化依赖

**具体修改**：

**文件**：`files_ingestion/standalone_document_parser/Dockerfile`
```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
    # 编译工具
    gcc \
    g++ \
    make \
    build-essential \
    # 图像处理依赖（更新包名）
    libgl1-mesa-glx \        # 保持兼容性
    libsm6 \
    libxext6 \
    libxrender1 \            # 使用运行时包而不是dev包
    libgomp1 \
    # PDF处理依赖
    poppler-utils \
    # OCR依赖 - 只安装核心包
    tesseract-ocr \
    tesseract-ocr-eng \      # 仅安装英文语言包
    # 其他必要依赖
    curl \
    wget \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* \
    && rm -rf /tmp/* \
    && rm -rf /var/tmp/*
```

**包名变更说明**：
- ~~`libgl1-mesa-dri`~~ → `libgl1-mesa-glx`：使用更兼容的包
- ~~`libxrender-dev`~~ → `libxrender1`：使用运行时库而非开发库
- ~~`libglib2.0-0`~~：可能不是必需的，可以移除
- ~~`tesseract-ocr-chi-sim`~~：中文简体包（离线环境可选）
- ~~`tesseract-ocr-chi-tra`~~：中文繁体包（离线环境可选）

#### 第五阶段：离线环境预构建策略

**问题描述**：即使解决了所有配置问题，在真正的离线环境中，`apt-get update`和`apt-get install`仍然无法工作。

**根本原因**：
- 离线环境无法访问任何外部网络，包括清华镜像源
- Docker构建过程中必须下载系统依赖包
- 无法在构建时安装新的系统包

**最终解决方案**：预构建完整镜像

**实施步骤**：

1. **在有网络的环境下预构建镜像**

**文件**：`scripts/build-files-ingestion.sh`
```bash
#!/bin/bash
# 预构建files-ingestion镜像，包含所有系统依赖

IMAGE_NAME="foundation-files-ingestion"
IMAGE_TAG="latest"

# 构建完整镜像
docker build -f files_ingestion/standalone_document_parser/Dockerfile \
  -t ${IMAGE_NAME}:${IMAGE_TAG} \
  ./files_ingestion

# 导出镜像为tar文件
docker save -o docker-images/${IMAGE_NAME}_${IMAGE_TAG}.tar ${IMAGE_NAME}:${IMAGE_TAG}

# 生成MD5校验值
md5sum docker-images/${IMAGE_NAME}_${IMAGE_TAG}.tar > docker-images/${IMAGE_NAME}_${IMAGE_TAG}.tar.md5
```

2. **离线环境导入镜像**

**文件**：`scripts/import-images.sh`
```bash
#!/bin/bash
# 导入预构建的Docker镜像

TAR_DIR="./docker-images"

# 导入镜像
for tar_file in ${TAR_DIR}/*.tar; do
    echo "导入镜像: $tar_file"
    docker load -i "$tar_file"
done
```

3. **离线启动脚本自动导入**

**文件**：`bin/start-dev-offline.sh`
```bash
#!/bin/bash

# 检查并导入缺失的镜像
REQUIRED_IMAGES=(
    "foundation-files-ingestion:latest"
    "python:3.10-slim-bookworm"
    "node:20"
    # ... 其他镜像
)

for image in "${REQUIRED_IMAGES[@]}"; do
    if ! docker image inspect "$image" >/dev/null 2>&1; then
        echo "镜像不存在: $image，尝试导入..."
        ./scripts/import-images.sh --no-md5
    fi
done

# 启动服务
docker compose -f docker-compose.yml -f docker-compose.dev.offline-only.yml --env-file env.dev up -d
```

**预构建方案的优势**：
- ✅ 所有系统依赖在有网络环境预先安装
- ✅ 离线环境只需导入tar文件，无需网络访问
- ✅ 避免离线环境的apt-get问题
- ✅ 构建过程快速且可靠

### 8. 前端界面显示Nginx默认页面问题

**问题描述**：访问前端界面时，显示Nginx的默认欢迎页面，而不是React应用。

**错误现象**：
```
Welcome to nginx!
If you see this page, the nginx web server is successfully installed and working.
```

**根本原因分析**：

#### 原因1：前端容器构建失败
- 前端Dockerfile构建过程中断
- `npm install`或`npm run build`失败
- 静态文件未正确生成到`/usr/share/nginx/html`

#### 原因2：Nginx配置错误
- Nginx配置文件中的`root`路径错误
- 静态文件目录为空或权限问题
- Nginx回退到默认配置

#### 原因3：Docker Compose配置问题
- 前端服务的volumes映射错误
- 覆盖了容器内的静态文件目录
- build context路径不正确

**诊断步骤**：

```bash
# 1. 检查前端容器状态
docker ps | grep frontend
docker logs foundation-frontend-offline-container --tail=50

# 2. 检查静态文件是否存在
docker exec foundation-frontend-offline-container ls -la /usr/share/nginx/html

# 3. 检查Nginx配置
docker exec foundation-frontend-offline-container cat /etc/nginx/conf.d/default.conf

# 4. 检查构建日志
docker compose -f docker-compose.yml -f docker-compose.dev.offline-only.yml logs frontend | grep -i error
```

**解决方案**：

**文件**：`docker-compose.dev.offline-only.yml`
```yaml
frontend:
  container_name: foundation-frontend-offline-container
  build:
    context: ./frontend
    dockerfile: Dockerfile.dev  # 确认使用正确的Dockerfile
    args:
      - VITE_API_URL=http://${DOMAIN:-localhost}:8000
      - VITE_ENVIRONMENT=development
  volumes:
    - ./frontend:/app           # 开发模式：映射源代码
    - /app/node_modules         # 防止覆盖node_modules
  ports:
    - "5173:5173"               # 开发模式使用Vite端口
  # 注意：开发模式不使用Nginx，直接运行Vite dev server
```

**文件**：`frontend/Dockerfile.dev`
```dockerfile
FROM node:20

WORKDIR /app

COPY package*.json ./
RUN npm install --no-audit --no-fund

COPY . .

EXPOSE 5173

# 开发模式：运行Vite dev server
CMD ["npm", "run", "dev", "--", "--host", "0.0.0.0"]
```

**生产模式配置**：

**文件**：`docker-compose.prod.offline-only.yml`
```yaml
frontend:
  container_name: foundation-frontend-offline-container
  build:
    context: ./frontend
    dockerfile: Dockerfile      # 生产模式使用标准Dockerfile
    args:
      - VITE_API_URL=https://${DOMAIN}
      - VITE_ENVIRONMENT=production
  # 注意：生产模式不映射volumes
  ports:
    - "80:80"
```

**文件**：`frontend/Dockerfile`
```dockerfile
# 构建阶段
FROM node:20 AS build-stage
WORKDIR /app
COPY package*.json ./
RUN npm install --no-audit --no-fund
COPY . .
ARG VITE_API_URL
ARG VITE_ENVIRONMENT
RUN npm run build

# 生产阶段
FROM nginx:latest
COPY --from=build-stage /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**关键检查点**：
1. ✅ 开发模式使用Vite dev server（端口5173）
2. ✅ 生产模式使用Nginx提供静态文件（端口80）
3. ✅ 确认`npm run build`成功生成`dist`目录
4. ✅ Nginx配置正确指向静态文件目录

### 9. 登录功能报错问题

**问题描述**：前端登录时，提交表单后返回错误，无法成功登录。

**常见错误现象**：

#### 错误1：CORS跨域错误
```
Access to XMLHttpRequest at 'http://localhost:8000/api/v1/login/access-token' 
from origin 'http://localhost:5173' has been blocked by CORS policy
```

**根本原因**：
- 后端CORS配置不包含前端开发服务器地址
- 前端API URL配置错误
- 代理配置缺失或错误

**解决方案**：

**文件**：`backend/app/core/config.py`
```python
class Settings(BaseSettings):
    # CORS配置
    BACKEND_CORS_ORIGINS: list[str] = [
        "http://localhost",
        "http://localhost:5173",    # Vite dev server
        "http://localhost:3000",    # 备用端口
        "http://127.0.0.1:5173",
    ]
    
    @property
    def all_cors_origins(self) -> list[str]:
        return [str(origin).rstrip("/") for origin in self.BACKEND_CORS_ORIGINS]
```

**文件**：`backend/app/main.py`
```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.all_cors_origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

#### 错误2：数据库连接失败
```
sqlalchemy.exc.OperationalError: (psycopg2.OperationalError) 
connection to server at "db" (172.18.0.2), port 5432 failed: 
FATAL: password authentication failed for user "postgres"
```

**根本原因**：
- 数据库服务未启动或未就绪
- 环境变量中的数据库密码错误
- prestart脚本未成功初始化数据库

**解决方案**：

```bash
# 1. 检查数据库容器状态
docker ps | grep db
docker logs foundation-db-1 --tail=20

# 2. 检查环境变量
cat env.dev | grep POSTGRES

# 3. 验证数据库连接
docker exec foundation-db-1 psql -U postgres -d app -c "SELECT 1;"

# 4. 重启数据库和后端服务
docker compose restart db
sleep 10
docker compose restart prestart backend
```

**文件**：`docker-compose.dev.offline-only.yml`
```yaml
backend:
  depends_on:
    db:
      condition: service_healthy    # 等待数据库健康检查通过
    prestart:
      condition: service_completed_successfully  # 等待初始化完成
```

#### 错误3：JWT Token配置问题
```
401 Unauthorized
{"detail": "Could not validate credentials"}
```

**根本原因**：
- SECRET_KEY未配置或为空
- Token过期时间配置错误
- Token签名算法不匹配

**解决方案**：

**文件**：`env.dev`
```bash
# JWT配置
SECRET_KEY=your-secret-key-here-change-in-production-min-32-chars
ACCESS_TOKEN_EXPIRE_MINUTES=60
ALGORITHM=HS256
```

**生成安全的SECRET_KEY**：
```bash
# 生成随机SECRET_KEY
python -c "import secrets; print(secrets.token_urlsafe(32))"
```

#### 错误4：用户不存在或密码错误
```
400 Bad Request
{"detail": "Incorrect email or password"}
```

**根本原因**：
- 数据库未初始化超级用户
- prestart脚本未成功执行
- 用户输入的邮箱或密码错误

**解决方案**：

```bash
# 1. 检查prestart日志
docker logs foundation-prestart-offline-container

# 2. 手动创建超级用户
docker exec -it foundation-backend-offline-container python -c "
from app.core.db import engine
from app.models import User
from sqlmodel import Session, select
from app.core.security import get_password_hash

with Session(engine) as session:
    # 检查是否存在超级用户
    user = session.exec(select(User).where(User.email == 'admin@example.com')).first()
    if not user:
        user = User(
            email='admin@example.com',
            hashed_password=get_password_hash('changethis'),
            is_superuser=True,
            full_name='Admin User'
        )
        session.add(user)
        session.commit()
        print('超级用户创建成功')
    else:
        print('超级用户已存在')
"
```

**默认登录凭据**（开发环境）：
- 邮箱：`admin@example.com`
- 密码：`changethis`

#### 错误5：网络连接问题
```
NetworkError: Failed to fetch
```

**根本原因**：
- 后端服务未启动或崩溃
- API URL配置错误
- Docker网络配置问题

**诊断步骤**：

```bash
# 1. 检查后端服务状态
docker ps | grep backend
docker logs foundation-backend-offline-container --tail=50

# 2. 测试API端点
curl http://localhost:8000/api/v1/utils/health-check/

# 3. 检查Docker网络
docker network ls
docker network inspect foundation_default

# 4. 检查前端环境变量
docker exec foundation-frontend-offline-container env | grep VITE
```

**解决方案**：

**文件**：`.env.dev`
```bash
# API配置
DOMAIN=localhost
VITE_API_URL=http://localhost:8000
```

**文件**：`docker-compose.dev.offline-only.yml`
```yaml
backend:
  networks:
    - default
  ports:
    - "8000:8000"
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:8000/api/v1/utils/health-check/"]
    interval: 30s
    timeout: 10s
    retries: 3

frontend:
  networks:
    - default
  environment:
    - VITE_API_URL=http://localhost:8000
```

**完整的登录问题排查清单**：

1. ✅ CORS配置正确
2. ✅ 数据库服务运行且健康
3. ✅ prestart初始化成功
4. ✅ SECRET_KEY已配置
5. ✅ 超级用户已创建
6. ✅ 后端服务运行且健康
7. ✅ 前端API URL配置正确
8. ✅ Docker网络连接正常

---

## 离线部署最佳实践

### 1. 推荐的离线部署方案 ⭐

**核心思路**：直接使用原有的`start-dev.sh`、`start-staging.sh`、`start-prod.sh`脚本，无需创建额外的offline脚本。

**有网络环境（预构建阶段）**：
```bash
# 1. 拉取所有基础镜像
./scripts/predownload-images.sh

# 2. 预构建files-ingestion镜像（包含系统依赖）
./scripts/build-files-ingestion.sh

# 3. 导出所有镜像为tar文件（用于传输到离线环境）
docker save $(docker images --format "{{.Repository}}:{{.Tag}}") -o docker-images/all-images.tar

# 4. 生成MD5校验（可选）
md5sum docker-images/all-images.tar > docker-images/all-images.tar.md5
```

**离线环境（部署阶段）**：
```bash
# 1. 导入镜像
./scripts/import-images.sh --no-md5

# 2. 验证镜像（可选）
./scripts/verify-offline-images.sh dev

# 3. 使用原有脚本启动服务
./bin/start-dev.sh        # 开发环境
# 或
./bin/start-staging.sh    # 测试环境
# 或
./bin/start-prod.sh       # 生产环境
```

**关键优势**：
- ✅ 无需修改docker-compose配置文件
- ✅ 无需创建额外的offline启动脚本
- ✅ Docker Compose会自动优先使用本地镜像
- ✅ 配置统一，维护简单
- ✅ 可在有网络和无网络环境无缝切换

**工作原理**：
- 当执行`docker compose up -d --build`时，Docker会先检查本地是否有镜像
- 如果本地镜像存在，直接使用，不会尝试从网络拉取
- 只有当本地不存在时，才会尝试从Docker Hub拉取
- 因此，只要提前下载好所有镜像，离线环境就能正常工作

### 2. 替代方案（创建专用offline脚本）

如果需要更严格的离线控制，可以创建专用的offline脚本，但会增加维护成本：

**有网络环境（预构建阶段）**：
```bash
# 1. 拉取所有基础镜像
./scripts/predownload-images.sh

# 2. 预构建自定义镜像
./scripts/build-files-ingestion.sh

# 3. 导出所有镜像
docker save $(docker images --format "{{.Repository}}:{{.Tag}}") -o docker-images/all-images.tar

# 4. 生成MD5校验
md5sum docker-images/all-images.tar > docker-images/all-images.tar.md5
```

**离线环境（部署阶段）**：
```bash
# 1. 导入镜像
./scripts/import-images.sh --no-md5

# 2. 验证镜像
./scripts/verify-offline-images.sh dev

# 3. 使用专用offline脚本启动服务
./bin/start-dev-offline.sh        # 需要额外创建
./bin/start-staging-offline.sh    # 需要额外创建
./bin/start-prod-offline.sh       # 需要额外创建
```

**不推荐这个方案的原因**：
- ❌ 需要维护额外的配置文件（3个环境×3个文件=9个文件）
- ❌ 需要维护额外的启动脚本（3个环境×2个脚本=6个脚本）
- ❌ 配置同步复杂，容易出现不一致
- ❌ 实际上并无额外优势

### 3. 环境配置检查清单

**部署前检查**：
- [ ] 所有Docker镜像已导入
- [ ] 环境变量文件已配置（`env.dev`）
- [ ] SECRET_KEY已生成并配置
- [ ] 数据库密码已设置
- [ ] Docker服务运行正常
- [ ] 磁盘空间充足（建议>20GB）

**部署后验证**：
- [ ] 所有容器运行正常（`docker ps`）
- [ ] 数据库健康检查通过
- [ ] 后端API健康检查通过（`curl http://localhost:8000/api/v1/utils/health-check/`）
- [ ] 前端页面可访问（`http://localhost:5173`）
- [ ] 登录功能正常
- [ ] 文件上传功能正常

### 4. 常见问题快速诊断

**问题**：服务无法启动
```bash
# 检查容器日志
docker compose logs --tail=50

# 检查特定服务
docker logs foundation-backend-offline-container --tail=50
```

**问题**：前端显示Nginx默认页面
```bash
# 检查是否使用了正确的Dockerfile
docker compose config | grep -A5 frontend

# 检查静态文件
docker exec foundation-frontend-offline-container ls -la /usr/share/nginx/html
```

**问题**：登录失败
```bash
# 检查后端日志
docker logs foundation-backend-offline-container | grep -i error

# 检查数据库连接
docker exec foundation-backend-offline-container python -c "from app.core.db import engine; engine.connect()"

# 验证超级用户
docker exec foundation-db-1 psql -U postgres -d app -c "SELECT email, is_superuser FROM users;"
```

---

### 工作流执行问题：App Node 状态无法更新和输出为空

**问题描述**：在运行工作流时，App Node 节点状态一直保持 IDLE，无法从 IDLE 变为 running，然后 completed。即使 DAG 任务成功完成，App Node 的 All Outputs 仍然为空，Node Results 只显示 Start Node completed，没有 App Node 和 Data Transform Node 的内容。

**问题演进过程**：

#### 第一阶段：节点类型不匹配问题

**错误现象**：
```
Error executing workflow task: Found edge starting at unknown node 'app-1'
```

**用户报告的问题**：
- App Node 节点状态一直是 IDLE，没有变为 running，然后 completed
- 在 DAG 还在 running 时，已经有多次 GET 请求返回 "status": "failed"
- Recent Tasks 显示 FAILED，但 DAG 任务还在 running 中

**根本原因分析**：
1. 前端发送的节点类型是 `'application'`（从 `FlowWorkspace.tsx` 第 806 行可以看到 `type: n.type`，而前端节点类型是 `'application'`）
2. 后端 `build_graph` 方法中只检查 `node_type == 'app'`（`workflow_execution_service.py:171`）
3. 当节点类型是 `'application'` 时，它不会被添加到图中，导致后续添加边时找不到源节点，报错 "Found edge starting at unknown node 'app-1'"

**建议方案**：
- **方案 1**：修改后端支持 `'application'` 类型（推荐）
  - 在 `build_graph` 中同时支持 `'app'` 和 `'application'`，或统一为 `'application'`
- **方案 2**：修改前端统一为 `'app'`
  - 将前端节点类型改为 `'app'`，与后端一致

**实际采纳方案**：方案 1（修改后端支持 `'application'` 类型）

**最终解决方法**：在 `build_graph` 中同时支持 `'app'` 和 `'application'` 类型

**具体修改**：

**文件**：`backend/app/services/workflow/workflow_execution_service.py`
```python
# 修改前（第 171 行）：
elif node_type == 'app':
    # Pass app_node_configs to app node
    app_config = app_node_configs.get(node_id) if app_node_configs else None
    workflow.add_node(node_id, self._create_app_node(node, app_config, session, user_id))

# 修改后：
elif node_type == 'app' or node_type == 'application':
    # Support both 'app' and 'application' types (frontend uses 'application')
    # Pass app_node_configs to app node
    app_config = app_node_configs.get(node_id) if app_node_configs else None
    workflow.add_node(node_id, self._create_app_node(node, app_config, session, user_id))
```

**增强措施**：同时添加了节点和边的验证逻辑，以及详细的日志记录：
```python
# 添加节点验证
added_node_ids = set()
# ... 节点添加逻辑 ...
if source not in added_node_ids:
    raise ValueError(f"Found edge starting at unknown node '{source}'. Available nodes: {added_node_ids}")
```

#### 第二阶段：State 字段缺失问题

**错误现象**：
```
Error executing workflow task: KeyError: 'errors'
```

**用户报告的问题**：
- 任务状态从 running 变为 failed
- 后端日志显示 `KeyError: 'errors'`
- App Node 仍然没有执行

**根本原因分析**：
1. 在 `data_transform_node.py` 中，代码访问了 `state['errors']`（第 128、137、159、168 行）
2. 但在 `workbench.py` 的 `initial_state` 中，没有初始化 `errors` 字段
3. 当 Data Transform Node 执行时，尝试访问不存在的 `errors` 字段，导致 `KeyError: 'errors'`
4. 工作流因此失败，App Node 的状态没有更新

**建议方案**：在 `initial_state` 中初始化所有 `WorkflowState` 必需字段

**实际采纳方案**：完全采纳

**最终解决方法**：在 `initial_state` 中添加所有必需字段的初始化

**具体修改**：

**文件**：`backend/app/api/v1/workbench.py`
```python
# 修改前（第 415-424 行）：
initial_state = {
    "input_data": request.input_data,
    "workflow_definition": {
        "nodes": request.nodes,
        "edges": request.edges
    },
    "node_results": {},
    "current_node_id": None
}

# 修改后：
initial_state = {
    "instance_id": 0,  # Workflow instance ID (not used in current implementation)
    "input_data": request.input_data or {},
    "workflow_definition": {
        "nodes": request.nodes,
        "edges": request.edges
    },
    "node_results": {},
    "current_node_id": None,
    "structured_data": {},  # Extracted structured data (DataTransform Node output)
    "execution_history": [],  # List of node IDs in execution order
    "errors": []  # Error information (required by DataTransform Node)
}
```

**文件**：`backend/app/services/workflow/state.py`
```python
# 修改前（第 25 行）：
current_node_id: str

# 修改后：
current_node_id: Optional[str]  # 因为初始值可以是 None
```

#### 第三阶段：Reducer 使用错误问题

**错误现象**：
```
Error executing workflow task: Message dict must contain 'role' and 'content' keys, got {}
```

**用户报告的问题**：
- 任务状态从 running 变为 failed
- 后端日志显示 `Message dict must contain 'role' and 'content' keys, got {}`
- App Node 仍然没有执行

**根本原因分析**：
1. 在 `state.py` 中，`node_results` 使用了 `add_messages` reducer：
   ```python
   node_results: Annotated[Dict[str, Any], add_messages]
   ```
2. `add_messages` 是 LangGraph 的消息列表 reducer，期望接收消息对象（包含 `role` 和 `content` 键）
3. 代码中直接赋值字典到 `node_results[node_id]`：
   ```python
   state['node_results'][node['id']] = {
       'status': 'completed',
       'output': app_output
   }
   ```
4. LangGraph 尝试将字典转换为消息对象时失败，因为字典中没有 `role` 和 `content` 键

**建议方案**：
- **方案 1**：移除 `add_messages`，使用字典合并 reducer（推荐）
  - `node_results` 是节点执行结果字典，不是消息列表
  - 使用 `operator.add` 或自定义 reducer 来合并字典
- **方案 2**：保持 `add_messages`，但改变赋值方式
  - 将结果包装成消息对象，但这不符合业务语义

**实际采纳方案**：方案 1（移除 `add_messages`，使用自定义字典合并 reducer）

**最终解决方法**：创建自定义 `merge_dict` reducer 来合并字典

**具体修改**：

**文件**：`backend/app/services/workflow/state.py`
```python
# 修改前：
from langgraph.graph.message import add_messages

class WorkflowState(TypedDict):
    # ...
    node_results: Annotated[Dict[str, Any], add_messages]

# 修改后：
def merge_dict(left: Dict[str, Any], right: Dict[str, Any]) -> Dict[str, Any]:
    """Merge two dictionaries, with right dict values taking precedence"""
    result = left.copy() if left else {}
    if right:
        result.update(right)
    return result

class WorkflowState(TypedDict):
    # ...
    # Use merge_dict reducer to allow updating individual node results
    node_results: Annotated[Dict[str, Any], merge_dict]
```

#### 第四阶段：错误处理和日志不足问题

**错误现象**：
```
Error executing workflow task: the connection is closed
```

**用户报告的问题**：
- 工作流执行时间较长（约3分钟），期间有大量 GET 请求返回 "status": "running"
- 最后出现 "the connection is closed" 错误
- 从日志看，没有看到任何节点执行的详细日志（如 `[WF][start]`、`[WF][app]` 等）

**根本原因分析**：
1. 工作流执行时间较长，期间 checkpoint 的数据库连接可能超时关闭
2. 节点执行过程中，checkpoint 尝试保存状态时，连接已关闭，导致错误
3. 错误处理不完整：只记录了错误消息，缺少堆栈跟踪，难以定位问题
4. 日志不足：没有足够的日志追踪节点执行流程

**建议方案**：
1. 增强错误处理：记录完整的异常堆栈跟踪
2. 添加节点执行日志：在关键位置添加日志，追踪节点执行流程
3. 检查 checkpoint 连接管理：确保连接在长时间运行后仍然有效

**实际采纳方案**：增强错误处理和添加节点执行日志

**最终解决方法**：
1. 增强错误处理，添加 `exc_info=True` 来记录完整的异常堆栈跟踪
2. 添加详细的节点执行日志

**具体修改**：

**文件**：`backend/app/api/v1/workbench.py`
```python
# 修改前（第 502 行）：
except Exception as e:
    logger.error(f"Error executing workflow task {task_id}: {e}")

# 修改后：
except Exception as e:
    logger.error(f"Error executing workflow task {task_id}: {e}", exc_info=True)
```

```python
# 修改前（第 430-445 行）：
# Stream workflow execution to send node updates
final_state = initial_state
async for event in graph.astream(initial_state, config):
    for node_id, updated_state in event.items():
        if node_id == "__end__" or not isinstance(updated_state, dict):
            continue
        
        # Get node result from updated state
        node_results = updated_state.get("node_results", {})
        node_result = node_results.get(node_id)
        
        if node_result:

# 修改后：
logger.info(f"[WF][{task_id}] Starting workflow execution with {len(request.nodes)} nodes")

# Stream workflow execution to send node updates
final_state = initial_state
try:
    async for event in graph.astream(initial_state, config):
        logger.debug(f"[WF][{task_id}] Received event: {list(event.keys())}")
        for node_id, updated_state in event.items():
            if node_id == "__end__":
                logger.info(f"[WF][{task_id}] Workflow execution ended")
                continue
            if not isinstance(updated_state, dict):
                logger.warning(f"[WF][{task_id}] Invalid updated_state for node {node_id}: {type(updated_state)}")
                continue
            
            logger.info(f"[WF][{task_id}] Processing node: {node_id}")
            
            # Get node result from updated state
            node_results = updated_state.get("node_results", {})
            node_result = node_results.get(node_id)
            
            if node_result:
                logger.info(f"[WF][{task_id}] Node {node_id} completed with status: {node_result.get('status')}")
            else:
                logger.warning(f"[WF][{task_id}] Node {node_id} has no result in node_results. Available nodes: {list(node_results.keys())}")
            
            if node_result:
except Exception as e:
    logger.error(f"[WF][{task_id}] Error during workflow stream execution: {e}", exc_info=True)
    # If streaming fails, use the last known state or initial state
    # final_state is already set to the last updated_state or initial_state
    # Continue to update task status with available results
```

#### 第五阶段：节点状态更新时机问题

**用户报告的问题**：
- App Node 状态显示 UPLOADED，但没有从 IDLE 变为 running，然后 completed
- 节点输出结果无法正确显示

**根本原因分析**：
- 节点状态更新逻辑需要优化
- `node_started` 的广播时机需要调整，确保在节点处理开始时立即通知前端

**建议方案**：调整 `node_started` 的广播时机，确保在节点处理开始时立即通知前端

**实际采纳方案**：用户自行调整了代码，将 `node_started` 的广播提前到节点处理开始时

**最终解决方法**：调整 `node_started` 的广播时机

**具体修改**（用户自行修改）：

**文件**：`backend/app/api/v1/workbench.py`
```python
# 修改前：
for node_id, updated_state in event.items():
    # ...
    if node_result:
        # Broadcast node started (before execution)
        await manager.broadcast_to_room(...)

# 修改后：
for node_id, updated_state in event.items():
    # ...
    logger.info(f"[WF][{task_id}] Processing node: {node_id}")
    
    # Broadcast node started (when processing begins)
    await manager.broadcast_to_room(f"workflow_{task_id}", {
        "type": "node_started",
        "task_id": task_id,
        "node_id": node_id,
        "status": "running"
    })
    
    # Get node result from updated state
    node_results = updated_state.get("node_results", {})
    node_result = node_results.get(node_id)
    
    if node_result:
        # Update task node_results in real-time
        # ...
```

**验证步骤**：

1. **重新启动后端服务**：
```bash
./bin/stop-dev.sh && ./bin/start-dev.sh
```

2. **测试工作流执行**：
   - 配置 App Node（选择文件）
   - 连接 Start Node → App Node → Data Transform Node
   - 点击 "Run Workflow"
   - 检查后端日志，确认不再出现相关错误
   - 观察节点状态是否从 IDLE 变为 running，然后 completed
   - 检查任务状态是否从 running 变为 completed（而不是 failed）
   - 检查 Node Results 是否包含所有节点的结果

**技术要点**：

- **节点类型兼容性**：前端和后端的节点类型定义需要保持一致，或后端支持多种类型
- **State 初始化**：LangGraph 的 `TypedDict` 要求所有必需字段在初始化时都要提供
- **Reducer 选择**：不同类型的字段需要使用不同的 reducer，字典类型应该使用字典合并 reducer，而不是消息列表 reducer
- **错误处理**：完整的异常堆栈跟踪对于定位问题至关重要
- **日志记录**：详细的日志可以帮助追踪节点执行流程，快速定位问题
- **状态更新时机**：节点状态的更新时机需要与前端 UI 更新逻辑匹配

**预防措施**：

1. **类型定义检查**：在修改节点类型时，确保前端和后端保持一致
2. **State 完整性**：在创建 `initial_state` 时，确保包含所有 `WorkflowState` 定义的必需字段
3. **Reducer 选择**：根据字段的实际用途选择合适的 reducer
4. **错误处理**：始终使用 `exc_info=True` 记录完整的异常堆栈跟踪
5. **日志记录**：在关键执行点添加日志，便于问题追踪

**相关文件**：

- `backend/app/services/workflow/workflow_execution_service.py`（节点类型支持、节点和边验证）
- `backend/app/services/workflow/state.py`（State 定义、Reducer 定义）
- `backend/app/api/v1/workbench.py`（State 初始化、错误处理、日志记录、节点状态更新）
- `frontend/src/components/Workbench/FlowWorkspace.tsx`（前端节点类型定义）

---

### 工作流执行问题：WebSocket 连接在节点流式事件中异常关闭

**问题描述**：
- 在流式处理节点事件时，WebSocket 连接关闭
- 连接关闭导致前端无法实时接收节点执行状态更新
- 工作流执行可能因为 WebSocket 错误而中断

**问题演进过程**：

#### 第一阶段：WebSocket 连接关闭导致工作流中断

**错误现象**：
- Docker 日志中出现 `the connection is closed` 错误
- 工作流执行过程中 WebSocket 连接突然断开
- 前端无法接收到 `node_finished` 等事件

**根本原因分析**：

参考之前处理 App Node 时 checkpoint 连接关闭的思路，WebSocket 连接关闭可能由以下原因导致：

1. **客户端主动断开**：
   - 浏览器刷新、页面关闭、网络中断
   - 客户端主动关闭 WebSocket 连接

2. **长时间运行导致连接超时**：
   - 工作流执行时间较长（如 3 分钟），WebSocket 连接可能超时
   - 网络代理或负载均衡器的超时设置

3. **错误处理不完整**：
   - `broadcast_to_room` 只记录简单错误，缺少堆栈跟踪和连接状态检查
   - WebSocket 发送失败时可能抛出异常，中断工作流执行

4. **日志不足**：
   - 无法追踪连接状态变化和断开原因
   - 缺少连接数统计和健康检查日志

**解决方案**：

参考 App Node checkpoint 连接关闭的处理思路，采用以下修复：

1. **增强 `broadcast_to_room` 的错误处理**：
   - 添加 `exc_info=True` 记录完整堆栈跟踪
   - 检查 WebSocket 连接状态（`client_state`）再发送
   - 添加详细日志（连接数、断开原因、消息类型）

2. **增强 `safe_broadcast` 的日志和容错**：
   - 记录消息类型、room_id、连接数等上下文信息
   - 区分错误类型（连接关闭、网络错误、其他异常）
   - 确保异常不影响工作流执行

3. **添加连接健康检查机制**：
   - 发送前检查连接状态，跳过已关闭连接
   - 自动清理已断开的连接
   - 记录连接状态变化

4. **增强工作流执行日志**：
   - 记录每个节点的 WebSocket 广播状态
   - 记录连接数变化
   - 便于调试和问题定位

**实际采纳方案**：采用上述完整方案，增强错误处理、日志记录和连接健康检查

**最终解决方法**：增强 WebSocket 连接管理和错误处理，确保连接关闭不影响工作流执行

**具体修改**：

**文件**：`backend/app/api/v1/workbench.py`

1. **增强 `ConnectionManager.broadcast_to_room` 方法**：

```python
async def broadcast_to_room(self, room_id: str, message: Dict[str, Any], exclude: WebSocket = None):
    """
    Broadcast message to all connections in a room with enhanced error handling.
    """
    if room_id not in self.active_connections:
        logger.debug(f"No active connections in room {room_id}")
        return
    
    connections = self.active_connections[room_id]
    total_connections = len(connections)
    message_type = message.get("type", "unknown")
    
    logger.debug(f"Broadcasting {message_type} to room {room_id} ({total_connections} connections)")
    
    disconnected = []
    successful = 0
    failed = 0
    
    for connection in connections:
        if connection == exclude:
            continue
        
        # Check connection state before sending
        try:
            client_state = connection.client_state
            if client_state.name != "CONNECTED":
                logger.warning(f"Connection in room {room_id} is not CONNECTED (state: {client_state.name})")
                disconnected.append(connection)
                failed += 1
                continue
        except Exception as state_check_error:
            logger.warning(f"Failed to check connection state in room {room_id}: {state_check_error}")
            disconnected.append(connection)
            failed += 1
            continue
        
        # Attempt to send message
        try:
            await connection.send_text(json.dumps(message))
            successful += 1
        except Exception as e:
            # Enhanced error logging with full stack trace
            error_type = type(e).__name__
            error_msg = str(e)
            
            # Classify error type
            if "connection is closed" in error_msg.lower():
                error_category = "CONNECTION_CLOSED"
            elif "broken pipe" in error_msg.lower():
                error_category = "BROKEN_PIPE"
            elif "timeout" in error_msg.lower():
                error_category = "TIMEOUT"
            else:
                error_category = "OTHER"
            
            logger.error(
                f"Error broadcasting {message_type} to connection in room {room_id}: "
                f"[{error_category}] {error_type}: {error_msg}",
                exc_info=True
            )
            disconnected.append(connection)
            failed += 1
    
    # Log broadcast summary
    if successful > 0 or failed > 0:
        logger.debug(
            f"Broadcast {message_type} to room {room_id} completed: "
            f"{successful} successful, {failed} failed, {len(disconnected)} to cleanup"
        )
    
    # Clean up disconnected connections
    for connection in disconnected:
        try:
            self.disconnect(connection, room_id)
        except Exception as cleanup_error:
            logger.warning(f"Error during connection cleanup in room {room_id}: {cleanup_error}", exc_info=True)
```

2. **增强 `safe_broadcast` 包装器**：

```python
async def safe_broadcast(room_id: str, message: Dict[str, Any], context: str):
    """
    Safely broadcast message via WebSocket without interrupting workflow execution.
    """
    message_type = message.get("type", "unknown")
    try:
        # Check if room has active connections
        active_connections = len(manager.active_connections.get(room_id, []))
        logger.debug(
            f"[WF][{task_id}] Broadcasting {message_type} during {context} "
            f"to room {room_id} ({active_connections} active connections)"
        )
        
        await manager.broadcast_to_room(room_id, message)
        
        logger.debug(f"[WF][{task_id}] Successfully broadcast {message_type} during {context}")
        
    except Exception as broadcast_error:
        # Enhanced error logging with error classification
        error_type = type(broadcast_error).__name__
        error_msg = str(broadcast_error)
        
        # Classify error type for better debugging
        if "connection is closed" in error_msg.lower():
            error_category = "CONNECTION_CLOSED"
            log_level = "warning"  # Connection closed is expected in some cases
        elif "broken pipe" in error_msg.lower():
            error_category = "BROKEN_PIPE"
            log_level = "warning"
        elif "timeout" in error_msg.lower():
            error_category = "TIMEOUT"
            log_level = "error"
        else:
            error_category = "OTHER"
            log_level = "error"
        
        # Log with appropriate level
        log_message = (
            f"[WF][{task_id}] Broadcast failed during {context} "
            f"(message_type={message_type}, room={room_id}): "
            f"[{error_category}] {error_type}: {error_msg}"
        )
        
        if log_level == "error":
            logger.error(log_message, exc_info=True)
        else:
            logger.warning(log_message)
        
        # Note: We intentionally do NOT re-raise the exception
        # Workflow execution should continue even if WebSocket broadcast fails
```

3. **增强工作流执行关键点的日志记录**：

```python
# Check WebSocket connection status before starting
workflow_room_id = f"workflow_{task_id}"
initial_connections = len(manager.active_connections.get(workflow_room_id, []))
logger.info(
    f"[WF][{task_id}] Starting workflow execution with {len(request.nodes)} nodes. "
    f"WebSocket room {workflow_room_id} has {initial_connections} active connection(s)"
)

# In node processing loop:
current_connections = len(manager.active_connections.get(workflow_room_id, []))
logger.info(
    f"[WF][{task_id}] Processing node: {node_id} "
    f"(WebSocket connections: {current_connections})"
)

# After node finished broadcast:
connections_after_broadcast = len(manager.active_connections.get(workflow_room_id, []))
if connections_after_broadcast != connections_before_broadcast:
    logger.info(
        f"[WF][{task_id}] WebSocket connection count changed after node_finished broadcast: "
        f"{connections_before_broadcast} -> {connections_after_broadcast}"
    )

# At workflow completion:
final_connections = len(manager.active_connections.get(workflow_room_id, []))
logger.info(
    f"[WF][{task_id}] Workflow execution completed. "
    f"Final WebSocket connections: {final_connections} (started with {initial_connections})"
)
```

**修复后的预期行为**：

1. **连接关闭不会中断工作流执行**：
   - WebSocket 发送失败时，工作流继续执行
   - 所有节点结果仍会写入数据库
   - 前端可以通过轮询 API 获取最终结果

2. **详细的错误日志**：
   - 完整的堆栈跟踪便于定位问题
   - 错误分类（CONNECTION_CLOSED、BROKEN_PIPE、TIMEOUT 等）
   - 连接数统计和状态变化记录

3. **自动连接清理**：
   - 已断开的连接自动清理，避免内存泄漏
   - 连接状态检查防止向已关闭连接发送消息

4. **更好的调试能力**：
   - 日志显示每个节点的广播状态
   - 连接数变化追踪
   - 便于定位连接关闭的具体原因

**验证步骤**：

1. **重新启动后端服务**：
```bash
./bin/stop-dev.sh && ./bin/start-dev.sh
```

2. **测试工作流执行**：
   - 配置 App Node（选择文件）
   - 连接 Start Node → App Node → Data Transform Node
   - 点击 "Run Workflow"
   - 检查后端日志，查看 WebSocket 连接状态和广播日志
   - 即使 WebSocket 连接关闭，工作流也应继续执行
   - 检查任务状态是否从 running 变为 completed
   - 检查 Node Results 是否包含所有节点的结果

3. **测试连接关闭场景**：
   - 在工作流执行过程中刷新浏览器页面（模拟连接关闭）
   - 检查后端日志，确认连接关闭被正确处理
   - 确认工作流继续执行并完成
   - 通过任务详情 API 验证节点结果已正确保存

**技术要点**：

- **连接状态检查**：在发送消息前检查 `client_state`，避免向已关闭连接发送
- **错误分类**：区分不同类型的错误（连接关闭、网络错误等），采用不同的日志级别
- **容错设计**：WebSocket 发送失败不应中断工作流执行，所有结果应写入数据库
- **连接管理**：自动清理已断开的连接，避免内存泄漏和无效连接累积
- **日志记录**：详细的日志帮助追踪连接状态变化和问题定位

**预防措施**：

1. **连接健康检查**：定期检查连接状态，及时清理断开连接
2. **错误处理**：所有 WebSocket 操作都应包含异常处理，确保不影响主流程
3. **日志记录**：在关键点记录连接状态，便于问题追踪
4. **容错设计**：WebSocket 作为实时通知机制，不应影响核心业务逻辑

**相关文件**：

- `backend/app/api/v1/workbench.py`（WebSocket 连接管理、错误处理、日志记录）

---

### 工作流执行问题：Data Transform Node 输出为空（WebSocket 连接关闭导致事件处理中断）

**问题描述**：
- Data Transform Node 执行成功（日志显示 SUCCESS），但前端显示 "No Output Data"
- 节点状态显示为 "IDLE"，没有变为 "completed"
- Docker 日志显示 `Error during workflow stream execution: the connection is closed`
- App Node 有正常输出，但 Data Transform Node 的输出丢失

**问题演进过程**：

#### 第一阶段：WebSocket 连接关闭导致事件处理中断

**错误现象**：
- Data Transform Node 执行成功（日志显示 `SUCCESS keys=['has_qualified_option', ...]`）
- 但前端显示 "No Output Data"
- Docker 日志显示：`Error during workflow stream execution: the connection is closed`
- 没有看到 `Processing node: dataTransform-3` 的日志

**根本原因分析**：

1. **Data Transform Node 执行成功**：
   - 日志显示 `SUCCESS keys=['has_qualified_option', 'qualified_option_names', 'qualified_option_count']`
   - 代码中确实设置了 `node_results[state['current_node_id']]`

2. **事件处理循环未处理该节点**：
   - 日志中有 `Processing node: app-1`，但没有 `Processing node: dataTransform-3`
   - 说明 `astream` 循环在处理 Data Transform Node 的事件前被中断

3. **WebSocket 连接关闭导致中断**：
   - 日志显示：`Error during workflow stream execution: the connection is closed`
   - 发生在 Data Transform Node 执行完成后（日志显示 SUCCESS）
   - `astream` 循环因 WebSocket 连接关闭而中断，后续事件处理未执行

4. **结果未写入数据库**：
   - `workbench.py` 的事件处理逻辑未执行到 Data Transform Node
   - 因此 `node_results` 未更新到数据库，前端无法获取

**问题流程**：

```
1. Data Transform Node 执行成功 ✅
   └─> 设置 state['node_results']['dataTransform-3'] ✅
   
2. LangGraph 准备发送事件到 astream 循环
   └─> 但此时 WebSocket 连接已关闭 ❌
   
3. astream 循环在处理事件时抛出异常
   └─> "the connection is closed" ❌
   
4. 异常处理尝试从 checkpoint 恢复
   └─> 但可能 checkpoint 中的 node_results 还未完全同步 ❌
   
5. 最终状态中缺少 Data Transform Node 的结果 ❌
```

**解决方案**：

1. **在 `astream` 循环中增加容错处理**：
   - 对每个节点的处理增加独立的 try-except
   - 捕获 WebSocket 相关异常但不中断循环
   - 确保节点结果先写入数据库，再尝试广播

2. **增强 checkpoint 恢复逻辑**：
   - 创建独立的 `_recover_state_from_checkpoint` 函数
   - 无论是否有异常，都尝试从 checkpoint 恢复最终状态
   - 确保所有 `node_results` 都被正确合并到 `final_state`
   - 立即将恢复的 `node_results` 持久化到数据库

3. **添加详细日志**：
   - 记录每个节点的事件处理状态
   - 记录 checkpoint 恢复过程
   - 记录数据库持久化状态

**实际采纳方案**：采用上述完整方案

**最终解决方法**：增强事件处理容错和 checkpoint 恢复机制

**具体修改**：

**文件**：`backend/app/api/v1/workbench.py`

1. **增强 `astream` 循环的容错处理**：

```python
# 对每个节点的处理增加独立的 try-except
for node_id, updated_state in event.items():
    try:
        # ... 处理节点 ...
        
        # CRITICAL: 先写入数据库，再尝试广播
        if node_result:
            # 先持久化到数据库
            task.node_results[node_id] = node_result
            async_session.commit()
            
            # 再尝试广播（即使失败也不影响数据持久化）
            await safe_broadcast(...)
    except Exception as node_processing_error:
        # 记录错误但继续处理其他节点
        logger.error(f"Error processing node {node_id}: {error_msg}", exc_info=True)
        # 继续处理下一个节点
```

2. **创建 `_recover_state_from_checkpoint` 辅助函数**：

```python
async def _recover_state_from_checkpoint(
    execution_service: WorkflowExecutionService,
    thread_id: str,
    task_id: str,
    current_final_state: Dict[str, Any],
    async_session: Session
) -> Dict[str, Any]:
    """
    Recover final state from checkpoint and merge with current state.
    This ensures all node_results are captured even if WebSocket connection closed.
    """
    # 1. 从 checkpoint 获取状态
    checkpoint_state = await execution_service.checkpoint.aget(config)
    
    # 2. 合并 node_results（checkpoint 优先）
    final_state["node_results"].update(checkpoint_node_results)
    
    # 3. CRITICAL: 立即持久化到数据库
    task.node_results.update(checkpoint_node_results)
    async_session.commit()
    
    return final_state
```

3. **在异常处理和正常流程中都调用 checkpoint 恢复**：

```python
except Exception as e:
    logger.error(f"Error during workflow stream execution: {e}", exc_info=True)
    # 异常时恢复
    await _recover_state_from_checkpoint(...)

# ALWAYS 尝试恢复，即使没有异常
# 这确保即使 WebSocket 错误静默发生，也能恢复状态
final_state = await _recover_state_from_checkpoint(...)
```

**修复后的预期行为**：

1. **节点结果优先持久化**：
   - 节点结果先写入数据库，再尝试 WebSocket 广播
   - 即使 WebSocket 失败，数据也已保存

2. **容错处理**：
   - 每个节点的处理都有独立的错误处理
   - WebSocket 错误不会中断整个工作流
   - 继续处理后续节点

3. **Checkpoint 恢复**：
   - 无论是否有异常，都尝试从 checkpoint 恢复
   - 确保所有 `node_results` 都被正确合并
   - 立即持久化到数据库

4. **详细日志**：
   - 记录每个节点的事件处理状态
   - 记录 checkpoint 恢复过程
   - 便于调试和问题定位

**验证步骤**：

1. **重新启动后端服务**：
```bash
./bin/stop-dev.sh && ./bin/start-dev.sh
```

2. **测试工作流执行**：
   - 配置 App Node（选择文件）
   - 连接 Start Node → App Node → Data Transform Node
   - 点击 "Run Workflow"
   - 检查后端日志，确认：
     - Data Transform Node 执行成功
     - 看到 `Processing node: dataTransform-3` 日志
     - 看到 `Successfully persisted node_results for dataTransform-3 to database` 日志
     - 看到 checkpoint 恢复日志
   - 检查前端，确认 Data Transform Node 有输出
   - 检查任务状态，确认所有节点的结果都在 `node_results` 中

3. **测试 WebSocket 连接关闭场景**：
   - 在工作流执行过程中刷新浏览器页面（模拟连接关闭）
   - 检查后端日志，确认：
     - 节点结果已写入数据库
     - checkpoint 恢复成功
     - 所有节点结果都被正确合并
   - 通过任务详情 API 验证节点结果已正确保存

**技术要点**：

- **数据持久化优先**：节点结果先写入数据库，再尝试 WebSocket 广播
- **容错设计**：每个节点的处理都有独立的错误处理，不会相互影响
- **Checkpoint 恢复**：无论是否有异常，都尝试从 checkpoint 恢复最终状态
- **日志记录**：详细的日志帮助追踪每个节点的处理状态和 checkpoint 恢复过程

**预防措施**：

1. **数据持久化优先**：确保关键数据先写入数据库，再尝试实时通知
2. **容错处理**：每个节点的处理都要有独立的错误处理
3. **Checkpoint 恢复**：工作流执行完成后，总是尝试从 checkpoint 恢复最终状态
4. **日志记录**：在关键点记录详细日志，便于问题追踪

**相关文件**：

- `backend/app/api/v1/workbench.py`（事件处理容错、checkpoint 恢复、数据持久化）

---

### 工作流执行问题：Data Transform Node 和 Condition Node 无输出 + App Node 超时问题

**问题描述**：

1. **Data Transform Node 执行成功但无输出**：
   - Data Transform Node 执行成功（日志显示 `SUCCESS keys=['is_request_approved', ...]`）
   - 但随后立即报错：`Error during workflow stream execution: 'True'`
   - 导致 Data Transform Node 和 Condition Node 都没有显示 output 结果
   - 前端显示 "No Output Data"

2. **App Node DAG 执行超时**：
   - App Node 等待 DAG 执行超时，报错：`DAG execution timeout after 300 seconds`
   - 某些 Airflow DAG 实例运行时间会超过 300 秒，导致工作流失败

**问题演进过程**：

#### 第一部分：Data Transform Node 路由错误导致无输出

**错误现象**：
```
2025-12-14 09:44:04,220 - app.api.workbench - ERROR - workbench.py:1046 - [WF][db5b3cfd-7bb0-44b5-9eae-753844c84ac2] Error during workflow stream execution: 'True'
Traceback (most recent call last):
  ...
  File "/usr/local/lib/python3.10/site-packages/langgraph/graph/graph.py", line 130, in <listcomp>
    r if isinstance(r, Send) else self.ends[r] for r in result
KeyError: 'True'
```

**根本原因分析**：

1. **Data Transform Node 执行成功**：
   - 日志显示 `[WF][dataTransform] node_id=dataTransform-4 SUCCESS keys=[...]`
   - 代码中确实设置了 `node_results[state['current_node_id']]`

2. **路由函数返回无效值**：
   - Condition 节点的路由函数 `_get_condition_function` 返回了 `'True'` 字符串
   - 但这个返回值不在 LangGraph 的 `routing_map` 的 key 集合中
   - LangGraph 尝试在 `self.ends` 字典中查找 `'True'`，但找不到（KeyError）

3. **问题根源**：
   - `_get_condition_function` 方法从 condition 节点的 output 中读取 `branch` 值
   - 如果 condition 节点尚未执行或返回值不匹配 `routing_map` 的 key，就会导致 KeyError
   - 错误发生在 LangGraph 路由阶段，导致后续节点处理中断，结果无法保存到数据库

**解决方案**：

修改 `_get_condition_function` 方法，确保返回值始终在 `routing_map` 的有效 key 集合中。

**具体修改**：

**文件**：`backend/app/services/workflow/workflow_execution_service.py`

1. **修改 `_get_condition_function` 方法签名和实现**（第730-749行）：

```python
# 修改前：
def _get_condition_function(self, node: dict):
    """Get condition function for conditional edges"""
    def condition_func(state: WorkflowState) -> str:
        node_id = node['id']
        node_result = state.get('node_results', {}).get(node_id, {})
        output = node_result.get('output', {})
        branch = output.get('branch', 'False')
        logger.debug(f"[WF] Condition function for node {node_id} returned branch: {branch}")
        return branch
    return condition_func

# 修改后：
def _get_condition_function(self, node: dict, routing_map: dict):
    """Get condition function for conditional edges"""
    # Get valid routing keys from routing_map
    valid_keys = set(routing_map.keys())
    # Determine default key: prefer 'False', then 'True', then first available key
    if 'False' in valid_keys:
        default_key = 'False'
    elif 'True' in valid_keys:
        default_key = 'True'
    elif valid_keys:
        default_key = list(valid_keys)[0]
    else:
        default_key = 'False'  # Fallback, should not happen
    
    logger.debug(
        f"[WF] Creating condition function for node {node['id']} with routing_map keys: {valid_keys}, "
        f"default_key: {default_key}"
    )
    
    def condition_func(state: WorkflowState) -> str:
        """
        Evaluate condition and return branch label ('True' or 'False')
        
        This function is called by LangGraph to determine which edge to follow
        after a Condition Node execution.
        
        Returns:
            A string key that MUST exist in routing_map to avoid KeyError
        """
        node_id = node['id']
        node_result = state.get('node_results', {}).get(node_id, {})
        
        # Check if condition node has been executed
        if not node_result or node_result.get('status') != 'completed':
            logger.warning(
                f"[WF] Condition function for node {node_id} called but node not yet executed. "
                f"Using default key '{default_key}'"
            )
            return default_key
        
        output = node_result.get('output', {})
        branch = output.get('branch', default_key)
        
        # CRITICAL: Ensure returned value exists in routing_map to avoid KeyError
        if branch not in valid_keys:
            logger.error(
                f"[WF] Condition function for node {node_id} returned branch '{branch}' "
                f"which is NOT in routing_map keys {valid_keys}. This will cause KeyError! "
                f"Using default '{default_key}' instead."
            )
            branch = default_key
        
        logger.info(
            f"[WF] Condition function for node {node_id} returning branch: '{branch}' "
            f"(valid_keys: {valid_keys})"
        )
        return branch
    
    return condition_func
```

2. **更新调用处，传入 routing_map 参数**（第346-352行）：

```python
# 修改前：
workflow.add_conditional_edges(
    source_id,
    self._get_condition_function(source_node),
    routing_map
)

# 修改后：
workflow.add_conditional_edges(
    source_id,
    self._get_condition_function(source_node, routing_map),
    routing_map
)
logger.info(f"[WF] Added conditional edges from condition node {source_id}: {routing_map}")
```

3. **增强路由 map 构建日志**（第335-344行）：

```python
# 添加日志记录
logger.info(
    f"[WF] Condition node {source_id} routing_map: {routing_map} "
    f"(true_target: {true_target}, false_target: {false_target})"
)
```

**修改要点**：

1. **验证返回值有效性**：确保路由函数返回的值在 `routing_map` 的 key 集合中
2. **默认值处理**：如果返回值无效，使用安全的默认值（优先 'False'，然后是 'True'）
3. **错误处理**：处理 condition 节点尚未执行的情况
4. **详细日志**：记录路由函数的执行过程和返回值，便于调试

**验证步骤**：

1. 重新启动后端服务
2. 测试工作流：Start Node → App Node → Data Transform Node → Condition Node → End Node
3. 检查日志，确认：
   - Data Transform Node 执行成功
   - Condition Node 执行成功
   - 没有 KeyError: 'True' 错误
   - 所有节点结果都正确保存到数据库
4. 检查前端，确认 Data Transform Node 和 Condition Node 都有输出显示

---

#### 第二部分：App Node DAG 执行超时时间调整

**错误现象**：
```
{
  "error": "DAG execution timeout after 300 seconds (dag_run_id: manual_2025-12-13T13:18:55.535586+00:00)",
  "dag_run_id": "manual_2025-12-13T13:18:55.535586+00:00",
  "input_received": {},
  "conclusion_detail": ""
}
```

**根本原因**：
- App Node 等待 DAG 执行完成的最大超时时间设置为 300 秒（5 分钟）
- 某些 Airflow DAG 实例运行时间会超过 300 秒，导致工作流在 DAG 完成前就超时失败

**解决方案**：
将 App Node 的 DAG 执行超时时间从 300 秒增加到 600 秒（10 分钟）。

**具体修改**：

**文件**：`backend/app/services/workflow/workflow_execution_service.py`

1. **修改 `_poll_dag_status` 方法调用处的超时参数**（第436行）：

```python
# 修改前：
result = await self._poll_dag_status(app_node_id, dag_run_id, session, max_wait_time=300)

# 修改后：
result = await self._poll_dag_status(app_node_id, dag_run_id, session, max_wait_time=600)
```

2. **修改 `_poll_dag_status` 方法的默认参数**（第575行）：

```python
# 修改前：
async def _poll_dag_status(self, app_node_id: int, dag_run_id: str, session, max_wait_time: int = 300) -> dict:
    """
    Poll DAG status until completion with fast failure on errors
    
    CRITICAL: Shortened timeout to 300s and added fast failure for 404/errors
    to prevent blocking workflow execution when DAG doesn't exist or fails.
    """

# 修改后：
async def _poll_dag_status(self, app_node_id: int, dag_run_id: str, session, max_wait_time: int = 600) -> dict:
    """
    Poll DAG status until completion with fast failure on errors
    
    CRITICAL: Timeout set to 600s and added fast failure for 404/errors
    to prevent blocking workflow execution when DAG doesn't exist or fails.
    """
```

**修改要点**：

1. **超时时间翻倍**：从 300 秒（5 分钟）增加到 600 秒（10 分钟）
2. **同步更新默认值和注释**：确保代码一致性

**影响评估**：

- ✅ 对于执行时间超过 300s 但小于 600s 的 DAG，不会再因为超时失败
- ⚠️ 工作流整体执行时间可能会增加（最长等待时间从 5 分钟增加到 10 分钟）
- ✅ 不影响快速完成的 DAG（仍然会在 DAG 完成后立即继续）

**验证步骤**：

1. 重新启动后端服务
2. 测试工作流，使用已知执行时间超过 300 秒的 DAG
3. 确认 DAG 在 300-600 秒之间完成时，工作流不会超时
4. 确认 DAG 完成后，App Node 能正常获取结果并继续执行

---

#### 第三部分：Node Results 显示优化

**问题描述**：
- Node Results 面板显示节点的详细 output JSON 数据，信息过多
- 用户希望只显示节点名和状态，隐藏 output 详情，但保留 error 信息

**解决方案**：
修改前端代码，移除 output 显示逻辑，保留节点名、状态和 error 信息。

**具体修改**：

**文件**：`frontend/src/components/Workbench/FlowWorkspace.tsx`

**修改位置**：第2138-2160行

```typescript
// 修改前：
const outputText = result?.output
  ? (typeof result.output === 'string'
      ? result.output
      : JSON.stringify(result.output, null, 2))
  : ''
return (
  <Box key={nid} borderBottom="1px solid" borderColor="gray.100" pb={2} mb={2} _last={{ borderBottom: 'none', pb: 0, mb: 0 }}>
    <HStack justify="space-between">
      <Text fontSize="xs" fontWeight="semibold">{nodeLabel}</Text>
      <Badge colorScheme={statusColor} fontSize="10px">{(result?.status || 'unknown').toUpperCase()}</Badge>
    </HStack>
    {outputText && (
      <Box mt={1} bg="gray.50" border="1px solid" borderColor="gray.100" borderRadius="sm" p={1} maxH="120px" overflowY="auto">
        <Text fontSize="10px" fontFamily="mono" whiteSpace="pre-wrap">
          {outputText}
        </Text>
      </Box>
    )}
    {result?.error && (
      <Text fontSize="10px" color="red.500" mt={1} whiteSpace="pre-wrap">
        {result.error}
      </Text>
    )}
  </Box>
)

// 修改后：
return (
  <Box key={nid} borderBottom="1px solid" borderColor="gray.100" pb={2} mb={2} _last={{ borderBottom: 'none', pb: 0, mb: 0 }}>
    <HStack justify="space-between">
      <Text fontSize="xs" fontWeight="semibold">{nodeLabel}</Text>
      <Badge colorScheme={statusColor} fontSize="10px">{(result?.status || 'unknown').toUpperCase()}</Badge>
    </HStack>
    {/* Output display removed - only show node name and status */}
    {result?.error && (
      <Text fontSize="10px" color="red.500" mt={1} whiteSpace="pre-wrap">
        {result.error}
      </Text>
    )}
  </Box>
)
```

**修改要点**：

1. **移除 output 显示**：删除 `outputText` 变量的定义和使用
2. **保留错误信息**：保留 `error` 信息的显示，便于排查问题
3. **简化界面**：Node Results 面板只显示节点名和状态，更加简洁

**显示效果**：

- ✅ 显示：节点名（如 "Start Node", "Data Transform Node"）
- ✅ 显示：状态（COMPLETED / FAILED / RUNNING）
- ❌ 不显示：output JSON 数据
- ✅ 显示：error 信息（如果节点执行失败）

**相关文件**：

- `backend/app/services/workflow/workflow_execution_service.py`（路由函数修复、超时时间调整）
- `frontend/src/components/Workbench/FlowWorkspace.tsx`（Node Results 显示优化）

---

### 工作流审批消息中心显示问题：新消息立即显示为Overdue + NEW标签显示错误

**问题描述**：

1. **新消息立即显示为Overdue**：
   - 刚创建的workflow审批消息，配置的超时时间是1小时
   - 但消息一创建就出现在Message Center的Overdue栏，而不是Todo栏的Pending列
   - 消息应该等待1小时后才显示为Overdue

2. **NEW标签显示在旧消息上**：
   - 最新创建的消息zzzz777没有显示NEW标签
   - 而旧消息zzzz666显示了NEW标签
   - NEW标签应该显示在最新未读消息上

**问题演进过程**：

#### 第一阶段：数据库查询确认问题根源

**数据库查询结果**：
- `due_date`字段类型：`timestamp without time zone`（无时区信息）
- `create_time`字段类型：`timestamp without time zone`（无时区信息）
- 实际数据：
  - zzzz777: `due_date` = 2025-12-28 00:47:36, `create_time` = 2025-12-27 23:47:36, `is_read` = true
  - zzzz666: `due_date` = 2025-12-27 23:53:48, `create_time` = 2025-12-27 22:53:48, `is_read` = true
- 数据库判断：zzzz777未过期（`is_overdue_db = false`），zzzz666已过期（`is_overdue_db = true`）

**根本原因分析**：

1. **时区处理不一致**：
   - 后端创建时使用 `datetime.now(timezone.utc)`（带时区）
   - 数据库字段是 `timestamp without time zone`（无时区）
   - 存储到数据库时，PostgreSQL会去掉时区信息
   - 如果数据库服务器时区是UTC+8，存储的UTC时间可能被解释为UTC+8时间

2. **前端解析问题**：
   - 前端使用 `new Date(task.due_date)` 解析
   - 如果后端返回的字符串没有`Z`后缀，JavaScript会当作本地时间解析
   - 导致时区转换错误，未来时间被误判为过去时间

3. **排序逻辑问题**：
   - 排序优先级中，`create_time`的优先级较低（排在第5位）
   - 导致最新未读消息不一定显示在最前面

**建议方案**：

1. **方案1：修复数据库字段类型（推荐，但需要迁移）**
   - 将 `timestamp without time zone` 改为 `timestamp with time zone`
   - 优点：从根源解决时区问题
   - 缺点：需要数据库迁移，可能影响现有数据

2. **方案2：修复后端序列化（快速修复）**
   - 确保返回的datetime字符串包含时区信息（添加`Z`后缀）
   - 优点：不需要数据库迁移
   - 缺点：数据库存储仍无时区信息，但API返回正确

3. **方案3：修复前端解析（临时方案）**
   - 增强前端`isOverdue`函数，正确处理无时区的时间字符串
   - 优点：不需要修改后端
   - 缺点：治标不治本

4. **方案4：调整排序逻辑**
   - 提高`create_time`的排序优先级，让最新未读消息优先显示

**实际采纳方案**：组合方案（方案2 + 方案3 + 方案4）

- ✅ 短期：修复后端序列化，确保API返回的datetime包含时区信息（添加`Z`后缀）
- ✅ 中期：修复前端解析，增强时区处理逻辑
- ✅ 调整排序逻辑，让最新未读消息优先显示
- ❌ 不进行数据库迁移（保持现有字段类型）

**最终解决方法**：

#### 1. 修复后端序列化

**文件**：`backend/app/api/v1/steering_tasks.py`

**修改位置**：第192-230行

```python
# 添加ensure_utc_datetime辅助函数
def ensure_utc_datetime(dt):
    """Ensure datetime is timezone-aware and in UTC, return ISO format with Z suffix"""
    if dt is None:
        return None
    if isinstance(dt, datetime):
        # If naive, assume UTC
        if dt.tzinfo is None:
            dt = dt.replace(tzinfo=timezone.utc)
        # Convert to UTC if not already
        if dt.tzinfo != timezone.utc:
            dt = dt.astimezone(timezone.utc)
        # Return ISO format with Z suffix to indicate UTC
        return dt.isoformat().replace('+00:00', 'Z')
    return dt

# 创建serialize_task函数，确保所有datetime字段都包含时区信息
def serialize_task(task: SteeringTask) -> SteeringTaskRead:
    """Serialize task with proper datetime timezone handling"""
    task_dict = task.model_dump()
    # Fix datetime fields to include timezone information (Z suffix for UTC)
    if task_dict.get('due_date'):
        task_dict['due_date'] = ensure_utc_datetime(task.due_date)
    if task_dict.get('create_time'):
        task_dict['create_time'] = ensure_utc_datetime(task.create_time)
    if task_dict.get('update_time'):
        task_dict['update_time'] = ensure_utc_datetime(task.update_time)
    if task_dict.get('read_at'):
        task_dict['read_at'] = ensure_utc_datetime(task.read_at)
    if task_dict.get('completed_at'):
        task_dict['completed_at'] = ensure_utc_datetime(task.completed_at)
    return SteeringTaskRead(**task_dict)

# 在list_steering_tasks函数中使用serialize_task
result = [serialize_task(task) for task in paginated_tasks]
return result
```

#### 2. 修复前端解析

**文件**：
- `frontend/src/components/MessageCenter/MessageCenterBoard.tsx`
- `frontend/src/components/MessageCenter/TaskCard.tsx`

**修改位置**：`MessageCenterBoard.tsx` 第24-72行，`TaskCard.tsx` 第106-126行

```typescript
// 修改前：
const isOverdue = (task: SteeringTask): boolean => {
  if (!task.due_date) return false
  if (task.status === "completed" || task.status === "cancelled") return false
  try {
    const dueDate = new Date(task.due_date)
    const now = new Date()
    const dueTime = dueDate.getTime()
    const nowTime = now.getTime()
    return dueTime < nowTime
  } catch {
    return false
  }
}

// 修改后：
const isOverdue = (task: SteeringTask): boolean => {
  if (!task.due_date) return false
  if (task.status === "completed" || task.status === "cancelled") return false
  try {
    // Parse due_date - handle both UTC (with Z) and naive datetime strings
    let dueDate: Date
    const dueDateStr = String(task.due_date)
    
    // Check if string ends with Z (UTC indicator) or has timezone offset
    if (dueDateStr.endsWith('Z') || dueDateStr.includes('+') || dueDateStr.includes('-', 10)) {
      // Has timezone information - parse directly
      dueDate = new Date(dueDateStr)
    } else {
      // Naive datetime string - assume it's UTC and append Z
      // This handles cases where backend returns datetime without timezone info
      dueDate = new Date(dueDateStr.endsWith('Z') ? dueDateStr : dueDateStr + 'Z')
    }
    
    // Validate parsed date
    if (isNaN(dueDate.getTime())) {
      console.warn(`Invalid due_date format: ${task.due_date}`)
      return false
    }
    
    // Get current time (always UTC internally)
    const now = new Date()
    
    // Compare using UTC timestamps
    const dueTime = dueDate.getTime()
    const nowTime = now.getTime()
    
    // Task is overdue if due time is in the past
    const isOverdue = dueTime < nowTime
    
    // Debug logging (can be removed in production)
    if (process.env.NODE_ENV === 'development' && isOverdue) {
      console.debug(`Task ${task.id} (${task.title}) is overdue:`, {
        due_date: task.due_date,
        dueTime,
        nowTime,
        diff_seconds: Math.floor((nowTime - dueTime) / 1000)
      })
    }
    
    return isOverdue
  } catch (error) {
    console.error(`Error checking overdue for task ${task.id}:`, error)
    return false
  }
}
```

#### 3. 调整排序逻辑

**文件**：`backend/app/api/v1/steering_tasks.py`

**修改位置**：第162-185行

```python
# 修改前：
# Sort by:
# 1. Overdue status
# 2. is_read (unread first)
# 3. Priority descending
# 4. Due date ascending
# 5. Created_at descending (newest first)

return (
    not is_overdue,  # False (overdue) < True (not overdue)
    task.is_read,  # False (unread) comes before True (read)
    -priority_order.get(task.priority, 0),  # Negative for descending
    due_date_stamp,  # Ascending order
    -create_time_stamp  # Negative for descending
)

# 修改后：
# Sort by:
# 1. Overdue status
# 2. is_read (unread first)
# 3. Created_at descending (newest first) - NEW: Latest unread messages first
# 4. Priority descending
# 5. Due date ascending

return (
    not is_overdue,  # False (overdue) < True (not overdue)
    task.is_read,  # False (unread) comes before True (read)
    -create_time_stamp,  # Negative for descending (newer first) - NEW: Latest first
    -priority_order.get(task.priority, 0),  # Negative for descending
    due_date_stamp,  # Ascending order
)
```

**修复后的预期行为**：

1. **Overdue判断正确**：
   - 后端返回的datetime包含`Z`后缀（UTC标识）
   - 前端正确解析UTC时间
   - 避免时区转换错误，新消息不会立即显示为Overdue

2. **NEW标签显示正确**：
   - 排序逻辑确保最新未读消息优先显示
   - 如果消息被点击过（`is_read=true`），不会显示NEW标签（符合预期）

**验证步骤**：

1. **创建新的workflow审批任务**，验证：
   - 消息出现在Todo栏的Pending列，而不是Overdue列
   - 最新未读消息显示在最前面
   - NEW标签显示在最新未读消息上

2. **等待超时后验证**：
   - 消息正确移动到Overdue列

3. **点击消息后验证**：
   - NEW标签消失
   - `is_read`状态更新

**技术要点**：

- **时区处理一致性**：后端和前端必须统一使用UTC时间，并通过`Z`后缀明确标识
- **数据库字段类型**：`timestamp without time zone`需要特别注意时区处理
- **前端解析增强**：需要处理带时区和不带时区的datetime字符串
- **排序逻辑优化**：确保最新未读消息优先显示，提升用户体验

**预防措施**：

1. **时区统一**：所有时间字段统一使用UTC，并在序列化时添加`Z`后缀
2. **前端容错**：前端解析datetime时，需要处理多种格式（带时区、不带时区）
3. **排序优化**：排序逻辑应该优先考虑用户体验（最新未读消息优先）
4. **测试验证**：每次修改后都要测试时区相关的边界情况

**相关文件**：

- `backend/app/api/v1/steering_tasks.py`（序列化修复、排序逻辑调整）
- `frontend/src/components/MessageCenter/MessageCenterBoard.tsx`（前端解析增强）
- `frontend/src/components/MessageCenter/TaskCard.tsx`（前端解析增强）

---

### 工作流模板执行历史查询问题：Show Run History 返回 500 错误

**问题描述**：
- 运行 workflow template 成功后，点击 "Show Run History" 按钮
- 前端显示错误："Failed to fetch workflow instances: 500"
- 后端日志显示 workflow 执行成功，创建了 `WorkflowInstance 9`
- 但查询历史记录时返回 500 错误

**问题现象**：
- 前端错误提示：`Error: Failed to fetch workflow instances: 500`
- 后端日志显示 workflow 执行成功：
  ```
  [WF][f0caa79c-862e-489c-9797-44adf654cc93] Updated WorkflowInstance 9 with final results
  [WF][f0caa79c-862e-489c-9797-44adf654cc93] Workflow execution completed successfully
  ```
- 但查询 `/api/v1/workbench/workflow-instances?app_id=105` 时返回 500 错误

**根本原因分析**：

1. **UUID 序列化失败**：
   - `WorkflowInstance.created_by` 字段类型是 `uuid.UUID | None`
   - `JSONResponse` 无法直接序列化 UUID 对象
   - 当尝试序列化包含 UUID 的字典时，FastAPI 抛出异常，导致 500 错误

2. **错误日志缺少统一前缀**：
   - 错误日志没有使用 `[WF]` 前缀，不符合统一日志规范
   - 缺少查询参数和用户信息，难以定位问题

3. **查询参数问题**（已修复）：
   - 前端最初使用 `template_id` 参数查询，但后端支持的是 `app_id`
   - 已修复为使用 `app_id` 参数

**建议方案**：

1. **修复 UUID 序列化问题**（必须）：
   - 在返回 JSON 响应前，将 `created_by` 字段转换为字符串
   - 使用 `str(instance.created_by) if instance.created_by else None`

2. **统一日志前缀**（必须）：
   - 为错误日志添加 `[WF][GET_INSTANCES]` 前缀
   - 增强日志信息，包含查询参数和用户信息

3. **预防性修复**（推荐）：
   - 同时修复 `/workflow-instances/{instance_id}` 端点的 UUID 序列化问题
   - 确保所有返回 `WorkflowInstance` 的端点都正确处理 UUID

**实际采纳方案**：完全采纳所有建议方案

**最终解决方法**：

#### 1. 修复 `/workflow-instances` 端点的 UUID 序列化

**文件**：`backend/app/api/v1/workbench.py`

**修改位置**：第 2216 行

```python
# 修改前：
result.append({
    "id": instance.id,
    "template_id": instance.template_id,
    "app_id": instance.app_id,
    "status": instance.status,
    "current_node_id": instance.current_node_id,
    "created_by": instance.created_by,  # ❌ UUID 对象无法序列化
    "create_time": instance.create_time.isoformat() if instance.create_time else None,
    "update_time": instance.update_time.isoformat() if instance.update_time else None,
    "execution_time_seconds": execution_time
})

# 修改后：
result.append({
    "id": instance.id,
    "template_id": instance.template_id,
    "app_id": instance.app_id,
    "status": instance.status,
    "current_node_id": instance.current_node_id,
    "created_by": str(instance.created_by) if instance.created_by else None,  # ✅ 转换为字符串
    "create_time": instance.create_time.isoformat() if instance.create_time else None,
    "update_time": instance.update_time.isoformat() if instance.update_time else None,
    "execution_time_seconds": execution_time
})
```

#### 2. 修复错误日志

**文件**：`backend/app/api/v1/workbench.py`

**修改位置**：第 2230 行

```python
# 修改前：
except Exception as e:
    logger.error(f"Error getting workflow instances: {e}", exc_info=True)
    raise HTTPException(status_code=500, detail=f"Failed to get workflow instances: {str(e)}")

# 修改后：
except Exception as e:
    logger.error(f"[WF][GET_INSTANCES] ❌ Error getting workflow instances: user_id={current_user.id}, template_id={template_id}, app_id={app_id}, skip={skip}, limit={limit}, error={e}", exc_info=True)
    raise HTTPException(status_code=500, detail=f"Failed to get workflow instances: {str(e)}")
```

#### 3. 预防性修复 `/workflow-instances/{instance_id}` 端点

**文件**：`backend/app/api/v1/workbench.py`

**修改位置**：第 2276 行

```python
# 修改前：
result = {
    "id": instance.id,
    "template_id": instance.template_id,
    "app_id": instance.app_id,
    "status": instance.status,
    "current_node_id": instance.current_node_id,
    "execution_result": instance.execution_result,
    "created_by": instance.created_by,  # ❌ UUID 对象无法序列化
    "create_time": instance.create_time.isoformat() if instance.create_time else None,
    "update_time": instance.update_time.isoformat() if instance.update_time else None,
}

# 修改后：
result = {
    "id": instance.id,
    "template_id": instance.template_id,
    "app_id": instance.app_id,
    "status": instance.status,
    "current_node_id": instance.current_node_id,
    "execution_result": instance.execution_result,
    "created_by": str(instance.created_by) if instance.created_by else None,  # ✅ 转换为字符串
    "create_time": instance.create_time.isoformat() if instance.create_time else None,
    "update_time": instance.update_time.isoformat() if instance.update_time else None,
}
```

**修复后的预期行为**：

1. **UUID 正确序列化**：
   - `created_by` 字段转换为字符串格式
   - JSON 响应可以正常序列化，不再返回 500 错误

2. **统一日志格式**：
   - 所有 workflow 相关日志使用 `[WF]` 前缀
   - 错误日志包含完整的上下文信息（用户ID、查询参数等）
   - 便于使用 `docker logs -f foundation-backend-container 2>&1 | grep -E "\[WF\]"` 过滤日志

3. **查询历史正常显示**：
   - 前端可以正常获取 workflow 执行历史
   - "Show Run History" 功能正常工作

**验证步骤**：

1. **重新启动后端服务**：
```bash
./bin/stop-dev.sh && ./bin/start-dev.sh
```

2. **测试 workflow template 执行**：
   - 从 Application Marketplace 加载 workflow template
   - 配置并运行 workflow template
   - 等待执行完成

3. **测试 Show Run History**：
   - 点击 "Show Run History" 按钮
   - 确认不再出现 500 错误
   - 确认执行历史正常显示

4. **验证日志**：
```bash
docker logs -f foundation-backend-container 2>&1 | grep -E "\[WF\]"
```
   - 确认所有 workflow 相关日志都有 `[WF]` 前缀
   - 确认错误日志包含完整的上下文信息

**技术要点**：

- **UUID 序列化**：FastAPI 的 `JSONResponse` 无法直接序列化 Python 的 `uuid.UUID` 对象，需要转换为字符串
- **统一日志规范**：所有 workflow 相关操作使用 `[WF]` 前缀，便于日志过滤和问题定位
- **预防性修复**：同时修复所有可能返回 UUID 字段的端点，避免类似问题

**预防措施**：

1. **类型检查**：在返回 JSON 响应前，检查所有字段类型，确保可以序列化
2. **统一序列化**：创建统一的序列化函数，处理 UUID、datetime 等特殊类型
3. **日志规范**：所有 workflow 相关日志统一使用 `[WF]` 前缀
4. **测试验证**：每次修改后都要测试 JSON 序列化是否正常

**相关文件**：

- `backend/app/api/v1/workbench.py`（UUID 序列化修复、错误日志增强）
- `frontend/src/components/Workbench/WorkflowRunHistoryModal.tsx`（查询参数修复，已在前一次修改中完成）

---

### 问题：Workflow Template 节点配置覆盖不生效及前端 Response 读取错误

**问题描述**：
1. 从 workflow template 创建实例时，修改了 humanInTheLoop node 的 taskType 为 `request_for_update`，但执行时仍使用 template 中的旧配置（`approval`）
2. 前端报错："Failed to execute 'text' on 'Response': body stream already read"

**问题发生时间**：2025年1月13日

**问题现象**：

#### 现象1：配置覆盖不生效
- 用户在 UI 中将 humanInTheLoop node 的 taskType 改为 `request_for_update`
- 运行 workflow 后，消息中心显示的还是 `request_for_approval` 任务
- 日志显示：`taskType=approval`，`has_updateConfig=False`

#### 现象2：前端 Response 读取错误
- 运行 workflow 时，前端控制台报错：`Failed to execute 'text' on 'Response': body stream already read`
- 导致 workflow 执行结果无法正确显示

**根本原因分析**：

#### 原因1：配置覆盖机制缺失
**现象层**：
- 前端修改了节点配置，但后端执行时使用了 template 的原始配置

**本质层**：
1. **架构缺陷**：`create_workflow_instance` API 只支持 `app_node_configs` 覆盖，不支持其他节点类型（如 `humanInTheLoop`、`condition`）的配置覆盖
2. **数据流问题**：
   - 前端在 `FlowWorkspace.tsx` 中只收集了 `app_node_configs`
   - 后端直接从 `template.workflow_nodes` 加载节点，忽略了 UI 中的修改
   - 配置覆盖逻辑完全缺失

**哲学洞察**：
> "配置应该分层：Template 提供结构，Instance 提供运行时覆盖"

#### 原因2：Response Body 重复读取
**现象层**：
- 前端错误处理中多次读取同一个 Response 对象

**本质层**：
1. **设计缺陷**：错误处理逻辑中，先尝试 `response.json()`，失败后尝试 `response.text()`，但此时 body stream 已被消费
2. **防御性编程缺失**：没有使用 `response.clone()` 创建副本

**哲学洞察**：
> "资源只能消费一次，需要多次访问时应该先克隆"

**建议方案**：

#### 方案A：节点配置覆盖机制（推荐）
- 在 `WorkflowInstanceCreateRequest` 中添加 `node_configs` 字段
- 后端应用配置覆盖：从 template 加载 nodes，然后应用 `node_configs` 覆盖
- 前端收集所有修改过的节点配置（condition、humanInTheLoop）并发送

**优点**：
- 保持 template 结构不变
- 支持灵活的运行时配置覆盖
- 可扩展支持其他节点类型

#### 方案B：发送完整 nodes/edges
- 前端发送完整的修改后的 `nodes` 和 `edges` 列表

**缺点**：
- 可能允许结构变更（非预期行为）
- 实现简单但不够安全

#### 方案C：前端 Response 克隆
- 使用 `response.clone()` 创建副本，避免重复读取

**实际采纳的方案**：

1. **配置覆盖**：采用方案A
   - `node_configs` 格式：选项1（发送完整的 `node.data`）
   - 支持覆盖：`condition` node 和 `humanInTheLoop` node
   - 不支持：`dataTransform` node（已弃用）

2. **Response 处理**：采用方案C
   - 所有错误处理使用 `response.clone()` 机制

3. **深拷贝修复**：
   - 使用 `copy.deepcopy()` 替代 `node.copy()`，确保完全独立的节点副本

4. **日志统一**：
   - 统一所有 workflow 日志格式为 `[WF]` 前缀
   - 修复遗漏的日志前缀

**具体修改**：

#### 1. 后端 API 修改

**文件**：`backend/app/api/v1/workbench.py`

**修改1：添加 node_configs 字段**
```python
# 位置：第2055-2060行
class WorkflowInstanceCreateRequest(BaseModel):
    """Create workflow instance request"""
    template_id: int
    app_id: Optional[int] = None
    app_node_configs: Dict[str, Any] = {}
    node_configs: Optional[Dict[str, Dict[str, Any]]] = None  # 新增：节点配置覆盖
    input_data: Dict[str, Any] = {}
```

**修改2：实现配置覆盖逻辑**
```python
# 位置：第2098-2146行
# 2. Apply node configuration overrides
import copy
nodes = [copy.deepcopy(node) for node in template.workflow_nodes]  # 深拷贝
if request.node_configs:
    logger.info(f"[WF][CREATE_INSTANCE] Applying node configuration overrides: {len(request.node_configs)} nodes")
    for node_id, node_config in request.node_configs.items():
        node = next((n for n in nodes if n.get('id') == node_id), None)
        if node:
            node_type = node.get('type', '')
            original_data = node.get('data', {})
            
            # Merge configuration into node.data
            node['data'] = {**original_data, **node_config}
            
            # 详细日志记录...
            # 特殊处理 humanInTheLoop 和 condition 节点...
```

**修改3：使用覆盖后的 nodes**
```python
# 位置：第2165-2195行
# 5. Create WorkflowExecutionTask (use nodes with config overrides applied)
task_dict = {
    "task_id": task_id,
    "instance_id": instance.id,
    "nodes": nodes,  # 使用覆盖后的 nodes
    "edges": template.workflow_edges,
    # ...
}

# 6. Prepare execution request (use nodes with config overrides applied)
execution_request = WorkflowExecuteRequest(
    nodes=nodes,  # 使用覆盖后的 nodes
    # ...
)
```

**修改4：增强异常处理和日志**
```python
# 位置：第2176-2195行
try:
    task = WorkflowExecutionTask.model_validate(task_dict)
    session.add(task)
    session.commit()
    # ...
except Exception as task_error:
    logger.error(
        f"[WF][CREATE_INSTANCE] ❌ Failed to create WorkflowExecutionTask: task_id={task_id}, error={task_error}",
        exc_info=True
    )
    session.rollback()
    raise HTTPException(
        status_code=500,
        detail=f"Failed to create workflow execution task: {str(task_error)}"
    )
```

**修改5：修复日志前缀**
```python
# 位置：第892行
# 修改前：
logger.error(f"Task {task_id} not found")

# 修改后：
logger.error(f"[WF][{task_id}] Task not found in database")
```

#### 2. 前端修改

**文件**：`frontend/src/components/Workbench/FlowWorkspace.tsx`

**修改1：收集节点配置覆盖**
```typescript
// 位置：第916-944行
// 4. Collect node configuration overrides (for condition, humanInTheLoop nodes)
const nodeConfigs: Record<string, any> = {}
if (isTemplateMode) {
  for (const node of nodes) {
    // Collect configs for condition and humanInTheLoop nodes
    if (node.type === 'condition' || node.type === 'humanInTheLoop') {
      // Send complete node.data as override
      if (node.data) {
        nodeConfigs[node.id] = node.data
        console.log(`[Workflow] Collected node config override for ${node.type} node ${node.id}:`, {
          nodeId: node.id,
          nodeType: node.type,
          configKeys: Object.keys(node.data),
        })
      }
    }
  }
}
```

**修改2：发送 node_configs**
```typescript
// 位置：第961-973行
response = await fetch(`/api/v1/workbench/workflows/create-instance`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`,
  },
  body: JSON.stringify({
    template_id: workflowTemplate.templateId,
    app_id: templateAppId,
    input_data: {},
    app_node_configs: appNodeConfigs,
    node_configs: Object.keys(nodeConfigs).length > 0 ? nodeConfigs : undefined,  // 新增
  }),
})
```

**修改3：修复 Response 读取错误（3处）**

**位置1：create-instance API 错误处理（第976-981行）**
```typescript
// 修改前：
if (!response.ok) {
  const errorData = await response.json().catch(async () => ({ detail: await response.text() }))
  throw new Error(errorData.detail || `API request failed: ${response.status}`)
}

// 修改后：
if (!response.ok) {
  // Clone response before reading to avoid "body stream already read" error
  const responseClone = response.clone()
  let errorData: any
  try {
    errorData = await response.json()
  } catch {
    // If JSON parsing fails, try text
    errorData = await responseClone.text()
    errorData = { detail: errorData }
  }
  throw new Error(errorData.detail || `API request failed: ${response.status}`)
}
```

**位置2：execute API 错误处理（第1027-1032行）**
```typescript
// 修改前：
if (!response.ok) {
  const errorData = await response.text()
  throw new Error(`API request failed: ${response.status} ${errorData}`)
}

// 修改后：
if (!response.ok) {
  // Clone response before reading to avoid "body stream already read" error
  const responseClone = response.clone()
  let errorData: string
  try {
    errorData = await response.text()
  } catch {
    errorData = await responseClone.text()
  }
  throw new Error(`API request failed: ${response.status} ${errorData}`)
}
```

**位置3：save-template API 错误处理（第1191-1196行）**
```typescript
// 修改前：
if (!response.ok) {
  const errorData = await response.json().catch(async () => ({ detail: await response.text() }))
  throw new Error(errorData.detail || `API request failed: ${response.status}`)
}

// 修改后：
if (!response.ok) {
  // Clone response before reading to avoid "body stream already read" error
  const responseClone = response.clone()
  let errorData: any
  try {
    errorData = await response.json()
  } catch {
    // If JSON parsing fails, try text
    errorData = await responseClone.text()
    errorData = { detail: errorData }
  }
  throw new Error(errorData.detail || `API request failed: ${response.status}`)
}
```

**验证方法**：

1. **配置覆盖验证**：
   - 从 template 创建 workflow instance
   - 修改 humanInTheLoop node 的 taskType 为 `request_for_update`
   - 设置 updateTitle 和 updateJobTitleIds
   - 运行 workflow
   - 检查日志：应显示 `taskType=request_for_update`，`has_updateConfig=True`
   - 检查消息中心：应显示 `REQUEST FOR UPDATE` 任务

2. **Response 读取验证**：
   - 运行 workflow
   - 检查浏览器控制台：不应出现 "body stream already read" 错误
   - 检查错误处理：API 错误应正确显示

**相关文件**：

- `backend/app/api/v1/workbench.py`（配置覆盖逻辑、异常处理、日志修复）
- `frontend/src/components/Workbench/FlowWorkspace.tsx`（配置收集、Response 处理修复）

**经验总结**：

1. **配置分层设计**：Template 提供结构，Instance 提供运行时覆盖，两者分离
2. **防御性编程**：Response 等资源只能消费一次，需要多次访问时先克隆
3. **深拷贝重要性**：配置覆盖时使用深拷贝，避免意外修改原始数据
4. **日志统一性**：统一日志格式便于问题排查和系统监控
5. **异常处理完整性**：关键操作（如数据库写入）应该有完整的异常捕获和回滚机制

---

*最后更新：2025年1月13日*