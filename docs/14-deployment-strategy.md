# 14 — Deployment Strategy

> **Document:** Deployment Strategy v1.0
> **Project:** AI Video Editor
> **Date:** 2026-08-01

---

## 1. Distribution Model

The AI Video Editor is a **self-contained desktop application**:
- No server-side infrastructure required for core functionality
- All processing is local (video, AI, rendering)
- Optional cloud features (backup, marketplace) require internet

---

## 2. Build Targets

| Platform | Architecture | Format | Distribution |
|----------|-------------|--------|-------------|
| Windows 10/11 | x64 | `.exe` (NSIS installer) | Direct download |
| Windows 10/11 | x64 | `.msix` | Microsoft Store |
| macOS 13+ | Universal (Intel + ARM) | `.dmg` | Direct download |
| macOS 13+ | Universal | `.pkg` | Mac App Store |
| Linux | x64 | `.AppImage` | Direct download |
| Linux | x64 | `.deb` | APT repository |
| Linux | x64 | `.rpm` | RPM repository |

---

## 3. Application Bundle Structure

```
AI Video Editor.app/  (or install directory)
|
+-- electron/                    -- Electron runtime
|   +-- electron.exe             -- Main process
|   +-- resources/
|       +-- app.asar             -- Compiled React app
|       +-- app.asar.unpacked/   -- Native modules
|
+-- python/                      -- Embedded Python runtime
|   +-- python.exe               -- Python interpreter
|   +-- Lib/                     -- Standard library
|   +-- site-packages/           -- All backend dependencies
|   |   +-- fastapi/
|   |   +-- torch/               -- PyTorch (~2GB with CUDA)
|   |   +-- cv2/
|   |   +-- faster_whisper/
|   |   +-- ultralytics/
|   +-- backend/                 -- Our application code
|       +-- main.py
|       +-- api/
|       +-- services/
|       +-- ...
|
+-- bin/                         -- Binary tools
|   +-- ffmpeg.exe               -- FFmpeg binary
|   +-- ffprobe.exe
|
+-- models/                      -- AI models (optional, lazy downloaded)
|   +-- README.txt               -- "Models downloaded on first use"
|
+-- resources/                   -- App resources
    +-- icons/
    +-- fonts/
    +-- effects/                 -- Built-in effect shaders
    +-- luts/                    -- Built-in LUT files
```

---

## 4. Python Bundling Strategy

The Python runtime is embedded using **PyInstaller** + **python-build-standalone**:

```bash
# Step 1: Download standalone Python
# Uses python-build-standalone for a clean, relocatable Python

# Step 2: Install dependencies into the bundle
./python/python.exe -m pip install -r requirements-bundle.txt \
  --target ./python/site-packages \
  --no-cache-dir

# Step 3: Strip unnecessary files
find ./python -name "*.pyc" -delete
find ./python -name "__pycache__" -type d -exec rm -rf {} +
find ./python -name "*.pyi" -delete
find ./python/site-packages -name "tests" -type d -exec rm -rf {} +

# Total bundle size targets:
# Without PyTorch CUDA: ~800MB
# With PyTorch CUDA:    ~4GB
# With PyTorch CPU:     ~2GB
```

### 4.1 AI Models: Lazy Download Strategy
Large AI models are NOT bundled:
- App ships without model weights
- On first AI feature use, model downloads automatically
- Progress dialog shows download progress
- Models stored in `~/.aivideoedit/models/`

```python
class ModelDownloader:
    MODEL_REGISTRY = {
        "whisper-medium": {
            "url": "https://cdn.aivideoedit.com/models/whisper-medium.bin",
            "size_mb": 769,
            "sha256": "..."
        },
        "sam2-large": {
            "url": "https://cdn.aivideoedit.com/models/sam2_hiera_large.pt",
            "size_mb": 856,
            "sha256": "..."
        },
    }
    
    async def ensure_model(self, model_key: str, progress: ProgressCallback) -> Path:
        model_path = self._get_model_path(model_key)
        if model_path.exists() and self._verify_checksum(model_path, model_key):
            return model_path
        
        await self._download(model_key, model_path, progress)
        return model_path
```

---

## 5. Auto-Update System

### Update Strategy
- Uses `electron-updater` (included in electron-builder)
- Update server: GitHub Releases + S3 CDN
- Check frequency: Once per day (app startup)
- Update types:
  - Patch (1.0.x): Silent background download, install on restart
  - Minor (1.x.0): Notification with changelog, user installs
  - Major (x.0.0): Mandatory notification, release notes

### Update Flow
```
App starts
    |
    v
Check update server (background)
    |
    +-- No update: Continue
    |
    +-- Update available:
            |
            v
        Download in background (delta update if available)
            |
            v
        Notify user: "Update ready to install"
            |
            v
        User clicks "Install & Restart"
            |
            v
        electron-updater installs and restarts
```

---

## 6. Code Signing

### Windows
- Certificate: EV Code Signing Certificate (DigiCert)
- Signed with: `signtool.exe`
- SmartScreen: EV certificate bypasses SmartScreen warnings

### macOS
- Certificate: Apple Developer ID Application
- Notarized with Apple Notary Service
- Hardened Runtime enabled
- Entitlements: camera access denied, disk access scoped

### Linux
- GPG signed `.deb` and `.rpm` packages
- AppImage signed with GPG

---

## 7. Installation Locations

| Platform | Data Directory | Config | Cache |
|----------|---------------|--------|-------|
| Windows | `%APPDATA%\AIVideoEditor` | `%APPDATA%\AIVideoEditor\config.json` | `%LOCALAPPDATA%\AIVideoEditor\Cache` |
| macOS | `~/Library/Application Support/AIVideoEditor` | `~/Library/Preferences/com.aivideoedit.plist` | `~/Library/Caches/AIVideoEditor` |
| Linux | `~/.config/aivideoedit` | `~/.config/aivideoedit/config.json` | `~/.cache/aivideoedit` |

---

## 8. Build Matrix

```yaml
# Platforms built in CI:
jobs:
  build-windows:
    runs-on: windows-2022
    steps:
      - Build frontend (Vite)
      - Bundle Python runtime
      - Run electron-builder (NSIS + MSIX)
      - Sign with EV certificate
      - Upload to S3

  build-macos:
    runs-on: macos-14  # Apple Silicon runner
    steps:
      - Build frontend (Vite)
      - Bundle Python runtime (universal)
      - Run electron-builder (DMG + PKG)
      - Sign + Notarize
      - Upload to S3

  build-linux:
    runs-on: ubuntu-22.04
    steps:
      - Build frontend (Vite)
      - Bundle Python runtime
      - Run electron-builder (AppImage + DEB + RPM)
      - GPG sign packages
      - Upload to S3
```

---

## 9. Release Checklist

- [ ] All tests pass on CI
- [ ] Version bumped in `package.json` and `backend/version.py`
- [ ] `CHANGELOG.md` updated
- [ ] Release branch created: `release/v1.x.x`
- [ ] Builds pass on all 3 platforms
- [ ] Code signing verified
- [ ] macOS notarization verified
- [ ] Update manifest pushed to update server
- [ ] GitHub Release created with binaries
- [ ] CDN cache invalidated
- [ ] Announcement sent (Discord, website)
