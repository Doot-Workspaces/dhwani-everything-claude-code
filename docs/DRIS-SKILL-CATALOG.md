# DRIS Skill Catalog

> Everything AI by Dhwani RIS — 49 skills across 5 clusters
>
> Interactive graph: [`skill-graph.html`](./skill-graph.html) · Mermaid diagrams: [`SKILL_GRAPH.md`](./SKILL_GRAPH.md) · Data index: [`SKILL_GRAPH.json`](./SKILL_GRAPH.json)

---

## 🟡 Domain Knowledge (6 skills)

India social sector domain expertise. Load these as context alongside builder or product skills — they don't generate code, they provide regulatory and operational knowledge.

| # | Skill | What It Knows | When to Use | When NOT to Use |
|---|-------|--------------|-------------|-----------------|
| 1 | **csr-compliance-india** | Companies Act 2013 §135, Schedule VII, 11 MCA reports, penalties, spend calculation, implementing agency rules | Indian CSR law, MCA forms, Schedule VII | Non-India contexts; technical implementation |
| 2 | **ngo-operations** | Trust/Society/Section 8 structures, 12A/80G/FCRA/DARPAN/CSR-1, governance, HR, procurement | NGO governance, registrations, financial management | For-profit company operations |
| 3 | **development-sector-grants** | Grant lifecycle (9 stages), Theory of Change, LogFrame, M&E, SDGs, SROI, due diligence | Program design, M&E planning, grant lifecycle | Non-grant projects; pure software work |
| 4 | **grant-management-operations** | Fund flow, UCs, SoEs, fund requests, budget management, grant closure, multi-donor apportionment | Fund flow, UCs, SoEs, budget tracking | Conceptual program design |
| 5 | **fcra-compliance** | FCRA 2010 + 2020 amendments, designated SBI account, 20% admin cap, FC-4/FC-6, sub-granting ban | Foreign funding, FCRA registration, FC-4 filing | Domestic-only funding |
| 6 | **data-mis-design** | ODK, KoBoToolbox, mForm, MIS design, dashboards, data governance, DPDP Act | Building data collection systems, dashboards, MIS | Non-data tasks |

**Cross-references:** These 6 form a tightly interconnected sub-graph. Every node connects to at least 3 others. Load 2–3 relevant ones together, not all 6.

---

## 🟢 Design Skills (7 skills)

UI/UX/Architecture frameworks. The design review chain is: Frontend Designer → UX Reviewer → Accessibility Auditor.

| # | Skill | What It Does | When to Use | When NOT to Use |
|---|-------|-------------|-------------|-----------------|
| 7 | **frontend-designer** | Anti-AI-slop design, 30+ aesthetic tones, brand defaults, design thinking protocol | Any new UI work, landing pages, dashboards | Pure backend work; Frappe-only UI |
| 8 | **ux-reviewer** | 10-priority UX review (accessibility → data viz), PASS/FAIL scoring | Reviewing existing UI for quality | Building new UI (use frontend-designer first) |
| 9 | **accessibility-auditor** | WCAG 2.1 AA audit/fix, contrast calculation, auto-fix mode, Dhwani brand validation | WCAG compliance check, fixing a11y issues | Non-UI work |
| 10 | **react-composition-patterns** | Compound components, providers, React 19 changes, boolean prop elimination | Refactoring React components with anti-patterns | Vue/Angular/Svelte; simple components |
| 11 | **frappe-ui-design** | ui.frappe.io components only, light theme, Dhwani brand, module scaffolding | Building Frappe-based product UIs | Non-Frappe projects; backend-only work |
| 12 | **system-architecture** | 5-pillar thinking, 20-question framework, AI-aware decomposition, constraint navigation | Architecture decisions, integration planning, tech choices | Simple CRUD; single-service apps |
| 13 | **figma-to-powerbi** | Figma dashboard → PowerBI: theme JSON, layout spec, PBIR files, DAX measures | Converting Figma dashboard designs to PowerBI reports | Non-PowerBI targets; no Figma source |

---

## 🔴 Product Workflows (17 skills)

Dhwani RIS-specific PM, BA, engineering, and delivery workflows. The Golden Path runs through 10 of these. The Skill Selector (🔶) routes to all others.

| # | Skill | Role | What It Does | Input → Output |
|---|-------|------|-------------|----------------|
| 14 | **dris-business-analyst** ⭐① | BA | Structures client calls/emails/MOMs into formal requirements | Raw inputs → BRD + Requirements Matrix |
| 15 | **dris-feature-spec** ⭐② | PM | Creates structured feature specs from ideas | Raw idea → .docx feature spec |
| 16 | **dris-technical-pm** ⭐③ | PM/Architect | Solution architecture using COVE method | Feature spec → Solution architecture |
| 17 | **brd-to-implementation-spec** ⭐④ | Architect | Transforms BRD into Claude Code-ready tech spec (6 phases) | BRD → Frappe v16 tech spec |
| 18 | **dris-frappe-doctype** ⭐⑤ | Engineer | Designs DocType JSON with workflows, permissions, approval chains | Spec → DocType JSON + workflow defs |
| 19 | **dris-pr-raiser** ⭐⑧ | Engineer | Structures code into clean GitHub PRs | Completed changes → GitHub PR |
| 20 | **dris-release-notes** ⭐⑨ | PM | Generates release notes from Git + Linear | Git log → Internal notes + customer email |
| 21 | **dris-email-drafter** ⭐⑩ | PM | Pattern-based professional client emails | Context → Draft email for Outlook |
| 22 | **dris-ticketing** | PM | Generates structured Linear tickets from specs (batch) | Feature spec → Linear tickets |
| 23 | **dris-skill-selector** 🔶 | Router | Central dispatcher — routes any task to the right skill | Task description → Skill + execution order |
| 24 | **dris-product-docs** | Writer | Generates Frappe Wiki user documentation | Completed feature → Wiki .md chapters |
| 25 | **dris-weekly-update** | PM | Friday reports from real Git commit history | Git log + Linear → .docx report |
| 26 | **dris-mgrant-donor** | Domain | 40+ DocTypes, 5 grant flows — complete donor platform context | mGrant Donor questions → Product context |
| 27 | **dris-mgrant-csr** | Domain | MCA §135, budget lifecycle — CSR module compliance | CSR questions → MCA context |
| 28 | **dris-mgrant-nuo** | Domain | NGO grants, sub-grants, FCRA — NGO-facing platform | NuO questions → NGO grant context |
| 29 | **dris-mgrant-bulk-upload** | Engineer | PREP-MAP-SAMPLE-EXECUTE with PM gates for safe import | Data → Gated bulk import |
| 30 | **dris-mgrant-setup** | Engineer | 11-phase configuration for new deployments | Setup requirements → Config sequence |

⭐ = Golden Path step (number indicates order)

---

## 🔵 Builder / Technical (13 skills)

Technical implementation patterns built by Ankit Jangir. `frappe-development` and `qa-testing` are the most comprehensive Frappe-specific patterns in any public repo.

| # | Skill | Focus | When to Use |
|---|-------|-------|-------------|
| 31 | **frappe-development** ⭐⑥ | DocType design, API security, hooks, testing — 500 lines | ALL Frappe code |
| 32 | **qa-testing** ⭐⑦ | Playwright E2E, POM, CI, permission testing — 569 lines | E2E test setup |
| 33 | **pm-workflow** | Figma-to-code, walkthrough generation, demo prep | Demo preparation |
| 34 | **angular-node-patterns** | Angular + Express + MongoDB microservices | Angular/Express projects |
| 35 | **mongodb-patterns** | Schema design, aggregation, encryption | MongoDB projects |
| 36 | **microservices-patterns** | Service communication, API gateway | Multi-service architecture |
| 37 | **devops-loadtesting** | k6, Locust, monitoring — performance testing | Performance validation |
| 38 | **devops-scheduling** | Frappe scheduler, PM2, cron | Background jobs, scheduled tasks |
| 39 | **github-analytics** | GitHub CLI, DORA metrics | Engineering metrics |
| 40 | **postgres-frappe-patterns** | PostgreSQL optimization for Frappe | Query optimization |
| 41 | **govt-csr-compliance** | Audit trails, PII, RBAC, WCAG — implementation level | Building compliance in code |
| 42 | **knowledge-base** | Team KBs, CLAUDE.md templates | Project knowledge management |
| 43 | **documentation-workflow** | API docs, user guides, release notes workflow | API documentation, technical writing |

---

## ⚪ Templates & Guides (6 files)

Reference material — use at project start or when creating new skills.

| # | File | Purpose | When to Use |
|---|------|---------|-------------|
| 44 | **claude-template** | Generator prompt for CLAUDE.md in any repo | Start of every new repo |
| 45 | **evals-template** | Skill evaluation test framework — 3+ test cases | Every new skill |
| 46 | **spec-template** | Technical spec skeleton — 13 sections + 3 appendices (Frappe v16) | Frappe v16 projects |
| 47 | **tooling-reference** | ECC + Autoresearch + Paperclip install and usage | Setting up AI tools |
| 48 | **vibe-coding-how-to** | Non-technical onboarding guide for Claude Code | Onboarding PMs |
| 49 | **skill-authoring-standards** | Frontmatter, evals, semver standards for new skills | Creating new skills |

---

## Contributors

| Person | Affiliation | What They Built |
|--------|------------|----------------|
| **Affaan Mustafa** | Community (Everything Claude Code) | Base repo — 120+ community skills, hooks, commands, agents, rules |
| **Ankit Jangir** | Dhwani RIS | 13 technical skills, 2 Frappe agents, 5 rules, DRIS company context, safety guardrails |
| **Swapnil Agarwal** | Dhwani RIS | BRD-to-implementation-spec skill, spec template, tooling reference — the BRD → Code pipeline |
| **Nihaan Mohammed** | Dhwani RIS | 6 domain skills, 6 design skills, 16 product workflow skills, templates, guides, skill graph architecture |

*This is a living document. As new skills are added via PR, update this catalog and the corresponding `SKILL_GRAPH.json`.*

---

## The Multiplier Effect

| Metric | Before | After |
|--------|--------|-------|
| BRD → Code time | 2–3 weeks | 2–3 days |
| Domain errors (CSR, FCRA) | Frequent | Near-zero |
| Design consistency | Variable | Enforced |
| New PM/engineer onboarding | 2 weeks shadowing | Day 1 productive |
| Client communication quality | Inconsistent | Templated + contextualized |
