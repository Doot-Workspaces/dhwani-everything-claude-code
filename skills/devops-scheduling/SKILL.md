---
name: devops-scheduling
description: DevOps patterns for application scheduling, cron jobs, process management, timer setup, health monitoring, and infrastructure automation for Frappe bench and Node.js services.
origin: DRIS
---

# DevOps Scheduling & Infrastructure

## When to Activate

- Setting up cron jobs and scheduled tasks
- Configuring process managers (Supervisor, PM2, systemd)
- Setting up health monitoring and alerting
- Automating deployments and maintenance windows
- Managing multiple Frappe sites or Node.js services
- Infrastructure provisioning and configuration

## Frappe Scheduler Setup

### Bench Scheduler Configuration

```bash
# Frappe uses its own scheduler via Redis + background workers
# Check scheduler status
bench --site mysite.local scheduler status

# Enable/disable scheduler
bench --site mysite.local scheduler enable
bench --site mysite.local scheduler disable

# Run scheduler manually (for testing)
bench --site mysite.local execute frappe.utils.scheduler.enqueue_events

# View pending jobs
bench --site mysite.local show-pending-jobs
```

### Custom Scheduled Tasks in hooks.py

```python
# hooks.py
scheduler_events = {
    # Run every minute
    "all": [
        "my_app.tasks.process_pending_notifications",
    ],

    # Run every 5 minutes (via cron expression)
    "cron": {
        "*/5 * * * *": [
            "my_app.tasks.sync_external_data",
        ],
        # Daily at 6 AM IST — send reminders
        "0 6 * * *": [
            "my_app.tasks.send_daily_reminders",
        ],
        # Monday 9 AM IST — weekly digest
        "0 9 * * 1": [
            "my_app.tasks.send_weekly_report",
        ],
        # First day of month — monthly report
        "0 8 1 * *": [
            "my_app.tasks.generate_monthly_report",
        ],
        # Every 6 hours — cleanup temp files
        "0 */6 * * *": [
            "my_app.tasks.cleanup_temp_files",
        ],
    },

    # Built-in frequency shortcuts
    "hourly": [
        "my_app.tasks.check_deadlines",
    ],
    "daily": [
        "my_app.tasks.send_daily_digest",
        "my_app.tasks.check_project_deadlines",
        "my_app.tasks.archive_old_logs",
    ],
    "weekly": [
        "my_app.tasks.send_weekly_summary",
    ],
    "monthly": [
        "my_app.tasks.generate_monthly_metrics",
    ],
}
```

### Robust Scheduled Task Implementation

```python
# tasks.py
import frappe
from frappe.utils import now, add_days, today, get_datetime

def send_daily_digest():
    """Send daily project digest to all project managers.

    Scheduled: Daily at 6 AM IST.
    Must be idempotent — safe to run multiple times.
    """
    try:
        # Check if already sent today (idempotency guard)
        already_sent = frappe.db.exists("Communication", {
            "subject": f"Daily Digest - {today()}",
            "communication_type": "Automated Message"
        })
        if already_sent:
            frappe.logger().info("Daily digest already sent today, skipping")
            return

        projects = frappe.get_all("Project",
            filters={"status": "Open"},
            fields=["name", "project_name", "percent_complete", "project_manager"]
        )

        for project in projects:
            if not project.project_manager:
                continue

            overdue_tasks = frappe.db.count("Task", {
                "project": project.name,
                "status": ["not in", ["Completed", "Cancelled"]],
                "exp_end_date": ["<", today()]
            })

            frappe.sendmail(
                recipients=[project.project_manager],
                subject=f"Daily Digest - {today()} - {project.project_name}",
                message=f"""
                    <h3>{project.project_name}</h3>
                    <p>Progress: {project.percent_complete}%</p>
                    <p>Overdue tasks: {overdue_tasks}</p>
                """,
            )

        frappe.db.commit()
        frappe.logger().info(f"Daily digest sent for {len(projects)} projects")

    except Exception as e:
        frappe.log_error(f"Daily digest failed: {str(e)}", "Scheduled Task Error")
        # Don't re-raise — failed scheduled tasks should log, not crash scheduler


def cleanup_temp_files():
    """Clean up temporary files older than 24 hours."""
    try:
        cutoff = add_days(now(), -1)
        old_files = frappe.get_all("File",
            filters={
                "is_private": 1,
                "attached_to_doctype": "",  # Orphaned files
                "creation": ["<", cutoff],
            },
            pluck="name",
            limit=100
        )

        for file_name in old_files:
            frappe.delete_doc("File", file_name, ignore_permissions=True)

        frappe.db.commit()
        frappe.logger().info(f"Cleaned up {len(old_files)} temp files")

    except Exception as e:
        frappe.log_error(f"Cleanup failed: {str(e)}", "Scheduled Task Error")
```

## Node.js Process Management

### PM2 for Amgrant v2 Services

```javascript
// ecosystem.config.js
module.exports = {
  apps: [
    {
      name: 'auth-service',
      script: './services/auth/dist/server.js',
      instances: 2,
      exec_mode: 'cluster',
      env: {
        NODE_ENV: 'production',
        PORT: 3001,
        SERVICE_NAME: 'auth-service',
      },
      max_memory_restart: '500M',
      error_file: './logs/auth-error.log',
      out_file: './logs/auth-out.log',
      merge_logs: true,
      log_date_format: 'YYYY-MM-DD HH:mm:ss Z',
    },
    {
      name: 'project-service',
      script: './services/project/dist/server.js',
      instances: 2,
      exec_mode: 'cluster',
      env: {
        NODE_ENV: 'production',
        PORT: 3002,
        SERVICE_NAME: 'project-service',
      },
      max_memory_restart: '500M',
    },
    {
      name: 'beneficiary-service',
      script: './services/beneficiary/dist/server.js',
      instances: 4,  // More instances — highest traffic
      exec_mode: 'cluster',
      env: {
        NODE_ENV: 'production',
        PORT: 3003,
        SERVICE_NAME: 'beneficiary-service',
      },
      max_memory_restart: '1G',
    },
    {
      name: 'reports-service',
      script: './services/reports/dist/server.js',
      instances: 1,  // Reports are less frequent
      env: {
        NODE_ENV: 'production',
        PORT: 3004,
        SERVICE_NAME: 'reports-service',
      },
      max_memory_restart: '1G',  // Reports can use more memory
    },
    // Scheduled jobs runner
    {
      name: 'scheduler',
      script: './services/scheduler/dist/index.js',
      instances: 1,  // MUST be single instance
      cron_restart: '0 */6 * * *',  // Restart every 6 hours for memory
      env: {
        NODE_ENV: 'production',
        SERVICE_NAME: 'scheduler',
      },
    },
  ],
};
```

```bash
# PM2 Commands
pm2 start ecosystem.config.js          # Start all services
pm2 restart all                          # Restart all
pm2 restart auth-service                 # Restart specific service
pm2 reload all                           # Zero-downtime reload
pm2 stop all                             # Stop all
pm2 delete all                           # Remove all from PM2

# Monitoring
pm2 status                               # Service status
pm2 monit                                # Real-time dashboard
pm2 logs                                 # Tail all logs
pm2 logs auth-service --lines 50         # Specific service logs

# Save and auto-start on reboot
pm2 save
pm2 startup                              # Generate startup script
```

### Node.js Cron Jobs with node-cron

```typescript
// services/scheduler/index.ts
import cron from 'node-cron';
import { logger } from '../shared/logger';

// Validate cron expressions at startup
const schedules = [
  {
    name: 'sync-external-data',
    expression: '*/15 * * * *',  // Every 15 minutes
    task: syncExternalData,
  },
  {
    name: 'daily-backup',
    expression: '0 2 * * *',    // 2 AM daily
    task: triggerBackup,
  },
  {
    name: 'weekly-report',
    expression: '0 9 * * 1',    // Monday 9 AM
    task: generateWeeklyReport,
  },
  {
    name: 'cleanup-sessions',
    expression: '0 */4 * * *',  // Every 4 hours
    task: cleanupExpiredSessions,
  },
];

for (const schedule of schedules) {
  if (!cron.validate(schedule.expression)) {
    logger.error(`Invalid cron expression for ${schedule.name}: ${schedule.expression}`);
    continue;
  }

  cron.schedule(schedule.expression, async () => {
    const start = Date.now();
    logger.info(`[SCHEDULER] Starting: ${schedule.name}`);
    try {
      await schedule.task();
      logger.info(`[SCHEDULER] Completed: ${schedule.name} (${Date.now() - start}ms)`);
    } catch (error) {
      logger.error(`[SCHEDULER] Failed: ${schedule.name}`, { error });
      // Alert on critical job failure
      if (['daily-backup', 'sync-external-data'].includes(schedule.name)) {
        await sendAlert(`Scheduled job failed: ${schedule.name}`, error);
      }
    }
  });

  logger.info(`[SCHEDULER] Registered: ${schedule.name} (${schedule.expression})`);
}
```

## Supervisor Configuration (Frappe)

```ini
; /etc/supervisor/conf.d/frappe-bench.conf

[program:frappe-bench-web]
command=/home/deploy/frappe-bench/env/bin/gunicorn
    -b 0.0.0.0:8000
    -w 9
    --timeout 120
    --graceful-timeout 30
    --max-requests 5000
    --max-requests-jitter 500
    frappe.app:application
directory=/home/deploy/frappe-bench/sites
user=deploy
autostart=true
autorestart=true
stopwaitsecs=30
stdout_logfile=/home/deploy/frappe-bench/logs/web.log
stderr_logfile=/home/deploy/frappe-bench/logs/web.error.log

[program:frappe-bench-worker-default]
command=/home/deploy/frappe-bench/env/bin/python -m frappe.utils.background_jobs
    --queue default
directory=/home/deploy/frappe-bench/sites
user=deploy
autostart=true
autorestart=true
numprocs=2
process_name=%(program_name)s-%(process_num)02d
stdout_logfile=/home/deploy/frappe-bench/logs/worker-default.log

[program:frappe-bench-worker-long]
command=/home/deploy/frappe-bench/env/bin/python -m frappe.utils.background_jobs
    --queue long
directory=/home/deploy/frappe-bench/sites
user=deploy
autostart=true
autorestart=true
numprocs=1
stdout_logfile=/home/deploy/frappe-bench/logs/worker-long.log

[program:frappe-bench-scheduler]
command=/home/deploy/frappe-bench/env/bin/python -m frappe.utils.scheduler
directory=/home/deploy/frappe-bench/sites
user=deploy
autostart=true
autorestart=true
numprocs=1
stdout_logfile=/home/deploy/frappe-bench/logs/scheduler.log

[group:frappe-bench]
programs=frappe-bench-web,frappe-bench-worker-default,frappe-bench-worker-long,frappe-bench-scheduler
```

## systemd Timer Setup

```ini
; /etc/systemd/system/frappe-backup.service
[Unit]
Description=Frappe Bench Backup
After=network.target

[Service]
Type=oneshot
User=deploy
WorkingDirectory=/home/deploy/frappe-bench
ExecStart=/home/deploy/frappe-bench/env/bin/bench --site production.dris.com backup --with-files
StandardOutput=journal
StandardError=journal

; /etc/systemd/system/frappe-backup.timer
[Unit]
Description=Run Frappe Backup Daily at 2 AM

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true
RandomizedDelaySec=300

[Install]
WantedBy=timers.target
```

```bash
# Enable and start timer
sudo systemctl enable frappe-backup.timer
sudo systemctl start frappe-backup.timer

# Check timer status
sudo systemctl list-timers | grep frappe

# View logs
journalctl -u frappe-backup.service --since today
```

## Health Monitoring & Alerting

### Monitoring Script

```bash
#!/bin/bash
# scripts/health-monitor.sh
# Run via cron every 5 minutes: */5 * * * * /home/deploy/scripts/health-monitor.sh

set -euo pipefail

SITES=("production.dris.com" "staging.dris.com")
SERVICES=("auth-service:3001" "project-service:3002" "beneficiary-service:3003")
ALERT_WEBHOOK="${SLACK_WEBHOOK_URL:-}"

check_frappe_site() {
    local site="$1"
    local status
    status=$(curl -s -o /dev/null -w "%{http_code}" "https://${site}/api/method/frappe.client.get_count?doctype=User" --max-time 10)
    if [ "$status" != "200" ]; then
        send_alert "Frappe site ${site} is DOWN (HTTP ${status})"
        return 1
    fi
}

check_node_service() {
    local service="$1"
    local name="${service%%:*}"
    local port="${service##*:}"
    local status
    status=$(curl -s -o /dev/null -w "%{http_code}" "http://localhost:${port}/health" --max-time 5)
    if [ "$status" != "200" ]; then
        send_alert "Node service ${name} is DOWN (HTTP ${status})"
        return 1
    fi
}

check_disk_space() {
    local usage
    usage=$(df -h / | awk 'NR==2 {print $5}' | tr -d '%')
    if [ "$usage" -gt 85 ]; then
        send_alert "Disk usage at ${usage}% — cleanup needed"
    fi
}

check_memory() {
    local available
    available=$(free -m | awk 'NR==2 {printf "%.0f", $7/$2*100}')
    if [ "$available" -lt 15 ]; then
        send_alert "Memory available: ${available}% — investigate high usage"
    fi
}

send_alert() {
    local message="$1"
    echo "[ALERT] $(date): ${message}"

    if [ -n "$ALERT_WEBHOOK" ]; then
        curl -s -X POST "$ALERT_WEBHOOK" \
            -H 'Content-type: application/json' \
            -d "{\"text\": \"[DRIS Monitor] ${message}\"}"
    fi
}

# Run all checks
for site in "${SITES[@]}"; do check_frappe_site "$site" || true; done
for service in "${SERVICES[@]}"; do check_node_service "$service" || true; done
check_disk_space
check_memory
```

## Application-Specific Setup Helpers

### Setup New Frappe Site

```bash
#!/bin/bash
# scripts/setup-frappe-site.sh
set -euo pipefail

SITE_NAME="$1"
ADMIN_PASSWORD="$2"

echo "=== Setting up new Frappe site: ${SITE_NAME} ==="

cd /home/deploy/frappe-bench

# Create site
bench new-site "$SITE_NAME" \
    --db-type postgres \
    --admin-password "$ADMIN_PASSWORD" \
    --install-app amgrant

# Configure site
bench --site "$SITE_NAME" set-config developer_mode 0
bench --site "$SITE_NAME" set-config rate_limit '{"window": 86400, "limit": 1000}'
bench --site "$SITE_NAME" set-config session_expiry "06:00"

# Setup scheduler
bench --site "$SITE_NAME" scheduler enable

# Run initial setup
bench --site "$SITE_NAME" migrate
bench --site "$SITE_NAME" clear-cache
bench build

echo "=== Site ${SITE_NAME} is ready ==="
```

### Setup Node.js Service

```bash
#!/bin/bash
# scripts/setup-node-service.sh
set -euo pipefail

SERVICE_NAME="$1"
PORT="$2"

echo "=== Setting up Node.js service: ${SERVICE_NAME} on port ${PORT} ==="

cd /home/deploy/amgrant-v2/services/$SERVICE_NAME

# Install dependencies
npm ci --production

# Build
npm run build

# Register with PM2
pm2 start dist/server.js \
    --name "$SERVICE_NAME" \
    --instances 2 \
    --exec-mode cluster \
    --max-memory-restart 500M \
    -- --port "$PORT"

pm2 save

echo "=== Service ${SERVICE_NAME} running on port ${PORT} ==="
```
