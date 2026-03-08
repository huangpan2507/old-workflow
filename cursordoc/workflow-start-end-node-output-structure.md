# Workflow Start Node and End Node Output Data Structure

## Overview

This document describes the output data structure for Start Node and End Node in workflow execution.

## Start Node Output Structure

**Location**: `backend/app/services/workflow/workflow_execution_service.py` - `_create_start_node()` method (line 604-621)

**Output Format**:
```python
{
  'status': 'completed',
  'output': input_data  # Workflow initial input data
}
```

**Details**:
- **`status`**: Always `'completed'` for start node
- **`output`**: Contains the workflow's initial input data (`state.get('input_data', {})`)
  - This typically includes:
    - File upload information (for file-based workflows)
    - URL (for URL-based workflows)
    - Configuration parameters from `app_node_configs`
    - Any other initial workflow input

**Storage Location**:
- Stored in `state['node_results'][start_node_id]`
- Persisted in `WorkflowExecutionTask.node_results` in database
- Available in `WorkflowInstance.execution_result` (via related task)

**Example**:
```json
{
  "status": "completed",
  "output": {
    "files": ["document.pdf"],
    "dagRunId": "manual__2026-01-12T01:52:53.330015+00:00",
    "inputMode": "file"
  }
}
```

## End Node Output Structure

**Location**: `backend/app/services/workflow/workflow_execution_service.py` - `_create_end_node()` method (line 1926-1935)

**Output Format**:
```python
{
  'status': 'completed',
  'output': {
    'workflow_completed': True
  }
}
```

**Details**:
- **`status`**: Always `'completed'` for end node
- **`output`**: Contains only a completion marker `{'workflow_completed': True}`
  - **No business data**: End node does NOT contain the final workflow result
  - **Completion marker only**: Used to indicate workflow execution completed

**Storage Location**:
- Stored in `state['node_results'][end_node_id]`
- Persisted in `WorkflowExecutionTask.node_results` in database
- Available in `WorkflowInstance.execution_result` (via related task)

**Important Note**: 
- The actual workflow result/output should be extracted from the **last business node** before the end node (e.g., App Node, Condition Node, Human in the Loop Node)
- End node is only a termination marker in the LangGraph workflow graph
- For user-facing results, use the output from the last executed business node (typically the node that connects to the end node)

## Workflow Final Output Storage

**Location**: `backend/app/api/v1/workbench.py` - `execute_workflow_task()` method (line 1412-1419)

**Storage Format**:
```python
WorkflowInstance.execution_result = {
  "final_output": final_result.get("final_data", {}),  # Note: Currently always {} as final_data is not set
  "summary": {
    "total_nodes": len(task.nodes),
    "completed_nodes": len(final_result.get("node_results", {})),
    "execution_time": str(datetime.now() - instance.create_time)
  }
}
```

**Current Implementation**:
- `final_data` is not actually extracted from the workflow execution
- The actual node results are stored in `WorkflowExecutionTask.node_results`
- To get the final workflow result, query the last business node's output from `node_results`

## Recommendation for Run History Feature

For displaying workflow execution results to end users:

1. **INPUT**: Extract from the first business node's `app_node_configs` (NOT from Start Node output)
   - **Important**: Start node has no upstream nodes, so its input should be empty
   - Find the first business node (start node's next node via edges)
   - Extract configuration from `app_node_configs[first_business_node_id]`
   - File names/URLs (from `dagRunId` or `url`)
   - Input mode (file/URL) from `inputMode`
   - Configuration parameters (requirement, etc.)

2. **OUTPUT**: Extract from the last business node in the actual execution path
   - **Important**: Use `execution_history` to determine the actual execution path (not just edges)
   - Find the last business node from `execution_history` (exclude start and end nodes)
   - Extract that node's output from `node_results[last_business_node_id]['output']`
   - Format according to node type (App Node: `conclusion_detail`, `structured_output`, etc.)
   - **Critical**: Must use actual execution path, not just edges, because there may be multiple branches (condition nodes)

3. **Do NOT use End Node output** for business results - it only contains `{'workflow_completed': True}`
