# CI/CD Pipelines

## Titsly — Automated Build, Containerisation & Deployment Pipeline

This page documents the end-to-end CI/CD pipeline I designed and built for the **Titsly** project — a Dockerised Python/Flask URL shortener backed by PostgreSQL, with DigitalOcean as the configured deployment target (currently run locally).

The pipeline automates three things:

- **Build** — produce a fresh Docker image from the current codebase on every run
- **Deploy** — tear down old containers cleanly and start the new ones
- **Verify** — confirm containers are running and healthy before the pipeline closes

---

## Tools

| Tool | Role |
|---|---|
| Jenkins | Pipeline orchestration — defines and runs each build/deploy stage |
| Docker | Builds and runs the application container |
| Docker Compose | Manages multi-container lifecycle, networking, health checks, and startup order |
| DigitalOcean | Deployment target (cloud Droplet / server) |
| GitHub | Source control; Jenkins pulls from the repository on trigger |

---

## Pipeline Flow

The Jenkins pipeline is defined in a `Jenkinsfile` at the root of the repository. It runs sequentially — a failure in any stage stops the pipeline and prevents a broken build from reaching the running environment.

```
Checkout Code → Verify Workspace → Build Docker Images → Deploy Containers → Verify Running Containers
```

---

## Jenkins Pipeline — Stage by Stage

### Stage 1: Checkout Code

```groovy
stage('checkout code') {
    steps {
        echo 'Checking out source code...'
        checkout scm
    }
}
```

Jenkins pulls the latest code from the configured GitHub source. The `checkout scm` instruction reads the source control configuration from the Jenkins job definition rather than hardcoding a branch name — so the pipeline works correctly regardless of which branch triggers it.

---

### Stage 2: Verify Workspace

```groovy
stage('Verify workspace') {
    steps {
        sh '''
            echo "Current directory:"
            pwd
            echo "Files in workspace:"
            ls -l
        '''
    }
}
```

A deliberate sanity check before anything is built. This confirms the workspace was checked out correctly — that the `Dockerfile`, `docker-compose.yml`, and application files are present — and logs a snapshot of the working directory. If a later stage fails, this log entry makes it immediately clear whether the issue was a checkout problem or something further along.

---

### Stage 3: Build Docker Images

```groovy
stage('Build Docker Images') {
    steps {
        sh '''
            docker-compose build
        '''
    }
}
```

Builds a fresh Docker image from the `Dockerfile` in the repository. Every pipeline run produces a clean build — no stale image layers from a previous deployment can mask a broken build. If the Dockerfile has an error, a dependency is missing, or the pip install fails, this stage catches it before any running container is touched.

---

### Stage 4: Deploy Containers

```groovy
stage('Deploy Containers') {
    steps {
        sh '''
            docker-compose down || true
            docker-compose up -d
        '''
    }
}
```

Two commands, both intentional:

- `docker-compose down || true` — tears down any currently running containers. The `|| true` appended to the command prevents the pipeline from failing if no containers are running at the time (for example, on a first deployment or after a previous failure). Without it, running `down` against nothing would return a non-zero exit code and break the build.
- `docker-compose up -d` — starts both the PostgreSQL database container and the web container in detached mode (background). Docker Compose manages the startup sequence using the health check dependency defined in `docker-compose.yml`.

---

### Stage 5: Verify Running Containers

```groovy
stage('Verify Running Containers') {
    steps {
        sh '''
            docker ps
            docker-compose ps
        '''
    }
}
```

Confirms that containers came up successfully after deployment. `docker ps` lists all running containers on the host; `docker-compose ps` shows the status of the services in this specific project. The output is written into the Jenkins build log — so every successful deployment has a timestamped record of exactly what was running when the pipeline completed.

This stage catches a silent failure mode: a container that starts, immediately crashes, and exits — without this check, a pipeline could report success while the application is actually down.

---

## Docker Compose — Service Orchestration

The `docker-compose.yml` defines two services and manages the dependency between them.

### The Problem It Solves

A common failure point in multi-container applications is starting the web service before the database is ready to accept connections. Docker Compose's `depends_on` field tells the web container to wait for the database — but by default, "started" only means the container process launched. It does not mean the database is accepting connections.

The result: the web container starts, attempts a database connection, fails, and crashes — especially on first boot when PostgreSQL needs time to initialise.

### The Fix: Health-Gated Startup

The pipeline uses `condition: service_healthy` to enforce true readiness before the web container starts:

```yaml
web:
  depends_on:
    db:
      condition: service_healthy
```

The web container is held until the `db` service passes its health check. The database health check uses `pg_isready` — PostgreSQL's own built-in connectivity probe:

```yaml
db:
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U postgres -d titsly"]
    interval: 10s
    timeout: 5s
    retries: 5
```

Docker checks every 10 seconds, allows up to 5 retries, and only marks the database as healthy when PostgreSQL is genuinely accepting connections. The web service is blocked until this passes.

### Web Service Health Check

The Flask application exposes a `/health` endpoint that Docker polls to monitor its own status after startup:

```yaml
web:
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
    interval: 30s
    timeout: 10s
    retries: 5
  restart: 'unless-stopped'
```

The `restart: unless-stopped` policy means Docker automatically restarts the web container if it crashes unexpectedly — without needing a new deployment. This provides basic self-healing at the container level.

### Persistent Storage

PostgreSQL data is stored in a named Docker volume:

```yaml
volumes:
  postgres_data:
```

Without a named volume, every `docker-compose down` would destroy the database, including all stored URL records. The named volume persists data across container restarts and redeployments.

---

## Full docker-compose.yml

```yaml
version: "3.9"

services:
  db:
    image: postgres:15
    container_name: titsly-db
    environment:
      POSTGRES_USER: titsly
      POSTGRES_PASSWORD: titslypass
      POSTGRES_DB: titslydb
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d titsly"]
      interval: 10s
      timeout: 5s
      retries: 5

  web:
    build: .
    container_name: titsly-app
    ports:
      - 5000:5000
    environment:
      DATABASE_URL: postgresql://titsly:titslypass@db:5432/titslydb
    depends_on:
      db:
        condition: service_healthy
    restart: 'unless-stopped'
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
      interval: 30s
      timeout: 10s
      retries: 5

volumes:
  postgres_data:
```

---

## What This Pipeline Prevents

| Risk | How the pipeline addresses it |
|---|---|
| Deploying a broken build | Build stage fails fast before any running container is affected |
| Web service crashing on startup because DB isn't ready | `condition: service_healthy` holds the web container until `pg_isready` passes |
| Silent failure after deployment | Verify stage checks container status and logs it into the build record |
| Pipeline failing on first deploy (no containers to tear down) | `docker-compose down \|\| true` handles the empty case gracefully |
| Data loss on redeploy | Named volume persists PostgreSQL data across `docker-compose down` cycles |
| Crashed container requiring manual restart | `restart: unless-stopped` recovers automatically |

---

## What I Learned Building This

**`|| true` matters.** A teardown command that fails on an empty environment will block every subsequent deploy if you don't account for it. Small detail, real consequence.

**`depends_on` alone is not enough.** Using it without `condition: service_healthy` produces intermittent startup failures that are difficult to reproduce because they depend on how fast PostgreSQL initialises — which varies by machine. The health-gated approach is deterministic.

**End on a verify step.** A pipeline that only builds and deploys trusts the containers to be fine. Adding a verification step at the end means the pipeline itself catches the failure — not a user, and not an alert at 2am.

---

*Full source code: [github.com/tobianita/Titsly](https://github.com/tobianita/Titsly)*