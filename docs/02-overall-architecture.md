# 02 — Overall Architecture

> **Document:** Architecture Overview v1.0
> **Project:** AI Video Editor
> **Date:** 2026-08-01

---

## 1. Architectural Style

The AI Video Editor uses a **Layered Hexagonal Architecture** combined with an **Event-Driven** communication model.

### Primary Patterns
| Pattern | Where Used | Reason |
|---------|-----------|--------|
| Layered Architecture | Entire app | Clear separation of concerns |
| Hexagonal (Ports & Adapters) | Backend services | Testability, swap-ability |
| Event-Driven | Frontend state, IPC | Decoupled real-time updates |
| CQRS (light) | Timeline operations | Optimized read vs write paths |
| Repository Pattern | Database access | Abstracted persistence |
| Observer | Effect/AI pipelines | Reactive updates |

---

## 2. Top-Level System Architecture

```
+=====================================================================+
|                         AI VIDEO EDITOR                             |
+=====================================================================+
|                                                                     |
|  +--------------------------+    +-----------------------------+    |
|  |   RENDERER PROCESS       |    |       MAIN PROCESS          |    |
|  |   (Electron BrowserWindow|    |       (Electron Main)       |    |
|  |                          |    |                             |    |
|  |  +--------------------+  |    |  +-----------------------+  |    |
|  |  |  React Application |  |    |  |  IPC Bridge           |  |    |
|  |  |  (TypeScript)       |  |IPC|  |  (ipcMain handlers)   |  |    |
|  |  |                     |<--->|  +-----------+-----------+  |    |
|  |  |  +--------------+   |  |    |              |             |    |
|  |  |  | Zustand Store|   |  |    |  +-----------v-----------+|    |
|  |  |  +--------------+   |  |    |  |  HTTP/WS Client       ||    |
|  |  |  +--------------+   |  |    |  +----------+------------+|    |
|  |  |  | Canvas/WebGL |   |  |    |             |              |    |
|  |  |  | Preview      |   |  |    +-------------|-------------+    |
|  |  |  +--------------+   |  |                  |                  |
|  |  +--------------------+  |                   | REST + WebSocket  |
|  +--------------------------+                   |                  |
|                                    +------------v-----------+       |
|                                    |    PYTHON BACKEND      |       |
|                                    |    (FastAPI Process)   |       |
|                                    |                        |       |
|                                    |  +------------------+  |       |
|                                    |  | API Router Layer |  |       |
|                                    |  +--------+---------+  |       |
|                                    |           |             |       |
|                                    |  +--------v---------+  |       |
|                                    |  | Service Layer    |  |       |
|                                    |  | (Business Logic) |  |       |
|                                    |  +--+---+---+---+--+  |       |
|                                    |     |   |   |   |      |       |
|                                    |  +--v+ +v+ +v+ +v--+  |       |
|                                    |  |FFm| |AI| |DB| |FS|  |       |
|                                    |  |peg| |En| |  | |  |  |       |
|                                    |  +---+ +--+ +--+ +--+  |       |
|                                    +------------------------+       |
+=====================================================================+
```

---

## 3. Process Model

### 3.1 Electron Process Architecture
```
Electron Application
    |
    +-- Main Process (Node.js)
    |       - App lifecycle management
    |       - Native OS integration
    |       - IPC message routing
    |       - Python process spawning
    |       - File system operations (privileged)
    |       - Window management
    |       - Tray / menu bar
    |
    +-- Renderer Process (Chromium)
    |       - React UI
    |       - WebGL preview canvas
    |       - Zustand state management
    |       - Timeline virtualization
    |
    +-- Preload Script
    |       - contextBridge (safe IPC exposure)
    |       - No direct Node.js access from renderer
    |
    +-- Python Backend Process
            - FastAPI HTTP server (localhost:8755)
            - WebSocket server (localhost:8756)
            - FFmpeg subprocess management
            - AI model inference (PyTorch)
            - Database access (SQLite)
```

### 3.2 Communication Channels
| Channel | Protocol | Usage |
|---------|---------|-------|
| Renderer <-> Main | Electron IPC (contextBridge) | File dialogs, OS calls |
| Main <-> Backend | HTTP REST (localhost) | Request/response operations |
| Main <-> Backend | WebSocket (localhost) | Real-time progress, events |
| Backend <-> FFmpeg | subprocess stdin/stdout | Encode/decode commands |
| Backend <-> AI Models | Python function calls | Direct in-process |

---

## 4. Layered Architecture

### Frontend Layers
```
Layer 5: UI Components (React Components)
    - Presentational, pure, stateless where possible
Layer 4: Feature Modules (Timeline, MediaBin, Preview...)
    - Feature-specific logic and composition
Layer 3: State Management (Zustand stores)
    - Application state, actions, selectors
Layer 2: API Client (services/)
    - Axios REST calls, WebSocket subscriptions
Layer 1: IPC Bridge (electron-api.ts)
    - Wraps contextBridge calls
```

### Backend Layers
```
Layer 5: API Routes (FastAPI routers)
    - HTTP handlers, request validation (Pydantic)
Layer 4: Service Layer (services/)
    - Business logic, orchestration
Layer 3: Domain Models (models/)
    - Pure Python domain objects
Layer 2: Repository Layer (repositories/)
    - Database CRUD abstractions
Layer 1: Infrastructure (db/, ffmpeg/, ai/)
    - SQLite, FFmpeg, PyTorch adapters
```

---

## 5. Data Flow: Core Editing Loop

```
User Action (click/drag in timeline)
    |
    v
React Component
    |
    v
Zustand Action (optimistic update)
    |
    v
API Service call --> HTTP POST to FastAPI
    |                       |
    v                       v
UI updates immediately   Backend validates & persists
                              |
                              v
                         WebSocket event --> Renderer
                              |
                              v
                         Zustand confirms/corrects state
```

---

## 6. Module Organization

### Frontend Module Boundaries
```
src/
  app/              -- App entry, routing
  features/
    timeline/       -- Timeline editing module
    media-bin/      -- Media management module
    preview/        -- Playback & preview module
    color/          -- Color grading module
    audio/          -- Audio mixing module
    effects/        -- Effects & transitions module
    ai-tools/       -- AI feature UIs module
    export/         -- Export & delivery module
    plugins/        -- Plugin system UI module
    settings/       -- App settings module
  shared/
    components/     -- Shared UI components
    hooks/          -- Shared React hooks
    stores/         -- Zustand stores
    services/       -- API clients
    utils/          -- Pure utility functions
    types/          -- TypeScript types/interfaces
```

### Backend Module Boundaries
```
backend/
  api/
    v1/
      projects/     -- Project CRUD endpoints
      timeline/     -- Timeline operations
      media/        -- Media management
      render/       -- Render job management
      ai/           -- AI task endpoints
      export/       -- Export endpoints
      plugins/      -- Plugin management
  services/
    project_service.py
    timeline_service.py
    media_service.py
    render_service.py
    ai_service.py
    export_service.py
    plugin_service.py
  repositories/
    project_repo.py
    media_repo.py
    timeline_repo.py
  domain/
    models/         -- Pydantic domain models
    events/         -- Domain event definitions
  infrastructure/
    db/             -- SQLite connection, migrations
    ffmpeg/         -- FFmpeg wrapper
    ai/             -- AI model wrappers
    storage/        -- File system abstraction
```

---

## 7. Security Architecture

```
+----------------------------------+
|   SECURITY PERIMETER             |
|                                  |
|  Renderer Process                |
|  - contextIsolation: true        |
|  - nodeIntegration: false        |
|  - sandbox: true                 |
|  - CSP headers enforced          |
|                                  |
|  Plugin Sandbox                  |
|  - Worker thread isolation       |
|  - No fs access                  |
|  - Message passing only          |
|                                  |
|  Backend API                     |
|  - localhost only (127.0.0.1)    |
|  - No external network exposure  |
|  - Request origin validation     |
+----------------------------------+
```

---

## 8. State Management Architecture

### Frontend State Domains (Zustand)
| Store | Responsibility |
|-------|---------------|
| `useProjectStore` | Current project, settings |
| `useTimelineStore` | Clips, tracks, playhead, zoom |
| `useMediaStore` | Media bin contents |
| `usePlaybackStore` | Playback state, current frame |
| `useColorStore` | Color grading parameters |
| `useAudioStore` | Audio mixer state |
| `useEffectsStore` | Applied effects per clip |
| `useAIStore` | AI task queue and results |
| `useExportStore` | Export queue and progress |
| `useUIStore` | Panel visibility, layouts |

---

## 9. Deployment Architecture

```
Distribution Package
    |
    +-- Windows: .exe installer (NSIS) + .msix (Microsoft Store)
    +-- macOS: .dmg (Universal Binary)
    +-- Linux: .AppImage + .deb + .rpm
    
Each package contains:
    - Electron runtime
    - React build (bundled)
    - Python runtime (embedded)
    - AI models (optional download on first use)
    - FFmpeg binary
    - SQLite (bundled with Python)
```
