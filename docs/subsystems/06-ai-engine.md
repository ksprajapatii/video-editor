# 06 — AI Engine

> **Document:** AI Engine v1.0
> **Project:** AI Video Editor
> **Date:** 2026-08-01

---

## 1. Overview

The AI Engine is a modular subsystem running in the Python backend process. It manages all ML/AI inference tasks — from transcription to background removal — in a non-blocking, GPU-accelerated manner.

---

## 2. Architecture

```
+------------------------------------------------------------------+
|                        AI ENGINE                                  |
|                                                                   |
|  +-----------------------+    +-----------------------------+     |
|  |   AI Task Queue       |    |   Model Registry            |     |
|  |   (AsyncIO queue)     |    |   - Lazy load on demand     |     |
|  +-----------+-----------+    |   - LRU eviction from VRAM  |     |
|              |                +-----------------------------+     |
|              v                                                    |
|  +-----------+--------------------------------------------+      |
|  |              AI Worker Pool (asyncio tasks)            |      |
|  |                                                        |      |
|  | +----------+ +---------+ +----------+ +-----------+   |      |
|  | |Whisper   | |SAM2     | |YOLO v10  | |ESRGAN     |   |      |
|  | |Worker    | |Worker   | |Worker    | |Worker     |   |      |
|  | +----------+ +---------+ +----------+ +-----------+   |      |
|  |                                                        |      |
|  | +----------+ +---------+ +----------+                 |      |
|  | |Scene Det.| |Color    | |Noise Red.|                 |      |
|  | |Worker    | |Match    | |Worker    |                 |      |
|  | +----------+ +---------+ +----------+                 |      |
|  +--------------------------------------------------------+      |
|                                                                   |
|  +-------------------------+                                      |
|  |   GPU Memory Manager    |  (VRAM budget: 2GB default)         |
|  +-------------------------+                                      |
+------------------------------------------------------------------+
```

---

## 3. AI Task System

### 3.1 Task Definition

```python
from pydantic import BaseModel
from enum import Enum
from typing import Any

class AITaskType(str, Enum):
    TRANSCRIPTION = "transcription"
    BACKGROUND_REMOVAL = "background_removal"
    OBJECT_DETECTION = "object_detection"
    OBJECT_TRACKING = "object_tracking"
    UPSCALING = "upscaling"
    SCENE_DETECTION = "scene_detection"
    COLOR_MATCH = "color_match"
    NOISE_REDUCTION = "noise_reduction"
    AUTO_CUT = "auto_cut"
    HIGHLIGHT_REEL = "highlight_reel"
    FACE_DETECTION = "face_detection"

class AITask(BaseModel):
    id: str
    type: AITaskType
    media_id: str
    parameters: dict[str, Any]
    priority: int = 5  # 1-10, 10 is highest
    status: str = "pending"
    progress: float = 0.0
    result: Any = None
    error: str | None = None
```

### 3.2 Task Lifecycle

```
Client submits AITask via POST /api/v1/ai/tasks
    |
    v
Task validated by Pydantic
    |
    v
Task enqueued in AsyncIO PriorityQueue
    |
    v
Worker picks up task (priority order)
    |
    v
Model loaded (or retrieved from cache)
    |
    v
Inference runs (GPU/CPU)
    |-- Progress updates sent via WebSocket every 0.5s
    |
    v
Result stored (DB + file system)
    |
    v
WebSocket event: task_complete { taskId, result }
```

---

## 4. AI Models

### 4.1 Transcription (Whisper / faster-whisper)

```python
class TranscriptionWorker:
    """Transcribes audio using faster-whisper."""
    
    MODEL_SIZES = ["tiny", "base", "small", "medium", "large-v3"]
    
    def __init__(self, model_size: str = "medium"):
        from faster_whisper import WhisperModel
        self.model = WhisperModel(
            model_size,
            device="cuda" if torch.cuda.is_available() else "cpu",
            compute_type="float16"  # 2x speed on GPU
        )
    
    async def transcribe(self, audio_path: str, language: str = None) -> TranscriptionResult:
        segments, info = await asyncio.to_thread(
            self.model.transcribe,
            audio_path,
            language=language,
            word_timestamps=True,
            vad_filter=True  # Remove silence
        )
        return TranscriptionResult(
            segments=[Segment(start=s.start, end=s.end, text=s.text, words=s.words) 
                     for s in segments],
            language=info.language,
            duration=info.duration
        )
```

**Output:** SRT/VTT subtitle file + structured JSON with word-level timestamps

### 4.2 Background Removal (SAM2)

```python
class BackgroundRemovalWorker:
    """Uses SAM2 for video background segmentation."""
    
    def __init__(self):
        from sam2.build_sam import build_sam2_video_predictor
        self.predictor = build_sam2_video_predictor(
            "sam2_hiera_large.pt",
            device="cuda"
        )
    
    async def remove_background(
        self, 
        video_path: str, 
        output_path: str,
        prompt_frame: int = 0,
        prompt_points: list[tuple] = None
    ) -> str:
        # Initialize video predictor
        with torch.inference_mode(), torch.autocast("cuda", dtype=torch.bfloat16):
            state = self.predictor.init_state(video_path)
            # Add prompt (click point or bounding box)
            if prompt_points:
                self.predictor.add_new_points(state, frame_idx=prompt_frame, 
                                              obj_id=1, points=prompt_points)
            # Propagate through video
            for frame_idx, obj_ids, masks in self.predictor.propagate_in_video(state):
                await self._write_masked_frame(frame_idx, masks, output_path)
        return output_path
```

**Output:** Video with alpha channel (RGBA) or green-screen composite

### 4.3 Object Detection & Tracking (YOLO v10 + ByteTrack)

```python
class ObjectTrackingWorker:
    """Real-time object detection and tracking."""
    
    def __init__(self):
        from ultralytics import YOLO
        self.model = YOLO("yolov10n.pt")  # Nano for speed
    
    async def track_objects(
        self, 
        video_path: str,
        classes: list[int] = None,  # None = all classes
        confidence: float = 0.5
    ) -> list[TrackResult]:
        results = []
        async for frame_result in self._stream_track(video_path, classes, confidence):
            results.append(frame_result)
        return results
    
    async def _stream_track(self, video_path, classes, confidence):
        for result in self.model.track(
            source=video_path,
            classes=classes,
            conf=confidence,
            tracker="bytetrack.yaml",
            stream=True
        ):
            yield TrackResult(
                frame=result.orig_img,
                boxes=result.boxes.xyxy.tolist(),
                track_ids=result.boxes.id.tolist() if result.boxes.id else [],
                class_ids=result.boxes.cls.tolist(),
                confidences=result.boxes.conf.tolist()
            )
```

**Output:** JSON with frame-by-frame bounding boxes and track IDs

### 4.4 AI Upscaling (Real-ESRGAN)

```python
class UpscalingWorker:
    """Video upscaling using Real-ESRGAN."""
    
    SCALE_FACTORS = [2, 4]
    
    def __init__(self, scale: int = 4):
        from realesrgan import RealESRGANer
        from basicsr.archs.rrdbnet_arch import RRDBNet
        
        model = RRDBNet(num_in_ch=3, num_out_ch=3, num_feat=64, 
                        num_block=23, num_grow_ch=32, scale=scale)
        self.upsampler = RealESRGANer(
            scale=scale,
            model_path=f"RealESRGAN_x{scale}plus.pth",
            model=model,
            half=True  # FP16 for speed
        )
    
    async def upscale_video(self, input_path: str, output_path: str,
                            progress_cb: callable = None) -> str:
        cap = cv2.VideoCapture(input_path)
        total_frames = int(cap.get(cv2.CAP_PROP_FRAME_COUNT))
        
        for frame_idx in range(total_frames):
            ret, frame = cap.read()
            if not ret:
                break
            output_frame, _ = self.upsampler.enhance(frame, outscale=self.scale)
            await asyncio.to_thread(self._write_frame, output_frame, output_path)
            
            if progress_cb:
                await progress_cb(frame_idx / total_frames)
        
        return output_path
```

### 4.5 Scene Detection (Auto-Cut)

```python
class SceneDetectionWorker:
    """Detects scene boundaries for auto-cutting."""
    
    def __init__(self):
        from scenedetect import detect, ContentDetector, AdaptiveDetector
        self.detector_type = "adaptive"  # More accurate than threshold
    
    async def detect_scenes(self, video_path: str, 
                            threshold: float = 27.0) -> list[SceneBoundary]:
        scenes = await asyncio.to_thread(
            detect, video_path, AdaptiveDetector(adaptive_threshold=threshold)
        )
        return [
            SceneBoundary(
                start_frame=scene[0].get_frames(),
                end_frame=scene[1].get_frames(),
                start_time=scene[0].get_seconds(),
                end_time=scene[1].get_seconds(),
                confidence=scene[2] if len(scene) > 2 else 1.0
            )
            for scene in scenes
        ]
```

### 4.6 Audio Noise Reduction

```python
class NoiseReductionWorker:
    """AI-powered audio noise reduction using DeepFilterNet."""
    
    def __init__(self):
        from df.enhance import enhance, init_df, load_audio, save_audio
        self.model, self.df_state, _ = init_df()
    
    async def reduce_noise(self, audio_path: str, output_path: str,
                           strength: float = 0.8) -> str:
        audio, _ = load_audio(audio_path, sr=self.df_state.sr())
        enhanced = enhance(self.model, self.df_state, audio, atten_lim_db=strength*40)
        save_audio(output_path, enhanced, self.df_state.sr())
        return output_path
```

---

## 5. Model Registry & Memory Management

```python
class ModelRegistry:
    """LRU-based model cache to manage GPU VRAM."""
    
    def __init__(self, max_vram_gb: float = 2.0):
        self._models: OrderedDict[str, AIModel] = OrderedDict()
        self._max_vram = max_vram_gb * 1024 ** 3
        self._current_vram = 0
    
    def get(self, model_key: str) -> AIModel | None:
        if model_key in self._models:
            self._models.move_to_end(model_key)  # LRU update
            return self._models[model_key]
        return None
    
    def register(self, model_key: str, model: AIModel, vram_size: int):
        while self._current_vram + vram_size > self._max_vram and self._models:
            evicted_key, evicted = self._models.popitem(last=False)
            self._current_vram -= evicted.vram_size
            evicted.unload()
        
        self._models[model_key] = model
        self._current_vram += vram_size
```

---

## 6. AI Processing Modes

| Mode | GPU Required | Speed | Quality | Use Case |
|------|-------------|-------|---------|---------|
| Fast | No | Fastest | Lower | Quick preview |
| Balanced | Recommended | Medium | Good | Default |
| Quality | Yes (CUDA) | Slower | Best | Final output |

---

## 7. AI API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/ai/tasks` | POST | Submit AI task |
| `/api/v1/ai/tasks/{id}` | GET | Get task status |
| `/api/v1/ai/tasks/{id}` | DELETE | Cancel task |
| `/api/v1/ai/models` | GET | List available models |
| `/api/v1/ai/models/download` | POST | Download a model |
| `/api/v1/ai/queue` | GET | View task queue |
| `/ws/ai/progress` | WS | Real-time progress |
