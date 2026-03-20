# AI Usage Safety Guardrails

> These rules apply to ALL users — technical and non-technical — using Claude Code or any AI assistant within DRIS projects. They are designed to prevent accidental harm, data leaks, and unauthorized changes.

## Universal Rules (Everyone)

### Never Share Sensitive Data with AI

- NEVER paste real Aadhaar numbers, PAN numbers, or phone numbers into prompts
- NEVER paste production database credentials, API keys, or tokens
- NEVER paste real beneficiary data — use anonymized or synthetic data
- NEVER paste internal salary, financial, or employee personal information
- If you accidentally paste sensitive data, inform your team lead immediately

### Never Push AI-Generated Code Without Review

- ALL AI-generated code must be reviewed by a human before merging
- Code review is mandatory regardless of how simple the change appears
- Non-developers must get a developer to review any generated code
- PMs generating Playwright tests must have QA review before execution

### Never Run AI-Generated Commands on Production

- AI-suggested shell commands must be reviewed before execution
- NEVER run `rm`, `drop`, `delete`, `truncate`, or destructive commands from AI on production
- NEVER run `bench migrate` or database migration commands without DevOps approval
- Test all commands on local/staging environments first
- If unsure, ask in the team channel before running any command

## Technical User Rules

### Code Generation Safety

```
BEFORE pushing any AI-generated code, verify:
□ No hardcoded credentials, API keys, or tokens
□ No ignore_permissions=True (Frappe) or permission bypass
□ No SQL/NoSQL injection vulnerabilities (string interpolation in queries)
□ No hard deletes of government data
□ No console.log/print statements with sensitive data
□ No disabled security features (CORS *, rate limit off, etc.)
□ Input validation present on all user-facing endpoints
□ Error messages don't expose internal details
```

### Database Safety

- NEVER run AI-generated SQL directly on production
- Always use transactions for multi-step operations
- Test migrations on a fresh backup before production
- AI-generated aggregation pipelines must be reviewed for performance
- No `DROP TABLE`, `DELETE FROM` without WHERE clause
- Always have a rollback plan for schema changes

### Git Safety

- NEVER force push to main, develop, or release branches
- NEVER commit .env files, credentials, or secrets
- AI-generated commit messages must accurately describe changes
- Review diffs before committing — AI might include unintended changes
- Use feature branches for all AI-assisted work

### Deployment Safety

- AI-generated deployment scripts must be reviewed by DevOps
- NEVER auto-deploy AI-generated code to production
- Staging deployment required before production
- Load test results must meet thresholds before go-live
- Rollback procedure must be documented before deployment

## Non-Technical User Rules

### PMs and Project Managers

```
ALLOWED:
✓ Generate wireframes and HTML prototypes
✓ Create walkthrough documents
✓ Write scope documents and project plans
✓ Generate test scenarios (for QA to implement)
✓ Create demo data (synthetic, never real data)

NOT ALLOWED:
✗ Modify production code or databases
✗ Push code to any branch
✗ Run scripts or commands on servers
✗ Share real client data in AI prompts
✗ Commit to timelines without team input
```

### Directors

```
ALLOWED:
✓ Query GitHub for read-only analytics
✓ Generate reports from public data
✓ Analyze team metrics and trends
✓ Create presentations from aggregated data

NOT ALLOWED:
✗ Merge or close PRs
✗ Modify repository settings
✗ Access production databases
✗ Run commands on servers
✗ Use individual developer output for performance evaluation
```

### Sales Team

```
ALLOWED:
✓ Generate proposal drafts
✓ Create scope document templates
✓ Build comparison tables
✓ Generate FAQ documents

NOT ALLOWED:
✗ Commit to technical architecture decisions
✗ Share internal code or technical details
✗ Promise features without technical validation
✗ Access any production system
```

### HR Team

```
ALLOWED:
✓ Generate onboarding documents
✓ Create policy drafts
✓ Build training materials
✓ Generate job descriptions

NOT ALLOWED:
✗ Process employee data through AI
✗ Use AI for performance evaluations
✗ Generate termination or disciplinary documents without legal review
✗ Access any technical system
```

## Data Classification

| Level | Description | AI Usage |
|-------|-------------|----------|
| **Public** | Marketing content, public docs | Free to use in prompts |
| **Internal** | Process docs, meeting notes | Use with caution, no PII |
| **Confidential** | Client data, financials | Never paste in AI prompts |
| **Restricted** | Aadhaar, PAN, credentials | Strictly prohibited in AI |

## Incident Response

If you suspect AI-generated code caused a security issue:

1. **Stop**: Don't try to fix it yourself
2. **Report**: Notify your team lead and DevOps immediately
3. **Document**: Save the AI conversation/prompt that caused the issue
4. **Don't delete evidence**: Keep all logs and generated code
5. **Learn**: The incident will be reviewed to update these guardrails

## Claude Code Permission Model

For DRIS projects, use these permission settings:

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Glob",
      "Grep",
      "Agent"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(drop table)",
      "Bash(bench --site * drop-site)",
      "Bash(git push --force)",
      "Bash(git push origin main)"
    ]
  }
}
```

## Compliance

- These guardrails are mandatory for all DRIS employees and contractors
- Violations should be reported without blame — the goal is learning
- Rules are reviewed monthly and updated based on incidents
- New team members must be briefed on these rules during onboarding
