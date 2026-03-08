# Visual Workflow Designer Design Document

## Overview

This document outlines the design for a visual workflow designer to replace the current text-based approval workflow configuration system. The new system will provide an intuitive, drag-and-drop interface similar to Smartsheet and SharePoint Power Automate.

## Current Problems

1. **Unfriendly Node Management**: Each approval node must be created separately, leading to duplicate step_order values
2. **Lack of Visualization**: No visual representation of the entire approval process
3. **Complex Configuration**: Manual setup of detailed information for each node
4. **Poor User Experience**: Text-based forms are not intuitive for workflow design

## Design Principles

### 1. Visual-First Approach
- Drag-and-drop interface for building workflows
- Real-time visual representation of the approval process
- Clear visual indicators for different node types and connections

### 2. Template-Based Design
- Pre-built templates for common approval patterns
- Customizable templates that users can modify
- Quick-start options for different organizational structures

### 3. Progressive Disclosure
- Start with simple configurations
- Advanced options available when needed
- Contextual help and guidance

### 4. Real-Time Validation
- Immediate feedback on configuration errors
- Visual indicators for incomplete or invalid configurations
- Auto-save functionality

## Interface Design

### Main Components

#### 1. Workflow Canvas
- **Central Design Area**: Large canvas for building workflows
- **Grid System**: Snap-to-grid for clean alignment
- **Zoom Controls**: Zoom in/out for detailed work
- **Minimap**: Overview of the entire workflow

#### 2. Node Palette
- **Start Node**: Workflow initiation point
- **Approval Node**: Individual approval steps
- **Condition Node**: Decision points and branching
- **End Node**: Workflow completion
- **Parallel Node**: Parallel approval paths

#### 3. Properties Panel
- **Node Configuration**: Detailed settings for selected node
- **Validation Messages**: Real-time error checking
- **Preview**: Live preview of node behavior

#### 4. Toolbar
- **Save/Export**: Save workflow configurations
- **Undo/Redo**: Version control
- **Test Mode**: Simulate workflow execution
- **Template Library**: Access to pre-built templates

### Node Types

#### Approval Nodes
- **Department Head**: Automatic department head assignment
- **Specific User**: Manually assigned user
- **Role-Based**: Assignment based on user roles
- **Manager Hierarchy**: Automatic manager chain assignment

#### Condition Nodes
- **Amount Threshold**: Conditional approval based on policy value
- **Department Type**: Different paths for different departments
- **Policy Category**: Category-specific approval paths
- **Custom Logic**: User-defined conditions

#### Parallel Processing
- **Parallel Approval**: Multiple approvers at the same level
- **Sequential Approval**: One-after-another approval
- **Mixed Mode**: Combination of parallel and sequential

## Technical Implementation

### Frontend Architecture

#### 1. Workflow Designer Component
```typescript
interface WorkflowDesigner {
  canvas: WorkflowCanvas;
  palette: NodePalette;
  properties: PropertiesPanel;
  toolbar: Toolbar;
}
```

#### 2. Node System
```typescript
interface WorkflowNode {
  id: string;
  type: NodeType;
  position: { x: number; y: number };
  connections: Connection[];
  configuration: NodeConfiguration;
}
```

#### 3. Connection System
```typescript
interface Connection {
  id: string;
  from: string; // source node id
  to: string;   // target node id
  condition?: Condition;
}
```

### Backend API Changes

#### 1. Workflow Definition Schema
```json
{
  "template_id": 1,
  "name": "Standard Policy Approval",
  "description": "Two-level approval workflow",
  "nodes": [
    {
      "id": "start",
      "type": "start",
      "position": { "x": 100, "y": 100 }
    },
    {
      "id": "dept_head",
      "type": "approval",
      "approver_type": "department_head",
      "position": { "x": 300, "y": 100 },
      "step_order": 1
    },
    {
      "id": "policy_admin",
      "type": "approval", 
      "approver_type": "role_based",
      "approver_role_code": "policy_admin",
      "position": { "x": 500, "y": 100 },
      "step_order": 2
    },
    {
      "id": "end",
      "type": "end",
      "position": { "x": 700, "y": 100 }
    }
  ],
  "connections": [
    { "from": "start", "to": "dept_head" },
    { "from": "dept_head", "to": "policy_admin" },
    { "from": "policy_admin", "to": "end" }
  ]
}
```

#### 2. New API Endpoints
- `POST /api/v1/workflow-templates/` - Create workflow template
- `PUT /api/v1/workflow-templates/{id}` - Update workflow template
- `GET /api/v1/workflow-templates/{id}/visual` - Get visual representation
- `POST /api/v1/workflow-templates/{id}/validate` - Validate workflow

### Database Schema Updates

#### 1. Enhanced Template Table
```sql
ALTER TABLE policy_approval_templates ADD COLUMN visual_config JSON;
ALTER TABLE policy_approval_templates ADD COLUMN workflow_type VARCHAR(50) DEFAULT 'sequential';
```

#### 2. Node Definitions Table
```sql
ALTER TABLE policy_approval_node_definitions ADD COLUMN node_id VARCHAR(100);
ALTER TABLE policy_approval_node_definitions ADD COLUMN position_x INTEGER;
ALTER TABLE policy_approval_node_definitions ADD COLUMN position_y INTEGER;
ALTER TABLE policy_approval_node_definitions ADD COLUMN parent_node_id VARCHAR(100);
```

## User Experience Flow

### 1. Template Selection
- User opens workflow configuration
- Chooses from template library or starts blank
- Template loads with pre-configured nodes

### 2. Workflow Building
- Drag nodes from palette to canvas
- Connect nodes by dragging between connection points
- Configure each node using properties panel
- Real-time validation and error checking

### 3. Testing and Validation
- Test mode allows simulation of workflow
- Validation checks for logical errors
- Preview shows how workflow will execute

### 4. Save and Deploy
- Save workflow configuration
- Set as working template
- Deploy to production use

## Migration Strategy

### Phase 1: Basic Visual Designer
- Implement drag-and-drop canvas
- Support basic node types (start, approval, end)
- Simple sequential workflows

### Phase 2: Advanced Features
- Parallel processing support
- Condition nodes and branching
- Template library

### Phase 3: Enhanced UX
- Real-time collaboration
- Version control
- Advanced testing tools

## Benefits

1. **Improved User Experience**: Intuitive visual interface
2. **Reduced Errors**: Visual validation prevents configuration mistakes
3. **Faster Configuration**: Drag-and-drop is faster than form filling
4. **Better Understanding**: Visual representation helps users understand workflows
5. **Easier Maintenance**: Visual interface makes updates simpler

## Success Metrics

- **Configuration Time**: Reduce time to create workflows by 70%
- **Error Rate**: Reduce configuration errors by 80%
- **User Satisfaction**: Increase user satisfaction scores
- **Adoption Rate**: Increase usage of workflow configuration features

## Conclusion

The visual workflow designer will transform the current text-based configuration system into an intuitive, user-friendly interface that follows industry best practices. This will significantly improve the user experience and reduce configuration errors while making workflow management more accessible to all users.
