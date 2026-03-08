# 任务计划：Human in the Loop（Assign App Task）结果传给下游节点

## 目标
在**不引入 merged_context** 的前提下，让 Workbench 中「Human in the Loop、Task Type = Assign App Task」节点在问卷/报告完成后，将产生的结果（含 questionnaire 填写数据）传递给下游节点。

## v4.0 结构约定（澄清）
- 各节点统一输出结构：`status` + `output`（`current_node`、`upstream_nodes`、`structured_output`、`_metadata`）。
- **current_node** 与 **structured_output** 均为**当前节点自己的**输出（后者供 UI 友好展示）；**output.upstream_nodes** 存在且承载该节点视角下的祖先链，**不作为下游获取『所有祖先』的统一接口，仅用于结构/审计等**；下游需要「所有祖先」时统一通过 **`_collect_upstream_summaries(state, 当前节点 id)`** 获取。
- **v4.0 不再使用顶层 merged_context**。各节点（App、Question Classifier、Human 五种 task type）的完整 output 结构见：`cursordocs/workflow-v4-output-structure-clarification.md`。

## 阶段

### Phase 1：需求与根因确认 [已完成]
- [x] 梳理 Assign App Task 完成后的 output 结构（v4.0，无顶层 merged_context）
- [x] 确认下游当前取数方式（get_context_from_upstream_output 依赖 merged_context，导致 v4.0 上游返回空）
- [x] 确认不引入 merged_context，改为基于 v4.0 的 current_node.output + structured_output 传下游

### Phase 2：方案设计 [已完成]
- [x] 撰写 v4.0 结构澄清文档（workflow-v4-output-structure-clarification.md）
- [x] 撰写「无 merged_context」的传下游方案（assign-app-task-result-to-downstream.md）
- [ ] 与用户确认方案后再进入实现

### Phase 3：实现（待确认后执行）
- [x] 扩展 get_context_from_upstream_output：当 upstream 为 v4.0 时，从该节点的 current_node.output + structured_output 构建 context（单节点上游），不再依赖顶层 merged_context；对 assign_app_task 的 completion_results 直接传递完整内容（目前不做截断或摘要，未来可根据需求再考虑体积控制）
- [x] 扩展 _extract_node_summary（运行时摘要）：对 task_type=assign_app_task 的节点，从 completion_results 生成额外的说明性 summary 文本，仅写入运行时 _collect_upstream_summaries 的结果列表，**不修改、不截断原始 completion_results 本身**
- [ ] 单元/集成测试与回归

### Phase 4：文档与收尾
- [x] 若有需要，更新 workflow-node-output-structure-new.md 中 assign_app_task 的 output 约定
- [ ] 更新 progress.md

## 决策记录
| 决策点 | 结论 |
|--------|------|
| 是否再引入 merged_context | **否**。v4.0 已明确各节点结果分块存储，不通过 merged_context 合并 |
| 下游如何取 assign_app_task 结果 | 单节点：从 v4.0 该节点的 current_node.output.completion_results；所有祖先：统一通过 _collect_upstream_summaries(state, 当前节点 id)。output.upstream_nodes 不作为下游获取『所有祖先』的统一接口，仅用于结构/审计等 |
| 是否破坏现有 approval/assign/request_for_update | 否，仅扩展 v4.0 分支与 assign_app_task 摘要逻辑 |

## 参考
- cursordocs/workflow-v4-output-structure-clarification.md
- cursordocs/assign-app-task-result-to-downstream.md
- cursordocs/workflow-upstream-context-design.md
- backend: human_in_the_loop_node.py, node_utils.py, workflow_execution_service.py
