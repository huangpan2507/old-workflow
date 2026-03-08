# Technology Stack

## Project Type
Full-stack web application combining a FastAPI backend with a React (Vite + TypeScript + Chakra UI) frontend. The AI Grid workspace and disk execution tooling run as part of this monolithic stack.

## Core Technologies

### Primary Language(s)
- **Language**: Python 3.11 (backend), TypeScript 5.x (frontend)
- **Runtime/Compiler**: CPython interpreter; Vite build tool with ESBuild/Rollup pipeline
- **Language-specific tools**: Poetry/`pip` for backend dependencies; `npm` for frontend packages

### Key Dependencies/Libraries
- **FastAPI / SQLModel**: REST API layer and ORM for PostgreSQL persistence.
- **Chakra UI**: Component system for the AI Grid workspace, menus, dialogs, and toolbars.
- **React Grid Layout**: Drives grid positioning and resizing for `AIGridLayout`.
- **TanStack Router & Query**: Client-side routing and data fetching/caching.
- **React Markdown + remark-gfm**: Renders disk execution Markdown with tables and code blocks.
- **ECharts (echarts-for-react)**: Visualizes dashboard charts embedded inside disk cards.

### Application Architecture
Monolithic client-server architecture. Backend exposes REST endpoints for grid spaces, disks, applications, and file ingestion. Frontend consumes services (`GridSpaceService`, `DiskService`, `AppNodeService`) to render the AI workspace and trigger autosave.

### Data Storage (if applicable)
- **Primary storage**: PostgreSQL (grid spaces, disks, execution results).
- **Caching**: In-memory caching through TanStack Query on the frontend.
- **Data formats**: JSON payloads for API transport; Markdown/ECharts JSON stored in `conclusion_detail` fields.

### External Integrations (if applicable)
- **APIs**: Internal FastAPI services for disk execution, document ingestion, and application marketplace data.
- **Protocols**: HTTPS/HTTP REST endpoints; optional S3-compatible storage used in upload flows.
- **Authentication**: JWT bearer tokens issued by backend auth endpoints.

### Monitoring & Dashboard Technologies (if applicable)
- **Dashboard Framework**: React + Chakra UI components inside `ReportDiskStandardView` and AI Grid.
- **Real-time Communication**: Polling via TanStack Query invalidations; no dedicated WebSocket layer yet.
- **Visualization Libraries**: ECharts for data charts, Chakra badges and tables for status views.
- **State Management**: React hooks with local state plus TanStack Query caches for server data.

## Development Environment

### Build & Development Tools
- **Build System**: `npm` scripts (`vite`, `tsc`), backend `make`/Python scripts.
- **Package Management**: `pip`/`poetry` for Python, `npm` for frontend.
- **Development workflow**: Hot reload via `npm run dev` (Vite) and FastAPI auto-reload in dev containers.

### Code Quality Tools
- **Static Analysis**: `mypy`, `flake8` (backend); `tsc`, ESLint (frontend).
- **Formatting**: `black`, `isort` (backend); `prettier` (frontend).
- **Testing Framework**: `pytest` for backend unit/integration tests; Playwright for end-to-end frontend tests.
- **Documentation**: Markdown-based docs under `README.md`, `TroubleShooting.md`, and `cursordocs/`.

### Version Control & Collaboration
- **VCS**: Git.
- **Branching Strategy**: GitHub Flow (feature branches merged into main).
- **Code Review Process**: Pull requests reviewed via GitHub; Cursor specs ensure requirements/design/tasks alignment.

### Dashboard Development (if applicable)
- **Live Reload**: Vite HMR for frontend changes.
- **Port Management**: Frontend dev server on 5173, backend on 8000/8001 (documented in env files).
- **Multi-Instance Support**: Docker compose enables concurrent backend/frontend containers; local dev typically single instance.

## Deployment & Distribution (if applicable)
- **Target Platform(s)**: Dockerized services for cloud or on-premise deployment.
- **Distribution Method**: Docker images via `docker-compose.*.yml` stacks.
- **Installation Requirements**: Docker 27.4.1+, Compose v2.29.7, configured `.env` secrets.
- **Update Mechanism**: Rebuild/pull latest Docker images and migrate database as required.

## Technical Requirements & Constraints

### Performance Requirements
- Ensure autosave operations (`GridSpaceService.autoSaveGridSpace`) complete within 3s to preserve editing UX.
- UI interactions (menu opening, toolbar rendering) target < 100ms response on modern browsers.

### Compatibility Requirements
- **Platform Support**: Modern Chromium-based browsers; backend runs on Linux containers.
- **Dependency Versions**: Align with template defaults (FastAPI >=0.110, Chakra UI 2.x, React 18).
- **Standards Compliance**: RESTful APIs returning JSON; follow JWT RFC 7519 semantics.

### Security & Compliance
- **Security Requirements**: JWT authentication, HTTPS termination via reverse proxy, role-based disk access.
- **Compliance Standards**: Data stored in PostgreSQL with audit logging; align with organizational governance policies.
- **Threat Model**: Protect autosave payloads and upload endpoints from unauthorized access; sanitize Markdown to prevent XSS.

### Scalability & Reliability
- **Expected Load**: Single-digit concurrent analysts per workspace; scaling handled by increasing backend replicas.
- **Availability Requirements**: 99% uptime target for AI workspace; autosave retries mitigate transient failures.
- **Growth Projections**: Grid spaces and disks scale horizontally through pagination and lazy-loading of execution results.

## Technical Decisions & Rationale
1. **React Grid Layout for Canvas**: Provides mature drag-drop/resizing primitives; alternatives (Konva, custom) would increase maintenance.
2. **Markdown + ECharts Rendering**: Ensures disk results display rich content without bespoke visualization code.
3. **Autosave via GridSpaceService**: Serializing layouts/items as JSON keeps backend schema flexible for future disk attributes.

## Known Limitations
- **Rich Text Editing Gap**: Current Markdown rendering lacks inline editing; upcoming toolbar must integrate without breaking autosave pipeline.
- **Execution Refresh Latency**: Disk results require manual refresh after uploads; no WebSocket push yet to notify completion.
