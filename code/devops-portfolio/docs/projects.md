# Titsly — URL Shortener

**Stack:** Python · Flask · PostgreSQL · Docker · Docker Compose · Jenkins · DigitalOcean  
**Source code:** [github.com/tobianita/Titsly](https://github.com/tobianita/Titsly)

---

## What This Project Is

Titsly is a URL shortening service. A user submits a long URL, the application generates a short code, stores the mapping in a PostgreSQL database, and returns a shortened link. When someone visits the short URL, the app looks up the code and redirects them to the original destination.

The application itself is a Python/Flask backend service. My contribution as the DevOps engineer was taking that service and making it deployment-ready: containerising it, wiring up a database with proper health monitoring, managing secrets and environment configuration, and building an automated pipeline that handles the full path from a code push to a running, verified deployment.

> **Current status:** the pipeline and containers run locally against Docker Compose, and the pipeline is written to deploy to a DigitalOcean droplet as its target. It is not a live production instance yet. Everything below describes the pipeline and configuration as built.

---

## Architecture

The application runs as two Docker containers managed by Docker Compose:

```
                    ┌──────────────────────┐
                    │  DigitalOcean (target)│
                    │      Droplet         │
                    │                      │
  HTTP Request ───► │  ┌────────────────┐  │
  port 5000         │  │  titsly-app    │  │
                    │  │  (Flask / py)  │  │
                    │  └───────┬────────┘  │
                    │          │ SQL       │
                    │  ┌───────▼────────┐  │
                    │  │  titsly-db     │  │
                    │  │  (PostgreSQL)  │  │
                    │  └────────────────┘  │
                    │                      │
                    └──────────────────────┘
```

| Container | Image | Port | Purpose |
|---|---|---|---|
| `titsly-app` | Custom build (Flask) | 5000 | Web server, handles requests and redirects |
| `titsly-db` | `postgres:15` | 5432 | Stores URL shortcode-to-destination mappings |

Both containers run on the same Docker Compose network, which means the web container reaches the database using the service name (`db`) as the hostname rather than an IP address. Docker handles the DNS resolution internally.

---

## Environment Configuration

The web container is connected to the database using a `DATABASE_URL` environment variable passed through Docker Compose:

```yaml
environment:
  DATABASE_URL: postgresql://titsly:titslypass@db:5432/titslydb
```

This approach keeps connection details outside the application code and allows the same Docker image to be pointed at a different database by changing a single environment variable — no code change, no image rebuild.

In a production environment, credentials like these would be injected via Docker secrets or a secrets manager rather than a compose file. This project demonstrates the pattern; the next iteration would remove the credentials from the compose file entirely.

---

## Health Monitoring

### Database Health Check

PostgreSQL is ready to accept connections some seconds after the container starts, not immediately. The health check runs `pg_isready` — PostgreSQL's own built-in connectivity probe — on a 10-second interval with 5 retries:

```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U postgres -d titsly"]
  interval: 10s
  timeout: 5s
  retries: 5
```

The database container is only marked `healthy` once this check passes.

### Application Health Check

The Flask app exposes a `/health` endpoint. Docker polls it every 30 seconds to confirm the web service is responsive:

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
  interval: 30s
  timeout: 10s
  retries: 5
```

You can verify the service is healthy at any time:

```bash
curl http://localhost:5000/health
```

### Service Startup Order

The web container does not start until the database passes its health check:

```yaml
web:
  depends_on:
    db:
      condition: service_healthy
```

`condition: service_healthy` is the key detail here. Without it, `depends_on` only waits for the database container to *start* — not for PostgreSQL itself to be ready. The web service would attempt a database connection immediately, fail, and crash. The health condition enforces true readiness.

---

## Data Persistence

Database data is stored in a named Docker volume:

```yaml
volumes:
  postgres_data:
```

Named volumes live outside the container lifecycle. A `docker-compose down` tears down containers but leaves the volume — and all URL records — intact. Without this, every redeployment would wipe the database.

---

## CI/CD Pipeline

The deployment pipeline is automated with Jenkins and defined in a `Jenkinsfile` at the root of the repository. It runs five stages in sequence:

1. Checkout the latest code from GitHub
2. Verify the workspace (confirm required files are present)
3. Build a fresh Docker image
4. Tear down existing containers and redeploy
5. Verify the new containers are running

A failure in any stage stops the pipeline before a broken build can reach the running deployment.

→ Full pipeline documentation: [CI/CD Pipelines](cicd.md)

---

## Deployment

The pipeline targets a DigitalOcean Droplet, with Jenkins running on the same host and executing the pipeline directly — no separate build agent required for a project of this size. I currently run the full pipeline locally against Docker Compose; the DigitalOcean droplet is the configured deployment target rather than a live production instance at this stage.

Deployment sequence (run by Jenkins):

```bash
# Jenkins runs these commands automatically
docker-compose down || true     # tear down (|| true handles first deploy)
docker-compose up -d            # start fresh containers
docker ps                       # verify containers are running
docker-compose ps               # verify project service status
```

---

## Challenges I Solved

### Web service crashing before the database was ready

Early in the setup, the web container was starting before PostgreSQL had finished initialising. The Flask app would attempt to connect to the database immediately, fail, and the container would exit. The error was intermittent — it happened on first boot or after a full restart — which made it hard to reproduce consistently.

The fix was replacing `depends_on: db` with `depends_on: db: condition: service_healthy`. This held the web container in a waiting state until `pg_isready` confirmed the database was accepting connections. The intermittent crash disappeared.

### Pipeline failing on first deployment

When I first ran the Jenkins pipeline on a fresh server, the `docker-compose down` command returned a non-zero exit code because there were no containers to tear down. Jenkins interpreted this as a failure and stopped the pipeline before building anything.

The fix was appending `|| true` to the teardown command — a standard shell idiom that prevents a non-zero exit from a command from stopping the script when failure is expected and acceptable:

```bash
docker-compose down || true
```

### Managing environment variables across local and production

Running the app locally required a different database URL than the one used in the Docker Compose environment. Initially, the `DATABASE_URL` was hardcoded in different places for different contexts, which was fragile.

The solution was to pass all environment configuration through Docker Compose's `environment` block and ensure the application reads exclusively from environment variables — no hardcoded values in code. Local development uses a `.env` file (excluded from version control); the deployed environment uses the compose file directly.

---

## What I Would Do Differently

**Move credentials to Docker secrets.** The database credentials are currently in the compose file. In a production environment, I would use Docker secrets or a tool like HashiCorp Vault to inject them at runtime.

**Add container resource limits.** The current setup doesn't cap CPU or memory per container. On a shared Droplet, an unconstrained container could starve the other one under load. `mem_limit` and `cpus` in the compose file would address this.

**Separate Jenkins from the application server.** Having Jenkins and the application on the same Droplet means a Jenkins job that goes wrong could affect the running service. The cleaner architecture is a dedicated build server that deploys to a separate application server via SSH.

**Add automated tests to the pipeline.** The current pipeline builds and deploys but has no test stage. Adding a test stage before the build would ensure broken code is caught before an image is even built.

---

## Key DevOps Concepts Demonstrated

- Multi-container orchestration with Docker Compose
- Health-gated service startup with `condition: service_healthy`
- Container-level self-healing with `restart: unless-stopped`
- Environment-based configuration (no hardcoded values in application code)
- Persistent storage with named volumes
- Automated build and deployment with a declarative Jenkins pipeline
- Post-deployment verification as part of the pipeline