# Workbench Redis — Single, Reusable Redis + Web UI (One-Compose Setup)

A production-grade **developer Redis** you can spin up once and reuse across many projects — paired with a **minimal admin UI** (redis-commander). Everything runs from a **single root `docker-compose.yml`**, with clean localhost bindings and sane defaults.

> Goal: Clone → `docker compose build` → `docker compose up -d` → manage at `http://localhost:8081` → point any app to Redis — **no per-project duplication**.

---

## Table of Contents

1. [Why This Exists](#why-this-exists)
2. [What You Get](#what-you-get)
3. [Repository Layout](#repository-layout)
4. [Quick Start](#quick-start)
5. [Configuration Files](#configuration-files)
   - [`docker-compose.yml` (root)](#docker-composeyml-root)
   - [`.env` (root)](#env-root)
   - [`server/Dockerfile`](#serverdockerfile)
   - [`server/redis.conf`](#serverredisconf)
   - [`.gitignore` (root)](#gitignore-root)
6. [Operational Commands (Full List)](#operational-commands-full-list)
7. [Using the Web UI](#using-the-web-ui)
8. [Connecting from other projects — 3 modes (Default: A)](#connecting-from-other-projects--3-modes-default-a)
9. [docker-compose: exact lines per mode (Default: A)](#docker-compose-exact-lines-per-mode-default-a)
10. [Integrating With Your Apps](#integrating-with-your-apps)
11. [Diagnostics &amp; Healthcheck](#diagnostics--healthcheck)
12. [Security &amp; Hardening](#security--hardening)
13. [Persistence &amp; Backups](#persistence--backups)
14. [Troubleshooting](#troubleshooting)
15. [FAQ](#faq)
16. [License](#license)

---

## Why This Exists

- Many projects need Redis (queues, cache, rate limits). Prefer **one reusable local service**.
- **Durable queues** (AOF on), **bounded memory** (LRU), **local-only exposure** by default.
- Avoid duplicating a Redis service in every repo.
- Have a **simple Web UI** to inspect keys/values on demand.

## What You Get

- **Single root compose**:
  - `dev-redis` → Redis 7 (alpine), AOF, memory policy, healthcheck
  - `redis-ui` → redis-commander (web UI)
- **Local-only port mapping** by default (127.0.0.1)
- **Named volume** for persistence (`redisdata`)
- Clean **internal networking**: UI → `dev-redis:6379`
- Patterns for host-native apps & other Docker projects

## Repository Layout

```
.
├─ docker-compose.yml      # single compose (server + ui)
├─ .env                    # shared env (ports, UI target, optional auth)
├─ .gitignore
├─ README.md               # this file
└─ server/
   ├─ Dockerfile           # FROM redis:7-alpine
   └─ redis.conf           # AOF, memory policy, requirepass (default)
```

---

## Quick Start

```bash
# 1) Clone & enter
# HTTPS
git clone https://github.com/EraysWorkbench/workbench-redis.git
# SSH
# git clone git@github.com:EraysWorkbench/workbench-redis.git
cd workbench-redis

# 2) Review and CHANGE the default password before first run
#    Protected-mode is ON by default. To connect from the UI/container network,
#    you MUST use a password (recommended) OR explicitly disable protected-mode.
#    Recommended (default): keep protected-mode ON and set a strong password.
#    - .env                 → REDIS_PASSWORD=change_me_first
#    - server/redis.conf    → requirepass change_me_first  (must match)
#    Alternative (dev quick, not recommended): no password + protected-mode OFF
#    - server/redis.conf    → protected-mode no  (and comment out requirepass)
#    - .env                 → 4-field REDIS_HOSTS: REDIS_HOSTS=local:dev-redis:6379

# 3) Build once
docker compose build

# 4) Run in background
docker compose up -d

# 5) Verify
docker compose ps

# 6) Use it
# Redis: 127.0.0.1:6379
# UI:    http://localhost:8081
```

---

## Configuration Files

### `docker-compose.yml` (root)

> See also the section **“docker-compose: exact lines per mode (Default: A)”** for the minimal diffs per mode.

```yaml
name: workbench-redis

services:
  dev-redis:
    build:
      context: ./server
    image: workbench-redis:server
    container_name: dev-redis
    ports:
      - "127.0.0.1:${REDIS_PORT:-6379}:6379"   # bind to localhost only (Mode A/C)
      # - "${REDIS_PORT:-6379}:6379"           # Mode B (open to LAN) – uncomment to enable
    volumes:
      - redisdata:/data
      - ./server/redis.conf:/usr/local/etc/redis/redis.conf:ro
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 2s
      retries: 20
    restart: unless-stopped
    networks:
      - workbench-net   # Mode A (default): shared bridge network

  redis-ui:
    image: ghcr.io/joeferner/redis-commander:latest
    container_name: workbench-redis-ui
    ports:
      - "127.0.0.1:${UI_PORT:-8081}:8081"      # keep UI localhost-only
    environment:
      # Connect via internal compose network (service name)
      REDIS_HOSTS: "${REDIS_HOSTS:-local:dev-redis:6379}"
      # If Redis AUTH is enabled, prefer the 5-field form (db + password):
      # REDIS_HOSTS: "local:dev-redis:6379:0:${REDIS_PASSWORD}"
      # Optional UI Basic Auth (enable BOTH in .env)
      HTTP_USER: "${HTTP_USER:-}"
      HTTP_PASSWORD: "${HTTP_PASSWORD:-}"
    depends_on:
      dev-redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8081"]
      interval: 5s
      timeout: 3s
      retries: 20
    restart: unless-stopped
    networks:
      - workbench-net   # Mode A (default)

volumes:
  redisdata:

networks:
  workbench-net:
    name: workbench-net
    driver: bridge
    external: false
```

### `.env` (root)

```ini
# Host port mappings
REDIS_PORT=6379
UI_PORT=8081

# Default (recommended): protected-mode ON + password required
REDIS_PASSWORD=change_me_first
REDIS_HOSTS=local:dev-redis:6379:0:${REDIS_PASSWORD}

# Optional HTTP Basic Auth for the UI (redis-commander)
# Uncomment both to enable:
# HTTP_USER=admin
# HTTP_PASSWORD=change_me
```

### `server/Dockerfile`

```dockerfile
FROM redis:7-alpine
```

### `server/redis.conf`

```conf
# --- Security / binding ---
protected-mode yes
bind 0.0.0.0
port 6379

# --- Authentication ---
# Password must match the REDIS_PASSWORD in .env
requirepass change_me_first

# --- Persistence ---
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec

# Optional RDB snapshots (dev-friendly defaults)
save 900 1
save 300 10
save 60 10000

# --- Memory policy (dev-friendly) ---
maxmemory 256mb
maxmemory-policy allkeys-lru

# --- Logging ---
loglevel notice
```

### `.gitignore` (root)

```gitignore
# Environment file
.env

# Redis persistence artifacts (safety)
*.rdb
*.aof
*.aof.*
dump.rdb
```

---

## Operational Commands (Full List)

```bash
# Build images
docker compose build

# Start both services in background
docker compose up -d

# Logs
docker logs -f dev-redis
docker logs -f workbench-redis-ui

# Health & status
docker compose ps
docker exec -it dev-redis redis-cli ping

# Inspect Redis data directory inside the container
docker exec -it dev-redis sh -lc 'ls -lh /data'

# Stop (keep data)
docker compose down

# Remove everything including data (IRREVERSIBLE!)
# docker compose down -v
```

---

## Using the Web UI

### Default (recommended): protected-mode ON + password

- Open **`http://localhost:8081`**.
- The UI connects to `dev-redis:6379` via the internal network.
- Ensure `.env` has the **5-field** form with a password:
  ```ini
  REDIS_PASSWORD=change_me_first
  REDIS_HOSTS=local:dev-redis:6379:0:${REDIS_PASSWORD}
  ```
- Optional HTTP Basic Auth:
  ```ini
  HTTP_USER=admin
  HTTP_PASSWORD=change_me
  ```

  Then: `docker compose up -d --force-recreate`.

### Alternative (dev quick): protected-mode OFF, no password

- `server/redis.conf`:
  ```conf
  protected-mode no
  # requirepass ...
  ```
- `.env` (**4-field**):
  ```ini
  REDIS_HOSTS=local:dev-redis:6379
  ```
- Recreate: `docker compose up -d --force-recreate`.

---

## Connecting from other projects — 3 modes (Default: A)

There are three ways for **other projects** to connect to this Redis. The default and recommended is **Mode A**.

### Mode A — Shared external Docker network (Default)

**When to use:** Make other compose projects connect **without** exposing Redis to your LAN.

**How it works:**

- This stack creates a bridge network **`workbench-net`** and attaches both services.
- Your other project joins the same network and talks to **`dev-redis:6379`**.

**Steps:**

1. Start this stack:
   ```bash
   docker compose up -d
   ```
2. In your other project’s compose:
   ```yaml
   services:
     app:
       # ...
       networks:
         - workbench-net

   networks:
     workbench-net:
       external: true   # join the network created here
   ```
3. In that project’s `.env`:
   ```ini
   REDIS_HOST=dev-redis
   REDIS_PORT=6379
   REDIS_PASSWORD=<same as server/redis.conf requirepass>
   ```
4. Recreate:
   ```bash
   docker compose up -d --force-recreate
   ```

### Mode B — Open host port (0.0.0.0:$ → 6379)

**When to use:** Quick tests or you can’t touch the other project’s compose.
**Security:** Exposes Redis to your LAN. Keep `requirepass` enabled and/or firewall the port.

**Steps:**

1. In this repo’s `docker-compose.yml`, change the mapping:
   ```yaml
   services:
     dev-redis:
       # ...
       ports:
         # - "127.0.0.1:${REDIS_PORT:-6379}:6379"
         - "${REDIS_PORT:-6379}:6379"
   ```
2. Recreate:
   ```bash
   docker compose up -d --force-recreate
   ```
3. In the other project’s `.env`:
   ```ini
   REDIS_HOST=172.17.0.1      # or host.docker.internal if available
   REDIS_PORT=6379
   REDIS_PASSWORD=<same as server/redis.conf requirepass>
   ```

> **Note:** You do **not** need to modify any `networks` settings for Mode B. Keep `workbench-net` as-is. The only change is the port mapping + a recreate.

### Mode C — Host-native app (no Docker)

**When to use:** Your app runs directly on the host.

```ini
REDIS_HOST=127.0.0.1
REDIS_PORT=6379
REDIS_PASSWORD=<same as server/redis.conf requirepass>
```

---

## docker-compose: exact lines per mode (Default: A)

These snippets show **only the lines you change**.

### Mode A — Shared external Docker network (Default)

```yaml
# services.dev-redis
ports:
  - "127.0.0.1:${REDIS_PORT:-6379}:6379"
networks:
  - workbench-net

# services.redis-ui
ports:
  - "127.0.0.1:${UI_PORT:-8081}:8081"
networks:
  - workbench-net

# bottom of file
networks:
  workbench-net:
    name: workbench-net
    driver: bridge
    external: false
```

### Mode B — Open host port (0.0.0.0:$ → 6379)

```yaml
# services.dev-redis (switch the ports mapping)
ports:
  # - "127.0.0.1:${REDIS_PORT:-6379}:6379"
  - "${REDIS_PORT:-6379}:6379"

# networks: NO CHANGE NEEDED
```

### Mode C — Host-native app (localhost)

```yaml
# services.dev-redis
ports:
  - "127.0.0.1:${REDIS_PORT:-6379}:6379"

# networks: keep as-is; it doesn't affect host-native usage
```

---

## Integrating With Your Apps

### Host-native apps (Mode C)

```ini
REDIS_HOST=127.0.0.1
REDIS_PORT=6379
```

### Other containers (outside this compose)

Pick a mode:

- **Mode A (Default, recommended):** join `workbench-net` and use `REDIS_HOST=dev-redis`.
- **Mode B:** open `${REDIS_PORT}:6379` and use `REDIS_HOST=172.17.0.1` (or `host.docker.internal`). *No networks changes needed; just change the port mapping and recreate.*

### Laravel `.env` Examples (Copy-Paste)

**A) Containerized Laravel → Mode A (shared network)**

```dotenv
QUEUE_CONNECTION=redis
REDIS_HOST=dev-redis
REDIS_PORT=6379

# Avoid collisions — set these uniquely per project
REDIS_DB=0
REDIS_CACHE_DB=1
REDIS_PREFIX=myapp_
HORIZON_PREFIX=myapp_horizon:
```

**B) Containerized Laravel → Mode B (host bridge IP)**

```dotenv
QUEUE_CONNECTION=redis
REDIS_HOST=172.17.0.1
REDIS_PORT=6379

REDIS_DB=0
REDIS_CACHE_DB=1
REDIS_PREFIX=myapp_
HORIZON_PREFIX=myapp_horizon:
```

**C) Host-native Laravel → Mode C (localhost)**

```dotenv
QUEUE_CONNECTION=redis
REDIS_HOST=127.0.0.1
REDIS_PORT=6379

REDIS_DB=0
REDIS_CACHE_DB=1
REDIS_PREFIX=myapp_
HORIZON_PREFIX=myapp_horizon:
```

> If Redis auth is enabled here, remember to configure your app’s Redis client with the same password.

---

## Diagnostics & Healthcheck

Step-by-step commands to diagnose connectivity/auth issues:

```bash
# UI logs (reconnecting/auth errors)
docker logs -f workbench-redis-ui

# Redis logs (protected-mode/AUTH errors)
docker logs -f dev-redis

# From the UI container: resolve Redis service name
docker exec -it workbench-redis-ui getent hosts dev-redis

# From the UI container: TCP test (installs nc if needed)
docker exec -it workbench-redis-ui sh -lc 'apk add --no-cache busybox-extras >/dev/null 2>&1 || true; nc -zv dev-redis 6379'

# From Redis container: authenticate and ping (password mode)
docker exec -it dev-redis redis-cli AUTH "$REDIS_PASSWORD"
docker exec -it dev-redis redis-cli ping
```

Common cues & fixes:

- **`DENIED Redis is running in protected mode ...`** → Either disable protected-mode *or* enable password and use **5-field** `REDIS_HOSTS`.
- **UI shows `reconnecting`** → name resolution, protected-mode, or password mismatch.
- **Password mismatch** → `requirepass` in `server/redis.conf` must match `.env` `REDIS_PASSWORD`; UI must use 5 fields.

---

## Security & Hardening

- **Local-only bindings** by default (`6379`, `8081` → `127.0.0.1`).
- **UI Basic Auth** via `.env` (`HTTP_USER`, `HTTP_PASSWORD`).
- **Redis AUTH**: `requirepass` in `server/redis.conf`, `.env` → `REDIS_PASSWORD`, UI → 5-field `REDIS_HOSTS`.
- Prefer `FLUSHDB` over `FLUSHALL`.
- If you open `${REDIS_PORT}:6379` (Mode B), consider firewall rules.

---

## Persistence & Backups

- AOF enabled (`appendfsync everysec`) — safer than `no`, faster than `always`.
- Optional snapshots add another safety net.
- Quick backup of the named volume:
  ```bash
  docker run --rm -v redisdata:/data -v "$PWD":/backup alpine \
    sh -lc 'cd /data && tar czf /backup/redisdata.tgz .'
  ```

---

## Troubleshooting

- **UI loads but shows no keys** → DB index mismatch; point UI with 5-field `REDIS_HOSTS` (correct DB).
- **“Connection refused” from other containers** → In Mode B use `172.17.0.1`; in Mode A join `workbench-net`.
- **Port in use** → Change `REDIS_PORT` or `UI_PORT` in `.env`.
- **Evictions** → Increase `maxmemory` or adjust `maxmemory-policy` in `redis.conf`.

---

## FAQ

**Why not include Redis in every project’s compose?**
One durable local Redis is simpler and reusable.

**Is this production-ready?**
Defaults are safe for dev. For prod: HA, monitoring, backups.

**How do I see keys?**
Web UI at `http://localhost:8081`, or `docker exec -it dev-redis redis-cli`.

**Can multiple apps share the same DB?**
Prefer separate DB indices and prefixes to avoid collisions.

---

## License

MIT — do what you want, no warranty.
