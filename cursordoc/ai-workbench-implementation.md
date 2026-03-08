# AI Workbench Implementation Guide

## Overview

AI Workbench is a collaborative visual workspace that allows multiple users to create, edit, and organize sticky notes in real-time. Built with React-Konva for the canvas, Yjs for collaboration, and FastAPI WebSocket for real-time communication.

## Architecture

### Frontend Stack
- **React + TypeScript**: Core UI framework
- **Chakra UI**: Component library for consistent design
- **React-Konva**: Canvas rendering and interaction
- **Yjs**: Conflict-free replicated data types (CRDT) for collaboration
- **Zustand**: State management
- **TanStack Router**: Routing

### Backend Stack
- **FastAPI**: Web framework with WebSocket support
- **WebSocket**: Real-time communication
- **Connection Manager**: Custom WebSocket connection management

## Features

### ✅ Implemented Features

#### Core Canvas
- **Interactive Canvas**: Zoom, pan, grid background
- **Sticky Notes**: Create, edit, drag, resize, delete
- **Tool Selection**: Select, note creation, move tools
- **Responsive Layout**: Sidebar, canvas, properties panel

#### Collaboration
- **Real-time Sync**: Yjs-based CRDT synchronization
- **Multi-user Support**: Multiple users in same workspace
- **WebSocket Communication**: FastAPI WebSocket backend
- **User Presence**: Online user indicators

#### UI/UX
- **Internationalization**: English/Chinese support
- **Responsive Design**: Adapts to different screen sizes
- **Keyboard Shortcuts**: Ctrl+Enter for quick actions
- **Toast Notifications**: User feedback

### 🚧 Planned Features

#### Advanced Collaboration
- **Cursor Tracking**: Real-time cursor positions
- **User Avatars**: Visual user identification
- **Conflict Resolution**: Advanced merge strategies
- **Offline Support**: Work offline, sync when reconnected

#### Enhanced Canvas
- **Note Templates**: Pre-designed note styles
- **Grouping**: Group related notes
- **Layers**: Organize content in layers
- **Export/Import**: Save and load workspaces

## File Structure

```
frontend/src/
├── routes/_layout/
│   └── workbench.tsx           # Main workbench page
├── stores/
│   └── workbenchStore.ts       # Zustand state management
├── components/Common/
│   └── Sidebar.tsx             # Updated with workbench menu
└── locales/
    ├── en.json                 # English translations
    └── zh.json                 # Chinese translations

backend/app/api/v1/
└── workbench.py                # WebSocket endpoints
```

## API Endpoints

### WebSocket
- `WS /api/v1/workbench/ws/workbench/{room_id}`: Real-time collaboration

### REST API
- `GET /api/v1/workbench/rooms/{room_id}/participants`: Get room participants
- `GET /api/v1/workbench/rooms`: List active rooms
- `POST /api/v1/workbench/rooms/{room_id}/broadcast`: Server broadcast
- `GET /api/v1/workbench/health`: Health check

## Data Models

### StickyNote
```typescript
interface StickyNote {
  id: string
  x: number
  y: number
  width: number
  height: number
  content: string
  color: string
  createdBy: string
  createdAt: string
  updatedAt?: string
}
```

### User
```typescript
interface User {
  id: string
  name: string
  color: string
  isOnline: boolean
  cursor?: { x: number; y: number }
}
```

## State Management

### Zustand Store Structure
```typescript
interface WorkbenchState {
  // Canvas state
  zoom: number
  stagePos: { x: number; y: number }
  selectedTool: 'select' | 'note' | 'move'
  selectedNoteId: string | null
  
  // Notes and collaboration
  stickyNotes: StickyNote[]
  onlineUsers: User[]
  currentUser: User | null
  roomId: string | null
  
  // Yjs collaboration
  ydoc: Y.Doc | null
  provider: WebsocketProvider | null
  yNotes: Y.Map<any> | null
}
```

## Collaboration Flow

### 1. User Joins Room
1. User opens workbench page
2. Store initializes collaboration with room ID
3. WebSocket connection established
4. User added to room participants
5. Existing participants notified

### 2. Note Operations
1. User creates/modifies note
2. Change applied to Yjs document
3. Yjs automatically syncs via WebSocket
4. Other users receive updates
5. UI updates reflect changes

### 3. Real-time Sync
- **Yjs CRDT**: Handles conflict resolution automatically
- **WebSocket**: Provides transport layer
- **Connection Manager**: Manages room participants

## Development Setup

### Prerequisites
```bash
# Frontend dependencies already installed
cd frontend
npm install react-konva konva yjs y-websocket zustand

# Backend - no additional dependencies needed
# WebSocket support included in FastAPI
```

### Running the Application
```bash
# Start backend (with WebSocket support)
cd backend
uvicorn app.main:app --reload --host 0.0.0.0 --port 8001

# Start frontend
cd frontend
npm run dev
```

### Testing Collaboration
1. Open multiple browser tabs/windows
2. Navigate to `/workbench` in each
3. Create notes in one tab
4. Observe real-time updates in other tabs

## Configuration

### Environment Variables
```bash
# Frontend
NODE_ENV=development  # Determines WebSocket URL

# Backend
# No additional config needed for basic WebSocket
```

### WebSocket URLs
- **Development**: `ws://localhost:8001/api/v1/workbench/ws/workbench/{roomId}`
- **Production**: `wss://{host}/api/v1/workbench/ws/workbench/{roomId}`

## Security Considerations

### Current Implementation
- Basic room-based isolation
- User identification via query parameters
- No authentication on WebSocket (development mode)

### Production Recommendations
- JWT token authentication for WebSocket
- Room access control
- Rate limiting for WebSocket messages
- Input validation and sanitization

## Performance Optimization

### Current Optimizations
- Efficient Konva rendering
- Zustand state management
- WebSocket connection pooling
- Grid rendering optimization

### Future Optimizations
- Virtual scrolling for large canvases
- Note clustering for performance
- WebSocket message batching
- Canvas viewport culling

## Troubleshooting

### Common Issues

#### WebSocket Connection Failed
```bash
# Check backend is running
curl http://localhost:8001/api/v1/workbench/health

# Check WebSocket endpoint
wscat -c ws://localhost:8001/api/v1/workbench/ws/workbench/test-room
```

#### Notes Not Syncing
1. Check browser console for errors
2. Verify WebSocket connection status
3. Check Yjs document state
4. Restart both frontend and backend

#### Performance Issues
1. Reduce canvas size if too large
2. Limit number of notes
3. Check for memory leaks in browser dev tools
4. Monitor WebSocket message frequency

## Future Enhancements

### Phase 1: Core Improvements
- [ ] Cursor tracking and user presence
- [ ] Note editing with rich text
- [ ] Undo/redo functionality
- [ ] Keyboard shortcuts

### Phase 2: Advanced Features
- [ ] Note templates and themes
- [ ] File attachments
- [ ] Drawing tools
- [ ] Voice/video integration

### Phase 3: Enterprise Features
- [ ] Workspace permissions
- [ ] Audit logging
- [ ] Integration with external tools
- [ ] Advanced analytics

## Contributing

### Code Style
- Follow existing TypeScript/React patterns
- Use Chakra UI components consistently
- Maintain internationalization support
- Write comprehensive error handling

### Testing
- Test multi-user scenarios
- Verify WebSocket reconnection
- Check mobile responsiveness
- Validate accessibility features

## License

This implementation is part of the Foundation v2.0 project and follows the same licensing terms.




