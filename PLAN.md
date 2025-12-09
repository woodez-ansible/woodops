# WoodOps

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

```python
# Example API endpoints
GET    /api/v1/agents/                 # List all agents
GET    /api/v1/agents/{id}/            # Agent details + facts
POST   /api/v1/agents/{id}/execute/    # Trigger playbook on agent
GET    /api/v1/playbooks/              # List playbooks
POST   /api/v1/playbooks/{id}/run/     # Execute playbook
GET    /api/v1/events/                 # Event history
POST   /api/v1/rules/                  # Create event rule
GET    /api/v1/executions/             # Execution history
```

#### Event Broker (ZeroMQ Hub)

- **PUB/SUB** for broadcasting events and commands
- **ROUTER/DEALER** for request/reply patterns
- Event filtering and routing rules
- Dead letter queue for failed deliveries

#### Playbook Engine

- Wraps `ansible-runner` for playbook execution
- Dynamic inventory from agent registrations
- Parallel execution across host groups
- Callback plugins for real-time status updates

#### Scheduler (Celery + Celery Beat)

- Periodic playbook execution
- Scheduled compliance checks
- Maintenance windows
- Retry logic for failed executions

### Agent Daemon

Lightweight Python daemon installed on managed hosts.

#### Core Responsibilities

- Register with management server on startup
- Maintain heartbeat connection
- Gather and report system facts periodically
- Watch for local events (file changes, service status, etc.)
- Execute local commands when triggered

#### Event Watchers (Pluggable)

| Watcher | Description |
|---------|-------------|
| `FileWatcher` | inotify/fsevents for file changes |
| `ServiceWatcher` | systemd/launchd service state changes |
| `ProcessWatcher` | Process start/stop/crash detection |
| `LogWatcher` | Pattern matching in log files |
| `MetricWatcher` | Threshold-based alerts (CPU, memory, disk) |
| `PortWatcher` | Network port state changes |
| `PackageWatcher` | Package install/update/remove events |
| `UserWatcher` | User/group modifications |
| `CronWatcher` | Scheduled local checks |

#### Fact Collectors

- OS info, hardware, network configuration
- Installed packages and versions
- Running services and their states
- Custom facts via user-defined scripts

## Event-Driven Workflows

### Event Types

```yaml
# Agent-originated events
agent.registered          # New agent comes online
agent.heartbeat           # Periodic health check
agent.facts.updated       # Facts have changed
agent.file.changed        # Watched file modified
agent.service.failed      # Service crashed or stopped
agent.disk.threshold      # Disk usage exceeded limit
agent.process.crashed     # Monitored process died
agent.compliance.drift    # Desired state mismatch

# Server-originated events
playbook.triggered        # Playbook execution requested
playbook.started          # Playbook began running
playbook.completed        # Playbook finished successfully
playbook.failed           # Playbook execution failed
schedule.triggered        # Scheduled task fired
api.request.received      # External API call received
```

### Event-to-Playbook Mapping

```yaml
# event_rules.yaml
rules:
  - name: "Auto-restart failed services"
    event: "agent.service.failed"
    conditions:
      - "event.service_name in ['nginx', 'postgresql', 'redis']"
      - "event.restart_count < 3"
    playbook: "playbooks/restart_service.yml"
    vars:
      service: "{{ event.service_name }}"

  - name: "Disk cleanup on threshold"
    event: "agent.disk.threshold"
    conditions:
      - "event.percent_used > 85"
    playbook: "playbooks/disk_cleanup.yml"

  - name: "Security patching on CVE"
    event: "external.cve.announced"
    conditions:
      - "event.severity == 'critical'"
    playbook: "playbooks/emergency_patch.yml"
    approval_required: true

  - name: "New agent provisioning"
    event: "agent.registered"
    playbook: "playbooks/bootstrap_agent.yml"
    vars:
      hostname: "{{ event.hostname }}"
```

## ZeroMQ Messaging

### Message Patterns

#### Heartbeat (PUB/SUB)

```
Server PUB  ─────►  Agents SUB
             heartbeat_check

Agents PUB  ─────►  Server SUB
             heartbeat_response + facts
```

#### Commands (ROUTER/DEALER)

```
Server ROUTER ◄────► Agent DEALER
                command_request
                command_response
```

#### Events (PUB/SUB with topics)

```
Agents PUB  ─────►  Server SUB
             topic: agent.{hostname}.{event_type}

Server PUB  ─────►  Agents SUB
             topic: broadcast.* or host.{hostname}.*
```

### Message Format

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": "2025-12-08T10:30:00Z",
  "source": "web-server-01",
  "type": "agent.service.failed",
  "payload": {
    "service_name": "nginx",
    "exit_code": 1,
    "stderr": "bind() to 0.0.0.0:80 failed",
    "facts": {
      "os": "ubuntu",
      "version": "22.04"
    }
  },
  "correlation_id": "parent-request-uuid",
  "signature": "hmac-sha256-signature"
}
```

## Key Features

### Push & Pull Hybrid

- **Push**: Server pushes playbooks to agents via ZeroMQ
- **Pull**: Agents can request playbook runs (with approval workflow)
- **Local**: Agents execute lightweight remediation locally without server

### Dynamic Inventory

```python
# Inventory automatically built from registered agents
{
    "all": {
        "hosts": {
            "web-01": {"ansible_host": "10.0.1.10", "os": "ubuntu"},
            "web-02": {"ansible_host": "10.0.1.11", "os": "ubuntu"},
            "db-01": {"ansible_host": "10.0.2.10", "os": "rhel"}
        },
        "children": {
            "webservers": {
                "hosts": ["web-01", "web-02"]
            },
            "databases": {
                "hosts": ["db-01"]
            }
        }
    }
}
```

### Compliance as Code

```yaml
# desired_state.yml
hosts:
  webservers:
    packages:
      present: [nginx, certbot, python3]
      absent: [apache2, httpd]
    services:
      running: [nginx, sshd]
      stopped: [telnetd, rshd]
    files:
      /etc/nginx/nginx.conf:
        checksum: sha256:abc123...
        owner: root
        mode: "0644"
    ports:
      listening: [80, 443, 22]
      blocked: [23, 3389]
```

### Approval Workflows

- Certain events require human approval before playbook execution
- Slack/Teams/Email integration for approval requests
- Time-boxed auto-approval for non-critical changes
- Audit trail of all approvals

### Rollback Support

- Snapshot state before playbook execution
- Automatic rollback on failure (configurable)
- Manual rollback via API or CLI
- Rollback history and diffing

## Technology Stack

| Component | Technology |
|-----------|------------|
| Server Runtime | Python 3.11+ |
| Web Framework | Django 5.x |
| REST API | Django REST Framework |
| Messaging | ZeroMQ (pyzmq) |
| Task Queue | Celery + Redis |
| Playbook Engine | ansible-runner |
| Database | PostgreSQL 15+ |
| Cache | Redis |
| Secrets | HashiCorp Vault |
| Agent Runtime | Python 3.11+ |
| CLI | Click |
| UI (optional) | React + WebSocket |

## Project Structure

```
woodops/
├── server/
│   ├── woodops/              # Django project settings
│   │   ├── settings.py
│   │   ├── urls.py
│   │   ├── celery.py
│   │   └── wsgi.py
│   ├── api/                  # Django REST Framework app
│   │   ├── models.py         # Agent, Playbook, Event, Execution models
│   │   ├── serializers.py    # DRF serializers
│   │   ├── views.py          # API viewsets
│   │   ├── urls.py           # API routing
│   │   └── permissions.py    # RBAC permissions
│   ├── broker/               # ZeroMQ event broker
│   │   ├── server.py         # Main broker process
│   │   ├── handlers.py       # Event handlers
│   │   └── routing.py        # Topic routing
│   ├── engine/               # Ansible runner wrapper
│   │   ├── runner.py         # Playbook execution
│   │   ├── inventory.py      # Dynamic inventory builder
│   │   └── callbacks.py      # Ansible callback plugins
│   ├── rules/                # Event-to-playbook mapping
│   │   ├── engine.py         # Rule evaluation
│   │   ├── conditions.py     # Condition parsing
│   │   └── actions.py        # Action execution
│   └── tasks/                # Celery tasks
│       ├── playbook.py       # Playbook execution tasks
│       ├── compliance.py     # Compliance check tasks
│       └── maintenance.py    # Scheduled maintenance
├── agent/
│   ├── woodops_agent/        # Agent package
│   │   ├── daemon.py         # Main agent daemon
│   │   ├── config.py         # Agent configuration
│   │   └── transport.py      # ZeroMQ client
│   ├── watchers/             # Pluggable event watchers
│   │   ├── base.py           # Base watcher class
│   │   ├── file.py           # File change watcher
│   │   ├── service.py        # Service state watcher
│   │   ├── process.py        # Process watcher
│   │   ├── log.py            # Log pattern watcher
│   │   ├── metric.py         # System metrics watcher
│   │   └── port.py           # Network port watcher
│   ├── facts/                # Fact collectors
│   │   ├── base.py           # Base collector class
│   │   ├── system.py         # OS and hardware facts
│   │   ├── network.py        # Network configuration
│   │   ├── packages.py       # Installed packages
│   │   └── services.py       # Running services
│   └── executor/             # Local command execution
│       ├── runner.py         # Command runner
│       └── sandbox.py        # Execution sandbox
├── playbooks/
│   ├── remediation/          # Auto-triggered playbooks
│   │   ├── restart_service.yml
│   │   ├── disk_cleanup.yml
│   │   └── kill_process.yml
│   ├── compliance/           # Drift correction
│   │   ├── enforce_packages.yml
│   │   ├── enforce_services.yml
│   │   └── enforce_files.yml
│   ├── maintenance/          # Scheduled tasks
│   │   ├── update_packages.yml
│   │   ├── rotate_logs.yml
│   │   └── backup_configs.yml
│   └── bootstrap/            # New agent setup
│       └── bootstrap_agent.yml
├── shared/
│   ├── messages/             # Message schemas
│   │   ├── events.py         # Event message types
│   │   ├── commands.py       # Command message types
│   │   └── responses.py      # Response message types
│   ├── crypto/               # Signing/encryption
│   │   ├── signing.py        # HMAC signing
│   │   └── curve.py          # CurveZMQ encryption
│   └── config/               # Shared configuration
│       └── schema.py         # Config validation
├── cli/
│   └── woodctl/              # Operator CLI tool
│       ├── main.py           # CLI entry point
│       ├── agents.py         # Agent commands
│       ├── playbooks.py      # Playbook commands
│       └── events.py         # Event commands
├── tests/
│   ├── server/               # Server tests
│   ├── agent/                # Agent tests
│   └── integration/          # Integration tests
├── docker/
│   ├── Dockerfile.server     # Server container
│   ├── Dockerfile.agent      # Agent container
│   └── docker-compose.yml    # Development stack
├── docs/
│   ├── api.md                # API documentation
│   ├── agent.md              # Agent installation guide
│   ├── events.md             # Event reference
│   └── playbooks.md          # Playbook conventions
├── requirements/
│   ├── base.txt              # Shared dependencies
│   ├── server.txt            # Server dependencies
│   └── agent.txt             # Agent dependencies
├── manage.py                 # Django management
├── pyproject.toml            # Project metadata
└── README.md                 # This file
```

## Installation

### Server

```bash
# Clone repository
git clone https://github.com/yourorg/woodops.git
cd woodops

# Create virtual environment
python -m venv venv
source venv/bin/activate

# Install server dependencies
pip install -r requirements/server.txt

# Configure database
cp .env.example .env
# Edit .env with your database credentials

# Run migrations
python manage.py migrate

# Create superuser
python manage.py createsuperuser

# Start services (development)
python manage.py runserver          # Django API
celery -A woodops worker -l info    # Celery worker
celery -A woodops beat -l info      # Celery beat
python -m broker.server             # ZeroMQ broker
```

### Agent

```bash
# On each managed host
pip install woodops-agent

# Configure agent
woodops-agent configure \
  --server zmq://management-server:5555 \
  --token <agent-token>

# Start agent
systemctl enable woodops-agent
systemctl start woodops-agent
```

## Configuration

### Server Configuration

```python
# server/woodops/settings.py

WOODOPS = {
    # ZeroMQ settings
    "ZMQ_PUB_BIND": "tcp://*:5555",
    "ZMQ_ROUTER_BIND": "tcp://*:5556",

    # Agent settings
    "AGENT_HEARTBEAT_INTERVAL": 30,  # seconds
    "AGENT_TIMEOUT": 90,  # seconds before marking offline

    # Playbook settings
    "PLAYBOOK_DIR": "/etc/woodops/playbooks",
    "ANSIBLE_INVENTORY_PLUGIN": "woodops.engine.inventory",

    # Event settings
    "EVENT_RETENTION_DAYS": 30,
    "MAX_RULE_RETRIES": 3,

    # Security
    "REQUIRE_AGENT_AUTH": True,
    "MESSAGE_SIGNING": True,
    "CURVE_ENABLED": True,
}
```

### Agent Configuration

```yaml
# /etc/woodops/agent.yml

server:
  endpoint: "tcp://management-server:5555"
  command_endpoint: "tcp://management-server:5556"

agent:
  hostname: "{{ ansible_hostname }}"  # Auto-detected if not set
  labels:
    environment: production
    role: webserver

heartbeat:
  interval: 30

watchers:
  - type: service
    services: ["nginx", "postgresql"]
  - type: file
    paths:
      - /etc/nginx/nginx.conf
      - /etc/ssh/sshd_config
  - type: metric
    thresholds:
      disk_percent: 85
      memory_percent: 90
      cpu_percent: 95

facts:
  gather_interval: 300  # 5 minutes
  custom_scripts:
    - /etc/woodops/facts.d/*.sh
```

## API Examples

### List Agents

```bash
curl -X GET http://localhost:8000/api/v1/agents/ \
  -H "Authorization: Token your-api-token"
```

```json
{
  "count": 3,
  "results": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "hostname": "web-01",
      "ip_address": "10.0.1.10",
      "status": "online",
      "last_seen": "2025-12-08T10:30:00Z",
      "labels": {"environment": "production", "role": "webserver"},
      "facts": {
        "os": "Ubuntu 22.04",
        "kernel": "5.15.0-generic",
        "memory_mb": 8192
      }
    }
  ]
}
```

### Execute Playbook

```bash
curl -X POST http://localhost:8000/api/v1/playbooks/restart_service/run/ \
  -H "Authorization: Token your-api-token" \
  -H "Content-Type: application/json" \
  -d '{
    "hosts": ["web-01", "web-02"],
    "vars": {
      "service_name": "nginx"
    }
  }'
```

```json
{
  "execution_id": "exec-123456",
  "status": "pending",
  "playbook": "restart_service",
  "hosts": ["web-01", "web-02"],
  "created_at": "2025-12-08T10:35:00Z"
}
```

### Create Event Rule

```bash
curl -X POST http://localhost:8000/api/v1/rules/ \
  -H "Authorization: Token your-api-token" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Auto-restart nginx",
    "event_type": "agent.service.failed",
    "conditions": [
      {"field": "service_name", "operator": "eq", "value": "nginx"}
    ],
    "playbook": "restart_service",
    "vars": {"service": "{{ event.service_name }}"},
    "enabled": true
  }'
```

## CLI Examples

```bash
# List all agents
woodctl agents list

# Show agent details
woodctl agents show web-01

# Run playbook on hosts
woodctl playbook run restart_service --hosts web-01,web-02 --var service=nginx

# Watch events in real-time
woodctl events watch

# Check agent status
woodctl agents ping web-01

# View execution history
woodctl executions list --limit 10

# Trigger compliance check
woodctl compliance check --hosts all
```

## Security

### Agent Authentication

Agents authenticate using pre-shared tokens or mutual TLS:

```yaml
# Token-based (simpler)
agent:
  auth:
    method: token
    token: "{{ lookup('env', 'WOODOPS_AGENT_TOKEN') }}"

# mTLS (more secure)
agent:
  auth:
    method: mtls
    cert: /etc/woodops/agent.crt
    key: /etc/woodops/agent.key
    ca: /etc/woodops/ca.crt
```

### Message Security

- **Signing**: All messages signed with HMAC-SHA256
- **Encryption**: CurveZMQ for transport encryption
- **Secrets**: Never transmitted in messages; reference Vault paths

### Access Control

Django REST Framework permissions with role-based access:

| Role | Permissions |
|------|-------------|
| Admin | Full access |
| Operator | Execute playbooks, view all |
| Viewer | Read-only access |
| Agent | Register, heartbeat, events |

## Development

### Running Tests

```bash
# All tests
pytest

# Server tests only
pytest tests/server/

# Agent tests only
pytest tests/agent/

# With coverage
pytest --cov=woodops --cov-report=html
```

### Development Environment

```bash
# Start all services with Docker Compose
docker-compose -f docker/docker-compose.yml up -d

# View logs
docker-compose -f docker/docker-compose.yml logs -f

# Run development server
docker-compose -f docker/docker-compose.yml exec server python manage.py runserver 0.0.0.0:8000
```

## Roadmap

- [ ] **Phase 1**: Core server and agent communication
- [ ] **Phase 2**: Event watchers and rule engine
- [ ] **Phase 3**: Django REST API and authentication
- [ ] **Phase 4**: Playbook engine integration
- [ ] **Phase 5**: Compliance and drift detection
- [ ] **Phase 6**: Web UI dashboard
- [ ] **Phase 7**: Multi-tenancy support
- [ ] **Phase 8**: High availability clustering

## Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Acknowledgments

- [Ansible](https://www.ansible.com/) - Automation engine
- [ZeroMQ](https://zeromq.org/) - Messaging library
- [Django](https://www.djangoproject.com/) - Web framework
- [Django REST Framework](https://www.django-rest-framework.org/) - API toolkit
