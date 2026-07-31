# 17 — Mermaid Diagrams

> **Document:** Mermaid Diagrams v1.0
> **Project:** AI Video Editor
> **Date:** 2026-08-01

---

## 1. System Architecture Overview

```mermaid
graph TB
    subgraph Electron["Electron Desktop App"]
        direction TB
        subgraph Renderer["Renderer Process (Chromium)"]
            React["React 18 + TypeScript"]
            Zustand["Zustand State"]
            WebGL["WebGL Preview"]
            React --> Zustand
            React --> WebGL
        end
        subgraph Main["Main Process (Node.js)"]
            IPC["IPC Bridge"]
            BM["Backend Manager"]
            AU["Auto Updater"]
        end
        Renderer <-->|"contextBridge IPC"| Main
    end

    subgraph Backend["Python Backend Process"]
        direction TB
        FastAPI["FastAPI + Uvicorn"]
        Services["Services Layer"]
        subgraph Engines["Processing Engines"]
            FFmpeg["FFmpeg Adapter"]
            AI["AI Engine (PyTorch)"]
            DB["SQLite (SQLAlchemy)"]
        end
        FastAPI --> Services
        Services --> Engines
    end

    Main <-->|"HTTP REST + WebSocket localhost"| Backend

    subgraph External["External Services (Optional)"]
        CDN["Model CDN"]
        YT["YouTube API"]
        VM["Vimeo API"]
    end

    Backend -->|"Model download"| CDN
    Backend -->|"Upload"| YT
    Backend -->|"Upload"| VM
```

---

## 2. Frontend Module Architecture

```mermaid
graph LR
    subgraph App["React Application"]
        Router["App Router"]
        
        subgraph Features["Feature Modules"]
            TL["timeline/"]
            MB["media-bin/"]
            PV["preview/"]
            CG["color/"]
            AU["audio/"]
            EF["effects/"]
            AI["ai-tools/"]
            EX["export/"]
            PL["plugins/"]
            ST["settings/"]
        end
        
        subgraph Shared["Shared Layer"]
            Stores["stores/ (Zustand)"]
            Services["services/ (API)"]
            Hooks["hooks/"]
            Types["types/"]
            Utils["utils/"]
            Components["components/"]
        end
    end

    Router --> Features
    Features --> Shared
    Services -->|"HTTP + WS"| Backend["FastAPI Backend"]
```

---

## 3. Timeline Editing Data Flow

```mermaid
sequenceDiagram
    participant User
    participant Timeline as Timeline Component
    participant Store as Zustand Store
    participant API as API Service
    participant Backend as FastAPI

    User->>Timeline: Drag clip to position
    Timeline->>Store: addClip(trackId, clip, positionMs)
    Store->>Store: Optimistic update (immediate)
    Store->>Timeline: Re-render with new state
    Timeline-->>User: UI updated instantly

    Store->>API: POST /timeline/tracks/{id}/clips
    API->>Backend: HTTP Request
    Backend->>Backend: Validate + persist to DB
    Backend->>API: 201 Created {clipId}
    API->>Store: Confirm clip ID
    Store->>Store: Update clip with server ID

    alt Validation fails
        Backend->>API: 422 Error
        API->>Store: rollbackClip()
        Store->>Timeline: Re-render (remove failed clip)
        Timeline-->>User: Show error toast
    end
```

---

## 4. AI Task Pipeline

```mermaid
flowchart TD
    A([User Requests AI Task]) --> B[POST /api/v1/ai/tasks]
    B --> C{Model Available?}
    C -->|No| D[Download Model]
    D --> E[Model Downloaded]
    E --> F
    C -->|Yes| F[Enqueue Task]
    F --> G{GPU Available?}
    G -->|Yes| H[GPU Inference]
    G -->|No| I[CPU Inference]
    H --> J[Process Frames]
    I --> J
    J --> K{Progress Update}
    K --> L[WS: task.progress]
    L --> M[Frontend Progress Bar]
    K --> J
    J --> N[Task Complete]
    N --> O[Save Result to Disk]
    O --> P[WS: task.complete]
    P --> Q([Frontend Shows Result])

    style A fill:#4A90E2,color:#fff
    style Q fill:#27AE60,color:#fff
    style D fill:#E67E22,color:#fff
```

---

## 5. Export Pipeline

```mermaid
flowchart LR
    A([Create Export Job]) --> B[Select Encoder]
    
    subgraph EncoderSelect["Encoder Selection"]
        B --> C{NVENC?}
        C -->|Yes| G[NVENC h264/h265]
        C -->|No| D{QuickSync?}
        D -->|Yes| H[QSV h264/h265]
        D -->|No| E{AMD AMF?}
        E -->|Yes| I[AMF h264/h265]
        E -->|No| J[libx264/libx265 CPU]
    end

    G --> K[Build FFmpeg Command]
    H --> K
    I --> K
    J --> K
    
    subgraph FFmpegPipeline["FFmpeg Pipeline"]
        K --> L[Apply Timeline Order]
        L --> M[Apply Color Grade]
        M --> N[Apply Effects]
        N --> O[Concat Clips]
        O --> P[Mux Audio]
        P --> Q[Encode Output]
    end
    
    Q --> R[Output File]
    R --> S([Export Complete])

    style A fill:#4A90E2,color:#fff
    style S fill:#27AE60,color:#fff
```

---

## 6. Plugin System Architecture

```mermaid
graph TB
    subgraph Marketplace["Plugin Marketplace (Cloud)"]
        API["REST API"]
        Registry["Plugin Registry"]
    end

    subgraph App["Application"]
        subgraph PluginManager["Plugin Manager"]
            Loader["Plugin Loader"]
            Validator["Manifest Validator"]
            Signer["Signature Verifier"]
        end

        subgraph Sandboxes["Execution Sandboxes"]
            FESandbox["Frontend Worker Sandbox"]
            BESandbox["Backend Python Sandbox"]
        end

        subgraph Hooks["Extension Points"]
            EffectHook["Effect Chain Hook"]
            AIHook["AI Task Hook"]
            ImportHook["Import Hook"]
            ExportHook["Export Hook"]
            UISlot["UI Panel Slot"]
        end
    end

    Registry -->|"plugin.json + download"| Loader
    Loader --> Validator
    Validator --> Signer
    Signer --> FESandbox
    Signer --> BESandbox
    FESandbox --> UISlot
    FESandbox --> EffectHook
    BESandbox --> AIHook
    BESandbox --> ImportHook
    BESandbox --> ExportHook
```

---

## 7. Database Entity Relationship

```mermaid
erDiagram
    PROJECTS {
        text id PK
        text name
        datetime created_at
        datetime updated_at
        text settings_json
    }

    MEDIA_ITEMS {
        text id PK
        text project_id FK
        text name
        text file_path
        text type
        integer duration_ms
        text codec
    }

    TIMELINE_TRACKS {
        text id PK
        text project_id FK
        text type
        text name
        integer position
        boolean is_locked
        boolean is_muted
    }

    TIMELINE_CLIPS {
        text id PK
        text track_id FK
        text media_id FK
        integer position_ms
        integer duration_ms
        real speed_factor
    }

    EFFECTS {
        text id PK
        text clip_id FK
        text effect_type
        text params_json
        boolean is_enabled
    }

    KEYFRAMES {
        text id PK
        text clip_id FK
        text parameter
        integer time_ms
        text value
        text easing
    }

    COLOR_GRADES {
        text id PK
        text clip_id FK
        real lift_r
        real gamma_r
        real gain_r
        text curve_master
        text lut_path
    }

    AI_TASKS {
        text id PK
        text project_id FK
        text media_id FK
        text type
        text status
        real progress
        text result_json
    }

    EXPORT_JOBS {
        text id PK
        text project_id FK
        text status
        text format
        text codec
        real progress
    }

    PROJECTS ||--o{ MEDIA_ITEMS : "has"
    PROJECTS ||--o{ TIMELINE_TRACKS : "has"
    PROJECTS ||--o{ AI_TASKS : "has"
    PROJECTS ||--o{ EXPORT_JOBS : "has"
    TIMELINE_TRACKS ||--o{ TIMELINE_CLIPS : "contains"
    TIMELINE_CLIPS ||--o{ EFFECTS : "has"
    TIMELINE_CLIPS ||--o{ KEYFRAMES : "has"
    TIMELINE_CLIPS ||--o| COLOR_GRADES : "has"
    MEDIA_ITEMS ||--o{ TIMELINE_CLIPS : "used in"
```

---

## 8. CI/CD Pipeline Flow

```mermaid
flowchart TD
    A([Developer Push]) --> B{Branch?}
    B -->|"feature/*"| C[PR Checks]
    B -->|"main"| D[Main Pipeline]
    B -->|"v*.*.*"| E[Release Pipeline]

    subgraph PR["PR Checks (Every PR)"]
        C --> C1[Lint TypeScript]
        C --> C2[Lint Python]
        C1 --> C3[Unit Tests FE]
        C2 --> C4[Unit Tests BE]
        C3 --> C5[Integration Tests]
        C4 --> C5
        C5 --> C6[Build Check]
    end

    subgraph Main["Main Pipeline"]
        D --> D1[All PR Checks]
        D1 --> D2[E2E Tests]
        D2 --> D3[Build Win/Mac/Linux]
        D3 --> D4[Upload Dev Builds]
    end

    subgraph Release["Release Pipeline"]
        E --> E1[All Tests]
        E1 --> E2[Sign Win Build]
        E1 --> E3[Sign + Notarize Mac]
        E1 --> E4[Sign Linux Build]
        E2 --> E5[GitHub Release]
        E3 --> E5
        E4 --> E5
        E5 --> E6[Update CDN]
        E6 --> E7([Notify Users])
    end

    style A fill:#4A90E2,color:#fff
    style E7 fill:#27AE60,color:#fff
```

---

## 9. Rendering Engine Flow

```mermaid
flowchart TB
    A([Playhead Moves]) --> B[Frame Compositor]
    
    subgraph Compositor["Frame Compositor"]
        B --> C{Cache Hit?}
        C -->|Yes| D[Cached Frame]
        C -->|No| E[Request from Backend]
        E --> F[FFmpeg Decode]
        F --> G[Return Pixel Buffer]
        G --> H[Store in LRU Cache]
        H --> D
    end
    
    D --> I[Upload to WebGL Texture]
    
    subgraph ShaderPipeline["WebGL Shader Pipeline"]
        I --> J[Color Grade Pass]
        J --> K[Effects Pass]
        K --> L[Composite Layers]
        L --> M[Output to Canvas]
    end
    
    M --> N([Display to User])

    style A fill:#4A90E2,color:#fff
    style N fill:#27AE60,color:#fff
```
