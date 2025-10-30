# Architettura del Sistema

## Diagramma Completo

```
┌──────────────────────────────────────────────────────────────────┐
│                        DEVELOPER                                  │
│                                                                   │
│  ┌─────────────┐                                                 │
│  │ Local Code  │                                                 │
│  │   Editor    │                                                 │
│  └──────┬──────┘                                                 │
│         │ git add, commit, push                                  │
└─────────┼────────────────────────────────────────────────────────┘
          │
          ▼
┌──────────────────────────────────────────────────────────────────┐
│                       GITHUB                                      │
│                                                                   │
│  ┌────────────────┐                                              │
│  │   Repository   │                                              │
│  │                │                                              │
│  │ ├── app.js     │                                              │
│  │ ├── Dockerfile │                                              │
│  │ ├── .github/   │                                              │
│  │     └── workflows/                                            │
│  │         └── deploy.yml ◄─── Trigger on push                  │
│  └────────┬───────┘                                              │
│           │                                                       │
│           │ Webhook event                                        │
│           ▼                                                       │
│  ┌────────────────┐                                              │
│  │ GitHub Actions │                                              │
│  │   Runner       │                                              │
│  │  (Ubuntu VM)   │                                              │
│  │                │                                              │
│  │ 1. Checkout    │                                              │
│  │ 2. Read secrets│                                              │
│  │ 3. SSH connect │                                              │
│  └────────┬───────┘                                              │
└───────────┼──────────────────────────────────────────────────────┘
            │
            │ SSH (port 22)
            │ auth: VPS_SSH_KEY
            │
            ▼
┌──────────────────────────────────────────────────────────────────┐
│                      VPS SERVER                                   │
│                  (srv.example.com)                                │
│                                                                   │
│  ┌──────────────────────────────────────────────────────┐        │
│  │ SSH Daemon (sshd)                                    │        │
│  │ authorized_keys: github-actions.pub                  │        │
│  └────────────┬─────────────────────────────────────────┘        │
│               │                                                   │
│               │ Commands executed                                │
│               ▼                                                   │
│  ┌──────────────────────────────────────────────────────┐        │
│  │ /opt/my-project/                                     │        │
│  │                                                       │        │
│  │ ├── app.js           ◄── Updated via git pull        │        │
│  │ ├── Dockerfile                                        │        │
│  │ ├── docker-compose.yml                                │        │
│  │ ├── scripts/                                          │        │
│  │ │   ├── check-port.sh                                 │        │
│  │ │   └── deploy.sh  ◄── Executed                       │        │
│  │ └── .env (generated)                                  │        │
│  └────────────┬─────────────────────────────────────────┘        │
│               │                                                   │
│               │ docker compose build                             │
│               │ docker compose up -d                             │
│               ▼                                                   │
│  ┌──────────────────────────────────────────────────────┐        │
│  │ Docker Engine                                         │        │
│  │                                                       │        │
│  │  ┌────────────────────────────────────────┐          │        │
│  │  │ Container: my-project                  │          │        │
│  │  │                                         │          │        │
│  │  │ ┌────────────────────────────────────┐ │          │        │
│  │  │ │ File System (isolated)             │ │          │        │
│  │  │ │                                     │ │          │        │
│  │  │ │ /app/                               │ │          │        │
│  │  │ │  ├── node_modules/                  │ │          │        │
│  │  │ │  ├── app.js                         │ │          │        │
│  │  │ │  └── package.json                   │ │          │        │
│  │  │ └────────────────────────────────────┘ │          │        │
│  │  │                                         │          │        │
│  │  │ Process:                                │          │        │
│  │  │  PID 1: npm start                       │          │        │
│  │  │  ├── node app.js                        │          │        │
│  │  │  └── listening on 0.0.0.0:3000          │          │        │
│  │  │                                         │          │        │
│  │  │ Network:                                │          │        │
│  │  │  Container port 3000 ◄─┐                │          │        │
│  │  └──────────────────────┼─────────────────┘          │        │
│  │                         │                             │        │
│  │  Port mapping:          │                             │        │
│  │  Host :3000 ───────────┘                             │        │
│  └────────────┬─────────────────────────────────────────┘        │
│               │                                                   │
│               │ Exposed on host                                  │
│               ▼                                                   │
│  ┌──────────────────────────────────────────────────────┐        │
│  │ NGINX (optional)                                      │        │
│  │                                                       │        │
│  │ listen 80/443                                         │        │
│  │ proxy_pass → http://localhost:3000                    │        │
│  └────────────┬─────────────────────────────────────────┘        │
└───────────────┼──────────────────────────────────────────────────┘
                │
                │ Public internet
                ▼
      ┌─────────────────┐
      │   END USERS     │
      │                 │
      │ Browser/API     │
      │ Clients         │
      └─────────────────┘
```

## Componenti Dettagliati

### 1. GitHub Repository

**Responsabilità:**
- Archiviare codice sorgente
- Versioning con Git
- Triggare workflows su eventi

**Struttura Chiave:**
```
my-project/
├── .github/
│   └── workflows/
│       ├── deploy.yml        # Auto-deploy workflow
│       └── test.yml           # Test workflow
├── app.js                     # Application code
├── package.json               # Dependencies
├── package-lock.json          # Locked dependencies
├── Dockerfile                 # Container definition
├── docker-compose.yml         # Orchestration
├── scripts/
│   ├── check-port.sh          # Port availability check
│   └── deploy.sh              # Deployment logic
└── README.md                  # Documentation
```

### 2. GitHub Actions

**Responsabilità:**
- Orchestrazione CI/CD
- Esecuzione automatica task
- Integrazione con VPS

**Runner Environment:**
```
Ubuntu 24.04 LTS
├── Docker 28.x
├── Git 2.x
├── Node.js (multiple versions)
├── Python 3.x
└── Standard Unix tools
```

**Workflow Lifecycle:**
```
Event (push) → Trigger → Allocate Runner → Execute Jobs → Cleanup
```

**Secrets Management:**
```
GitHub Encrypted Storage
         ↓
  Runtime Decryption
         ↓
Environment Variables (masked in logs)
         ↓
Used by workflow steps
```

### 3. SSH Connection

**Componenti:**
```
┌─────────────────┐         ┌─────────────────┐
│ GitHub Actions  │         │   VPS Server    │
│                 │         │                 │
│ Private Key     │←──────→│  Public Key     │
│ (from secret)   │  Auth   │  (in .ssh/)     │
│                 │         │                 │
│ SSH Client      │─────────│  SSH Daemon     │
│                 │  TCP    │  (port 22)      │
└─────────────────┘  22     └─────────────────┘
```

**Autenticazione:**
1. GitHub Actions legge `VPS_SSH_KEY` (private key)
2. Connette a `VPS_HOST:VPS_PORT`
3. SSH daemon verifica chiave con `authorized_keys`
4. Se match → sessione stabilita
5. Comandi eseguiti con privilegi dell'utente

**Security:**
- Chiavi Ed25519 (moderne, sicure)
- No password authentication
- Keys rotabili senza downtime

### 4. VPS Server

**Componenti Installati:**
```
Ubuntu Server
├── Docker Engine
│   ├── Containerd
│   ├── Docker Compose v2
│   └── Docker CLI
├── Git
├── lsof (port checking)
├── curl (health checks)
└── NGINX (optional, reverse proxy)
```

**Directory Structure:**
```
/
├── opt/
│   ├── my-project/          # Deployed app
│   │   ├── .git/
│   │   ├── app.js
│   │   ├── .env
│   │   └── ...
│   ├── another-app/         # Other apps
│   └── ...
├── root/
│   └── .ssh/
│       ├── authorized_keys  # Public keys
│       └── github-actions   # Private key (backup)
└── var/
    └── lib/
        └── docker/          # Docker data
            ├── containers/
            ├── images/
            └── volumes/
```

### 5. Docker Architecture

**Layered Filesystem:**
```
┌─────────────────────────────────────┐
│ Layer 4: Application Code (R/W)    │  ← Container writable layer
├─────────────────────────────────────┤
│ Layer 3: node_modules               │
├─────────────────────────────────────┤
│ Layer 2: package.json               │
├─────────────────────────────────────┤
│ Layer 1: Working directory          │
├─────────────────────────────────────┤
│ Layer 0: Node.js + Alpine Linux     │  ← Base image (read-only)
└─────────────────────────────────────┘
```

**Container Networking:**
```
┌─────────────────────────────────────────┐
│ Host (VPS)                              │
│                                         │
│  eth0: public IP                        │
│  lo: 127.0.0.1                          │
│                                         │
│  ┌───────────────────────────────────┐ │
│  │ Docker Network (bridge)           │ │
│  │                                   │ │
│  │  docker0: 172.17.0.1              │ │
│  │                                   │ │
│  │  ┌─────────────────────────────┐ │ │
│  │  │ Container                   │ │ │
│  │  │ eth0: 172.17.0.2            │ │ │
│  │  │ app listening on :3000      │ │ │
│  │  └─────────────────────────────┘ │ │
│  │           ▲                       │ │
│  │           │ Port mapping          │ │
│  │           │ 3000:3000             │ │
│  └───────────┼───────────────────────┘ │
│              │                         │
│  Host :3000 ─┘                         │
└─────────────────────────────────────────┘
              ▲
              │
         Public access
```

**Resource Isolation:**
```
Container Limits (optional):
├── CPU: 0.5 cores
├── Memory: 512MB
├── Disk I/O: throttled
└── Network: isolated namespace
```

### 6. Deploy Script Flow

```
deploy.sh execution:

1. Configuration
   ├── Read PROJECT_DIR
   ├── Set DEFAULT_PORT
   └── Load environment

2. Port Check
   ├── Run check-port.sh
   ├── Test port availability
   └── Find free port if needed

3. Environment Setup
   ├── Generate .env file
   │   ├── COMPOSE_PROJECT_NAME
   │   ├── APP_PORT
   │   ├── HOST_PORT
   │   └── NODE_ENV
   └── Write to disk

4. Container Lifecycle
   ├── docker compose down
   │   ├── Stop container
   │   ├── Remove container
   │   └── Clean networks
   ├── docker compose build
   │   ├── Read Dockerfile
   │   ├── Execute build steps
   │   └── Tag image
   └── docker compose up -d
       ├── Create container
       ├── Start process
       └── Detach (background)

5. Verification
   ├── Wait 5 seconds
   ├── docker compose ps
   ├── Check status "Up"
   └── Return exit code
```

### 7. Health Check System

**Applicazione Level:**
```javascript
// app.js
app.get('/health', (req, res) => {
  res.json({
    status: 'healthy',
    timestamp: new Date().toISOString()
  });
});
```

**Docker Level:**
```dockerfile
HEALTHCHECK --interval=30s --timeout=3s \
  CMD node -e "require('http').get('http://localhost:3000/health', ...)"
```

**Workflow Level:**
```bash
curl -f http://localhost:3000/health || exit 1
```

**Monitoring Flow:**
```
Every 30s:
  Docker → HTTP GET /health
          ↓
    Response 200 OK?
    ├── Yes → Container status: healthy
    └── No  → Container status: unhealthy
            ↓
         After 3 failures → Container restarted
```

## Data Flow

### Deploy Flow (Detailed)

```
1. Code Push
   Developer → Git → GitHub Repository

2. Trigger
   GitHub Repository → Webhook → GitHub Actions

3. Checkout
   GitHub Actions → Git clone → Runner filesystem

4. SSH Setup
   GitHub Actions → Read VPS_SSH_KEY → SSH client config

5. Connection
   SSH client → Network → VPS SSH daemon
                       → Authentication
                       → Session established

6. Repository Sync
   SSH session → cd /opt/project
              → git fetch origin
              → git reset --hard origin/main

7. Script Execution
   SSH session → bash scripts/deploy.sh
              → Execute each line
              → Capture output (stdout/stderr)
              → Monitor exit code

8. Port Detection
   deploy.sh → scripts/check-port.sh
            → lsof system call
            → Return free port

9. Docker Build
   deploy.sh → docker compose build
            → Read Dockerfile
            → Pull base image (if needed)
            → Execute RUN commands
            → Create layers
            → Tag final image

10. Container Start
    deploy.sh → docker compose up -d
             → Create container from image
             → Configure networking
             → Set environment variables
             → Start init process (PID 1)
             → Execute CMD

11. Health Check
    deploy.sh → sleep 5
             → docker compose ps
             → curl http://localhost:PORT/health

12. Verification
    SSH session → Exit code 0 (success) or 1 (failure)
               → Send to GitHub Actions

13. Reporting
    GitHub Actions → Parse output
                  → Update workflow status
                  → Display in UI
```

## Scalabilità

### Single VPS, Multiple Apps

```
VPS Server
├── /opt/app1/ → Container app1:3000
├── /opt/app2/ → Container app2:3001
├── /opt/app3/ → Container app3:3002
└── ...

NGINX:
├── app1.domain.com → localhost:3000
├── app2.domain.com → localhost:3001
└── app3.domain.com → localhost:3002
```

### Multi-Service App

```yaml
# docker-compose.yml
services:
  app:
    build: .
    ports:
      - "3000:3000"
    depends_on:
      - database
      - redis

  database:
    image: postgres:16
    volumes:
      - db-data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine

volumes:
  db-data:
```

## Security

### Secrets Flow

```
Developer → GitHub UI → Encrypted storage
                             ↓
                       GitHub Actions runtime
                             ↓
                       Decrypted in memory only
                             ↓
                       Used in workflow
                             ↓
                       Masked in logs (never visible)
                             ↓
                       Destroyed after workflow
```

### Network Security

```
Internet
   ↓
Firewall (VPS)
   ├── Port 22 (SSH) ← GitHub Actions only (IP whitelist optional)
   ├── Port 80 (HTTP) ← Public
   ├── Port 443 (HTTPS) ← Public
   └── Other ports CLOSED
          ↓
   Docker containers (isolated network)
```

### Best Practices

1. **Secrets rotation:** Cambia chiavi SSH periodicamente
2. **Minimal permissions:** Container run as non-root user
3. **Network isolation:** Container su network privato
4. **Resource limits:** CPU/Memory limits per container
5. **Image scanning:** Scan immagini per vulnerabilità

## Monitoring & Logging

### Log Aggregation

```
Container stdout/stderr
        ↓
Docker logging driver
        ↓
/var/lib/docker/containers/[id]/[id]-json.log
        ↓
docker compose logs
        ↓
External logging (optional):
├── CloudWatch
├── Elasticsearch
└── Syslog
```

### Metrics Collection (Optional)

```
Container
   ↓ metrics
Prometheus exporter
   ↓
Prometheus server
   ↓ queries
Grafana dashboard
```

## Next Steps

- [Setup Guide](SETUP.md) - Come configurare il sistema
- [Workflows](WORKFLOWS.md) - Personalizzare GitHub Actions
- [Scripts](SCRIPTS.md) - Dettagli script di deployment
