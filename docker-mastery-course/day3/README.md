# Day 3: Image Engineering and Multi-Stage Builds
## Theme: Building Production-Grade Images (Not Just "Working" Images)

**Goal:** By end of Day 3, you will know how to build minimal, secure, cache-efficient images
using multi-stage builds. You will understand every instruction in a Dockerfile at a mechanical
level and stop copy-pasting from Stack Overflow.

**Estimated time:** 5–6 hours

---

## Background: The Image Is the Deployment Unit

In production, the image IS your deployment artifact. A poorly built image means:
- Slow CI/CD pipelines (large image pulls)
- Security vulnerabilities (unpatched base images, unnecessary packages)
- Non-reproducible builds (no pinned versions)
- Credential leaks (secrets baked into layers)
- Broken signal handling (PID 1 problem at runtime)

Professional image engineering is a discipline, not an afterthought.

---

## Exercise 3.1 — Understanding Every Dockerfile Instruction

**What you'll learn:** What each Dockerfile instruction actually does at the layer level.

**Setup:**
```bash
mkdir -p ~/docker-day3/instrtest
cd ~/docker-day3/instrtest
```

Create `Dockerfile.explore`:
```dockerfile
FROM ubuntu:22.04

# LABEL adds metadata to the image (zero-byte layer)
LABEL maintainer="you@example.com" version="1.0"

# ARG: build-time variable (NOT available at runtime)
ARG BUILD_VERSION=dev

# ENV: runtime environment variable (available in container)
ENV APP_ENV=production \
    APP_VERSION=$BUILD_VERSION

# RUN: executes command in new layer, commits result
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*

# WORKDIR: creates dir + sets working dir (preferred over RUN mkdir + cd)
WORKDIR /app

# COPY: copies files from build context into image layer
COPY ./myapp.sh .

# RUN again = another layer
RUN chmod +x myapp.sh

# EXPOSE: documents the port (does NOT publish it — informational only)
EXPOSE 8080

# USER: switch to non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser
USER appuser

# HEALTHCHECK: tells Docker how to check if the container is healthy
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

# CMD: default command when container starts (overridable at runtime)
CMD ["./myapp.sh"]
```

Create `myapp.sh`:
```bash
#!/bin/sh
echo "App version: $APP_VERSION"
echo "Env: $APP_ENV"
sleep 3600
```

**Build and inspect:**
```bash
docker build --build-arg BUILD_VERSION=1.2.3 \
  -f Dockerfile.explore -t instrtest:v1 .

# See all layers
docker history instrtest:v1

# Inspect labels
docker inspect instrtest:v1 --format '{{json .Config.Labels}}'

# Inspect environment variables baked into image
docker inspect instrtest:v1 --format '{{json .Config.Env}}'

# Run and verify runtime env
docker run --rm instrtest:v1 env | grep APP_
```

**Critical experiment — ARG vs ENV:**
```bash
# ARG is NOT in the image after build
docker inspect instrtest:v1 --format '{{json .Config.Env}}' | grep BUILD_VERSION
# Not found — ARG is build-time only

# ENV IS in the image
docker inspect instrtest:v1 --format '{{json .Config.Env}}' | grep APP_VERSION
# Found: APP_VERSION=1.2.3
```

> **Security note:** Never use `ARG` to pass secrets — they appear in `docker history`.
> Never use `ENV` for secrets either — they're baked into the image permanently.

---

## Exercise 3.2 — The Secrets Leak Trap

**What you'll learn:** Why naive secret handling in Dockerfiles leaks credentials into image layers.

**Scenario:** A developer wrote this Dockerfile to pull from a private package registry:

```dockerfile
FROM python:3.12-slim

ARG PRIVATE_TOKEN=ghp_supersecret123

RUN pip install --extra-index-url \
    https://token:${PRIVATE_TOKEN}@private.registry.example.com/simple/ \
    myprivatepkg

COPY app.py .
CMD ["python", "app.py"]
```

**Why this leaks (demonstrate it):**

```bash
mkdir -p ~/docker-day3/secretleak
cd ~/docker-day3/secretleak

cat > Dockerfile.leak << 'EOF'
FROM alpine
ARG SECRET_VALUE=mysupersecret
RUN echo "Fetching with token: $SECRET_VALUE" && echo "done"
CMD ["sh"]
EOF

docker build --build-arg SECRET_VALUE=myactualpassword \
  -f Dockerfile.leak -t leakyimage:v1 .

# Extract the secret from docker history:
docker history --no-trunc leakyimage:v1 | grep -i secret
```

The secret appears in the build history!

**The correct approach — BuildKit secrets:**

```bash
# Create a secret file (simulating a token)
echo -n "myactualpassword" > /tmp/mysecret.txt

cat > Dockerfile.safe << 'EOF'
# syntax=docker/dockerfile:1
FROM alpine
RUN --mount=type=secret,id=mytoken \
    TOKEN=$(cat /run/secrets/mytoken) && \
    echo "Using token (length: ${#TOKEN})" && \
    echo "done"
CMD ["sh"]
EOF

DOCKER_BUILDKIT=1 docker build \
  --secret id=mytoken,src=/tmp/mysecret.txt \
  -f Dockerfile.safe -t safeimage:v1 .

# Verify secret is NOT in history
docker history --no-trunc safeimage:v1 | grep myactualpassword
```

**Nothing found.** The secret is mounted during build but never committed to a layer.

> **Hint:** To check if sensitive data is in an image layer, use:
> `docker save <image> | tar -xO | strings | grep -i <keyword>`
> This extracts ALL layer content and searches for the string.

---

## Exercise 3.3 — Multi-Stage Builds: The Production Standard

**What you'll learn:** How to use multi-stage builds to create minimal production images.

**The problem:** A Go application compiled on a full build environment produces a 1.2 GB image.
The same app in a multi-stage build can be under 20 MB.

**Setup:**
```bash
mkdir -p ~/docker-day3/multistage
cd ~/docker-day3/multistage
```

Create `main.go`:
```go
package main

import (
    "fmt"
    "net/http"
    "os"
)

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        hostname, _ := os.Hostname()
        fmt.Fprintf(w, "Hello from container: %s\n", hostname)
    })
    fmt.Println("Server starting on :8080")
    http.ListenAndServe(":8080", nil)
}
```

**Single-stage (BAD) Dockerfile:**
```dockerfile
FROM golang:1.22

WORKDIR /app
COPY main.go .
RUN go build -o server main.go
EXPOSE 8080
CMD ["./server"]
```

**Multi-stage (GOOD) Dockerfile:**
```dockerfile
# Stage 1: Builder — has all build tools
FROM golang:1.22-alpine AS builder

WORKDIR /app
COPY main.go .

# Build a fully static binary
RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags="-w -s" \
    -o server main.go

# Stage 2: Runtime — minimal image
FROM scratch

# Copy ONLY the compiled binary from builder stage
COPY --from=builder /app/server /server

EXPOSE 8080
CMD ["/server"]
```

**Build both and compare:**
```bash
# Bad image
docker build -f Dockerfile.single -t goapp:single .

# Good image
docker build -f Dockerfile.multi -t goapp:multi .

# Compare sizes
docker images | grep goapp
```

**Expected result:**
- `goapp:single`: ~800 MB (full Go toolchain included)
- `goapp:multi`: ~8 MB (just the binary, on scratch)

**Test both still work:**
```bash
docker run -d --name gotest -p 8080:8080 goapp:multi
curl localhost:8080
docker rm -f gotest
```

**Exercise: Now try to exec into the scratch-based container:**
```bash
docker run -d --name goscratch goapp:multi
docker exec -it goscratch sh
```

**It fails** — `scratch` has no shell, no tools, no anything. This is maximum security but
makes debugging harder. This is the tradeoff to understand.

> **Hint for debugging scratch/distroless containers:** Use `docker cp` to extract files,
> or use a debug sidecar pattern:
> `docker run --rm --pid=container:goscratch --network=container:goscratch \
>   nicolaka/netshoot sh`
> This joins the namespaces of the running container without needing a shell inside it.

---

## Exercise 3.4 — Python Multi-Stage with Virtual Environment

**Scenario:** Build a production Python image that:
- Builds in a full environment (with build tools)
- Runs in slim without build tools
- Does not run as root
- Passes a security scan

```bash
mkdir -p ~/docker-day3/pyapp
cd ~/docker-day3/pyapp
```

Create `requirements.txt`:
```
flask==3.0.0
gunicorn==21.2.0
```

Create `app.py`:
```python
from flask import Flask, jsonify
app = Flask(__name__)

@app.route('/health')
def health():
    return jsonify({"status": "ok"})

@app.route('/')
def home():
    return jsonify({"message": "Production-grade container!"})
```

Create `Dockerfile`:
```dockerfile
# Stage 1: Build dependencies in full environment
FROM python:3.12 AS builder

WORKDIR /app

# Install build dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# Stage 2: Runtime — slim base
FROM python:3.12-slim AS runtime

# Create non-root user
RUN groupadd -r appuser && useradd -r -g appuser -d /app appuser

WORKDIR /app

# Copy installed packages from builder
COPY --from=builder /root/.local /home/appuser/.local

# Copy application code
COPY app.py .

# Fix permissions
RUN chown -R appuser:appuser /app

# Switch to non-root
USER appuser

# Ensure local packages are on PATH
ENV PATH=/home/appuser/.local/bin:$PATH

EXPOSE 5000

HEALTHCHECK --interval=15s --timeout=5s --start-period=10s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:5000/health')"

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "app:app"]
```

**Build and verify:**
```bash
docker build -t pyapp:prod .

# Verify it runs as non-root
docker run --rm pyapp:prod whoami

# Test health endpoint
docker run -d --name pytest -p 5000:5000 pyapp:prod
sleep 3
curl localhost:5000/health
curl localhost:5000/

# Check health status
docker inspect pytest --format '{{.State.Health.Status}}'

docker rm -f pytest
```

---

## Exercise 3.5 — .dockerignore: What You're Probably Forgetting

**What you'll learn:** How missing `.dockerignore` bloats your build context and leaks secrets.

**Demonstration:**

```bash
mkdir -p ~/docker-day3/contexttest
cd ~/docker-day3/contexttest

# Simulate a project with sensitive files and junk
mkdir -p .git node_modules dist
echo "DB_PASSWORD=supersecret" > .env
echo "private-key" > credentials.pem
dd if=/dev/zero of=node_modules/bigfile.bin bs=1M count=50 2>/dev/null
touch Dockerfile app.js
```

**Without .dockerignore:**
```bash
cat > Dockerfile << 'EOF'
FROM node:20-alpine
WORKDIR /app
COPY . .
CMD ["node", "app.js"]
EOF

# Build and watch the "Sending build context" size
docker build -t contexttest:v1 . 2>&1 | head -5
```

The build context includes `node_modules` (50 MB), `.git`, `.env`, `credentials.pem`!

**Now add .dockerignore:**
```bash
cat > .dockerignore << 'EOF'
# Dependencies (install inside container)
node_modules
npm-debug.log

# Build artifacts
dist
.next
build

# Secrets and credentials (NEVER send to Docker daemon)
.env
*.pem
*.key
secrets/

# Version control
.git
.gitignore

# OS junk
.DS_Store
Thumbs.db

# IDE files
.vscode
.idea
EOF

# Rebuild and compare context size
docker build -t contexttest:v2 . 2>&1 | head -5
```

**Context shrinks dramatically.** More importantly, sensitive files are no longer sent to the
Docker daemon and cannot accidentally end up in a layer.

> **Critical:** Even if you `COPY . .` and then `RUN rm .env`, the `.env` is still in the
> previous layer. Layer history is permanent. `.dockerignore` prevents the file from ever
> reaching the daemon.

---

## Exercise 3.6 — Image Analysis and Security Scanning

**What you'll learn:** How to inspect what's inside an image and scan for vulnerabilities.

**Install Trivy (open source scanner):**
```bash
# Linux/macOS:
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin

# Verify
trivy version
```

**Scan your images:**
```bash
# Scan a common base image
trivy image python:3.12

# Scan your production app image
trivy image pyapp:prod

# Scan with severity filter (only HIGH and CRITICAL)
trivy image --severity HIGH,CRITICAL pyapp:prod

# Scan and output as table (default) vs JSON
trivy image --format json pyapp:prod | python3 -m json.tool | head -50
```

**Compare base image vulnerability counts:**
```bash
echo "=== python:3.12 (full) ==="
trivy image --severity HIGH,CRITICAL --quiet python:3.12 2>/dev/null | tail -5

echo "=== python:3.12-slim ==="
trivy image --severity HIGH,CRITICAL --quiet python:3.12-slim 2>/dev/null | tail -5

echo "=== python:3.12-alpine ==="
trivy image --severity HIGH,CRITICAL --quiet python:3.12-alpine 2>/dev/null | tail -5
```

**Inspect image filesystem layer by layer:**
```bash
# Install dive (layer explorer)
# Linux:
wget -q https://github.com/wagoodman/dive/releases/latest/download/dive_linux_amd64.tar.gz \
  -O /tmp/dive.tar.gz && tar -xf /tmp/dive.tar.gz -C /usr/local/bin dive

# Explore your production image
dive pyapp:prod
```

Dive shows you exactly what changed in each layer and what's wasting space.

---

## Day 3 Checkpoint

Answer these without looking:

1. What is the difference between `ARG` and `ENV` in a Dockerfile?
2. Why does `ARG SECRET=value` leak credentials even if you don't use `ENV`?
3. What is a multi-stage build? What does `COPY --from=builder` do?
4. What is a `scratch` base image? What is its advantage and limitation?
5. What does `.dockerignore` prevent, and why can't you fix a leaked secret by deleting it in
   a later `RUN` step?
6. What does `CMD ["./app"]` (exec form) do differently than `CMD ./app` (shell form)?
7. What does `EXPOSE 8080` in a Dockerfile actually do? (Hint: not what most people think)

---

## Day 3 Summary

| Concept | What You Proved |
|---|---|
| Dockerfile instruction mechanics | ARG vs ENV, layer creation, EXPOSE semantics |
| Secrets leak via image history | Extracted secret from `docker history` |
| BuildKit secret mounts | Secret used during build, not stored in layer |
| Multi-stage builds (Go) | 800 MB → 8 MB production image |
| Multi-stage builds (Python) | Non-root, gunicorn, healthcheck in slim image |
| .dockerignore impact | Build context size and secret exclusion |
| Image scanning with Trivy | Found CVEs in base images, compared slim vs full |
