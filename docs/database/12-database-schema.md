# 12 — Database Schema

> **Document:** Database Schema v1.0
> **Project:** AI Video Editor
> **Date:** 2026-08-01

---

## 1. Database Technology

- **Engine:** SQLite 3.45+ (WAL mode)
- **ORM:** SQLAlchemy 2.0 (async)
- **Migrations:** Alembic
- **File Location:** `~/.aivideoedit/projects/<project_id>/project.db`
- **Backup:** Auto-backup every save to `project.db.bak`

### WAL Mode Configuration
```sql
PRAGMA journal_mode = WAL;
PRAGMA synchronous = NORMAL;
PRAGMA foreign_keys = ON;
PRAGMA cache_size = -64000;  -- 64MB page cache
PRAGMA temp_store = MEMORY;
```

---

## 2. Schema Overview

```
+----------------+        +------------------+
|    projects    |        |   media_items    |
|----------------|        |------------------|
| id (PK)        |<--+    | id (PK)          |
| name           |   |    | project_id (FK)  |
| created_at     |   +--->| file_path        |
| updated_at     |        | type             |
| settings_json  |        | duration_ms      |
| version        |        | width            |
+----------------+        | height           |
                           | fps              |
                           | codec            |
+------------------+       | file_size_bytes  |
| timeline_tracks  |       | thumbnail_path   |
|------------------|       | proxy_path       |
| id (PK)          |       | waveform_path    |
| project_id (FK)  |       | metadata_json    |
| type             |       | created_at       |
| name             |       +------------------+
| position         |
| height           |       +------------------+
| is_locked        |       | timeline_clips   |
| is_muted         |       |------------------|
| is_hidden        |       | id (PK)          |
| created_at       |<----->| track_id (FK)    |
+------------------+       | media_id (FK)    |
                           | position_ms      |
                           | duration_ms      |
                           | in_point_ms      |
                           | out_point_ms     |
                           | speed_factor     |
                           | is_reversed      |
                           | label_color      |
                           | created_at       |
                           +------------------+
```

---

## 3. Full Table Definitions

### 3.1 projects
```sql
CREATE TABLE projects (
    id              TEXT PRIMARY KEY,           -- UUID v4
    name            TEXT NOT NULL,
    description     TEXT DEFAULT '',
    created_at      DATETIME NOT NULL DEFAULT (datetime('now')),
    updated_at      DATETIME NOT NULL DEFAULT (datetime('now')),
    version         INTEGER NOT NULL DEFAULT 1, -- Schema version for migrations
    settings_json   TEXT NOT NULL DEFAULT '{}', -- JSON: resolution, fps, color space
    thumbnail_path  TEXT,
    is_archived     BOOLEAN NOT NULL DEFAULT FALSE,
    last_opened_at  DATETIME
);

CREATE INDEX idx_projects_created_at ON projects(created_at DESC);
CREATE INDEX idx_projects_is_archived ON projects(is_archived);
```

### 3.2 media_items
```sql
CREATE TABLE media_items (
    id              TEXT PRIMARY KEY,
    project_id      TEXT NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    file_path       TEXT NOT NULL,             -- Absolute path
    type            TEXT NOT NULL,             -- 'video' | 'audio' | 'image'
    duration_ms     INTEGER,                   -- NULL for images
    width           INTEGER,
    height          INTEGER,
    fps             REAL,
    codec           TEXT,
    audio_codec     TEXT,
    file_size_bytes INTEGER NOT NULL,
    thumbnail_path  TEXT,
    proxy_path      TEXT,                      -- Lower-res proxy file
    waveform_path   TEXT,                      -- JSON waveform data file
    metadata_json   TEXT DEFAULT '{}',         -- Extra codec metadata
    bin_folder_id   TEXT REFERENCES media_bins(id),
    is_offline      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      DATETIME NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX idx_media_items_project_id ON media_items(project_id);
CREATE INDEX idx_media_items_type ON media_items(type);
CREATE INDEX idx_media_items_bin_folder_id ON media_items(bin_folder_id);
```

### 3.3 media_bins (folder organization)
```sql
CREATE TABLE media_bins (
    id          TEXT PRIMARY KEY,
    project_id  TEXT NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name        TEXT NOT NULL,
    parent_id   TEXT REFERENCES media_bins(id) ON DELETE CASCADE,
    position    INTEGER NOT NULL DEFAULT 0,
    created_at  DATETIME NOT NULL DEFAULT (datetime('now'))
);
```

### 3.4 timeline_tracks
```sql
CREATE TABLE timeline_tracks (
    id          TEXT PRIMARY KEY,
    project_id  TEXT NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    type        TEXT NOT NULL,             -- 'video' | 'audio' | 'text' | 'effect'
    name        TEXT NOT NULL,
    position    INTEGER NOT NULL,          -- Track order (0 = top)
    height      INTEGER NOT NULL DEFAULT 64,
    is_locked   BOOLEAN NOT NULL DEFAULT FALSE,
    is_muted    BOOLEAN NOT NULL DEFAULT FALSE,
    is_hidden   BOOLEAN NOT NULL DEFAULT FALSE,
    is_solo     BOOLEAN NOT NULL DEFAULT FALSE,
    volume_db   REAL NOT NULL DEFAULT 0.0,
    pan         REAL NOT NULL DEFAULT 0.0, -- -1.0 to 1.0
    created_at  DATETIME NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX idx_timeline_tracks_project_id ON timeline_tracks(project_id);
CREATE INDEX idx_timeline_tracks_position ON timeline_tracks(project_id, position);
```

### 3.5 timeline_clips
```sql
CREATE TABLE timeline_clips (
    id              TEXT PRIMARY KEY,
    track_id        TEXT NOT NULL REFERENCES timeline_tracks(id) ON DELETE CASCADE,
    media_id        TEXT REFERENCES media_items(id) ON DELETE SET NULL,
    name            TEXT NOT NULL,
    position_ms     INTEGER NOT NULL,      -- Position in timeline
    duration_ms     INTEGER NOT NULL,
    in_point_ms     INTEGER NOT NULL DEFAULT 0,    -- Media in point
    out_point_ms    INTEGER,               -- NULL = use full duration
    speed_factor    REAL NOT NULL DEFAULT 1.0,     -- 2.0 = 2x speed
    is_reversed     BOOLEAN NOT NULL DEFAULT FALSE,
    label_color     TEXT DEFAULT '#4A90E2',
    opacity         REAL NOT NULL DEFAULT 1.0,
    blend_mode      TEXT NOT NULL DEFAULT 'normal',
    created_at      DATETIME NOT NULL DEFAULT (datetime('now')),
    updated_at      DATETIME NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX idx_timeline_clips_track_id ON timeline_clips(track_id);
CREATE INDEX idx_timeline_clips_position_ms ON timeline_clips(track_id, position_ms);
CREATE INDEX idx_timeline_clips_media_id ON timeline_clips(media_id);
```

### 3.6 keyframes
```sql
CREATE TABLE keyframes (
    id          TEXT PRIMARY KEY,
    clip_id     TEXT NOT NULL REFERENCES timeline_clips(id) ON DELETE CASCADE,
    parameter   TEXT NOT NULL,             -- e.g., 'opacity', 'scale', 'position_x'
    time_ms     INTEGER NOT NULL,          -- Time relative to clip start
    value       TEXT NOT NULL,             -- JSON-encoded value
    easing      TEXT NOT NULL DEFAULT 'linear',  -- 'linear' | 'ease-in' | 'ease-out' | 'bezier'
    bezier_json TEXT,                      -- Bezier control points for custom easing
    created_at  DATETIME NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX idx_keyframes_clip_id ON keyframes(clip_id);
CREATE UNIQUE INDEX idx_keyframes_clip_param_time ON keyframes(clip_id, parameter, time_ms);
```

### 3.7 effects
```sql
CREATE TABLE effects (
    id          TEXT PRIMARY KEY,
    clip_id     TEXT NOT NULL REFERENCES timeline_clips(id) ON DELETE CASCADE,
    effect_type TEXT NOT NULL,             -- e.g., 'blur', 'color-grade', 'chroma-key'
    plugin_id   TEXT,                      -- NULL for built-in effects
    position    INTEGER NOT NULL DEFAULT 0,-- Order in effect chain
    params_json TEXT NOT NULL DEFAULT '{}',
    is_enabled  BOOLEAN NOT NULL DEFAULT TRUE,
    created_at  DATETIME NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX idx_effects_clip_id ON effects(clip_id);
```

### 3.8 color_grades
```sql
CREATE TABLE color_grades (
    id              TEXT PRIMARY KEY,
    clip_id         TEXT NOT NULL UNIQUE REFERENCES timeline_clips(id) ON DELETE CASCADE,
    -- Primary wheels
    lift_r          REAL NOT NULL DEFAULT 0.0,
    lift_g          REAL NOT NULL DEFAULT 0.0,
    lift_b          REAL NOT NULL DEFAULT 0.0,
    gamma_r         REAL NOT NULL DEFAULT 1.0,
    gamma_g         REAL NOT NULL DEFAULT 1.0,
    gamma_b         REAL NOT NULL DEFAULT 1.0,
    gain_r          REAL NOT NULL DEFAULT 1.0,
    gain_g          REAL NOT NULL DEFAULT 1.0,
    gain_b          REAL NOT NULL DEFAULT 1.0,
    -- Global adjustments
    brightness      REAL NOT NULL DEFAULT 0.0,
    contrast        REAL NOT NULL DEFAULT 0.0,
    saturation      REAL NOT NULL DEFAULT 1.0,
    hue             REAL NOT NULL DEFAULT 0.0,
    temperature     REAL NOT NULL DEFAULT 0.0,
    tint            REAL NOT NULL DEFAULT 0.0,
    -- Curves (stored as JSON arrays of [x, y] points)
    curve_master     TEXT NOT NULL DEFAULT '[[0,0],[1,1]]',
    curve_red        TEXT NOT NULL DEFAULT '[[0,0],[1,1]]',
    curve_green      TEXT NOT NULL DEFAULT '[[0,0],[1,1]]',
    curve_blue       TEXT NOT NULL DEFAULT '[[0,0],[1,1]]',
    -- LUT
    lut_path        TEXT,
    lut_intensity   REAL NOT NULL DEFAULT 1.0,
    created_at      DATETIME NOT NULL DEFAULT (datetime('now')),
    updated_at      DATETIME NOT NULL DEFAULT (datetime('now'))
);
```

### 3.9 export_jobs
```sql
CREATE TABLE export_jobs (
    id              TEXT PRIMARY KEY,
    project_id      TEXT NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    status          TEXT NOT NULL DEFAULT 'queued', -- 'queued'|'running'|'done'|'failed'|'cancelled'
    output_path     TEXT NOT NULL,
    format          TEXT NOT NULL,         -- 'mp4' | 'mov' | 'mkv' | 'webm' | 'gif'
    codec           TEXT NOT NULL,         -- 'h264' | 'h265' | 'prores' | 'vp9'
    resolution      TEXT NOT NULL,         -- '1920x1080'
    fps             REAL NOT NULL,
    bitrate_kbps    INTEGER,
    encoder         TEXT,                  -- 'nvenc' | 'qsv' | 'amf' | 'libx264'
    settings_json   TEXT NOT NULL DEFAULT '{}',
    progress        REAL NOT NULL DEFAULT 0.0,
    total_frames    INTEGER,
    elapsed_ms      INTEGER,
    error_message   TEXT,
    created_at      DATETIME NOT NULL DEFAULT (datetime('now')),
    started_at      DATETIME,
    completed_at    DATETIME
);

CREATE INDEX idx_export_jobs_project_id ON export_jobs(project_id);
CREATE INDEX idx_export_jobs_status ON export_jobs(status);
```

### 3.10 ai_tasks
```sql
CREATE TABLE ai_tasks (
    id              TEXT PRIMARY KEY,
    project_id      TEXT NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    media_id        TEXT REFERENCES media_items(id) ON DELETE CASCADE,
    type            TEXT NOT NULL,         -- 'transcription' | 'background_removal' | ...
    status          TEXT NOT NULL DEFAULT 'pending',
    priority        INTEGER NOT NULL DEFAULT 5,
    parameters_json TEXT NOT NULL DEFAULT '{}',
    progress        REAL NOT NULL DEFAULT 0.0,
    result_json     TEXT,
    result_path     TEXT,                  -- Path to result file if any
    error_message   TEXT,
    model_version   TEXT,
    created_at      DATETIME NOT NULL DEFAULT (datetime('now')),
    started_at      DATETIME,
    completed_at    DATETIME
);

CREATE INDEX idx_ai_tasks_project_id ON ai_tasks(project_id);
CREATE INDEX idx_ai_tasks_status ON ai_tasks(status);
```

### 3.11 plugin_installations
```sql
CREATE TABLE plugin_installations (
    id              TEXT PRIMARY KEY,      -- plugin manifest id
    name            TEXT NOT NULL,
    version         TEXT NOT NULL,
    type            TEXT NOT NULL,
    install_path    TEXT NOT NULL,
    is_enabled      BOOLEAN NOT NULL DEFAULT TRUE,
    settings_json   TEXT DEFAULT '{}',
    installed_at    DATETIME NOT NULL DEFAULT (datetime('now')),
    updated_at      DATETIME NOT NULL DEFAULT (datetime('now'))
);
```

---

## 4. Migration Strategy

- **Tool:** Alembic
- **Migration files:** `backend/db/migrations/versions/`
- **Versioning:** Each migration has up() and down() functions
- **Auto-migration:** Run on app startup before backend serves requests

```python
# Example migration
def upgrade():
    op.add_column('timeline_clips', 
        sa.Column('label_color', sa.Text(), nullable=True, default='#4A90E2'))

def downgrade():
    op.drop_column('timeline_clips', 'label_color')
```

---

## 5. Data Access Patterns

| Operation | Frequency | Query Pattern |
|-----------|-----------|--------------|
| Load project timeline | On open | JOIN tracks + clips + effects |
| Update clip position | Very High | UPDATE clips WHERE id = ? |
| Add keyframe | High | INSERT INTO keyframes |
| List media items | Medium | SELECT WHERE project_id = ? |
| Load AI task progress | High | SELECT progress WHERE id = ? |
| Export job status | Medium | SELECT WHERE project_id = ? ORDER BY created_at |
