# Day 4: Docker Compose — Simulating Production Environments
## Theme: Multi-Service Applications, Dependency Ordering, Health Checks, and Failure Scenarios

**Goal:** By end of Day 4, you will be able to design and operate a realistic multi-service
Compose topology that mirrors production. You will understand dependency ordering, health-based
startup, restart policies, environment management, and how to simulate real failures locally.

**Estimated time:** 5–6 hours

---

## Background: Compose Is Not Just a Dev Tool

Most engineers treat Docker Compose as a shortcut for running multiple containers locally.
Senior engineers treat it as a **local production simulator** — the closest thing to a real
distributed system you can run on a laptop. If you design your Compose file correctly, your
local environment will expose the same failure modes as production.

Key shift in mindset:
- `depends_on` without `condition: service_healthy` is nearly useless in real apps
- Restart policies determine your application's resilience profile
- Environment variable management in Compose reflects how you'd manage config in production
- Networks in Compose map directly to the network segmentation concepts from Day 2

---

## Exercise 4.1 — The `depends_on` Trap

**What you'll learn:** Why naive `depends_on` fails and how to fix it with health checks.

**Setup:**
```bash
mkdir -p ~/docker-day4/depstrap
cd ~/docker-day4/depstrap
```

Create `docker-compose.yml` (the broken version):
```yaml
version: "3.9"

services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: appdb

  app:
    image: alpine
    depends_on:
      - db
    command: sh -c 'echo "Connecting to DB..." && sleep 2 && echo "Connected!"'
```

**Run it:**
```bash
docker compose up
```

The app says "Connected!" even though Postgres may not be ready to accept connections yet.
`depends_on` only waits for the container to **start** — not for the service inside to be
**ready**. This causes race conditions that are extremely common in production.

**Fix — add a health check to `db` and use `condition: service_healthy`:**
```yaml
version: "3.9"

services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: appdb
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d appdb"]
      interval: 5s
      timeout: 5s
      retries: 10
      start_period: 10s

  app:
    image: alpine
    depends_on:
      db:
        condition: service_healthy
    command: sh -c 'echo "DB is healthy, connecting..." && sleep 2 && echo "Done!"'
```

**Run the fixed version:**
```bash
docker compose down
docker compose up
```

Now `app` only starts **after** `db` passes its health check. Watch the startup sequence —
you'll see Compose waiting for the `db` health check to pass before starting `app`.

> **Hint:** To watch health check status in real time, open a second terminal and run:
> `docker compose ps` repeatedly, or `watch docker compose ps`.
> Health states: `starting` → `healthy` / `unhealthy`

---

## Exercise 4.2 — Building a Realistic 3-Tier Application Stack

**What you'll learn:** Full Compose topology with networking, volumes, health checks, and
environment management.

**Scenario:** Deploy a realistic web application stack: Nginx (reverse proxy) → Flask API →
PostgreSQL database, all properly networked and health-checked.

```bash
mkdir -p ~/docker-day4/fullstack/{api,nginx}
cd ~/docker-day4/fullstack
```

**Create the Flask API:**

`api/app.py`:
```python
from flask import Flask, jsonify
import psycopg2
import os
import time

app = Flask(__name__)

def get_db():
    return psycopg2.connect(
        host=os.environ['DB_HOST'],
        port=os.environ.get('DB_PORT', 5432),
        dbname=os.environ['DB_NAME'],
        user=os.environ['DB_USER'],
        password=os.environ['DB_PASSWORD']
    )

@app.route('/health')
def health():
    try:
        conn = get_db()
        conn.close()
        return jsonify({"status": "ok", "db": "connected"})
    except Exception as e:
        return jsonify({"status": "error", "db": str(e)}), 500

@app.route('/')
def index():
    conn = get_db()
    cur = conn.cursor()
    cur.execute('SELECT version()')
    version = cur.fetchone()[0]
    conn.close()
    return jsonify({"message": "Hello!", "db_version": version})
```

`api/requirements.txt`:
```
flask==3.0.0
gunicorn==21.2.0
psycopg2-binary==2.9.9
```

`api/Dockerfile`:
```dockerfile
FROM python:3.12-slim

RUN groupadd -r api && useradd -r -g api api

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .
RUN chown -R api:api /app

USER api
EXPOSE 5000

HEALTHCHECK --interval=10s --timeout=5s --start-period=15s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:5000/health')"

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "app:app"]
```

**Create the Nginx reverse proxy config:**

`nginx/default.conf`:
```nginx
upstream api_backend {
    server api:5000;
}

server {
    listen 80;

    location /api/ {
        proxy_pass http://api_backend/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_connect_timeout 5s;
        proxy_read_timeout 30s;
    }

    location /health {
        access_log off;
        return 200 "nginx ok\n";
        add_header Content-Type text/plain;
    }
}
```

**Create the Compose file:**

`docker-compose.yml`:
```yaml
version: "3.9"

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge

volumes:
  pgdata:

services:
  db:
    image: postgres:16-alpine
    networks:
      - backend
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD:-secret}
      POSTGRES_DB: ${DB_NAME:-appdb}
      POSTGRES_USER: ${DB_USER:-appuser}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER:-appuser} -d ${DB_NAME:-appdb}"]
      interval: 5s
      timeout: 5s
      retries: 10
      start_period: 15s
    restart: unless-stopped

  api:
    build: ./api
    networks:
      - backend
      - frontend
    environment:
      DB_HOST: db
      DB_PORT: 5432
      DB_NAME: ${DB_NAME:-appdb}
      DB_USER: ${DB_USER:-appuser}
      DB_PASSWORD: ${DB_PASSWORD:-secret}
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    networks:
      - frontend
    ports:
      - "8080:80"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      api:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost/health"]
      interval: 10s
      timeout: 5s
      retries: 3
    restart: unless-stopped
```

**Create `.env` file (for local overrides — never commit this):**
```bash
cat > .env << 'EOF'
DB_PASSWORD=localdevpassword
DB_NAME=appdb
DB_USER=appuser
EOF
```

**Run the full stack:**
```bash
docker compose up --build
```

In a second terminal:
```bash
# Test through nginx reverse proxy
curl http://localhost:8080/api/
curl http://localhost:8080/api/health
curl http://localhost:8080/health

# Check all service health status
docker compose ps

# View logs for a specific service
docker compose logs api --follow
```

**Verify network segmentation:**
```bash
# db should NOT be reachable from nginx (different network)
docker compose exec nginx ping -c 2 db

# api CAN reach db (on backend network)
docker compose exec api ping -c 2 db
```

---

## Exercise 4.3 — Restart Policies and Failure Simulation

**What you'll learn:** How restart policies work, when they fire, and what the failure modes are.

**Restart policy options:**
| Policy | Behavior |
|---|---|
| `no` | Never restart (default) |
| `always` | Always restart, even on clean exit |
| `on-failure` | Restart only on non-zero exit code |
| `unless-stopped` | Restart always unless manually stopped |

**Simulate a crashing service:**

Add this to your `docker-compose.yml` services section:
```yaml
  crasher:
    image: alpine
    command: sh -c 'echo "Starting..."; sleep 3; echo "Crashing!"; exit 1'
    restart: on-failure
```

```bash
docker compose up crasher
```

Watch it crash and restart. Docker applies an **exponential backoff** between restarts
(1s, 2s, 4s, 8s, up to 60s max). This prevents a crash loop from consuming 100% resources.

**Observe the restart count:**
```bash
docker inspect $(docker compose ps -q crasher) \
  --format 'Restarts: {{.RestartCount}}'
```

**Change to `restart: always` and exit the process cleanly (exit 0):**
```yaml
command: sh -c 'echo "Exiting cleanly"; exit 0'
restart: always
```

`always` restarts even on exit code 0. `on-failure` would NOT restart on clean exit.
This distinction matters: a task container (runs and completes) should use `on-failure`
or `no`, never `always`.

---

## Exercise 4.4 — Environment Variable Management Patterns

**What you'll learn:** The correct hierarchy for managing config in Compose environments.

**Compose environment variable resolution order (highest priority first):**
1. Shell environment variables (`export VAR=value`)
2. `.env` file in project directory
3. `environment:` key in docker-compose.yml
4. `env_file:` referenced file
5. Default values in image (`ENV` in Dockerfile)

**Demonstrate the priority:**
```bash
mkdir -p ~/docker-day4/envtest
cd ~/docker-day4/envtest

cat > .env << 'EOF'
MY_VAR=from_dotenv
EOF

cat > docker-compose.yml << 'EOF'
version: "3.9"
services:
  test:
    image: alpine
    environment:
      - MY_VAR=${MY_VAR:-default_fallback}
    command: sh -c 'echo "MY_VAR is: $MY_VAR"'
EOF

# Test 1: uses .env value
docker compose run --rm test

# Test 2: shell env overrides .env
MY_VAR=from_shell docker compose run --rm test

# Test 3: remove .env, falls back to default
mv .env .env.bak
docker compose run --rm test
mv .env.bak .env
```

**Using `env_file` for service-specific config:**
```yaml
services:
  api:
    image: myapi
    env_file:
      - ./common.env      # shared vars
      - ./api.env         # service-specific vars
    environment:
      - OVERRIDE_VAR=this_wins  # still overrides env_file
```

> **Production pattern:** Use `.env` for local development only (add to `.gitignore`).
> In CI/CD and production, inject variables via the shell environment or a secrets manager.
> Never commit `.env` files with real credentials.

---

## Exercise 4.5 — Compose Override Files: Dev vs Production Config

**What you'll learn:** How to use `docker-compose.override.yml` and `-f` flag to manage
environment-specific configurations without duplicating your base Compose file.

```bash
mkdir -p ~/docker-day4/overrides
cd ~/docker-day4/overrides
```

**Base `docker-compose.yml` (production-like):**
```yaml
version: "3.9"

services:
  api:
    image: myapi:${TAG:-latest}
    restart: unless-stopped
    environment:
      - APP_ENV=production
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:5000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

**`docker-compose.override.yml` (dev overrides — auto-loaded locally):**
```yaml
version: "3.9"

services:
  api:
    build: ./api          # build from source in dev, not pull image
    environment:
      - APP_ENV=development
      - FLASK_DEBUG=1
    volumes:
      - ./api:/app        # live code reload in dev
    ports:
      - "5000:5000"       # expose for direct access in dev
    restart: "no"         # don't restart in dev (want clean failures)
    healthcheck:
      disable: true       # skip health checks in dev for faster startup
```

**`docker-compose.prod.yml` (explicit production config):**
```yaml
version: "3.9"

services:
  api:
    restart: unless-stopped
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

**Usage patterns:**
```bash
# Development (auto-loads override)
docker compose up

# Production (explicit files only, no override)
docker compose -f docker-compose.yml -f docker-compose.prod.yml up

# CI (base only, no dev mounts)
docker compose -f docker-compose.yml up
```

---

## Exercise 4.6 — Simulating Production Failures in Compose

**Scenario (real-world incident simulation):** Your Compose stack is running. The database
crashes. Diagnose the cascade failure, understand what Compose does, and restore service.

**Setup — run the fullstack from Exercise 4.2 first:**
```bash
cd ~/docker-day4/fullstack
docker compose up -d
sleep 15  # wait for healthy state
curl http://localhost:8080/api/health
```

**Simulate DB crash:**
```bash
# Kill the DB container abruptly (simulates OOM kill or node failure)
docker compose kill db
```

**Observe the cascade:**
```bash
# Check API health — it should now fail
curl http://localhost:8080/api/health

# Check Compose service states
docker compose ps

# Watch API logs — connection errors to DB
docker compose logs api --tail 20

# Watch nginx logs — 502 Bad Gateway upstream errors
docker compose logs nginx --tail 20
```

**Restore service:**
```bash
docker compose start db

# Watch DB health check recover
watch docker compose ps

# Once db is healthy, verify API recovers automatically
curl http://localhost:8080/api/health
```

**Questions to answer:**
> 1. When the DB restarted, did the API automatically reconnect without restart? (It depends on
>    whether the app has connection retry logic — this is an application concern, not Docker's)
> 2. Did Nginx return 502 during the outage? Why? (upstream unreachable)
> 3. Did any data get lost from the named volume `pgdata`? Why not?

---

## Exercise 4.7 — Compose Observability: Logs, Events, and Stats

**What you'll learn:** How to observe a running Compose stack without external tools.

```bash
cd ~/docker-day4/fullstack

# Stream logs from all services with timestamps
docker compose logs -f --timestamps

# Stream logs from specific services only
docker compose logs -f api nginx

# Watch resource usage across all services
docker stats $(docker compose ps -q)

# Listen to Docker events for this Compose project
docker events --filter label=com.docker.compose.project=fullstack

# Run a one-off command in the api service context
docker compose exec api env | sort

# Scale the API service (multiple instances)
docker compose up -d --scale api=3

# Nginx will now round-robin between 3 API instances
for i in {1..6}; do curl -s http://localhost:8080/api/ | python3 -m json.tool; done
```

> **Note on scaling:** When scaling services in Compose, avoid port mappings on scaled
> services (each instance can't bind the same host port). The nginx reverse proxy handles
> this correctly because it connects to the `api` DNS name, which Compose resolves to all
> instances using round-robin DNS.

---

## Day 4 Checkpoint

Answer these without looking:

1. Why is `depends_on: [db]` insufficient for production startup ordering?
2. What is the difference between `restart: always` and `restart: unless-stopped`?
3. What is the Compose environment variable resolution order?
4. What does `docker-compose.override.yml` do and when is it auto-loaded?
5. How does `docker compose exec` differ from `docker exec`?
6. When you scale a service with `--scale api=3`, how does Compose handle DNS for that service?
7. Why should you never add port mappings to a scaled service?

---

## Day 4 Summary

| Concept | What You Proved |
|---|---|
| `depends_on` with `service_healthy` | Controlled startup ordering via health checks |
| 3-tier Compose topology | Nginx → API → Postgres with network segmentation |
| Restart policies | `on-failure` vs `always` vs `unless-stopped` behavior |
| Environment variable hierarchy | Shell > .env > environment: > env_file > Dockerfile ENV |
| Compose override files | Dev vs prod config without duplication |
| Failure simulation | DB crash, cascade failure, and recovery observation |
| Scaling in Compose | `--scale`, round-robin DNS, Nginx load balancing |
