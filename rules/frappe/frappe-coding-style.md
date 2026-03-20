# Frappe Coding Style Rules

## Python (Server-Side)

- Use `frappe.qb` (Query Builder) for all database queries. Raw SQL only for complex CTEs or window functions.
- Never use string formatting (`f""`, `.format()`, `%`) in SQL queries. Always use parameterized queries (`%s`).
- Every `@frappe.whitelist()` method must include explicit permission checks (`frappe.has_permission()` or `doc.check_permission()`).
- Never use `ignore_permissions=True` in production code except in system-level scheduled tasks.
- Use `frappe.utils.cstr()`, `cint()`, `flt()` to sanitize user inputs.
- Parse JSON string arguments explicitly with `json.loads()` — Frappe sends args as strings.
- Place whitelisted API methods in `api.py`, not scattered across controller files.
- Use `frappe.db.get_value()` when you need one field, not `frappe.get_doc()`.
- Background jobs (`frappe.enqueue()`) must call `frappe.db.commit()` at the end.
- Use `create_batch()` for processing large datasets with periodic commits.

## JavaScript (Client-Side)

- Use `frappe.call()` for all API calls — never raw `fetch()` or `$.ajax()`.
- Always include `error` callback in `frappe.call()`.
- Use `frm.set_query()` to filter Link fields based on form context.
- Use `frm.set_value()` in fetch callbacks, never direct property assignment.
- Follow Frappe event naming: `fieldname(frm)` for field events, `refresh(frm)` for form events.
- Use `__(text)` for all user-facing strings (translation support).

## DocType Design

- Enable `track_changes = 1` on all DocTypes that store business data.
- Set appropriate naming rules (not always autoname — consider field-based for readability).
- Add indexes on fields used in list view filters and common WHERE clauses.
- Use submittable DocTypes for documents that need approval workflows.
- Child tables must have a clear parent link and appropriate field ordering.

## Security

- Never use `allow_guest=True` on whitelisted methods without security review.
- Use Frappe's Role Permission Manager — never hardcode role checks.
- Encrypt PII (Aadhaar, phone) at rest. Mask in API responses for non-privileged users.
- Log all data export operations for audit compliance.
- Government projects: never delete data — use status flags for soft deletion.
