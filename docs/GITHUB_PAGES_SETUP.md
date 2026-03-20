# GitHub Pages Setup — Skill Graph

## Steps (takes 2 minutes)

### 1. Merge PR #3
Standard merge to `main`.

### 2. Enable GitHub Pages
1. Go to **repo Settings** → **Pages** (left sidebar, under "Code and automation")
2. **Source**: select **Deploy from a branch**
3. **Branch**: select `main`
4. **Folder**: select `/docs`
5. Click **Save**

### 3. Wait 60 seconds
GitHub builds and deploys automatically. The green checkmark appears in the Actions tab.

### 4. Access the live site
URL: `https://doot-workspaces.github.io/dhwani-everything-claude-code/skill-graph.html`

That's it. The HTML file auto-fetches `SKILL_GRAPH.json` from the same directory. When anyone updates the JSON and pushes to `main`, the live graph updates on the next page load.

---

## How the live data works

The `skill-graph.html` fetches data in this priority:
1. `./SKILL_GRAPH.json` (same directory — GitHub Pages path)
2. `https://raw.githubusercontent.com/.../SKILL_GRAPH.json` (fallback if not on Pages)
3. Embedded static data (if both fetches fail)

A badge in the bottom-right corner shows the data source:
- **🟢 Live** — data loaded from SKILL_GRAPH.json
- **🟡 Embedded** — using static fallback (JSON fetch failed)

## Updating skills

When a new skill is added via PR:
1. Add the skill entry to `docs/SKILL_GRAPH.json` (in the `skills` object)
2. Add any edges to the `edges` array
3. Merge to main
4. GitHub Pages auto-deploys — the live graph picks it up on next page load

No build step. No npm. No CI config. Just push JSON changes.

## Cost

GitHub Pages free tier: 1GB storage, 100GB bandwidth/month. The entire `/docs` folder is under 500KB. Zero cost concern.
