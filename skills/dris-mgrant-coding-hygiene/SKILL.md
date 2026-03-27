# mGrant / Frappe Coding Hygiene — Developer Practices

> Org-wide coding standards for all Dhwani RIS developers working on Frappe-based products (mGrant Donor, CSR, NuO, client apps).
> Activate this skill alongside `frappe-development` and `govt-csr-compliance` for complete context.

---

## 1. Architecture: 4-Layer App Stack

Every mGrant deployment follows a strict 4-layer architecture. **Never modify Layers 1-3 for client-specific work.**

```
Layer 4 — CLIENT APP (unique per client: frappe_pwc, frappe_bfs, frappe_tri)
Layer 3 — CSR APP ("csr") — 3 modules, 12 DocTypes
Layer 2 — MGRANT DONOR ("mgrant") — 15 modules, 192 DocTypes
Layer 1 — THEME + UTILITIES ("frappe_theme" + "sva_frappe")
```

**Rules:**
- All client-specific changes go in Layer 4 only
- Client apps may ADD restrictions but never REMOVE permissions defined in base apps
- If a change requires modifying Layer 2 or 3, it is a product change — raise a ticket, get PM approval, merge to core
- Layer 1 is shared infrastructure — never touch for any client work

---

## 2. Configuration Decision Tree

Before writing any code, determine the correct mechanism. Wrong choice = silent breakage or deployment conflicts.

### Decision: What mechanism should I use?

```
Is this a DISPLAY-ONLY rename (label, sidebar text)?
  YES → Translation (fixtures/translation.json)

Is this a STRUCTURAL field change (reqd, depends_on, options list, hidden)?
  YES → Property Setter (via setup script)

Is this a NEW field that doesn't exist in base DocType?
  YES → Custom Field (fixtures/custom_field.json, must use custom_ prefix)

Is this form-level conditional logic or validation?
  YES → Client Script (public/js/ + doctype_js hook)

Is this server-side business logic?
  YES → Python controller in client app (hooks.py doc_events)

Is this site-level configuration (System Settings, site_config)?
  YES → Setup script (config/site_setup.py, idempotent)

Is this a new DocType?
  YES → DocType JSON in client app (custom/ directory)
```

### The Critical Property Setter Rule

**NEVER rename Select option VALUES via Property Setter.**

Core mGrant code hardcodes option values in `depends_on` rules, client scripts (JS), and Python logic. A Property Setter rename changes the stored value, silently breaking all code that references the old value.

**Example of what goes wrong:**
- PS changed RFP `invitation_type` from "Open Call" to "Open RFP"
- Core code still referenced "Open Call" in `depends_on` rules
- Result: "Copy Invitation URL" button = null, form sections disappeared
- Fix: Reverted PS, used Translation instead

**Rule:** Use **Translation** for display-only renames — it changes what users SEE without changing the stored value that code references.

### One Source of Truth

Every configuration item has exactly ONE source:
- Fixture OR runtime creation — never both
- If it's in `translation.json`, don't also create it via a patch
- If it's a Property Setter in `site_setup.py`, don't also set it via bench console and forget

---

## 3. Fixture Discipline

### File Locations (Client App)

| What | Where | Format |
|------|-------|--------|
| Custom Fields | `fixtures/custom_field.json` | JSON array |
| Translations | `fixtures/translation.json` | JSON array |
| DocType overrides | `custom/*.json` | Frappe DocType JSON |
| Site configuration | `config/site_setup.py` | Python (idempotent) |
| Schema migrations | `patches/*.py` | Python patch |
| Client scripts | `public/js/*.js` | JavaScript |
| Server APIs | `apis/*.py` | Python with `@frappe.whitelist()` |

### Rules

1. **Fixtures are idempotent** — running them multiple times produces the same result
2. **Patches are for schema migrations ONLY** — never for site config, translations, or seed data
3. **Seed/master data via API** — never hardcode in fixtures. Use versioned scripts under `scripts/` or `apis/`
4. **Custom fields MUST use `custom_` prefix** — Frappe convention
5. **Import order matters:** Custom Fields before DocType changes, Translations last
6. **After fixture changes:** `bench --site {site} migrate` + `bench --site {site} clear-cache`

### Setup Script Pattern (Idempotent)

```python
def execute():
    """Idempotent setup for {client} instance.
    Run: bench --site {site} execute {app}.config.site_setup.execute
    """
    set_property_setters()
    insert_translations()
    configure_theme()
    setup_master_data()

def set_property_setters():
    """Each PS is documented with ticket reference."""
    ps_list = [
        # DRI-15: Hide internal field from NGO portal
        {"doctype": "Grant", "field": "internal_notes", "property": "hidden", "value": "1"},
    ]
    for ps in ps_list:
        # Idempotent: check if exists before creating
        if not frappe.db.exists("Property Setter", {
            "doc_type": ps["doctype"],
            "field_name": ps["field"],
            "property": ps["property"]
        }):
            frappe.make_property_setter(ps["doctype"], ps["field"], ps["property"], ps["value"])
```

---

## 4. Permission Architecture

Frappe permissions = row-level security. Every custom DocType MUST have explicit permissions.

### Role Hierarchy (mGrant Standard)

| Layer | Roles | Access Pattern |
|-------|-------|----------------|
| System | Administrator, System Manager | Full access. Never assigned to client users. |
| Donor-side | Grant Manager, Program Officer, Finance Officer | Filtered by donor org via User Permissions. |
| NGO-side | NGO Admin, NGO User | Filtered by linked NGO. Must never see other NGOs' data. |
| CSR-specific | CSR Admin, CSR Finance | MCA compliance permissions. Budget freeze controls. |
| Portal | Website User (NGO Portal) | Portal-only. No Desk access. Strict DocType subset. |

### Permission Rules

1. **Every custom DocType needs a `permissions` array in its JSON** — no exceptions
2. **No wildcard permissions** — never grant access to `All` or `Guest` (unless genuinely public, which is rare in mGrant)
3. **Use `if_owner`** where creators should only access their own records
4. **Use User Permissions** for row-level isolation (NGO sees only their grants)
5. **Test each role** after setup to verify isolation works

### Permission Anti-Patterns (Reject on Review)

| Anti-Pattern | Why It's Dangerous |
|---|---|
| `ignore_permissions = True` outside setup scripts | Bypasses all access control |
| `frappe.get_all()` without filters | Returns all records regardless of permissions |
| Whitelisted method accepting `user` param from client | Client can impersonate any user |
| DocType with `track_changes = 0` on financial docs | No audit trail for compliance |
| API returning unfiltered querysets | Data leak across organizations |

### Portal Security

- NGO Portal pages must enforce `get_list_query` and `get_query_condition` hooks
- Portal form submissions must validate write permission
- File uploads via portal: validate type, size, and DocType linkage
- Never copy production data to UAT without anonymization

---

## 5. Security Standards

### Secrets — Absolute Rules

**Never commit:**
- `db_password`, `secret_key`, `encryption_key`
- API keys, SMTP credentials
- Any value from `site_config.json`
- URLs to UAT/staging/production environments (`*.mgrant.in`)

**Secrets live in:** `site_config.json` (per-site) or environment variables. **Never in code.**

**AI-generated code scan:** LLMs hallucinate credentials. Every AI-generated file must be scanned for `sk_`, `pk_`, `Bearer ey`, `password=`, `secret=`, base64 blobs BEFORE commit.

### SQL — Parameterised Queries ONLY

```python
# CORRECT — parameterised
frappe.db.sql("SELECT name FROM `tabGrant` WHERE ngo = %s", (ngo_name,))

# REJECT — SQL injection risk
frappe.db.sql(f"SELECT name FROM `tabGrant` WHERE ngo = '{ngo_name}'")
```

- No string concatenation or f-strings in SQL
- No direct DDL (`ALTER TABLE`) — schema changes via DocType JSON + `bench migrate`
- Financial documents must have `track_changes = 1`

### API Security

- `@frappe.whitelist()` default = authenticated (correct)
- `@frappe.whitelist(allow_guest=True)` must be justified in PR
- Every whitelisted method MUST validate inputs with `frappe.throw()` for invalid states
- Never accept `user` or `role` as client parameter — server reads `frappe.session.user`
- Never trust `frappe.session.user` on client side — always verify server-side
- Rate limit guest-accessible endpoints via `frappe.rate_limiter` or Nginx

### Client-Side Trust — Anti-Patterns

- NEVER trust `frappe.session.user` on client — read server-side
- NEVER pass user as parameter from client to server
- Client scripts = validation and conditional visibility ONLY — never expose secrets

### Infrastructure

- `site_config.json` permissions: `600` (owner read/write only)
- Bench processes run under non-root dedicated user
- HTTPS mandatory on all `*.mgrant.in` domains
- Rate limit `/api/method/` at Nginx (30 req/s per IP)
- Block direct access to `/api/method/frappe.client.*` from external IPs unless explicit

---

## 6. Frappe Patterns & Lifecycle

### Workflow State Handling

**Never hardcode workflow state names.** States may vary by client or version.

```python
# CORRECT — use helper
from frappe_mgrant.utils import get_state_closure_by_type
states = get_state_closure_by_type("Grant", "approved")

# REJECT — breaks when client renames states
if doc.workflow_state == "Approved":
```

### DocType Lifecycle Hooks

| Hook | When | Use For |
|------|------|---------|
| `validate` | Before save | Business logic validation |
| `before_save` | Before save | Data transformation |
| `after_insert` | After first save | Post-insert side effects |
| `on_submit` | On submission | Ledger entries, state transitions |
| `on_cancel` | On cancellation | Reverse submissions, revert state |

- Financial docs (`on_submit`/`on_cancel`) MUST log action with user and timestamp
- All hooks registered in `hooks.py` and documented in CODE_JOURNAL.md

### Client Scripts (Form-Level)

- Use `frm.trigger()`, `frm.set_value()`, `frm.toggle_display()`, `frm.call()` for form interaction
- Validation and conditional visibility only — never expose secret data
- Avoid direct DOM manipulation — use Frappe's form API

### Caching

- After role changes via API: `bench clear-cache`
- After DB changes (PS, Translation, Theme): `bench --site {site} clear-cache`
- After code changes (PR merged): `migrate` + `build --app {app}` + `restart`

### Known Gotchas

| What | Problem | Correct Approach |
|------|---------|-----------------|
| `frappe.client.set_value` for workspace visibility | Unreliable, silent failures | Fetch full doc, modify, save |
| `frappe.client.delete` for Custom Fields | Doesn't work reliably | Use REST resource endpoint |
| Query Report for comparison views | Wrong tool — no interactivity | Use Form tab + API + `doctype_js` |
| `depends_on` referencing renamed option values | Silently breaks when PS renames values | Always check option dependencies before PS |

---

## 7. Common Mistakes — Learn From These

| # | Mistake | Lesson |
|---|---------|--------|
| 1 | PS renamed "Open Call" to "Open RFP" — broke Copy Invitation URL | Use Translation for display renames. Never rename option VALUES via PS. |
| 2 | Translation in both fixture AND patch — duplicates | One source of truth. Never both. |
| 3 | Patches used for site config (not schema migrations) | Patches = schema migrations ONLY. Site config in setup scripts. |
| 4 | Partner Comparison built as Query Report | Use Form tab + API + `doctype_js` (HMIF pattern). |
| 5 | Property Setter added/reverted/re-added (3 commits of churn) | Decide mechanism BEFORE writing code. |
| 6 | T&C gate: 6 failed branches | Understand Frappe request lifecycle first. `/apps` -> API -> `/app`. |
| 7 | Playwright tests with wrong field names — 50 failures | Always verify selectors against live API schema before writing tests. |
| 8 | Vibe-coded without permission/security review | Every AI-generated code MUST be checked for: `ignore_permissions`, raw SQL, guest endpoints, client-side trust. |

---

## 8. Change Documentation (CODE_JOURNAL.md)

Every repository with client-facing code MUST maintain a `CODE_JOURNAL.md`. Update after every PR merge.

### Format

```markdown
## Branch: {branch-name}

**Changes vs `{base_branch}`:**
| File | What changed |

### Fixes in this branch
| Ticket | Fix | Method |

### Deployment
bench --site {site} migrate
bench --site {site} execute {app}.config.site_setup.execute
bench --site {site} clear-cache

### Security Checklist
- [ ] No secrets in code
- [ ] Parameterised SQL only
- [ ] No `ignore_permissions` (or justified)
- [ ] No `allow_guest` (or justified)
- [ ] Input validation on all whitelisted methods
- [ ] No client-side `session.user` trust
- [ ] Hooks documented in hooks.py
```

---

## 9. Deployment & Verification

### Pre-Merge Checklist

- [ ] DocType permissions defined and tested for each role
- [ ] Workflow states have correct role-based transitions
- [ ] User Permissions tested (row-level isolation works)
- [ ] Backup verified (for DB changes)
- [ ] Rollback plan documented
- [ ] CODE_JOURNAL.md updated with ticket numbers
- [ ] AI-code flag in PR description if any part was AI-generated

### Deployment Flow

```bash
# 1. Pull
cd apps/{app} && git pull origin {branch} && cd ../..

# 2. Migrate
bench --site {site} migrate

# 3. Build (if JS changed)
bench --site {site} build --app {app}

# 4. Setup script (if config changed)
bench --site {site} execute {app}.config.site_setup.execute

# 5. Clear cache
bench --site {site} clear-cache

# 6. Restart
bench restart

# 7. Smoke test: login, portal access, grant lifecycle action
```

### Rollback Plan (Always Required)

- Property Setter revert: delete via bench console — names are deterministic: `{DocType}-{field}-{property}`
- Custom Field revert: use REST resource endpoint (not `frappe.client.delete`)
- Data patch revert: SQL rollback script or restore from backup
- Document all rollback steps in CODE_JOURNAL before deploying

---

## 10. Quick Reference — Where Things Live

| Question | Answer |
|----------|--------|
| Secrets | `site_config.json` or env vars. Never in code. |
| Permissions | DocType JSON `permissions` array + User Permissions for row-level |
| Rate limiting | Nginx config + `frappe.rate_limiter` |
| File security | Frappe File DocType permissions + `private/files` |
| Audit trail | `tabActivity Log`, `tabAccess Log`, `tabVersion` (track_changes) |
| Custom fields | `fixtures/custom_field.json`. Prefix: `custom_` |
| Translations | `fixtures/translation.json`. Display only, not stored values. |
| Property Setters | `config/site_setup.py` (idempotent). Revert via bench console. |
| Master data | API inserts in versioned scripts. Not in fixtures. |
| Workflows | DocType JSON `workflow_name` + separate Workflow DocType |
| Scheduled jobs | `hooks.py` `scheduler_events` |
| REST endpoints | `@frappe.whitelist()` methods in controller or app module |
| Client scripts | `public/js/` + `doctype_js` hook in `hooks.py` |

---

## 11. Environment Segregation

| Environment | Data | Access | Rule |
|-------------|------|--------|------|
| Production (`*.mgrant.in`) | Real client data | Restricted. No dev access without approval. | Never run bench commands without PM approval. |
| UAT (`uat.*.mgrant.in`) | Anonymised/demo data | Dev + QA + PM | Always test here before staging. |
| Staging (`stg.*.mgrant.in`) | Test data only | Dev team | Free to break. |

**Rule:** UAT before staging. Never skip UAT for production deployment.

---

## Activation

Load this skill when:
- Starting any mGrant or Frappe client app development
- Reviewing PRs for coding standards compliance
- Setting up a new client deployment
- Onboarding a new developer to the mGrant codebase

Pair with: `frappe-development`, `govt-csr-compliance`, `qa-testing`, `postgres-frappe-patterns`
