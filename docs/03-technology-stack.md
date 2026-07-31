# 03 — Technology Stack

> **Document:** Tech Stack v1.0
> **Project:** AI Video Editor
> **Date:** 2026-08-01

---

## 1. Overview

The technology stack is chosen for:
1. **Performance** — GPU acceleration, native binaries
2. **AI/ML capability** — Best-in-class Python ecosystem
3. **Developer productivity** — TypeScript safety, hot reload
4. **Cross-platform** — Single codebase, all OS
5. **Community & longevity** — Widely supported, open source

---

## 2. Frontend Stack

### 2.1 Electron 30+
- **Role:** Desktop shell, OS integration, window management
- **Why:** Only mature cross-platform desktop framework with:
  - Full native OS API access
  - Built-in auto-updater
  - Code signing support
  - Best-in-class Chromium rendering
- **Alternatives rejected:** Tauri (less mature ecosystem, no PyTorch), NW.js (smaller community)

### 2.2 React 18+
- **Role:** UI component library
- **Why:**
  - Concurrent rendering for smooth 60fps UI with heavy computation
  - Largest ecosystem (timeline libs, virtualization)
  - `useTransition` / `useDeferredValue` for non-blocking timeline updates
  - React Compiler (RC) for automatic memoization
- **Alternatives rejected:** Vue 3 (smaller component ecosystem for pro tools), Svelte (less mature for complex apps)

### 2.3 TypeScript 5.5+
- **Role:** Type-safe JavaScript superset
- **Why:**
  - Catches entire classes of timeline state bugs at compile time
  - Excellent IDE support (IntelliSense, refactoring)
  - Mandatory for production codebase of this complexity

### 2.4 Zustand 4+
- **Role:** Client-side state management
- **Why:**
  - Minimal boilerplate vs Redux
  - Fine-grained subscriptions (no unnecessary re-renders)
  - Works seamlessly with Immer for immutable updates
  - Middleware support (persist, devtools)
- **Alternatives rejected:** Redux Toolkit (verbose), Jotai (less structured for complex state), MobX (harder to debug)

### 2.5 React Query (TanStack Query) 5+
- **Role:** Server state management, caching, synchronization
- **Why:**
  - Automatic background refetching
  - Optimistic updates
  - Request deduplication
  - Separates server state from UI state

### 2.6 Konva.js + React-Konva
- **Role:** 2D canvas rendering for timeline
- **Why:**
  - Hardware-accelerated canvas (not DOM)
  - Handles thousands of timeline clips efficiently
  - Mouse/touch events on canvas elements
- **Alternatives rejected:** SVG (too slow at scale), pure DOM (impossible performance)

### 2.7 WebGL / Three.js
- **Role:** Real-time video preview rendering, effects preview
- **Why:**
  - GPU-accelerated frame rendering
  - Shader-based effect preview
  - 60fps preview without blocking main thread

### 2.8 Vite 5+
- **Role:** Frontend build tool and dev server
- **Why:**
  - Fastest HMR (Hot Module Replacement) in the ecosystem
  - Native ES module support
  - Optimized production builds via Rollup
  - electron-vite plugin for Electron integration

### 2.9 Tailwind CSS 4
- **Role:** Utility-first CSS framework
- **Why:**
  - Consistent design system
  - Minimal CSS bundle size (purge unused)
  - Dark mode utilities built-in

### 2.10 Framer Motion 11+
- **Role:** Animation library
- **Why:**
  - Declarative animations
  - Gesture support
  - Layout animations for panel resizing

---

## 3. Backend Stack

### 3.1 Python 3.12+
- **Role:** Primary backend language
- **Why:**
  - #1 language for AI/ML (PyTorch, OpenCV, Whisper, etc.)
  - Excellent FFmpeg bindings
  - Fast enough when using native extensions
  - UV package manager for fast dependency resolution

### 3.2 FastAPI 0.111+
- **Role:** HTTP API framework
- **Why:**
  - Automatic OpenAPI documentation
  - Native async support (asyncio)
  - Pydantic v2 integration (fast validation)
  - WebSocket support built-in
  - Fastest Python web framework (Starlette-based)
- **Alternatives rejected:** Flask (sync, slower), Django (too heavy), Tornado (less ergonomic)

### 3.3 Uvicorn + Gunicorn
- **Role:** ASGI server
- **Why:**
  - Production-grade ASGI server
  - Multi-worker support
  - Handles WebSocket natively

### 3.4 FFmpeg 6+ (binary)
- **Role:** Video/audio encode, decode, transcode, filter
- **Why:**
  - Industry standard, nothing comparable
  - Supports virtually every codec and format
  - Hardware acceleration (NVENC, QuickSync, AMF)
  - Used by YouTube, Netflix, VLC, etc.
- **Python bindings:** ffmpeg-python + direct subprocess for complex pipelines

### 3.5 OpenCV 4.9+
- **Role:** Computer vision, frame processing
- **Why:**
  - Hardware-optimized image operations
  - Scene detection, motion analysis
  - Used alongside PyTorch for pre/post-processing

### 3.6 PyTorch 2.3+
- **Role:** Deep learning inference
- **Why:**
  - Industry standard for AI research and production
  - CUDA/MPS/CPU support
  - TorchScript / ONNX export for optimized inference
  - Supports all our AI models
- **Alternatives rejected:** TensorFlow (less Pythonic, heavier), JAX (less mature for production)

### 3.7 Whisper (OpenAI) via faster-whisper
- **Role:** Speech-to-text, auto-transcription
- **Why:**
  - State-of-the-art accuracy
  - faster-whisper uses CTranslate2 for 4x speed
  - Speaker diarization with pyannote.audio

### 3.8 Segment Anything Model (SAM2)
- **Role:** Object/background segmentation
- **Why:**
  - Meta's state-of-the-art segmentation
  - Real-time video segmentation in SAM2
  - Enables "remove background", "track object"

### 3.9 YOLO v10 (Ultralytics)
- **Role:** Object detection and tracking
- **Why:**
  - Real-time inference speed
  - Pre-trained on 80+ classes
  - Ultralytics API is clean and production-ready

### 3.10 Real-ESRGAN
- **Role:** AI video upscaling
- **Why:**
  - Best open-source upscaling model
  - 2x, 4x scaling factors
  - Video-specialized model (realesrgan-x4plus-anime)

### 3.11 SQLite 3.45+ (via SQLAlchemy 2.0)
- **Role:** Local project database
- **Why:**
  - Serverless (no daemon process)
  - File-based (portable projects)
  - WAL mode for concurrent reads
  - ACID compliant
  - SQLAlchemy async for non-blocking queries
- **Alternatives rejected:** PostgreSQL (overkill, requires server), IndexedDB (frontend only)

### 3.12 Pydantic v2
- **Role:** Data validation and serialization
- **Why:**
  - Rust-core, extremely fast
  - Type-safe request/response models
  - JSON schema generation

### 3.13 Celery + Redis (optional, heavy projects)
- **Role:** Background task queue for long AI jobs
- **Why:**
  - Decouples long-running AI from API thread
  - Priority queues for real-time vs batch AI
- **Note:** For single-user desktop, asyncio background tasks are primary; Celery is optional for future multi-user.

---

## 4. DevOps & Tooling Stack

### 4.1 electron-builder
- **Role:** Package and distribute Electron app
- **Why:** Best-in-class auto-update, code signing, multi-platform

### 4.2 GitHub Actions
- **Role:** CI/CD pipelines
- **Why:** Native integration with GitHub, free for public repos, matrix builds

### 4.3 Vitest
- **Role:** Frontend unit/integration testing
- **Why:** Vite-native, fast, Jest-compatible API

### 4.4 Playwright
- **Role:** End-to-end testing
- **Why:** Cross-browser, Electron support, fast parallel execution

### 4.5 Pytest + pytest-asyncio
- **Role:** Backend testing
- **Why:** Industry standard, excellent fixture system, async support

### 4.6 ESLint + Prettier
- **Role:** Frontend linting and formatting
- **Why:** Industry standard, TypeScript support

### 4.7 Ruff + Black + mypy
- **Role:** Python linting, formatting, type checking
- **Why:** Ruff is 100x faster than flake8, Black is opinionated (no config needed)

### 4.8 Husky + lint-staged
- **Role:** Git pre-commit hooks
- **Why:** Enforce standards before commit reaches CI

---

## 5. Stack Decision Matrix

| Concern | Chosen | Score (1-10) | Runner-up |
|---------|--------|-------------|-----------|
| Desktop Shell | Electron | 9 | Tauri |
| UI Framework | React 18 | 9 | Vue 3 |
| State | Zustand | 9 | Redux Toolkit |
| Backend Language | Python | 10 | Rust |
| API Framework | FastAPI | 10 | Flask |
| Video Processing | FFmpeg | 10 | GStreamer |
| ML Framework | PyTorch | 10 | TensorFlow |
| Database | SQLite | 9 | PostgreSQL |
| Build Tool | Vite | 9 | Webpack |
| Testing (FE) | Vitest | 8 | Jest |
| Testing (E2E) | Playwright | 9 | Cypress |

---

## 6. Version Lock Policy

- All production dependencies are pinned to exact versions in `package-lock.json` and `requirements.txt`
- Security patches applied via `npm audit fix` and `pip-audit`
- Major version upgrades require architecture review
