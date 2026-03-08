# Workflow XCom 数据格式验证分析报告（基于事实）

## 已验证的事实

### 事实1: Airflow 数据库中存储的格式

**验证方法**: 直接查询 Airflow PostgreSQL 数据库

**查询结果**:
```
value_type: bytea (PostgreSQL 二进制类型)
value_length: 22894 bytes
value_preview (hex): \x7b2273756363657373223a20747275652c20226d61726b646f776e5f636f6e636c7573696f6e223a20224578656375746976652073756d6d6172795c6e2d20536f757263653a20436f6e736f6c69646174656420726573756c74732066726f6d207468
```

**解码结果**:
```json
{"success": true, "markdown_conclusion": "Executive summary\n- Source: Consolidated results from th
```

**结论**: 
- Airflow 数据库中存储的是 **标准 JSON 格式**
- 使用双引号 (`"`)
- 使用 JSON 布尔值 (`true`/`false`)

### 事实2: 后端接收到的数据格式

**代码位置**: `backend/app/api/v1/report.py:1364-1402`

**日志证据**:
```
[WF][XCom] XCom value has 'value' field: type=str, is_dict=False, is_str=True
[WF][XCom] Value field is string, length=11407, attempting parse...
[WF][XCom] JSON parsing failed: Expecting property name enclosed in double quotes: line 1 column 2 (char 1)
[WF][XCom] Successfully parsed value with eval after NaN replacement
```

**结论**:
- `value_field` 是字符串类型
- `json.loads(value_field)` 失败
- `eval(value_field)` 成功

### 事实3: AirflowService.get_xcom_value() 返回结构

**代码位置**: `backend/app/services/airflow_service.py:464`
```python
xcom_value = response.json()  # Airflow API 返回 JSON
```

**代码位置**: `backend/app/api/v1/report.py:1364`
```python
if isinstance(xcom_value, dict) and 'value' in xcom_value:
    value_field = xcom_value['value']
```

**结论**:
- Airflow API 返回的 JSON 包含 `'value'` 字段
- `value_field = xcom_value['value']` 是字符串类型

---

## 问题根源分析

### 矛盾点

1. **数据库存储**: JSON 格式（标准 JSON，双引号，true/false）
2. **API 返回**: `value` 字段是字符串，且 `json.loads()` 失败
3. **解析成功**: `eval()` 能够成功解析

### 可能的原因

**假设1: Airflow API 双重序列化**
- Airflow 数据库存储 JSON 字符串
- Airflow API 返回时，将 JSON 字符串再次序列化为 JSON 字符串
- 导致 `value` 字段是 JSON 字符串的 JSON 字符串（双重编码）

**假设2: Airflow 使用 Python repr() 序列化**
- Airflow 在序列化时使用 `repr(dict)` 而不是 `json.dumps()`
- 导致返回的是 Python 字典的字符串表示

**假设3: Airflow API 返回格式不一致**
- 不同版本的 Airflow 或不同配置下，API 返回格式可能不同
- 某些情况下返回反序列化的对象，某些情况下返回序列化的字符串

---

## 需要进一步验证

为了给出准确的修改方案，需要确认：

1. **Airflow API 实际返回的原始数据**
   - 查看 `response.text` 的原始内容
   - 确认 `value` 字段在 JSON 响应中的实际格式

2. **Airflow 版本和配置**
   - 确认 Airflow 版本
   - 检查是否有自定义的 XCom 序列化配置

3. **实际 value_field 内容**
   - 等待下一次 workflow 执行，查看我添加的临时日志
   - 或直接查询 Airflow API 获取实际响应

---

## 临时解决方案（基于当前事实）

### 当前代码逻辑（已实现）

```python
# 1. 先尝试 JSON 解析
try:
    actual_data = json.loads(value_field)
except json.JSONDecodeError:
    # 2. JSON 解析失败，使用 eval 解析 Python 字典格式
    actual_data = eval(cleaned_value)
```

**问题**: 使用 `eval()` 存在安全风险

### 改进方案（待验证后实施）

**方案A: 使用 ast.literal_eval()（更安全）**
```python
import ast
try:
    actual_data = json.loads(value_field)
except json.JSONDecodeError:
    try:
        actual_data = ast.literal_eval(value_field)
    except (ValueError, SyntaxError):
        # Fallback to eval only if necessary
        actual_data = eval(cleaned_value)
```

**方案B: 规范化 DAG 返回值（根本解决）**
```python
# 在 DAG 中返回前转换为 JSON 字符串
import json
return json.dumps(base_agents_result, ensure_ascii=False)
```

---

## 下一步行动

1. **等待下一次 workflow 执行**
   - 查看我添加的临时日志输出
   - 确认 `value_field` 的实际内容

2. **或直接查询 Airflow API**
   - 获取实际的 API 响应
   - 分析 `value` 字段的格式

3. **根据验证结果决定修改方案**
   - 如果确认是双重编码：修改后端解析逻辑
   - 如果确认是 Python dict 字符串：改进解析逻辑或规范化 DAG 返回值
