# 13 — API Architecture

> **Document:** API Architecture v1.0
> **Project:** AI Video Editor
> **Date:** 2026-08-01

---

## 1. API Design Principles

1. **RESTful** — Resource-based URLs, correct HTTP methods
2. **Versioned** — All endpoints under `/api/v1/`
3. **Consistent** — Standard response envelope across all endpoints
4. **Documented** — Auto-generated OpenAPI spec via FastAPI
5. **Validated** — All inputs validated via Pydantic v2
6. **Typed** — Full TypeScript types generated from OpenAPI spec

---

## 2. Base URL & Versioning

```
HTTP Base:      http://127.0.0.1:8755/api/v1
WebSocket:      ws://127.0.0.1:8756/ws
Health Check:   http://127.0.0.1:8755/health
OpenAPI Docs:   http://127.0.0.1:8755/docs
```

---

## 3. Standard Response Envelope

```typescript
// Shared response type (TypeScript)
interface APIResponse<T> {
  success: boolean;
  data: T | null;
  error: APIError | null;
  meta: ResponseMeta;
}

interface APIError {
  code: string;     // Machine-readable: 'MEDIA_NOT_FOUND'
  message: string;  // Human-readable
  details: Record<string, unknown>;
}

interface ResponseMeta {
  timestamp: string;    // ISO 8601
  version: string;      // API version
  requestId: string;    // UUID for tracing
}

// Paginated list response
interface PaginatedResponse<T> extends APIResponse<T[]> {
  pagination: {
    total: number;
    page: number;
    pageSize: number;
    hasNext: boolean;
    nextToken: string | null;
  };
}
```

---

## 4. Complete API Endpoint Reference

### 4.1 Health & System

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/health` | Health check (no envelope) |
| GET | `/api/v1/system/info` | App version, GPU info, disk space |
| GET | `/api/v1/system/capabilities` | Available encoders, AI models |

```json
// GET /health
{ "status": "healthy", "version": "1.0.0", "uptime_seconds": 3600 }
```

### 4.2 Projects

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/projects` | List all projects |
| POST | `/api/v1/projects` | Create project |
| GET | `/api/v1/projects/{id}` | Get project |
| PUT | `/api/v1/projects/{id}` | Update project settings |
| DELETE | `/api/v1/projects/{id}` | Delete project |
| POST | `/api/v1/projects/{id}/duplicate` | Duplicate project |
| POST | `/api/v1/projects/{id}/export-xml` | Export to FCP XML |
| POST | `/api/v1/projects/{id}/import-xml` | Import from FCP XML |

```json
// POST /api/v1/projects
// Request:
{
  "name": "My Documentary",
  "resolution": "3840x2160",
  "frameRate": 24.0,
  "colorSpace": "rec709",
  "sampleRate": 48000
}

// Response 201:
{
  "success": true,
  "data": {
    "id": "proj_abc123",
    "name": "My Documentary",
    "createdAt": "2026-08-01T00:00:00Z",
    "settings": { "resolution": "3840x2160", "frameRate": 24.0 }
  }
}
```

### 4.3 Media

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/projects/{id}/media` | List media items |
| POST | `/api/v1/projects/{id}/media` | Import media |
| GET | `/api/v1/projects/{id}/media/{mediaId}` | Get media item |
| DELETE | `/api/v1/projects/{id}/media/{mediaId}` | Remove media |
| POST | `/api/v1/projects/{id}/media/{mediaId}/proxy` | Generate proxy |
| GET | `/api/v1/projects/{id}/media/{mediaId}/thumbnail` | Get thumbnail |
| GET | `/api/v1/projects/{id}/media/{mediaId}/waveform` | Get waveform data |
| POST | `/api/v1/projects/{id}/media/bins` | Create media bin |
| DELETE | `/api/v1/projects/{id}/media/bins/{binId}` | Delete media bin |

```json
// POST /api/v1/projects/{id}/media
// Request (multipart/form-data OR JSON with paths):
{
  "paths": ["/Users/user/Desktop/interview.mp4", "/Users/user/Desktop/broll.mp4"]
}

// Response 202 (accepted, processing async):
{
  "success": true,
  "data": {
    "importJobId": "import_xyz",
    "totalFiles": 2,
    "status": "processing"
  }
}
```

### 4.4 Timeline

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/projects/{id}/timeline` | Get full timeline |
| POST | `/api/v1/projects/{id}/timeline/tracks` | Add track |
| DELETE | `/api/v1/projects/{id}/timeline/tracks/{trackId}` | Remove track |
| PUT | `/api/v1/projects/{id}/timeline/tracks/{trackId}` | Update track |
| POST | `/api/v1/projects/{id}/timeline/tracks/{trackId}/clips` | Add clip |
| PUT | `/api/v1/projects/{id}/timeline/tracks/{trackId}/clips/{clipId}` | Update clip |
| DELETE | `/api/v1/projects/{id}/timeline/tracks/{trackId}/clips/{clipId}` | Remove clip |
| POST | `/api/v1/projects/{id}/timeline/clips/{clipId}/split` | Split clip |
| POST | `/api/v1/projects/{id}/timeline/clips/{clipId}/keyframes` | Add keyframe |
| PUT | `/api/v1/projects/{id}/timeline/clips/{clipId}/keyframes/{kfId}` | Update keyframe |
| DELETE | `/api/v1/projects/{id}/timeline/clips/{clipId}/keyframes/{kfId}` | Delete keyframe |

```json
// POST /api/v1/projects/{id}/timeline/tracks/{trackId}/clips
// Request:
{
  "mediaId": "media_abc",
  "positionMs": 5000,
  "inPointMs": 0,
  "outPointMs": 10000
}

// Response 201:
{
  "success": true,
  "data": {
    "id": "clip_xyz",
    "trackId": "track_abc",
    "mediaId": "media_abc",
    "positionMs": 5000,
    "durationMs": 10000
  }
}
```

### 4.5 Rendering

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/projects/{id}/render/frame` | Render single frame |
| POST | `/api/v1/projects/{id}/render/range` | Render frame range |
| GET | `/api/v1/projects/{id}/render/jobs` | List render jobs |
| DELETE | `/api/v1/projects/{id}/render/jobs/{jobId}` | Cancel render |

```json
// POST /api/v1/projects/{id}/render/frame
// Request:
{ "timeMs": 5000, "quality": "full", "width": 1920, "height": 1080 }

// Response 200 (frame as base64 or binary):
{
  "success": true,
  "data": {
    "frameData": "base64encodedJPEG...",
    "timeMs": 5000,
    "width": 1920,
    "height": 1080
  }
}
```

### 4.6 Color Grading

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/projects/{id}/clips/{clipId}/color` | Get color grade |
| PUT | `/api/v1/projects/{id}/clips/{clipId}/color` | Update color grade |
| DELETE | `/api/v1/projects/{id}/clips/{clipId}/color` | Reset to default |
| POST | `/api/v1/projects/{id}/clips/{clipId}/color/match` | Color match from reference |
| GET | `/api/v1/projects/{id}/luts` | List installed LUTs |
| POST | `/api/v1/projects/{id}/luts` | Import LUT |

### 4.7 Effects

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/effects` | List all available effects |
| GET | `/api/v1/effects/{effectType}` | Get effect definition |
| GET | `/api/v1/projects/{id}/clips/{clipId}/effects` | Get clip effects |
| POST | `/api/v1/projects/{id}/clips/{clipId}/effects` | Add effect |
| PUT | `/api/v1/projects/{id}/clips/{clipId}/effects/{effectId}` | Update effect params |
| DELETE | `/api/v1/projects/{id}/clips/{clipId}/effects/{effectId}` | Remove effect |

### 4.8 AI Tasks

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/ai/models` | List available models |
| POST | `/api/v1/ai/models/download` | Download model |
| GET | `/api/v1/ai/queue` | View task queue |
| POST | `/api/v1/ai/tasks` | Submit AI task |
| GET | `/api/v1/ai/tasks/{taskId}` | Get task status |
| DELETE | `/api/v1/ai/tasks/{taskId}` | Cancel task |
| GET | `/api/v1/ai/tasks/{taskId}/result` | Get task result |

```json
// POST /api/v1/ai/tasks
// Request:
{
  "type": "transcription",
  "mediaId": "media_abc123",
  "parameters": {
    "language": "en",
    "modelSize": "medium",
    "wordTimestamps": true
  },
  "priority": 8
}

// Response 202:
{
  "success": true,
  "data": {
    "id": "task_xyz789",
    "type": "transcription",
    "status": "queued",
    "progress": 0.0,
    "estimatedMinutes": 3
  }
}
```

### 4.9 Export

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/export/presets` | List export presets |
| POST | `/api/v1/projects/{id}/export` | Create export job |
| GET | `/api/v1/projects/{id}/export/jobs` | List export jobs |
| GET | `/api/v1/projects/{id}/export/jobs/{jobId}` | Get job status |
| DELETE | `/api/v1/projects/{id}/export/jobs/{jobId}` | Cancel export |

### 4.10 Plugins

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/plugins` | List installed plugins |
| POST | `/api/v1/plugins/install` | Install plugin |
| DELETE | `/api/v1/plugins/{pluginId}` | Uninstall plugin |
| PUT | `/api/v1/plugins/{pluginId}/toggle` | Enable/disable plugin |
| GET | `/api/v1/marketplace/plugins` | Browse marketplace |

---

## 5. WebSocket Events

```
Channel: ws://127.0.0.1:8756/ws

Server -> Client events:
  render.frame_ready     { frameId, data }
  render.job.progress    { jobId, progress, fps, eta }
  ai.task.progress       { taskId, progress, stage }
  ai.task.complete       { taskId, resultPath }
  ai.task.error          { taskId, error }
  export.progress        { jobId, progress, fps, eta, framesDone, totalFrames }
  export.complete        { jobId, outputPath, fileSizeBytes }
  media.import.progress  { importJobId, progress, currentFile }
  media.import.complete  { importJobId, mediaIds[] }
  proxy.ready            { mediaId, proxyPath }
  project.auto_saved     { projectId, timestamp }
  backend.status         { cpu, ram, gpu, vram }
```

---

## 6. Error Codes

| Code | HTTP | Description |
|------|------|-------------|
| `PROJECT_NOT_FOUND` | 404 | Project ID does not exist |
| `MEDIA_NOT_FOUND` | 404 | Media ID does not exist |
| `CLIP_NOT_FOUND` | 404 | Clip ID does not exist |
| `VALIDATION_ERROR` | 422 | Pydantic validation failed |
| `MEDIA_FILE_MISSING` | 409 | Media file not at expected path |
| `ENCODER_NOT_AVAILABLE` | 409 | Requested encoder not supported |
| `AI_MODEL_NOT_LOADED` | 503 | Model needs to download first |
| `RENDER_FAILED` | 500 | FFmpeg returned non-zero exit |
| `TASK_ALREADY_CANCELLED` | 409 | Task was already cancelled |
| `PLUGIN_NOT_FOUND` | 404 | Plugin ID not installed |
| `PLUGIN_PERMISSION_DENIED` | 403 | Plugin lacks required permission |

---

## 7. TypeScript API Client Generation

The TypeScript API client is auto-generated from the FastAPI OpenAPI spec:

```bash
# Generate client types
npx openapi-typescript http://127.0.0.1:8755/openapi.json \
  --output src/types/api-generated.ts

# Or use openapi-fetch for type-safe fetch calls
npx openapi-fetch generate
```

This ensures the frontend types always stay in sync with the backend API.
