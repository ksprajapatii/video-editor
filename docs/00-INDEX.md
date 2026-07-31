# AI Video Editor — Documentation Index

> **Version:** 1.0.0-architecture
> **Status:** Phase 1 — Software Architecture
> **Last Updated:** 2026-08-01

---

## Document Map

### Core Architecture
| # | Document | Description |
|---|----------|-------------|
| 01 | [Software Requirements](./01-software-requirements.md) | Functional & non-functional requirements |
| 02 | [Overall Architecture](./02-overall-architecture.md) | High-level system design |
| 03 | [Technology Stack](./03-technology-stack.md) | Tech choices with rationale |

### Subsystem Design
| # | Document | Description |
|---|----------|-------------|
| 04 | [Subsystems Overview](./subsystems/04-subsystems-overview.md) | All subsystem descriptions |
| 05 | [Rendering Engine](./subsystems/05-rendering-engine.md) | GPU/CPU rendering pipeline |
| 06 | [AI Engine](./subsystems/06-ai-engine.md) | AI/ML subsystem design |
| 07 | [Plugin Architecture](./subsystems/07-plugin-architecture.md) | Plugin system & marketplace |
| 08 | [IPC & Communication](./subsystems/08-ipc-communication.md) | Frontend <-> Backend communication |

### Standards & Conventions
| # | Document | Description |
|---|----------|-------------|
| 09 | [Coding Standards](./standards/09-coding-standards.md) | Code style, linting, formatting |
| 10 | [Naming Conventions](./standards/10-naming-conventions.md) | Variable, file, module naming |
| 11 | [Testing Strategy](./standards/11-testing-strategy.md) | Unit, integration, E2E testing |

### Data & API
| # | Document | Description |
|---|----------|-------------|
| 12 | [Database Schema](./database/12-database-schema.md) | SQLite + schema definitions |
| 13 | [API Architecture](./api/13-api-architecture.md) | REST + WebSocket API design |

### Deployment & CI/CD
| # | Document | Description |
|---|----------|-------------|
| 14 | [Deployment Strategy](./14-deployment-strategy.md) | Build, package, distribution |
| 15 | [CI/CD Strategy](./15-cicd-strategy.md) | GitHub Actions pipelines |

### Diagrams
| # | Document | Description |
|---|----------|-------------|
| 16 | [UML Diagrams](./diagrams/16-uml-diagrams.md) | Class, sequence, component UML |
| 17 | [Mermaid Diagrams](./diagrams/17-mermaid-diagrams.md) | Architecture & flow diagrams |
| 18 | [Dependency Graph](./diagrams/18-dependency-graph.md) | Module dependency map |

### Roadmap
| # | Document | Description |
|---|----------|-------------|
| 19 | [Project Directory](./19-project-directory.md) | Full folder structure |
| 20 | [Development Roadmap](./roadmap/20-development-roadmap.md) | Phased milestones & sprints |

---

## Architecture Summary

```
+----------------------------------------------------------+
|                   AI VIDEO EDITOR                         |
|                                                           |
|  +--------------+     IPC      +---------------------+   |
|  |   FRONTEND   |<------------>|      BACKEND        |   |
|  |  Electron +  |   REST +     | FastAPI + Python     |   |
|  |  React + TS  |  WebSocket   | + FFmpeg + PyTorch   |   |
|  +--------------+              +---------------------+   |
|                                                           |
|  +---------------------------------------------------------+|
|  |              SHARED LAYER                              ||
|  |  SQLite DB  |  File System  |  Plugin Registry         ||
|  +---------------------------------------------------------+|
+----------------------------------------------------------+
```
