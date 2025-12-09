# WoodOps Goal

An event-driven automation platform that combines Ansible's declarative playbook model with ZeroMQ's high-performance messaging to create a reactive infrastructure management system.

## Overview

WoodOps provides:
- **Event-driven automation**: React to infrastructure events in real-time
- **Ansible at the core**: Leverage existing playbooks and Ansible ecosystem
- **Lightweight agents**: Minimal footprint daemons on managed hosts
- **Scalable messaging**: ZeroMQ for high-throughput, low-latency communication
- **REST API**: Django REST Framework for integrations and UI

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Management Server                             │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   REST API   │  │  Event Bus   │  │  Scheduler   │          │
│  │   (Django)   │  │   (ZeroMQ)   │  │  (Celery)    │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│         │                 │                 │                   │
│  ┌──────────────────────────────────────────────────┐          │
│  │              Playbook Engine                      │          │
│  │  (ansible-runner + dynamic inventory)            │          │
│  └──────────────────────────────────────────────────┘          │
│         │                 │                 │                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   State DB   │  │  Playbook    │  │   Secrets    │          │
│  │  (Postgres)  │  │   Store      │  │   (Vault)    │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
                              │
                    ZeroMQ (PUB/SUB + ROUTER/DEALER)
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐     ┌───────────────┐     ┌───────────────┐
│    Agent      │     │    Agent      │     │    Agent      │
│  (Linux VM)   │     │  (Windows)    │     │  (Container)  │
├───────────────┤     ├───────────────┤     ├───────────────┤
│ - Event Watch │     │ - Event Watch │     │ - Event Watch │
│ - Fact Gather │     │ - Fact Gather │     │ - Fact Gather │
│ - Local Exec  │     │ - Local Exec  │     │ - Local Exec  │
└───────────────┘     └───────────────┘     └───────────────┘
```

## Components

### Management Server

#### REST API (Django REST Framework)

The API provides endpoints for:
- Agent management (registration, status, facts)
- Playbook management (CRUD, execution triggers)
- Event rules configuration
- Execution history and audit logs
- User authentication and RBAC