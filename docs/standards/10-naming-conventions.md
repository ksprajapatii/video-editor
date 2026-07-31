# 10 — Naming Conventions

> **Document:** Naming Conventions v1.0
> **Project:** AI Video Editor
> **Date:** 2026-08-01

---

## 1. General Rules

1. Names must be **descriptive** — avoid abbreviations unless universally known (URL, ID, FPS)
2. Names must **explain intent**, not implementation
3. Boolean names start with `is`, `has`, `can`, `should`
4. Event handlers prefixed with `handle` (not `on`) when defined; `on` for props
5. Async functions do NOT add `Async` suffix — the `await` keyword is sufficient

---

## 2. TypeScript Naming

### 2.1 Variables & Functions
```typescript
// camelCase for all variables and functions
const clipDurationMs = 5000;
const isPlaying = false;
const hasUnsavedChanges = true;

function calculateTimelineWidth(tracks: Track[], zoomLevel: number): number { }
async function loadProjectFromDisk(projectPath: string): Promise<Project> { }

// Event handlers: handleXxx when defined as a function
const handleClipDrop = (event: DragEvent) => { };
const handlePlayheadSeek = (timeMs: number) => { };

// Props: onXxx
interface ClipProps {
  onDrop: (event: DragEvent) => void;
  onSeek: (timeMs: number) => void;
}
```

### 2.2 Types, Interfaces, Enums, Classes
```typescript
// PascalCase for all type-level constructs
interface TimelineClip { }        // Interface
type ClipId = string;             // Type alias
enum PlaybackState {              // Enum
  Idle = 'IDLE',
  Playing = 'PLAYING',
  Paused = 'PAUSED',
}
class ClipRepository { }         // Class

// Generic type params: T, U, TData, TError (descriptive preferred)
function useQuery<TData, TError extends Error>(): QueryResult<TData, TError> { }
```

### 2.3 React Components
```typescript
// PascalCase for component functions and files
export function TimelineTrack(): JSX.Element { }    // Component
export function useClipKeyframes(): KeyframeHook { } // Hook

// File names match component names exactly
// TimelineTrack.tsx -> TimelineTrack component
// useClipKeyframes.ts -> useClipKeyframes hook
```

### 2.4 Constants
```typescript
// SCREAMING_SNAKE_CASE for module-level constants
const MAX_UNDO_HISTORY = 100;
const DEFAULT_FRAME_RATE = 24;
const API_BASE_URL = 'http://127.0.0.1:8755/api/v1';
const BACKEND_WS_URL = 'ws://127.0.0.1:8756/ws';

// Enum values also SCREAMING_SNAKE_CASE
enum ExportPreset {
  YOUTUBE_4K = 'YOUTUBE_4K',
  INSTAGRAM_REEL = 'INSTAGRAM_REEL',
  TIKTOK = 'TIKTOK',
}
```

### 2.5 Store Naming
```typescript
// Zustand store: useXxxStore (camelCase, use prefix, Store suffix)
const useTimelineStore = create<TimelineState>()( ... );
const useMediaStore = create<MediaState>()( ... );
const useProjectStore = create<ProjectState>()( ... );
```

### 2.6 CSS Class Names
```
BEM (Block__Element--Modifier) for custom classes:
  .timeline-track                  -- Block
  .timeline-track__clip            -- Element
  .timeline-track__clip--selected  -- Modifier
  .timeline-track--collapsed       -- Block modifier

Tailwind utilities used as-is:
  className="flex items-center gap-2 text-sm font-medium"
```

---

## 3. Python Naming

### 3.1 Variables & Functions
```python
# snake_case for all variables and functions
clip_duration_ms = 5000
is_playing = False
has_unsaved_changes = True

def calculate_timeline_width(tracks: list[Track], zoom_level: float) -> int: ...
async def load_project_from_disk(project_path: Path) -> Project: ...
```

### 3.2 Classes
```python
# PascalCase for classes
class VideoClip: ...
class TimelineService: ...
class BackgroundRemovalWorker: ...
class AITaskQueue: ...
```

### 3.3 Constants & Module-Level
```python
# SCREAMING_SNAKE_CASE for constants
MAX_VRAM_GB = 2.0
DEFAULT_FRAME_RATE = 24.0
BACKEND_PORT = 8755
WEBSOCKET_PORT = 8756

# Private module constants start with underscore
_INTERNAL_BUFFER_SIZE = 4096
```

### 3.4 Private Members
```python
class ProjectService:
    # Public interface (no prefix)
    async def create(self, request: CreateProjectRequest) -> Project: ...
    
    # Private implementation detail (single underscore)
    async def _validate_settings(self, settings: ProjectSettings) -> None: ...
    
    # Name-mangled (double underscore) — rarely used
    def __secret_internal(self) -> None: ...
```

### 3.5 Pydantic Models
```python
# Models: PascalCase, suffix indicates type
class CreateProjectRequest(BaseModel): ...   # Request body
class ProjectResponse(BaseModel): ...        # Response body
class Project(BaseModel): ...                # Domain model
class ProjectSettings(BaseModel): ...        # Nested model
```

### 3.6 FastAPI Routers
```python
# Router variables: snake_case, suffix 'router'
project_router = APIRouter(prefix="/projects", tags=["projects"])
media_router = APIRouter(prefix="/media", tags=["media"])
ai_router = APIRouter(prefix="/ai", tags=["ai"])
```

---

## 4. File & Directory Naming

### 4.1 Frontend Files
```
PascalCase  -- React components, TypeScript classes
  TimelineTrack.tsx
  ClipItem.tsx
  VideoPreview.tsx
  ProjectService.ts   (service class)

camelCase   -- Hooks, utilities, stores, config
  useClipSelection.ts
  useTimelineZoom.ts
  timeline-store.ts   (Zustand stores use kebab-case)
  format-time.ts      (utility files use kebab-case)
  api-client.ts

kebab-case  -- All other files: configs, styles, tests
  tsconfig.json
  vite.config.ts
  timeline-track.module.css
  ClipItem.test.tsx   (test files match component name + .test.)
```

### 4.2 Backend Files
```
snake_case  -- All Python files
  project_service.py
  media_repository.py
  ai_engine.py
  render_worker.py
  test_project_service.py   (tests: test_ prefix)

__init__.py  -- All package directories
```

### 4.3 Directory Names
```
Frontend:
  features/timeline/       -- kebab-case for feature directories
  features/media-bin/
  shared/components/
  shared/hooks/

Backend:
  backend/services/        -- snake_case for Python directories
  backend/repositories/
  backend/api/v1/
```

---

## 5. API Naming Conventions

### 5.1 REST Endpoints
```
Nouns, plural, kebab-case:
  GET    /projects              -- List projects
  POST   /projects              -- Create project
  GET    /projects/{id}         -- Get project
  PUT    /projects/{id}         -- Update project
  DELETE /projects/{id}         -- Delete project
  
  GET    /projects/{id}/clips   -- Nested resource
  POST   /projects/{id}/export  -- Action on resource
  
  GET    /ai/tasks              -- Resource collection
  POST   /ai/tasks              -- Create task
  GET    /ai/tasks/{id}         -- Get task
  DELETE /ai/tasks/{id}         -- Cancel task
```

### 5.2 Query Parameters
```
camelCase for all query parameters:
  ?sortBy=createdAt
  ?filterStatus=pending
  ?pageSize=20
  ?pageToken=abc123
```

### 5.3 JSON Fields
```
camelCase for all JSON fields (frontend):
  { "clipId": "abc", "startTimeMs": 1000, "isSelected": true }

snake_case for Python Pydantic models (auto-converted):
  class Clip(BaseModel):
      clip_id: str
      start_time_ms: int
      is_selected: bool = False
```

---

## 6. Database Naming

```sql
-- Tables: snake_case, plural nouns
projects
media_items
timeline_tracks
timeline_clips
effect_instances
keyframes
export_jobs
ai_tasks
plugin_installations

-- Columns: snake_case
project_id    -- Primary keys: table_name_id
created_at    -- Timestamps: *_at
updated_at
is_deleted    -- Booleans: is_*, has_*, can_*
start_time_ms -- Units in name when ambiguous
file_size_bytes

-- Indexes: idx_table_column(s)
idx_timeline_clips_track_id
idx_media_items_project_id
```

---

## 7. Environment Variables

```
# SCREAMING_SNAKE_CASE
# Prefix: AVE_ (AI Video Editor)

AVE_BACKEND_PORT=8755
AVE_WS_PORT=8756
AVE_LOG_LEVEL=INFO
AVE_DATA_DIR=/Users/user/.aivideoedit
AVE_CUDA_DEVICE=0
AVE_MAX_VRAM_GB=2.0
AVE_ENABLE_TELEMETRY=false
```
