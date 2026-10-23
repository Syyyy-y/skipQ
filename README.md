# SkipQ backend

Flask and MongoDB API for diner ordering, vendor fulfilment and sales. Payments and refunds are simulated.

[Frontend repository](https://github.com/ICT381/ict381-2026-t02-tma-frontend-Syyyy-y)

## Setup

Requires Python 3.12 and MongoDB. Run in PowerShell from the backend folder:

```powershell
py -3.12 -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
Copy-Item .env.example .env
```

Edit `.env` for your MongoDB server. Defaults:

```dotenv
MONGODB_HOST=mongodb://localhost:27017/
MONGODB_DB=skipq_dev
```

Keep the database name in `MONGODB_DB`, outside the URI. 

Seed an empty database, then start the API:

```powershell
.\.venv\Scripts\python.exe -m db_seed.seed
.\.venv\Scripts\python.exe -m flask --app app:create_app run --host 127.0.0.1 --port 5000
```

Seeding refuses populated application collections. Seed only once per database.

## Demo accounts

All accounts use password **SkipQ-demo-2026!**.

| Email | Role |
| --- | --- |
| diner1@example.test | Diner |
| diner2@example.test | Second diner |
| vendor1@example.test | Rice Corner — open |
| vendor2@example.test | Noodle House — closed |

The seed creates 4 users, 4 menu items and 11 orders. See report Q2c for coverage.

## API example

With the server running, open another backend terminal:

```powershell
$baseUrl = 'http://127.0.0.1:5000'
$body = @{email='diner1@example.test'; password='SkipQ-demo-2026!'} | ConvertTo-Json
$session = Invoke-RestMethod -Method Post -Uri "$baseUrl/api/user/gettoken" -ContentType 'application/json' -Body $body
$headers = @{Authorization="Bearer $($session.token)"}
Invoke-RestMethod -Uri "$baseUrl/api/diner/stalls" -Headers $headers
```

Tokens expire after eight hours. Signing in again replaces the account’s token; logout revokes it.

## Q4 tests

Functional tests require an empty database beginning `skipq_test_`, separate from development. Fixtures seed it and remove it after the run.

```powershell
.\.venv\Scripts\python.exe -m pip install -r requirements-dev.txt
.\.venv\Scripts\python.exe -m pytest tests/unit -q

$env:TEST_MONGODB_HOST = 'mongodb://localhost:27017/'
$env:TEST_MONGODB_DB = 'skipq_test_q4'
.\.venv\Scripts\python.exe -m pytest tests/functional -q
if ($LASTEXITCODE -ne 0) { throw 'First functional run failed' }
.\.venv\Scripts\python.exe -m pytest tests/functional -q
```

If an interrupted run leaves data behind, choose a new empty test database.

## Q6 setup

In a backend terminal, use a separate database and seed it once:

```powershell
.\.venv\Scripts\python.exe -m pip install -r requirements-q6.txt
.\.venv\Scripts\python.exe -m playwright install chromium
$env:MONGODB_HOST = 'mongodb://localhost:27017/'
$env:MONGODB_DB = 'skipq_q6'
.\.venv\Scripts\python.exe -m db_seed.seed
.\.venv\Scripts\python.exe -m flask --app app:create_app run --host 127.0.0.1 --port 5001
```

In a frontend terminal, using Node.js 22.12+ and pnpm 11:

```powershell
pnpm install --frozen-lockfile
$env:API_PROXY_TARGET = 'http://127.0.0.1:5001'
pnpm dev -- --host 127.0.0.1 --port 5174
```

### Browser test

In another backend terminal, run twice without resetting the database:

```powershell
$env:Q6_BASE_URL = 'http://127.0.0.1:5174'
$env:Q6_BROWSER_OUTPUT = 'q6-browser-results/run-1'
.\.venv\Scripts\python.exe -m pytest -c tests/playwright/pytest.ini tests/playwright -q -s
if ($LASTEXITCODE -ne 0) { throw 'First browser run failed' }

$env:Q6_BROWSER_OUTPUT = 'q6-browser-results/run-2'
.\.venv\Scripts\python.exe -m pytest -c tests/playwright/pytest.ini tests/playwright -q -s
```

Each run creates an order through the UI and verifies both personas through Pending → Preparing → Ready → Collected.

### Load test

Keep the Q6 API running. Do not sign in as `diner1` elsewhere during the test, because that replaces its token.

```powershell
$env:Q6_RUN_LABEL = 'rerun-baseline'
$env:Q6_MANIFEST = 'q6-performance/rerun-baseline-environment.json'
.\.venv\Scripts\python.exe -m locust -f tests/stress/locustfile.py --host http://127.0.0.1:5001 --headless -u 5 -r 1 -t 30s --stop-timeout 15 --only-summary
if ($LASTEXITCODE -ne 0) { throw 'Baseline load failed' }

$env:Q6_RUN_LABEL = 'rerun-main'
$env:Q6_MANIFEST = 'q6-performance/rerun-main-environment.json'
.\.venv\Scripts\python.exe -m locust -f tests/stress/locustfile.py --host http://127.0.0.1:5001 --headless -u 20 -r 5 -t 60s --stop-timeout 15 --only-summary
```

The workload reads one seeded order with a shared token and waits four seconds after each response. Login and setup are excluded from measurements.

Results are saved to `q6-performance/<Q6_RUN_LABEL>_final_stats.csv`. Rerun labels preserve the original results cited in the report.

The retained diagnostic JSON/JSONL files support the original database analysis. Their temporary measurement wrapper is not included; these commands reproduce the HTTP workload only.

## Lab provenance

Adapted from [StaycationX](https://github.com/ICT381/StaycationX). See report Q1b for the source-file mapping and adaptation rationale.

## Demonstration

Video link: [Video](https://github.com/ICT381/ict381-2026-t02-tma-backend-Syyyy-y/tree/main/q7-screencast)