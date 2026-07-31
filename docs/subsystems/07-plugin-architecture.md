# 07 — Plugin Architecture

> **Document:** Plugin Architecture v1.0
> **Project:** AI Video Editor
> **Date:** 2026-08-01

---

## 1. Overview

The plugin system enables third-party developers to extend the AI Video Editor with:
- Custom effects and filters
- Custom AI tools
- Custom importers/exporters
- Custom UI panels
- Custom transitions

---

## 2. Plugin Types

| Type | Description | API Surface |
|------|-------------|------------|
| `effect` | Video/audio effect | Effect chain hook |
| `ai-tool` | AI processing tool | AI task handler |
| `importer` | Custom media format | Media import hook |
| `exporter` | Custom export format | Export pipeline hook |
| `ui-panel` | Custom side panel | React component slot |
| `transition` | Custom transition | Transition renderer |
| `theme` | Visual theme | CSS variables override |

---

## 3. Plugin Architecture

```
+------------------------------------------------------------------+
|                     PLUGIN SYSTEM                                 |
|                                                                   |
|  Plugin Marketplace (cloud)                                        |
|       |                                                           |
|       | download                                                  |
|       v                                                           |
|  Plugin Store (local filesystem)                                  |
|  ~/.aivideoedit/plugins/<plugin-id>/                              |
|       |                                                           |
|       v                                                           |
|  Plugin Registry (in-memory)                                      |
|       |                                                           |
|  +----+-------+--------+--------+--------+--------+              |
|  |            |        |        |        |        |              |
|  v            v        v        v        v        v              |
| [Effect    [AI Tool [Import  [Export  [UI Panel [Theme           |
|  Sandbox]  Sandbox] Sandbox] Sandbox] Slot]     Override]        |
|  Worker     Worker                    iframe                      |
|  Thread     Thread                    sandbox                     |
+------------------------------------------------------------------+
```

---

## 4. Plugin Manifest (plugin.json)

```json
{
  "id": "com.example.cinematic-luts",
  "name": "Cinematic LUTs Pack",
  "version": "1.2.0",
  "type": "effect",
  "description": "50 professional cinematic LUT presets",
  "author": "Example Studio",
  "license": "MIT",
  "homepage": "https://example.com/plugin",
  "icon": "icon.png",
  "minAppVersion": "1.0.0",
  "maxAppVersion": "2.x",
  "permissions": [
    "read-media",
    "write-temp"
  ],
  "entrypoints": {
    "frontend": "dist/frontend.js",
    "backend": "backend/plugin.py"
  },
  "settings": {
    "schema": "settings-schema.json"
  },
  "marketplace": {
    "category": "effects",
    "tags": ["lut", "color", "cinematic"],
    "price": "free",
    "downloads": 12450,
    "rating": 4.8
  }
}
```

---

## 5. Plugin Permissions Model

```
PERMISSIONS (Least Privilege):
  read-media      -- Read source media files
  write-temp      -- Write to plugin temp directory
  write-output    -- Write final output files
  network         -- Make HTTP requests (DANGEROUS - requires approval)
  system-execute  -- Execute system commands (DISABLED - never allowed)
  filesystem-full -- Full filesystem access (DISABLED - never allowed)
  gpu             -- Use GPU for inference
  audio           -- Access audio API
```

---

## 6. Frontend Plugin API

```typescript
// Plugin SDK - frontend
interface PluginEffectAPI {
  // Register an effect in the effects library
  registerEffect(definition: EffectDefinition): void;
  
  // Register a UI panel slot
  registerPanel(definition: PanelDefinition): void;
  
  // Access timeline (read-only)
  getTimeline(): ReadonlyTimeline;
  
  // Subscribe to events
  on(event: PluginEvent, handler: EventHandler): Unsubscribe;
  
  // Show notifications
  notify(message: string, type: 'info' | 'success' | 'error'): void;
}

interface EffectDefinition {
  id: string;
  name: string;
  category: string;
  icon: string;
  parameters: ParameterDefinition[];
  // GLSL shader source
  fragmentShader: string;
  // Parameter uniform bindings
  uniformBindings: UniformBinding[];
}
```

### 6.1 Plugin GLSL Effect Example

```glsl
// Plugin-provided fragment shader
uniform sampler2D inputTexture;
uniform float intensity;   // From parameter definition
uniform vec3 tintColor;    // From parameter definition

void main() {
    vec4 color = texture2D(inputTexture, vTexCoord);
    // Apply tint effect
    vec4 tinted = mix(color, vec4(tintColor, color.a), intensity);
    gl_FragColor = tinted;
}
```

---

## 7. Backend Plugin API

```python
# Plugin SDK - backend
from aivideoedit.plugin_sdk import PluginBase, AITaskPlugin, register_plugin

class MyAIPlugin(AITaskPlugin):
    """Example AI plugin for custom processing."""
    
    PLUGIN_ID = "com.example.my-ai-plugin"
    
    def get_task_types(self) -> list[str]:
        return ["my_custom_ai_task"]
    
    async def process(self, task: AITask, progress: ProgressCallback) -> dict:
        media_path = task.parameters["media_path"]
        
        # Run custom AI processing
        result = await self._run_inference(media_path, progress)
        
        return {"output_path": result.output_path, "metadata": result.metadata}
    
    async def _run_inference(self, path: str, progress: ProgressCallback) -> Result:
        # Plugin-defined AI inference
        ...

register_plugin(MyAIPlugin)
```

---

## 8. Plugin Sandbox Security

### Frontend Sandbox (Web Worker)
```typescript
// Plugin runs in isolated Worker
const worker = new Worker(pluginEntrypoint, {
  type: 'module',
  credentials: 'omit',
});

// Only allowed message types pass through
const ALLOWED_MESSAGES = ['effect-result', 'panel-update', 'notification'];

worker.addEventListener('message', (event) => {
  if (!ALLOWED_MESSAGES.includes(event.data.type)) {
    console.warn('Plugin attempted unauthorized message:', event.data.type);
    return;
  }
  handlePluginMessage(event.data);
});
```

### Backend Sandbox (Python)
```python
# Plugin runs in restricted Python environment
import sys
import importlib

class PluginSandbox:
    # Modules forbidden from plugin imports
    BLOCKED_MODULES = [
        'subprocess', 'os.system', 'socket', 
        'ctypes', 'importlib', 'builtins.exec'
    ]
    
    def load_plugin(self, plugin_path: str) -> PluginBase:
        # Validate plugin code statically (AST analysis)
        with open(plugin_path) as f:
            source = f.read()
        self._validate_ast(source)
        
        # Load in restricted namespace
        sandbox_globals = self._build_sandbox_globals()
        exec(compile(source, plugin_path, 'exec'), sandbox_globals)
        return sandbox_globals.get('plugin_instance')
```

---

## 9. Plugin Lifecycle

```
Installation
    |
    v
Manifest validation (schema check)
    |
    v
Permission approval (user dialog)
    |
    v
Code signature verification (if signed)
    |
    v
Plugin registered in Plugin Registry
    |
    v
Frontend entry loaded in Worker
    |
    v
Backend entry loaded in sandbox
    |
    v
Plugin active (hooks connected)
    |
    v
[Uninstall / Update / Disable]
```

---

## 10. Plugin Marketplace API

| Endpoint | Description |
|----------|-------------|
| `GET /marketplace/plugins` | List plugins with filters |
| `GET /marketplace/plugins/{id}` | Plugin details |
| `POST /marketplace/install` | Install plugin |
| `DELETE /marketplace/plugins/{id}` | Uninstall plugin |
| `POST /marketplace/update/{id}` | Update plugin |
| `GET /marketplace/updates` | Check for updates |

---

## 11. Plugin Development Kit (PDK)

The PDK will be published as:
- `@aivideoedit/plugin-sdk` (npm package for frontend plugins)
- `aivideoedit-plugin-sdk` (PyPI package for backend plugins)

Developer tooling:
- `npx create-aivideoedit-plugin` scaffold
- Hot-reload dev mode
- Plugin testing framework
- API documentation site
- Example plugins repository
