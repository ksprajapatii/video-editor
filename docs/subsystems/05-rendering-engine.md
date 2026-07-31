# 05 — Rendering Engine

> **Document:** Rendering Engine v1.0
> **Project:** AI Video Editor
> **Date:** 2026-08-01

---

## 1. Overview

The Rendering Engine is responsible for producing video frames — both for **real-time preview** and **final export**. It uses a dual-path architecture:

- **Preview Path:** WebGL-based compositor in the Electron renderer process
- **Export Path:** FFmpeg-based pipeline in the Python backend process

---

## 2. Architecture

```
+--------------------------------------------------------+
|              RENDERING ENGINE ARCHITECTURE              |
|                                                         |
|  [Timeline State]                                       |
|       |                                                 |
|       v                                                 |
|  [Frame Composer]                                       |
|       |                                                 |
|  +----+----+                                           |
|  |         |                                           |
|  v         v                                           |
| [Preview  [Export                                      |
|  Path]    Path]                                        |
|  |         |                                           |
|  v         v                                           |
| [WebGL    [FFmpeg                                      |
|  GPU]     Pipeline]                                    |
|  |         |                                           |
|  v         v                                           |
| [Screen   [Output                                      |
|  Display] File]                                        |
+--------------------------------------------------------+
```

---

## 3. Preview Rendering Path

### 3.1 Frame Compositor (Frontend)

The preview renderer runs in the Electron Renderer process using WebGL.

```
Timeline Tracks
    |
    v
[Track Compositor]
    |
    |-- Video Track 1 --> [Frame Sampler] --> [Effect Chain] --|
    |-- Video Track 2 --> [Frame Sampler] --> [Effect Chain] --+--> [Blend Compositor] --> [Output Frame]
    |-- Text Track    --> [Text Renderer] ----------------------|
    |
    v
[Color Grading Pass (LUT/Wheels)]
    |
    v
[Output Frame --> Canvas/WebGL]
```

### 3.2 Frame Sampler
- Decodes video frames at the current playhead position
- Uses pre-decoded frame cache (LRU)
- For proxy mode: uses lower-resolution proxy file
- Communicates with backend via WebSocket to request decoded frames when cache misses

### 3.3 WebGL Effect Pipeline

```glsl
// Simplified GLSL effect chain
uniform sampler2D inputTexture;
uniform mat4 colorMatrix;   // Color grading
uniform float brightness;
uniform float contrast;
uniform float saturation;

void main() {
    vec4 color = texture2D(inputTexture, vTexCoord);
    // Apply effects in sequence
    color = applyColorGrade(color, colorMatrix);
    color = applyBrightness(color, brightness);
    color = applyContrast(color, contrast);
    color = applySaturation(color, saturation);
    gl_FragColor = color;
}
```

### 3.4 Frame Cache
- **Type:** LRU Cache
- **Size:** Configurable (default: 256 MB)
- **Key:** (mediaId, frameNumber)
- **Thread safety:** SharedArrayBuffer with Atomics (between Worker and main thread)

### 3.5 Playback Loop

```
requestAnimationFrame callback
    |
    v
Calculate target frame number from playhead time
    |
    v
Check frame cache
    |-- HIT: render cached frame immediately
    |-- MISS: request frame from backend decoder
              |
              v
         Backend decodes frame (FFmpeg)
              |
              v
         Frame sent to frontend (ArrayBuffer over WebSocket)
              |
              v
         Upload to WebGL texture
              |
              v
         Store in cache
              |
              v
         Render to canvas
```

---

## 4. Export Rendering Path

### 4.1 Export Pipeline (Backend)

```
Export Job Created
    |
    v
[Export Planner]
    - Resolve clip order from timeline
    - Calculate total frames
    - Select encoder (NVENC / QuickSync / AMF / CPU)
    |
    v
[FFmpeg Command Builder]
    - Build complex filtergraph
    - Apply color grade (via ffmpeg color filters)
    - Apply effects (via ffmpeg vf filters)
    |
    v
[FFmpeg Process]
    - stdin: frame data if needed
    - stdout: encoded output
    - stderr: progress parsing
    |
    v
[Progress Reporter]  ---> WebSocket ---> Frontend
    |
    v
[Output File]
```

### 4.2 FFmpeg Filtergraph Generation

For each clip, generate an FFmpeg filtergraph node:

```python
def build_filtergraph(timeline: Timeline) -> str:
    nodes = []
    # Input labels
    for i, clip in enumerate(timeline.clips):
        nodes.append(f"[{i}:v]")
    
    # Apply effects per clip
    for i, clip in enumerate(timeline.clips):
        filter_chain = build_effect_chain(clip)
        nodes.append(f"[{i}:v]{filter_chain}[v{i}]")
    
    # Concat all clips
    concat_inputs = "".join(f"[v{i}]" for i in range(len(timeline.clips)))
    nodes.append(f"{concat_inputs}concat=n={len(timeline.clips)}:v=1:a=0[vout]")
    
    return ";".join(nodes)
```

### 4.3 Hardware Encoder Selection

```python
class EncoderSelector:
    PRIORITY_ORDER = ["nvenc", "amf", "qsv", "videotoolbox", "libx264"]
    
    def select_encoder(self, codec: str) -> str:
        for encoder in self.PRIORITY_ORDER:
            if self._test_encoder(encoder, codec):
                return encoder
        return self._get_software_fallback(codec)
    
    def _test_encoder(self, encoder: str, codec: str) -> bool:
        # Try a 1-frame encode to test availability
        result = subprocess.run(
            ["ffmpeg", "-f", "lavfi", "-i", "color=black:size=64x64",
             "-frames:v", "1", "-c:v", f"h264_{encoder}", "-f", "null", "-"],
            capture_output=True, timeout=5
        )
        return result.returncode == 0
```

---

## 5. Multi-Pass Rendering

For complex effects and AI features, multi-pass rendering is used:

```
Pass 1: Raw frame decode (FFmpeg)
    |
Pass 2: AI processing (background removal, upscaling, etc.)
    |
Pass 3: Effect chain (color grade, transitions)
    |
Pass 4: Composite (blend layers)
    |
Pass 5: Final encode (FFmpeg)
```

---

## 6. GPU Memory Management

- Maximum GPU memory usage: 2 GB (configurable)
- Texture pool with pre-allocated slots
- Automatic fallback to system RAM if GPU VRAM exhausted
- PyTorch model loading uses `torch.cuda.empty_cache()` between jobs

---

## 7. Render Quality Settings

| Quality | Resolution | GPU Usage | Use Case |
|---------|----------|----------|---------|
| Draft | 1/4 source | Low | Fast scrubbing |
| Preview | 1/2 source | Medium | Editing |
| Full | Source | High | Final review |
| Export | Source + renders | Maximum | Final output |

---

## 8. Rendering Performance Targets

| Target | Metric |
|--------|--------|
| 1080p preview | 30 FPS minimum |
| 4K preview | 24 FPS minimum (proxy mode) |
| 1080p export | 2x realtime (GPU) |
| 4K export | 1x realtime (GPU) |
| AI upscale 1080p→4K | 5 FPS minimum |

---

## 9. Render Job Queue

```python
class RenderJobQueue:
    """Priority queue for render jobs."""
    
    PRIORITY_PREVIEW = 10    # Highest - blocking UI
    PRIORITY_THUMBNAIL = 5   # Medium - background
    PRIORITY_EXPORT = 1      # Lowest - batch
    
    def enqueue(self, job: RenderJob, priority: int) -> str:
        job_id = uuid4()
        heapq.heappush(self._queue, (-priority, job_id, job))
        return str(job_id)
    
    async def process(self):
        while True:
            if self._queue:
                _, job_id, job = heapq.heappop(self._queue)
                await self._execute_job(job_id, job)
            await asyncio.sleep(0.01)
```
