# 04 — Subsystems Overview

> **Document:** Subsystems v1.0
> **Project:** AI Video Editor
> **Date:** 2026-08-01

---

## 1. Subsystem Catalog

The AI Video Editor is composed of **12 major subsystems**:

| # | Subsystem | Layer | Primary Technology |
|---|-----------|-------|-------------------|
| 1 | Project Manager | Full-stack | SQLite + React |
| 2 | Timeline Engine | Frontend | React + Konva |
| 3 | Media Management | Full-stack | FFmpeg + IndexedDB |
| 4 | Playback Engine | Frontend | WebGL + Web Audio |
| 5 | Rendering Engine | Backend | FFmpeg + NVENC |
| 6 | AI Engine | Backend | PyTorch + CUDA |
| 7 | Color Engine | Full-stack | GLSL shaders + OpenCV |
| 8 | Audio Engine | Full-stack | Web Audio API + FFmpeg |
| 9 | Effects Engine | Full-stack | WebGL + FFmpeg |
| 10 | Export Engine | Backend | FFmpeg |
| 11 | Plugin System | Full-stack | Worker threads + Python |
| 12 | IPC / Comms Layer | Infrastructure | Electron IPC + FastAPI |

---

## 2. Subsystem Relationships

```
+------------------------------------------------------------------+
|                     SUBSYSTEM MAP                                 |
|                                                                   |
|  User Input                                                       |
|      |                                                            |
|      v                                                            |
|  [Timeline Engine] <--> [Project Manager]                         |
|      |         |                                                  |
|      |         +---> [Playback Engine] <--> [Audio Engine]        |
|      |                     |                                      |
|      |              [Rendering Engine]                            |
|      |                     |                                      |
|      +---> [Media Manager] |                                      |
|      |         |           v                                      |
|      |         |    [Effects Engine]                              |
|      |         |           |                                      |
|      +---> [Color Engine]  |                                      |
|      |                     |                                      |
|      +---> [AI Engine] ----+                                      |
|      |         |                                                  |
|      |         v                                                  |
|      |    [Export Engine]                                         |
|      |                                                            |
|      +---> [Plugin System]                                        |
|                                                                   |
|  All subsystems communicate via [IPC / Comms Layer]              |
+------------------------------------------------------------------+
```

---

## 3. Subsystem Descriptions

### 3.1 Project Manager
**Responsibility:** Project lifecycle (create, open, save, version, export metadata)

**Components:**
- Project file format (.avep — JSON-based)
- SQLite project database
- Auto-save manager (30s debounce)
- Undo/Redo command stack (Command pattern)
- Project migration system (schema versioning)

**Key Interfaces:**
```
createProject(settings) -> Project
openProject(path) -> Project
saveProject(project) -> void
exportProjectXML(project) -> string (FCP XML)
getUndoHistory() -> Command[]
```

### 3.2 Timeline Engine
**Responsibility:** Non-linear editing canvas, clip management, track management

**Components:**
- Track model (Video, Audio, Text, Effect tracks)
- Clip model with in/out points
- Editing tools (blade, ripple, roll, slip, slide)
- Snapping engine
- Keyframe system
- Multi-cam synchronizer

**Key Interfaces:**
```
addClip(trackId, clip, position) -> Clip
removeClip(clipId) -> void
splitClip(clipId, position) -> [Clip, Clip]
rippleDelete(clipId) -> void
setKeyframe(clipId, param, time, value) -> Keyframe
```

### 3.3 Media Management
**Responsibility:** Import, organize, proxy generation, thumbnail/waveform

**Components:**
- Media importer (FFmpeg-backed)
- Proxy generator (background thread)
- Thumbnail extractor
- Waveform analyzer
- Media bin with folder tree

**Key Interfaces:**
```
importMedia(paths[]) -> MediaItem[]
generateProxy(mediaId) -> ProxyFile
extractThumbnails(mediaId, count) -> Image[]
analyzeWaveform(mediaId) -> WaveformData
```

### 3.4 Playback Engine
**Responsibility:** Real-time video playback at target FPS

**Components:**
- Frame scheduler (requestAnimationFrame loop)
- WebGL compositor (layers to screen)
- Frame cache (LRU, configurable size)
- Seek engine (keyframe-aware)
- Transport controls

**Key Interfaces:**
```
play() -> void
pause() -> void
seek(timeMs) -> void
setPlaybackRate(rate) -> void
setQuality(quality: '1/1'|'1/2'|'1/4') -> void
```

### 3.5 Rendering Engine
**Responsibility:** Frame rendering for preview and final export

*See dedicated document: 05-rendering-engine.md*

### 3.6 AI Engine
**Responsibility:** All ML inference tasks

*See dedicated document: 06-ai-engine.md*

### 3.7 Color Engine
**Responsibility:** Color grading, LUT management, scopes

**Components:**
- Color wheel (lift/gamma/gain)
- Curves editor (RGB, individual channels)
- HSL qualifier
- LUT loader and applier
- Scope renderer (waveform, vectorscope, histogram)
- Color match algorithm

**Key Interfaces:**
```
applyColorGrade(clipId, gradeParams) -> void
applyLUT(clipId, lutPath) -> void
matchColor(sourceClipId, refClipId) -> ColorGrade
renderScope(frame, scopeType) -> ScopeData
```

### 3.8 Audio Engine
**Responsibility:** Multi-track audio mixing, effects, monitoring

**Components:**
- Web Audio API graph
- Volume/pan automation curves
- Audio effects chain (EQ, Compressor, Reverb, etc.)
- LUFS metering
- Audio normalize processor
- Audio sync (waveform alignment)

**Key Interfaces:**
```
setTrackVolume(trackId, volumeDb) -> void
setTrackPan(trackId, pan) -> void  // -1.0 to 1.0
addAudioEffect(trackId, effectType, params) -> EffectNode
normalizeLUFS(trackId, targetLUFS) -> void
```

### 3.9 Effects Engine
**Responsibility:** Video effects, transitions, motion effects

**Components:**
- Effect registry (built-in + plugin effects)
- WebGL shader pipeline
- Effect parameter keyframe system
- Transition renderer
- Motion graphics compositor

**Key Interfaces:**
```
addEffect(clipId, effectType, params) -> Effect
removeEffect(effectId) -> void
addTransition(clipId1, clipId2, type, duration) -> Transition
previewEffect(clipId, effectId, frame) -> Frame
```

### 3.10 Export Engine
**Responsibility:** Final render and delivery

**Components:**
- Export queue manager
- Preset library
- FFmpeg command builder
- Hardware encoder selector (NVENC/QuickSync/AMF/CPU)
- Progress tracker
- Upload manager (YouTube, Vimeo)

**Key Interfaces:**
```
createExportJob(settings) -> ExportJob
queueExport(job) -> void
getExportProgress(jobId) -> Progress
cancelExport(jobId) -> void
uploadToYouTube(jobId, credentials) -> UploadResult
```

### 3.11 Plugin System
**Responsibility:** Third-party plugin loading, sandboxing, marketplace

*See dedicated document: 07-plugin-architecture.md*

### 3.12 IPC / Communications Layer
**Responsibility:** All communication between processes

*See dedicated document: 08-ipc-communication.md*

---

## 4. Subsystem Initialization Order

```
1. Main Process starts
2. Database initialized (migrations run)
3. Python backend process spawned
4. Backend health-check (retry 5x)
5. IPC bridge registered
6. React app loads
7. Project Manager initializes
8. Subsystems register (lazy load)
9. Last project restored (if configured)
10. UI ready
```

---

## 5. Error Handling Strategy Per Subsystem

| Subsystem | Error Strategy |
|-----------|---------------|
| Project Manager | Show save dialog on crash, write emergency backup |
| Timeline Engine | Rollback to last valid state, log error |
| Media Management | Graceful degradation (show placeholder) |
| Rendering Engine | Fallback to software render, notify user |
| AI Engine | Cancel job, show error toast, continue editing |
| Export Engine | Retry 3x, then show detailed error log |
| Plugin System | Disable offending plugin, sandbox prevents crash |
