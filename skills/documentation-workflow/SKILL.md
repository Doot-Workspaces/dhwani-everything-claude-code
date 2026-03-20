---
name: documentation-workflow
description: Company documentation standards and AI-assisted generation patterns for technical docs, user guides, API documentation, release notes, and internal process documentation.
origin: DRIS
---

# Documentation Workflow

## When to Activate

- Writing or updating technical documentation
- Generating API documentation from code
- Creating user guides for non-technical stakeholders
- Building internal process documentation
- Writing release notes and changelogs
- Creating client-facing documentation for government agencies

## Documentation Types

| Type | Audience | Format | Owner |
|------|----------|--------|-------|
| API Docs | Developers, integrators | Markdown/OpenAPI | Backend team |
| User Guide | End users (govt staff) | PDF/HTML with screenshots | PM team |
| Walkthrough | Client stakeholders | Step-by-step with images | PM/QA team |
| Architecture Doc | Developers, DevOps | Markdown with diagrams | Tech lead |
| Release Notes | All stakeholders | Markdown | Product owner |
| Runbook | DevOps, on-call | Markdown with commands | DevOps team |
| Onboarding Guide | New hires | Markdown/Notion | HR + Tech lead |
| Scope Document | Client, management | PDF/Docx | PM/Sales |

## API Documentation

### OpenAPI Spec from Express Routes

```typescript
// Generate OpenAPI documentation inline with routes

/**
 * @openapi
 * /api/beneficiaries:
 *   get:
 *     summary: List beneficiaries
 *     tags: [Beneficiary]
 *     security:
 *       - BearerAuth: []
 *     parameters:
 *       - in: query
 *         name: page
 *         schema:
 *           type: integer
 *           default: 1
 *       - in: query
 *         name: limit
 *         schema:
 *           type: integer
 *           default: 20
 *       - in: query
 *         name: district
 *         schema:
 *           type: string
 *       - in: query
 *         name: status
 *         schema:
 *           type: string
 *           enum: [active, inactive, archived]
 *     responses:
 *       200:
 *         description: Paginated list of beneficiaries
 *       401:
 *         description: Authentication required
 *       403:
 *         description: Insufficient permissions
 */
router.get('/', authenticate, controller.getAll);
```

### Frappe API Documentation Template

```markdown
# API: [Module Name]

## Endpoints

### Get Project Dashboard
**Method:** `my_app.api.get_project_dashboard`
**HTTP:** `GET /api/method/my_app.api.get_project_dashboard`

**Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| project | string | Yes | Project ID (e.g., "PROJ-001") |

**Response:**
```json
{
  "message": {
    "status": "Open",
    "progress": 45.5,
    "tasks": [
      {"name": "TASK-001", "subject": "Setup", "status": "Completed"}
    ],
    "budget": {
      "allocated": 500000,
      "spent": 225000,
      "remaining": 275000
    }
  }
}
```

**Permissions:** Requires "Project" read permission.

**Errors:**
| Code | Description |
|------|-------------|
| 403 | User lacks permission to read this project |
| 404 | Project not found |
```

## User Guide Template

### For Government/CSR End Users

```markdown
# [Feature Name] - User Guide

## Overview
[One paragraph: what this feature does and who uses it]

## Prerequisites
- [ ] You have a [Role Name] account
- [ ] You have access to [Module Name]

## How To: [Primary Task]

### Step 1: Navigate to [Page]
1. Login to the system
2. Click on **[Module]** in the sidebar
3. You will see the [Page Name]

   ![Screenshot: Navigation](./images/step1-navigation.png)

### Step 2: [Action]
1. Click the **[Button]** button
2. Fill in the following fields:

   | Field | What to Enter | Example |
   |-------|--------------|---------|
   | [Field 1] | [Description] | [Example value] |
   | [Field 2] | [Description] | [Example value] |

3. Click **Save**

   ![Screenshot: Form filled](./images/step2-form.png)

### Step 3: [Verification]
1. You should see a success message: "[Message]"
2. The record will appear in the list with status **[Status]**

## Troubleshooting

| Problem | Solution |
|---------|----------|
| "Permission Error" | Contact your administrator to grant [Role] access |
| Form won't save | Check that all mandatory fields (marked with *) are filled |
| Can't find the module | Your role may not have access — contact admin |

## Getting Help
- Contact: [support email/phone]
- Hours: [working hours]
```

## Release Notes Template

```markdown
# Release Notes - [Product Name] v[X.Y.Z]

**Date:** [Release Date]
**Environment:** [Production/Staging]

## Summary
[2-3 sentences summarizing this release]

## New Features
- **[Feature 1]**: [One-line description]. [Who benefits].
- **[Feature 2]**: [One-line description]. [Who benefits].

## Improvements
- [Improvement 1]: [What changed and why]
- [Improvement 2]: [What changed and why]

## Bug Fixes
- Fixed: [Description of bug and fix]. Reported by [team/client].
- Fixed: [Description of bug and fix].

## Breaking Changes
- [If any — describe what changes and what to do]

## Migration Steps
1. [Step 1]
2. [Step 2]

## Known Issues
- [Issue]: [Workaround if any]. Fix planned for v[X.Y.Z+1].

## Deployment Notes
- [Any special deployment instructions]
- [Database migration required: Yes/No]
- [Downtime expected: Yes/No, duration]
```

## Runbook Template (DevOps)

```markdown
# Runbook: [Service/Process Name]

## Service Overview
- **Service:** [Name]
- **Owner:** [Team/Person]
- **Repository:** [GitHub URL]
- **Monitoring:** [Dashboard URL]
- **Alerting:** [Alert channel]

## Health Check
```bash
# Quick health check
curl -s https://[service-url]/health | jq .

# Expected response
# {"status":"ok","database":"connected","workers":4}
```

## Common Operations

### Restart Service
```bash
# Frappe (bench)
sudo supervisorctl restart all

# Node.js (PM2)
pm2 restart [service-name]

# Docker
docker-compose restart [service-name]
```

### View Logs
```bash
# Frappe
tail -f ~/frappe-bench/logs/web.log
tail -f ~/frappe-bench/logs/worker.log

# Node.js
pm2 logs [service-name] --lines 100

# Docker
docker-compose logs -f --tail=100 [service-name]
```

### Rollback
```bash
# Frappe
cd ~/frappe-bench
git -C apps/my_app checkout [previous-tag]
bench --site [site] migrate
bench restart

# Node.js (Docker)
docker-compose pull [service-name]:[previous-tag]
docker-compose up -d [service-name]
```

## Incident Response
1. **Assess:** Check health endpoint and logs
2. **Communicate:** Post in #incidents channel
3. **Mitigate:** Restart service or rollback
4. **Investigate:** Check recent deployments and changes
5. **Document:** Write post-mortem
```

## Documentation Generation with Claude

### Prompt Patterns

```markdown
# Auto-generate docs from code:
"Read [file path] and generate API documentation in the format described
in skills/documentation-workflow"

# Generate user guide from walkthrough:
"Convert this walkthrough into a user guide suitable for non-technical
government staff. Include troubleshooting section."

# Generate release notes from git log:
"Read the git log since last release tag and generate release notes
following the template in skills/documentation-workflow"

# Generate runbook from deployment scripts:
"Read the deployment scripts in scripts/ and generate a runbook for
DevOps following the template in skills/documentation-workflow"
```

## Documentation Review Checklist

- [ ] Accurate (reflects current system behavior)
- [ ] Complete (covers all user-facing features/APIs)
- [ ] Clear language (no unexplained jargon for non-tech audience)
- [ ] Screenshots current (match current UI)
- [ ] Versioned (tied to specific release)
- [ ] Accessible (readable on mobile, Hindi translation if needed)
- [ ] Secure (no internal URLs, credentials, or sensitive config exposed)
