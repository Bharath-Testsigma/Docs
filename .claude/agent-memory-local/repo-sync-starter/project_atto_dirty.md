---
name: Atto browser agent dirty working tree
description: atto-browser-agent-v2 prod branch has local modifications that block git pull every sync session
type: project
---

The atto-browser-agent-v2 repo (branch: prod) consistently has a dirty working tree at sync time:
- README.md is modified (89 insertions, 7 deletions as of 2026-04-19 — appears to be in-progress documentation work)
- Untracked files under docs/: api-reference.md, environment-variables.md, local-setup.md

**Why:** These appear to be documentation files being drafted locally but not yet committed or pushed. They are not staged, so pull is blocked by default git safety rules.
**How to apply:** During every sync, skip the pull step for atto-browser-agent-v2 and flag this to the user. Suggest options: commit the docs changes, stash them, or discard them. Do not auto-resolve.
