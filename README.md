# LoomNova

LoomNova is a Java/Spring performance lab for concurrency, stress testing, and CI gatekeeping.

This repository includes:
- GitHub Actions
- Maven (Spring Boot app)
- Docker / Docker Compose
- k6

## App features

- **Concurrency engine**
  - Virtual thread per task (Java 21 when available, safe fallback on older JDKs)
  - Latency simulation endpoint via programmable delay
  - JVM/OS stats endpoint + WebSocket stream every 100ms
- **Functional pages**
  - `GET /` serves the LoomNova dashboard
- **APIs**
  - `POST /api/register`
  - `POST /api/login`
  - `POST /api/simulate`
  - `GET /api/system/stats`
  - `POST /api/gate/validate`
  - `GET /api/health`

## Project memory (set once)

This project now stores real app tuning in `perf/project.env`.
Edit these values once per project:
- endpoint paths (`HEALTH_PATH`, `LOGIN_PATH`)
- login payload (`LOGIN_PAYLOAD`)
- validation (`EXPECTED_STATUS`, `EXPECTED_BODY_CONTAINS`)
- load profile (`K6_*`)

The same values are reused by Docker Compose, k6, and CI.

## Run locally

### 1) Build app
```bash
mvn clean package
```

### 2) Start app + k6 (default `load` profile)
```bash
docker compose -f docker-compose.perf.yml up --abort-on-container-exit --exit-code-from k6
```

### 3) Run smoke or stress profile
```bash
# Bash
K6_PROFILE=smoke docker compose -f docker-compose.perf.yml up --abort-on-container-exit --exit-code-from k6
K6_PROFILE=stress docker compose -f docker-compose.perf.yml up --abort-on-container-exit --exit-code-from k6

# PowerShell
$env:K6_PROFILE="smoke"; docker compose -f docker-compose.perf.yml up --abort-on-container-exit --exit-code-from k6
$env:K6_PROFILE="stress"; docker compose -f docker-compose.perf.yml up --abort-on-container-exit --exit-code-from k6
```

## Key files

- `src/main/resources/static/index.html` : load/stress dashboard with worker loop + chart
- `src/main/java/com/example/perfwebapp/api/PerfController.java` : simulate/stats/gate endpoints
- `src/main/java/com/example/perfwebapp/perf/SystemStatsWebSocketHandler.java` : live system stream
- `perf/project.env` : project-specific perf configuration
- `perf/k6/load-test.js` : k6 script with thresholds
- `.github/workflows/perf-pipeline.yml` : CI pipeline

## Notes

- Default perf API target is `POST /api/login` using the seeded test account.
- Health check endpoint is `GET /api/health`.
- k6 thresholds fail the pipeline if latency/error budget is violated.
- k6 uses `ramping-arrival-rate` for load/stress to model request rate more realistically.
- Tune profile values in `perf/project.env` based on your environment capacity.

## CI stoplight pipeline

- Pull requests run a fast `smoke` profile gate.
- Pushes to `main`, scheduled runs, and manual runs execute `load` and `stress`.
- Any threshold failure breaks the job, preventing unsafe builds from passing.
