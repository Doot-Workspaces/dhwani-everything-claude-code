---
name: govt-csr-compliance
description: Compliance patterns for building government and CSR solutions including audit trails, data handling rules, accessibility requirements, and security standards for Indian government agencies.
origin: DRIS
---

# Government & CSR Compliance Patterns

## When to Activate

- Building applications for government agencies
- Implementing CSR project management systems
- Handling sensitive citizen/beneficiary data
- Designing audit trail systems
- Implementing role-based access for government workflows
- Preparing for security audits and compliance reviews

## Audit Trail Requirements

### Complete Audit Logging

```python
# Every data change must be traceable to a user and timestamp.
# Frappe's Version DocType handles this — ensure it's enabled.

# In DocType JSON: "track_changes": 1

# For custom audit beyond Frappe's built-in:
import frappe

def log_audit_event(action, doctype, docname, details=None):
    """Log an audit event for compliance tracking.

    Use for actions beyond CRUD: exports, print, bulk operations.
    """
    frappe.get_doc({
        "doctype": "Audit Log",  # Custom DocType
        "action": action,
        "reference_doctype": doctype,
        "reference_name": docname,
        "user": frappe.session.user,
        "ip_address": frappe.local.request_ip,
        "timestamp": frappe.utils.now(),
        "details": details or "",
    }).insert(ignore_permissions=True)
    frappe.db.commit()

# Usage in sensitive operations
@frappe.whitelist()
def export_beneficiary_data(project):
    frappe.has_permission("Beneficiary", "export", throw=True)

    data = frappe.get_all("Beneficiary", filters={"project": project},
                          fields=["name", "beneficiary_name", "location", "status"])

    # Log the export action
    log_audit_event("data_export", "Beneficiary", project,
                    f"Exported {len(data)} records by {frappe.session.user}")

    return data
```

### Data Retention Policy

```python
# Implement data retention for compliance
# Government data often has specific retention periods

def apply_retention_policy():
    """Archive or anonymize data beyond retention period.

    NEVER delete government data — archive or anonymize instead.
    """
    retention_rules = {
        "Communication": 365 * 3,     # 3 years
        "Activity Log": 365 * 2,      # 2 years
        "Error Log": 90,              # 90 days
    }

    for doctype, days in retention_rules.items():
        cutoff = frappe.utils.add_days(frappe.utils.today(), -days)

        # Archive, don't delete
        old_records = frappe.get_all(doctype,
            filters={"creation": ["<", cutoff]},
            pluck="name", limit=1000)

        for name in old_records:
            doc = frappe.get_doc(doctype, name)
            # Move to archive table
            archive_doc = frappe.get_doc({
                "doctype": f"{doctype} Archive",
                "original_name": name,
                "data": doc.as_json(),
                "archived_on": frappe.utils.today(),
            })
            archive_doc.insert(ignore_permissions=True)
            # Only then remove from active table
            frappe.delete_doc(doctype, name, ignore_permissions=True, force=True)

        if old_records:
            frappe.db.commit()
```

## Data Protection

### PII Handling

```python
# Mask sensitive fields in API responses and logs

def mask_aadhaar(aadhaar_number):
    """Mask Aadhaar number — show only last 4 digits."""
    if not aadhaar_number or len(aadhaar_number) < 4:
        return "****"
    return "XXXX-XXXX-" + aadhaar_number[-4:]

def mask_phone(phone):
    """Mask phone number — show only last 4 digits."""
    if not phone or len(phone) < 4:
        return "****"
    return "XXXXX-" + phone[-4:]

# In DocType controller:
class Beneficiary(Document):
    def as_dict(self, *args, **kwargs):
        """Override to mask PII in API responses."""
        data = super().as_dict(*args, **kwargs)

        # Mask based on user role
        if not frappe.has_permission("Beneficiary", "export"):
            if data.get("aadhaar_number"):
                data["aadhaar_number"] = mask_aadhaar(data["aadhaar_number"])
            if data.get("phone"):
                data["phone"] = mask_phone(data["phone"])

        return data
```

### Encryption at Rest

```python
# Encrypt sensitive fields before storage
from cryptography.fernet import Fernet
import frappe

def get_encryption_key():
    """Get or create encryption key from site config."""
    key = frappe.conf.get("encryption_key")
    if not key:
        frappe.throw("encryption_key not configured in site_config.json")
    return key.encode()

def encrypt_field(value):
    """Encrypt a field value."""
    if not value:
        return value
    f = Fernet(get_encryption_key())
    return f.encrypt(value.encode()).decode()

def decrypt_field(encrypted_value):
    """Decrypt a field value."""
    if not encrypted_value:
        return encrypted_value
    f = Fernet(get_encryption_key())
    return f.decrypt(encrypted_value.encode()).decode()

# Usage in DocType:
class Beneficiary(Document):
    def before_save(self):
        if self.aadhaar_number and not self.aadhaar_number.startswith("gAAAA"):
            self.aadhaar_number = encrypt_field(self.aadhaar_number)

    def after_load(self):
        if self.aadhaar_number and self.aadhaar_number.startswith("gAAAA"):
            if frappe.has_permission("Beneficiary", "export"):
                self.aadhaar_number = decrypt_field(self.aadhaar_number)
            else:
                self.aadhaar_number = "XXXX-XXXX-****"
```

## Role-Based Access for Government Workflows

### Permission Matrix Template

| Role | DocType | Read | Write | Create | Submit | Cancel | Export |
|------|---------|------|-------|--------|--------|--------|--------|
| District Officer | Beneficiary | Own District | Own District | Yes | Yes | No | No |
| State Admin | Beneficiary | All | All | Yes | Yes | Yes | Yes |
| Block Coordinator | Beneficiary | Own Block | Own Block | Yes | No | No | No |
| Auditor | Beneficiary | All | No | No | No | No | Read-only |
| Data Entry | Beneficiary | Own | Own | Yes | No | No | No |

### Implementing Hierarchical Access

```python
# User Permission setup for district-level access
def setup_district_permissions(user, district):
    """Set up user permissions for district-level access."""
    # Clear existing district permissions
    frappe.db.delete("User Permission", {
        "user": user,
        "allow": "District",
    })

    # Add new permission
    frappe.get_doc({
        "doctype": "User Permission",
        "user": user,
        "allow": "District",
        "for_value": district,
        "apply_to_all_doctypes": 1,
    }).insert(ignore_permissions=True)
```

## Accessibility Requirements

### WCAG 2.1 AA Compliance Checklist

For government-facing web applications:

```html
<!-- Semantic HTML structure -->
<main role="main" aria-label="Project Dashboard">
  <h1>Project Summary</h1>

  <!-- Data tables must be accessible -->
  <table role="grid" aria-label="Beneficiary List">
    <caption>List of registered beneficiaries</caption>
    <thead>
      <tr>
        <th scope="col">Name</th>
        <th scope="col">District</th>
        <th scope="col">Status</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td>Name value</td>
        <td>District value</td>
        <td><span class="status-badge" role="status">Active</span></td>
      </tr>
    </tbody>
  </table>

  <!-- Forms must have labels -->
  <form aria-label="Search beneficiaries">
    <label for="search-name">Search by Name</label>
    <input id="search-name" type="text" aria-required="true" />

    <label for="district-filter">Filter by District</label>
    <select id="district-filter" aria-label="Select district">
      <option value="">All Districts</option>
    </select>
  </form>
</main>
```

### Localization for Indian Languages

```python
# Support for Hindi and regional languages in Frappe
# Add translations in my_app/translations/hi.csv

# hi.csv format:
# source_text,translated_text,context
# Project,परियोजना,DocType
# Beneficiary,लाभार्थी,DocType
# Status,स्थिति,Field Label
# Completed,पूर्ण,Status Value
# Submit,जमा करें,Button

# Load translations in hooks.py:
# override_doctype_translations = {
#     "hi": "my_app/translations/hi.csv"
# }
```

## Security Standards

### OWASP Top 10 for Frappe Apps

1. **Injection** — Use `frappe.qb` or parameterized queries, never string formatting
2. **Broken Auth** — Use Frappe's built-in auth, enforce 2FA for admin roles
3. **Sensitive Data Exposure** — Encrypt PII, mask in API responses
4. **XML External Entities** — Frappe handles this; don't parse untrusted XML manually
5. **Broken Access Control** — Always check permissions in whitelisted methods
6. **Security Misconfiguration** — Disable debug mode in production, restrict CORS
7. **XSS** — Use `frappe.utils.escape_html()`, avoid `innerHTML` in client scripts
8. **Insecure Deserialization** — Use `json.loads()` not `eval()` or `pickle`
9. **Known Vulnerabilities** — Keep Frappe updated, run `bench update`
10. **Insufficient Logging** — Enable audit trails, monitor failed login attempts

### Production Security Checklist

```bash
# Verify security settings
bench --site production.dris.com set-config disable_website_cache 0
bench --site production.dris.com set-config rate_limit '{"window": 86400, "limit": 1000}'
bench --site production.dris.com set-config enable_two_factor_auth 1

# Check for common issues
bench --site production.dris.com doctor
bench --site production.dris.com show-config | grep -i "debug\|cors\|rate"
```
