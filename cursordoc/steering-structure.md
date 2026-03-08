# Project Structure

## Directory Organization

```
foundation/
├── backend/                  # FastAPI services, SQLModel models, API routers
├── frontend/                 # React + TypeScript SPA (Vite-based)
│   ├── src/
│   │   ├── components/       # Chakra UI-based reusable components (AIGrid, ReportDisk, Workbench, etc.)
│   │   ├── routes/           # TanStack Router file-based routes (e.g., `_layout/aigrid.tsx`)
│   │   ├── services/         # Typed API clients (`GridSpaceService`, `DiskService`, `AppNodeService`)
│   │   ├── client/           # OpenAPI generated client code
│   │   └── assets/           # Frontend static assets
│   └── public/               # Public static files served by Vite
├── scripts/                  # Helper scripts (client generation, env setup)
├── bin/                      # Shell utilities for environment bootstrapping
├── docker-*/                 # Deployment descriptors for different environments
├── cursordocs/               # Living documentation generated during Cursor sessions
└── TroubleShooting.md        # Accumulated operational/debug notes
```

## Naming Conventions

### Files
- **Components/Modules**: `PascalCase` for React components (e.g., `AIGridItem.tsx`).
- **Services/Handlers**: `CamelCase` suffix `Service` (e.g., `GridSpaceService.ts`).
- **Utilities/Helpers**: `camelCase` descriptive names (e.g., `gridUtils.ts`).
- **Tests**: Backend tests end with `_test.py`; frontend tests follow `.test.tsx`.

### Code
- **Classes/Types**: `PascalCase` (`GridItemData`, `DiskExecutionResult`).
- **Functions/Methods**: `camelCase` (`handleGridEdit`, `loadGridSpaceData`).
- **Constants**: `UPPER_SNAKE_CASE` or `PascalCase` depending on context (`GRID_CONFIG`, `DEFAULT_ITEM_SIZE`).
- **Variables**: `camelCase` for both backend and frontend local bindings.

## Import Patterns

### Import Order
1. External dependencies (React, Chakra UI, FastAPI, etc.).
2. Internal modules via alias (`@/components/...`, `@/services/...`).
3. Relative imports for sibling files.
4. Styles (`.css`) imported last when required.

### Module/Package Organization
```
- Frontend uses Vite path aliases (`@/`) for absolute imports rooted at `frontend/src`.
- Backend modules leverage Python package imports from `app.` namespace.
- Shared service modules encapsulate REST calls; UI layers consume services via absolute aliases.
- Avoid deep relative paths (../../); prefer configured aliases.
```

## Code Structure Patterns

### Module/Class Organization
```
1. Imports
2. Constants / configuration objects (e.g., `GRID_CONFIG`)
3. Type/interface definitions
4. Component or service implementation
5. Helper functions
6. Export statements (`export const AIGridItem = ...`)
```

### Function/Method Organization
```
- Validate inputs or early-return guards first
- Execute core logic with clear hook usage (React) or service calls
- Apply error handling via `try/catch` (async functions) or toast notifications
- Return JSX/JSON payload last
```

### File Organization Principles
```
- One primary React component or service per file
- Related helpers colocated when tightly coupled
- Public exports defined at bottom; helper declarations kept local unless reused
- Extensive files (e.g., `AIGridLayout.tsx`) segmented with comment blocks and callback sections
```

## Code Organization Principles

1. **Single Responsibility**: Grid items manage presentation; services manage API interactions.
2. **Modularity**: Disk sidebar, grid layout, and report viewer exist as discrete components.
3. **Testability**: Business logic resides in services/utilities; React components rely on props for deterministic rendering.
4. **Consistency**: Follow Chakra UI styling patterns, reuse `useCallback` hooks for event handlers, and mirror autosave flows across features.

## Module Boundaries
- **Frontend UI vs Services**: UI components interact with backend strictly through typed service modules.
- **Grid vs Disk Sidebar**: `AIGridLayout` owns selection state; `AIGridDiskSidebar` receives explicit props and does not mutate grid layout directly.
- **Report Modules**: `ReportDiskStandardView` encapsulates upload/execution pipeline; other views consume its hooks rather than duplicating logic.
- **Backend Separation**: API routers expose disk/grid endpoints; database models isolated under SQLModel definitions.

## Code Size Guidelines
- **File size**: Aim to keep React component files under 500 lines; when exceeding (e.g., `AIGridLayout`), consider extracting hooks/utilities.
- **Function/Method size**: Prefer functions under ~80 lines; factor out helpers for larger workflows.
- **Class/Module complexity**: Limit nested callbacks to three levels; reuse hooks for shared behavior.
- **Nesting depth**: Keep JSX nesting shallow by splitting subcomponents.

## Dashboard/Monitoring Structure (if applicable)
```
frontend/src/components/ReportDisk/
├── ReportDiskStandardView.tsx   # Standard Mode uploader/analyzer
├── hooks/                       # Reusable upload hooks
├── components/                  # Status list, forms, result displays
```
- Separation: Report modules isolate ingestion state from AI Grid; they can be reused in modals or standalone routes.

## Documentation Standards
- Major features require Markdown docs under `cursordocs/` with implementation notes.
- `README.md` must mirror top-level directory structure after structural changes.
- Inline comments reserved for complex logic (e.g., grid auto-placement, Markdown table transforms).
- Backend routers and services documented via docstrings aligned with FastAPI schema metadata.
