# Question Classifier Node 配置澄清文档

## 一、三个配置项的作用机制

### 1. Class Name（分类名称）

**作用**：定义一个分类分支的人类可读标识

**技术实现**：
- 存储在 `classes[].name` 字段
- 会被组装到 LLM prompt 中的 `Name: xxx` 行
- 用于日志、UI 显示、调试输出
- **不直接影响 LLM 的分类判断**，但作为上下文帮助 LLM 理解分类含义

**示例**：
```
Name: supplier admission approval
```

---

### 2. Description (helps LLM classify)

**作用**：**这是影响 LLM 分类决策的核心配置**

**技术实现**：
- 存储在 `classes[].description` 字段
- 会被组装到 LLM prompt 中的 `Description: xxx` 行
- **LLM 根据 Description 来判断上游输入应该匹配哪个分类**

**LLM 看到的 prompt 结构**：
```
Available Classes:
- ID: class_1
  Name: supplier admission approval
  Description: 假设你是一个供应商审核专家，帮我供应商文件里审核该供应商是否符合资质；

- ID: class_2
  Name: supplier information update
  Description: 如果该节点的输入内容里有显示缺失文件，请帮我展示缺失哪些文件内容；
```

**核心原则**：Description 应该描述**什么情况下选择这个分类**，而不是描述分类本身。

---

### 3. Additional Instruction (Optional)

**作用**：为 LLM 提供全局性的分类指导，影响整体分类策略

**技术实现**：
- 存储在 `config.instruction` 字段
- 会被注入到 prompt 中作为 `Additional guidance: xxx`
- 用于处理**边界情况**、**优先级规则**、**特殊业务逻辑**

**示例**：
```
多条分支里只能最终选择一条class
```

---

## 二、你的业务场景分析

### 场景描述

```
App Node 输出 (Markdown格式):
├── 供应商建议: 拒绝 / 有条件的批准 / 同意
└── 缺失文件/材料列表 (可能有，也可能没有)
```

### 分类需求

| 分支 | Task Type | 触发条件 |
|------|-----------|----------|
| Branch 1 | approve | "同意" 或 "有条件的批准" |
| Branch 2 | request for update | 存在缺失文件/材料 |

### 核心问题：如果同时满足两个条件怎么办？

**场景**：App Node 输出 = "有条件的批准" + "缺失材料：营业执照副本"

这是一个**设计决策问题**，有两种业务理解：

---

## 三、两种业务理解及配置方案

### 方案 A：结论优先（批准类型决定分支）

**业务理解**：
- "有条件的批准" 本身就意味着需要补充材料
- 既然说了"批准"，主要流程走 approve，让审核人决定是否需要补材料
- 材料补充是审核流程的一部分，不是独立分支

**推荐配置**：

```yaml
Class 1:
  Name: supplier admission approval
  Description: |
    当上游输入包含以下任一关键词时选择此分类：
    - "同意"
    - "通过"
    - "有条件的批准"
    - "approved"
    - "conditionally approved"
    即使同时提到缺失材料，只要结论是批准类，就选择此分类。

Class 2:
  Name: supplier information update  
  Description: |
    仅当上游输入明确表示"拒绝"且原因是材料不足时选择此分类。
    或者当结论明确要求补充材料后重新提交审核时选择此分类。
    关键词："拒绝"、"材料不足"、"需要补充后重新申请"

Additional Instruction:
  如果输入同时包含批准相关词汇和缺失材料信息，优先选择 class_1 (approval)。
  只有在明确拒绝或明确要求补材料重审时才选择 class_2。
```

---

### 方案 B：材料状态优先（缺失材料决定分支）

**业务理解**：
- 只要有缺失材料，就必须先补全材料
- "有条件的批准"实际上是"等材料齐全了才能批准"
- 必须先走 request for update 流程补材料

**推荐配置**：

```yaml
Class 1:
  Name: supplier admission approval
  Description: |
    当上游输入满足以下条件时选择此分类：
    1. 结论是"同意"或"通过"
    2. 且没有列出任何缺失文件或材料
    即：完全通过审核，无需补充任何材料。

Class 2:
  Name: supplier information update
  Description: |
    当上游输入满足以下任一条件时选择此分类：
    1. 结论是"有条件的批准"（意味着需要补材料）
    2. 明确列出了缺失的文件或材料
    3. 提到需要补充、更新、完善任何信息
    优先级：只要有缺失材料，就选择此分类，无论最终结论如何。

Additional Instruction:
  材料完整性优先于批准结论。
  如果提到任何缺失材料，必须选择 class_2 进行材料补充流程。
  只有完全通过且无需补材料时才选择 class_1。
```

---

## 四、关于"拒绝"的处理

从你的图中只看到两个分支，但 App Node 有三种结论：拒绝/有条件批准/同意。

**问题**：如果结论是"拒绝"，走哪个分支？

### 建议增加第三个分支

```yaml
Class 3:
  Name: supplier rejection
  Description: |
    当上游输入明确表示拒绝该供应商准入时选择此分类：
    - "拒绝"
    - "不通过"
    - "rejected"
    - "不符合准入条件"
```

如果不想加第三个分支，需要在 Additional Instruction 中明确：
```
如果结论是"拒绝"，走 class_2 (request for update) 分支，让被通知人知道需要重新准备材料后再次申请。
```

---

## 五、最终推荐配置（基于你的截图场景）

基于你截图中的配置，我建议修改为：

```yaml
Class 1 (class_1):
  Name: supplier admission approval
  Description: |
    选择此分类的条件：
    上游 App Node 的审核结论是"同意该供应商准入"或"有条件的批准"。
    关键判断：看结论是否为正向（批准类），而非看是否有缺失材料。

Class 2 (class_2):
  Name: supplier information update
  Description: |
    选择此分类的条件：
    1. 上游 App Node 的审核结论是"拒绝"
    2. 或者明确要求必须先补全材料才能继续审核流程
    关键判断：当前状态无法进入批准流程，需要先补材料。

Additional Instruction:
  分类决策规则（按优先级）：
  1. 如果结论包含"同意"、"批准"、"通过"等词汇 → class_1
  2. 如果结论包含"有条件的批准" → class_1（审核人可在approve流程中要求补材料）
  3. 如果结论是"拒绝"或明确要求补材料后重审 → class_2
  4. 如果无法判断 → 默认选择 class_1
```

---

## 六、技术实现原理

### LLM 分类 Prompt 结构

```
You are a classifier. Your task is to analyze the input and select EXACTLY ONE class that best matches.

Available Classes:
- ID: class_1
  Name: supplier admission approval
  Description: [你配置的 Description]

- ID: class_2
  Name: supplier information update
  Description: [你配置的 Description]

Additional guidance: [你配置的 Additional Instruction]

IMPORTANT RULES:
1. You MUST select exactly ONE class - no more, no less
2. Return ONLY the class_id of the best matching class
3. If multiple classes could apply, choose the MOST relevant one
4. If no class seems to match well, choose the closest match

Please classify the following input:
[App Node 的 Markdown 输出内容]

Select the best matching class and provide your response in JSON format.
```

### 输出结构

```json
{
  "class_id": "class_1",
  "reason": "The input mentions '有条件的批准', which indicates approval category"
}
```

---

## 七、关键设计洞察

> **Question Classifier Node 的本质是"单选路由器"，不是"多选广播器"。**

这意味着：

1. **必须做出唯一选择**：即使输入满足多个条件，LLM 也必须选择一个分支
2. **Description 定义选择规则**：不是描述分类是什么，而是描述**什么时候选这个分类**
3. **Additional Instruction 处理边界**：当多个条件冲突时，明确优先级

---

## 八、调试建议

如果分类结果不符合预期：

1. **查看 Classifier Node 的 Output Tab**
   - `classification_reason` 字段显示 LLM 为什么选择这个分类
   
2. **检查上游 App Node 的 `conclusion_detail`**
   - 这是 Classifier 实际读取的输入文本
   - 确认关键词是否在输出中

3. **优化 Description**
   - 使用明确的关键词匹配规则
   - 避免模糊的描述
