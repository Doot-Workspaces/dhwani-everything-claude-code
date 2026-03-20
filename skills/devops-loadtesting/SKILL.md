---
name: devops-loadtesting
description: DevOps patterns for load testing (k6, Locust), CI/CD pipelines, infrastructure monitoring, deployment automation, and performance benchmarking for Frappe applications on PostgreSQL.
origin: DRIS
---

# DevOps & Load Testing Patterns

## When to Activate

- Setting up load tests for Frappe applications
- Configuring CI/CD pipelines for bench-based deployments
- Performance benchmarking before go-live
- Infrastructure monitoring and alerting
- Deployment automation for Frappe sites
- Capacity planning for government/CSR projects

## Load Testing with k6

### Frappe API Load Test

```javascript
// load-tests/frappe-api.js
import http from 'k6/http';
import { check, sleep, group } from 'k6';
import { Rate, Trend } from 'k6/metrics';

// Custom metrics
const errorRate = new Rate('errors');
const loginDuration = new Trend('login_duration');
const apiDuration = new Trend('api_call_duration');

export const options = {
  stages: [
    { duration: '1m', target: 10 },   // Ramp up to 10 users
    { duration: '3m', target: 50 },   // Ramp up to 50 users
    { duration: '5m', target: 50 },   // Stay at 50 users
    { duration: '2m', target: 100 },  // Peak load
    { duration: '1m', target: 0 },    // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<3000'],  // 95th percentile under 3s
    errors: ['rate<0.05'],              // Less than 5% errors
    login_duration: ['p(95)<2000'],     // Login under 2s
    api_call_duration: ['p(95)<1500'],  // API calls under 1.5s
  },
};

const BASE_URL = __ENV.SITE_URL || 'http://localhost:8000';

// Setup: Login and get session cookie
export function setup() {
  const loginRes = http.post(`${BASE_URL}/api/method/login`, {
    usr: __ENV.TEST_USER || 'admin@dris.com',
    pwd: __ENV.TEST_PASSWORD,
  });

  check(loginRes, { 'login successful': (r) => r.status === 200 });

  return {
    cookies: loginRes.cookies,
  };
}

export default function (data) {
  const params = {
    cookies: data.cookies,
    headers: { 'Content-Type': 'application/json' },
  };

  group('List View Load', () => {
    const start = Date.now();
    const res = http.get(
      `${BASE_URL}/api/resource/Project?fields=["name","project_name","status"]&limit_page_length=20`,
      params
    );
    apiDuration.add(Date.now() - start);
    check(res, { 'list view 200': (r) => r.status === 200 });
    errorRate.add(res.status !== 200);
  });

  group('Form View Load', () => {
    const start = Date.now();
    const res = http.get(
      `${BASE_URL}/api/resource/Project/PROJ-001`,
      params
    );
    apiDuration.add(Date.now() - start);
    check(res, { 'form view 200': (r) => r.status === 200 });
    errorRate.add(res.status !== 200);
  });

  group('Report Generation', () => {
    const start = Date.now();
    const res = http.get(
      `${BASE_URL}/api/method/frappe.desk.query_report.run?report_name=Project%20Summary&filters={}`,
      params
    );
    apiDuration.add(Date.now() - start);
    check(res, { 'report 200': (r) => r.status === 200 });
    errorRate.add(res.status !== 200);
  });

  group('Dashboard API', () => {
    const start = Date.now();
    const res = http.get(
      `${BASE_URL}/api/method/my_app.api.get_project_dashboard?project=PROJ-001`,
      params
    );
    apiDuration.add(Date.now() - start);
    check(res, { 'dashboard 200': (r) => r.status === 200 });
    errorRate.add(res.status !== 200);
  });

  sleep(1); // Think time between actions
}
```

### Concurrent User Simulation

```javascript
// load-tests/concurrent-workflows.js
import http from 'k6/http';
import { check, sleep, group } from 'k6';

export const options = {
  scenarios: {
    // Simulate different user types concurrently
    project_managers: {
      executor: 'constant-vus',
      vus: 10,
      duration: '5m',
      exec: 'pmWorkflow',
    },
    data_entry: {
      executor: 'constant-vus',
      vus: 30,
      duration: '5m',
      exec: 'dataEntryWorkflow',
    },
    report_viewers: {
      executor: 'ramping-vus',
      startVUs: 5,
      stages: [
        { duration: '2m', target: 20 },
        { duration: '3m', target: 20 },
      ],
      exec: 'reportViewerWorkflow',
    },
  },
  thresholds: {
    'http_req_duration{scenario:project_managers}': ['p(95)<3000'],
    'http_req_duration{scenario:data_entry}': ['p(95)<2000'],
    'http_req_duration{scenario:report_viewers}': ['p(95)<5000'],
  },
};

const BASE_URL = __ENV.SITE_URL || 'http://localhost:8000';

export function pmWorkflow() {
  // PM: View project → check tasks → update status
  group('PM: View Project', () => {
    http.get(`${BASE_URL}/api/resource/Project?limit=10`);
    sleep(2);
  });
  group('PM: Check Tasks', () => {
    http.get(`${BASE_URL}/api/resource/Task?filters=[["project","=","PROJ-001"]]`);
    sleep(1);
  });
  sleep(3); // Think time
}

export function dataEntryWorkflow() {
  // Data entry: Create records in bulk
  group('Data Entry: Create Record', () => {
    const payload = JSON.stringify({
      doctype: 'My Doctype',
      project: 'PROJ-001',
      description: `Load test entry ${Date.now()}`,
    });
    http.post(`${BASE_URL}/api/resource/My Doctype`, payload, {
      headers: { 'Content-Type': 'application/json' },
    });
    sleep(0.5);
  });
}

export function reportViewerWorkflow() {
  // Report viewers: Heavy read queries
  group('Report: Project Summary', () => {
    http.get(`${BASE_URL}/api/method/frappe.desk.query_report.run?report_name=Project%20Summary`);
    sleep(5);
  });
}
```

### Running k6 Tests

```bash
# Basic run
k6 run load-tests/frappe-api.js -e SITE_URL=https://staging.dris.com -e TEST_PASSWORD=secret

# With HTML report
k6 run load-tests/frappe-api.js --out json=results.json
# Then: k6-reporter results.json -o report.html

# Cloud run (k6 Cloud)
k6 cloud load-tests/frappe-api.js

# With Grafana/InfluxDB output
k6 run load-tests/frappe-api.js --out influxdb=http://localhost:8086/k6
```

## Load Testing with Locust (Python)

```python
# load-tests/locustfile.py
from locust import HttpUser, task, between, tag
import json

class FrappeUser(HttpUser):
    wait_time = between(1, 5)

    def on_start(self):
        """Login on start."""
        self.client.post("/api/method/login", data={
            "usr": "admin@dris.com",
            "pwd": self.environment.parsed_options.test_password,
        })

    @tag("read")
    @task(3)  # Weight: 3x more likely than write tasks
    def view_project_list(self):
        self.client.get("/api/resource/Project?limit_page_length=20",
                       name="/api/resource/Project [LIST]")

    @tag("read")
    @task(2)
    def view_project_detail(self):
        self.client.get("/api/resource/Project/PROJ-001",
                       name="/api/resource/Project [DETAIL]")

    @tag("read")
    @task(1)
    def run_report(self):
        self.client.get(
            "/api/method/frappe.desk.query_report.run",
            params={"report_name": "Project Summary"},
            name="/api/method/query_report [REPORT]"
        )

    @tag("write")
    @task(1)
    def create_task(self):
        self.client.post("/api/resource/Task", json={
            "subject": f"Load test task",
            "project": "PROJ-001",
            "status": "Open",
        }, name="/api/resource/Task [CREATE]")

    @tag("dashboard")
    @task(2)
    def view_dashboard(self):
        self.client.get(
            "/api/method/my_app.api.get_project_dashboard",
            params={"project": "PROJ-001"},
            name="/api/method/dashboard [CUSTOM]"
        )
```

```bash
# Run Locust
locust -f load-tests/locustfile.py --host=https://staging.dris.com --test-password=secret

# Headless mode for CI
locust -f load-tests/locustfile.py --host=https://staging.dris.com \
  --headless --users 50 --spawn-rate 5 --run-time 10m \
  --csv=results/load-test --html=results/report.html
```

## CI/CD Pipeline for Frappe

### GitHub Actions Pipeline

```yaml
# .github/workflows/deploy.yml
name: Frappe CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: test_password
          POSTGRES_DB: test_site
        ports: ['5432:5432']
      redis:
        image: redis:7
        ports: ['6379:6379']

    steps:
      - uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install Frappe Bench
        run: |
          pip install frappe-bench
          bench init --frappe-branch version-15 --python python3.11 frappe-bench
          cd frappe-bench
          bench get-app --branch main ${{ github.workspace }}
          bench new-site test.localhost --db-type postgres --db-host localhost \
            --db-password test_password --admin-password admin --install-app my_app

      - name: Run Tests
        run: |
          cd frappe-bench
          bench --site test.localhost run-tests --app my_app --failfast

  load-test:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: grafana/k6-action@v0.3.1
        with:
          filename: load-tests/frappe-api.js
        env:
          SITE_URL: ${{ secrets.STAGING_URL }}
          TEST_PASSWORD: ${{ secrets.STAGING_PASSWORD }}

  deploy-staging:
    needs: [test, load-test]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Staging
        run: |
          ssh deploy@staging.dris.com "cd /home/deploy/frappe-bench && \
            bench update --pull --reset && \
            bench --site staging.dris.com migrate && \
            bench --site staging.dris.com clear-cache && \
            sudo supervisorctl restart all"
```

## Performance Monitoring

### PostgreSQL Query Monitoring

```sql
-- Find slow queries (run on PostgreSQL)
SELECT
  query,
  calls,
  mean_exec_time::numeric(10,2) as avg_ms,
  total_exec_time::numeric(10,2) as total_ms,
  rows
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;

-- Table bloat check
SELECT
  schemaname || '.' || relname as table_name,
  pg_size_pretty(pg_total_relation_size(relid)) as total_size,
  n_dead_tup as dead_rows,
  n_live_tup as live_rows,
  round(n_dead_tup::numeric / NULLIF(n_live_tup, 0) * 100, 2) as dead_pct
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 10;

-- Missing indexes
SELECT
  relname as table_name,
  seq_scan,
  idx_scan,
  seq_scan - idx_scan as too_many_seq_scans,
  pg_size_pretty(pg_relation_size(relid)) as table_size
FROM pg_stat_user_tables
WHERE seq_scan > idx_scan AND pg_relation_size(relid) > 1000000
ORDER BY seq_scan - idx_scan DESC;
```

### Frappe Performance Checklist

```bash
# Check Redis status
bench doctor

# Monitor background jobs
bench --site mysite.local show-pending-jobs

# Check scheduler status
bench --site mysite.local scheduler status

# Database size
bench --site mysite.local backup --with-files 2>&1 | tail -1

# Process monitoring
bench --site mysite.local doctor
```

## Infrastructure Benchmarks

### Pre-Go-Live Checklist

| Metric | Target | How to Test |
|--------|--------|-------------|
| Page load time | < 3s | k6 + browser metrics |
| API response (p95) | < 1.5s | k6 thresholds |
| Concurrent users | 50-100 | k6 ramping VUs |
| Database query time | < 100ms avg | pg_stat_statements |
| Background job throughput | > 100/min | bench show-pending-jobs |
| Backup time | < 15 min | bench backup --with-files |
| Recovery time | < 30 min | Full restore test |
| SSL certificate | Valid | curl -vI https://site |
| Uptime target | 99.5% | Monitoring alerts |

### Capacity Planning

```bash
# Estimate server requirements
# Rule of thumb for Frappe on PostgreSQL:

# Small (< 20 concurrent users):
#   2 vCPU, 4GB RAM, 50GB SSD
#   1 gunicorn worker per CPU

# Medium (20-50 concurrent users):
#   4 vCPU, 8GB RAM, 100GB SSD
#   2 gunicorn workers per CPU, Redis 1GB

# Large (50-200 concurrent users):
#   8 vCPU, 16GB RAM, 200GB SSD
#   Separate DB server, Redis 2GB, 4 workers

# Gunicorn workers config (common_site_config.json)
# "gunicorn_workers": 9  # (2 * CPU) + 1
```

## Deployment Automation

### Bench Update Script

```bash
#!/bin/bash
# scripts/deploy.sh — Safe deployment script

set -euo pipefail

SITE="$1"
BENCH_DIR="/home/deploy/frappe-bench"

echo "=== Deploying to ${SITE} ==="

cd "$BENCH_DIR"

# Pre-deploy backup
echo "Taking backup..."
bench --site "$SITE" backup --with-files

# Pull latest code
echo "Pulling updates..."
bench update --pull --no-backup

# Run migrations
echo "Running migrations..."
bench --site "$SITE" migrate

# Build assets
echo "Building assets..."
bench build

# Clear cache
echo "Clearing cache..."
bench --site "$SITE" clear-cache

# Restart services
echo "Restarting services..."
sudo supervisorctl restart all

# Health check
echo "Health check..."
HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" "https://${SITE}/api/method/frappe.client.get_count?doctype=User")
if [ "$HTTP_STATUS" -eq 200 ]; then
    echo "Deployment successful! Site is healthy."
else
    echo "WARNING: Health check returned HTTP ${HTTP_STATUS}"
    exit 1
fi
```

## Monitoring & Alerting

### Health Check Endpoint

```python
# my_app/api.py
import frappe

@frappe.whitelist(allow_guest=True)
def health_check():
    """Lightweight health check for load balancers and monitoring."""
    try:
        # Check DB connectivity
        frappe.db.sql("SELECT 1")

        # Check Redis
        frappe.cache().set_value("health_check", "ok", expires_in_sec=10)
        assert frappe.cache().get_value("health_check") == "ok"

        # Check background workers
        from frappe.utils.background_jobs import get_queue
        default_queue = get_queue("default")
        workers = default_queue.worker_count

        return {
            "status": "healthy",
            "database": "connected",
            "redis": "connected",
            "workers": workers,
            "site": frappe.local.site,
        }
    except Exception as e:
        frappe.local.response.http_status_code = 503
        return {"status": "unhealthy", "error": str(e)}
```
