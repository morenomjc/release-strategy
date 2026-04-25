# Spring Framework Git Branching Strategy

**Source:** spring-projects/spring-framework on GitHub
**Researched:** 2026-04-25
**Confidence:** HIGH — primary sources are the official GitHub wiki, CONTRIBUTING.md, live branch/commit/milestone data pulled directly from the GitHub API, and raw files from the repository.

---

## 1. Overview: What Model Is This?

Spring Framework uses a **maintenance-branch model** (sometimes called a "release train" model), not git-flow and not trunk-based development in the strict sense. The key characteristics:

- There is always a **`main` branch** that serves as the integration point and default branch.
- Active feature development for the *next* unreleased generation targets `main` directly (or a named minor branch that is eventually merged into `main`).
- Each *shipped* minor version line gets a **long-lived `N.N.x` maintenance branch** that lives for the duration of OSS support.
- Fixes flow **forward** (older branch -> newer branch -> main) via `git merge --no-ff`. Fixes never flow backward.
- Cross-version fixes are handled via **backports** using cherry-picks and a backport bot.

This is not git-flow: there are no feature branches, develop branches, or release branches in the git-flow sense. It is closer to the Linux kernel stable-branch model or the pattern used by many long-lived Apache projects.

---

## 2. Branch Inventory (as of 2026-04-25)

All branches currently present in the repository, fetched live from the GitHub API:

| Branch | Role | Last Commit | Protected | Snapshot Version |
|--------|------|-------------|-----------|-----------------|
| `main` | Next generation (7.1.x feature dev) | 2026-04-23 | YES | `7.1.0-SNAPSHOT` |
| `7.0.x` | Current GA production line | 2026-04-23 | No | `7.0.8-SNAPSHOT` |
| `6.2.x` | Previous generation, active maintenance | 2026-04-23 | No | `6.2.19-SNAPSHOT` |
| `6.1.x` | Older maintenance, winding down | 2025-06-12 | YES | `6.1.22-SNAPSHOT` |
| `6.0.x` | EOL / archived | 2024-08-14 | YES | — |
| `5.3.x` | EOL / archived | 2024-08-14 | YES | — |
| `5.2.x` | EOL / archived | 2023-07-13 | YES | — |
| `5.1.x` | EOL | (older) | — | — |
| `5.0.x` | EOL | (older) | — | — |
| `4.3.x` | EOL | 2020-12-09 | YES | — |
| `4.2.x` — `3.0.x` | EOL / historical | (old) | — | — |
| `docs-build` | CI artifact branch for docs | — | — | — |
| `gh-pages` | GitHub Pages | — | — | — |
| `dependabot/*` | Automated dependency update PRs | — | — | — |

**Pattern:** Every released minor version (e.g., 6.1, 6.2, 7.0) gets its own `N.N.x` branch. The `.x` suffix is literal — it represents the patch dimension. There is no `7.0.1` or `7.0.2` branch; patch releases are tags on the `7.0.x` branch.

---

## 3. The Role of `main`

`main` is **not** the current stable release. It is the integration branch for the *next* upcoming generation. From CONTRIBUTING.md (direct quote):

> "Always check out the `main` branch and submit pull requests against it (for target version see settings.gradle). Backports to prior versions will be considered on a case-by-case basis and reflected as the fix version in the issue tracker."

At the time of research:
- `gradle.properties` on `main` shows `version=7.1.0-SNAPSHOT`
- `main` is the default/protected branch
- Commits on `main` frequently show `Merge branch '7.0.x'` — confirming the forward-merge flow

**The dual role of `main`:** When there is no upcoming major/minor release in progress, `main` IS the current branch (the current GA line's integration branch). When a new generation is being developed, `main` leads ahead of the current GA branch. From the wiki:

> "the Git main branch - this can be the current branch, or a branch dedicated to the next major/minor release"

---

## 4. Concurrent Release Trains: How It Works Right Now

As of April 2026, three lines are active simultaneously:

```
main (7.1.0-SNAPSHOT)        <- next generation feature dev
  ^
  | merge --no-ff
  |
7.0.x (7.0.8-SNAPSHOT)       <- current GA production line
  ^
  | cherry-pick (backport)
  |
6.2.x (6.2.19-SNAPSHOT)      <- previous generation, still in OSS support
  ^
  | cherry-pick (backport)
  |
6.1.x (6.1.22-SNAPSHOT)      <- winding down, last activity June 2025
```

**Forward merge flow (fixes that apply to all active lines):**
1. Fix is committed to the oldest affected branch (e.g., `6.2.x`)
2. That branch is merged forward into the next branch (`7.0.x`) with `git merge --no-ff`
3. `7.0.x` is merged forward into `main`

From the official wiki (direct quote from "Git branch management" page):

> "First, we need to fix the bug and merge it forward if necessary. If the current branch is the main / default branch, pushing changes to the main branch is enough and there is no need for forward merges."

Example git commands shown in the wiki for the forward-merge flow:
```bash
# Fix on the maintenance branch
git checkout 6.0.x && git pull
git add . && git commit
git push origin 6.0.x

# Merge forward into main
git checkout main && git pull
git merge --no-ff 6.0.x
```

**Backport flow (fixes that apply only to older lines):**
The team uses a **backport-bot**. Labels like `for: backport-to-5.3.x` trigger the bot to create a backport issue. A maintainer then cherry-picks manually:

```bash
git checkout 5.3.x
git cherry-pick c0ffee456 --edit
# Closes gh-3456 (backport issue)
# See gh-1234 (original issue)
git push origin 5.3.x
```

**Evidence of this pattern in live commit log:**
Recent commits on `main` (sampled via GitHub API, April 2026):
```
2026-04-23 | Merge branch '7.0.x'
2026-04-21 | Merge branch '7.0.x'
2026-04-21 | Merge branch '7.0.x'
2026-04-17 | Merge branch '7.0.x'
```
The same fix commits (e.g., "Warn against unsafe static resource locations", "Add doOnDiscard in MultipartHttpMessageReader") appear in the same order on `6.2.x`, `7.0.x`, and `main` — confirming commits propagate upward.

---

## 5. Branch Naming Convention

The convention is simple and consistent across all generations:

| Pattern | Meaning |
|---------|---------|
| `main` | Next unreleased generation (or current if no next is in development) |
| `N.N.x` | Maintenance branch for minor version N.N (e.g., `6.2.x`, `7.0.x`) |

The `.x` in `6.2.x` is a literal convention, not a glob — it signals "all patch releases of 6.2". There are no `release/N.N.x` prefixes, no `hotfix/` branches, no `develop` branch.

Tag naming (for actual releases) follows: `vN.N.N` (e.g., `v7.0.7`, `v6.2.18`) and `vN.N.N-M1`, `vN.N.N-RC1` for pre-releases.

---

## 6. PR Submission Target

Per CONTRIBUTING.md, contributors **always** submit PRs against `main`:

> "Always check out the `main` branch and submit pull requests against it"

This is confirmed by the live PR data — the overwhelming majority of closed PRs target `main`. A small number target `7.0.x` or `6.1.x` directly (typically from maintainers doing targeted fixes). Backports to maintenance branches are handled by the team after merging to main, not by contributors.

---

## 7. Branch Lifetime and Support Policy

From the **Spring Framework Versions** wiki page (last edited Feb 5, 2026 by Juergen Hoeller):

> "7.0.x is the start of a new framework generation and the current production line (November 2025), to be followed up by the 7.1.x feature branch (November 2026)."
>
> "6.2.x is the final feature branch of the 6th generation. Open source support ends in June 2026; commercial long-term support options are available."
>
> "5.3.x was the final feature branch of the 5th generation. Open source support ended in August 2024; commercial long-term support options remain available."

Support tiers in practice:

| Status | Description | Branch Activity |
|--------|-------------|-----------------|
| **Active development** | Next unreleased generation | `main` — receives new features, forward-merged into from current branch |
| **Current GA** | Latest released generation, fully active | `7.0.x` — bugfixes, forward-merged into `main` frequently |
| **Active maintenance** | Previous generation, OSS support | `6.2.x` — bugfixes, backports from `7.0.x` fixes; forward-merged into `7.0.x` |
| **Winding down** | Support ending soon or just ended | `6.1.x` — last OSS commit June 2025; branch still exists but frozen |
| **EOL / archived** | Support ended, branch preserved read-only | `6.0.x`, `5.3.x`, `5.2.x`, `4.3.x`, etc. — no new commits |

**Lifecycle of a branch:**
1. Branch is cut from `main` when a new minor release is being stabilized (e.g., `6.2.x` cut when 6.2 went to RC)
2. Branch is active for the OSS support window (typically ~1-2 years after GA)
3. Commercial long-term support (LTS) is available beyond OSS support (via VMware/Broadcom Tanzu)
4. Branch is never deleted — historical branches remain permanently in the repo

**Release cadence (from milestone data):**
Patch releases ship approximately monthly, synchronized across active lines:
```
2026-04-17 | 7.0.7 (closed) | 6.2.18 (closed)
2026-03-13 | 7.0.6 (closed) | 6.2.17 (closed)
2026-02-12 | 7.0.4 (closed) | 6.2.16 (closed)
2026-01-15 | 7.0.3 (closed)
2025-12-11 | 7.0.2 (closed) | 6.2.15 (closed)
```
Milestones for the next generation are labeled `N.N.0-M1`, `N.N.0-M2`, ..., `N.N.0-RC1`, `N.N.0-RC2`, `N.N.0`.

---

## 8. Key Structural Observations

**No feature branches in the public repo.** Contributors fork the repo and work in their own fork. All public branches are either long-lived maintenance branches or infrastructure branches (`docs-build`, `gh-pages`).

**`main` is the PR target, not the release branch.** Releases come from the `N.N.x` branches, not from `main`. The `main` branch is always ahead of the latest release.

**Protection asymmetry.** Older EOL branches are GitHub-protected (no direct pushes), while active branches like `7.0.x` and `6.2.x` are not protected (maintainers push directly). `main` is protected, presumably requiring review.

**Snapshot versions mark intent.** The `version` property in `gradle.properties` always carries the next release version as a `-SNAPSHOT`. This provides a definitive way to know what a branch is building toward:
- `main` at `7.1.0-SNAPSHOT` = next minor version in development
- `7.0.x` at `7.0.8-SNAPSHOT` = next patch of the current GA line
- `6.2.x` at `6.2.19-SNAPSHOT` = next patch of the maintenance line

---

## 9. Sources

All sources accessed 2026-04-25:

- **CONTRIBUTING.md (main branch):** https://raw.githubusercontent.com/spring-projects/spring-framework/main/CONTRIBUTING.md — HIGH confidence, official canonical source
- **Git branch management wiki:** https://github.com/spring-projects/spring-framework/wiki/Git-branch-management (last edited Jan 11, 2023 by Simon Baslé) — HIGH confidence, official team documentation
- **Spring Framework Versions wiki:** https://github.com/spring-projects/spring-framework/wiki/Spring-Framework-Versions (last edited Feb 5, 2026 by Juergen Hoeller) — HIGH confidence, official support policy
- **GitHub API — branches:** `GET /repos/spring-projects/spring-framework/branches` — HIGH confidence, live data
- **GitHub API — commits:** sampled from `main`, `7.0.x`, `6.2.x`, `6.1.x` — HIGH confidence, live data
- **GitHub API — milestones:** `GET /repos/spring-projects/spring-framework/milestones` — HIGH confidence, live data
- **gradle.properties across branches** — HIGH confidence, live file contents
