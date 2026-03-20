---
name: qa-testing
description: QA testing patterns for E2E testing with Playwright, functional testing, regression testing, test automation, and test case management for Frappe and web applications.
origin: DRIS
---

# QA Testing Patterns

## When to Activate

- QA team writing or running E2E tests with Playwright
- Building regression test suites for Frappe applications
- Functional testing of DocType workflows
- Cross-browser and responsive testing
- Test case management and reporting
- API testing for whitelisted Frappe methods

## Playwright E2E for Frappe Applications

### Project Setup

```bash
# Initialize Playwright in Frappe app
cd apps/my_app
npm init playwright@latest

# Recommended config
# playwright.config.ts
```

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests/e2e',
  timeout: 60_000,          // Frappe pages can be slow
  expect: { timeout: 10_000 },
  fullyParallel: false,      // Frappe sessions conflict if parallel
  retries: process.env.CI ? 2 : 0,
  workers: 1,                // Single worker for Frappe (shared session state)
  reporter: [
    ['html', { open: 'never' }],
    ['json', { outputFile: 'test-results/results.json' }],
  ],
  use: {
    baseURL: process.env.SITE_URL || 'http://localhost:8000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'on-first-retry',
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'mobile', use: { ...devices['Pixel 5'] } },   // Govt users on mobile
    { name: 'tablet', use: { ...devices['iPad Pro 11'] } }, // Field workers on tablets
  ],
});
```

### Page Object Model for Frappe

```typescript
// pages/frappe-login.page.ts
import { Page, expect } from '@playwright/test';

export class FrappeLoginPage {
  constructor(private page: Page) {}

  async goto() {
    await this.page.goto('/login');
    await this.page.waitForSelector('#login_email');
  }

  async login(email: string, password: string) {
    await this.page.fill('#login_email', email);
    await this.page.fill('#login_password', password);
    await this.page.click('.btn-login');
    await this.page.waitForURL(/\/app/);
  }

  async expectLoginError(message: string) {
    await expect(this.page.locator('.login-content .alert')).toContainText(message);
  }
}
```

```typescript
// pages/frappe-list.page.ts
import { Page, expect } from '@playwright/test';

export class FrappeListPage {
  constructor(private page: Page, private doctype: string) {}

  private get slug() {
    return this.doctype.toLowerCase().replace(/ /g, '-');
  }

  async goto() {
    await this.page.goto(`/app/${this.slug}`);
    await this.page.waitForSelector('.list-row');
  }

  async getRowCount(): Promise<number> {
    return await this.page.locator('.list-row:not(.list-row-head)').count();
  }

  async clickRow(name: string) {
    await this.page.click(`.list-row [data-name="${name}"]`);
    await this.page.waitForSelector('.page-title');
  }

  async createNew() {
    await this.page.click('.primary-action');
    await this.page.waitForSelector('.form-layout');
  }

  async searchFor(text: string) {
    await this.page.fill('.frappe-control[data-fieldname="name"] input', text);
    await this.page.keyboard.press('Enter');
    await this.page.waitForTimeout(1000);
  }

  async applyFilter(field: string, value: string) {
    await this.page.click('.filter-button');
    await this.page.selectOption('.fieldname-select', field);
    await this.page.fill('.filter-field input', value);
    await this.page.click('.apply-filters');
    await this.page.waitForTimeout(500);
  }
}
```

```typescript
// pages/frappe-form.page.ts
import { Page, expect } from '@playwright/test';

export class FrappeFormPage {
  constructor(private page: Page) {}

  async fillField(fieldname: string, value: string) {
    const field = this.page.locator(`[data-fieldname="${fieldname}"] input, [data-fieldname="${fieldname}"] textarea`);
    await field.fill(value);
  }

  async selectField(fieldname: string, value: string) {
    await this.page.click(`[data-fieldname="${fieldname}"] .control-input`);
    await this.page.click(`.awesomplete li:has-text("${value}")`);
  }

  async fillLinkField(fieldname: string, value: string) {
    const input = this.page.locator(`[data-fieldname="${fieldname}"] input`);
    await input.fill(value);
    await this.page.waitForSelector('.awesomplete li');
    await this.page.click(`.awesomplete li:has-text("${value}")`);
  }

  async save() {
    await this.page.keyboard.press('Control+s');
    await this.page.waitForSelector('.page-title .indicator-pill');
  }

  async submit() {
    await this.page.click('[data-action="submit"]');
    await this.page.click('.modal-dialog .btn-primary-dark'); // Confirm dialog
    await this.page.waitForSelector('.indicator-pill.blue');
  }

  async expectSaved() {
    await expect(this.page.locator('.page-title .indicator-pill')).toHaveText(/Saved|Not Saved/);
  }

  async expectValidationError(message: string) {
    await expect(this.page.locator('.msgprint-dialog')).toContainText(message);
    await this.page.click('.msgprint-dialog .btn-primary'); // Dismiss
  }

  async getFieldValue(fieldname: string): Promise<string> {
    return await this.page.locator(`[data-fieldname="${fieldname}"] .control-value, [data-fieldname="${fieldname}"] input`).inputValue();
  }

  async addChildRow(tablename: string, values: Record<string, string>) {
    await this.page.click(`[data-fieldname="${tablename}"] .btn-add`);
    const lastRow = this.page.locator(`[data-fieldname="${tablename}"] .rows .row`).last();
    for (const [field, value] of Object.entries(values)) {
      await lastRow.locator(`[data-fieldname="${field}"] input`).fill(value);
    }
  }
}
```

### Test Examples

```typescript
// tests/e2e/project-workflow.spec.ts
import { test, expect } from '@playwright/test';
import { FrappeLoginPage } from '../pages/frappe-login.page';
import { FrappeListPage } from '../pages/frappe-list.page';
import { FrappeFormPage } from '../pages/frappe-form.page';

test.describe('Project Workflow', () => {
  let loginPage: FrappeLoginPage;
  let listPage: FrappeListPage;
  let formPage: FrappeFormPage;

  test.beforeEach(async ({ page }) => {
    loginPage = new FrappeLoginPage(page);
    listPage = new FrappeListPage(page, 'Project');
    formPage = new FrappeFormPage(page);

    await loginPage.goto();
    await loginPage.login('admin@dris.com', process.env.ADMIN_PASSWORD!);
  });

  test('create project with mandatory fields', async ({ page }) => {
    await listPage.goto();
    await listPage.createNew();

    await formPage.fillField('project_name', 'Test CSR Project');
    await formPage.selectField('status', 'Open');
    await formPage.fillLinkField('company', 'Dhwani RIS');
    await formPage.fillField('expected_start_date', '2026-04-01');
    await formPage.fillField('expected_end_date', '2026-06-30');

    await formPage.save();
    await formPage.expectSaved();

    // Verify URL has the new project name
    await expect(page).toHaveURL(/\/app\/project\//);
  });

  test('cannot save project without company', async ({ page }) => {
    await listPage.goto();
    await listPage.createNew();

    await formPage.fillField('project_name', 'Incomplete Project');
    await formPage.save();

    await formPage.expectValidationError('Company is mandatory');
  });

  test('project status workflow', async ({ page }) => {
    await page.goto('/app/project/PROJ-TEST-001');

    // Verify initial state
    await expect(page.locator('.indicator-pill')).toContainText('Open');

    // Add task and verify progress updates
    await formPage.addChildRow('tasks', {
      title: 'Setup environment',
      status: 'Completed',
    });
    await formPage.save();

    // Check progress percentage updated
    const progress = await formPage.getFieldValue('percent_complete');
    expect(parseFloat(progress)).toBeGreaterThan(0);
  });
});
```

### Permission Testing

```typescript
// tests/e2e/permissions.spec.ts
import { test, expect } from '@playwright/test';
import { FrappeLoginPage } from '../pages/frappe-login.page';

test.describe('Role-Based Access Control', () => {
  test('regular user cannot access admin settings', async ({ page }) => {
    const loginPage = new FrappeLoginPage(page);
    await loginPage.goto();
    await loginPage.login('regular@dris.com', process.env.USER_PASSWORD!);

    // Try accessing admin page
    await page.goto('/app/system-settings');
    await expect(page.locator('.page-container')).toContainText('Insufficient Permission');
  });

  test('department user sees only own department data', async ({ page }) => {
    const loginPage = new FrappeLoginPage(page);
    await loginPage.goto();
    await loginPage.login('hr@dris.com', process.env.HR_PASSWORD!);

    await page.goto('/app/employee');
    const rows = page.locator('.list-row:not(.list-row-head)');

    // All visible employees should be from HR department
    for (let i = 0; i < await rows.count(); i++) {
      const dept = await rows.nth(i).locator('[data-field="department"]').textContent();
      expect(dept).toBe('HR');
    }
  });

  test('submitted document cannot be edited by non-admin', async ({ page }) => {
    const loginPage = new FrappeLoginPage(page);
    await loginPage.goto();
    await loginPage.login('regular@dris.com', process.env.USER_PASSWORD!);

    await page.goto('/app/my-doctype/DOC-SUBMITTED-001');

    // Edit button should not be visible for submitted docs
    await expect(page.locator('[data-action="edit"]')).not.toBeVisible();
  });
});
```

## Functional Testing Patterns

### API Testing for Frappe Methods

```typescript
// tests/api/whitelisted-methods.spec.ts
import { test, expect, request } from '@playwright/test';

let apiContext: any;

test.beforeAll(async () => {
  apiContext = await request.newContext({
    baseURL: process.env.SITE_URL || 'http://localhost:8000',
  });

  // Login and get session
  await apiContext.post('/api/method/login', {
    form: {
      usr: 'admin@dris.com',
      pwd: process.env.ADMIN_PASSWORD!,
    }
  });
});

test.afterAll(async () => {
  await apiContext.dispose();
});

test('get_project_dashboard returns correct structure', async () => {
  const response = await apiContext.get('/api/method/my_app.api.get_project_dashboard', {
    params: { project: 'PROJ-001' }
  });

  expect(response.ok()).toBeTruthy();
  const data = await response.json();

  expect(data.message).toHaveProperty('status');
  expect(data.message).toHaveProperty('progress');
  expect(data.message).toHaveProperty('tasks');
  expect(data.message).toHaveProperty('budget');
  expect(Array.isArray(data.message.tasks)).toBe(true);
});

test('unauthorized user gets permission error', async () => {
  // Create context without login
  const guestContext = await request.newContext({
    baseURL: process.env.SITE_URL || 'http://localhost:8000',
  });

  const response = await guestContext.get('/api/method/my_app.api.get_project_dashboard', {
    params: { project: 'PROJ-001' }
  });

  expect(response.status()).toBe(403);
  await guestContext.dispose();
});

test('bulk_update validates permissions', async () => {
  const response = await apiContext.post('/api/method/my_app.api.bulk_update_status', {
    data: {
      doctype: 'Task',
      names: JSON.stringify(['TASK-001', 'TASK-002']),
      status: 'Completed',
    }
  });

  expect(response.ok()).toBeTruthy();
  const data = await response.json();
  expect(data.message.count).toBe(2);
});
```

## Regression Testing Strategy

### Critical Path Tests

```typescript
// tests/e2e/regression/critical-paths.spec.ts
import { test } from '@playwright/test';

/**
 * Critical paths that must pass before every release.
 * Run these in CI on every PR merge.
 */

test.describe('Critical Regression Suite', () => {
  // Auth flows
  test('login with valid credentials', async ({ page }) => { /* ... */ });
  test('password reset flow', async ({ page }) => { /* ... */ });
  test('session timeout redirects to login', async ({ page }) => { /* ... */ });

  // Core CRUD for main DocTypes
  test('create, read, update project', async ({ page }) => { /* ... */ });
  test('submit and cancel workflow', async ({ page }) => { /* ... */ });

  // Reports
  test('project summary report loads', async ({ page }) => { /* ... */ });
  test('report export to CSV works', async ({ page }) => { /* ... */ });

  // Integrations
  test('email notification sent on submit', async ({ page }) => { /* ... */ });

  // Print
  test('print format renders correctly', async ({ page }) => { /* ... */ });
});
```

### Visual Regression

```typescript
// tests/e2e/visual/dashboard.spec.ts
import { test, expect } from '@playwright/test';

test('project dashboard visual check', async ({ page }) => {
  await page.goto('/app/project');
  await page.waitForSelector('.list-row');

  // Take full-page screenshot for visual comparison
  await expect(page).toHaveScreenshot('project-list.png', {
    maxDiffPixelRatio: 0.05,  // Allow 5% diff for dynamic content
  });
});

test('form layout visual check', async ({ page }) => {
  await page.goto('/app/project/PROJ-001');
  await page.waitForSelector('.form-layout');

  await expect(page.locator('.form-layout')).toHaveScreenshot('project-form.png', {
    maxDiffPixelRatio: 0.05,
  });
});
```

## Test Data Management

### Fixtures for Frappe

```typescript
// fixtures/test-data.ts
import { request } from '@playwright/test';

export async function setupTestData(baseURL: string) {
  const api = await request.newContext({ baseURL });

  // Login as admin
  await api.post('/api/method/login', {
    form: { usr: 'Administrator', pwd: process.env.ADMIN_PASSWORD! }
  });

  // Create test project
  await api.post('/api/resource/Project', {
    data: {
      project_name: 'E2E Test Project',
      company: 'Dhwani RIS',
      expected_start_date: '2026-04-01',
      expected_end_date: '2026-06-30',
      status: 'Open',
    }
  });

  // Create test users with roles
  const testUsers = [
    { email: 'qa-admin@dris.com', roles: ['System Manager'] },
    { email: 'qa-pm@dris.com', roles: ['Projects Manager'] },
    { email: 'qa-user@dris.com', roles: ['Projects User'] },
  ];

  for (const user of testUsers) {
    await api.post('/api/resource/User', {
      data: {
        email: user.email,
        first_name: 'QA',
        last_name: user.email.split('@')[0],
        new_password: process.env.TEST_PASSWORD!,
        roles: user.roles.map(r => ({ role: r })),
      }
    });
  }

  await api.dispose();
}

export async function cleanupTestData(baseURL: string) {
  const api = await request.newContext({ baseURL });
  await api.post('/api/method/login', {
    form: { usr: 'Administrator', pwd: process.env.ADMIN_PASSWORD! }
  });

  // Delete test data (use naming convention prefix for easy cleanup)
  await api.delete('/api/resource/Project/E2E Test Project');

  await api.dispose();
}
```

## Test Reporting

### CI Integration

```yaml
# .github/workflows/e2e-tests.yml
name: E2E Tests
on:
  pull_request:
    branches: [main, develop]

jobs:
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 18

      - name: Install Playwright
        run: npx playwright install --with-deps chromium

      - name: Run E2E tests
        run: npx playwright test
        env:
          SITE_URL: ${{ secrets.TEST_SITE_URL }}
          ADMIN_PASSWORD: ${{ secrets.TEST_ADMIN_PASSWORD }}

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 7
```

## QA Checklist by Feature Type

### New DocType Checklist
- [ ] All mandatory fields validated
- [ ] Link field filters working
- [ ] Naming rule generates correct names
- [ ] Permissions tested for each role
- [ ] Workflow transitions tested
- [ ] Print format renders correctly
- [ ] List view columns and filters working
- [ ] Search works in link fields
- [ ] Child table add/remove/reorder works
- [ ] Email notifications triggered correctly

### API Endpoint Checklist
- [ ] Returns correct response structure
- [ ] Permission check works (403 for unauthorized)
- [ ] Input validation (missing/invalid params)
- [ ] Edge cases (empty lists, large payloads)
- [ ] Rate limiting (if applicable)
- [ ] Idempotency for POST operations

### Report Checklist
- [ ] Loads without error
- [ ] Filters work correctly
- [ ] Data matches source records
- [ ] Export to CSV/Excel works
- [ ] Chart renders with data
- [ ] Empty state handled gracefully
- [ ] Date range filters work correctly
