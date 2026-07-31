# 20 — Development Roadmap

> **Document:** Development Roadmap v1.0
> **Project:** AI Video Editor
> **Date:** 2026-08-01

---

## 1. Release Strategy

```
v0.1.0  -- Alpha: Core editing (internal only)
v0.2.0  -- Alpha: Media + rendering
v0.3.0  -- Beta: AI features
v0.4.0  -- Beta: Export + color
v0.5.0  -- Beta: Audio + effects
v0.6.0  -- RC: Plugin system
v1.0.0  -- GA: Production release
v1.x.0  -- Post-launch feature releases
```

---

## 2. Phase Overview

| Phase | Duration | Version | Focus |
|-------|---------|---------|-------|
| Phase 1 | 2 weeks | - | Architecture & Documentation |
| Phase 2 | 6 weeks | v0.1 | Project foundation & setup |
| Phase 3 | 8 weeks | v0.2 | Core editing & playback |
| Phase 4 | 8 weeks | v0.3 | AI features |
| Phase 5 | 6 weeks | v0.4 | Color grading & export |
| Phase 6 | 6 weeks | v0.5 | Audio & effects |
| Phase 7 | 6 weeks | v0.6 | Plugin system |
| Phase 8 | 4 weeks | v1.0 | Polish & GA |
| **Total** | **46 weeks** | | ~11 months |

---

## 3. Phase 1 — Architecture & Documentation (CURRENT)
**Duration:** 2 weeks  
**Version:** Pre-development

### Deliverables
- [x] Software Requirements Specification
- [x] Overall Architecture Design
- [x] Technology Stack Selection
- [x] Subsystem Designs (12 subsystems)
- [x] IPC Communication Design
- [x] Rendering Engine Design
- [x] AI Engine Design
- [x] Plugin Architecture Design
- [x] Coding Standards
- [x] Naming Conventions
- [x] Testing Strategy
- [x] Database Schema
- [x] API Architecture
- [x] UML Diagrams
- [x] Mermaid Diagrams
- [x] Dependency Graph
- [x] Project Directory Design
- [x] Deployment Strategy
- [x] CI/CD Strategy
- [x] Development Roadmap

---

## 4. Phase 2 — Foundation Setup
**Duration:** 6 weeks (Weeks 3-8)  
**Target Version:** v0.1.0-alpha

### Sprint 2.1 (Weeks 3-4): Project Scaffolding
**Goal:** Working Electron + React + FastAPI skeleton

| Task | Priority | Complexity |
|------|---------|-----------|
| Initialize Electron + electron-vite project | P0 | Low |
| Setup TypeScript strict config | P0 | Low |
| Setup ESLint + Prettier + Husky | P0 | Low |
| Create preload contextBridge API | P0 | Medium |
| Initialize FastAPI backend structure | P0 | Medium |
| Setup SQLAlchemy + Alembic | P0 | Medium |
| Write initial database migrations | P0 | Medium |
| Setup Zustand stores (skeleton) | P0 | Low |
| Setup Axios client + React Query | P0 | Low |
| Setup Vitest + pytest | P0 | Low |
| Setup GitHub Actions CI | P0 | Medium |
| Backend process spawn from Main | P0 | High |
| Health check + retry logic | P0 | Medium |

### Sprint 2.2 (Weeks 5-6): Core UI Shell
**Goal:** App shell with panel layout

| Task | Priority | Complexity |
|------|---------|-----------|
| App main window layout | P0 | Medium |
| Resizable panel system | P0 | High |
| Keyboard shortcut system | P0 | Medium |
| Menu bar (File, Edit, View, ...) | P0 | Medium |
| Theme system (dark/light) | P1 | Medium |
| Toast notification system | P0 | Low |
| App settings skeleton | P1 | Low |
| First E2E test (app launches) | P0 | Low |

---

## 5. Phase 3 — Core Editing & Playback
**Duration:** 8 weeks (Weeks 9-16)  
**Target Version:** v0.2.0-alpha

### Sprint 3.1 (Weeks 9-10): Media Import
**Goal:** Import and display media files

| Task | Priority | Complexity |
|------|---------|-----------|
| File dialog integration (IPC) | P0 | Low |
| POST /media endpoint (backend) | P0 | Medium |
| ffprobe media analysis | P0 | Medium |
| Media bin UI (grid view) | P0 | High |
| Thumbnail generation (FFmpeg) | P0 | Medium |
| Waveform generation | P1 | Medium |
| Media bin folder system | P1 | High |
| Drag-to-timeline support | P0 | High |

### Sprint 3.2 (Weeks 11-12): Timeline Engine
**Goal:** Multi-track timeline with clips

| Task | Priority | Complexity |
|------|---------|-----------|
| Timeline canvas (Konva) | P0 | High |
| Track rows rendering | P0 | High |
| Clip blocks rendering | P0 | High |
| Timeline ruler | P0 | Medium |
| Zoom in/out | P0 | Medium |
| Horizontal scroll | P0 | Medium |
| Snap engine | P0 | High |
| Blade/split tool | P0 | High |
| Selection (single + multi) | P0 | Medium |
| Drag clip between tracks | P1 | High |
| Ripple edit | P1 | High |

### Sprint 3.3 (Weeks 13-14): Playback Engine
**Goal:** Real-time video playback

| Task | Priority | Complexity |
|------|---------|-----------|
| WebGL preview canvas | P0 | High |
| Frame decode endpoint (backend) | P0 | High |
| Frame cache (LRU) | P0 | High |
| Playback transport controls | P0 | Medium |
| JKL keyboard controls | P0 | Low |
| Seek bar | P0 | Medium |
| Playback quality selector | P1 | Medium |
| Source monitor | P1 | Medium |

### Sprint 3.4 (Weeks 15-16): Project Management
**Goal:** Save, open, undo/redo

| Task | Priority | Complexity |
|------|---------|-----------|
| Create/open/save project (IPC) | P0 | High |
| Auto-save system (30s debounce) | P0 | Medium |
| Undo/redo command stack | P0 | High |
| Recent projects list | P1 | Low |
| Project settings dialog | P1 | Medium |
| Proxy generation (background) | P1 | High |

---

## 6. Phase 4 — AI Features
**Duration:** 8 weeks (Weeks 17-24)  
**Target Version:** v0.3.0-beta

### Sprint 4.1 (Weeks 17-18): AI Infrastructure
| Task | Priority | Complexity |
|------|---------|-----------|
| AI task queue system | P0 | High |
| Model registry + LRU cache | P0 | High |
| Model downloader with progress | P0 | High |
| WebSocket progress streaming | P0 | Medium |
| AI tasks API endpoints | P0 | Medium |
| AI panel UI shell | P0 | Medium |

### Sprint 4.2 (Weeks 19-20): Transcription
| Task | Priority | Complexity |
|------|---------|-----------|
| faster-whisper integration | P0 | Medium |
| Transcription API endpoint | P0 | Low |
| Subtitle export (SRT, VTT) | P0 | Low |
| Auto-subtitle timeline track | P0 | High |
| Subtitle editor UI | P1 | High |
| Speaker diarization (pyannote) | P2 | High |

### Sprint 4.3 (Weeks 21-22): Background Removal & Tracking
| Task | Priority | Complexity |
|------|---------|-----------|
| SAM2 integration | P0 | High |
| Background removal UI | P0 | High |
| Click-to-segment interface | P0 | High |
| YOLO object detection | P1 | Medium |
| Object tracking overlay | P1 | High |
| Face detection + blur | P1 | Medium |

### Sprint 4.4 (Weeks 23-24): Auto-Cut & Upscaling
| Task | Priority | Complexity |
|------|---------|-----------|
| Scene detection (PySceneDetect) | P0 | Medium |
| Auto-cut UI and results | P0 | Medium |
| Real-ESRGAN upscaling | P1 | Medium |
| Audio noise reduction | P1 | Medium |
| Highlight reel generator | P2 | High |

---

## 7. Phase 5 — Color Grading & Export
**Duration:** 6 weeks (Weeks 25-30)  
**Target Version:** v0.4.0-beta

### Sprint 5.1 (Weeks 25-26): Color Grading
| Task | Priority | Complexity |
|------|---------|-----------|
| Color panel UI (wheels/curves) | P0 | High |
| GLSL color grade shader | P0 | High |
| LUT import (CUBE, 3DL) | P0 | Medium |
| HSL secondary color | P1 | High |
| Video scopes (waveform, vector) | P1 | High |
| Color match (AI-assisted) | P2 | High |

### Sprint 5.2 (Weeks 27-28): Export Engine
| Task | Priority | Complexity |
|------|---------|-----------|
| Export dialog UI | P0 | Medium |
| Export presets library | P0 | Medium |
| FFmpeg filtergraph builder | P0 | High |
| Hardware encoder selection | P0 | High |
| Export progress (WebSocket) | P0 | Medium |
| Batch export queue | P1 | Medium |

### Sprint 5.3 (Weeks 29-30): Delivery
| Task | Priority | Complexity |
|------|---------|-----------|
| YouTube OAuth + upload | P1 | High |
| Vimeo OAuth + upload | P2 | Medium |
| Chapter markers in export | P1 | Medium |
| Subtitle burn-in / soft subs | P1 | Medium |
| Export presets editor | P2 | Medium |

---

## 8. Phase 6 — Audio & Effects
**Duration:** 6 weeks (Weeks 31-36)  
**Target Version:** v0.5.0-beta

### Sprint 6.1 (Weeks 31-32): Audio Engine
| Task | Priority | Complexity |
|------|---------|-----------|
| Web Audio API mixer | P0 | High |
| Volume/pan per track | P0 | Medium |
| Audio waveform display | P0 | High |
| Audio effects (EQ, compressor) | P1 | High |
| LUFS metering | P1 | Medium |
| Audio normalization | P1 | Medium |

### Sprint 6.2 (Weeks 33-34): Effects Engine
| Task | Priority | Complexity |
|------|---------|-----------|
| Effects panel + registry | P0 | High |
| 10 core video effects | P0 | High |
| 10 core transitions | P0 | High |
| Keyframe system (UI) | P0 | High |
| Chroma key / green screen | P1 | High |
| Speed ramping tool | P1 | High |

### Sprint 6.3 (Weeks 35-36): Text & Graphics
| Task | Priority | Complexity |
|------|---------|-----------|
| Title/text editor | P0 | High |
| Lower thirds templates | P1 | Medium |
| Animated text | P1 | High |
| Subtitle import (SRT) | P0 | Medium |

---

## 9. Phase 7 — Plugin System
**Duration:** 6 weeks (Weeks 37-42)  
**Target Version:** v0.6.0-RC

### Sprints 7.1-7.3
| Task | Priority | Complexity |
|------|---------|-----------|
| Plugin manifest schema | P0 | Medium |
| Frontend Worker sandbox | P0 | High |
| Backend Python sandbox | P0 | High |
| Plugin registry | P0 | Medium |
| Plugin marketplace UI | P0 | High |
| Plugin install/update/remove | P0 | Medium |
| 3 example plugins (reference) | P0 | Medium |
| Plugin SDK npm package | P1 | Medium |
| Plugin SDK PyPI package | P1 | Medium |

---

## 10. Phase 8 — Polish & GA
**Duration:** 4 weeks (Weeks 43-46)  
**Target Version:** v1.0.0

### Tasks
| Task | Priority | Complexity |
|------|---------|-----------|
| Performance profiling + fixes | P0 | High |
| Accessibility audit (WCAG 2.1) | P0 | Medium |
| Keyboard shortcut customization | P0 | Medium |
| Onboarding tutorial | P1 | High |
| Crash reporting integration | P0 | Medium |
| Auto-updater final testing | P0 | Medium |
| Code signing all platforms | P0 | Medium |
| macOS notarization | P0 | Medium |
| App store submissions | P1 | High |
| Documentation website | P1 | High |
| Marketing site | P1 | High |
| v1.0.0 release | P0 | Low |

---

## 11. Post-v1.0 Roadmap

| Version | Feature |
|---------|---------|
| v1.1.0 | Multi-cam editing |
| v1.2.0 | Cloud project sync |
| v1.3.0 | Collaboration (real-time co-edit) |
| v1.4.0 | AI-powered music/soundtrack |
| v1.5.0 | 360-degree video support |
| v2.0.0 | Node-based effects compositing |

---

## 12. Risk Register

| Risk | Probability | Impact | Mitigation |
|------|-----------|--------|-----------|
| PyTorch VRAM exceeds user GPU | High | Medium | CPU fallback, VRAM settings |
| macOS Notarization rejection | Medium | High | Pre-test on macOS, check entitlements |
| FFmpeg binary size too large | Low | Medium | Bundle without uncommon codecs |
| AI model download fails | Medium | Medium | Retry + cache + manual install |
| Electron security vulnerability | Medium | High | Keep Electron updated, strict CSP |
| SQLite corruption on crash | Low | High | WAL mode + auto-backup |
| Plugin breaks app stability | Medium | High | Worker sandbox + auto-disable |

---

## 13. Definition of Done

A feature is "Done" when:
1. Code is written and passes lint/type checks
2. Unit tests written (coverage maintained)
3. Integration test added for API changes
4. PR reviewed and approved
5. CI pipeline passes
6. Feature manually tested on Windows + macOS + Linux
7. Documentation updated if needed
