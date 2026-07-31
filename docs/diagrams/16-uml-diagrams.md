# 16 — UML Diagrams

> **Document:** UML Diagrams v1.0
> **Project:** AI Video Editor
> **Date:** 2026-08-01

---

## 1. Class Diagram — Domain Model

```
+---------------------+          +------------------------+
|       Project        |          |       MediaItem        |
|---------------------|          |------------------------|
| - id: string        |          | - id: string           |
| - name: string      |          | - projectId: string    |
| - createdAt: Date   |  1    *  | - filePath: string     |
| - updatedAt: Date   |<-------->| - type: MediaType      |
| - settings: Settings|          | - durationMs: number   |
|---------------------|          | - width: number        |
| + save(): void      |          | - height: number       |
| + export(): void    |          | - fps: number          |
| + clone(): Project  |          | - codec: string        |
+---------------------+          | - proxyPath?: string   |
          |                      |------------------------|
          | 1                    | + generateProxy()      |
          |                      | + extractThumbnail()   |
          | *                    +------------------------+
+---------v-----------+
|      Timeline        |          +------------------------+
|---------------------|          |      TimelineTrack     |
| - projectId: string  |  1    * |------------------------|
| - tracks: Track[]   |<-------->| - id: string           |
| - durationMs: number|          | - type: TrackType      |
|---------------------|          | - name: string         |
| + addTrack()        |          | - position: number     |
| + removeTrack()     |          | - isLocked: boolean    |
| + getTotalDuration()|          | - isMuted: boolean     |
+---------------------+          |------------------------|
                                  | + addClip()           |
                                  | + removeClip()        |
                                  +------------------------+
                                           |
                                           | 1..*
                                           |
                                  +--------v---------------+
                                  |      TimelineClip      |
                                  |------------------------|
                                  | - id: string           |
                                  | - trackId: string      |
                                  | - mediaId?: string     |
                                  | - positionMs: number   |
                                  | - durationMs: number   |
                                  | - inPointMs: number    |
                                  | - outPointMs?: number  |
                                  | - speedFactor: number  |
                                  | - opacity: number      |
                                  |------------------------|
                                  | + split(timeMs): Clip[]|
                                  | + addEffect()          |
                                  | + setKeyframe()        |
                                  +------------------------+
                                           |
                              +------------+------------+
                              |                         |
                   +----------v--------+    +-----------v-------+
                   |      Effect        |    |     Keyframe      |
                   |-------------------|    |-------------------|
                   | - id: string      |    | - id: string      |
                   | - clipId: string  |    | - clipId: string  |
                   | - effectType: str |    | - parameter: str  |
                   | - params: object  |    | - timeMs: number  |
                   | - isEnabled: bool |    | - value: unknown  |
                   |-------------------|    | - easing: string  |
                   | + apply(frame)    |    |-------------------|
                   | + preview()       |    | + interpolate()   |
                   +-------------------+    +-------------------+
```

---

## 2. Class Diagram — Service Layer

```
+-------------------------+         +--------------------------+
|    ProjectService       |         |     MediaService         |
|-------------------------|         |--------------------------|
| - repo: ProjectRepo     |         | - repo: MediaRepo        |
| - eventBus: EventBus    |         | - ffmpegAdapter: FFmpeg  |
|-------------------------|         | - storageService: Storage|
| + create(req): Project  |         |--------------------------|
| + findById(id): Project |         | + import(paths): Media[] |
| + update(id, patch)     |         | + generateProxy(id)      |
| + delete(id): void      |         | + extractThumbnail(id)   |
| + exportXML(id): string |         | + analyzeWaveform(id)    |
+-------------------------+         +--------------------------+

+-------------------------+         +--------------------------+
|    TimelineService      |         |     AIService            |
|-------------------------|         |--------------------------|
| - repo: TimelineRepo    |         | - taskQueue: AITaskQueue |
| - renderService: Render |         | - modelRegistry: Registry|
| - eventBus: EventBus    |         | - eventBus: EventBus     |
|-------------------------|         |--------------------------|
| + addClip(trackId, clip)|         | + submitTask(task): str  |
| + removeClip(clipId)    |         | + getTaskStatus(id)      |
| + splitClip(clipId, ms) |         | + cancelTask(id)         |
| + addKeyframe(...)      |         | + listModels(): Model[]  |
| + rippleDelete(clipId)  |         +--------------------------+
+-------------------------+

+-------------------------+         +--------------------------+
|    RenderService        |         |    ExportService         |
|-------------------------|         |--------------------------|
| - ffmpeg: FFmpegAdapter |         | - ffmpeg: FFmpegAdapter  |
| - frameCache: LRUCache  |         | - exportQueue: Queue     |
| - jobQueue: RenderQueue |         | - eventBus: EventBus     |
|-------------------------|         |--------------------------|
| + renderFrame(timeMs)   |         | + createJob(settings)    |
| + renderRange(start,end)|         | + queueExport(jobId)     |
| + cancelJob(jobId)      |         | + getProgress(jobId)     |
+-------------------------+         | + cancelExport(jobId)    |
                                    +--------------------------+
```

---

## 3. Sequence Diagram — Import Media

```
User          Frontend           Main Process        Backend           FileSystem
 |                |                    |                 |                  |
 | Drop file      |                    |                 |                  |
 |--------------->|                    |                 |                  |
 |                | IPC: openFileDialog|                 |                  |
 |                |------------------->|                 |                  |
 |                |                    | Native dialog   |                  |
 |                |                    |<----------------|                  |
 |                |  { paths: [...] }  |                 |                  |
 |                |<-------------------|                 |                  |
 |                |                    |                 |                  |
 |                | POST /media { paths}|                 |                  |
 |                |---------------------------------->   |                  |
 |                |                    |           Validate files           |
 |                |                    |                 |-------->         |
 |                |                    |                 |<--------         |
 |                |                    |         Run ffprobe per file       |
 |                |                    |                 |-------->         |
 |                |                    |                 |<--------         |
 |                |                    |           Save to DB               |
 |                |                    |                 |                  |
 |                | 202 { importJobId }|                 |                  |
 |                |<-----------------------------------|                   |
 |                |                    |                 |                  |
 |                |    [background: generate thumbnails + waveforms]        |
 |                |                    |                 |-------->         |
 |                |                    |                 |<--------         |
 |                |                    |    WS: media.import.progress       |
 |                |<---------------------------------------------|          |
 | Update UI      |                    |                 |                  |
 |<---------------|                    |                 |                  |
 |                |                    |    WS: media.import.complete        |
 |                |<---------------------------------------------|          |
 | Show in bin    |                    |                 |                  |
 |<---------------|                    |                 |                  |
```

---

## 4. Sequence Diagram — AI Transcription

```
User          Frontend           Backend            WhisperWorker       DB
 |                |                 |                     |              |
 | Click          |                 |                     |              |
 | "Transcribe"   |                 |                     |              |
 |--------------->|                 |                     |              |
 |                | POST /ai/tasks  |                     |              |
 |                |---------------->|                     |              |
 |                |                 | Validate + save task|              |
 |                |                 |----------------------------->      |
 |                |  202 {taskId}   |                     |              |
 |                |<----------------|                     |              |
 | Show progress  |                 |                     |              |
 |<---------------|                 |                     |              |
 |                |                 | Dequeue task        |              |
 |                |                 |-------------------->|              |
 |                |                 |       Load Whisper model          |
 |                |                 |                     |              |
 |                |                 |       Run inference  |              |
 |                |                 |  WS progress every 2s             |
 |                |<------------------------------------|  |              |
 | Update progress|                 |                     |              |
 |<---------------|                 |                     |              |
 |                |                 |       Inference done |              |
 |                |                 |       Save result    |              |
 |                |                 |<--------------------|              |
 |                |                 |                     |--save SRT--> |
 |                |                 | WS: ai.task.complete |              |
 |                |<----------------|                     |              |
 | Show subtitles |                 |                     |              |
 |<---------------|                 |                     |              |
```

---

## 5. Component Diagram

```
+============================================================+
|                   ELECTRON APPLICATION                      |
|                                                             |
|  +--------------------+   +-----------------------------+  |
|  | Renderer Process   |   |       Main Process          |  |
|  |                    |   |                             |  |
|  | +----------------+ |   | +-------------------------+ |  |
|  | | React App      | |IPC| | BackendManager          | |  |
|  | | +------------+ | |<->| | - spawn python process  | |  |
|  | | |TimelineUI  | | |   | | - health check          | |  |
|  | | +------------+ | |   | +-------------------------+ |  |
|  | | |PreviewUI   | | |   | +-------------------------+ |  |
|  | | +------------+ | |   | | IPCBridge               | |  |
|  | | |MediaBinUI  | | |   | | - file dialogs          | |  |
|  | | +------------+ | |   | | - OS integration        | |  |
|  | | |ColorUI     | | |   | +-------------------------+ |  |
|  | | +------------+ | |   | +-------------------------+ |  |
|  | | |ExportUI    | | |   | | AutoUpdater             | |  |
|  | | +------------+ | |   | +-------------------------+ |  |
|  | | |AIUI        | | |   |                             |  |
|  | | +------------+ | |   +-----------------------------+  |
|  | +----------------+ |                                    |
|  +--------------------+   +-----------------------------+  |
|                           |    Python Backend Process   |  |
|                           |                             |  |
|                           | +-------------------------+ |  |
|                           | | FastAPI App             | |  |
|                           | | +---------------------+ | |  |
|                           | | | Project Router      | | |  |
|                           | | | Media Router        | | |  |
|                           | | | Timeline Router     | | |  |
|                           | | | Render Router       | | |  |
|                           | | | AI Router           | | |  |
|                           | | | Export Router       | | |  |
|                           | | +---------------------+ | |  |
|                           | | Services Layer         | |  |
|                           | | AI Engine              | |  |
|                           | | FFmpeg Adapter         | |  |
|                           | | SQLite DB              | |  |
|                           | +-------------------------+ |  |
|                           +-----------------------------+  |
+============================================================+
```

---

## 6. State Machine — Clip Lifecycle

```
         +----------+
         |  Created |
         +----+-----+
              |
              v addToTimeline()
         +----+-----+
         |  Active  |<--------+
         +----+-----+         |
              |               | undo()
     split() |   setSpeed()   |
              |   setEffect()  |
              v               |
         +----+-----+         |
         | Modified +-------->+
         +----+-----+
              |
    delete()  |  rippleDelete()
              |
              v
         +----+-----+
         | Deleted  |
         +----------+
```
