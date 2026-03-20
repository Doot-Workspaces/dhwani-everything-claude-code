# Contributing — Dhwani RIS Everything Claude Code

This repository is **internal to Dhwani Rural Information Systems**. All contributors must be Dhwani RIS team members with repository access granted by the maintainer.

**Maintainer:** Ankit Jangir (ankitjangir.1690@gmail.com)

---

## Contribution Flow

```
1. Pull latest main
2. Create a feature branch
3. Make your changes
4. Test locally with Claude Code
5. Push and create a PR
6. Ankit reviews and merges
```

### Branch Naming

```
feat/skill-name          — New skill or major addition
fix/skill-name           — Fix to an existing skill
docs/what-changed        — Documentation only
enhance/skill-name       — Enhancement to existing skill
```

### Commit Messages

```
feat(skills): add new-skill-name
feat(dris): add dris-prefixed-skill
fix(skills): correct typo in csr-compliance-india
docs: update README with new skill catalog
enhance(rules): add credential management to frappe-security
```

---

## Adding a New Skill

### 1. Check for Duplicates

Before creating a new skill, search existing skills:

```bash
ls skills/ | grep -i "your-keyword"
grep -ri "your-keyword" skills/ --include="*.md" -l
```

If a skill covers >30% of what you want to add, extend it instead of creating a new one.

### 2. Choose the Right Category

| Prefix/Location | Category | Who Uses It | License |
|-----------------|----------|------------|---------|
| `skills/dris-*` | Product workflow | PMs, BAs, delivery | MIT |
| `skills/csr-compliance-india/` etc. | Domain knowledge | Anyone needing sector expertise | **Proprietary** |
| `skills/dris-mgrant-*` | mGrant product | mGrant team | **Proprietary** |
| `skills/frontend-designer/` etc. | Design | UI/UX, frontend | MIT |
| `skills/frappe-development/` etc. | Technical | Engineers | MIT |
| `docs/templates/` | Templates | All | MIT |
| `docs/guides/` | Guides | All | MIT |

### 3. Create the Skill File

```
skills/your-skill-name/SKILL.md
```

Use Doot ECC frontmatter:

```yaml
---
name: your-skill-name
description: "One-line description — specific, action-oriented"
origin: ECC
---
```

### 4. Required Sections

Every skill must have:

| Section | Required | Purpose |
|---------|----------|---------|
| When to Use | Yes | Prevents false triggers and wasted context |
| Protocol / Steps | Yes | The actual workflow |
| Key Rules | Yes | Non-negotiable constraints |
| Output Format | Yes | What the skill produces |
| Examples | Yes | At least one worked example |
| Common Pitfalls | Recommended | What agents get wrong |
| Cross-References | Recommended | Links to related skills |

See `docs/templates/SKILL_TEMPLATE.md` for the full template.

### 5. Add Evals (Recommended)

Create `skills/your-skill-name/evals.json` using `docs/templates/evals-template.json` as a starting point. Minimum 3 test cases:
- 1 positive trigger test
- 1 negative trigger test (anti-trigger)
- 1 output quality test

### 6. Skill Quality Checklist

Before submitting your PR:

- [ ] YAML frontmatter parses without errors
- [ ] Description is specific and action-oriented (not "helps with X")
- [ ] "When to Use" section present
- [ ] At least one worked example
- [ ] No duplicate content with existing skills (>30% overlap = merge instead)
- [ ] No credentials, tokens, API keys, or secrets
- [ ] Under 500 lines (unless domain knowledge requires more)
- [ ] Cross-references to related skills where applicable

---

## Modifying Existing Skills

- **Technical/design/workflow skills:** Edit freely, submit PR
- **Domain knowledge skills** (csr-compliance-india, fcra-compliance, ngo-operations, etc.): These are proprietary. Edits require justification in the PR description — what changed in the law/regulation/practice that necessitates the update
- **mGrant product skills** (dris-mgrant-*): Coordinate with the product team before modifying

---

## Modifying Rules

Rules in `rules/` are always-enforced constraints. Changes have broad impact.

- **Frappe rules** (rules/frappe/): Ankit reviews all changes
- **Safety rules** (rules/safety/): Require explicit approval from both Ankit and PM

---

## Pull Request Process

### PR Title

```
feat(skills): add your-skill-name
```

### PR Description

```markdown
## Summary
What you're adding and why.

## Category
- [ ] Skill (domain / design / product / technical)
- [ ] Rule enhancement
- [ ] Template or guide
- [ ] Documentation

## Testing
How you tested this (e.g., "Loaded in Claude Code, ran 3 eval scenarios").

## Checklist
- [ ] Follows skill format from SKILL_TEMPLATE.md
- [ ] Tested with Claude Code
- [ ] No sensitive information
- [ ] No >30% overlap with existing skills
- [ ] Cross-references added where applicable
```

### Review

- All PRs are reviewed by **Ankit Jangir**
- Domain skill changes may also require PM review
- Feedback will be given within 48 hours
- Address feedback, then Ankit merges

---

## File Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Skill directory | lowercase-hyphenated | `csr-compliance-india/` |
| DRIS product skills | `dris-` prefix | `dris-feature-spec/` |
| mGrant skills | `dris-mgrant-` prefix | `dris-mgrant-donor/` |
| Skill file | Always `SKILL.md` | `skills/ux-reviewer/SKILL.md` |
| Eval file | Always `evals.json` | `skills/ux-reviewer/evals.json` |
| Templates | UPPERCASE or descriptive | `CLAUDE_TEMPLATE.md` |
| Guides | lowercase-hyphenated | `vibe-coding-how-to.md` |

---

## Questions?

Contact **Ankit Jangir** — repository maintainer.
