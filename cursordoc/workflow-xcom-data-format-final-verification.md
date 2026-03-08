# Workflow XCom 数据格式最终验证报告

## ✅ 已验证的事实（基于实际 API 响应）

### 事实1: Airflow API 实际返回的数据格式

**验证方法**: 直接调用 Airflow REST API，查看原始响应

**API 响应**:
```json
{
  "dag_id": "agent_file_ingest_unified",
  "task_id": "stage_1_base_agents",
  "key": "return_value",
  "value": "{'success': True, 'markdown_conclusion': 'Executive summary\\n- Source: Consolidated results from thr..."
}
```

**value 字段内容** (前100字符):
```
"{'success': True, 'markdown_conclusion': 'Executive summary\\n- Source: Consolidated results from thr"
```

**格式特征**:
- ✅ 使用单引号 (`'`) 包围字符串
- ✅ 使用 Python 布尔值 (`True`/`False`)
- ✅ 是 Python 字典的字符串表示（`repr(dict)` 或 `str(dict)` 的结果）
- ❌ 不是标准 JSON 格式（JSON 要求双引号和 `true`/`false`）

### 事实2: 数据库存储格式 vs API 返回格式

**数据库存储** (从 PostgreSQL 查询):
- 存储格式: JSON (使用双引号，`true`/`false`)
- 示例: `{"success": true, "markdown_conclusion": "..."}`

**API 返回** (从 REST API 查询):
- 返回格式: Python 字典字符串 (使用单引号，`True`/`False`)
- 示例: `"{'success': True, 'markdown_conclusion': '...'}"`

**结论**: 
- Airflow 在存储到数据库时，将 Python 对象序列化为 JSON
- Airflow API 在返回时，将存储的值反序列化后再使用 `repr()` 或 `str()` 序列化为字符串
- 导致 API 返回的是 Python 字典字符串，而不是标准 JSON

### 事实3: 后端解析逻辑

**当前代码** (`backend/app/api/v1/report.py:1381-1396`):
```python
# 先尝试 JSON 解析
try:
    actual_data = json.loads(value_field)  # ❌ 失败
except json.JSONDecodeError:
    # 使用 eval 解析 Python 字典格式
    actual_data = eval(cleaned_value)  # ✅ 成功
```

**验证结果**:
- `json.loads(value_field)` 失败: `Expecting property name enclosed in double quotes: line 1 column 2 (char 1)`
- `eval(value_field)` 成功: 能够正确解析 Python 字典字符串

---

## 准确结论

### 问题根源

**Airflow XCom API 返回的数据格式**:
- `value` 字段是 **Python 字典的字符串表示**
- 格式: `"{'key': 'value', 'bool': True}"` (单引号，Python 布尔值)
- 不是标准 JSON: `{"key": "value", "bool": true}` (双引号，JSON 布尔值)

**原因**:
- Airflow API 在序列化响应时，对 `value` 字段使用了 Python 的 `repr()` 或 `str()` 方法
- 而不是使用 `json.dumps()` 进行标准 JSON 序列化

**影响**:
- 后端必须使用 `eval()` 或 `ast.literal_eval()` 解析，而不能直接使用 `json.loads()`
- 使用 `eval()` 存在安全风险（虽然在这个场景下风险较低）

---

## 修改方案

### 方案1: 改进后端解析逻辑（推荐，立即实施）

**位置**: `backend/app/api/v1/report.py:1390-1396`

**修改**: 使用 `ast.literal_eval()` 替代 `eval()`（更安全）

```python
# 修改前
actual_data = eval(cleaned_value)

# 修改后
import ast
try:
    actual_data = ast.literal_eval(value_field)
except (ValueError, SyntaxError):
    # 如果 ast.literal_eval 失败，再尝试 eval（处理 NaN 等特殊情况）
    actual_data = eval(cleaned_value)
```

**优点**:
- 更安全（`ast.literal_eval()` 只能解析字面量，不能执行代码）
- 不需要修改 Airflow DAG
- 立即生效

### 方案2: 规范化 DAG 返回值（长期优化）

**位置**: `airflow/dags/agent_file_ingest_unified.py:stage_1_base_agents`

**修改**: 在返回前转换为 JSON 字符串

```python
# 修改前
return base_agents_result

# 修改后
import json
return json.dumps(base_agents_result, ensure_ascii=False)
```

**注意**: 
- 如果使用此方案，后端解析逻辑也需要相应调整（直接使用 `json.loads()`）
- 需要确保所有调用此 DAG 的地方都能处理 JSON 字符串返回值

---

## 建议的实施顺序

1. **立即**: 改进后端解析逻辑（使用 `ast.literal_eval()`）
2. **短期**: 验证改进后的解析逻辑是否正常工作
3. **长期**: 考虑规范化 DAG 返回值（如果 Airflow 版本升级或配置变更）

---

## 相关文件

- `backend/app/api/v1/report.py` - XCom 数据解析逻辑
- `airflow/dags/agent_file_ingest_unified.py` - DAG 返回值定义
- `backend/app/services/airflow_service.py` - Airflow API 调用
