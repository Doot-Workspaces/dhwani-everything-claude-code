# Frappe Testing Rules

## Unit Tests

- Every DocType must have a corresponding `test_<doctype>.py` file.
- Use `frappe.tests.utils.FrappeTestCase` as the base class.
- Tests must be idempotent — create their own test data, clean up after.
- Use `frappe.set_user()` to test permission scenarios with different roles.
- Test both positive paths (valid data saves) and negative paths (validation errors thrown).

## Test Data

- Never rely on demo data or production data existing.
- Create test records in `setUp()` or test fixtures.
- Use `frappe.mock()` sparingly — prefer real DocType instances for integration testing.
- Name test records with a prefix like `_Test` to easily identify and clean up.

## API Testing

- Test every `@frappe.whitelist()` method with valid and invalid inputs.
- Test permission enforcement: call as unauthorized user, expect `frappe.PermissionError`.
- Test with `allow_guest` scenarios — verify guest cannot access protected methods.
- Verify JSON parsing of string arguments (Frappe sends args as strings).

## E2E Testing (Playwright)

- Use Page Object Model for Frappe pages (login, list view, form view).
- Single worker (`workers: 1`) — Frappe sessions conflict with parallel tests.
- Set timeout to 60s — Frappe pages can be slow on first load.
- Test on Chromium + mobile viewport (government users on mobile devices).
- Include permission tests: verify users only see their authorized data.
- Test print formats — government clients always verify print output.

## Running Tests

```bash
# All tests for an app
bench --site test_site run-tests --app my_app

# Specific DocType
bench --site test_site run-tests --doctype "My Doctype"

# Specific test file
bench --site test_site run-tests --module my_app.my_module.test_my_module

# With coverage
bench --site test_site run-tests --app my_app --coverage
```

## CI Requirements

- All tests must pass before merge to main/develop.
- E2E tests run on PR to main branch.
- Load tests run before production deployment.
- Test results and coverage reports uploaded as artifacts.
