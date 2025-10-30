# Setup Completo - Guida Passo-Passo

Questa guida ti porta da zero a un sistema funzionante di auto-deployment.

## Prerequisiti

- Un VPS con accesso root (es. Hostinger, DigitalOcean, AWS)
- Account GitHub
- Git installato localmente
- GitHub CLI (`gh`) opzionale ma consigliato

## Step 1: Preparazione VPS

### 1.1 Installa Docker

```bash
# Connettiti al VPS
ssh root@your-vps.com

# Installa Docker
curl -fsSL https://get.docker.com | sh

# Aggiungi user al gruppo docker (opzionale se usi root)
sudo usermod -aG docker $USER

# Verifica installazione
docker --version
docker compose version
```

**Output atteso:**
```
Docker version 28.4.0
Docker Compose version v2.39.2
```

### 1.2 Installa Strumenti Necessari

```bash
# Git (dovrebbe già esserci)
apt install git -y

# lsof (per check porte)
apt install lsof -y

# curl (per health checks)
apt install curl -y
```

### 1.3 Crea Chiave SSH per GitHub Actions

```bash
# Genera chiave dedicata
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/github-actions -N ""

# Aggiungi chiave pubblica alle authorized_keys
cat ~/.ssh/github-actions.pub >> ~/.ssh/authorized_keys

# Imposta permessi corretti
chmod 600 ~/.ssh/github-actions
chmod 644 ~/.ssh/github-actions.pub
chmod 600 ~/.ssh/authorized_keys
```

### 1.4 Salva Chiave Privata

```bash
# Mostra chiave privata
cat ~/.ssh/github-actions
```

**COPIA TUTTO L'OUTPUT**, incluso:
- `-----BEGIN OPENSSH PRIVATE KEY-----`
- Il contenuto
- `-----END OPENSSH PRIVATE KEY-----`

La userai come secret `VPS_SSH_KEY` su GitHub.

### 1.5 Test Connessione SSH

```bash
# Da un altro terminale, testa la connessione
ssh -i ~/.ssh/github-actions root@your-vps.com

# Se funziona, sei pronto!
```

## Step 2: Crea Progetto da Template

### 2.1 Clona Template

```bash
# Sul tuo computer locale
git clone https://github.com/frede1983/docker-deploy-template my-project
cd my-project
```

### 2.2 Personalizza Progetto

Modifica `package.json`:

```json
{
  "name": "my-project",
  "version": "1.0.0",
  "description": "Il mio progetto con auto-deploy",
  ...
}
```

Modifica `app.js` con la tua applicazione.

### 2.3 Test Locale (Opzionale)

```bash
# Installa dipendenze
npm install

# Testa Docker build
docker compose build

# Avvia container
docker compose up

# Testa endpoint (altro terminale)
curl http://localhost:3000/health
```

Se tutto funziona, stop il container:

```bash
docker compose down
```

## Step 3: Crea Repository su GitHub

### 3.1 Con GitHub CLI (Raccomandato)

```bash
# Autenticati (prima volta)
gh auth login

# Inizializza git
git init
git add .
git commit -m "Initial commit"

# Crea repository su GitHub
gh repo create my-project --private --source . --push
```

### 3.2 Manualmente

1. Vai su https://github.com/new
2. Nome repository: `my-project`
3. Visibilità: Private o Public
4. NO README, NO .gitignore (già presenti)
5. Create repository

Poi localmente:

```bash
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/my-project.git
git push -u origin main
```

## Step 4: Configura GitHub Secrets

### 4.1 Vai su GitHub Repository Settings

```
https://github.com/YOUR_USERNAME/my-project/settings/secrets/actions
```

### 4.2 Aggiungi Secrets

Click "New repository secret" per ognuno:

#### Secret 1: VPS_HOST

**Nome:** `VPS_HOST`
**Valore:** `your-vps.com` (hostname o IP del VPS)

Esempio:
```
srv1013438.hstgr.cloud
```
oppure
```
123.45.67.89
```

#### Secret 2: VPS_USERNAME

**Nome:** `VPS_USERNAME`
**Valore:** `root` (o altro username)

#### Secret 3: VPS_PORT

**Nome:** `VPS_PORT`
**Valore:** `22` (porta SSH)

#### Secret 4: VPS_SSH_KEY

**Nome:** `VPS_SSH_KEY`
**Valore:** La chiave privata copiata prima (tutto incluso header/footer)

**Esempio:**
```
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
...
(molte righe)
...
AAAAB3NzaC1yc2EAAAADAQABAAABAQC3...
-----END OPENSSH PRIVATE KEY-----
```

**IMPORTANTE:**
- Deve includere TUTTO, dall'header al footer
- Non aggiungere spazi extra
- Non modificare il formato

### 4.3 Verifica Secrets

Dovresti vedere:

```
VPS_HOST       Updated now
VPS_USERNAME   Updated now
VPS_PORT       Updated now
VPS_SSH_KEY    Updated now
```

## Step 5: Primo Deploy

### 5.1 Trigger Deploy

Fai un piccolo cambiamento:

```bash
# Modifica README
echo "\n## Status\nDeploy in corso..." >> README.md

# Commit e push
git add README.md
git commit -m "Test auto-deploy"
git push
```

### 5.2 Monitora GitHub Actions

#### Opzione A: Web UI

1. Vai su https://github.com/YOUR_USERNAME/my-project/actions
2. Vedrai il workflow "Deploy to VPS" running
3. Click sul workflow per vedere dettagli
4. Click sul job "deploy" per vedere logs

#### Opzione B: CLI

```bash
# Watch workflow in tempo reale
gh run watch

# O lista runs
gh run list
```

### 5.3 Cosa Aspettarsi

**Workflow Steps:**
```
✓ Set up job (5s)
✓ Checkout repository (3s)
⏳ Deploy to VPS via SSH (30-45s)
  🚀 Starting deployment
  📥 Cloning repository
  🔍 Checking port
  🔨 Building Docker image
  ▶️  Starting container
  ✅ Deployment successful
✓ Verify deployment (5s)
  ✅ Container is running
  ✅ Health check passed
```

**Tempo totale:** ~45-60 secondi

### 5.4 Verifica sul VPS

```bash
# SSH al VPS
ssh root@your-vps.com

# Controlla container
cd /opt/my-project
docker compose ps

# Dovresti vedere:
# NAME         STATUS
# my-project   Up X seconds (healthy)

# Test endpoint
curl http://localhost:3000/health

# Output atteso:
# {"status":"healthy","timestamp":"...","service":"my-project"}
```

## Step 6: Setup Opzionali

### 6.1 Aggiungi NGINX Reverse Proxy (Raccomandato)

Se vuoi esporre l'app su dominio pubblico:

```bash
# Installa NGINX
apt install nginx -y

# Crea config
nano /etc/nginx/sites-available/my-project
```

Contenuto:

```nginx
server {
    listen 80;
    server_name my-project.your-domain.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Attiva:

```bash
ln -s /etc/nginx/sites-available/my-project /etc/nginx/sites-enabled/
nginx -t
systemctl reload nginx
```

### 6.2 Aggiungi SSL con Certbot

```bash
# Installa Certbot
apt install certbot python3-certbot-nginx -y

# Ottieni certificato
certbot --nginx -d my-project.your-domain.com

# Auto-renewal già configurato
```

### 6.3 Installa GitHub App (per @claude support)

Se vuoi assistenza automatica da Claude:

```bash
cd /opt/my-project
claude github install-app
```

Segui le istruzioni per autorizzare l'app.

## Step 7: Workflow Quotidiano

### 7.1 Sviluppo Normale

```bash
# Fai modifiche al codice
vim app.js

# Test locale (opzionale)
npm start

# Commit e push
git add .
git commit -m "Add new feature"
git push

# GitHub Actions deploya automaticamente!
```

### 7.2 Monitora Deploy

```bash
# Watch logs in tempo reale
gh run watch

# O sul VPS
ssh root@your-vps.com
docker compose logs -f
```

### 7.3 Verifica Deployment

```bash
# Test health
curl https://my-project.your-domain.com/health

# O con porta diretta
curl http://your-vps.com:3000/health
```

## Troubleshooting Setup

### Problema 1: GitHub Actions fallisce con "Permission denied (publickey)"

**Causa:** Chiave SSH non corretta o non autorizzata.

**Soluzione:**

```bash
# Sul VPS, verifica chiave pubblica
cat ~/.ssh/authorized_keys | grep github-actions

# Se non c'è, ri-aggiungi
cat ~/.ssh/github-actions.pub >> ~/.ssh/authorized_keys

# Verifica permessi
chmod 600 ~/.ssh/authorized_keys

# Test connessione locale
ssh -i ~/.ssh/github-actions root@localhost
```

Se connessione locale funziona, problema è nel secret GitHub:

1. Ri-copia chiave privata: `cat ~/.ssh/github-actions`
2. Update secret `VPS_SSH_KEY` su GitHub
3. Assicurati di copiare TUTTO incluso header/footer

### Problema 2: "docker: command not found"

**Causa:** Docker non installato o utente non nel gruppo docker.

**Soluzione:**

```bash
# Verifica Docker
docker --version

# Se non installato
curl -fsSL https://get.docker.com | sh

# Aggiungi user al gruppo (se non usi root)
sudo usermod -aG docker $USER

# Logout e login per applicare
```

### Problema 3: "port already in use"

**Causa:** Porta già occupata da altro servizio.

**Soluzione:** Lo script trova automaticamente una porta libera!

```bash
# Sul VPS, controlla quale porta è stata usata
cd /opt/my-project
cat .env | grep HOST_PORT

# Se vuoi cambiare porta default
vim scripts/deploy.sh
# Modifica: DEFAULT_PORT=8080
```

### Problema 4: Build fallisce con "npm ERR!"

**Causa:** Problema con dipendenze npm.

**Soluzione:**

```bash
# Verifica package-lock.json esista
ls -la package-lock.json

# Se non esiste, generalo
npm install
git add package-lock.json
git commit -m "Add package-lock.json"
git push

# Verifica Dockerfile usa npm ci
grep "npm ci" Dockerfile
# Deve essere: RUN npm ci --omit=dev
```

### Problema 5: Container non parte (health check fail)

**Causa:** App ha errori o non risponde su porta corretta.

**Soluzione:**

```bash
# Sul VPS, controlla logs
cd /opt/my-project
docker compose logs

# Test build locale
docker compose build
docker compose up

# Guarda errori
```

Errori comuni:
- App ascolta su porta sbagliata → verifica `PORT` in app.js
- Dependency mancante → verifica package.json
- Errore di sintassi → controlla app.js

## Next Steps

Ora che hai setup funzionante:

1. [Leggi come funziona il sistema](HOW_IT_WORKS.md)
2. [Personalizza workflows](WORKFLOWS.md)
3. [Aggiungi più servizi](ARCHITECTURE.md)
4. [Best practices](BEST_PRACTICES.md)

## Checklist Setup

Usa questa checklist per verificare che tutto sia configurato:

### VPS
- [ ] Docker installato e funzionante
- [ ] Chiave SSH generata
- [ ] Chiave pubblica in authorized_keys
- [ ] lsof, curl, git installati
- [ ] Connessione SSH testata

### GitHub
- [ ] Repository creato
- [ ] File `.github/workflows/deploy.yml` presente
- [ ] Secret `VPS_HOST` configurato
- [ ] Secret `VPS_USERNAME` configurato
- [ ] Secret `VPS_PORT` configurato
- [ ] Secret `VPS_SSH_KEY` configurato (con header/footer)

### Test
- [ ] Primo push eseguito
- [ ] GitHub Actions run completato con successo
- [ ] Container running sul VPS
- [ ] Health check risponde 200
- [ ] Logs accessibili

### Opzionale
- [ ] NGINX configurato
- [ ] SSL/HTTPS abilitato
- [ ] Dominio configurato
- [ ] GitHub App installata
- [ ] Monitoring setup

---

**Setup completato!** 🎉

Ora ogni push deploya automaticamente sul VPS.
