# 🐳 Module 01 — Docker

**Time:** ~4 hours · **Cost:** free · **AWS resources:** none

---

## 🎯 What you'll build

Three production-grade Dockerfiles and a Compose stack that runs the whole application on your laptop.

```text
   Browser :3000
        │
        ▼
   ┌──────────┐      ┌───────────────┐      ┌─────────┐
   │ frontend │─────►│ auth-service  │─────►│  mysql  │
   │  Node 22 │      │  Python/Flask │      │   8.0   │
   └────┬─────┘      └───────────────┘      └─────────┘
        │
        │            ┌──────────────────┐
        └───────────►│ roadmap-service  │
                     │ Java/Spring Boot │
                     └──────────────────┘
```

**Do this module before spending anything on AWS.** When it works, you know the application and the images are correct — so any failure in Module 04 is infrastructure, not code. That single fact will save you hours.

---

## 📋 Contents

- [Part 1 — The frontend Dockerfile, built up in 5 passes](#part-1)
- [Part 2 — auth-service: native dependencies](#part-2)
- [Part 3 — roadmap-service: the JVM in a container](#part-3)
- [Part 4 — .dockerignore](#part-4)
- [Part 5 — Compose, built up in 4 passes](#part-5)
- [Checkpoint](#checkpoint)

---

<a id="part-1"></a>

# Part 1 · The frontend Dockerfile

## Pass 1 — the smallest thing that works

Create `workspace/src/frontend/Dockerfile`. **Type it.**

```dockerfile
FROM node:22-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --omit=dev
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

Line by line:

| Line | What it does |
|---|---|
| `FROM node:22-alpine` | Base image. `alpine` is ~5 MB vs ~350 MB for the Debian variant. |
| `WORKDIR /app` | Sets the working directory *and creates it*. Every later `COPY`/`RUN` is relative to this. |
| `COPY package*.json ./` | Copies **only** the manifests. The wildcard catches `package.json` and `package-lock.json`. |
| `RUN npm install --omit=dev` | Installs production dependencies. `--omit=dev` skips devDependencies. |
| `COPY . .` | Now copy the source. |
| `EXPOSE 3000` | **Documentation only.** It publishes nothing — that's `-p` / `ports:`. |
| `CMD [...]` | Default command. Exec form (JSON array), not shell form. |

Build it:

```bash
cd workspace/src/frontend
docker build -t frontend:v1 .
docker images frontend:v1
```

```text
REPOSITORY   TAG   SIZE
frontend     v1    ~230MB
```

It works. Now let's find out what's wrong with it.

---

### 🔍 Why `COPY package*.json` comes *before* `COPY . .`

This is the single most important Dockerfile idea. Try it:

```bash
# Change one character of source code
echo "// touch" >> server.js

# Rebuild
docker build -t frontend:v1 .
```

Watch the output — `npm install` is **CACHED**. It didn't re-run.

Docker caches each layer. A layer is invalidated when its inputs change, **and every layer after it is invalidated too**. Because `package.json` didn't change, the `npm install` layer is reused.

Now flip the order to prove the point:

```dockerfile
COPY . .                        # ← source first
RUN npm install --omit=dev
```

```bash
echo "// touch again" >> server.js
docker build -t frontend:bad .
```

`npm install` re-runs — a full dependency download on **every single code change**. In CI that's minutes per build, every build.

> 💡 **The rule:** order Dockerfile instructions from *least likely to change* to *most likely to change*.

Revert that experiment before continuing.

---

## Pass 2 — multi-stage build

Look at what's in your image:

```bash
docker run --rm frontend:v1 sh -c "ls -a /app && du -sh /app/node_modules"
docker run --rm frontend:v1 npm --version
```

The image contains npm itself, its cache, and package metadata — **none of which runs your app.** Every extra file is attack surface a vulnerability scanner will flag.

Rewrite `Dockerfile`:

```dockerfile
# ------------------------------------------------------------------------------
# Stage 1 — resolve dependencies
# ------------------------------------------------------------------------------
FROM node:22-alpine AS deps

WORKDIR /app
COPY package*.json ./
RUN npm install --omit=dev --no-audit --no-fund

# ------------------------------------------------------------------------------
# Stage 2 — runtime
# ------------------------------------------------------------------------------
FROM node:22-alpine AS runtime

WORKDIR /app

# Copy ONLY the resolved node_modules from the previous stage.
# npm, its cache and all build metadata stay behind in `deps`.
COPY --from=deps /app/node_modules ./node_modules
COPY package*.json ./
COPY server.js ./
COPY views ./views
COPY public ./public

EXPOSE 3000
CMD ["node", "server.js"]
```

**What `AS deps` and `--from=deps` do:** each `FROM` starts a fresh, independent image. `--from=deps` reaches into a previous stage and copies files out. Everything else in that stage is discarded.

Note the explicit `COPY` of each directory instead of `COPY . .`. This is a *deny-by-default* posture: only listed files enter the image. Add a file to the repo and it won't silently ship.

```bash
docker build -t frontend:v2 .
docker images | grep frontend
```

The image is smaller and contains only what runs.

---

## Pass 3 — stop running as root

```bash
docker run --rm frontend:v2 whoami
```

```text
root
```

**Your web-facing process is root inside the container.** Combined with a container-escape vulnerability, that's root on the host.

The official Node image already ships a non-root user called `node` (uid 1000). Use it:

```dockerfile
FROM node:22-alpine AS runtime

WORKDIR /app

# --chown sets ownership during the copy. Doing it afterwards with
# `RUN chown -R` would duplicate every file into a second layer,
# roughly doubling the image size.
COPY --chown=node:node --from=deps /app/node_modules ./node_modules
COPY --chown=node:node package*.json ./
COPY --chown=node:node server.js ./
COPY --chown=node:node views ./views
COPY --chown=node:node public ./public

# Everything after this line runs unprivileged.
USER node

EXPOSE 3000
CMD ["node", "server.js"]
```

```bash
docker build -t frontend:v3 .
docker run --rm frontend:v3 whoami
```

```text
node
```

> 💡 In Module 04 your Kubernetes `securityContext` sets `runAsNonRoot: true`. A pod whose image only works as root will fail to start with `CreateContainerConfigError`. Fixing it here means it just works there.

---

## Pass 4 — 🔨 Break it: signal handling

Run the container and try to stop it. **Time it.**

```bash
docker run -d --name sigtest -p 3000:3000 frontend:v3
time docker stop sigtest
```

```text
real    0m10.４s        ← TEN SECONDS
```

`docker stop` sends `SIGTERM`, waits 10 seconds, then `SIGKILL`s. Your container ignored the `SIGTERM` entirely and was killed.

**Why?** Your Node process is **PID 1**. In Linux, PID 1 is special: it does not get default signal handlers. A normal process ignoring `SIGTERM` would still be terminated by the kernel default — PID 1 isn't. Node installs no handler, so nothing happens.

Consequences:
- Every deploy takes 10 extra seconds per container
- In-flight HTTP requests are severed mid-response
- In Kubernetes, pods sit `Terminating` through the whole grace period

### The fix — a real init process

```dockerfile
FROM node:22-alpine AS runtime

# tini is a ~10 KB init that correctly forwards signals to its child
# and reaps zombie processes.
RUN apk add --no-cache tini

WORKDIR /app
COPY --chown=node:node --from=deps /app/node_modules ./node_modules
COPY --chown=node:node package*.json ./
COPY --chown=node:node server.js ./
COPY --chown=node:node views ./views
COPY --chown=node:node public ./public

USER node
EXPOSE 3000

# ENTRYPOINT is the fixed part of the command; CMD is the overridable part.
# tini becomes PID 1, receives SIGTERM, and forwards it to node.
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "server.js"]
```

```bash
docker rm -f sigtest
docker build -t frontend:v4 .
docker run -d --name sigtest -p 3000:3000 frontend:v4
time docker stop sigtest
docker rm sigtest
```

```text
real    0m0.6s          ← instant
```

**You just fixed a bug most production Dockerfiles still have.**

<details>
<summary><b>ENTRYPOINT vs CMD — when to use which</b></summary>

```dockerfile
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "server.js"]
```

Docker concatenates them: `/sbin/tini -- node server.js`.

Arguments passed to `docker run` **replace CMD** but never ENTRYPOINT:

```bash
docker run frontend:v4 node --version
# runs: /sbin/tini -- node --version    (tini still PID 1)
```

That's the pattern: ENTRYPOINT for the wrapper you always want, CMD for the default that can be overridden.
</details>

---

## Pass 5 — healthcheck and metadata

Docker has no idea whether your app is *working* — only whether the process exists. A Node process stuck in an infinite loop looks perfectly healthy.

```dockerfile
# syntax=docker/dockerfile:1.7
FROM node:22-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm install --omit=dev --no-audit --no-fund

FROM node:22-alpine AS runtime

LABEL org.opencontainers.image.title="ivolve-frontend" \
      org.opencontainers.image.description="iVolve DevOps Roadmap — web frontend"

RUN apk add --no-cache tini

ENV NODE_ENV=production \
    PORT=3000

WORKDIR /app
COPY --chown=node:node --from=deps /app/node_modules ./node_modules
COPY --chown=node:node package*.json ./
COPY --chown=node:node server.js ./
COPY --chown=node:node views ./views
COPY --chown=node:node public ./public

USER node
EXPOSE 3000

HEALTHCHECK --interval=15s --timeout=5s --start-period=20s --retries=3 \
    CMD node -e "require('http').get('http://127.0.0.1:3000/',r=>process.exit(r.statusCode<500?0:1)).on('error',()=>process.exit(1))"

ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "server.js"]
```

**The healthcheck flags:**

| Flag | Meaning |
|---|---|
| `--interval=15s` | run every 15s |
| `--timeout=5s` | a check taking longer than this counts as failed |
| `--start-period=20s` | failures during startup don't count — prevents restart loops on slow boot |
| `--retries=3` | 3 consecutive failures ⇒ `unhealthy` |

`ENV NODE_ENV=production` matters: Express disables view caching and verbose error pages when it isn't set.

**No curl?** Alpine doesn't ship it and we don't want to add it just for a probe — so the check uses Node itself, which is already there.

```bash
docker build -t ivolve-frontend:local .
docker run -d --name fe -p 3000:3000 ivolve-frontend:local
sleep 25
docker ps --format "table {{.Names}}\t{{.Status}}"
```

```text
NAMES   STATUS
fe      Up 25 seconds (healthy)          ← (healthy)
```

```bash
docker rm -f fe
```

✅ **Frontend done.** Five passes, and you understand every line.

---

<a id="part-2"></a>

# Part 2 · auth-service — native dependencies

Python looks easy until a package contains C code.

## The problem

```bash
cat workspace/src/auth-service/requirements.txt
```

```text
Flask==3.1.1
mysql-connector-python==9.4.0
bcrypt==4.3.0
```

`bcrypt` is a C extension — it must be **compiled**, which needs `gcc`. If you install it naively, gcc and the whole build toolchain end up in your runtime image: ~200 MB of compiler that never runs, and the single largest source of CVE findings in a Python container.

## Add gunicorn first

Flask's built-in server is single-threaded and prints a production warning. Add a real WSGI server:

```bash
echo "gunicorn==23.0.0" >> workspace/src/auth-service/requirements.txt
```

## The Dockerfile

Create `workspace/src/auth-service/Dockerfile`:

```dockerfile
# syntax=docker/dockerfile:1.7
# ------------------------------------------------------------------------------
# Stage 1 — compile every dependency into a wheel
# ------------------------------------------------------------------------------
FROM python:3.12-slim AS builder

WORKDIR /build

RUN apt-get update \
    && apt-get install --no-install-recommends -y build-essential libffi-dev \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .

# `pip wheel` COMPILES each package into a .whl instead of installing it.
# The runtime stage then installs those pre-built wheels with no compiler.
RUN pip wheel --no-cache-dir --wheel-dir /wheels -r requirements.txt

# ------------------------------------------------------------------------------
# Stage 2 — runtime
# ------------------------------------------------------------------------------
FROM python:3.12-slim AS runtime

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

RUN apt-get update \
    && apt-get install --no-install-recommends -y curl tini \
    && rm -rf /var/lib/apt/lists/* \
    && useradd --create-home --shell /usr/sbin/nologin --uid 10001 appuser

WORKDIR /app

COPY --from=builder /wheels /wheels
COPY requirements.txt .

# --no-index forbids reaching out to PyPI. Everything must come from /wheels.
# This makes the build reproducible and air-gappable.
RUN pip install --no-cache-dir --no-index --find-links=/wheels -r requirements.txt \
    && rm -rf /wheels

COPY --chown=appuser:appuser app.py .

USER appuser
EXPOSE 5000

HEALTHCHECK --interval=15s --timeout=5s --start-period=30s --retries=5 \
    CMD curl --fail --silent http://127.0.0.1:5000/health || exit 1

ENTRYPOINT ["/usr/bin/tini", "--"]

CMD ["gunicorn", \
     "--bind", "0.0.0.0:5000", \
     "--workers", "2", \
     "--threads", "4", \
     "--timeout", "60", \
     "--access-logfile", "-", \
     "app:app"]
```

### The two `ENV` lines earn their place

| Variable | Without it |
|---|---|
| `PYTHONDONTWRITEBYTECODE=1` | `.pyc` files litter the filesystem — breaks a read-only root filesystem, which Module 04 enables |
| `PYTHONUNBUFFERED=1` | stdout is block-buffered, so **`docker logs` shows nothing** until the buffer flushes. Looks like a hung container. |

### `app:app` — what that means

`module_name:variable_name`. Gunicorn imports `app.py` and looks for the WSGI callable named `app`:

```python
app = Flask(__name__)     # ← this object
```

`--workers 2 --threads 4` = 8 concurrent requests, instead of Flask's one-at-a-time.

> 💡 **`app.py` has a 30-attempt MySQL wait loop** in its `__main__` block. Gunicorn never executes `__main__`, so that loop is skipped — deliberately. Startup ordering belongs to Compose (`depends_on: service_healthy`) and Kubernetes (initContainer + readiness probe), which are the correct layers for it.

```bash
cd workspace/src/auth-service
docker build -t ivolve-auth-service:local .
```

---

<a id="part-3"></a>

# Part 3 · roadmap-service — the JVM in a container

## The Dockerfile

Create `workspace/src/roadmap-service/Dockerfile`:

```dockerfile
# syntax=docker/dockerfile:1.7
# ------------------------------------------------------------------------------
# Stage 1 — compile and package
# ------------------------------------------------------------------------------
FROM maven:3.9.11-eclipse-temurin-21 AS build

WORKDIR /build

# Resolve dependencies in their own cached layer. A change to a .java file
# then re-runs only the compile — it doesn't re-download Spring Boot.
COPY pom.xml .
RUN mvn -B dependency:go-offline

COPY src ./src
RUN mvn -B clean package -DskipTests

# ------------------------------------------------------------------------------
# Stage 2 — runtime
# ------------------------------------------------------------------------------
FROM eclipse-temurin:21-jre-alpine AS runtime

RUN apk add --no-cache curl tini \
    && addgroup --system --gid 10001 appgroup \
    && adduser  --system --uid 10001 --ingroup appgroup --shell /sbin/nologin appuser

WORKDIR /app
COPY --from=build --chown=appuser:appgroup /build/target/roadmap-service-1.0.0.jar app.jar

USER appuser
EXPOSE 8080

ENV JAVA_OPTS="-XX:MaxRAMPercentage=75.0 -XX:+UseSerialGC -XX:+ExitOnOutOfMemoryError"

HEALTHCHECK --interval=15s --timeout=5s --start-period=45s --retries=5 \
    CMD curl --fail --silent http://127.0.0.1:8080/api/roadmap || exit 1

ENTRYPOINT ["/sbin/tini", "--"]
CMD ["sh", "-c", "exec java $JAVA_OPTS -jar /app/app.jar"]
```

## ⚠️ `MaxRAMPercentage` — the classic "Java in Kubernetes" bug

This flag is the most important line in the file.

**Without it:** the JVM reads the *host's* total RAM to size its heap. On a 64 GB node it happily sets a 16 GB max heap — then the container's 512 MB cgroup limit is exceeded the moment the heap grows, and the kernel OOM-kills the pod. You see `Exit Code 137` and no Java stack trace, because the JVM never got to handle it.

**With it:** the JVM reads the *cgroup* limit and sizes the heap to 75% of it. The remaining 25% covers metaspace, thread stacks and native buffers.

Prove it to yourself:

```bash
cd workspace/src/roadmap-service
docker build -t ivolve-roadmap-service:local .

# Heap sized against the container limit
docker run --rm -m 512m ivolve-roadmap-service:local \
  java -XX:MaxRAMPercentage=75.0 -XX:+PrintFlagsFinal -version | grep -i maxheapsize

# Heap sized against the HOST — the bug
docker run --rm -m 512m ivolve-roadmap-service:local \
  java -XX:+PrintFlagsFinal -version | grep -i maxheapsize
```

The second number will be far larger than 512 MB. **That is the bug, visible.**

**`+ExitOnOutOfMemoryError`:** die immediately on OOM so the orchestrator restarts a clean process, instead of leaving a JVM thrashing the garbage collector forever while still passing a TCP health check.

---

<a id="part-4"></a>

# Part 4 · .dockerignore

Everything in the build directory is uploaded to the Docker daemon as "build context" — *before* any instruction runs.

```bash
cd workspace/src/frontend
npm install 2>/dev/null || true      # create node_modules locally
docker build -t ctx-test . 2>&1 | head -1
```

```text
Sending build context to Docker daemon  48.7MB     ← your local node_modules
```

Three problems: slow, and `COPY . .` would overwrite the container's Linux-built `node_modules` with your host's (which may contain macOS/Windows binaries), and **any local `.env` file gets baked into a layer and pushed to ECR**.

Create `workspace/src/frontend/.dockerignore`:

```gitignore
node_modules/
npm-debug.log*

.env
.env.*
*.pem
*.key

.git/
.gitignore
.dockerignore
Dockerfile
*.md

.vscode/
.idea/
.DS_Store
```

```bash
docker build -t ctx-test . 2>&1 | head -1
```

```text
Sending build context to Docker daemon  1.2MB      ← 40× smaller
```

Now `workspace/src/auth-service/.dockerignore`:

```gitignore
__pycache__/
*.py[cod]
.venv/
venv/
.pytest_cache/

.env
.env.*
*.pem
*.key

.git/
.dockerignore
Dockerfile
*.md
```

And `workspace/src/roadmap-service/.dockerignore`:

```gitignore
# If you ever run `mvn package` locally, a stale JAR here would be copied
# into the build context and could shadow the one the build produces.
target/
.mvn/
mvnw
mvnw.cmd

.env
*.pem
*.key

.git/
.dockerignore
Dockerfile
*.md
*.iml
```

---

<a id="part-5"></a>

# Part 5 · Docker Compose

Four containers that must start in the right order and find each other.

## Pass 1 — MySQL only

Create `workspace/01-Docker/docker-compose.yml`:

```yaml
name: ivolve

services:
  mysql:
    image: mysql:8.0
    container_name: ivolve-mysql
    environment:
      MYSQL_ROOT_PASSWORD: temporary
      MYSQL_DATABASE: ivolve
      MYSQL_USER: ivolve_user
      MYSQL_PASSWORD: temporary
    ports:
      - "127.0.0.1:3306:3306"
```

> ⚠️ **No `version:` key.** It was deprecated in Compose v2 and now emits `WARN the attribute 'version' is obsolete`. Many tutorials still show `version: '3.8'` — they're out of date.

`name: ivolve` sets the project name. Without it Compose derives it from the directory (`01-docker`) and you get containers called `01-docker-mysql-1`.

**`127.0.0.1:3306:3306`** binds to loopback only. Plain `3306:3306` binds `0.0.0.0` — your database reachable by anyone on the café wifi.

```bash
cd workspace/01-Docker
docker compose up -d
docker compose ps
```

---

## Pass 2 — 🔨 Break it: startup ordering

Add auth-service:

```yaml
  auth-service:
    build:
      context: ../src/auth-service
    container_name: ivolve-auth-service
    environment:
      DB_HOST: mysql
      DB_PORT: "3306"
      DB_NAME: ivolve
      DB_USER: ivolve_user
      DB_PASSWORD: temporary
    ports:
      - "127.0.0.1:5000:5000"
    depends_on:
      - mysql          # ← this looks right. It isn't.
```

```bash
docker compose down -v
docker compose up -d --build
sleep 5
docker compose logs auth-service | tail -5
```

```text
Can't connect to MySQL server on 'mysql:3306' (111 Connection refused)
```

**Why?** Plain `depends_on` waits only for the container to be **created** — not for the software inside to be ready. MySQL's first boot initialises its data directory, which takes 30+ seconds. auth-service connects immediately and fails.

### The fix — a healthcheck, then wait on it

```yaml
  mysql:
    image: mysql:8.0
    container_name: ivolve-mysql
    environment:
      MYSQL_ROOT_PASSWORD: temporary
      MYSQL_DATABASE: ivolve
      MYSQL_USER: ivolve_user
      MYSQL_PASSWORD: temporary
    ports:
      - "127.0.0.1:3306:3306"
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h 127.0.0.1 -u root -p\"$$MYSQL_ROOT_PASSWORD\" --silent"]
      interval: 10s
      timeout: 5s
      start_period: 40s      # first boot initialises the data dir
      retries: 10

  auth-service:
    # ...
    depends_on:
      mysql:
        condition: service_healthy      # ← waits for the healthcheck to pass
```

> ⚠️ **`$$MYSQL_ROOT_PASSWORD` has two dollar signs.** Compose performs its own variable substitution first; `$$` escapes it so a literal `$` reaches the shell inside the container. Write `$MYSQL_ROOT_PASSWORD` and Compose substitutes it as an empty string at parse time.

```bash
docker compose down -v
docker compose up -d --build
docker compose ps
```

Both healthy, and auth-service started *after* MySQL was ready.

---

## Pass 3 — the full stack

```yaml
name: ivolve

services:

  mysql:
    image: mysql:8.0
    container_name: ivolve-mysql
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD:?set MYSQL_ROOT_PASSWORD in .env}
      MYSQL_DATABASE: ${MYSQL_DATABASE:-ivolve}
      MYSQL_USER: ${MYSQL_USER:-ivolve_user}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD:?set MYSQL_PASSWORD in .env}
    volumes:
      - mysql-data:/var/lib/mysql
    networks: [ivolve-net]
    ports:
      - "127.0.0.1:3306:3306"
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h 127.0.0.1 -u root -p\"$$MYSQL_ROOT_PASSWORD\" --silent"]
      interval: 10s
      timeout: 5s
      start_period: 40s
      retries: 10

  auth-service:
    build:
      context: ../src/auth-service
      target: runtime
    image: ivolve-auth-service:local
    container_name: ivolve-auth-service
    restart: unless-stopped
    environment:
      DB_HOST: mysql
      DB_PORT: "3306"
      DB_NAME: ${MYSQL_DATABASE:-ivolve}
      DB_USER: ${MYSQL_USER:-ivolve_user}
      DB_PASSWORD: ${MYSQL_PASSWORD:?set MYSQL_PASSWORD in .env}
    networks: [ivolve-net]
    ports:
      - "127.0.0.1:5000:5000"
    depends_on:
      mysql:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "--fail", "--silent", "http://127.0.0.1:5000/health"]
      interval: 15s
      timeout: 5s
      start_period: 30s
      retries: 5

  roadmap-service:
    build:
      context: ../src/roadmap-service
      target: runtime
    image: ivolve-roadmap-service:local
    container_name: ivolve-roadmap-service
    restart: unless-stopped
    networks: [ivolve-net]
    ports:
      - "127.0.0.1:8080:8080"
    healthcheck:
      # NOTE: /api/roadmap, not /health — this service has no /health route.
      test: ["CMD", "curl", "--fail", "--silent", "http://127.0.0.1:8080/api/roadmap"]
      interval: 15s
      timeout: 5s
      start_period: 45s        # JVM + Spring context startup is slow
      retries: 5

  frontend:
    build:
      context: ../src/frontend
      target: runtime
    image: ivolve-frontend:local
    container_name: ivolve-frontend
    restart: unless-stopped
    environment:
      AUTH_SERVICE_URL: http://auth-service:5000
      ROADMAP_SERVICE_URL: http://roadmap-service:8080
      SESSION_SECRET: ${SESSION_SECRET:?set SESSION_SECRET in .env}
    networks: [ivolve-net]
    ports:
      - "3000:3000"            # the ONLY service exposed beyond loopback
    depends_on:
      auth-service:
        condition: service_healthy
      roadmap-service:
        condition: service_healthy

networks:
  ivolve-net:
    name: ivolve-net
    driver: bridge

volumes:
  mysql-data:
    name: ivolve-mysql-data
```

### Three things worth understanding

**1. `${VAR:?message}` vs `${VAR:-default}`**

| Syntax | Behaviour |
|---|---|
| `${VAR:-ivolve}` | use `VAR`, or `ivolve` if unset |
| `${VAR:?message}` | use `VAR`, or **fail immediately** with `message` |

Secrets use `:?` — the stack refuses to start rather than silently running with an empty password.

**2. Service discovery by DNS**

`AUTH_SERVICE_URL: http://auth-service:5000` — the hostname is the *service name*. Compose runs a DNS resolver on the user-defined network that resolves service names to container IPs.

> 💡 This is exactly how it works in Kubernetes too, with a different resolver. That's why moving to Module 04 changes only the hostname — the pattern is identical.

**3. Why a user-defined network**

The default bridge has **no DNS** — containers can only reach each other by IP. A user-defined network (`ivolve-net`) provides automatic name resolution and isolates these containers from unrelated ones.

---

## Pass 4 — secrets out of the file

Create `workspace/01-Docker/.env.example` (committed):

```bash
# Copy to .env and replace every CHANGE_ME.
# Generate values with:  openssl rand -base64 24

MYSQL_ROOT_PASSWORD=CHANGE_ME_root_password
MYSQL_DATABASE=ivolve
MYSQL_USER=ivolve_user
MYSQL_PASSWORD=CHANGE_ME_app_password

# Signs the session cookie. server.js falls back to the literal string
# "change-me-in-k8s" — anyone who knows it can forge a session for any user.
SESSION_SECRET=CHANGE_ME_long_random_secret
```

Create your real `.env` (**never committed**):

```bash
cd workspace/01-Docker
cp .env.example .env

# Generate three strong values and paste them in
openssl rand -base64 24
openssl rand -base64 24
openssl rand -base64 24
```

Make sure it can never be committed:

```bash
cd workspace
cat >> .gitignore <<'EOF'
.env
*.pem
*.key
EOF
```

> 💡 **`.env` is read automatically** by Compose from the same directory as the compose file. No flag needed.

---

<a id="checkpoint"></a>

# ✅ Checkpoint

```bash
cd workspace/01-Docker

# Validate before running — catches YAML and variable errors
docker compose config --quiet && echo "compose valid"

docker compose up -d --build
```

First build takes 3–5 minutes (Maven downloads Spring Boot).

```bash
docker compose ps --format "table {{.Name}}\t{{.Status}}"
```

```text
NAME                     STATUS
ivolve-mysql             Up 2 minutes (healthy)
ivolve-auth-service      Up 1 minute (healthy)
ivolve-roadmap-service   Up 1 minute (healthy)
ivolve-frontend          Up 1 minute (healthy)
```

**All four `(healthy)`.** Now use it:

1. Open <http://localhost:3000>
2. **Sign up** — password must be ≥ 8 characters
3. **Log in**
4. You should see the **DevOps Roadmap** page with 8 topics

### Verify the backends directly

```bash
curl -s http://127.0.0.1:5000/health | jq
```
```json
{ "status": "UP", "database": "connected" }
```

```bash
curl -s http://127.0.0.1:8080/api/roadmap | jq '.[0]'
```
```json
{ "title": "OS", "description": "Learn Linux commands, processes, networking and shell scripting." }
```

### Verify your user reached the database

```bash
docker compose exec mysql \
  mysql -u ivolve_user -p"$(grep MYSQL_PASSWORD .env | cut -d= -f2)" \
  ivolve -e "SELECT id, username, created_at FROM users;"
```

Your account should be listed, with a bcrypt-hashed password you never see.

### Verify persistence

```bash
docker compose down          # NOT -v
docker compose up -d
```

Log in again with the same credentials. It works — the named volume survived. Now:

```bash
docker compose down -v       # -v deletes the volume
docker compose up -d
```

Your account is gone. **That is the difference between `down` and `down -v`**, and it's worth knowing before you type it on something that matters.

---

## 🧹 Clean up

```bash
docker compose down -v
docker image prune -f
```

---

## 🧠 Before you continue

1. Why does `COPY package*.json` come before `COPY . .`?
2. What does `tini` fix, and how did you *observe* the problem?
3. Why does `depends_on: [mysql]` not prevent a connection-refused error?
4. Why `$$MYSQL_ROOT_PASSWORD` and not `$MYSQL_ROOT_PASSWORD`?
5. What does `MaxRAMPercentage` fix? What exit code do you see without it?
6. Why does roadmap-service's healthcheck use `/api/roadmap`?
7. What's the difference between `${VAR:-x}` and `${VAR:?x}`?

---

## 📊 Compare with the reference

```bash
diff workspace/01-Docker/docker-compose.yml \
     ../CloudDevOpsProject/01-Docker/docker-compose.yml
```

Differences are fine — the reference adds resource limits, log rotation, `cap_drop`, and YAML anchors. **Read those additions and decide whether you want them.** Understanding *why* they're there matters more than matching character for character.

---

<div align="center">

**Next → [Module 02 — Terraform](02-terraform.md)** 💰

*Billing starts. Make sure your budget alarm is set.*

</div>
