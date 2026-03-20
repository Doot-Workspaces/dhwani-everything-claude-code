---
name: pm-workflow
description: PM and Project Manager workflow patterns for Figma-to-code pipeline, wireframe generation, demo preparation, walkthrough creation, and Playwright test writing from product specs.
origin: DRIS
---

# PM & Project Manager Workflow Patterns

## When to Activate

- PM creating wireframes or first demos from specs
- Converting Figma designs to working prototypes
- Writing product walkthroughs and documentation
- Generating Playwright test scripts from user stories
- Preparing client demos for government/CSR stakeholders
- Building project proposals and scope documents

## Figma-to-Code Pipeline

### Step 1: Extract Design Tokens from Figma

```javascript
// Design token extraction pattern
// PM exports design tokens from Figma → feed to Claude for scaffolding

const designTokens = {
  colors: {
    primary: '#1a73e8',        // From Figma styles
    secondary: '#5f6368',
    success: '#34a853',
    warning: '#fbbc04',
    error: '#ea4335',
    background: '#ffffff',
    surface: '#f8f9fa',
  },
  typography: {
    heading: { family: 'Inter', weight: 600, size: '24px' },
    body: { family: 'Inter', weight: 400, size: '14px' },
    caption: { family: 'Inter', weight: 400, size: '12px' },
  },
  spacing: { xs: '4px', sm: '8px', md: '16px', lg: '24px', xl: '32px' },
  borderRadius: { sm: '4px', md: '8px', lg: '12px' },
};
```

### Step 2: Generate Component Scaffold

Prompt pattern for Claude:

```
Given this Figma screen [describe or paste screenshot], generate:
1. HTML structure with semantic elements
2. CSS using the design tokens above
3. Interactive states (hover, active, disabled)
4. Responsive breakpoints (mobile-first)
5. Accessibility attributes (aria-labels, roles)
```

### Step 3: First Demo Assembly

```bash
# Quick prototype setup for client demos
npx create-react-app demo-prototype --template typescript
# OR for Frappe-based demos:
bench new-site demo.local
bench --site demo.local install-app my_app
```

## Walkthrough Generation

### User Story to Walkthrough Pattern

```markdown
## Walkthrough: [Feature Name]

### Prerequisites
- User role: [Admin/Manager/User]
- Required data: [What needs to exist]

### Steps

1. **Navigate to [Module]**
   - URL: `/app/[doctype]`
   - Expected: List view with [columns] visible

2. **Create New [Document]**
   - Click "+ Add [Doctype]"
   - Fill required fields:
     - Field 1: [description, validation rules]
     - Field 2: [description, validation rules]
   - Click "Save"
   - Expected: Document saved with status "Draft"

3. **Submit for Approval**
   - Click "Submit" button
   - Confirm in dialog
   - Expected: Status changes to "Pending Approval"
   - Expected: Email notification sent to [role]

### Error Scenarios
- [What happens if X field is empty]
- [What happens if user lacks permission]

### Screenshots
- [Reference Figma frame: Frame Name]
```

### Auto-Generate Walkthrough from DocType

```python
# Utility: Generate walkthrough skeleton from DocType definition
import frappe
import json

def generate_walkthrough(doctype):
    """Generate a walkthrough template from DocType metadata."""
    meta = frappe.get_meta(doctype)

    walkthrough = {
        "title": f"Walkthrough: {doctype}",
        "module": meta.module,
        "url": f"/app/{frappe.scrub(doctype)}",
        "fields": [],
        "workflow_states": [],
    }

    for field in meta.fields:
        if field.reqd or field.in_list_view:
            walkthrough["fields"].append({
                "label": field.label,
                "fieldtype": field.fieldtype,
                "required": bool(field.reqd),
                "options": field.options,
                "description": field.description or "",
            })

    if meta.has_workflow:
        workflow = frappe.get_doc("Workflow", {"document_type": doctype})
        for state in workflow.states:
            walkthrough["workflow_states"].append({
                "state": state.state,
                "allowed_roles": state.allow_edit,
            })

    return walkthrough
```

## Playwright Test Generation from User Stories

### Pattern: User Story → Test Script

Given a user story:
> As a **Project Manager**, I want to **create a project milestone** so that I can **track delivery progress**.

Generate Playwright test:

```typescript
import { test, expect } from '@playwright/test';

test.describe('Project Milestone Creation', () => {
  test.beforeEach(async ({ page }) => {
    // Login as Project Manager
    await page.goto('/login');
    await page.fill('#login_email', 'pm@dris.com');
    await page.fill('#login_password', process.env.TEST_PASSWORD!);
    await page.click('.btn-login');
    await page.waitForURL('/app');
  });

  test('PM can create a new milestone', async ({ page }) => {
    // Navigate to project
    await page.goto('/app/project/PROJ-001');
    await page.waitForSelector('.page-title');

    // Open milestone section
    await page.click('[data-doctype="Project Milestone"] .btn-add');

    // Fill milestone details
    await page.fill('[data-fieldname="milestone"] input', 'UAT Completion');
    await page.fill('[data-fieldname="milestone_date"] input', '2026-04-15');

    // Save
    await page.click('[data-action="save"]');

    // Verify
    await expect(page.locator('.msgprint')).toContainText('Saved');
    await expect(page.locator('[data-doctype="Project Milestone"]'))
      .toContainText('UAT Completion');
  });

  test('PM cannot create milestone without date', async ({ page }) => {
    await page.goto('/app/project/PROJ-001');
    await page.click('[data-doctype="Project Milestone"] .btn-add');
    await page.fill('[data-fieldname="milestone"] input', 'Missing Date Test');

    // Try to save without date
    await page.click('[data-action="save"]');

    // Verify validation error
    await expect(page.locator('.msgprint-dialog')).toBeVisible();
    await expect(page.locator('.msgprint-dialog')).toContainText('Milestone Date is mandatory');
  });

  test('milestone appears in project timeline', async ({ page }) => {
    await page.goto('/app/project/PROJ-001');

    // Check timeline/activity
    const milestones = page.locator('[data-doctype="Project Milestone"] .rows .row');
    await expect(milestones).toHaveCount(await milestones.count());

    // Verify milestone data is correct
    const firstMilestone = milestones.first();
    await expect(firstMilestone).toContainText('UAT Completion');
  });
});
```

### Batch Test Generation Prompt

```
Given these user stories for [DocType/Feature]:

1. [User story 1]
2. [User story 2]
3. [User story 3]

Generate Playwright tests covering:
- Happy path for each story
- Validation/error cases
- Permission checks (test with unauthorized role)
- Data integrity (verify DB state after actions)

Use Page Object Model pattern with:
- pages/login.page.ts
- pages/[feature].page.ts
- fixtures/test-data.ts
```

## Client Demo Preparation

### Demo Checklist for Government/CSR Clients

```markdown
## Pre-Demo Checklist

### Data Setup
- [ ] Sample master data loaded (departments, employees, projects)
- [ ] Realistic test data (not lorem ipsum — use domain-relevant content)
- [ ] Demo user accounts created with correct roles
- [ ] Workflows configured and tested end-to-end

### Environment
- [ ] Demo site on stable branch (not dev)
- [ ] Performance acceptable (no loading spinners > 2s)
- [ ] Mobile responsive tested (many govt users on tablets)
- [ ] Print formats working (govt clients always check print)

### Presentation Flow
1. Login → Dashboard overview (30 sec)
2. Core workflow demo (5 min)
3. Reports and analytics (3 min)
4. Mobile/tablet view (2 min)
5. Q&A buffer (5 min)

### Backup Plan
- [ ] Screenshots of each step saved locally
- [ ] Offline backup of demo site
- [ ] Fallback slides ready if site is slow
```

## Scope Document Template

```markdown
# Project Scope: [Project Name]

## Client: [Government Agency / CSR Partner]
## Date: [Date]
## Version: [1.0]

### 1. Objectives
- [Primary objective]
- [Secondary objective]

### 2. Modules in Scope
| Module | Features | Priority | Phase |
|--------|----------|----------|-------|
| [Module 1] | [Feature list] | High | Phase 1 |
| [Module 2] | [Feature list] | Medium | Phase 2 |

### 3. Out of Scope
- [Explicitly list what's NOT included]

### 4. Technical Requirements
- **Framework**: Frappe v15+
- **Database**: PostgreSQL
- **Hosting**: [Client infra / Cloud]
- **Integration**: [List external systems]

### 5. Deliverables & Timeline
| Deliverable | Target Date | Acceptance Criteria |
|-------------|-------------|---------------------|
| Phase 1 Demo | [Date] | [Criteria] |
| UAT | [Date] | [Criteria] |
| Go-Live | [Date] | [Criteria] |

### 6. Assumptions & Dependencies
- [List assumptions about client infra, data, access]
```

## Tips for PMs Using AI Effectively

1. **Be specific in prompts**: "Create a Frappe DocType for tracking CSR project beneficiaries with fields for name, Aadhaar number (masked), location, and benefit amount" beats "create a beneficiary tracker"
2. **Iterate on wireframes**: Start with low-fi HTML, get client feedback, then build full UI
3. **Use Claude for test scripts**: Write user stories, let Claude generate Playwright tests — then review
4. **Demo data matters**: Generate realistic Indian names, addresses, and project names for demos
5. **Version your walkthroughs**: Keep walkthroughs in git alongside code — they drift fast otherwise
