# Workflow XCom 数据格式事实分析报告

## 基于代码事实的准确结论

### 事实1: Airflow XCom API 返回结构

**代码位置**: `backend/app/services/airflow_service.py:464`
```python
xcom_value = response.json()
```

**事实**: Airflow XCom API 返回的是 JSON 格式，后端使用 `response.json()` 解析，得到 Python 对象（通常是 dict）。

### 事实2: XCom API 返回的数据结构

**代码位置**: `backend/app/api/v1/report.py:1364-1365`
```python
if isinstance(xcom_value, dict) and 'value' in xcom_value:
    value_field = xcom_value['value']
```

**事实**: Airflow XCom API 返回的数据结构包含 `'value'` 字段，实际数据存储在 `xcom_value['value']` 中。

**结构**:
```python
{
    "value": <actual_data>  # 实际数据在这里
}
```

### 事实3: value 字段的数据类型

**代码位置**: `backend/app/api/v1/report.py:1373-1402`
```python
# 如果 value 是字典，直接使用
if isinstance(value_field, dict):
    actual_data = value_field
    
# 如果 value 字段是字符串，尝试解析
elif isinstance(value_field, str):
    try:
        # 先尝试 JSON 解析
        actual_data = json.loads(value_field)
    except json.JSONDecodeError as je:
        # JSON 解析失败，使用 eval 解析 Python 字典格式
        actual_data = eval(cleaned_value)
```

**事实**: `value_field` 可能是两种类型：
1. **字典类型** (`dict`)：直接使用
2. **字符串类型** (`str`)：需要解析，可能是：
   - 标准 JSON 字符串（`json.loads()` 成功）
   - Python 字典字符串（`json.loads()` 失败，`eval()` 成功）

### 事实4: 日志证据

**用户提供的日志**:
```
WARNING:app.api.v1.report:[WF][XCom] JSON parsing failed: Expecting property name enclosed in double quotes: line 1 column 2 (char 1), attempting eval for Python dict format...
```

**事实**: 
- `value_field` 是字符串类型
- `json.loads(value_field)` 失败，错误信息表明不是标准 JSON 格式
- 代码随后使用 `eval()` 成功解析

### 事实5: Airflow DAG 返回值

**代码位置**: `airflow/dags/agent_file_ingest_unified.py:1534`
```python
return base_agents_result  # 直接返回 Python 字典
```

**事实**: DAG 函数直接返回 Python 字典对象，Airflow 会自动将其推送到 XCom。

---

## 准确结论

### 数据流路径

1. **Airflow DAG** (`stage_1_base_agents`)
   - 返回: Python 字典对象 `base_agents_result`
   - 格式: `{"success": True, "markdown_conclusion": "...", ...}`

2. **Airflow XCom 存储**
   - Airflow 接收 Python 字典返回值
   - Airflow 将数据序列化存储到 XCom
   - **关键**: Airflow 在序列化时，将 Python 字典转换为字符串存储在 `value` 字段中

3. **Airflow XCom API 返回**
   - API 返回 JSON 格式: `{"value": "<stringified_dict>"}`
   - `value` 字段是字符串，内容是 Python 字典的字符串表示

4. **后端解析** (`report.py`)
   - `xcom_value = response.json()` → 得到 `{"value": "<string>"}`
   - `value_field = xcom_value['value']` → 得到字符串
   - `json.loads(value_field)` → **失败**（因为不是标准 JSON）
   - `eval(value_field)` → **成功**（因为它是 Python 字典字符串）

### 问题根源

**准确说法**:
- Airflow 在存储 XCom 时，将 Python 字典序列化为字符串存储在 `value` 字段中
- 这个字符串是 Python 字典的字符串表示（如 `"{'success': True, 'markdown_conclusion': '...'}"`），而不是标准 JSON 格式
- 标准 JSON 要求使用双引号，而 Python 字典字符串使用单引号
- 标准 JSON 要求布尔值为 `true`/`false`，而 Python 字典字符串使用 `True`/`False`

**不是猜测，是基于以下事实**:
1. 代码逻辑显示 `value_field` 是字符串类型（第1379行）
2. `json.loads()` 失败，错误信息表明不是标准 JSON（日志证据）
3. `eval()` 成功解析（代码逻辑）
4. DAG 函数返回 Python 字典（代码事实）

---

## 需要进一步确认的事实

为了给出更准确的结论，需要确认：

1. **Airflow 版本**: 不同版本的 Airflow 可能使用不同的序列化方式
2. **实际 XCom 数据**: 查看实际存储的 XCom 数据格式（可以通过日志或数据库查询）
3. **Airflow 配置**: 是否有配置影响 XCom 序列化方式

---

## 建议的验证方法

1. **查看日志中的实际数据**:
   - 查看 `[WF][XCom] Value field is string, length=...` 日志
   - 查看 `First 200 chars: {value_field[:200]}` 日志

2. **直接查询 Airflow XCom**:
   - 通过 Airflow UI 查看 XCom 数据
   - 或通过 API 直接获取原始数据

3. **检查 Airflow 版本和配置**:
   - 确认 Airflow 版本
   - 检查是否有自定义的 XCom 序列化配置

---

## 修改建议（待确认事实后）

在确认实际数据格式后，可以考虑：

1. **如果确认是 Python 字典字符串**:
   - 在 DAG 中返回前使用 `json.dumps()` 转换为标准 JSON 字符串
   - 或改进后端解析逻辑，优先使用 `ast.literal_eval()`（更安全）

2. **如果确认是其他格式**:
   - 根据实际格式调整解析逻辑
