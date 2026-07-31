# 08 — IPC & Communication Architecture

> **Document:** IPC & Communication v1.0
> **Project:** AI Video Editor
> **Date:** 2026-08-01

---

## 1. Overview

The AI Video Editor uses three distinct communication channels:

| Channel | Parties | Protocol | Usage |
|---------|---------|---------|-------|
| IPC Bridge | Renderer <-> Main | Electron contextBridge | OS operations, file dialogs |
| REST API | Main/Renderer <-> Backend | HTTP/1.1 (localhost) | Request/response operations |
| WebSocket | Main/Renderer <-> Backend | WS (localhost) | Real-time events & streaming |

---

## 2. Electron IPC Architecture

### 2.1 Security Model
```
Renderer Process
  (contextIsolation: true, nodeIntegration: false)
      |
      | window.electronAPI.xxx()   <- contextBridge
      |
      v
Preload Script (bridge)
      |
      | ipcRenderer.invoke('channel', ...args)
      |
      v
Main Process
      |
      | ipcMain.handle('channel', handler)
      |
      v
Native OS API / File System / Backend HTTP client
```

### 2.2 IPC Channel Definitions

```typescript
// preload.ts — contextBridge API surface
const electronAPI = {
  // File system operations
  openFileDialog: (options: OpenDialogOptions) => Promise<string[]>,
  saveFileDialog: (options: SaveDialogOptions) => Promise<string | null>,
  readFile: (path: string) => Promise<Buffer>,
  writeFile: (path: string, data: Buffer) => Promise<void>,
  showInExplorer: (path: string) => Promise<void>,
  
  // App operations
  getAppVersion: () => Promise<string>,
  getPlatform: () => Promise<string>,
  getResourcePath: () => Promise<string>,
  
  // Window operations
  minimizeWindow: () => void,
  maximizeWindow: () => void,
  closeWindow: () => void,
  setTitle: (title: string) => void,
  
  // Backend proxy (Main process forwards to backend)
  backendURL: () => Promise<string>,   // localhost:8755
  
  // OS keychain
  getSecret: (key: string) => Promise<string | null>,
  setSecret: (key: string, value: string) => Promise<void>,
  deleteSecret: (key: string) => Promise<void>,
  
  // App events
  onUpdateAvailable: (cb: (info: UpdateInfo) => void) => Unsubscribe,
  onMenuAction: (cb: (action: MenuAction) => void) => Unsubscribe,
}

contextBridge.exposeInMainWorld('electronAPI', electronAPI)
```

---

## 3. REST API Communication

### 3.1 Base URL
```
http://127.0.0.1:8755/api/v1/
```

### 3.2 Request/Response Format

All API requests and responses use JSON.

**Standard Response Envelope:**
```json
{
  "success": true,
  "data": { ... },
  "error": null,
  "meta": {
    "timestamp": "2026-08-01T00:00:00Z",
    "version": "1.0.0",
    "requestId": "uuid"
  }
}
```

**Error Response:**
```json
{
  "success": false,
  "data": null,
  "error": {
    "code": "MEDIA_NOT_FOUND",
    "message": "Media item with id 'abc123' not found",
    "details": {}
  },
  "meta": { ... }
}
```

### 3.3 Axios Client Setup (Frontend)

```typescript
// services/api-client.ts
import axios from 'axios';

const apiClient = axios.create({
  baseURL: 'http://127.0.0.1:8755/api/v1',
  timeout: 30000,
  headers: {
    'Content-Type': 'application/json',
    'X-App-Version': APP_VERSION,
  },
});

// Request interceptor: add correlation ID
apiClient.interceptors.request.use((config) => {
  config.headers['X-Request-Id'] = crypto.randomUUID();
  return config;
});

// Response interceptor: unwrap envelope
apiClient.interceptors.response.use(
  (response) => response.data.data,
  (error) => {
    const appError = transformBackendError(error.response?.data?.error);
    return Promise.reject(appError);
  }
);
```

### 3.4 HTTP Status Code Policy

| Code | Meaning | When Used |
|------|---------|----------|
| 200 | OK | Successful GET, PUT |
| 201 | Created | Successful POST (creates resource) |
| 202 | Accepted | Long-running task started |
| 204 | No Content | Successful DELETE |
| 400 | Bad Request | Validation error |
| 404 | Not Found | Resource not found |
| 409 | Conflict | Duplicate / state conflict |
| 422 | Unprocessable | Pydantic validation error |
| 500 | Server Error | Unhandled exception |
| 503 | Unavailable | Backend starting up |

---

## 4. WebSocket Communication

### 4.1 WebSocket URL
```
ws://127.0.0.1:8756/ws
```

### 4.2 WebSocket Message Protocol

```typescript
// All WS messages follow this schema
interface WSMessage {
  type: WSMessageType;
  id: string;          // correlation ID
  timestamp: string;   // ISO 8601
  payload: unknown;
}

type WSMessageType = 
  // Server -> Client (events)
  | 'render.progress'
  | 'ai.task.progress'
  | 'ai.task.complete'
  | 'ai.task.error'
  | 'export.progress'
  | 'export.complete'
  | 'media.imported'
  | 'proxy.ready'
  | 'project.saved'
  | 'backend.ready'
  
  // Client -> Server (subscriptions)
  | 'subscribe'
  | 'unsubscribe'
  | 'ping'
  | 'pong'
```

### 4.3 WebSocket Client (Frontend)

```typescript
// services/websocket-client.ts
class WebSocketClient {
  private ws: WebSocket | null = null;
  private handlers = new Map<WSMessageType, Set<Handler>>();
  private reconnectTimer: NodeJS.Timeout | null = null;
  
  connect(url: string) {
    this.ws = new WebSocket(url);
    
    this.ws.onopen = () => {
      this.emit('connected');
      this.startHeartbeat();
    };
    
    this.ws.onmessage = (event) => {
      const message: WSMessage = JSON.parse(event.data);
      this.dispatch(message);
    };
    
    this.ws.onclose = () => {
      this.emit('disconnected');
      this.scheduleReconnect();
    };
    
    this.ws.onerror = (error) => {
      this.emit('error', error);
    };
  }
  
  on<T>(type: WSMessageType, handler: (payload: T) => void): Unsubscribe {
    if (!this.handlers.has(type)) {
      this.handlers.set(type, new Set());
    }
    this.handlers.get(type)!.add(handler as Handler);
    return () => this.handlers.get(type)?.delete(handler as Handler);
  }
  
  private scheduleReconnect() {
    this.reconnectTimer = setTimeout(() => this.connect(this.url), 3000);
  }
  
  private startHeartbeat() {
    setInterval(() => {
      this.send({ type: 'ping', id: crypto.randomUUID(), timestamp: new Date().toISOString(), payload: {} });
    }, 30000);
  }
}
```

### 4.4 WebSocket Server (Backend)

```python
# backend/api/websocket.py
from fastapi import WebSocket, WebSocketDisconnect
from collections import defaultdict
import asyncio, json

class ConnectionManager:
    def __init__(self):
        self.active: list[WebSocket] = []
        self.subscriptions: dict[str, set[WebSocket]] = defaultdict(set)
    
    async def connect(self, websocket: WebSocket):
        await websocket.accept()
        self.active.append(websocket)
    
    def disconnect(self, websocket: WebSocket):
        self.active.remove(websocket)
        for subscribers in self.subscriptions.values():
            subscribers.discard(websocket)
    
    async def broadcast(self, message_type: str, payload: dict):
        message = {
            "type": message_type,
            "id": str(uuid4()),
            "timestamp": datetime.utcnow().isoformat(),
            "payload": payload
        }
        dead = []
        for ws in self.active:
            try:
                await ws.send_json(message)
            except Exception:
                dead.append(ws)
        for ws in dead:
            self.disconnect(ws)

manager = ConnectionManager()

@router.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await manager.connect(websocket)
    try:
        while True:
            data = await websocket.receive_json()
            await handle_client_message(websocket, data, manager)
    except WebSocketDisconnect:
        manager.disconnect(websocket)
```

---

## 5. Backend Process Management

### 5.1 Backend Startup (Main Process)

```typescript
// main/backend-manager.ts
import { spawn, ChildProcess } from 'child_process';
import axios from 'axios';

class BackendManager {
  private process: ChildProcess | null = null;
  private port = 8755;
  
  async start(): Promise<void> {
    const pythonPath = this.getBundledPythonPath();
    const scriptPath = this.getBackendScriptPath();
    
    this.process = spawn(pythonPath, [scriptPath, '--port', String(this.port)], {
      stdio: ['ignore', 'pipe', 'pipe'],
      env: { ...process.env, PYTHONUNBUFFERED: '1' }
    });
    
    this.process.stdout?.on('data', (data) => logger.info('[Backend]', data.toString()));
    this.process.stderr?.on('data', (data) => logger.error('[Backend]', data.toString()));
    
    await this.waitForReady();
  }
  
  private async waitForReady(): Promise<void> {
    for (let i = 0; i < 30; i++) {
      try {
        await axios.get(`http://127.0.0.1:${this.port}/health`, { timeout: 1000 });
        return;
      } catch {
        await new Promise(r => setTimeout(r, 500));
      }
    }
    throw new Error('Backend failed to start within 15 seconds');
  }
  
  async stop(): Promise<void> {
    if (this.process) {
      this.process.kill('SIGTERM');
      this.process = null;
    }
  }
}
```

---

## 6. Error Handling in Communication

### Retry Strategy (REST)
```typescript
const retryConfig = {
  retries: 3,
  retryDelay: (attemptNumber: number) => Math.pow(2, attemptNumber) * 1000,
  retryCondition: (error: AxiosError) => {
    return error.response?.status === 503 || !error.response;
  }
};
```

### WebSocket Reconnection
- Exponential backoff: 1s, 2s, 4s, 8s, max 30s
- Store pending messages and replay on reconnect
- UI shows "Reconnecting..." indicator

---

## 7. Performance Considerations

| Concern | Solution |
|---------|---------|
| Large media uploads | Chunked upload via multipart/form-data |
| Frame streaming | Binary WebSocket frames (ArrayBuffer) |
| Timeline state sync | Debounce 200ms before sending updates |
| Progress updates | Throttle to max 10 updates/sec |
| API response cache | React Query with 30s stale time |
