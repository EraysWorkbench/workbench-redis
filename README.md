# Workbench Redis — Single, Reusable Redis + Web UI (One-Compose Setup)

A production-grade **developer Redis** you can spin up once and reuse across many projects — paired with a **minimal admin UI** (redis-commander). Everything runs from a **single root `docker-compose.yml`**, with clean localhost bindings and sane defaults.

> Goal: Clone → `docker compose build` → `docker compose up -d` → manage at `http://localhost:8081` → point any app to `127.0.0.1:6379` (or host bridge from other containers) — **no per-project compose duplication**.

---

## Table of Contents

1. [Why This Exists](#why-this-exists)
2. [What You Get](#what-you-get)
3. [Repository Layout](#repository-layout)
4. [Quick Start](#quick-start)
5. [Configuration Files](#configuration-files)

   * [`docker-compose.yml` (root)](#docker-composeyml-root)
   * [`.env` (root)](#env-root)
   * [`server/Dockerfile`](#serverdockerfile)
   * [`server/redis.conf`](#serverredisconf)
   * [`.gitignore` (root)](#gitignore-root)
6. [Operational Commands (Full List)](#operational-commands-full-list)
7. [Using the Web UI](#using-the-web-ui)
8. [Integrating With Your Apps](#integrating-with-your-apps)

   * [Host-native apps](#host-native-apps)
   * [Other containers (outside this compose)](#other-containers-outside-this-compose)
   * [Laravel `.env` Examples (Copy-Paste)](#laravel-env-examples-copy-paste)
9. [Security &amp; Hardening](#security--hardening)
10. [Persistence &amp; Backups](#persistence--backups)
11. [Troubleshooting](#troubleshooting)
12. [FAQ](#faq)
13. [License](#license)

---

## Why This Exists

* You have many projects that need Redis (queues, cache, rate limits) and you prefer **one reusable local service**.
* You want **durable queues** (AOF on), **bounded memory** (LRU), and **local-only exposure** (safe by default).
* You do **not** want to duplicate a Redis service in every repo’s compose.
* You want a **simple Web UI** to inspect keys and values on demand.

## What You Get

* **Single root compose** running:

  * `dev-redis` → Redis 7 (alpine), with AOF, memory policy, healthcheck
  * `redis-ui` → redis-commander (web UI) bound to localhost
* **Local-only port mapping** (127.0.0.1) for both services
* **Named volume** for persistence (`redisdata`)
* Clean **internal networking**: UI talks to Redis by service name (`dev-redis:6379`)
* Drop-in integration patterns for host-native apps & other containers

## Repository Layout

```
.
├─ docker-compose.yml      # single compose (server + ui)
├─ .env                    # shared env (ports, UI target, optional auth)
├─ .gitignore
├─ README.md               # this file
└─ server/
   ├─ Dockerfile           # FROM redis:7-alpine
   └─ redis.conf           # AOF, memory policy, optional requirepass
```

---

## Quick Start

````bash
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
#    - .env                 → use 4-field REDIS_HOSTS: REDIS_HOSTS=local:dev-redis:6379

# 3) Build once
docker compose build

# 4) Run in background
docker compose up -d

# 5) Verify
docker compose ps

# 6) Use it
# Redis: 127.0.0.1:6379
# UI:    http://localhost:8081
````

> Tip (Linux): if another containerized project must connect to this Redis, use `172.17.0.1:6379` (Docker bridge) or `host.docker.internal:6379` (if available) as the host in that project’s `.env`.

---

## Configuration Files

### `docker-compose.yml` (root)

```yaml
name: workbench-redis

services:
  dev-redis:
    build:
      context: ./server
    image: workbench-redis:server
    container_name: dev-redis
    ports:
      - "127.0.0.1:${REDIS_PORT:-6379}:6379"   # bind to localhost only
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

  redis-ui:
    image: ghcr.io/joeferner/redis-commander:latest
    container_name: workbench-redis-ui
    ports:
      - "127.0.0.1:${UI_PORT:-8081}:8081"      # bind to localhost only
    environment:
      # Connect to Redis via the internal compose network (service name)
      REDIS_HOSTS: "${REDIS_HOSTS:-local:dev-redis:6379}"

      # Enable UI Basic Auth by setting both in .env (leave empty to disable):
      HTTP_USER: "${HTTP_USER:-}"
      HTTP_PASSWORD: "${HTTP_PASSWORD:-}"

      # If Redis auth is enabled, prefer the 5-field form:
      # name:host:port[:db][:password]
      # Example using the same password as in server/redis.conf:
      # REDIS_HOSTS: "local:dev-redis:6379:0:${REDIS_PASSWORD}"
    depends_on:
      dev-redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8081"]
      interval: 5s
      timeout: 3s
      retries: 20
    restart: unless-stopped

volumes:
  redisdata:
```

### `.env` (root)

````ini
# Host port mappings
REDIS_PORT=6379
UI_PORT=8081

# UI connects to Redis over compose internal network (service name)
# Default (recommended): protected-mode ON + password required
REDIS_PASSWORD=change_me_first
REDIS_HOSTS=local:dev-redis:6379:0:${REDIS_PASSWORD}

# Optional HTTP Basic Auth for the UI (redis-commander)
# Uncomment both to enable:
# HTTP_USER=admin
# HTTP_PASSWORD=change_me
```ini
# Host port mappings
REDIS_PORT=6379
UI_PORT=8081

# UI connects to Redis over compose internal network (service name)
REDIS_HOSTS=local:dev-redis:6379

# If you enable `requirepass` in server/redis.conf, set and keep this in sync:
# REDIS_PASSWORD=change_me

# Optional HTTP Basic Auth for the UI (redis-commander)
# Uncomment both to enable:
# HTTP_USER=admin
# HTTP_PASSWORD=change_me
````

### `server/Dockerfile`

```dockerfile
FROM redis:7-alpine
```

### `server/redis.conf`

````conf
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
# safer than 'no', faster than 'always'
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
```conf
# --- Security / binding ---
protected-mode yes
bind 0.0.0.0
port 6379

# --- Authentication (optional) ---
# If you want a password, uncomment and set your own strong value:
# requirepass yourStrongPasswordHere

# --- Persistence ---
appendonly yes
appendfilename "appendonly.aof"
# safer than 'no', faster than 'always'
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
````

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

# Tail logs
docker logs -f dev-redis
# (UI)
docker logs -f workbench-redis-ui

# Health & status
docker compose ps
docker exec -it dev-redis redis-cli ping

# Inspect Redis data directory inside the container
docker exec -it dev-redis sh -lc 'ls -lh /data'

# Stop and remove containers (volume persists)
docker compose down

# Remove everything including data (IRREVERSIBLE!)
# docker compose down -v
```

---

## Diagnostics & Healthcheck

Step-by-step commands to diagnose connectivity/auth issues:

```bash
# UI logs (look for reconnecting, auth errors)
docker logs -f workbench-redis-ui

# Redis logs (look for protected-mode or AUTH errors)
docker logs -f dev-redis

# From the UI container: resolve the Redis service name
docker exec -it workbench-redis-ui getent hosts dev-redis

# From the UI container: TCP connectivity test (installs nc if needed)
docker exec -it workbench-redis-ui sh -lc 'apk add --no-cache busybox-extras >/dev/null 2>&1 || true; nc -zv dev-redis 6379'

# From Redis container: authenticate and ping (password mode)
docker exec -it dev-redis redis-cli AUTH "$REDIS_PASSWORD"
docker exec -it dev-redis redis-cli ping
```

Common error cue and fixes:

* **`DENIED Redis is running in protected mode ...`** → either disable protected-mode *or* enable password and use the **5-field** form in `REDIS_HOSTS`.
* **UI shows `reconnecting`** → usually name resolution, protected-mode, or password mismatch.
* **Password mismatch** → `server/redis.conf` `requirepass` must match `.env` `REDIS_PASSWORD` and UI must use 5 fields.

---

## Using the Web UI

### Default (recommended): protected-mode ON + password

* Open **`http://localhost:8081`**.
* The UI connects to `dev-redis:6379` over the internal compose network.
* Ensure `.env` has the **5-field** form with a password:

  ```ini
  REDIS_PASSWORD=change_me_first
  REDIS_HOSTS=local:dev-redis:6379:0:${REDIS_PASSWORD}
  ```
* (Optional) Protect the UI with HTTP Basic Auth:

  ```ini
  HTTP_USER=admin
  HTTP_PASSWORD=change_me
  ```

  Then recreate: `docker compose up -d --force-recreate`.

### Alternative (dev quick): protected-mode OFF, no password

* Edit `server/redis.conf`:

  ```conf
  protected-mode no
  # requirepass ...  (comment out)
  ```
* Edit `.env` to use the **4-field** form:

  ```ini
  REDIS_HOSTS=local:dev-redis:6379
  ```
* Recreate: `docker compose up -d --force-recreate`.

---

## Integrating With Your Apps

### Host-native apps

Point them to the localhost mapping:

```ini
REDIS_HOST=127.0.0.1
REDIS_PORT=6379
```

### Other containers (outside this compose)

Containers in **other** compose projects cannot resolve `dev-redis` by name. Use your host bridge:

* **Linux (bridge)**: `REDIS_HOST=172.17.0.1`
* **Generic** (if available): `REDIS_HOST=host.docker.internal`
* Port stays `6379` unless you changed it in this repo’s `.env`.

> Find your Docker bridge IP with: `ip route | grep docker0` (look for `via 172.17.0.1`).

### Laravel `.env` Examples (Copy-Paste)

**A) Containerized Laravel → host Redis via bridge IP (quickest)**

```dotenv
QUEUE_CONNECTION=redis
REDIS_HOST=172.17.0.1
REDIS_PORT=6379

# Avoid collisions — set these uniquely per project
REDIS_DB=0
REDIS_CACHE_DB=1
REDIS_PREFIX=myapp_
HORIZON_PREFIX=myapp_horizon:
```

**B) Containerized Laravel → host Redis via host.docker.internal (clean DNS)**

```dotenv
QUEUE_CONNECTION=redis
REDIS_HOST=host.docker.internal
REDIS_PORT=6379

REDIS_DB=0
REDIS_CACHE_DB=1
REDIS_PREFIX=myapp_
HORIZON_PREFIX=myapp_horizon:
```

**C) Host-native Laravel → host Redis**

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

## Security & Hardening

* **Local-only bindings**: Both `6379` and `8081` are mapped to `127.0.0.1`.
* **UI Basic Auth**: enable via `.env` (`HTTP_USER` / `HTTP_PASSWORD`).
* **Redis AUTH**: uncomment `requirepass` in `server/redis.conf`, set `.env` → `REDIS_PASSWORD`, and update `REDIS_HOSTS` to the 5-field form.
* Prefer `FLUSHDB` over `FLUSHALL` (the latter affects all DBs).
* If you change port mappings, keep your clients in sync.

---

## Persistence & Backups

* AOF is enabled (`appendfsync everysec`) — safer than `no`, faster than `always`.
* Optional snapshots provide an extra safety net.
* Quick backup of the named volume:

  ```bash
  docker run --rm -v redisdata:/data -v "$PWD":/backup alpine \
    sh -lc 'cd /data && tar czf /backup/redisdata.tgz .'
  ```
* Restore by stopping the stack, removing/recreating the volume, and untarring into `/data`.

---

## Troubleshooting

* **UI loads but shows no keys** → Check that your app uses the same DB index; point UI with 5-field `REDIS_HOSTS` (see above).
* **“Connection refused” from other containers** → They can’t resolve `dev-redis`. Use `172.17.0.1` or `host.docker.internal`.
* **Port already in use** → Change `REDIS_PORT` or `UI_PORT` in `.env`.
* **Password mismatch** → If you enabled `requirepass`, update both your apps and `REDIS_HOSTS` accordingly.
* **Evictions** → Increase `maxmemory` or change `maxmemory-policy` in `server/redis.conf`.

---

## FAQ

**Why not include Redis in every project’s compose?**
You can, but this creates duplication and overhead. One durable local Redis is simpler and reusable.

**Is this production-ready?**
The defaults are safe for dev (local-only bind, AOF on). For production, deploy Redis with HA, monitoring, and backups.

**How do I see keys?**
Use the Web UI at `http://localhost:8081`, or `docker exec -it dev-redis redis-cli`.

**Can multiple apps share the same DB?**
Prefer separate DB indices and prefixes to avoid collisions.

---

## License

MIT — do what you want, no warranty.
