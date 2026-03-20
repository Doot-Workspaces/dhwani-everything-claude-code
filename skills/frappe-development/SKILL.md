---
name: frappe-development
description: Frappe framework development patterns, DocType design, API usage, dashboard building, hooks, whitelisted methods, and ERPNext customization for building government and CSR solutions.
origin: DRIS
---

# Frappe Development Patterns

## When to Activate

- Building or modifying Frappe DocTypes
- Creating dashboards using Frappe API
- Writing server-side scripts (Python) or client-side scripts (JS)
- Designing whitelisted API methods
- Working with Frappe hooks, scheduled tasks, or background jobs
- Customizing ERPNext modules
- Building custom Frappe apps for government/CSR clients

## Core Architecture

### Frappe App Structure

```
my_app/
├── my_app/
│   ├── __init__.py
│   ├── hooks.py                 # App hooks and overrides
│   ├── patches/                 # Data migration patches
│   │   └── v1_0/
│   │       └── fix_field.py
│   ├── my_module/
│   │   ├── __init__.py
│   │   ├── doctype/
│   │   │   └── my_doctype/
│   │   │       ├── my_doctype.json    # DocType definition
│   │   │       ├── my_doctype.py      # Server controller
│   │   │       ├── my_doctype.js      # Client controller
│   │   │       └── test_my_doctype.py # Tests
│   │   ├── api.py               # Whitelisted API methods
│   │   ├── report/              # Script reports
│   │   └── page/                # Custom pages
│   ├── public/                  # Static assets
│   ├── templates/               # Jinja templates
│   └── www/                     # Web pages
├── setup.py
└── requirements.txt
```

## DocType Design Patterns

### Standard DocType Controller

```python
# my_doctype.py
import frappe
from frappe.model.document import Document

class MyDoctype(Document):
    def validate(self):
        """Runs before save — use for validation logic."""
        self.validate_required_fields()
        self.calculate_totals()

    def before_save(self):
        """Runs after validate, before writing to DB."""
        self.set_status()

    def on_submit(self):
        """Runs when document is submitted (for submittable DocTypes)."""
        self.create_gl_entries()
        self.notify_stakeholders()

    def on_cancel(self):
        """Runs when submitted document is cancelled."""
        self.reverse_gl_entries()

    def before_insert(self):
        """Runs before first save only."""
        self.set_defaults_from_project()

    def validate_required_fields(self):
        if not self.project:
            frappe.throw("Project is mandatory")

    def calculate_totals(self):
        self.total_amount = sum(d.amount for d in self.items)
```

### Client-Side Controller

```javascript
// my_doctype.js
frappe.ui.form.on('My Doctype', {
    refresh(frm) {
        // Add custom buttons based on workflow state
        if (frm.doc.docstatus === 1) {
            frm.add_custom_button(__('Create Invoice'), () => {
                frappe.call({
                    method: 'my_app.my_module.api.create_invoice',
                    args: { source_name: frm.doc.name },
                    callback(r) {
                        if (r.message) {
                            frappe.set_route('Form', 'Sales Invoice', r.message);
                        }
                    }
                });
            }, __('Actions'));
        }

        // Set query filters for Link fields
        frm.set_query('employee', () => ({
            filters: { department: frm.doc.department }
        }));
    },

    project(frm) {
        // Fetch values when project changes
        if (frm.doc.project) {
            frappe.db.get_value('Project', frm.doc.project,
                ['project_name', 'client', 'expected_end_date'],
                (r) => {
                    frm.set_value('project_name', r.project_name);
                    frm.set_value('client', r.client);
                }
            );
        }
    }
});

// Child table events
frappe.ui.form.on('My Doctype Item', {
    amount(frm, cdt, cdn) {
        // Recalculate totals when child row changes
        let total = 0;
        frm.doc.items.forEach(d => { total += d.amount || 0; });
        frm.set_value('total_amount', total);
    }
});
```

## Whitelisted API Methods

### Secure API Pattern

```python
# api.py
import frappe
from frappe import _

@frappe.whitelist()
def get_project_dashboard(project):
    """Fetch dashboard data for a project.

    Whitelisted methods are accessible via /api/method/<path>.
    Always validate permissions explicitly.
    """
    # Permission check — NEVER skip this
    if not frappe.has_permission("Project", "read", project):
        frappe.throw(_("Not permitted"), frappe.PermissionError)

    project_doc = frappe.get_doc("Project", project)

    return {
        "status": project_doc.status,
        "progress": project_doc.percent_complete,
        "tasks": frappe.get_all("Task", filters={"project": project}, fields=[
            "name", "subject", "status", "priority", "assigned_to"
        ]),
        "milestones": frappe.get_all("Project Milestone", filters={"parent": project}, fields=[
            "milestone", "milestone_date", "status"
        ]),
        "budget": {
            "allocated": project_doc.estimated_costing,
            "spent": project_doc.total_costing_amount,
            "remaining": project_doc.estimated_costing - project_doc.total_costing_amount
        }
    }


@frappe.whitelist()
def bulk_update_status(doctype, names, status):
    """Bulk update status for multiple documents.

    Args are always received as strings from the client.
    Always parse JSON args explicitly.
    """
    import json
    if isinstance(names, str):
        names = json.loads(names)

    if not frappe.has_permission(doctype, "write"):
        frappe.throw(_("Not permitted"), frappe.PermissionError)

    updated = []
    for name in names:
        doc = frappe.get_doc(doctype, name)
        doc.status = status
        doc.save(ignore_permissions=False)  # Never use ignore_permissions=True
        updated.append(name)

    frappe.db.commit()
    return {"updated": updated, "count": len(updated)}


@frappe.whitelist(allow_guest=False)
def create_invoice(source_name):
    """Create Sales Invoice from custom document.

    allow_guest=False is default but be explicit for clarity.
    """
    source = frappe.get_doc("My Doctype", source_name)
    source.check_permission("submit")

    invoice = frappe.new_doc("Sales Invoice")
    invoice.customer = source.client
    invoice.project = source.project
    for item in source.items:
        invoice.append("items", {
            "item_code": item.item_code,
            "qty": item.qty,
            "rate": item.rate
        })
    invoice.insert()
    return invoice.name
```

### API Call from Frontend

```javascript
// Proper frappe.call pattern
frappe.call({
    method: 'my_app.my_module.api.get_project_dashboard',
    args: { project: 'PROJ-001' },
    freeze: true,
    freeze_message: __('Loading dashboard...'),
    callback(r) {
        if (r.message) {
            render_dashboard(r.message);
        }
    },
    error(r) {
        frappe.msgprint(__('Failed to load dashboard'));
    }
});
```

## Dashboard Building with Frappe

### Custom Page Dashboard

```python
# page/project_dashboard/project_dashboard.py
import frappe

@frappe.whitelist()
def get_dashboard_data(filters=None):
    """Aggregate data for dashboard charts."""
    import json
    if isinstance(filters, str):
        filters = json.loads(filters)

    # Use frappe.qb for safe query building (NOT string concatenation)
    Project = frappe.qb.DocType("Project")
    Task = frappe.qb.DocType("Task")

    # Project status distribution
    status_data = (
        frappe.qb.from_(Project)
        .select(Project.status, frappe.qb.functions.Count("*").as_("count"))
        .where(Project.company == filters.get("company"))
        .groupby(Project.status)
        .run(as_dict=True)
    )

    # Task completion trend (last 30 days)
    task_trend = (
        frappe.qb.from_(Task)
        .select(
            frappe.qb.functions.Date(Task.completed_on).as_("date"),
            frappe.qb.functions.Count("*").as_("count")
        )
        .where(Task.status == "Completed")
        .where(Task.completed_on >= frappe.utils.add_days(frappe.utils.today(), -30))
        .groupby(frappe.qb.functions.Date(Task.completed_on))
        .run(as_dict=True)
    )

    return {
        "status_distribution": status_data,
        "task_completion_trend": task_trend
    }
```

### Report Builder Pattern

```python
# report/project_summary/project_summary.py
import frappe

def execute(filters=None):
    columns = get_columns()
    data = get_data(filters)
    chart = get_chart(data)
    return columns, data, None, chart

def get_columns():
    return [
        {"label": "Project", "fieldname": "project", "fieldtype": "Link", "options": "Project", "width": 200},
        {"label": "Client", "fieldname": "client", "fieldtype": "Link", "options": "Customer", "width": 150},
        {"label": "Status", "fieldname": "status", "fieldtype": "Data", "width": 100},
        {"label": "Progress %", "fieldname": "progress", "fieldtype": "Percent", "width": 100},
        {"label": "Budget", "fieldname": "budget", "fieldtype": "Currency", "width": 120},
        {"label": "Spent", "fieldname": "spent", "fieldtype": "Currency", "width": 120},
    ]

def get_data(filters):
    conditions = ""
    if filters.get("status"):
        conditions += " AND p.status = %(status)s"
    if filters.get("department"):
        conditions += " AND p.department = %(department)s"

    # Use parameterized queries — NEVER string format user input
    return frappe.db.sql("""
        SELECT
            p.name as project,
            p.customer as client,
            p.status,
            p.percent_complete as progress,
            p.estimated_costing as budget,
            p.total_costing_amount as spent
        FROM `tabProject` p
        WHERE p.company = %(company)s {conditions}
        ORDER BY p.modified DESC
    """.format(conditions=conditions), filters, as_dict=True)

def get_chart(data):
    return {
        "data": {
            "labels": [d.project for d in data[:10]],
            "datasets": [{"values": [d.progress for d in data[:10]]}]
        },
        "type": "bar"
    }
```

## Hooks Best Practices

### hooks.py Patterns

```python
# hooks.py

app_name = "my_app"
app_title = "My App"

# Document events — use sparingly, prefer controller methods
doc_events = {
    "Task": {
        "on_update": "my_app.handlers.task_on_update",
        "after_insert": "my_app.handlers.task_after_insert",
    },
    "Project": {
        "validate": "my_app.handlers.project_validate",
    },
    # Wildcard — runs for ALL doctypes (use with extreme caution)
    "*": {
        "on_update": "my_app.handlers.log_audit_trail",
    }
}

# Scheduled tasks
scheduler_events = {
    "daily": [
        "my_app.tasks.send_daily_digest",
        "my_app.tasks.check_project_deadlines",
    ],
    "hourly": [
        "my_app.tasks.sync_external_data",
    ],
    "cron": {
        "0 9 * * 1": "my_app.tasks.send_weekly_report",  # Monday 9 AM
    }
}

# Override whitelisted methods
override_whitelisted_methods = {
    "frappe.client.get_count": "my_app.overrides.custom_get_count"
}

# Jinja template methods (available in print formats)
jinja = {
    "methods": [
        "my_app.utils.format_indian_currency",
        "my_app.utils.get_state_code",
    ]
}

# Fixtures — export to version control
fixtures = [
    {"dt": "Custom Field", "filters": [["module", "=", "My Module"]]},
    {"dt": "Property Setter", "filters": [["module", "=", "My Module"]]},
    {"dt": "Workflow", "filters": [["module", "=", "My Module"]]},
]
```

## Database Patterns (PostgreSQL + Frappe)

### Safe Query Patterns

```python
# GOOD: Use frappe.qb (Query Builder) — SQL injection safe
Project = frappe.qb.DocType("Project")
results = (
    frappe.qb.from_(Project)
    .select(Project.name, Project.status)
    .where(Project.company == company_name)
    .where(Project.status.isin(["Open", "In Progress"]))
    .limit(50)
    .run(as_dict=True)
)

# GOOD: Parameterized SQL when qb isn't enough
results = frappe.db.sql("""
    SELECT name, status FROM `tabProject`
    WHERE company = %s AND status IN %s
""", (company_name, ("Open", "In Progress")), as_dict=True)

# BAD: String interpolation — SQL injection vulnerability
# results = frappe.db.sql(f"SELECT * FROM `tabProject` WHERE company = '{company_name}'")
```

### Bulk Operations

```python
# Efficient bulk insert
values = [(name, status, frappe.utils.now()) for name, status in items]
frappe.db.bulk_insert("My Doctype", fields=["name", "status", "creation"],
                       values=values, ignore_duplicates=True)

# Batch processing for large datasets
for batch in frappe.utils.create_batch(all_records, batch_size=500):
    for record in batch:
        process(record)
    frappe.db.commit()
```

## Security Checklist for Government Projects

- Always use `frappe.has_permission()` before data access
- Never use `ignore_permissions=True` in production code
- Use parameterized queries or `frappe.qb` — never string formatting
- Set DocType permissions via Role Permission Manager
- Use `frappe.only_for("System Manager")` for admin-only methods
- Implement row-level security via User Permission when needed
- Always validate file uploads (type, size) before saving
- Use `frappe.utils.cstr()` and `frappe.utils.cint()` to sanitize inputs
- Enable audit trail via `track_changes = 1` in DocType
- Test with non-admin users to verify permission enforcement

## Bench Commands Reference

```bash
# Development
bench new-app my_app              # Create new app
bench get-app <git-url>           # Install from git
bench --site mysite.local install-app my_app
bench --site mysite.local migrate  # Run patches and sync
bench build                        # Build JS/CSS assets
bench clear-cache                  # Clear Redis cache

# Database
bench --site mysite.local backup                    # Full backup
bench --site mysite.local restore <backup-file>     # Restore
bench --site mysite.local console                   # Python console
bench --site mysite.local mariadb                   # DB shell (or postgres)

# Testing
bench --site test_site run-tests --app my_app
bench --site test_site run-tests --doctype "My Doctype"
bench --site test_site run-tests --module my_app.my_module.test_utils

# Production
bench setup production <user>
bench setup supervisor
bench setup nginx
bench restart
```

## Common Anti-Patterns to Avoid

1. **Using `frappe.db.sql` with string formatting** — Always parameterize
2. **`ignore_permissions=True` everywhere** — Only in system-level background jobs
3. **Heavy logic in hooks.py doc_events** — Move to controller methods
4. **Not committing in background jobs** — Use `frappe.db.commit()` in enqueue'd functions
5. **Fetching full documents when you need one field** — Use `frappe.db.get_value()`
6. **Not using `frappe.qb`** — Preferred over raw SQL for type safety
7. **Wildcard doc_events** — Runs on every save, massive performance hit
8. **Missing index on frequently filtered fields** — Add via customize form or patches
