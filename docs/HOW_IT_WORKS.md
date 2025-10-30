# Come Funziona il Sistema di Auto-Deployment

Guida dettagliata che spiega **esattamente** come funziona ogni parte del sistema.

## Indice

1. [Panoramica del Flusso](#panoramica-del-flusso)
2. [Componenti Principali](#componenti-principali)
3. [Step-by-Step: Cosa Succede Durante un Deploy](#step-by-step-cosa-succede-durante-un-deploy)
4. [GitHub Actions: Il Cervello](#github-actions-il-cervello)
5. [SSH: La Connessione Sicura](#ssh-la-connessione-sicura)
6. [Docker: L'Isolamento](#docker-lisolamento)
7. [Scripts: L'Automazione](#scripts-lautomazione)
8. [Esempio Concreto](#esempio-concreto)

---

## Panoramica del Flusso

```
                    TU SVILUPPATORE
                          │
                          │ git push
                          ▼
                    ┌─────────────┐
                    │   GitHub    │
                    │  Repository │
                    └──────┬──────┘
                           │
                           │ Webhook trigger
                           ▼
                    ┌─────────────┐
                    │   GitHub    │
                    │   Actions   │◄────── Workflow file (.github/workflows/deploy.yml)
                    └──────┬──────┘
                           │
                           │ 1. Checkout code
                           │ 2. SSH connection
                           ▼
                    ┌─────────────┐
                    │  VPS Server │
                    │  (Hostinger)│
                    └──────┬──────┘
                           │
                           │ 3. Clone/update repository
                           │ 4. Run deploy script
                           ▼
                    ┌─────────────┐
                    │ Deploy      │
                    │ Script      │◄────── scripts/deploy.sh
                    └──────┬──────┘
                           │
                           │ 5. Check port availability
                           │ 6. Build Docker image
                           ▼
                    ┌─────────────┐
                    │   Docker    │
                    │   Build     │
                    └──────┬──────┘
                           │
                           │ 7. Start container
                           ▼
                    ┌─────────────┐
                    │   Running   │
                    │  Container  │◄────── Your app is LIVE!
                    └──────┬──────┘
                           │
                           │ 8. Health check
                           ▼
                    ┌─────────────┐
                    │    ✅       │
                    │  Success!   │
                    └─────────────┘
```

---

## Componenti Principali

### 1. GitHub Repository

**Ruolo:** Contiene il tuo codice sorgente e i file di configurazione.

**File Chiave:**
- `.github/workflows/deploy.yml` - Definisce QUANDO e COME deployare
- `Dockerfile` - Istruzioni per buildare l'immagine Docker
- `docker-compose.yml` - Configurazione container
- `scripts/deploy.sh` - Script di deployment
- Il tuo codice applicazione

### 2. GitHub Actions

**Ruolo:** Esegue automaticamente i task quando avvengono eventi (push, PR, etc).

**Come Funziona:**
- GitHub monitora il repository per eventi
- Quando fai `git push`, GitHub triggera il workflow
- Un "runner" (server virtuale di GitHub) esegue i comandi
- Il runner ha accesso ai tuoi secrets (VPS_HOST, VPS_SSH_KEY, etc)

### 3. SSH Connection

**Ruolo:** Connessione sicura tra GitHub Actions e il tuo VPS.

**Come Funziona:**
- GitHub Actions usa la chiave privata SSH (da secrets)
- Si connette al VPS usando username + chiave
- Esegue comandi remoti sul VPS
- La chiave pubblica è nel file `~/.ssh/authorized_keys` del VPS

### 4. Docker

**Ruolo:** Isola e containerizza la tua applicazione.

**Componenti:**
- **Dockerfile** - Ricetta per creare l'immagine
- **Image** - Snapshot dell'applicazione pronta per essere eseguita
- **Container** - Istanza in esecuzione dell'immagine

---

## Step-by-Step: Cosa Succede Durante un Deploy

### Step 1: Fai un Push

```bash
git add .
git commit -m "Update feature X"
git push origin main
```

**Cosa succede:**
- Git invia il codice a GitHub
- GitHub riceve il push sul branch `main`
- GitHub cerca workflow che hanno trigger `push` su `main`

### Step 2: GitHub Triggera il Workflow

**File:** `.github/workflows/deploy.yml`

```yaml
on:
  push:
    branches:
      - main
      - master
```

**Cosa succede:**
- GitHub trova il workflow `deploy.yml`
- Alloca un runner (Ubuntu latest)
- Legge i job da eseguire

### Step 3: Checkout del Codice

```yaml
- name: Checkout repository
  uses: actions/checkout@v4
  with:
    fetch-depth: 1
```

**Cosa succede:**
- GitHub Actions clona il repository nel runner
- Tutti i file del progetto sono ora disponibili nel runner

### Step 4: Connessione SSH al VPS

```yaml
- name: Deploy to VPS via SSH
  uses: appleboy/ssh-action@v1.0.3
  with:
    host: ${{ secrets.VPS_HOST }}
    username: ${{ secrets.VPS_USERNAME }}
    key: ${{ secrets.VPS_SSH_KEY }}
    port: ${{ secrets.VPS_PORT }}
    script: |
      # Comandi da eseguire sul VPS
```

**Cosa succede:**
1. GitHub Actions legge i secrets (VPS_HOST, VPS_SSH_KEY, etc)
2. Crea una connessione SSH al VPS
3. Autentica usando la chiave privata
4. Esegue i comandi nello script

### Step 5: Clone/Update Repository sul VPS

**Comandi eseguiti sul VPS:**

```bash
DEPLOY_DIR="/opt/docker-deploy-template"

if [ -d ".git" ]; then
  # Repository già esiste, aggiorna
  git fetch origin
  git reset --hard origin/main
else
  # Prima volta, clona
  git clone https://github.com/frede1983/docker-deploy-template .
fi
```

**Cosa succede:**
- Se è il primo deploy, clona il repository in `/opt/docker-deploy-template`
- Se repository già exists, fa un `git pull` per aggiornare

**Perché clona di nuovo?**
- Il codice nel runner di GitHub Actions è temporaneo
- Serve una copia permanente sul VPS
- Questa copia viene usata per il build Docker

### Step 6: Esecuzione Deploy Script

```bash
chmod +x scripts/*.sh
bash scripts/deploy.sh
```

**Cosa succede:**
- Rende eseguibili gli script
- Esegue `deploy.sh` che fa il deployment vero e proprio

### Step 7: Deploy Script - Controllo Porta

**Script:** `scripts/check-port.sh`

```bash
# Controlla se porta 3000 è libera
if lsof -Pi :3000 -sTCP:LISTEN -t >/dev/null 2>&1 ; then
  # Porta occupata, trova una libera
  for port in $(seq 3001 65535); do
    if ! lsof -Pi :$port -sTCP:LISTEN -t >/dev/null 2>&1 ; then
      echo $port
      exit 0
    fi
  done
else
  # Porta 3000 libera, usala
  echo 3000
fi
```

**Cosa succede:**
- Usa `lsof` per controllare se la porta è in uso
- Se libera, usa quella porta
- Se occupata, cerca la prima porta libera da 3001 in su

### Step 8: Creazione File .env

```bash
cat > .env <<EOF
COMPOSE_PROJECT_NAME=docker-deploy-template
APP_PORT=3000
HOST_PORT=3000
NODE_ENV=production
EOF
```

**Cosa succede:**
- Crea/aggiorna file `.env` con configurazione
- Docker Compose legge questo file per le variabili ambiente

### Step 9: Stop Container Esistente

```bash
docker compose down --remove-orphans 2>/dev/null || true
```

**Cosa succede:**
- Ferma e rimuove il container esistente (se presente)
- `--remove-orphans` rimuove container orfani
- `|| true` previene errori se non ci sono container

### Step 10: Build Immagine Docker

```bash
docker compose build --no-cache
```

**Cosa succede:**

1. **Legge Dockerfile:**

```dockerfile
FROM node:22-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --omit=dev

COPY . .

EXPOSE 3000
```

2. **Esegue ogni istruzione:**
   - `FROM` - Scarica immagine base Node 22 Alpine
   - `WORKDIR` - Crea directory `/app` nel container
   - `COPY package*.json` - Copia solo package files
   - `RUN npm ci` - Installa dipendenze
   - `COPY . .` - Copia tutto il codice
   - `EXPOSE` - Dichiara porta 3000

3. **Crea immagine Docker**
   - Ogni step crea un "layer"
   - L'immagine finale è uno stack di layers
   - Layers sono cachati per velocità

### Step 11: Avvio Container

```bash
docker compose up -d
```

**Cosa succede:**

1. **Legge docker-compose.yml:**

```yaml
services:
  app:
    build: .
    container_name: docker-deploy-template
    environment:
      - NODE_ENV=production
      - PORT=3000
    ports:
      - "3000:3000"
```

2. **Crea container:**
   - Usa l'immagine appena buildata
   - Mappa porta host → container
   - Imposta variabili ambiente
   - `-d` = detached (background)

3. **Avvia processo:**
   - Esegue `npm start` (dal Dockerfile CMD)
   - Il processo parte in background

### Step 12: Attesa Health Check

```bash
sleep 5
```

**Cosa succede:**
- Aspetta 5 secondi per dare tempo al container di avviarsi
- L'app deve inizializzare (connessioni DB, etc)

### Step 13: Verifica Container Running

```bash
if docker compose ps | grep -q "Up"; then
  echo "✅ Deployment successful!"
else
  echo "❌ Deployment failed!"
  exit 1
fi
```

**Cosa succede:**
- `docker compose ps` mostra status container
- `grep "Up"` cerca la parola "Up" (container running)
- Se non trova "Up", deployment fallito → exit 1

### Step 14: GitHub Actions Verifica Health

**Di nuovo dal workflow:**

```yaml
- name: Verify deployment
  script: |
    HEALTH_STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/health)

    if [ "$HEALTH_STATUS" = "200" ]; then
      echo "✅ Health check passed"
    else
      echo "❌ Health check failed"
      exit 1
    fi
```

**Cosa succede:**
- Fa una richiesta HTTP a `/health`
- Controlla che risponda con status 200
- Se fallisce, l'intero workflow fallisce

### Step 15: Deployment Completato!

**Output finale:**

```
✅ Deployment successful!
📊 Container status:
NAME                     IMAGE                        STATUS
docker-deploy-template   docker-deploy-template-app   Up 10 seconds (healthy)

🌐 Service available at:
   - http://localhost:3000/health
   - http://localhost:3000/api/info
```

---

## GitHub Actions: Il Cervello

### Workflow File Anatomy

```yaml
name: Deploy to VPS                    # Nome del workflow

on:                                     # QUANDO eseguire
  push:
    branches:
      - main

jobs:                                   # COSA eseguire
  deploy:                               # Nome del job
    runs-on: ubuntu-latest              # Dove eseguire

    steps:                              # Passi da eseguire
      - name: Checkout                  # Step 1
        uses: actions/checkout@v4

      - name: Deploy                    # Step 2
        uses: appleboy/ssh-action@v1.0.3
        with:                           # Parametri
          host: ${{ secrets.VPS_HOST }}
```

### Come GitHub Actions Accede ai Secrets

1. **Salvi secrets su GitHub:**
   - Settings → Secrets and variables → Actions
   - Add secret: `VPS_SSH_KEY` = `[chiave privata]`

2. **Workflow li legge:**
   ```yaml
   key: ${{ secrets.VPS_SSH_KEY }}
   ```

3. **GitHub Actions:**
   - Decripta il secret al runtime
   - Lo passa come variabile ambiente
   - **MAI** lo mostra nei log (viene mascherato con `***`)

### Runner Environment

Il runner è un server Ubuntu con:
- Git installato
- Docker installato
- Node.js/Python/Java/etc disponibili
- Rete pubblica per SSH

**Durata:** Il runner vive solo per la durata del workflow, poi viene distrutto.

---

## SSH: La Connessione Sicura

### Come Funziona l'Autenticazione SSH

1. **Generazione Chiavi (fatto una volta):**

```bash
ssh-keygen -t ed25519 -C "github-actions" -f ~/.ssh/github-actions
```

Crea:
- `github-actions` = chiave **privata** (SECRET, va su GitHub)
- `github-actions.pub` = chiave **pubblica** (va sul VPS)

2. **Setup VPS:**

```bash
cat ~/.ssh/github-actions.pub >> ~/.ssh/authorized_keys
```

Il VPS ora accetta connessioni con quella chiave.

3. **GitHub Actions si connette:**

```yaml
with:
  key: ${{ secrets.VPS_SSH_KEY }}  # Chiave privata
```

SSH fa:
- Cripta un messaggio con la chiave privata
- VPS decripta con la chiave pubblica
- Se OK → connessione stabilita

### Cosa Succede Durante SSH

```yaml
script: |
  cd /opt/docker-deploy-template
  bash scripts/deploy.sh
```

**Sequenza:**
1. SSH connection aperta
2. Esegue `cd /opt/docker-deploy-template`
3. Esegue `bash scripts/deploy.sh`
4. Legge output (stdout/stderr)
5. Se exit code ≠ 0 → workflow fallisce
6. SSH connection chiusa

---

## Docker: L'Isolamento

### Perché Docker?

**Senza Docker:**
```
VPS Server
├── App 1 (Node 14)  ← Conflitto!
├── App 2 (Node 22)  ← Conflitto!
├── App 3 (Python 3.9)
└── Tutte condividono:
    ├── Porte (conflict!)
    ├── Dipendenze (conflict!)
    └── File system
```

**Con Docker:**
```
VPS Server
├── Container 1 (Node 14) ← Isolato
├── Container 2 (Node 22) ← Isolato
└── Container 3 (Python 3.9) ← Isolato

Ogni container ha:
- Proprio filesystem
- Proprie porte (mappate)
- Proprie dipendenze
```

### Build Process

**Dockerfile:**

```dockerfile
FROM node:22-alpine         # Layer 0: Base image

WORKDIR /app                # Layer 1: Create /app dir

COPY package*.json ./       # Layer 2: Copy package files
RUN npm ci --omit=dev       # Layer 3: Install deps

COPY . .                    # Layer 4: Copy app code

CMD ["npm", "start"]        # Metadata: start command
```

**Build:**

```bash
docker compose build
```

Crea uno stack:

```
┌──────────────────────┐
│  Layer 4: App code   │  ← Your app.js
├──────────────────────┤
│  Layer 3: node_modules│ ← npm packages
├──────────────────────┤
│  Layer 2: package.json│
├──────────────────────┤
│  Layer 1: /app dir   │
├──────────────────────┤
│  Layer 0: Node.js    │  ← Base Alpine + Node
└──────────────────────┘
     = Docker Image
```

**Run:**

```bash
docker compose up -d
```

Crea container:

```
        Docker Image
             │
             ▼
     ┌──────────────┐
     │  Container   │  ← Running process
     │  PID 1234    │
     │              │
     │  /app/       │  ← Isolated filesystem
     │  PORT 3000   │  ← Mapped to host:3000
     └──────────────┘
```

### Port Mapping

```yaml
ports:
  - "3000:3000"
    ───┬─── ───┬───
     Host   Container
```

**Cosa significa:**
- Container ascolta su porta 3000 INTERNA
- Host (VPS) mappa porta 3000 ESTERNA → 3000 container
- Request a `http://vps:3000` → rediretto al container

**Con porta dinamica:**

```yaml
ports:
  - "${HOST_PORT}:${APP_PORT}"
```

File `.env`:
```
HOST_PORT=3001  ← VPS porta libera trovata
APP_PORT=3000   ← Container always 3000
```

---

## Scripts: L'Automazione

### check-port.sh - Logica Dettagliata

```bash
#!/bin/bash

DEFAULT_PORT=3000

# Funzione per controllare porta
check_port() {
    local port=$1
    # lsof = List Open Files
    # -Pi :$port = Process using Internet protocol on $port
    # -sTCP:LISTEN = State TCP LISTEN
    # -t = Show only PIDs

    if lsof -Pi :$port -sTCP:LISTEN -t >/dev/null 2>&1 ; then
        return 1  # Porta occupata
    else
        return 0  # Porta libera
    fi
}

# Test porta default
if check_port $DEFAULT_PORT; then
    echo $DEFAULT_PORT  # 3000 libera
    exit 0
else
    # Trova prima porta libera
    for port in $(seq 3001 65535); do
        if check_port $port; then
            echo $port  # es. 3005
            exit 0
        fi
    done
fi
```

**Uso:**

```bash
AVAILABLE_PORT=$(bash scripts/check-port.sh 3000)
echo "Porta da usare: $AVAILABLE_PORT"
```

### deploy.sh - Flow Completo

```bash
#!/bin/bash
set -e  # Exit on any error

# 1. Config
PROJECT_DIR=$(pwd)
DEFAULT_PORT=3000

# 2. Trova porta
echo "🔍 Checking port..."
AVAILABLE_PORT=$(bash scripts/check-port.sh $DEFAULT_PORT)

# 3. Crea .env
echo "📝 Creating .env..."
cat > .env <<EOF
COMPOSE_PROJECT_NAME=my-app
APP_PORT=$AVAILABLE_PORT
HOST_PORT=$AVAILABLE_PORT
NODE_ENV=production
EOF

# 4. Stop old container
echo "🛑 Stopping old container..."
docker compose down --remove-orphans || true

# 5. Build new image
echo "🔨 Building image..."
docker compose build --no-cache

# 6. Start container
echo "▶️  Starting container..."
docker compose up -d

# 7. Wait for startup
echo "⏳ Waiting..."
sleep 5

# 8. Verify
if docker compose ps | grep -q "Up"; then
    echo "✅ Success!"
    docker compose ps
else
    echo "❌ Failed!"
    docker compose logs
    exit 1
fi
```

---

## Esempio Concreto

Scenario: Modifichi la homepage del tuo sito e vuoi deployare.

### 1. Tu Modifichi Codice

```javascript
// app.js
app.get('/', (req, res) => {
  res.json({ message: 'Hello World v2.0!' }); // ← MODIFICATO
});
```

### 2. Commit e Push

```bash
git add app.js
git commit -m "Update homepage message"
git push origin main
```

### 3. GitHub Riceve Push

```
[GitHub Server]
Event: push
Branch: main
Commit: abc123
Author: frede1983
```

### 4. GitHub Triggera Workflow

```
[GitHub Actions]
Loading: .github/workflows/deploy.yml
Trigger: on.push.branches.main ✓
Allocating: Runner ubuntu-latest
Starting: Job 'deploy'
```

### 5. Runner Esegue Steps

```
[Step 1: Checkout]
git clone https://github.com/frede1983/my-app
cd my-app
git checkout main

[Step 2: SSH Connection]
ssh -i [secret_key] root@srv.example.com

[Now on VPS]
cd /opt/my-app
```

### 6. Sul VPS: Update Repository

```bash
[VPS]$ git fetch origin
[VPS]$ git reset --hard origin/main
HEAD is now at abc123 Update homepage message
```

### 7. Sul VPS: Deploy Script

```bash
[VPS]$ bash scripts/deploy.sh

🔍 Checking port availability...
✅ Port 3000 is available
✅ Using port: 3000

📝 Creating .env file...
✅ .env created

🛑 Stopping existing containers...
Stopping my-app ... done

🔨 Building Docker image...
Step 1/6 : FROM node:22-alpine
Step 2/6 : WORKDIR /app
Step 3/6 : COPY package*.json ./
Step 4/6 : RUN npm ci --omit=dev
Step 5/6 : COPY . .
Step 6/6 : CMD ["npm", "start"]
Successfully built image123

▶️  Starting container...
Creating my-app ... done

⏳ Waiting for health check...

✅ Deployment successful!
```

### 8. Container Running

```bash
[VPS]$ docker compose ps

NAME    IMAGE       STATUS
my-app  my-app-img  Up 10 seconds (healthy)
```

### 9. Verifica Health

```bash
[VPS]$ curl http://localhost:3000/
{"message":"Hello World v2.0!"}
```

### 10. GitHub Actions Success

```
[GitHub Actions]
✅ Deploy to VPS via SSH
✅ Verify deployment
   Health: ✅ OK (HTTP 200)

Workflow completed in 45s
```

### 11. Notifica GitHub

```
[GitHub UI]
✓ All checks have passed

Commit abc123
✓ Deploy to VPS (45s)
✓ Test (30s)
```

### 12. Tuo Browser

Visiti il sito:

```
https://my-app.example.com
→ NGINX proxy
→ localhost:3000
→ Docker container
→ {"message":"Hello World v2.0!"}
```

**DEPLOY COMPLETATO!**

---

## Domande Frequenti

### Q: Cosa succede se il build fallisce?

**A:** Il workflow si ferma e non deploya nulla. Il vecchio container continua a girare.

```
Build failed at Step 5
→ Workflow exits with error
→ Old container still running
→ No downtime!
```

### Q: Posso vedere i log in tempo reale?

**A:** Sì, su GitHub Actions:

```bash
gh run watch
```

O sul VPS:

```bash
docker compose logs -f
```

### Q: Quanto tempo ci vuole un deploy?

**A:** Circa 30-60 secondi:
- GitHub trigger: ~2s
- SSH connection: ~3s
- Git pull: ~2s
- Docker build: ~20-40s (dipende da cache)
- Container start: ~5s
- Health check: ~5s

### Q: Posso deployare più app sullo stesso VPS?

**A:** Sì! Ogni app ha:
- Propria directory: `/opt/app1`, `/opt/app2`
- Propria porta: 3000, 3001, ...
- Proprio container isolato

### Q: E se voglio rollback?

**A:** Due modi:

1. **Git revert:**
```bash
git revert HEAD
git push
# Auto-deploya versione precedente
```

2. **Manuale sul VPS:**
```bash
cd /opt/my-app
git checkout [old-commit]
bash scripts/deploy.sh
```

---

## Prossimi Passi

Ora che capisci come funziona, puoi:

1. [Configurare il tuo primo progetto](SETUP.md)
2. [Personalizzare il deployment](WORKFLOWS.md)
3. [Aggiungere più servizi](ARCHITECTURE.md)
4. [Ottimizzare le performance](BEST_PRACTICES.md)

---

**Hai domande?** Apri un issue su GitHub!
