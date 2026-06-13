# 🐳 Production Deployment Guide — Docker + VPS + Nginx + CI/CD

**A reusable, in-depth reference template for deploying any multi-service application**

[![Docker](https://img.shields.io/badge/Docker-multi--stage-blue?logo=docker)](https://www.docker.com/)
[![CI/CD](https://img.shields.io/badge/CI%2FCD-GitHub%20Actions-2088FF?logo=githubactions)](https://github.com/features/actions)
[![Nginx](https://img.shields.io/badge/Reverse%20Proxy-Nginx-009639?logo=nginx)](https://nginx.org/)
[![Postgres](https://img.shields.io/badge/Database-PostgreSQL-336791?logo=postgresql)](https://www.postgresql.org/)
[![Redis](https://img.shields.io/badge/Cache-Redis-DC382D?logo=redis)](https://redis.io/)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)

---

## 📋 Table of Contents

- [The Big Picture](#-the-big-picture)
- [Core Concepts (Read This First)](#-core-concepts-read-this-first)
- [Phase 1 — Dockerize Locally](#-phase-1--dockerize-locally)
- [Phase 2 — VPS Setup](#-phase-2--vps-setup)
- [Phase 3 — Nginx Reverse Proxy + SSL](#-phase-3--nginx-reverse-proxy--ssl)
- [Phase 4 — GitHub Actions CI/CD](#-phase-4--github-actions-cicd)
- [Production docker-compose.prod.yml (Template)](#-production-docker-composeprodyml-template)
- [Container Management Cheat Sheet](#-container-management-cheat-sheet)
- [Common Issues & Fixes](#-common-issues--fixes)
- [Adapting This Template](#-adapting-this-template)
- [License](#-license)

---

## 🎯 The Big Picture

No matter what your stack is (Express, NestJS, Next.js, Django, Laravel...), production deployment always follows the **same shape**:

```
                     ┌──────────────────────────┐
                     │   git push origin main     │
                     └─────────────┬─────────────┘
                                    │
                                    ▼
                     ┌──────────────────────────┐
                     │     GitHub Actions CI/CD   │
                     │  1. Run tests               │
                     │  2. Build Docker image(s)   │
                     │  3. Push to Docker Hub      │
                     │  4. SSH into VPS & deploy   │
                     └─────────────┬─────────────┘
                                    │
                                    ▼
┌───────────────────────────── VPS (Ubuntu) ─────────────────────────────┐
│                                                                            │
│   ┌──────────┐   ┌──────────┐   ┌────────────┐   ┌────────────┐         │
│   │ database  │   │  cache    │   │  backend    │   │  frontend   │  ...  │
│   │ (postgres │◄──│ (redis)   │◄──│  (api)      │   │  (web app)  │         │
│   │  /mongo)  │   │           │   │             │   │             │         │
│   └──────────┘   └──────────┘   └─────┬──────┘   └─────┬──────┘         │
│                                         │                 │                 │
│                              app_network (bridge)                          │
└──────────────────────────────────────┬──┴─────────────────┴───────────────┘
                                       │
                                       ▼
                              ┌──────────────────┐
                              │  Nginx (reverse    │
                              │  proxy) + SSL       │
                              └─────────┬─────────┘
                                       │
                    ┌────────────────────┼────────────────────┐
                    ▼                    ▼                    ▼
            yourdomain.com      api.yourdomain.com    admin.yourdomain.com
```

**Why this shape, always:**

- **Docker** = your app + its exact dependencies, packaged once, runs identically everywhere (no "works on my machine").
- **Docker Hub (or any registry)** = the handoff point between CI and your server — the VPS never needs your source code, only the built image.
- **VPS** = a plain Ubuntu machine that only knows how to run containers and route traffic — it stays "dumb" on purpose.
- **Nginx** = the single entry point on ports 80/443 that decides *which container* handles a request based on the domain name (this is called **virtual hosting**).
- **GitHub Actions** = the automation that connects all of the above so `git push` → live site, with zero manual steps.

---

## 🧠 Core Concepts (Read This First)

These ideas apply to **every** project, regardless of stack. Understanding them prevents 90% of deployment bugs.

### 1. Build-time vs. Runtime Configuration

There are two completely different moments when "configuration" can enter your app:

| | **Build time** | **Runtime** |
|---|---|---|
| When | While `docker build` runs | While the container is running |
| How | `ARG` + `ENV` in Dockerfile, `--build-arg` | `env_file` / `environment` in compose |
| Used for | Frontend public vars (`NEXT_PUBLIC_*`, `VITE_*`), values compiled into JS bundles | Database URLs, secrets, API keys, anything a *backend process* reads at startup |
| Mistake | Setting `NEXT_PUBLIC_API_URL` only in `.env` on the VPS → frontend still shows `undefined`, because it was already baked into the JS bundle during build | Hardcoding secrets into the Dockerfile → leaks into the image layers, visible to anyone who pulls it |

**Rule of thumb:** if a frontend framework needs to expose a variable to the *browser*, it must be a build-arg. If a backend process reads `process.env.X` at startup, it can be a runtime `.env` variable.

### 2. The "No Code on the Server" Principle

Your VPS should contain **only**:
- `.env` files (secrets, never in git)
- `docker-compose.prod.yml`
- persistent data folders (`uploads/`, database volumes)

Everything else — your actual application code — exists only as a **Docker image** on a registry. This means:
- Your server is disposable. If it dies, a new VPS + these few files + `docker compose up` brings everything back.
- There's no `git pull` on the server, no `npm install` on the server, no version drift between "what's in the repo" and "what's running."

### 3. Health Checks and Startup Order

Containers don't start in a useful order by default — `depends_on` alone only waits for the container *process* to start, not for the database to be *ready to accept connections*. This causes "connection refused" crash loops on cold boot (especially after a VPS reboot).

The fix is `depends_on` + `condition: service_healthy`, combined with a `healthcheck` block on the dependency:

```yaml
backend:
  depends_on:
    database:
      condition: service_healthy
    cache:
      condition: service_healthy

database:
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U $${DB_USER}"]
    interval: 10s
    timeout: 5s
    retries: 5
    start_period: 10s
```

`start_period` gives the database extra grace time on first boot (e.g. initializing the data directory) before failed health checks count against `retries`.

### 4. One Reverse Proxy, Many Domains (Virtual Hosting)

A single VPS has one IP address but can serve unlimited domains/subdomains, because Nginx reads the `Host` header of every incoming request and routes accordingly:

```
Request to yourdomain.com       → Nginx → localhost:3000 (frontend container)
Request to api.yourdomain.com   → Nginx → localhost:8000 (backend container)
Request to admin.yourdomain.com → Nginx → localhost:3001 (admin container)
```

This is why each container exposes a different host port, and why each domain gets its own Nginx server block.

### 5. Image Tags = Your Rollback Mechanism

Tagging images with both `latest` and the git commit SHA (e.g. `myapp:a1b2c3d`) means:
- `latest` = "what's currently deployed"
- `a1b2c3d` = "exactly this commit's build, forever"

If a deploy breaks production, rollback is just:

```bash
docker pull myregistry/myapp:a1b2c3d   # previous good SHA
docker compose up -d --no-deps backend
```

No rebuild needed — the old image is already built and pushed.

---

## 🏗 Phase 1 — Dockerize Locally

### Generic Multi-Stage Dockerfile Pattern

Multi-stage builds keep your final image small by separating "things needed to build" from "things needed to run."

```dockerfile
# Stage 1 — Dependencies
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

# Stage 2 — Build
FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build
RUN npm prune --omit=dev   # remove devDependencies after build

# Stage 3 — Production runtime (small final image)
FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production

# Non-root user — never run containers as root in production
RUN addgroup --system --gid 1001 appgroup \
    && adduser --system --uid 1001 appuser

COPY --from=builder /app/package.json ./package.json
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist

USER appuser
EXPOSE 8000
CMD ["node", "dist/server.js"]
```

**Why each stage exists:**
- `deps` — caches `npm ci` separately, so editing source code doesn't force a full dependency reinstall (Docker layer caching).
- `builder` — compiles TypeScript/Next.js/etc, then strips devDependencies so they don't bloat the final image.
- `runner` — copies only the compiled output + production `node_modules`, runs as a non-root user (`appuser`), and is the smallest possible image.

### Next.js-Specific Note

If deploying a Next.js app, add this to `next.config.ts` **before** writing the Dockerfile — without it, the standalone server file won't exist and the container will fail to start:

```typescript
const nextConfig: NextConfig = {
  output: 'standalone',  // REQUIRED for Docker
};
```

Then in the builder stage, pass public env vars as build args (see [Core Concepts #1](#1-build-time-vs-runtime-configuration)):

```dockerfile
ARG NEXT_PUBLIC_API_URL
ENV NEXT_PUBLIC_API_URL=$NEXT_PUBLIC_API_URL

ARG NEXTAUTH_URL
ENV NEXTAUTH_URL=$NEXTAUTH_URL

RUN npm run build
```

And copy the standalone output in the runner stage:

```dockerfile
COPY --from=builder /app/public ./public
COPY --from=builder --chown=appuser:appgroup /app/.next/standalone ./
COPY --from=builder --chown=appuser:appgroup /app/.next/static ./.next/static
CMD ["node", "server.js"]
```

### `.dockerignore` (Always Include)

```
node_modules
.next
dist
.env
.env.*
.git
.gitignore
*.md
```

This prevents secrets, build artifacts, and unnecessary files from being copied into the image — keeping it small and secure.

### Local Dev Compose File (`docker-compose.yml`)

For local development, use `build:` (not `image:`) so changes to your code are reflected on rebuild:

```yaml
services:
  app:
    build:
      context: .
      args:
        NEXT_PUBLIC_API_URL: http://localhost:8000/api/v1
    container_name: <project>_app
    env_file:
      - .env
    ports:
      - "3000:3000"
```

### Build and Push to a Registry

```bash
docker login

docker build \
  --build-arg NEXT_PUBLIC_API_URL=https://api.<yourdomain.com>/api/v1 \
  --build-arg NEXTAUTH_URL=https://<yourdomain.com> \
  -t <username>/<your-image-name>:latest .

docker push <username>/<your-image-name>:latest
```

---

## 🖥 Phase 2 — VPS Setup

### Step 1 — Connect and Update

```bash
ssh root@<your_vps_ip>
sudo apt update && sudo apt upgrade -y
sudo apt install curl git -y
```

### Step 2 — Install Docker

```bash
curl -fsSL https://get.docker.com | sh
sudo apt install docker-compose-plugin -y
docker -v && docker compose version
```

> Docker's `restart: unless-stopped` policy replaces tools like PM2's `pm2 startup` — containers auto-restart on crash or server reboot, no extra setup needed.

### Step 3 — Directory Structure

The general pattern, regardless of how many services you have:

```
/var/www/
├── docker-compose.prod.yml      ← ONE file manages ALL containers
├── <service-1>/
│   ├── .env                     ← runtime secrets for this service
│   └── uploads/                 ← persistent files (if any)
├── <service-2>/
│   └── .env
└── <service-3>/
    └── .env
```

```bash
mkdir -p /var/www/<service-1>/uploads
mkdir -p /var/www/<service-2>
mkdir -p /var/www/<service-3>
```

### Step 4 — Create `.env` Files

Each service that runs as a **backend process** (reads `process.env` at runtime) gets its own `.env` here. Created once, manually, and never touched by CI/CD again.

```bash
nano /var/www/<service-1>/.env
```

```env
NODE_ENV=production
PORT=8000
DATABASE_URL=postgresql://<user>:<password>@<db-container-name>:5432/<db-name>
REDIS_URL=redis://:<password>@<cache-container-name>:6379
JWT_SECRET=<your_secret>
FRONTEND_URL=https://<yourdomain.com>
```

> Note: `NEXT_PUBLIC_*` and `NEXTAUTH_URL` do **not** go here — they were already baked into the image during build (Phase 1).

### Step 5 — Login to Registry and Pull Images

```bash
docker login
cd /var/www
docker compose -f docker-compose.prod.yml pull
docker compose -f docker-compose.prod.yml up -d
docker ps
docker logs <container-name>
```

---

## 🌐 Phase 3 — Nginx Reverse Proxy + SSL

### Why Nginx Sits in Front

- Terminates SSL (HTTPS) so your app containers only deal with plain HTTP internally
- Routes multiple domains/subdomains to different container ports (virtual hosting)
- Handles large file uploads, WebSocket upgrades, and request buffering

### Install

```bash
sudo apt install nginx -y
```

### Generic Server Block Template

Create one file per domain in `/etc/nginx/sites-available/`:

```nginx
server {
    listen 80;
    server_name <subdomain>.<yourdomain.com>;
    client_max_body_size 100M;   # increase if you accept large file uploads

    location / {
        proxy_pass http://localhost:<container_port>;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

**What each directive does:**
- `server_name` → which domain this block handles (Nginx matches incoming `Host` headers against this)
- `proxy_pass` → forwards the request to the container's exposed port on `localhost`
- `Upgrade` / `Connection 'upgrade'` → required for WebSocket support (Socket.IO, live reload, etc.) — without these, WebSocket connections fail with a 502
- `client_max_body_size` → Nginx's default is 1MB; raise this if your app accepts file uploads

### Enable Sites

```bash
sudo ln -s /etc/nginx/sites-available/<service> /etc/nginx/sites-enabled/
# repeat for each service/domain

sudo nginx -t            # always test config before reloading
sudo systemctl restart nginx
```

### SSL with Let's Encrypt

```bash
sudo apt install certbot python3-certbot-nginx -y
certbot --nginx -d <subdomain>.<yourdomain.com>
# repeat for each domain, or combine: certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

Certbot automatically edits your Nginx config to add SSL and redirect HTTP → HTTPS.

### Auto-Renewal

Let's Encrypt certificates expire every 90 days. Automate renewal:

```bash
sudo apt install cron -y
sudo systemctl enable cron && sudo systemctl start cron
crontab -e
```

Add this line:

```
0 3 * * * certbot renew --quiet
```

### Required DNS Records (Before Running Certbot)

| Type | Name | Value | Notes |
|---|---|---|---|
| A | `@` | `<VPS_IP>` | Root domain |
| A | `www` | `<VPS_IP>` | If serving `www.` |
| A | `<subdomain>` | `<VPS_IP>` | One per subdomain (api, admin, etc.) |

> Certbot fails with `no valid A records found` if any of these are missing. DNS changes can take 5–30 minutes to propagate.

---

## ⚡ Phase 4 — GitHub Actions CI/CD

### The Pipeline Shape

```
┌───────┐    ┌──────────────────┐    ┌──────────┐
│  test  │ →  │ build & push image │ →  │  deploy   │
└───────┘    └──────────────────┘    └──────────┘
```

### Step 1 — Generate an SSH Key (on Your Local Machine)

```bash
ssh-keygen -t ed25519 -C "you@example.com"
cat ~/.ssh/id_ed25519.pub   # → add to VPS
cat ~/.ssh/id_ed25519       # → add to GitHub Secrets (private key)
```

```bash
# On the VPS:
nano ~/.ssh/authorized_keys
# paste the public key content
```

### Step 2 — Add GitHub Repository Secrets

Go to: **GitHub repo → Settings → Secrets and variables → Actions**

| Secret | Value | Why |
|---|---|---|
| `VPS_HOST` | Run `curl -4 ifconfig.me` on the VPS | Must be IPv4 — GitHub Actions can't reach IPv6-only hosts, causing `i/o timeout` |
| `VPS_USER` | usually `root` | SSH login user |
| `SSH_KEY` | full private key content (incl. `BEGIN`/`END` lines) | Authenticates the SSH connection |
| `DOCKERHUB_USERNAME` | your registry username | For `docker login` in CI |
| `DOCKERHUB_TOKEN` | a registry access token (not your password) | Safer than password, revocable |
| *(per service)* `DB_*`, `REDIS_*` | test database credentials | Used only by ephemeral test containers in CI, not production |

### Step 3 — Generic Workflow File

`.github/workflows/deploy.yml`:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  DOCKER_IMAGE: <username>/<your-image-name>

jobs:
  # ── Stage 1: Test ──────────────────────────────────────
  build:
    runs-on: ubuntu-latest

    services:
      database:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: ${{ secrets.DB_USER }}
          POSTGRES_PASSWORD: ${{ secrets.DB_PASSWORD }}
          POSTGRES_DB: ${{ secrets.DB_NAME }}
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      cache:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci
      - run: npm run build
        env:
          DATABASE_URL: postgresql://${{ secrets.DB_USER }}:${{ secrets.DB_PASSWORD }}@localhost:5432/${{ secrets.DB_NAME }}
          REDIS_URL: redis://localhost:6379

  # ── Stage 2: Build & Push Image ────────────────────────
  build-and-push-docker:
    needs: build
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - id: sha
        run: echo "sha_short=$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT

      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          # Build args go here for frontend projects (see Core Concepts #1)
          # build-args: |
          #   NEXT_PUBLIC_API_URL=https://api.<yourdomain.com>/api/v1
          #   NEXTAUTH_URL=https://<yourdomain.com>
          tags: |
            ${{ env.DOCKER_IMAGE }}:latest
            ${{ env.DOCKER_IMAGE }}:${{ steps.sha.outputs.sha_short }}

  # ── Stage 3: Deploy ─────────────────────────────────────
  deploy:
    needs: build-and-push-docker
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest

    steps:
      - uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.SSH_KEY }}
          script: |
            cd /var/www
            docker compose -f docker-compose.prod.yml pull
            docker compose -f docker-compose.prod.yml up -d --no-deps <service-name>
            docker image prune -f
```

**What each stage actually buys you:**
- **Stage 1 (test)** catches broken code *before* anything is built or deployed — fast feedback, no wasted Docker builds.
- **Stage 2 (build & push)** only runs on `main` (not PRs) and produces a deployable artifact tagged twice — once for "current" (`latest`) and once for "this exact commit forever" (SHA).
- **Stage 3 (deploy)** is intentionally minimal — it doesn't rebuild anything, just pulls the already-built image and restarts one container. This keeps deploys fast (seconds, not minutes) and means the VPS never runs `npm install` or `npm run build`.

---

## 📦 Production docker-compose.prod.yml (Template)

This lives at `/var/www/docker-compose.prod.yml` — **never in any git repo**. Copy and adapt the services you need.

```yaml
services:
  # ── Database (optional — remove if not used) ──────────
  database:
    image: postgres:16-alpine   # or mysql, mongo, etc.
    container_name: <project>_database
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
    volumes:
      - db_data:/var/lib/postgresql/data
    networks:
      - app_network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s

  # ── Cache (optional — remove if not used) ──────────────
  cache:
    image: redis:7-alpine
    container_name: <project>_cache
    restart: unless-stopped
    command: redis-server --requirepass ${REDIS_PASSWORD} --appendonly yes
    volumes:
      - cache_data:/data
    networks:
      - app_network
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ── Backend / API ───────────────────────────────────────
  backend:
    image: <username>/<backend-image>:latest
    container_name: <project>_backend
    restart: unless-stopped
    env_file:
      - /var/www/backend/.env
    ports:
      - "8000:8000"
    volumes:
      - /var/www/backend/uploads:/app/uploads   # if you handle file uploads
    depends_on:
      database:
        condition: service_healthy
      cache:
        condition: service_healthy
    networks:
      - app_network

  # ── Frontend (repeat block for admin, etc.) ─────────────
  frontend:
    image: <username>/<frontend-image>:latest
    container_name: <project>_frontend
    restart: unless-stopped
    env_file:
      - /var/www/frontend/.env
    ports:
      - "3000:3000"
    depends_on:
      - backend
    networks:
      - app_network

volumes:
  db_data:
    driver: local
  cache_data:
    driver: local

networks:
  app_network:
    driver: bridge
```

**Adapting this:**
- No database? Delete the `database` service, its volume, and any `depends_on` references to it.
- More frontends (e.g. admin panel, marketing site)? Copy the `frontend` block, rename, change the host port (`3001:3001`, etc.), and add a matching Nginx server block.
- Different database (MongoDB instead of Postgres)? Swap the `image:`, environment variable names, and `healthcheck.test` command (e.g. `mongosh --eval "db.adminCommand('ping')"`).

---

## 🔧 Container Management Cheat Sheet

```bash
# Status
docker ps

# Start / restart everything
docker compose -f /var/www/docker-compose.prod.yml up -d

# Restart ONE service (without affecting others)
docker compose -f /var/www/docker-compose.prod.yml up -d --no-deps <service-name>

# Quick restart without pulling a new image
docker compose -f /var/www/docker-compose.prod.yml restart <service-name>

# Logs
docker logs <container-name> --tail 50
docker logs <container-name> -f          # follow live

# Cleanup unused images (run after every deploy)
docker image prune -f
```

| If you're used to PM2 | Docker equivalent |
|---|---|
| `pm2 list` | `docker ps` |
| `pm2 logs api` | `docker logs <container-name>` |
| `pm2 logs api -f` | `docker logs <container-name> -f` |
| `pm2 restart api` | `docker compose restart <service-name>` |
| `pm2 startup` | `restart: unless-stopped` (already in compose) |

---

## 🐞 Common Issues & Fixes

| Symptom | Cause | Fix |
|---|---|---|
| Frontend API calls hit `.../undefined/...` | `NEXT_PUBLIC_*` set only in runtime `.env`, not at build time | Pass as `--build-arg` / workflow `build-args`, rebuild image |
| CORS error on one subdomain but not another | Backend CORS only allows one origin | Add all frontend domains (e.g. `FRONTEND_URL`, `ADMIN_URL`) to the CORS `origin` array |
| `no valid A records found` (Certbot) | Missing DNS record for that (sub)domain | Add the `A` record, wait for propagation, retry |
| GitHub Actions: `dial tcp ***:22: i/o timeout` | `VPS_HOST` secret is an IPv6 address | Use `curl -4 ifconfig.me` to get the IPv4 address |
| Login redirects to `localhost` in production | `NEXTAUTH_URL` baked in from local `.env` during build | Pass `NEXTAUTH_URL` as a build-arg pointing to the production domain |
| Backend container restarts repeatedly on VPS reboot | Database/cache not ready when backend starts | Add `healthcheck` + `depends_on: condition: service_healthy` |
| `the attribute 'version' is obsolete` warning | Old Compose v3.x syntax | Remove the top-level `version:` key — not needed in Compose v2 |
| 502 Bad Gateway on WebSocket connections | Missing `Upgrade`/`Connection` headers in Nginx | Add `proxy_set_header Upgrade $http_upgrade;` and `proxy_set_header Connection 'upgrade';` |
| `failed to connect to docker API` (Windows local) | Docker Desktop not running | Start Docker Desktop, wait for the whale icon to stop animating |

---

## 🧩 Adapting This Template

When starting a **new project**, walk through this checklist:

1. **List your services** — database? cache? how many frontends/backends?
2. **Write a Dockerfile per service** using the [multi-stage pattern](#-phase-1--dockerize-locally)
3. **Decide build-time vs runtime vars** for each service (see [Core Concepts #1](#1-build-time-vs-runtime-configuration))
4. **Copy the `docker-compose.prod.yml` template**, add/remove services, fix image names and ports
5. **Write one Nginx server block per public domain/subdomain**
6. **Run Certbot** for each domain
7. **Copy the GitHub Actions workflow**, update secrets and the deploy script's `--no-deps <service-name>`
8. **First deploy is manual** (`docker compose up -d` on the VPS) — after that, CI/CD takes over

> The structure never changes. Only the names, ports, number of services, and domains do. Once you've done this once, every future project is mostly find-and-replace.

---

## 📄 License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.

**Use freely as a personal/team reference template.**

---

<p align="center">
  Made with ❤️ for developers who want production-grade deployments, every time.
</p>