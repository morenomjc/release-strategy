---
marp: true
theme: default
paginate: true
style: |
  section {
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
    font-size: 1.4rem;
    padding: 2rem 3rem;
  }
  h1 {
    font-size: 2rem;
    color: #1a5276;
    border-bottom: 3px solid #1a5276;
    padding-bottom: 0.3rem;
  }
  h2 {
    font-size: 1.6rem;
    color: #1a5276;
  }
  code {
    background: #f4f6f7;
    padding: 0.1em 0.4em;
    border-radius: 3px;
    font-size: 0.95em;
  }
  pre {
    background: #f4f6f7;
    padding: 1rem;
    border-left: 4px solid #1a5276;
    font-size: 0.8rem;
  }
  table {
    width: 100%;
    font-size: 0.9rem;
  }
  th {
    background: #1a5276;
    color: white;
  }
  footer {
    font-size: 0.7rem;
    color: #aaa;
  }
---

<!-- _paginate: false -->
<!-- _footer: "" -->

# Version Bumps and Release Notes

### When commits are made, who makes them, and how tooling fits — in Git Flow and trunk-based development

&nbsp;

- Covers **both branching models** used on enterprise teams
- After this you will know exactly where the version number changes, where the changelog is authored, and which tool to reach for
- Audience: developers on enterprise teams; all changes to `main` go through PRs

---

# The Core Question

> "When does the version number actually change — and who commits it?"

The answer differs completely between Git Flow and TBD:

| | Git Flow | Trunk-Based Development |
|---|---|---|
| **Version bump timing** | On release branch cut, before stabilization | At tag time — via a bump PR or automated by CI |
| **Who commits the bump** | Release manager (or CI triggered by branch cut) | Developer (bump PR) or CI bot (release-please / semantic-release) |
| **Where the bump lives** | `release/x.y.z` branch | `main` — always via a merged PR |
| **Changelog authored** | On the release branch, after stabilization | Generated from trunk commits since last tag |
| **Changelog committed** | Release branch PR → `main` | `main` — via Release PR or CI bot PR |

<!--
SPEAKER NOTE: This table is the whole talk in one view. Everything else is the detail behind each cell. Direct pushes to main are not permitted on enterprise teams — every version bump and changelog commit reaches main through a PR.
-->

---

# Git Flow: Version Bump on Release Branch Cut

The bump commit is the **first commit on the release branch**, before any QA or stabilization work:

```bash
# 1. Cut the release branch from develop
git checkout develop && git pull origin develop
git checkout -b release/2.4.0

# 2. Bump the version file immediately
npm version 2.4.0 --no-git-tag-version
git add package.json
git commit -m "chore(release): bump version to 2.4.0"

# 3. Stabilization fixes go here — all carry the correct version identity
```

&nbsp;

- Every commit on the branch has a deterministic version — QA knows exactly what it is testing
- The release branch reaches `main` via a **PR** — no direct pushes to main permitted
- The bump commit travels with that PR

<!--
SPEAKER NOTE: The reason to bump immediately on cut is version identity. If you bump at the end of stabilization, commits made during QA have an ambiguous version. Bumping first means every commit, test result, and artifact on the branch is unambiguously 2.4.0.
-->

---

# Git Flow: Merge-Back Keeps Develop Current

After the release PR merges to `main` and is tagged, `develop` must catch up:

```bash
# After release/2.4.0 has merged to main and been tagged v2.4.0:
git checkout develop
git merge release/2.4.0

# Immediately bump develop to the next development cycle version
git add package.json
git commit -m "chore(release): bump develop to 2.4.1-SNAPSHOT"
```

&nbsp;

**Why this matters:** Without the merge-back, `develop` still carries `2.3.0`. The next release branch cut would start from the wrong version. A fix merged into `2.4.0` could silently regress in `2.5.0`.

---

# TBD: Version Bump via PR, Then Tag

No release branch cut moment — trunk is always releasable. The bump goes through a short-lived branch and PR:

```bash
# 1. Create a short-lived bump branch from main
git checkout main && git pull origin main
git checkout -b chore/bump-2.4.0

# 2. Bump the version file
npm version 2.4.0 --no-git-tag-version
git add package.json
git commit -m "chore(release): bump version to 2.4.0"

# 3. Open a PR targeting main — get it reviewed and merged

# 4. After merge, pull main and tag the merge commit
git checkout main && git pull origin main
git tag v2.4.0 && git push origin v2.4.0

# 5. CI picks up the tag and builds the release artifact
```

<!--
SPEAKER NOTE: The key constraint: no direct push to main. Even for a one-line version bump, the path is branch → PR → merge → tag. The tag is pushed directly (not the commit) — tags bypassing branch protection is a deliberate GitHub design choice.
-->

---

# TBD: Automated Bump with release-please

release-please eliminates the manual bump PR — the tool opens it automatically:

```yaml
# .github/workflows/release-please.yml
on:
  push:
    branches: [main]

permissions:
  contents: write
  pull-requests: write

jobs:
  release-please:
    runs-on: ubuntu-latest
    steps:
      - uses: google-github-actions/release-please-action@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          release-type: node
          manifest-file: .release-please-manifest.json
```

**Workflow:** commits land on `main` → release-please opens/updates a Release PR (`chore(main): release X.Y.Z`) with bumped version + CHANGELOG diff → team merges when ready → action tags and creates GitHub Release.

The bump commit is made by the tool, not a developer. The PR is still the human review gate.

---

# Release Notes: Git Flow

Changelog is authored **on the release branch**, after stabilization, before the PR to `main`:

```markdown
## [2.4.0] - 2026-04-25

### Added
- feat(auth): support PKCE flow for OAuth2 clients (#412)
- feat(api): add pagination to /users endpoint (#398)

### Fixed
- fix(db): prevent connection pool exhaustion under high load (#407)
- fix(auth): handle expired refresh tokens gracefully (#401)

### Breaking Changes
- feat!(config): remove deprecated `AUTH_SECRET` env var — use `AUTH_TOKEN_SECRET` (#389)
```

&nbsp;

The release manager inspects commits from the previous tag to HEAD, groups by type, writes or generates the entry. This CHANGELOG commit travels to `main` inside the release branch PR.

---

# Release Notes: Trunk-Based Development

Three paths — all generate notes from trunk commits since the last tag:

| Path | Mechanism | Human gate |
|---|---|---|
| **release-please** | Opens a Release PR with auto-generated CHANGELOG diff | Yes — team merges the PR when ready |
| **semantic-release** | Fully automated CI job post-merge: writes CHANGELOG, commits, tags, publishes | No — requires bot bypass exemption (see next slide) |
| **Manual** | `git log v2.3.0..HEAD`, write CHANGELOG entry, commit to short-lived branch, PR to `main`, then tag | Yes — normal PR review |

&nbsp;

- release-please: highest team visibility, lowest automation
- semantic-release: highest automation, requires configuration for enterprise branch protection
- Manual: full control, most effort

---

# semantic-release and Branch Protection

semantic-release's `@semantic-release/git` plugin **commits directly to the release branch** — which conflicts with enterprise branch protection requiring PRs.

**Two options for enterprise teams:**

**Option 1 — Grant the release bot a bypass exemption**
Add a dedicated `release-bot` GitHub App identity to the branch protection bypass list. The bot commits directly; all human pushes still require a PR.

**Option 2 — Remove `@semantic-release/git` from the plugin list**
Commit `CHANGELOG.md` in a pre-release CI step via a separate PR, then let semantic-release tag and publish from the already-committed state.

&nbsp;

Option 1 is simpler to configure. Option 2 keeps branch protection fully uniform at the cost of a more complex pipeline.

<!--
SPEAKER NOTE: This is the most common stumbling block when teams adopt semantic-release on enterprise repos. Surfacing it early avoids a pipeline that works in a test repo but silently fails against a protected main branch.
-->

---

# Conventional Commits: The Input All Tools Require

All three tools parse commit messages. The format:

```
<type>(<scope>): <description>

[optional body]

[optional footer(s)]
```

| Type | Version bump |
|---|---|
| `feat` | Minor (1.x.0 → 1.x+1.0) |
| `fix` | Patch (1.x.y → 1.x.y+1) |
| `feat!` or `BREAKING CHANGE:` footer | Major (x.0.0) |
| `chore`, `docs`, `refactor`, `test`, `perf` | None |

**Enforcement:** `commitlint` (`@commitlint/config-conventional`) in CI + git hooks. Adopt this before introducing any release tooling.

---

# git-cliff: Changelog Without the Automation

Rust-based changelog generator — reads git history, writes `CHANGELOG.md`. Does not bump versions or create tags.

```toml
# cliff.toml (minimal)
[git]
tag_pattern = "v[0-9].*"
commit_parsers = [
  { message = "^feat", group = "Added" },
  { message = "^fix", group = "Fixed" },
  { body = ".*BREAKING CHANGE", group = "Breaking Changes" },
  { message = ".*", skip = true },
]
```

```bash
# Generate changelog for the upcoming release
git cliff --tag 2.4.0 -o CHANGELOG.md

# Preview unreleased commits without writing
git cliff --unreleased
```

**Fit:** Git Flow — run on the release branch before the PR to `main`. TBD — run in the CI release job before tagging. Works anywhere because it only writes a file.

---

# Choosing Your Tooling Stack

| Decision | Recommended tool |
|---|---|
| "I want a PR to review before the release is cut" | **release-please** |
| "I want fully automated, zero-touch releases on every merge to main" | **semantic-release** — requires bot bypass exemption in branch protection |
| "I want to generate the changelog without automating the release" | **git-cliff** |
| "I'm using Git Flow with manual release management" | **git-cliff** + manual version bump on the release branch |
| "I want to enforce conventional commits in CI and locally" | **commitlint** (`@commitlint/config-conventional`) |

&nbsp;

**Adopt commitlint first.** All three changelog tools parse conventional commits. If your team does not write them consistently, the generated output will be incomplete or wrong regardless of tool choice.

---

<!-- _paginate: false -->

# What You Now Know

| Question | Git Flow | TBD |
|---|---|---|
| When is the version bumped? | On release branch cut, immediately | At tag time — via bump PR or automated |
| Who makes the bump commit? | Release manager or CI | Developer (PR) or CI bot |
| Where does the bump commit live? | `release/x.y.z` branch | `main` — always via a merged PR |
| When are release notes authored? | After stabilization, on release branch | Generated from trunk commits at release time |
| What tooling fits naturally? | git-cliff + commitlint | release-please or semantic-release + commitlint |

&nbsp;

- **No direct pushes to `main`** — every version bump and changelog commit reaches main through a PR, whether authored by a developer or a CI bot
- Adopt **commitlint first**, then layer on changelog/release tooling

&nbsp;

Reference doc: `docs/enterprise/version-and-release-notes.md`

<!--
SPEAKER NOTE: Leave time for questions. Common ones: (1) "Can we use semantic-release with Git Flow?" — yes, on main post-merge, but it has no awareness of release branches; use git-cliff on the release branch instead. (2) "Does the release-please PR count as a human review?" — yes, that is by design; the team explicitly merges when they are ready to ship. (3) "What if we already have a CHANGELOG.md format we like?" — git-cliff templates are fully customizable via Tera; you can match any existing format.
-->
