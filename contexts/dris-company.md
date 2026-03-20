---
name: dris-company-context
description: Company context for Dhwani Rural Information System (DRIS) — tech stack, team structure, departments, and AI usage patterns.
---

# Dhwani Rural Information System (DRIS) — Company Context

## About

Dhwani Rural Information System (DRIS) builds custom technology solutions for government agencies and CSR (Corporate Social Responsibility) clients. The company specializes in digital platforms for rural development, beneficiary management, project tracking, and impact measurement.

## Products

### Amgrant v3 (Current)
- **Stack**: Frappe framework (Python/JS), MySQL
- **Architecture**: Frappe monolith with multiple modules
- **Purpose**: Primary product — government and CSR project management

### Amgrant v2 (Legacy, Active)
- **Stack**: Angular (frontend), Node.js microservices (backend), MongoDB
- **Architecture**: Microservices with API gateway (Nginx)
- **Purpose**: Legacy version — still running for some clients, gradual migration to v3

### Amgrant Form (Active)
- **Stack**: Angular (frontend), Express.js (backend), MongoDB
- **Architecture**: Monolith
- **Purpose**: Form/data collection application

## Tech Stack

- **Frappe Stack**: Frappe v15+, MySQL, Redis, Python, Vue.js (Frappe UI)
- **Node Stack**: Angular, Node.js, Express.js, MongoDB, Redis
- **Database**: MySQL (Frappe/v3), MongoDB (v2/Form), PostgreSQL (some projects)
- **Frontend**: Frappe UI (Vue.js), Angular
- **Testing**: Playwright (E2E), Frappe test runner, Jest/Jasmine (Angular/Node)
- **DevOps**: Bench (Frappe), PM2 (Node), Docker, GitHub Actions, Supervisor, Nginx
- **AI Tools**: Claude Code, Claude UI, Figma AI, GitHub Copilot

## Team Structure & Departments

### Development
- **Frappe Developers** — Build custom Frappe apps, DocTypes, APIs, dashboards
- **Backend Developers** — Server-side logic, integrations, background jobs
- **Frontend Developers** — Custom pages, dashboards, reports, UI/UX
- **Database Team** — PostgreSQL optimization, migrations, data modeling

### Product & Project Management
- **Product Managers (PMs)** — Figma wireframing, first demos, product specs
- **Project Managers** — Client delivery, milestone tracking, walkthroughs
- **Directors** — GitHub analytics, team oversight, strategic decisions

### Quality & Operations
- **QA Team** — E2E testing (Playwright), functional testing, regression testing, test automation
- **DevOps** — Load testing (k6/Locust), CI/CD, infrastructure, monitoring, deployments
- **Sales** — Proposals, client demos, scoping
- **HR & Admin** — Internal processes, onboarding

## Current AI Usage Patterns

| Role | AI Usage |
|------|----------|
| PMs | Figma wireframing, first demo generation from Claude, walkthrough writing |
| Frappe Devs | Building dashboards via Frappe API, DocType generation |
| Directors | GitHub repo analytics, developer productivity insights |
| Developers | Code generation, feature building, documentation |
| Project Managers | Walkthroughs, Playwright test code, scope documents |
| QA | Playwright E2E tests, functional test generation, regression suites |
| DevOps | Load testing scripts, CI/CD pipelines, infrastructure automation |
| Analysts/Data Engineers | SQL queries, report building, data pipelines |

## Domain Context

- **Government Projects**: District/state-level data systems, beneficiary tracking, scheme management
- **CSR Projects**: Impact measurement, project tracking, reporting dashboards
- **Key Concerns**: Data security, audit trails, role-based access, Hindi/regional language support, accessibility (WCAG), offline-capable features for rural areas
- **Compliance**: Government IT security guidelines, data protection, audit requirements

## Relevant ECC Skills & Agents

### Skills to Use
- `frappe-development` — Core Frappe patterns and best practices
- `pm-workflow` — PM and project manager workflows
- `qa-testing` — QA testing patterns with Playwright
- `devops-loadtesting` — Load testing and infrastructure
- `github-analytics` — Director-facing GitHub insights
- `postgres-frappe-patterns` — PostgreSQL optimization for Frappe
- `govt-csr-compliance` — Government and CSR compliance patterns

### Agents to Use
- `frappe-reviewer` — Review Frappe code for security and quality
- `database-reviewer` — PostgreSQL query and schema review
- `code-reviewer` — General code quality review
- `e2e-runner` — Run Playwright E2E tests

### Existing ECC Resources (Already Available)
- `/code-review` — General code review command
- `/tdd` — Test-driven development
- `/e2e` — E2E test generation
- `/plan` — Implementation planning
- `/build-fix` — Build error resolution
- `security` rules — Security guidelines
- `playwright` MCP config — Browser automation
- `github` MCP config — GitHub integration
