# Product Overview

## Product Purpose
The Foundation project delivers an enterprise AI workspace that combines disk-based knowledge management, workflow automation, and analytics inside a unified grid editor. Users orchestrate applications, attach AI disks, and trigger analysis pipelines without leaving the grid canvas.

## Target Users
- Internal analysts who curate document collections ("disks") and run automated payment or compliance checks.
- Product operators who build and publish reusable AI applications from the Application Marketplace.
- Administrators who manage disk metadata, prompts, and execution history through the grid interface.

## Key Features
1. **AI Grid Workspace**: Drag-and-drop layout with edit mode, disk pinning, and disk property sidebars for each grid item.
2. **Disk Execution & Results**: Persists latest execution outputs (`latestResult`) per disk, rendering structured Markdown, tables, and ECharts dashboards.
3. **Document Upload & Processing**: Standard Mode flow for URL/file ingestion, leveraging `ReportDiskStandardView` to drive analysis pipelines and status tracking.
4. **Application & Disk Management**: Integration with `GridSpaceService`, `DiskService`, and `AppNodeService` for CRUD operations, autosave, and grid serialization.
5. **Security & Auth**: Foundation inherits FastAPI-based authentication, JWT handling, and role-aware API access from the template stack.

## Business Objectives
- Streamline analyst workflows by centralizing disk editing, execution, and reporting.
- Increase reuse of configured disks/applications through publishable grid spaces.
- Provide auditable execution history stored in PostgreSQL via FastAPI services.

## Success Metrics
- Grid autosave success rate (target 99%+) via `GridSpaceService.autoSaveGridSpace` responses.
- Mean execution turnaround time captured in `DiskService.getDiskExecutionResults` payloads.
- Active grid spaces count per user derived from `grid_space` table records.

## Product Principles
1. **Workspace First**: The AI grid remains the primary surface; auxiliary dialogs (disk sidebar, run modal) must not break the context.
2. **Persist Everything**: Text edits, layout changes, and execution reports serialize back through existing services so sessions are resumable.
3. **Composable Automation**: Disks follow consistent APIs, enabling analysts to combine upload pipelines, prompts, and dashboards interchangeably.

## Monitoring & Visibility (if applicable)
- **Dashboard Type**: Web-based grid with Chakra UI-driven panels and disk detail cards.
- **Real-time Updates**: TanStack Query requests refresh disk metadata; autosave hooks monitor unsaved changes per grid space.
- **Key Metrics Displayed**: Execution status badges, created-at timestamps, dashboard charts from disk requirements.
- **Sharing Capabilities**: Grid spaces persist via backend services, enabling published states for collaboration.

## Future Vision
### Potential Enhancements
- **Remote Access**: Surface read-only grid snapshots for stakeholders via published disk views.
- **Analytics**: Aggregate execution KPIs across disks directly in the workspace dashboard.
- **Collaboration**: Multi-user editing with presence awareness layered on `AIGridLayout` selections.
