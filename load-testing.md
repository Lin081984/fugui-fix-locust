# Load Testing with Locust

This directory contains a **Locust-based load testing suite** for HELPDESK.AI's critical API endpoints. It simulates concurrent users, measures response times under load, and generates benchmarking reports with SLA threshold alerts.

## Quick Start

### Prerequisites

- Python 3.10+
- The backend server running (local or remote)

### Install Locust

```bash
pip install locust
# Or from the load test requirements:
pip install -r backend/tests/load/requirements-load.txt
```

### Run a Quick Smoke Test

```bash
cd backend
bash tests/load/run_load_test.sh --quick
```

This runs 10 users at 2/sec for 60 seconds against `http://localhost:8000`.

### Run a Standard Load Test

```bash
bash tests/load/run_load_test.sh --standard
```

This runs 50 users at 5/sec for 5 minutes.

### Custom Parameters

```bash
bash tests/load/run_load_test.sh -u 100 -r 10 -t 10m -H http://localhost:8000
```

Or using environment variables:

```bash
LOCUST_HOST=http://staging.example.com \
LOCUST_USERS=30 \
LOCUST_SPAWN_RATE=3 \
LOCUST_RUN_TIME=120s \
bash tests/load/run_load_test.sh
```

## What Gets Tested

The load test covers these critical API endpoints:

| Endpoint | Description | Workload Type |
|---|---|---|
| `GET /health` | Service health check | Lightweight read |
| `GET /ready` | Readiness probe | Lightweight read |
| `GET /tickets` | Paginated ticket listing | Read (with DB query) |
| `POST /ai/analyze` | AI ticket analysis (read-only) | ML inference heavy |
| `POST /ai/analyze_ticket` | Legacy AI analysis endpoint | ML inference heavy |
| `POST /tickets/save` | Save ticket to database | Write (with DB insert) |
| `POST /ai/troubleshoot` | Gemini troubleshooting | AI inference heavy |
| `POST /auth/login` | User authentication | Auth with DB |

### User Simulation Profiles

1. **HelpdeskLoadUser** (default) — Sequential workflow simulating a realistic user flow: health check → login → browse tickets → analyze ticket → save ticket → troubleshoot.

2. **ReadOnlyUser** — Lightweight profile doing only health checks and ticket listing. Useful for baseline performance measurement.

3. **HeavyWorkloadUser** — Stress testing profile that focuses on AI analysis and ticket creation with minimal wait times.

## SLA Thresholds

SLA thresholds are defined in `backend/tests/load/sla_config.py`. Default values:

| Endpoint | p50 | p95 | p99 | Max Error |
|---|---|---|---|---|
| `GET /health` | 50ms | 100ms | 200ms | 0% |
| `GET /ready` | 50ms | 100ms | 200ms | 0% |
| `GET /tickets` | 200ms | 500ms | 1000ms | 0.1% |
| `POST /tickets/save` | 300ms | 600ms | 1200ms | 0.1% |
| `POST /ai/analyze` | 500ms | 1000ms | 2000ms | 0.1% |
| `POST /ai/analyze_ticket` | 500ms | 1000ms | 2000ms | 0.1% |
| `POST /ai/troubleshoot` | 1500ms | 3000ms | 5000ms | 0.1% |
| `POST /auth/login` | 800ms | 1500ms | 3000ms | 1% |

To customize, edit `backend/tests/load/sla_config.py`.

## Benchmark Reports

After each test run, the report generator creates:

1. **HTML Report** — `reports/report.html` (from Locust's `--html` flag)
2. **CSV Stats** — `reports/loadtest_stats.csv` (raw data)
3. **JSON Benchmark** — `reports/benchmark.json` (pass/fail per endpoint)

### Generate Report Manually

```bash
python backend/tests/load/report_generator.py \
  --csv-prefix backend/tests/load/reports/loadtest \
  --output backend/tests/load/reports/benchmark.json
```

## Performance Budget

Set the `PERFORMANCE_BUDGET` environment variable to enforce a maximum average response time:

```bash
PERFORMANCE_BUDGET=5000 bash backend/tests/load/run_load_test.sh --standard
```

If any endpoint's average response time exceeds the budget, the run exits with code 1.

## CI/CD Integration

A **GitHub Actions workflow** (`.github/workflows/performance.yml`) runs load tests nightly at 02:00 UTC against the staging deployment.

### Workflow Features

- **Scheduled**: Runs daily at 02:00 UTC
- **Manual trigger**: Via `workflow_dispatch` with custom parameters
- **Artifacts**: HTML report and JSON benchmark are uploaded
- **Performance budget**: Set via `PERFORMANCE_BUDGET` input (default: 10000ms)
- **Failure on breach**: CI fails if performance budget is exceeded

To view results:
1. Go to GitHub → Actions → Nightly Performance Tests
2. Download the `locust-report` artifact
3. Open `report.html` in a browser

## Running with Docker

```bash
docker run --rm -p 8089:8089 \
  -v $(pwd):/mnt/locust \
  locustio/locust \
  -f /mnt/locust/backend/tests/load/locustfile.py \
  --host=http://host.docker.internal:8000
```

Then open http://localhost:8089 for the web UI.

## Architecture Overview

```
backend/tests/load/
├── locustfile.py              # Locust user simulation classes
├── sla_config.py              # SLA thresholds per endpoint
├── report_generator.py        # Benchmark report generator
├── run_load_test.sh           # Convenience runner script
├── requirements-load.txt      # Dependencies for load testing
└── reports/                   # Generated reports (gitignored)
    ├── report.html
    ├── loadtest_stats.csv
    ├── loadtest_stats_history.csv
    └── benchmark.json
```

## Test Data

The load test includes a realistic dataset of support ticket descriptions covering:
- Login & access issues
- Network connectivity problems
- Printer & hardware failures
- Software crashes & installation issues
- VPN & firewall configurations
- Permission and access requests

All test data is embedded in `locustfile.py` — no external dependencies required.
