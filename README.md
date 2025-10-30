# VPS Auto-Deploy System

Sistema completo di auto-deployment per VPS utilizzando **GitHub Actions**, **Docker** e **SSH**.

## Overview

Questo sistema permette di deployare automaticamente applicazioni su un VPS ogni volta che viene fatto un push su GitHub. Il deployment avviene tramite GitHub Actions che si connettono al VPS via SSH, buildano l'immagine Docker e avviano il container.

## Features

- ✅ **Auto-deploy su push** - Deployment automatico ad ogni push su main/master
- ✅ **Controllo porte intelligente** - Trova automaticamente una porta libera
- ✅ **Docker containerizzato** - Applicazioni isolate e riproducibili
- ✅ **Health checks** - Verifica automatica dello stato del servizio
- ✅ **Test CI/CD** - Test automatici su ogni Pull Request
- ✅ **Zero downtime** - Gestione graceful shutdown e restart
- ✅ **Rollback automatico** - Fallback in caso di errori
- ✅ **Logs centralizzati** - Monitoraggio tramite Docker logs

## Architettura

```
┌─────────────────┐
│  Developer      │
│  git push       │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  GitHub         │
│  Repository     │
└────────┬────────┘
         │ webhook
         ▼
┌─────────────────┐
│  GitHub Actions │
│  Workflow       │
└────────┬────────┘
         │ SSH
         ▼
┌─────────────────┐      ┌──────────────┐
│  VPS Server     │─────▶│   Docker     │
│  srv.example.com│      │   Container  │
└─────────────────┘      └──────────────┘
```

## Quick Start

### 1. Prerequisiti VPS

Sul tuo VPS, installa:

```bash
# Docker
curl -fsSL https://get.docker.com | sh

# Aggiungi user corrente al gruppo docker
sudo usermod -aG docker $USER
```

### 2. Crea progetto da template

Clona il template:

```bash
git clone https://github.com/frede1983/docker-deploy-template my-project
cd my-project
```

### 3. Configura GitHub Secrets

Genera chiave SSH per GitHub Actions:

```bash
# Sul VPS
ssh-keygen -t ed25519 -C "github-actions" -f ~/.ssh/github-actions -N ""
cat ~/.ssh/github-actions.pub >> ~/.ssh/authorized_keys

# Mostra chiave privata (da copiare in GitHub Secret)
cat ~/.ssh/github-actions
```

Vai su GitHub → Settings → Secrets and variables → Actions e aggiungi:

| Secret | Valore |
|--------|--------|
| `VPS_HOST` | Hostname del VPS (es. `srv1013438.hstgr.cloud`) |
| `VPS_USERNAME` | Username SSH (es. `root`) |
| `VPS_SSH_KEY` | Chiave privata SSH (output di `cat ~/.ssh/github-actions`) |
| `VPS_PORT` | Porta SSH (default: `22`) |

### 4. Push e Deploy

```bash
git add .
git commit -m "Initial deploy"
git push
```

GitHub Actions deploya automaticamente sul VPS!

## Documentazione

- [Setup Completo](docs/SETUP.md) - Guida dettagliata passo-passo
- [Architettura](docs/ARCHITECTURE.md) - Design e componenti del sistema
- [Workflows](docs/WORKFLOWS.md) - Spiegazione GitHub Actions workflows
- [Script](docs/SCRIPTS.md) - Documentazione script di deployment
- [Troubleshooting](docs/TROUBLESHOOTING.md) - Risoluzione problemi comuni
- [Best Practices](docs/BEST_PRACTICES.md) - Consigli e ottimizzazioni

## Template Repository

Il template di esempio è disponibile qui:
**https://github.com/frede1983/docker-deploy-template**

Include:
- Express.js API con health checks
- Dockerfile ottimizzato
- Docker Compose configuration
- Script di deployment con port detection
- GitHub Actions workflows (deploy + test)
- Documentazione completa

## Componenti

### 1. GitHub Actions Workflows

#### Deploy Workflow (`.github/workflows/deploy.yml`)

Triggered su push a main/master:
1. Checkout repository
2. SSH al VPS
3. Clone/update repository
4. Esegue script di deployment
5. Verifica health checks

#### Test Workflow (`.github/workflows/test.yml`)

Triggered su PR e push:
1. Valida configurazione Docker
2. Build immagine
3. Test container localmente
4. Verifica endpoints

### 2. Scripts

#### `scripts/check-port.sh`

Controlla se una porta è disponibile, altrimenti trova la prima porta libera.

```bash
./scripts/check-port.sh 3000
# Output: 3000 (se libera) o porta alternativa
```

#### `scripts/deploy.sh`

Script principale di deployment:
1. Controlla porta disponibile
2. Crea/aggiorna file `.env`
3. Ferma container esistenti
4. Build nuova immagine Docker
5. Avvia container
6. Verifica health check

### 3. Docker Configuration

#### Dockerfile

Multi-stage build ottimizzato per Node.js:
- Base image: `node:22-alpine`
- Copia solo `package*.json` prima
- Installa dipendenze production only
- Health check integrato

#### docker-compose.yml

Orchestrazione semplificata:
- Configurazione porta tramite env vars
- Health checks
- Auto-restart policy
- Network isolation

## Utilizzo

### Deploy Automatico

Ogni push su `main` o `master` triggera automaticamente il deploy:

```bash
git add .
git commit -m "Update feature"
git push
```

### Deploy Manuale

Triggera deployment senza push:

```bash
gh workflow run deploy.yml
```

### Monitoring

Sul VPS:

```bash
# Status container
cd /opt/[project-name]
docker compose ps

# Logs real-time
docker compose logs -f

# Logs ultimi 100
docker compose logs --tail=100

# Restart
docker compose restart
```

### Verifica Deployment

```bash
# Health check
curl http://localhost:[PORT]/health

# Info servizio
curl http://localhost:[PORT]/api/info
```

## Customizzazione

### Cambiare porta default

Modifica `scripts/deploy.sh`:

```bash
DEFAULT_PORT=8080  # Cambia qui
```

### Aggiungere variabili ambiente

Modifica `docker-compose.yml`:

```yaml
environment:
  - NODE_ENV=production
  - API_KEY=${API_KEY}  # Nuovo
```

E aggiungi secret su GitHub.

### Multiple services

Aggiungi servizi a `docker-compose.yml`:

```yaml
services:
  app:
    # ...

  database:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
```

## Troubleshooting

### Deploy fallisce con "port already in use"

Lo script trova automaticamente una porta libera. Controlla il file `.env`:

```bash
cat /opt/[project-name]/.env
```

### Container non si avvia

Controlla logs:

```bash
docker compose logs
```

Rebuilda immagine:

```bash
docker compose build --no-cache
docker compose up -d
```

### GitHub Actions: "Permission denied (publickey)"

Verifica che la chiave SSH sia corretta:

```bash
# Sul VPS - testa connessione
ssh -i ~/.ssh/github-actions $VPS_USERNAME@$VPS_HOST
```

Controlla che `VPS_SSH_KEY` contenga l'intera chiave privata (incluso header/footer).

## Best Practices

1. **Mai committare secrets** - Usa sempre GitHub Secrets
2. **Pin dependency versions** - Specifica versioni esatte in package.json
3. **Health checks obbligatori** - Ogni servizio deve avere un endpoint `/health`
4. **Logs strutturati** - Usa formato JSON per logging
5. **Graceful shutdown** - Gestisci SIGTERM correttamente
6. **Resource limits** - Imposta limiti memoria/CPU in docker-compose.yml

## Esempi

Nella directory `examples/` trovi progetti di esempio:

- `examples/nodejs-api/` - API REST Node.js
- `examples/python-worker/` - Worker Python
- `examples/nginx-static/` - Sito statico con NGINX
- `examples/multi-service/` - App multi-container

## Contribuire

Questo è un sistema in continua evoluzione. Contributi benvenuti!

## License

MIT License - Vedi [LICENSE](LICENSE) file.

## Credits

Sistema sviluppato per VPS Hostinger con Claude Code.

---

**Repository Template:** https://github.com/frede1983/docker-deploy-template
**Documentazione:** https://github.com/frede1983/vps-auto-deploy-system
