# 「单节点运行」澄清说明

> 在修改 SingleNodeRunModal / 后端 single-run 前，先澄清：单节点运行是什么、用在什么场景、实现什么功能，以及当前使用情况。

---

## 一、是什么

**单节点运行（Single Node Run）** = 在 Workbench 画布上**只执行某一个节点**，不跑完整 Workflow。

- **前端**：`SingleNodeRunModal` 弹窗 + 「Run Node」按钮。
- **后端**：`POST /workbench/nodes/single-run`，注释写的是 *"Execute a single node **independently for testing purposes**"*（独立执行单个节点，用于测试）。

---

## 二、实际使用流程（设计意图）

1. 用户在画布上**选中一个节点**（Start / End / App / Question Classifier / Human In The Loop 等）。
2. 通过某种入口**打开 Single Node Run 弹窗**（例如节点菜单里的「Run」、或配置面板里的「Test Run」等）。
3. 在弹窗内：
   - **非 App 节点**：选择输入来源 → **Manual**（手填 JSON）/ **Previous Node**（选上游节点，用其输出）/ **Workflow Input**（ workflow 级输入）→ 点击 **Run Node**。
   - **App 节点**：
     - **File 模式**：先在节点配置里选文件；若未上传则弹窗里先 `uploadMultipleFiles`（旧设计，上传即触发 DAG）拿到 `dagRunId`，再 Run Node，把 `dagRunId` 随 `app_node_config` 发给 single-run API。
     - **URL 模式**：在节点配置里填 URL，Run Node 时把 `url` 随 `app_node_config` 发给 API。
4. 前端调用 **`POST /workbench/nodes/single-run`**，传 `node`、`app_node_config`（含 `dagRunId` 或 `url`）、以及可选的 `input_data` / `previous_node_outputs` / `workflow_input_data`。
5. 后端用同样的节点逻辑（如 `_create_app_node`）跑这一节点，返回 `output`。
6. 弹窗展示 **Output**（含 `status`、`output` 等），父组件可把结果写回节点（如 `nodeResults`），用于**单独看这一个节点的执行结果**。

---

## 三、作用与业务场景

| 维度 | 说明 |
|------|------|
| **主要作用** | **单独测试 / 调试某一个节点**，不跑整条 Workflow。 |
| **典型场景** | 1. 配置好 App 节点（文件或 URL）后，先「跑一下这个节点」看 DAG 是否正常、输出是否符合预期，再跑全 Workflow。<br/>2. 调试 Question Classifier、HITL 等节点时，用手动或上游模拟输入，快速验证逻辑。<br/>3. 用 Previous Node / Workflow Input 模拟上游，做**局部链路验证**。 |
| **与 Run Workflow 的区别** | **Run Workflow**：按 DAG 从 Start 跑到 End，所有节点依次执行。<br/>**Single Node Run**：只跑你选中的那个节点，其余节点不执行；输入完全由弹窗里选的来源（Manual / Previous / Workflow）或 App 的 file/URL 配置决定。 |

所以，单节点运行实现的是 **「单节点级」的测试 / 调试能力**，面向的是 **「我想先验证这一个节点再跑全链路」** 的场景。

---

## 四、当前实现状态（基于代码检索）

| 部分 | 状态 |
|------|------|
| **后端** | `POST /workbench/nodes/single-run` 已实现，能跑单节点；App 的 file 模式用 `app_node_config.dagRunId`（旧设计）。 |
| **前端 SingleNodeRunModal** | 已实现：输入来源选择、App file 模式下的 `uploadMultipleFiles` + `dagRunId`、调用 single-run API、展示 Output。 |
| **弹窗入口** | **未找到**。FlowWorkspace 里有 `singleNodeRunModal` 状态、`handleSingleNodeRunResult`、以及弹窗的渲染逻辑，但 **没有任何地方 `setSingleNodeRunModal({ isOpen: true, nodeId })`**。ApplicationNode 的菜单只有 Configure / View Output / Delete，**没有 Run / Run Node**；NodeConfigPanel、AppNodeConfigPanel 也未见打开该弹窗的按钮。 |

因此，**单节点运行在代码里是「已实现但未接入 UI」**：API 和弹窗都在，但**没有可点击的入口**打开这个弹窗，正常用户目前用不到单节点运行功能。

---

## 五、数据流概览（单节点运行 + App Node File 模式，当前旧设计）

```
用户选择节点 → [入口未找到] 打开 Single Node Run 弹窗
  → App File 模式：选文件 → uploadMultipleFiles（上传即触发 DAG）→ 拿到 dagRunId
  → 点击 Run Node
  → POST /workbench/nodes/single-run
      body: { node, app_node_config: { inputMode, dagRunId, ... } }
  → 后端 _create_app_node 走 file + dagRunId 分支（不触发 DAG，用已有 run）
  → 当前实现中该分支也没有轮询，直接 mark completed，结论为空
  → 返回 output → 弹窗展示，父组件更新 nodeResults
```

---

## 六、和「新设计」的关系

- 你已确认：Workflow **统一改为「仅提取 + 运行再触发」**，**删除 file + dagRunId 回溯兼容**。
- 若 **单节点运行** 也要对齐新设计，则 App File 模式应改为：
  - 用 **`uploadFilesExtractOnly`**，拿 **`fileIngestionRecordIds`**；
  - `app_node_config` 只传 **`fileIngestionRecordIds`**（不传 `dagRunId`）；
  - 后端 single-run 里 App 节点 **只** 走 **file + fileIngestionRecordIds**（触发 DAG → 轮询 → 再构建 output），**不再** 支持 `dagRunId`。

由于**单节点运行目前没有 UI 入口**，有两种可能选择：

1. **仍然按约定改掉**：把 SingleNodeRunModal + 后端 single-run 的 App 逻辑改成 extract-only + fileIngestionRecordIds，去掉 dagRunId，为**以后**接入「Run Node」入口做好准备。
2. **暂不改**：维持现状，等真正加「Run Node」入口时，再一起改成新设计；**本次只改 Workflow 运行 + 配置相关**。

---

## 七、需要你拍板的两点

1. **单节点运行**的实际使用情况你那边是否更清楚？例如是否有内部入口、或计划中的「Run Node」按钮？若目前**完全没人用**，是否接受「先改逻辑、后补入口」？
2. **本次改造范围**是否包含单节点运行？
   - **包含**：SingleNodeRunModal（extract-only + fileIngestionRecordIds）+ 后端 single-run 去掉 dagRunId，与 Workflow 一致。
   - **不包含**：本次只做 Workflow 配置/运行 + FlowWorkspace 等已确认部分；单节点运行以后再说。

你确认后，再按你的选择改 SingleNodeRunModal 与 single-run 实现（或明确暂不修改）。

---

*文档位置：`cursordocs/workflow-single-node-run-clarification.md`*
