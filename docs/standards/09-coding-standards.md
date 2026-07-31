# 09 — Coding Standards

> **Document:** Coding Standards v1.0
> **Project:** AI Video Editor
> **Date:** 2026-08-01

---

## 1. General Principles

1. **Clarity over cleverness** — Code is read 10x more than written. Be explicit.
2. **Single Responsibility** — Each function/class does ONE thing.
3. **Fail fast** — Validate inputs at boundaries; crash loudly in dev, gracefully in prod.
4. **No magic numbers** — All constants named and documented.
5. **Immutability preferred** — Avoid mutating shared state.
6. **DRY (Do Not Repeat Yourself)** — But not at the cost of clarity.
7. **YAGNI (You Ain't Gonna Need It)** — Don't implement speculative features.

---

## 2. TypeScript / Frontend Standards

### 2.1 TypeScript Configuration

```json
// tsconfig.json (strict settings)
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "exactOptionalPropertyTypes": true,
    "useUnknownInCatchVariables": true
  }
}
```

### 2.2 React Component Standards

```typescript
// CORRECT: Typed props, explicit return type
interface ClipItemProps {
  clip: Clip;
  isSelected: boolean;
  onSelect: (id: string) => void;
  onDelete: (id: string) => void;
}

export function ClipItem({ clip, isSelected, onSelect, onDelete }: ClipItemProps): JSX.Element {
  const handleClick = useCallback(() => {
    onSelect(clip.id);
  }, [clip.id, onSelect]);
  
  return (
    <div
      className={cn('clip-item', { 'clip-item--selected': isSelected })}
      onClick={handleClick}
      data-testid={`clip-${clip.id}`}
    >
      {clip.name}
    </div>
  );
}

// WRONG: Implicit any, missing types
export default function ClipItem(props) {
  return <div onClick={() => props.onSelect(props.clip.id)}>{props.clip.name}</div>
}
```

### 2.3 Custom Hooks Standards

```typescript
// Custom hooks must start with 'use'
// Must return typed object (not array) when >2 values
export function useClipSelection(initialId?: string) {
  const [selectedIds, setSelectedIds] = useState<Set<string>>(
    initialId ? new Set([initialId]) : new Set()
  );
  
  const select = useCallback((id: string) => {
    setSelectedIds(prev => new Set([...prev, id]));
  }, []);
  
  const deselect = useCallback((id: string) => {
    setSelectedIds(prev => {
      const next = new Set(prev);
      next.delete(id);
      return next;
    });
  }, []);
  
  const isSelected = useCallback((id: string) => selectedIds.has(id), [selectedIds]);
  
  return { selectedIds, select, deselect, isSelected };
}
```

### 2.4 State Management Standards (Zustand)

```typescript
// Stores must use Immer for complex mutations
// Each store in separate file
// Selectors always memoized with shallow comparison

interface TimelineState {
  tracks: Track[];
  clips: Record<string, Clip>;
  playheadMs: number;
  zoomLevel: number;
}

interface TimelineActions {
  addClip: (trackId: string, clip: Clip) => void;
  removeClip: (clipId: string) => void;
  setPlayhead: (timeMs: number) => void;
}

export const useTimelineStore = create<TimelineState & TimelineActions>()(
  devtools(
    immer((set) => ({
      tracks: [],
      clips: {},
      playheadMs: 0,
      zoomLevel: 1,
      
      addClip: (trackId, clip) => set((state) => {
        state.clips[clip.id] = clip;
        const track = state.tracks.find(t => t.id === trackId);
        track?.clipIds.push(clip.id);
      }),
      
      removeClip: (clipId) => set((state) => {
        delete state.clips[clipId];
        state.tracks.forEach(track => {
          track.clipIds = track.clipIds.filter(id => id !== clipId);
        });
      }),
      
      setPlayhead: (timeMs) => set({ playheadMs: timeMs }),
    })),
    { name: 'TimelineStore' }
  )
);
```

### 2.5 File Length Limits
- React components: max 300 lines
- Custom hooks: max 150 lines
- Store files: max 200 lines
- Utility files: max 100 lines per function

### 2.6 Import Order (enforced by ESLint)
```typescript
// 1. Node built-ins
import path from 'path';

// 2. External packages
import React, { useState, useCallback } from 'react';
import { create } from 'zustand';

// 3. Internal absolute imports
import { Clip } from '@/types/timeline';
import { useTimelineStore } from '@/stores/timeline-store';

// 4. Relative imports
import { ClipThumbnail } from './ClipThumbnail';
import styles from './ClipItem.module.css';
```

---

## 3. Python / Backend Standards

### 3.1 Python Code Style

- **Formatter:** Black (line length: 100)
- **Linter:** Ruff with all rules enabled
- **Type checker:** mypy (strict mode)

```python
# CORRECT: Type hints, docstrings, proper error handling
from pathlib import Path
from typing import AsyncIterator

async def process_video_frames(
    video_path: Path,
    *,
    start_frame: int = 0,
    end_frame: int | None = None,
) -> AsyncIterator[VideoFrame]:
    """
    Process video frames from the given path.
    
    Args:
        video_path: Absolute path to the video file.
        start_frame: First frame to process (inclusive).
        end_frame: Last frame to process (exclusive). None means until end.
    
    Yields:
        VideoFrame objects with decoded pixel data.
    
    Raises:
        MediaNotFoundError: If video_path does not exist.
        CodecError: If video cannot be decoded.
    """
    if not video_path.exists():
        raise MediaNotFoundError(f"Video not found: {video_path}")
    
    # Implementation...

# WRONG: No types, no docstring, bare except
def process_video(path):
    try:
        # do something
        pass
    except:
        pass
```

### 3.2 FastAPI Endpoint Standards

```python
# CORRECT: Full Pydantic models, typed returns, documented
from fastapi import APIRouter, Depends, HTTPException, status
from pydantic import BaseModel, Field

router = APIRouter(prefix="/projects", tags=["projects"])

class CreateProjectRequest(BaseModel):
    name: str = Field(..., min_length=1, max_length=255, example="My Project")
    resolution: Resolution = Resolution.HD_1080p
    frame_rate: float = Field(24.0, ge=1.0, le=120.0)
    
class ProjectResponse(BaseModel):
    id: str
    name: str
    created_at: datetime
    
@router.post(
    "/",
    response_model=APIResponse[ProjectResponse],
    status_code=status.HTTP_201_CREATED,
    summary="Create a new project",
    description="Creates a new video editing project with the given settings.",
)
async def create_project(
    request: CreateProjectRequest,
    service: ProjectService = Depends(get_project_service),
) -> APIResponse[ProjectResponse]:
    project = await service.create(request)
    return APIResponse.success(ProjectResponse.from_domain(project))
```

### 3.3 Service Layer Standards

```python
# Services must be:
# 1. Async (async def)
# 2. Stateless (no mutable instance state)
# 3. Testable (injected dependencies)
# 4. Single responsibility

class ProjectService:
    def __init__(
        self,
        project_repo: ProjectRepository,
        media_service: MediaService,
        event_bus: EventBus,
    ) -> None:
        self._project_repo = project_repo
        self._media_service = media_service
        self._event_bus = event_bus
    
    async def create(self, request: CreateProjectRequest) -> Project:
        project = Project(
            id=str(uuid4()),
            name=request.name,
            settings=ProjectSettings(
                resolution=request.resolution,
                frame_rate=request.frame_rate,
            ),
            created_at=datetime.utcnow(),
        )
        await self._project_repo.save(project)
        await self._event_bus.publish(ProjectCreated(project_id=project.id))
        return project
```

### 3.4 Error Handling Standards

```python
# CORRECT: Specific exceptions with context
class AIVideoEditorError(Exception):
    """Base exception for all app errors."""

class MediaNotFoundError(AIVideoEditorError):
    """Raised when media file cannot be found."""

class RenderError(AIVideoEditorError):
    """Raised when rendering fails."""
    def __init__(self, message: str, exit_code: int | None = None):
        super().__init__(message)
        self.exit_code = exit_code

# WRONG: Generic exceptions
raise Exception("something went wrong")
raise ValueError("bad")
```

---

## 4. Git Standards

### 4.1 Commit Message Format (Conventional Commits)
```
<type>(<scope>): <description>

[optional body]

[optional footer]
```

**Types:**
| Type | When |
|------|------|
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `style` | Formatting, no logic change |
| `refactor` | Code restructuring |
| `test` | Adding tests |
| `chore` | Build, CI, tooling |
| `perf` | Performance improvement |
| `revert` | Reverting a commit |

**Examples:**
```
feat(timeline): add ripple edit tool
fix(rendering): correct frame timestamp calculation for variable FPS
docs(api): add OpenAPI descriptions for all endpoints
perf(ai-engine): cache model weights between tasks
```

### 4.2 Branch Strategy (GitHub Flow)
```
main                    -- Production-ready code, protected
  |
  +-- feat/timeline-keyframes      -- Feature branches
  +-- fix/audio-sync-bug          -- Fix branches
  +-- chore/upgrade-pytorch-2.4   -- Chore branches
```

- All code goes through Pull Request
- 1 approver required for merge
- CI must pass before merge
- Squash and merge (clean history)

---

## 5. Code Review Standards

### Must-Review Checklist
- [ ] Types correct and complete
- [ ] Error cases handled
- [ ] Tests written/updated
- [ ] No N+1 queries (backend)
- [ ] No blocking calls in async context
- [ ] Security: no user input passed to subprocess without sanitization
- [ ] Performance: no unnecessary re-renders (frontend)
- [ ] Logging added for important operations
- [ ] No secrets/keys in code
