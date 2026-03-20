# Code Push Safety Rules

> These rules ensure that no AI-generated or human-written code causes harm when pushed to shared repositories or deployed to environments.

## Branch Protection

### Protected Branches
- `main` — production deployments, requires 2 approvals
- `develop` — integration branch, requires 1 approval
- `release/*` — release candidates, requires 1 approval + QA sign-off

### Rules
- NO direct pushes to protected branches (use PRs only)
- NO force pushes to any shared branch
- All PRs require at least one human review (not just AI review)
- CI must pass before merge is allowed
- Stale PRs (> 7 days without activity) must be rebased before merge

## Pre-Push Checklist

Every developer must verify before pushing:

```
□ No .env files, credentials, or API keys in the commit
□ No hardcoded passwords or tokens
□ No ignore_permissions=True (Frappe)
□ No disabled security middleware
□ All tests pass locally
□ No console.log/print statements with sensitive data
□ No TODO/FIXME comments that reference security issues
□ Commit messages are descriptive and accurate
□ Changes match the PR description
```

## Git Hooks (Pre-commit)

```bash
#!/bin/bash
# .git/hooks/pre-commit — install via husky or manually

# Check for secrets
if git diff --cached --diff-filter=ACMR | grep -iE "(password|secret|api_key|token|aadhaar|pan_number)" | grep -v "test" | grep -v "mock" | grep -v "example"; then
    echo "ERROR: Potential secret or PII detected in commit. Review changes."
    exit 1
fi

# Check for .env files
if git diff --cached --name-only | grep -E "\.env$|\.env\.|credentials|secrets"; then
    echo "ERROR: Environment/credential file detected. These must not be committed."
    exit 1
fi

# Check for ignore_permissions=True in Frappe code
if git diff --cached --diff-filter=ACMR -- "*.py" | grep "ignore_permissions\s*=\s*True"; then
    echo "WARNING: ignore_permissions=True detected. Ensure this is justified."
    # Warning only, not blocking — but flagged in PR review
fi

# Check for force push markers
if git diff --cached --diff-filter=ACMR | grep -E "allow_guest\s*=\s*True"; then
    echo "WARNING: allow_guest=True detected. This requires security review."
fi
```

## PR Review Requirements

### For AI-Generated Code
- Must be labeled as `ai-generated` or `ai-assisted`
- Requires extra scrutiny on security patterns
- Reviewer must verify logic, not just style
- Test coverage must meet minimum threshold (80%)

### For Production Deployments
- Must pass all CI checks (lint, test, build)
- Must have QA sign-off on staging
- DevOps must approve deployment PR
- Rollback plan must be documented in PR description

## Deployment Pipeline Safety

```
Feature Branch → PR → Code Review → CI Tests → Merge to develop
     ↓
develop → Auto-deploy to staging → QA Testing → Manual approval
     ↓
release/* → PR to main → 2 approvals → Deploy to production
     ↓
Monitor for 30 minutes → Confirm stable → Done
```

### Rollback Triggers
- Error rate > 5% in first 30 minutes
- Response time p95 > 3x baseline
- Any 5xx errors on critical paths
- Data integrity alert from monitoring

## Environment Access Matrix

| Role | Local | Staging | Production |
|------|-------|---------|------------|
| Junior Dev | Full | Read + Deploy | Read-only logs |
| Senior Dev | Full | Full | Read + Emergency deploy |
| Tech Lead | Full | Full | Full (with approval) |
| DevOps | Full | Full | Full |
| QA | Read | Full (testing) | Read-only |
| PM | None | View only | None |
| Director | None | None | View dashboards only |
