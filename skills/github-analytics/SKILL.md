---
name: github-analytics
description: GitHub analytics patterns for directors and leads to track developer activity, PR velocity, repo health, code review metrics, and team productivity using GitHub API and CLI.
origin: DRIS
---

# GitHub Analytics for Directors & Leads

## When to Activate

- Directors reviewing developer productivity and repo health
- Tracking PR velocity and review turnaround
- Analyzing code review quality and participation
- Generating team activity reports
- Monitoring deployment frequency and lead time
- Understanding codebase contribution patterns

## Quick GitHub CLI Commands

### Developer Activity Overview

```bash
# PRs merged in last 7 days
gh pr list --state merged --search "merged:>$(date -d '7 days ago' +%Y-%m-%d)" \
  --json number,title,author,mergedAt,additions,deletions \
  --jq '.[] | "\(.number) | \(.author.login) | +\(.additions)/-\(.deletions) | \(.title)"'

# PRs currently open with review status
gh pr list --json number,title,author,reviewDecision,createdAt,labels \
  --jq '.[] | "\(.number) | \(.author.login) | \(.reviewDecision) | \(.title)"'

# Who has the most open PRs (potential bottleneck)
gh pr list --json author --jq '[.[] | .author.login] | group_by(.) | map({user: .[0], count: length}) | sort_by(.count) | reverse'

# Stale PRs (open > 7 days)
gh pr list --search "created:<$(date -d '7 days ago' +%Y-%m-%d)" \
  --json number,title,author,createdAt \
  --jq '.[] | "\(.number) | \(.author.login) | \(.createdAt[:10]) | \(.title)"'
```

### Code Review Metrics

```bash
# Review turnaround: time from PR open to first review
gh pr list --state merged --limit 20 \
  --json number,createdAt,reviews \
  --jq '.[] | select(.reviews | length > 0) |
    {pr: .number, created: .createdAt, first_review: .reviews[0].submittedAt}'

# Who reviews the most
gh pr list --state merged --limit 50 \
  --json reviews \
  --jq '[.[] | .reviews[] | .author.login] | group_by(.) | map({reviewer: .[0], count: length}) | sort_by(.count) | reverse'

# PRs merged without review (flag for process improvement)
gh pr list --state merged --limit 30 \
  --json number,title,author,reviewDecision \
  --jq '.[] | select(.reviewDecision != "APPROVED") | "\(.number) | \(.author.login) | \(.title)"'
```

### Repository Health

```bash
# Commit frequency by author (last 30 days)
gh api repos/{owner}/{repo}/stats/contributors \
  --jq '.[] | {author: .author.login, total_commits: .total, recent_weeks: [.weeks[-4:][] | .c] | add}'

# Open issues by label
gh issue list --json labels \
  --jq '[.[] | .labels[].name] | group_by(.) | map({label: .[0], count: length}) | sort_by(.count) | reverse'

# Release frequency
gh release list --limit 10 \
  --json tagName,publishedAt \
  --jq '.[] | "\(.tagName) | \(.publishedAt[:10])"'

# Branch count (detect stale branches)
gh api repos/{owner}/{repo}/branches --paginate \
  --jq '. | length' # Total branch count

# Stale branches (no commits in 30 days)
gh api repos/{owner}/{repo}/branches --paginate \
  --jq '.[] | select(.commit.sha != null) | .name'
```

### Team Productivity Dashboard Data

```bash
# Weekly summary script
echo "=== Weekly Dev Report ==="
echo ""
echo "## PRs Merged This Week"
gh pr list --state merged --search "merged:>$(date -d '7 days ago' +%Y-%m-%d)" \
  --json number,title,author,additions,deletions \
  --jq '.[] | "- #\(.number) by @\(.author.login): \(.title) (+\(.additions)/-\(.deletions))"'

echo ""
echo "## PRs In Review"
gh pr list --json number,title,author,reviewDecision \
  --jq '.[] | "- #\(.number) by @\(.author.login) [\(.reviewDecision // "PENDING")]: \(.title)"'

echo ""
echo "## Open Issues"
gh issue list --limit 10 --json number,title,assignees,labels \
  --jq '.[] | "- #\(.number): \(.title) [\(.assignees | map(.login) | join(", "))]"'
```

## Automated Reports with GitHub API

### Python Script for Director Dashboard

```python
#!/usr/bin/env python3
"""Generate weekly developer activity report for directors."""

import subprocess
import json
from datetime import datetime, timedelta
from collections import defaultdict

def gh_api(endpoint, method="GET"):
    """Call GitHub API via gh CLI."""
    result = subprocess.run(
        ["gh", "api", endpoint, "--method", method],
        capture_output=True, text=True
    )
    return json.loads(result.stdout) if result.stdout else []

def get_merged_prs(repo, days=7):
    """Get PRs merged in last N days."""
    since = (datetime.now() - timedelta(days=days)).strftime("%Y-%m-%d")
    result = subprocess.run(
        ["gh", "pr", "list", "--repo", repo, "--state", "merged",
         "--search", f"merged:>{since}",
         "--json", "number,title,author,mergedAt,additions,deletions,reviews"],
        capture_output=True, text=True
    )
    return json.loads(result.stdout) if result.stdout else []

def generate_report(repo):
    """Generate the weekly report."""
    prs = get_merged_prs(repo)

    # Aggregate by developer
    dev_stats = defaultdict(lambda: {"prs": 0, "additions": 0, "deletions": 0})
    for pr in prs:
        author = pr["author"]["login"]
        dev_stats[author]["prs"] += 1
        dev_stats[author]["additions"] += pr.get("additions", 0)
        dev_stats[author]["deletions"] += pr.get("deletions", 0)

    # Review stats
    review_stats = defaultdict(int)
    for pr in prs:
        for review in pr.get("reviews", []):
            review_stats[review["author"]["login"]] += 1

    report = {
        "period": f"Last 7 days ({datetime.now().strftime('%Y-%m-%d')})",
        "total_prs_merged": len(prs),
        "developer_activity": dict(dev_stats),
        "review_activity": dict(review_stats),
        "top_contributor": max(dev_stats, key=lambda x: dev_stats[x]["prs"]) if dev_stats else "N/A",
    }

    return report

if __name__ == "__main__":
    import sys
    repo = sys.argv[1] if len(sys.argv) > 1 else "dhwani-ris/main-app"
    report = generate_report(repo)
    print(json.dumps(report, indent=2))
```

### DORA Metrics Tracking

```bash
# Deployment Frequency: How often do we deploy?
gh release list --limit 30 --json publishedAt \
  --jq '[.[] | .publishedAt[:10]] | group_by(.)
        | map({date: .[0], count: length})
        | "Releases per week: \(length / 4 | floor)"'

# Lead Time for Changes: Time from commit to production
# (PR created → merged → deployed)
gh pr list --state merged --limit 20 \
  --json number,createdAt,mergedAt \
  --jq '.[] | {
    pr: .number,
    lead_time_hours: ((.mergedAt | fromdateiso8601) - (.createdAt | fromdateiso8601)) / 3600 | floor
  }'

# Change Failure Rate: How many deployments cause issues
gh issue list --label "bug,production" --state closed --limit 30 \
  --json number,createdAt,closedAt \
  --jq 'length as $bugs |
        "Production bugs in period: \($bugs)"'

# Mean Time to Recovery: How fast do we fix production issues
gh issue list --label "bug,production" --state closed --limit 20 \
  --json createdAt,closedAt \
  --jq '[.[] | ((.closedAt | fromdateiso8601) - (.createdAt | fromdateiso8601)) / 3600]
        | (add / length) | floor | "Avg recovery time: \(.) hours"'
```

## Tips for Directors

1. **Run reports weekly** — set up a GitHub Action to auto-generate and email
2. **Track trends, not absolutes** — lines of code and PR count are noisy; look at velocity trends
3. **Review turnaround is key** — slow reviews bottleneck the entire team
4. **Watch for bus factor** — if one person owns too much code, it's a risk
5. **PRs without reviews** — flag and fix process, especially for govt projects requiring audit trails
6. **Use GitHub Projects** — link issues to milestones for client delivery tracking
