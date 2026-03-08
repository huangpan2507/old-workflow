# App Node Structured Output Feature

## Overview

The App Node now supports structured output configuration, integrating the functionality previously provided by the Data Transform Node. This feature allows users to configure JSON Schema for structured data extraction from App Node text output using LLM.

## Features

### 1. Structured Output Configuration

- **Visual Editor**: Non-technical users can configure JSON Schema through a user-friendly visual interface
- **Quick Templates**: Pre-configured templates for common use cases (Supplier Check, Approval Workflow, Data Analysis)
- **Field Types Supported**: string, number, boolean, array, object
- **Required Fields**: Mark fields as required or optional
- **Extraction Guide**: Add detailed descriptions to guide LLM extraction (replaces Data Transform Node's extractionPrompt)
- **Field Type Help**: Contextual help and examples for each field type

### 2. Single Node Run

- **Independent Testing**: Run individual nodes without executing the entire workflow
- **Input Data Priority**:
  1. Previous node outputs (if available)
  2. Manual input (JSON format)
  3. Workflow input data
- **Result Preview**: View actual output data to configure subsequent nodes

### 3. Placeholder Variables

- **Syntax**: `{{node_id.field_name}}`
- **Usage**: Reference output from previous nodes in configuration
- **Nested Fields**: Support for nested fields like `{{node_id.structured_output.field_name}}`

## Migration from Data Transform Node

### Deprecation Notice

**Data Transform Node is now deprecated**. New workflows should use App Node with structured_output configuration instead.

### Migration Steps

1. **Remove Data Transform Node** from your workflow
2. **Configure App Node** with structured_output:
   - Enable "STRUCTURED" toggle in App Node configuration
   - Click "Configure" to open Visual Editor
   - Define output schema fields
3. **Update References**: If you used Data Transform Node output in other nodes, update to use App Node's `structured_output` field

### Backward Compatibility

- Existing workflows with Data Transform Node will continue to work
- No automatic migration is performed
- New workflows should use App Node with structured_output

## Usage Examples

### Example 1: Using Quick Template

1. Add App Node to workflow
2. Configure App Node with input (URL or file)
3. Enable "STRUCTURED" toggle
4. Click "Configure" to open Visual Editor
5. Click "Apply Template" and select "Supplier Check" template
6. Review and customize fields if needed
7. Run workflow - App Node will output both text and structured data

### Example 2: Custom Fields

1. Add App Node to workflow
2. Configure App Node with input (URL or file)
3. Enable "STRUCTURED" toggle
4. Click "Configure" and click "Add Field"
5. Configure each field:
   - **Field Name**: `has_qualified_option`
   - **Data Type**: `boolean` (with help text showing examples)
   - **Extraction Guide**: "Whether there are qualified supplier options available"
   - **Required**: Yes
6. Add more fields as needed
7. Run workflow - App Node will output both text and structured data

### Example 3: Single Node Testing

1. Configure App Node
2. Right-click node → "Run Node"
3. Select input source (previous node / manual / workflow input)
4. View output to understand data structure
5. Use output to configure subsequent nodes (e.g., IF node conditions)

### Example 4: Using Placeholder Variables

In a Condition Node's condition expression:
```
{{app-1.structured_output.has_qualified_option}} && {{app-1.structured_output.qualified_option_count}} > 0
```

## Technical Details

### Backend Implementation

- **Location**: `backend/app/services/workflow/workflow_execution_service.py`
- **Method**: `_extract_structured_output()`
- **LLM**: Uses Azure OpenAI GPT-5-mini for extraction
- **Validation**: JSON Schema validation with retry logic

### Frontend Implementation

- **Components**:
  - `AppNodeConfigPanel.tsx`: Main configuration panel
  - `StructuredOutputSchemaEditor.tsx`: Visual editor for schema
  - `SingleNodeRunModal.tsx`: Single node execution modal
- **API Endpoint**: `POST /api/v1/workbench/nodes/single-run`

## API Reference

### Single Node Run API

**Endpoint**: `POST /api/v1/workbench/nodes/single-run`

**Request Body**:
```json
{
  "node": {
    "id": "app-1",
    "type": "app",
    "data": { ... }
  },
  "node_type": "app",
  "input_data": { ... },  // Optional: manual input
  "previous_node_outputs": { ... },  // Optional: previous node outputs
  "workflow_input_data": { ... },  // Optional: workflow input
  "app_node_config": { ... }  // App node specific config
}
```

**Response**:
```json
{
  "success": true,
  "node_id": "app-1",
  "node_type": "app",
  "status": "completed",
  "output": {
    "text": "...",
    "structured_output": { ... }
  }
}
```

## Best Practices

1. **Use Templates**: Start with a quick template that matches your use case, then customize
2. **Start Simple**: Begin with basic field types (string, number, boolean)
3. **Clear Extraction Guides**: Write specific, clear descriptions in "Extraction Guide" field to help AI understand what to extract
4. **Test First**: Use single node run to verify output structure
5. **Iterate**: Refine schema based on actual output
6. **Mark Required Carefully**: Only mark fields as required if they're essential for downstream nodes
7. **Use Field Type Help**: Hover over the help icon next to "Data Type" to see examples and guidance

## Troubleshooting

### Structured Output Not Appearing

- Check if "STRUCTURED" toggle is enabled
- Verify schema is configured (click "Configure" to check)
- Review backend logs for extraction errors

### Single Node Run Fails

- Verify node configuration is complete
- Check input data format (must be valid JSON for manual input)
- Review API response for error details

### Placeholder Not Resolving

- Ensure referenced node has been executed
- Check node_id matches exactly
- Verify field path is correct (use dot notation for nested fields)


