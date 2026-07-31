# 19 — Project Directory Structure

> **Document:** Project Directory v1.0
> **Project:** AI Video Editor
> **Date:** 2026-08-01

---

## 1. Root Structure

```
AI-Video-Editor/
|
+-- .github/                         # GitHub configuration
|   +-- workflows/
|   |   +-- pr-checks.yml            # PR validation pipeline
|   |   +-- main.yml                 # Main branch pipeline
|   |   +-- release.yml              # Release pipeline
|   +-- ISSUE_TEMPLATE/
|   |   +-- bug_report.md
|   |   +-- feature_request.md
|   +-- PULL_REQUEST_TEMPLATE.md
|   +-- dependabot.yml               # Automated dependency updates
|
+-- docs/                            # Architecture documentation (Phase 1)
|   +-- 00-INDEX.md
|   +-- 01-software-requirements.md
|   +-- 02-overall-architecture.md
|   +-- 03-technology-stack.md
|   +-- 14-deployment-strategy.md
|   +-- 15-cicd-strategy.md
|   +-- 19-project-directory.md
|   +-- subsystems/
|   +-- standards/
|   +-- database/
|   +-- api/
|   +-- diagrams/
|   +-- roadmap/
|
+-- src/                             # Frontend Electron + React source
|   +-- main/                        # Electron Main process
|   +-- preload/                     # Electron Preload scripts
|   +-- renderer/                    # React application
|
+-- backend/                         # Python FastAPI backend
|
+-- resources/                       # Static app resources
|   +-- icons/
|   +-- fonts/
|   +-- effects/
|   +-- luts/
|
+-- scripts/                         # Build and dev scripts
|   +-- build-python-bundle.py
|   +-- download-ffmpeg.py
|   +-- generate-api-types.sh
|   +-- verify-bundle.py
|
+-- tests/
|   +-- e2e/                         # Playwright E2E tests
|
+-- electron-builder.yml             # electron-builder config
+-- package.json
+-- package-lock.json
+-- tsconfig.json                    # Root TypeScript config
+-- tsconfig.main.json               # Main process TS config
+-- tsconfig.preload.json            # Preload TS config
+-- vite.config.ts                   # Vite config (renderer)
+-- electron-vite.config.ts          # electron-vite config
+-- .eslintrc.json
+-- .prettierrc
+-- CHANGELOG.md
+-- LICENSE
+-- README.md
```

---

## 2. Frontend Source (`src/`)

```
src/
|
+-- main/                            # Electron Main Process
|   +-- index.ts                     # Entry point
|   +-- backend-manager.ts           # Python backend lifecycle
|   +-- ipc-handlers.ts              # IPC channel handlers
|   +-- auto-updater.ts              # Update management
|   +-- window-manager.ts            # BrowserWindow management
|   +-- menu-builder.ts              # App menu construction
|   +-- tray-manager.ts              # System tray
|   +-- logger.ts                    # Main process logging
|
+-- preload/
|   +-- index.ts                     # contextBridge API definitions
|   +-- electron-api.types.ts        # TypeScript types for bridge API
|
+-- renderer/                        # React Application
|   +-- index.html                   # HTML entry
|   +-- main.tsx                     # React entry point
|   |
|   +-- app/
|   |   +-- App.tsx                  # Root component
|   |   +-- Router.tsx               # React Router setup
|   |   +-- ErrorBoundary.tsx        # Global error boundary
|   |   +-- QueryProvider.tsx        # React Query setup
|   |   +-- ThemeProvider.tsx        # Theme context
|   |
|   +-- features/
|   |   |
|   |   +-- timeline/
|   |   |   +-- index.ts             # Public API for feature
|   |   |   +-- components/
|   |   |   |   +-- Timeline.tsx             # Main container
|   |   |   |   +-- TimelineRuler.tsx        # Time ruler
|   |   |   |   +-- TimelineTrack.tsx        # Single track row
|   |   |   |   +-- TimelineClip.tsx         # Single clip block
|   |   |   |   +-- TimelinePlayhead.tsx     # Playhead indicator
|   |   |   |   +-- TimelineScrollbar.tsx    # Horizontal scrollbar
|   |   |   |   +-- KeyframeEditor.tsx       # Keyframe panel
|   |   |   |   +-- TrackHeader.tsx          # Track name/controls
|   |   |   |   +-- ToolBar.tsx              # Edit tool selection
|   |   |   +-- hooks/
|   |   |   |   +-- useTimelineDrag.ts
|   |   |   |   +-- useTimelineZoom.ts
|   |   |   |   +-- useSnapping.ts
|   |   |   |   +-- useKeyframeEditor.ts
|   |   |   +-- utils/
|   |   |   |   +-- time-to-pixels.ts
|   |   |   |   +-- snap-engine.ts
|   |   |   |   +-- clip-operations.ts
|   |   |   +-- Timeline.test.tsx
|   |   |
|   |   +-- media-bin/
|   |   |   +-- components/
|   |   |   |   +-- MediaBin.tsx
|   |   |   |   +-- MediaGrid.tsx
|   |   |   |   +-- MediaListItem.tsx
|   |   |   |   +-- MediaFolder.tsx
|   |   |   |   +-- ImportButton.tsx
|   |   |   +-- hooks/
|   |   |   |   +-- useMediaImport.ts
|   |   |   |   +-- useMediaDrag.ts
|   |   |   +-- MediaBin.test.tsx
|   |   |
|   |   +-- preview/
|   |   |   +-- components/
|   |   |   |   +-- VideoPreview.tsx         # Main preview canvas
|   |   |   |   +-- PreviewControls.tsx      # Transport controls
|   |   |   |   +-- QualitySelector.tsx
|   |   |   |   +-- SafeAreaOverlay.tsx
|   |   |   +-- hooks/
|   |   |   |   +-- usePlayback.ts
|   |   |   |   +-- useWebGLRenderer.ts
|   |   |   |   +-- useFrameCache.ts
|   |   |   +-- gl/
|   |   |   |   +-- shaders/
|   |   |   |   |   +-- color-grade.frag.glsl
|   |   |   |   |   +-- effects.frag.glsl
|   |   |   |   |   +-- composite.frag.glsl
|   |   |   |   +-- WebGLRenderer.ts
|   |   |   |   +-- TexturePool.ts
|   |   |
|   |   +-- color/
|   |   |   +-- components/
|   |   |   |   +-- ColorPanel.tsx
|   |   |   |   +-- ColorWheels.tsx
|   |   |   |   +-- CurvesEditor.tsx
|   |   |   |   +-- HSLPanel.tsx
|   |   |   |   +-- LUTManager.tsx
|   |   |   |   +-- Scopes.tsx
|   |   |   +-- hooks/
|   |   |       +-- useColorGrade.ts
|   |   |       +-- useLUTImport.ts
|   |   |
|   |   +-- audio/
|   |   |   +-- components/
|   |   |   |   +-- AudioMixer.tsx
|   |   |   |   +-- AudioTrackChannel.tsx
|   |   |   |   +-- AudioEffectsChain.tsx
|   |   |   |   +-- LUFSMeter.tsx
|   |   |   +-- hooks/
|   |   |       +-- useAudioGraph.ts
|   |   |       +-- useWaveformDisplay.ts
|   |   |
|   |   +-- effects/
|   |   |   +-- components/
|   |   |   |   +-- EffectsPanel.tsx
|   |   |   |   +-- EffectCard.tsx
|   |   |   |   +-- EffectParams.tsx
|   |   |   |   +-- TransitionPicker.tsx
|   |   |   +-- effects-registry.ts
|   |   |
|   |   +-- ai-tools/
|   |   |   +-- components/
|   |   |   |   +-- AIToolsPanel.tsx
|   |   |   |   +-- TranscriptionTool.tsx
|   |   |   |   +-- BackgroundRemovalTool.tsx
|   |   |   |   +-- ObjectTrackingTool.tsx
|   |   |   |   +-- UpscalingTool.tsx
|   |   |   |   +-- SceneDetectionTool.tsx
|   |   |   |   +-- AIProgressCard.tsx
|   |   |   +-- hooks/
|   |   |       +-- useAITask.ts
|   |   |
|   |   +-- export/
|   |   |   +-- components/
|   |   |   |   +-- ExportDialog.tsx
|   |   |   |   +-- ExportPresets.tsx
|   |   |   |   +-- ExportQueue.tsx
|   |   |   |   +-- ExportProgressCard.tsx
|   |   |   +-- export-presets.ts
|   |   |
|   |   +-- plugins/
|   |   |   +-- components/
|   |   |   |   +-- PluginManager.tsx
|   |   |   |   +-- MarketplaceBrowser.tsx
|   |   |   |   +-- InstalledPlugins.tsx
|   |   |   +-- hooks/
|   |   |       +-- usePluginInstall.ts
|   |   |
|   |   +-- settings/
|   |       +-- components/
|   |           +-- SettingsDialog.tsx
|   |           +-- KeyboardShortcuts.tsx
|   |           +-- PerformanceSettings.tsx
|   |           +-- AppearanceSettings.tsx
|   |
|   +-- shared/
|       |
|       +-- components/
|       |   +-- ui/
|       |   |   +-- Button.tsx
|       |   |   +-- Input.tsx
|       |   |   +-- Select.tsx
|       |   |   +-- Slider.tsx
|       |   |   +-- Dialog.tsx
|       |   |   +-- ContextMenu.tsx
|       |   |   +-- Tooltip.tsx
|       |   |   +-- Toast.tsx
|       |   |   +-- Progress.tsx
|       |   |   +-- Spinner.tsx
|       |   |   +-- Badge.tsx
|       |   |   +-- Separator.tsx
|       |   +-- layout/
|       |       +-- PanelLayout.tsx
|       |       +-- ResizablePanels.tsx
|       |       +-- DockLayout.tsx
|       |
|       +-- stores/
|       |   +-- project-store.ts
|       |   +-- timeline-store.ts
|       |   +-- media-store.ts
|       |   +-- playback-store.ts
|       |   +-- color-store.ts
|       |   +-- audio-store.ts
|       |   +-- effects-store.ts
|       |   +-- ai-store.ts
|       |   +-- export-store.ts
|       |   +-- ui-store.ts
|       |
|       +-- services/
|       |   +-- api-client.ts         # Axios instance setup
|       |   +-- project-service.ts
|       |   +-- media-service.ts
|       |   +-- timeline-service.ts
|       |   +-- render-service.ts
|       |   +-- ai-service.ts
|       |   +-- export-service.ts
|       |   +-- plugin-service.ts
|       |   +-- websocket-client.ts   # WebSocket singleton
|       |
|       +-- hooks/
|       |   +-- useDebounce.ts
|       |   +-- useThrottle.ts
|       |   +-- useKeyboard.ts
|       |   +-- useLocalStorage.ts
|       |   +-- useBackendStatus.ts
|       |   +-- useUndoRedo.ts
|       |
|       +-- types/
|       |   +-- project.types.ts
|       |   +-- timeline.types.ts
|       |   +-- media.types.ts
|       |   +-- effects.types.ts
|       |   +-- ai.types.ts
|       |   +-- api.types.ts
|       |   +-- api-generated.ts     # Auto-generated from OpenAPI
|       |
|       +-- utils/
|       |   +-- format-time.ts       # ms -> HH:MM:SS:FF
|       |   +-- format-bytes.ts      # bytes -> human readable
|       |   +-- cn.ts                # className merger
|       |   +-- clamp.ts
|       |   +-- lerp.ts
|       |   +-- debounce.ts
|       |
|       +-- test-utils/
|           +-- setup.ts             # Vitest setup
|           +-- factories.ts         # Test data factories
|           +-- render-utils.tsx     # Custom render with providers
|           +-- mocks/
|               +-- handlers.ts      # MSW handlers
|               +-- server.ts        # MSW server setup
```

---

## 3. Backend Source (`backend/`)

```
backend/
|
+-- main.py                          # FastAPI app entry point
+-- config.py                        # Settings (pydantic-settings)
+-- dependencies.py                  # FastAPI dependency injection
+-- version.py                       # Version constant
|
+-- api/
|   +-- __init__.py
|   +-- websocket.py                 # WebSocket endpoint
|   +-- v1/
|       +-- __init__.py
|       +-- router.py                # Main v1 router aggregator
|       +-- projects.py              # /projects endpoints
|       +-- media.py                 # /media endpoints
|       +-- timeline.py              # /timeline endpoints
|       +-- render.py                # /render endpoints
|       +-- color.py                 # /color endpoints
|       +-- effects.py               # /effects endpoints
|       +-- ai.py                    # /ai endpoints
|       +-- export.py                # /export endpoints
|       +-- plugins.py               # /plugins endpoints
|       +-- system.py                # /system endpoints
|
+-- services/
|   +-- __init__.py
|   +-- project_service.py
|   +-- media_service.py
|   +-- timeline_service.py
|   +-- render_service.py
|   +-- color_service.py
|   +-- ai_service.py
|   +-- export_service.py
|   +-- plugin_service.py
|   +-- event_bus.py                 # Internal event publishing
|
+-- repositories/
|   +-- __init__.py
|   +-- project_repo.py
|   +-- media_repo.py
|   +-- timeline_repo.py
|   +-- ai_task_repo.py
|   +-- export_job_repo.py
|   +-- base_repo.py                 # Abstract base repository
|
+-- domain/
|   +-- __init__.py
|   +-- models/
|   |   +-- project.py
|   |   +-- media.py
|   |   +-- timeline.py
|   |   +-- clip.py
|   |   +-- effect.py
|   |   +-- keyframe.py
|   |   +-- color_grade.py
|   |   +-- ai_task.py
|   |   +-- export_job.py
|   +-- events/
|       +-- project_events.py
|       +-- media_events.py
|       +-- ai_events.py
|       +-- export_events.py
|
+-- infrastructure/
|   +-- __init__.py
|   +-- db/
|   |   +-- __init__.py
|   |   +-- session.py              # SQLAlchemy async session
|   |   +-- models.py               # SQLAlchemy ORM models
|   |   +-- migrations/
|   |       +-- env.py              # Alembic env
|   |       +-- versions/           # Migration files
|   |           +-- 001_initial_schema.py
|   +-- ffmpeg/
|   |   +-- __init__.py
|   |   +-- adapter.py              # FFmpeg command wrapper
|   |   +-- probe.py                # ffprobe media info
|   |   +-- filtergraph.py          # Complex filter builder
|   |   +-- encoder_selector.py     # Hardware encoder detection
|   +-- ai/
|   |   +-- __init__.py
|   |   +-- engine.py               # AI task queue + worker pool
|   |   +-- model_registry.py       # LRU model cache
|   |   +-- model_downloader.py     # Model download + verify
|   |   +-- workers/
|   |       +-- __init__.py
|   |       +-- transcription.py    # Whisper worker
|   |       +-- background_removal.py # SAM2 worker
|   |       +-- object_tracking.py  # YOLO worker
|   |       +-- upscaling.py        # ESRGAN worker
|   |       +-- scene_detection.py  # PySceneDetect worker
|   |       +-- noise_reduction.py  # DeepFilterNet worker
|   |       +-- color_match.py      # Color matching worker
|   +-- storage/
|       +-- __init__.py
|       +-- local_storage.py        # File system operations
|       +-- temp_manager.py         # Temp file lifecycle
|
+-- plugins/
|   +-- __init__.py
|   +-- sandbox.py                  # Python plugin sandbox
|   +-- loader.py                   # Plugin loader
|   +-- registry.py                 # Installed plugin registry
|   +-- sdk/
|       +-- __init__.py
|       +-- base.py                 # Plugin base class
|       +-- effect_plugin.py
|       +-- ai_plugin.py
|
+-- tests/
    +-- conftest.py                  # pytest fixtures
    +-- factories.py                 # factory_boy factories
    +-- unit/
    |   +-- test_project_service.py
    |   +-- test_timeline_service.py
    |   +-- test_media_service.py
    |   +-- test_ai_service.py
    |   +-- test_render_service.py
    |   +-- test_export_service.py
    |   +-- test_ffmpeg_adapter.py
    +-- integration/
    |   +-- test_project_api.py
    |   +-- test_media_api.py
    |   +-- test_timeline_api.py
    |   +-- test_ai_api.py
    |   +-- test_export_api.py
    +-- performance/
        +-- test_timeline_perf.py
        +-- test_render_perf.py
```

---

## 4. Configuration Files

```
Root config files:
  package.json                -- npm config, scripts, dependencies
  package-lock.json           -- Locked npm dependency tree
  tsconfig.json               -- TypeScript (renderer)
  tsconfig.main.json          -- TypeScript (main process)
  tsconfig.preload.json       -- TypeScript (preload)
  vite.config.ts              -- Vite bundler config
  electron-vite.config.ts     -- electron-vite config
  electron-builder.yml        -- Build/package config
  .eslintrc.json              -- ESLint rules
  .prettierrc                 -- Prettier formatting
  .gitignore
  .github/dependabot.yml

Python config files:
  requirements.txt            -- Production dependencies (pinned)
  requirements-dev.txt        -- Dev/test dependencies
  requirements-bundle.txt     -- Bundle-only dependencies (no dev tools)
  pyproject.toml              -- Ruff + Black + mypy + pytest config
  alembic.ini                 -- Alembic migration config
```
