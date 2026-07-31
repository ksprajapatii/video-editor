# 18 — Dependency Graph

> **Document:** Dependency Graph v1.0
> **Project:** AI Video Editor
> **Date:** 2026-08-01

---

## 1. Top-Level Package Dependencies

```mermaid
graph TD
    subgraph Frontend["Frontend (Node.js / npm)"]
        Electron["electron@30"]
        React["react@18"]
        ReactDOM["react-dom@18"]
        TS["typescript@5.5"]
        Vite["vite@5"]
        EVite["electron-vite@2"]
        Zustand["zustand@4"]
        TanStack["@tanstack/react-query@5"]
        Axios["axios@1"]
        Konva["konva@9 + react-konva@18"]
        Three["three@0.165"]
        Framer["framer-motion@11"]
        Immer["immer@10"]
        TW["tailwindcss@4"]
        Vitest["vitest@2"]
        Playwright["@playwright/test@1"]
        MSW["msw@2"]
        RTL["@testing-library/react@16"]
        EBuilder["electron-builder@24"]
        EUpdater["electron-updater@6"]
    end

    subgraph Backend["Backend (Python / pip)"]
        FastAPI["fastapi>=0.111"]
        Uvicorn["uvicorn>=0.30"]
        Pydantic["pydantic>=2.7"]
        SA["sqlalchemy>=2.0"]
        Alembic["alembic>=1.13"]
        Aiofiles["aiofiles>=23"]
        PyTorch["torch>=2.3 + torchvision"]
        OpenCV["opencv-python>=4.9"]
        FfmpegPy["ffmpeg-python>=0.2"]
        FasterWhisper["faster-whisper>=1.0"]
        SAM2["segment-anything-2"]
        Ultralytics["ultralytics>=8.2"]
        ESRGAN["realesrgan>=0.3"]
        SceneDetect["scenedetect>=0.6"]
        PyAnnote["pyannote.audio>=3.3"]
        DeepFilterNet["deepfilternet>=0.5"]
        Pytest["pytest>=8 + pytest-asyncio"]
        HTTPX["httpx>=0.27"]
        Ruff["ruff>=0.4"]
        Black["black>=24"]
        MyPy["mypy>=1.10"]
    end

    EVite --> Vite
    EVite --> Electron
    EUpdater --> Electron
    EBuilder --> Electron
    ReactDOM --> React
    Konva --> React
    Zustand --> Immer
    TanStack --> Axios
    SA --> Alembic
    Pydantic --> FastAPI
    Uvicorn --> FastAPI
    PyTorch --> OpenCV
    PyTorch --> SAM2
    PyTorch --> ESRGAN
    PyTorch --> FasterWhisper
    PyTorch --> Ultralytics
    PyAnnote --> PyTorch
    DeepFilterNet --> PyTorch
```

---

## 2. Frontend Internal Module Dependencies

```mermaid
graph TD
    subgraph App["app/"]
        AppRoot["App.tsx"]
        AppRouter["Router.tsx"]
    end

    subgraph Features["features/"]
        Timeline["timeline/"]
        MediaBin["media-bin/"]
        Preview["preview/"]
        Color["color/"]
        Audio["audio/"]
        Effects["effects/"]
        AITools["ai-tools/"]
        Export["export/"]
        Plugins["plugins/"]
    end

    subgraph Shared["shared/"]
        Stores["stores/"]
        Services["services/"]
        Hooks["hooks/"]
        Types["types/"]
        Utils["utils/"]
        Components["components/"]
    end

    AppRoot --> AppRouter
    AppRouter --> Features

    Timeline --> Stores
    Timeline --> Services
    Timeline --> Components
    Timeline --> Types

    MediaBin --> Stores
    MediaBin --> Services
    MediaBin --> Components

    Preview --> Stores
    Preview --> Hooks
    Preview --> Types

    Color --> Stores
    Color --> Services

    AITools --> Stores
    AITools --> Services

    Export --> Stores
    Export --> Services

    Stores --> Types
    Services --> Types
    Hooks --> Stores
    Components --> Types
    Utils --> Types
```

---

## 3. Backend Internal Module Dependencies

```mermaid
graph TD
    subgraph API["api/v1/"]
        ProjRouter["projects router"]
        MediaRouter["media router"]
        TLRouter["timeline router"]
        RenderRouter["render router"]
        AIRouter["ai router"]
        ExportRouter["export router"]
        PluginRouter["plugins router"]
        WSRouter["websocket router"]
    end

    subgraph Services["services/"]
        ProjSvc["project_service"]
        MediaSvc["media_service"]
        TLSvc["timeline_service"]
        RenderSvc["render_service"]
        AISvc["ai_service"]
        ExportSvc["export_service"]
        PluginSvc["plugin_service"]
        EventBus["event_bus"]
    end

    subgraph Repos["repositories/"]
        ProjRepo["project_repo"]
        MediaRepo["media_repo"]
        TLRepo["timeline_repo"]
        AIRepo["ai_task_repo"]
    end

    subgraph Infra["infrastructure/"]
        FFmpegAdapter["ffmpeg/adapter"]
        AIEngine["ai/engine"]
        DBSession["db/session"]
        Storage["storage/local"]
    end

    subgraph Domain["domain/"]
        Models["models/"]
        Events["events/"]
    end

    ProjRouter --> ProjSvc
    MediaRouter --> MediaSvc
    TLRouter --> TLSvc
    RenderRouter --> RenderSvc
    AIRouter --> AISvc
    ExportRouter --> ExportSvc
    PluginRouter --> PluginSvc

    ProjSvc --> ProjRepo
    ProjSvc --> EventBus
    MediaSvc --> MediaRepo
    MediaSvc --> FFmpegAdapter
    MediaSvc --> Storage
    TLSvc --> TLRepo
    TLSvc --> EventBus
    RenderSvc --> FFmpegAdapter
    AISvc --> AIEngine
    AISvc --> AIRepo
    AISvc --> EventBus
    ExportSvc --> FFmpegAdapter
    ExportSvc --> EventBus

    ProjRepo --> DBSession
    MediaRepo --> DBSession
    TLRepo --> DBSession
    AIRepo --> DBSession

    ProjRepo --> Models
    MediaRepo --> Models
    TLRepo --> Models
    AIRepo --> Models

    EventBus --> Events
```

---

## 4. Critical Dependency Chain — Video Rendering

```
User seeks to frame 1234
    |
    v
Frontend: PlaybackStore.seek(timeMs)
    |
    v
Preview Component: requests frame
    |
    v
FrameCache: miss
    |
    v
WebSocket: send "decode-frame" request
    |
    v
Backend: WebSocket handler
    |
    v
RenderService.render_frame(timeMs)
    |
    v
TimelineRepo: get_clips_at_time(timeMs)  -> SQLAlchemy -> SQLite
    |
    v
FFmpegAdapter.decode_frame(mediaPath, timeMs)  -> ffmpeg binary -> disk
    |
    v
Buffer returned as bytes
    |
    v
WebSocket: send binary frame
    |
    v
Frontend: receive ArrayBuffer
    |
    v
FrameCache: store frame
    |
    v
WebGL: uploadTexture(buffer)
    |
    v
Shader pipeline: color grade + effects
    |
    v
Canvas: drawFrame()
    |
    v
User sees frame (target: < 100ms)
```

---

## 5. AI Dependency Chain — Transcription

```
User requests transcription
    |
    v
Frontend: POST /api/v1/ai/tasks
    |
    v
Backend: AIRouter -> AIService.submit_task()
    |
    v
AITaskQueue.enqueue(task, priority=8)
    |
    v
AIWorkerPool.dequeue()
    |
    v
ModelRegistry.get("whisper-medium")
    |
    +-- Not loaded: ModelDownloader.ensure_model()
    |       |
    |       v
    |   Download from CDN -> verify SHA256 -> store in ~/.aivideoedit/models/
    |       |
    |       v
    |   WhisperModel.load() -> torch.load() -> CUDA/CPU
    |
    +-- Loaded: return cached model
    |
    v
MediaService.extract_audio(mediaId) -> FFmpegAdapter -> temp.wav
    |
    v
WhisperModel.transcribe(audio_path)
    |
    v
CTranslate2 inference (CPU/CUDA)
    |
    v
Segments with word timestamps
    |
    v
SubtitleExporter.to_srt(segments) -> .srt file
SubtitleExporter.to_json(segments) -> .json metadata
    |
    v
AITaskRepo.save_result(taskId, result)  -> SQLite
    |
    v
EventBus.publish(AITaskCompleted) -> WebSocket broadcast
    |
    v
Frontend: render subtitles on timeline
```

---

## 6. External Dependency Versions Summary

### Frontend Package Versions
| Package | Version | License |
|---------|---------|---------|
| electron | ^30.0.0 | MIT |
| react | ^18.3.0 | MIT |
| typescript | ^5.5.0 | Apache-2.0 |
| vite | ^5.3.0 | MIT |
| zustand | ^4.5.0 | MIT |
| @tanstack/react-query | ^5.50.0 | MIT |
| framer-motion | ^11.2.0 | MIT |
| konva | ^9.3.0 | MIT |
| three | ^0.165.0 | MIT |
| tailwindcss | ^4.0.0 | MIT |
| electron-builder | ^24.13.0 | MIT |

### Backend Package Versions
| Package | Version | License |
|---------|---------|---------|
| fastapi | >=0.111.0 | MIT |
| torch | >=2.3.0 | BSD |
| opencv-python | >=4.9.0 | Apache-2.0 |
| faster-whisper | >=1.0.0 | MIT |
| ultralytics | >=8.2.0 | AGPL-3.0 |
| sqlalchemy | >=2.0.30 | MIT |
| pydantic | >=2.7.0 | MIT |
| alembic | >=1.13.0 | MIT |
| scenedetect | >=0.6.4 | BSD |

> Note: `ultralytics` uses AGPL-3.0. If commercial distribution requires different licensing,
> consider YOLO alternatives or purchasing a commercial license from Ultralytics.
