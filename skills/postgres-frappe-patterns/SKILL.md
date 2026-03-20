---
name: postgres-frappe-patterns
description: PostgreSQL optimization patterns for Frappe applications including indexing strategies, query optimization, migration patterns, backup automation, and performance tuning for government-scale deployments.
origin: DRIS
---

# PostgreSQL Patterns for Frappe

## When to Activate

- Optimizing slow Frappe queries on PostgreSQL
- Designing database schema for custom DocTypes
- Writing data migration patches
- Setting up backup and recovery procedures
- Performance tuning for production Frappe sites
- Capacity planning for government project databases

## Query Optimization

### Identify Slow Queries

```sql
-- Enable pg_stat_statements (once, in postgresql.conf)
-- shared_preload_libraries = 'pg_stat_statements'

-- Top 10 slowest queries
SELECT
  left(query, 100) as query_preview,
  calls,
  round(mean_exec_time::numeric, 2) as avg_ms,
  round(total_exec_time::numeric, 2) as total_ms,
  rows
FROM pg_stat_statements
WHERE query NOT LIKE '%pg_stat%'
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Queries with most total time (optimization candidates)
SELECT
  left(query, 100) as query_preview,
  calls,
  round(total_exec_time::numeric / 1000, 2) as total_seconds,
  round(mean_exec_time::numeric, 2) as avg_ms
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

### Frappe-Specific Query Patterns

```sql
-- Find N+1 query patterns (many calls, low rows)
SELECT
  left(query, 80) as query_preview,
  calls,
  rows,
  round(rows::numeric / calls, 1) as rows_per_call,
  round(mean_exec_time::numeric, 2) as avg_ms
FROM pg_stat_statements
WHERE calls > 100 AND rows / NULLIF(calls, 0) <= 1
ORDER BY calls DESC
LIMIT 20;

-- Common Frappe table patterns to index
-- tabProject: status, company, modified (list view defaults)
-- tabTask: project, status, assigned_to
-- tabFile: attached_to_doctype, attached_to_name
-- tabComment: reference_doctype, reference_name
-- tabVersion: ref_doctype, docname
```

### Index Strategy for Frappe

```sql
-- Check existing indexes on a Frappe table
SELECT
  indexname,
  indexdef
FROM pg_indexes
WHERE tablename = 'tabProject'
ORDER BY indexname;

-- Recommended indexes for common Frappe patterns

-- List view filtering (most common queries)
CREATE INDEX CONCURRENTLY IF NOT EXISTS
  idx_tabproject_status_company
  ON "tabProject" (status, company);

-- Modified-based sorting (default list view order)
CREATE INDEX CONCURRENTLY IF NOT EXISTS
  idx_tabproject_modified_desc
  ON "tabProject" (modified DESC);

-- Task lookup by project (child-like relationship)
CREATE INDEX CONCURRENTLY IF NOT EXISTS
  idx_tabtask_project_status
  ON "tabTask" (project, status);

-- Full-text search on name/title fields
CREATE INDEX CONCURRENTLY IF NOT EXISTS
  idx_tabproject_name_trgm
  ON "tabProject" USING gin (name gin_trgm_ops);

-- Date range queries (common in reports)
CREATE INDEX CONCURRENTLY IF NOT EXISTS
  idx_tabtask_creation_date
  ON "tabTask" (creation);
```

### Query Builder vs Raw SQL

```python
# PREFERRED: Use frappe.qb for type safety
Project = frappe.qb.DocType("Project")
Task = frappe.qb.DocType("Task")

# Join query with aggregation
results = (
    frappe.qb.from_(Project)
    .left_join(Task).on(Task.project == Project.name)
    .select(
        Project.name,
        Project.project_name,
        Project.status,
        frappe.qb.functions.Count(Task.name).as_("task_count"),
        frappe.qb.functions.Sum(
            frappe.qb.terms.Case()
            .when(Task.status == "Completed", 1)
            .else_(0)
        ).as_("completed_tasks"),
    )
    .where(Project.company == "Dhwani RIS")
    .groupby(Project.name)
    .orderby(Project.modified, order=frappe.qb.desc)
    .limit(50)
    .run(as_dict=True)
)

# WHEN RAW SQL IS NEEDED: Complex CTEs, window functions
results = frappe.db.sql("""
    WITH project_metrics AS (
        SELECT
            p.name,
            p.project_name,
            COUNT(t.name) as total_tasks,
            SUM(CASE WHEN t.status = 'Completed' THEN 1 ELSE 0 END) as done,
            ROUND(
                SUM(CASE WHEN t.status = 'Completed' THEN 1.0 ELSE 0 END)
                / NULLIF(COUNT(t.name), 0) * 100, 1
            ) as completion_pct
        FROM "tabProject" p
        LEFT JOIN "tabTask" t ON t.project = p.name
        WHERE p.company = %(company)s
        GROUP BY p.name, p.project_name
    )
    SELECT *,
        RANK() OVER (ORDER BY completion_pct DESC) as rank
    FROM project_metrics
    ORDER BY completion_pct DESC
""", {"company": "Dhwani RIS"}, as_dict=True)
```

## Migration Patterns

### Safe Migration Patch

```python
# patches/v2_0/add_project_metrics_fields.py
import frappe

def execute():
    """Add computed metric fields to Project DocType.

    This patch:
    1. Adds custom fields if they don't exist
    2. Backfills data from existing records
    3. Is idempotent (safe to run multiple times)
    """

    # Step 1: Add fields (idempotent)
    custom_fields = {
        "Project": [
            {
                "fieldname": "total_tasks",
                "label": "Total Tasks",
                "fieldtype": "Int",
                "insert_after": "percent_complete",
                "read_only": 1,
            },
            {
                "fieldname": "overdue_tasks",
                "label": "Overdue Tasks",
                "fieldtype": "Int",
                "insert_after": "total_tasks",
                "read_only": 1,
            },
        ]
    }

    from frappe.custom.doctype.custom_field.custom_field import create_custom_fields
    create_custom_fields(custom_fields, update=True)

    # Step 2: Backfill in batches (safe for large datasets)
    projects = frappe.get_all("Project", pluck="name")

    for i in range(0, len(projects), 100):
        batch = projects[i:i+100]
        for project_name in batch:
            total = frappe.db.count("Task", {"project": project_name})
            overdue = frappe.db.count("Task", {
                "project": project_name,
                "status": ["not in", ["Completed", "Cancelled"]],
                "exp_end_date": ["<", frappe.utils.today()],
            })
            frappe.db.set_value("Project", project_name, {
                "total_tasks": total,
                "overdue_tasks": overdue,
            }, update_modified=False)

        frappe.db.commit()
        print(f"Processed {min(i+100, len(projects))}/{len(projects)} projects")
```

### Data Migration with Rollback

```python
# patches/v2_0/migrate_status_values.py
import frappe

def execute():
    """Migrate status field values. Includes rollback tracking."""

    # Track what we changed for potential rollback
    changes_log = []

    old_to_new = {
        "Work In Progress": "In Progress",
        "Pending Review": "Under Review",
        "Closed": "Completed",
    }

    for old_status, new_status in old_to_new.items():
        affected = frappe.get_all("Project",
            filters={"status": old_status},
            pluck="name"
        )

        if affected:
            frappe.db.sql("""
                UPDATE "tabProject"
                SET status = %(new)s
                WHERE status = %(old)s
            """, {"old": old_status, "new": new_status})

            changes_log.append({
                "old": old_status,
                "new": new_status,
                "count": len(affected),
                "affected": affected[:10],  # Sample for logging
            })

    frappe.db.commit()

    # Log changes
    for change in changes_log:
        print(f"Migrated {change['count']} records: '{change['old']}' → '{change['new']}'")
```

## Backup & Recovery

### Automated Backup Script

```bash
#!/bin/bash
# scripts/backup-frappe.sh
# Run daily via cron: 0 2 * * * /home/deploy/scripts/backup-frappe.sh

set -euo pipefail

SITE="production.dris.com"
BENCH_DIR="/home/deploy/frappe-bench"
BACKUP_DIR="/backups/frappe"
RETENTION_DAYS=30
DATE=$(date +%Y%m%d_%H%M%S)

# Take backup
cd "$BENCH_DIR"
bench --site "$SITE" backup --with-files

# Move to organized backup directory
mkdir -p "$BACKUP_DIR/$DATE"
mv "$BENCH_DIR/sites/$SITE/private/backups/"* "$BACKUP_DIR/$DATE/"

# Verify backup integrity
DB_BACKUP=$(ls "$BACKUP_DIR/$DATE/"*database* 2>/dev/null | head -1)
if [ -z "$DB_BACKUP" ]; then
    echo "ERROR: No database backup found!"
    exit 1
fi

# Check backup size (should be > 1MB for any real site)
BACKUP_SIZE=$(stat -f%z "$DB_BACKUP" 2>/dev/null || stat -c%s "$DB_BACKUP")
if [ "$BACKUP_SIZE" -lt 1048576 ]; then
    echo "WARNING: Backup suspiciously small: $BACKUP_SIZE bytes"
fi

# Clean old backups
find "$BACKUP_DIR" -maxdepth 1 -type d -mtime +$RETENTION_DAYS -exec rm -rf {} +

echo "Backup completed: $BACKUP_DIR/$DATE ($(du -sh "$BACKUP_DIR/$DATE" | cut -f1))"
```

## PostgreSQL Tuning for Frappe

```ini
# postgresql.conf — tuned for Frappe workload (8GB RAM server)

# Memory
shared_buffers = 2GB              # 25% of RAM
effective_cache_size = 6GB        # 75% of RAM
work_mem = 64MB                   # Per-operation sort/hash memory
maintenance_work_mem = 512MB      # For VACUUM, CREATE INDEX

# Write Performance
wal_buffers = 64MB
checkpoint_completion_target = 0.9
max_wal_size = 2GB

# Query Planner
random_page_cost = 1.1            # SSD storage
effective_io_concurrency = 200    # SSD storage
default_statistics_target = 100

# Connections
max_connections = 100             # Match gunicorn workers + background

# Logging (enable for debugging, disable in stable production)
log_min_duration_statement = 500  # Log queries > 500ms
log_statement = 'none'            # Set to 'all' for debugging only
```

## Common Frappe + PostgreSQL Issues

| Issue | Symptom | Fix |
|-------|---------|-----|
| Missing indexes | Slow list views | Add composite index on filter fields |
| N+1 queries | Many small queries in loops | Use `frappe.get_all()` with fields |
| Table bloat | Growing disk, slow queries | Schedule `VACUUM ANALYZE` |
| Lock contention | Timeout on save | Reduce transaction scope, use `select_for_update` carefully |
| Large attachments | Slow backups, disk full | Move files to S3/object storage |
| Unoptimized reports | Reports timeout | Add materialized views or background aggregation |
