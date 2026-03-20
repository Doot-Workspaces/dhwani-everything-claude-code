# Frappe Security Rules

## API Security

- Every `@frappe.whitelist()` must validate permissions before any data access or mutation.
- `allow_guest=True` is banned unless explicitly approved in security review. Default is `allow_guest=False`.
- Rate limit public-facing API endpoints. Configure via `site_config.json` rate_limit settings.
- Validate and sanitize all incoming arguments — they arrive as strings from the client.
- Return only necessary fields in API responses. Never return full documents when a subset suffices.

## SQL Injection Prevention

- Use `frappe.qb` (Query Builder) as the default for all queries.
- When raw SQL is unavoidable, use parameterized queries with `%s` placeholders.
- Never use f-strings, `.format()`, or `%` operator to build SQL.
- `frappe.db.sql()` with dict params `%(key)s` is acceptable.

## Authentication & Authorization

- Enable two-factor authentication for admin and manager roles.
- Use Frappe's built-in session management — never implement custom auth.
- Implement User Permission for row-level security (district, department, project-based access).
- Session timeout: configure `session_expiry` in site_config.json (recommended: 6 hours for govt).
- Log all failed login attempts and lock accounts after 5 failures.

## Data Protection

- Encrypt PII fields (Aadhaar, PAN, phone numbers) at rest using Fernet encryption.
- Mask PII in API responses based on user role — only authorized roles see full data.
- Never log PII in error logs or print statements.
- File uploads: validate type (whitelist extensions), size (max 10MB default), and scan for malware.
- Disable data export for roles that don't need it.

## Government Project Requirements

- All data mutations must be audit-logged (use `track_changes` + custom Audit Log DocType).
- No hard deletes — use Cancel/Amend workflow or status-based soft deletion.
- Data retention: archive per policy, never purge without explicit authorization.
- Implement IP whitelisting for admin access in production.
- Regular security patches: keep Frappe and dependencies updated.

## Production Hardening

- `disable_website_cache = 0` (enable caching)
- `developer_mode = 0` (disable in production)
- `allow_cors` restricted to known domains only
- HTTPS enforced with valid SSL certificates
- Regular backup verification (restore test monthly)
