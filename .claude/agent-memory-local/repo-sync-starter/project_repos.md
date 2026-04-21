---
name: Testsigma AI Repositories
description: Three local repos synced to TestsigmaInc GitHub — paths, remotes, active branches, worktrees
type: project
---

Three Testsigma AI repos live under /Users/bharath.bhaktha/Documents/AI-Stuff/:

**alpha** — git@github.com:TestsigmaInc/alpha.git
- Main branch: main
- Has a linked worktree at /Users/bharath.bhaktha/.warp/worktrees/alpha/mica-equinox (branch: mica-equinox)
- Uses SSH remote (git@github.com), others use HTTPS
- Active development area: analyzer sprint work, atto hooks, autohealing
- Tags follow vMAJOR.MINOR.PATCH pattern (latest: v9.0.2 as of 2026-04-19)

**atto-browser-agent-v2** — https://github.com/TestsigmaInc/atto-browser-agent-v2.git
- Main branch: prod
- No tags fetched as of 2026-04-19

**sherlock** — https://github.com/TestsigmaInc/sherlock.git
- Default branch: main (Initial commit only)
- Active working branch: feat/sherlocks-palace
- Both branches tracked; feat/sherlocks-palace is the primary work branch

**Why:** User syncs all three at session start to ensure local state matches GitHub before beginning work.
**How to apply:** Always sync all three repos together. For sherlock, always check feat/sherlocks-palace in addition to main. For atto, always warn about dirty working tree before attempting pull.
