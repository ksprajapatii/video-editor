# 11 — Testing Strategy

> **Document:** Testing Strategy v1.0
> **Project:** AI Video Editor
> **Date:** 2026-08-01

---

## 1. Testing Philosophy

- **Test behavior, not implementation** — Tests should survive refactoring
- **Pyramid over ice cream cone** — More unit tests, fewer E2E tests
- **Fast feedback loops** — Unit tests run in <5s, CI in <10min
- **No flaky tests** — Flaky tests are disabled immediately and fixed in next sprint

---

## 2. Testing Pyramid

```
        /\
       /  \         E2E Tests (Playwright)
      /    \        ~20 tests - critical user flows
     /------\
    /        \      Integration Tests
   /          \     ~100 tests - service + API layer
  /------------\
 /              \   Unit Tests
/________________\  ~500+ tests - components, hooks, services, utils

Coverage targets:
  Backend:  >= 80%
  Frontend: >= 70%
```

---

## 3. Frontend Testing

### 3.1 Unit Tests (Vitest)

**Tool:** Vitest + React Testing Library + MSW (Mock Service Worker)

**What to test:**
- React components (behavior, not DOM structure)
- Custom hooks
- Zustand store actions
- Utility functions
- Type guards

**File convention:** `ComponentName.test.tsx` co-located with component

```typescript
// Example: ClipItem.test.tsx
import { render, screen, fireEvent } from '@testing-library/react';
import { describe, it, expect, vi } from 'vitest';
import { ClipItem } from './ClipItem';
import { mockClip } from '@/test-utils/factories';

describe('ClipItem', () => {
  it('renders clip name', () => {
    const clip = mockClip({ name: 'Interview.mp4' });
    render(<ClipItem clip={clip} isSelected={false} onSelect={vi.fn()} onDelete={vi.fn()} />);
    expect(screen.getByText('Interview.mp4')).toBeInTheDocument();
  });
  
  it('calls onSelect with clip id when clicked', () => {
    const onSelect = vi.fn();
    const clip = mockClip({ id: 'clip-123' });
    render(<ClipItem clip={clip} isSelected={false} onSelect={onSelect} onDelete={vi.fn()} />);
    
    fireEvent.click(screen.getByTestId('clip-clip-123'));
    
    expect(onSelect).toHaveBeenCalledOnce();
    expect(onSelect).toHaveBeenCalledWith('clip-123');
  });
  
  it('shows selected state correctly', () => {
    const clip = mockClip();
    const { rerender } = render(<ClipItem clip={clip} isSelected={false} onSelect={vi.fn()} onDelete={vi.fn()} />);
    expect(screen.getByTestId(`clip-${clip.id}`)).not.toHaveClass('clip-item--selected');
    
    rerender(<ClipItem clip={clip} isSelected={true} onSelect={vi.fn()} onDelete={vi.fn()} />);
    expect(screen.getByTestId(`clip-${clip.id}`)).toHaveClass('clip-item--selected');
  });
});
```

### 3.2 Hook Testing

```typescript
// Example: useClipSelection.test.ts
import { renderHook, act } from '@testing-library/react';
import { describe, it, expect } from 'vitest';
import { useClipSelection } from './useClipSelection';

describe('useClipSelection', () => {
  it('starts with empty selection', () => {
    const { result } = renderHook(() => useClipSelection());
    expect(result.current.selectedIds.size).toBe(0);
  });
  
  it('selects a clip', () => {
    const { result } = renderHook(() => useClipSelection());
    act(() => result.current.select('clip-1'));
    expect(result.current.isSelected('clip-1')).toBe(true);
  });
  
  it('deselects a clip', () => {
    const { result } = renderHook(() => useClipSelection('clip-1'));
    act(() => result.current.deselect('clip-1'));
    expect(result.current.isSelected('clip-1')).toBe(false);
  });
});
```

### 3.3 Store Testing

```typescript
// Example: timeline-store.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { useTimelineStore } from './timeline-store';
import { mockClip, mockTrack } from '@/test-utils/factories';

describe('TimelineStore', () => {
  beforeEach(() => {
    useTimelineStore.setState({ tracks: [], clips: {}, playheadMs: 0 });
  });
  
  it('addClip adds clip to state and track', () => {
    const track = mockTrack({ id: 'track-1' });
    useTimelineStore.setState({ tracks: [track] });
    
    const clip = mockClip({ id: 'clip-1' });
    useTimelineStore.getState().addClip('track-1', clip);
    
    const state = useTimelineStore.getState();
    expect(state.clips['clip-1']).toEqual(clip);
    expect(state.tracks[0].clipIds).toContain('clip-1');
  });
});
```

### 3.4 API Mocking (MSW)

```typescript
// test-utils/mocks/handlers.ts
import { http, HttpResponse } from 'msw';

export const handlers = [
  http.get('http://127.0.0.1:8755/api/v1/projects', () => {
    return HttpResponse.json({
      success: true,
      data: [mockProjectResponse()],
    });
  }),
  
  http.post('http://127.0.0.1:8755/api/v1/ai/tasks', () => {
    return HttpResponse.json({
      success: true,
      data: { id: 'task-123', status: 'pending', progress: 0 },
    }, { status: 202 });
  }),
];
```

---

## 4. Backend Testing

### 4.1 Unit Tests (pytest)

**Tool:** pytest + pytest-asyncio + factory_boy

**File convention:** `test_<module_name>.py` in `tests/unit/`

```python
# tests/unit/test_timeline_service.py
import pytest
from unittest.mock import AsyncMock, MagicMock
from backend.services.timeline_service import TimelineService
from tests.factories import ClipFactory, TrackFactory

@pytest.fixture
def timeline_service():
    repo = AsyncMock()
    event_bus = AsyncMock()
    return TimelineService(repo=repo, event_bus=event_bus)

@pytest.mark.asyncio
async def test_add_clip_saves_to_repo(timeline_service):
    clip = ClipFactory()
    track = TrackFactory()
    
    await timeline_service.add_clip(track.id, clip)
    
    timeline_service._repo.save_clip.assert_awaited_once_with(clip)

@pytest.mark.asyncio
async def test_add_clip_publishes_event(timeline_service):
    clip = ClipFactory()
    track = TrackFactory()
    
    await timeline_service.add_clip(track.id, clip)
    
    timeline_service._event_bus.publish.assert_awaited_once()
    event = timeline_service._event_bus.publish.call_args[0][0]
    assert event.clip_id == clip.id
```

### 4.2 Integration Tests (pytest + httpx)

**Tool:** pytest + httpx.AsyncClient + pytest-asyncio

```python
# tests/integration/test_project_api.py
import pytest
from httpx import AsyncClient, ASGITransport
from backend.main import app

@pytest.mark.asyncio
async def test_create_project_returns_201():
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as client:
        response = await client.post("/api/v1/projects", json={
            "name": "Test Project",
            "resolution": "1080p",
            "frame_rate": 24.0
        })
    
    assert response.status_code == 201
    data = response.json()
    assert data["success"] is True
    assert data["data"]["name"] == "Test Project"
    assert "id" in data["data"]

@pytest.mark.asyncio
async def test_create_project_validates_name():
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as client:
        response = await client.post("/api/v1/projects", json={
            "name": "",  # Empty name should fail
            "resolution": "1080p",
            "frame_rate": 24.0
        })
    
    assert response.status_code == 422
```

### 4.3 Test Factories

```python
# tests/factories.py
import factory
from backend.domain.models import Clip, Track, Project

class ClipFactory(factory.Factory):
    class Meta:
        model = Clip
    
    id = factory.LazyFunction(lambda: str(uuid4()))
    name = factory.Sequence(lambda n: f"clip_{n}.mp4")
    start_time_ms = 0
    duration_ms = factory.LazyFunction(lambda: random.randint(1000, 10000))
    media_id = factory.LazyFunction(lambda: str(uuid4()))
```

---

## 5. End-to-End Tests (Playwright)

**Tool:** Playwright with Electron support

### Critical User Flows to Test

| Test | Priority |
|------|---------|
| App launches successfully | P0 |
| Create new project | P0 |
| Import media file | P0 |
| Add clip to timeline | P0 |
| Export project | P0 |
| Undo/redo operations | P1 |
| AI transcription flow | P1 |
| Background removal flow | P1 |
| Plugin installation | P2 |
| Color grading panel | P2 |

```typescript
// e2e/tests/create-project.spec.ts
import { test, expect, _electron as electron } from '@playwright/test';

test('create new project from blank', async () => {
  const app = await electron.launch({ args: ['dist/main.js'] });
  const page = await app.firstWindow();
  
  // Click "New Project"
  await page.getByTestId('btn-new-project').click();
  
  // Fill project settings
  await page.getByTestId('input-project-name').fill('E2E Test Project');
  await page.getByTestId('select-resolution').selectOption('1080p');
  await page.getByTestId('btn-create-project').click();
  
  // Verify timeline is visible
  await expect(page.getByTestId('timeline-container')).toBeVisible();
  await expect(page.getByTestId('window-title')).toContainText('E2E Test Project');
  
  await app.close();
});
```

---

## 6. Performance Tests

```python
# tests/performance/test_timeline_performance.py
import pytest
import time

@pytest.mark.performance
def test_timeline_with_100_clips_loads_under_1s():
    """Timeline with 100 clips should initialize in < 1 second."""
    clips = [ClipFactory() for _ in range(100)]
    
    start = time.perf_counter()
    timeline = Timeline(clips=clips)
    elapsed = time.perf_counter() - start
    
    assert elapsed < 1.0, f"Timeline initialization took {elapsed:.3f}s"
```

---

## 7. Test Configuration

```python
# pytest.ini
[pytest]
asyncio_mode = auto
markers =
    unit: Unit tests (fast)
    integration: Integration tests (medium)
    e2e: End-to-end tests (slow)
    performance: Performance benchmarks
    
testpaths = tests
filterwarnings = error
```

```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    environment: 'jsdom',
    setupFiles: ['./src/test-utils/setup.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html', 'lcov'],
      thresholds: { lines: 70, functions: 70, branches: 60 },
      exclude: ['**/*.test.*', '**/test-utils/**', '**/*.stories.*']
    }
  }
});
```

---

## 8. Test CI Matrix

```yaml
# Runs on every PR:
  - Unit tests (Vitest)          <- 2 min
  - Unit tests (pytest)          <- 2 min
  - Integration tests (httpx)    <- 3 min
  - Lint + type check (both)     <- 2 min
  
# Runs on merge to main:
  - All above +
  - E2E tests (Playwright)       <- 8 min
  - Performance benchmarks       <- 5 min

Total CI time target: < 10 minutes
```
