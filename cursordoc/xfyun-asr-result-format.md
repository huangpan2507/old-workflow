# iFlytek ASR Result Format Documentation

## Overview

科大讯飞语音转写API返回的`orderResult`字段包含转写结果，格式为JSON字符串，需要解析后才能使用。

## Data Structure

### 1. orderResult 结构

`orderResult`是一个JSON字符串，包含以下结构：

```json
{
  "lattice": [
    {
      "json_1best": "{...}",  // JSON字符串，需要再次解析
      "lid": "0",              // Lattice ID
      "begin": "0",            // 开始时间（毫秒）
      "end": "4570",           // 结束时间（毫秒）
      "spk": "段落-0"          // 说话人标识
    }
  ],
  "lattice2": [...]            // 二级网格（可选）
}
```

### 2. lattice（网格）说明

**lattice**是语音识别中的一种数据结构，类似于一个"网格"或"格子"，包含：

- **多个候选识别结果**：每个时间段可能有多个可能的识别结果
- **时间戳信息**：每个词都有精确的开始和结束时间
- **词级别详细信息**：包括词性、置信度等

在科大讯飞的API中，`lattice`数组中的每个元素代表一个时间段（segment）的识别结果。

### 3. json_1best 结构

`json_1best`是一个JSON字符串，解析后的结构如下：

```json
{
  "st": {
    "sc": "0.00",        // 句子置信度（sentence confidence）
    "pa": "0",           // 段落编号（paragraph）
    "rt": [              // 识别结果（recognition result）
      {
        "ws": [          // 词序列（word sequence）
          {
            "cw": [      // 候选词（candidate word）
              {
                "w": "中",           // 词（word）
                "wp": "n",           // 词性（word property）
                "wc": "0.0000",      // 词置信度（word confidence）
                "og": "..."          // 原始形式（original，可选）
              }
            ],
            "wb": 1,     // 词开始时间（word begin），单位：毫秒
            "we": 24     // 词结束时间（word end），单位：毫秒
          }
        ]
      }
    ]
  }
}
```

## Field Descriptions

### Top Level Fields

| Field | Type | Description |
|-------|------|-------------|
| `lattice` | Array | 网格数组，包含多个时间段的识别结果 |
| `lattice2` | Array | 二级网格（可选），包含更详细的信息 |

### Lattice Item Fields

| Field | Type | Description |
|-------|------|-------------|
| `json_1best` | String | 最佳识别结果的JSON字符串 |
| `lid` | String | Lattice ID，标识不同的网格 |
| `begin` | String | 开始时间（毫秒） |
| `end` | String | 结束时间（毫秒） |
| `spk` | String | 说话人标识（speaker） |

### Sentence (st) Fields

| Field | Type | Description |
|-------|------|-------------|
| `sc` | String | 句子置信度（sentence confidence），范围通常为0.00-1.00 |
| `pa` | String | 段落编号（paragraph），标识不同的段落 |
| `rt` | Array | 识别结果数组（recognition result） |

### Word Sequence (ws) Fields

| Field | Type | Description |
|-------|------|-------------|
| `cw` | Array | 候选词数组（candidate word） |
| `wb` | Number | 词开始时间（word begin），单位：毫秒 |
| `we` | Number | 词结束时间（word end），单位：毫秒 |

### Candidate Word (cw) Fields

| Field | Type | Description |
|-------|------|-------------|
| `w` | String | 识别出的词（word） |
| `wp` | String | 词性（word property），常见值：<br>- `n`: 名词<br>- `v`: 动词<br>- `p`: 标点符号<br>- `g`: 其他 |
| `wc` | String | 词置信度（word confidence），范围通常为0.0000-1.0000 |
| `og` | String | 原始形式（original，可选），如数字的汉字形式（如"7月"的`og`为"七月"） |

## Example Usage

### Parsing orderResult

```python
import json

# orderResult is a JSON string
order_result_str = response['content']['orderResult']

# Parse first level
order_result = json.loads(order_result_str)

# Extract text from lattice
for lattice_item in order_result.get('lattice', []):
    # Parse json_1best
    json_1best_str = lattice_item.get('json_1best', '')
    json_1best = json.loads(json_1best_str)
    
    # Extract words
    words = []
    for rt in json_1best.get('st', {}).get('rt', []):
        for ws in rt.get('ws', []):
            for cw in ws.get('cw', []):
                words.append(cw.get('w', ''))
    
    text = ''.join(words)
    print(text)
```

### Extracting Plain Text

最简单的方式是提取所有词（`w`字段）并连接：

```python
def extract_text_from_order_result(order_result_str):
    """Extract plain text from orderResult JSON string."""
    order_result = json.loads(order_result_str)
    words = []
    
    for lattice_item in order_result.get('lattice', []):
        json_1best_str = lattice_item.get('json_1best', '')
        json_1best = json.loads(json_1best_str)
        
        for rt in json_1best.get('st', {}).get('rt', []):
            for ws in rt.get('ws', []):
                for cw in ws.get('cw', []):
                    words.append(cw.get('w', ''))
    
    return ''.join(words)
```

### Extracting with Timestamps

如果需要时间戳信息：

```python
def extract_text_with_timestamps(order_result_str):
    """Extract text with word-level timestamps."""
    order_result = json.loads(order_result_str)
    result = []
    
    for lattice_item in order_result.get('lattice', []):
        json_1best_str = lattice_item.get('json_1best', '')
        json_1best = json.loads(json_1best_str)
        
        for rt in json_1best.get('st', {}).get('rt', []):
            for ws in rt.get('ws', []):
                word_info = {
                    'text': '',
                    'begin': ws.get('wb', 0),
                    'end': ws.get('we', 0)
                }
                
                for cw in ws.get('cw', []):
                    word_info['text'] += cw.get('w', '')
                
                result.append(word_info)
    
    return result
```

## Notes

1. **双重JSON编码**：`orderResult`本身是JSON字符串，其中的`json_1best`也是JSON字符串，需要两次解析。

2. **时间单位**：所有时间字段（`begin`, `end`, `wb`, `we`）的单位都是**毫秒**。

3. **置信度**：置信度字段（`sc`, `wc`）通常是字符串格式，值为"0.0000"到"1.0000"之间。

4. **词性标注**：`wp`字段提供词性信息，可用于文本分析。

5. **原始形式**：`og`字段（如果存在）提供数字或特殊词的原始形式，如"7月"的`og`为"七月"。

## References

- iFlytek API Documentation: https://www.xfyun.cn/doc/asr/ifasr_new/API.html
- Lattice in Speech Recognition: https://en.wikipedia.org/wiki/Lattice_(discrete_mathematics)#Speech_recognition

