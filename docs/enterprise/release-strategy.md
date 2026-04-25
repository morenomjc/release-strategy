# Enterprise Release Branching: Git Flow and Trunk-Based Development

**Audience:** Mid-level developers on enterprise teams currently using Git Flow, migrating toward trunk-based development
**Last updated:** 2026-04-25

---

## Quick Reference

A side-by-side comparison of the two models across the dimensions that matter most for day-to-day engineering:

| Dimension | Git Flow | Trunk-Based Development |
|---|---|---|
| **Stable branch** | `main` — production-tagged only; `develop` — integrated but unreleased | `main` (trunk) is always releasable; no separate stable branch |
| **Parallel features** | One long-lived branch per feature, isolated until completion | Short-lived branches (1-2 days max); features gated by flags, not branches |
| **Release process** | Branch from `develop` → stabilize on `release/x.y.z` → merge to `main` + tag → merge back | Cut `release/x.y.z` from trunk days before shipping; fixes go to trunk first, cherry-pick to release |
| **Hotfix process** | Branch from `main` → fix → merge to `main` + tag → merge back to `develop` | Fix on trunk; cherry-pick to active release branch if needed |
| **CI requirements** | Useful but optional; teams can merge without CI gates | Non-negotiable; CI must be fast (< 10 min), reliable (< 2% flaky), and block merges when red |
| **Feature flags needed?** | No — isolation is provided by the branch itself | Yes — flags are the primary mechanism for separating code delivery from feature exposure |
| **Merge ceremony** | High — four distinct merge choreographies with bidirectional flows | Low — one direction: short branch → trunk → optional release branch cherry-pick |
| **Deployment frequency** | Limited by release branch stabilization cycles (days to weeks) | On demand — trunk is always deployable |
| **DORA profile** | Common in medium/low performers | 2.3x more likely in elite performers (DORA 2024) |

---

## Table of Contents

1. [Git Flow](#1-git-flow)
2. [Trunk-Based Development](#2-trunk-based-development)
3. [Feature Flags](#3-feature-flags)
4. [Migrating from Git Flow to Trunk](#4-migrating-from-git-flow-to-trunk)
5. [Decision Guide](#5-decision-guide)

---

## 1. Git Flow

Git Flow is a branching model created by Vincent Driessen in 2010. It was designed for teams shipping versioned, packaged software on a release cadence of weeks to months. Understanding it precisely — including where it works, and where it breaks — is necessary before any migration decision.

### 1.1 Branch Structure and Roles

Git Flow defines five branch types with explicit, non-negotiable roles:

```
main              ← production-only; every commit is a release tag
develop           ← integration branch; "next release" in progress
feature/ABC-123   ← individual feature work; branches off develop
release/2.3.0     ← release stabilization; branches off develop
hotfix/fix-null   ← emergency fix on production; branches off main
```

**Permanent branches:** `main` and `develop` are never deleted. They exist for the life of the repository.

**Temporary branches:** `feature/*`, `release/*`, and `hotfix/*` are created for a specific purpose and deleted when they merge.

The key structural rule is that `main` and `develop` serve different contracts:

- `main` contains only code that has been released and tagged. Every commit on `main` corresponds to a version tag. If you check out `main`, you are on released code.
- `develop` is the accumulation surface for all work targeting the next release. It is integrated (tests pass, it compiles) but may contain bugs, half-finished features, or code that has not been reviewed for production readiness.

The separation means there is always a clear answer to "what is in production right now" (main) and "what will be in the next release" (develop). That clarity is Git Flow's primary value proposition.

### 1.2 Parallel Feature Development

Git Flow handles parallel features by giving each feature its own branch off `develop`. Features are completely isolated from each other until they are finished and merged back.

```bash
# Developer A starts feature in week 1
git checkout develop
git pull origin develop
git checkout -b feature/user-auth
# ... works for 2-3 weeks ...

# Developer B starts a different feature in the same week
git checkout develop
git pull origin develop
git checkout -b feature/payment-redesign
# ... also works for 2-3 weeks ...

# Both branches are active simultaneously, with no knowledge of each other
```

This model gives developers a clean working environment: Developer A cannot break Developer B's work, and neither branch sees the other's intermediate commits. The isolation is real and valuable during development.

The problem is that this isolation does not eliminate conflict — it defers it. When both `feature/user-auth` and `feature/payment-redesign` branch off `develop` on day 1 and both touch `src/app/router.ts`, neither developer knows about the other's changes until merge day. At that point, the conflicts are resolved by someone who may not have written the original code, under time pressure, without the context that would have made the conflict trivial to resolve three weeks earlier.

With ten concurrent features, this is not a manageable problem. It is a structural one.

### 1.3 What "Stable" Means in Git Flow

Git Flow uses a layered stability model where each branch type has a distinct contract:

| Branch | Stability Contract |
|--------|-------------------|
| `main` | Always production-deployable. Only released, tagged code. If you're here, you can ship it. |
| `release/2.3.0` | Intended to be stable for shipping. Only bug fixes are allowed — no new features after the cut. |
| `develop` | Integration-stable — compiles, tests pass. May contain unreleased bugs and unfinished work. |
| `feature/*` | Developer-stable — may be broken, work in progress, not expected to pass CI at all times. |
| `hotfix/*` | Surgical-stable — must be minimal and immediately production-ready. |

The promise is: if you deploy from `main`, you are deploying tested, reviewed, released code. The challenge is that the distance between `develop` and `main` can represent weeks or months of accumulated work. "Stable" on `develop` is a loose guarantee.

### 1.4 The Release Process: Cut, Stabilize, Tag

The Git Flow release lifecycle has three phases and four merge operations. Understanding all of them is necessary to understand where the failure modes come from.

**Phase 1: Cut the release branch**

When `develop` contains the features intended for a release, you create the release branch:

```bash
git checkout develop
git pull origin develop
git checkout -b release/2.3.0
# Bump the version number and update the changelog
git commit -am "Bump version to 2.3.0"
git push origin release/2.3.0
```

At this point, `develop` is open for the next release's feature work immediately. Teams can start `feature/2.4.0-things` on `develop` while `release/2.3.0` stabilizes. This is a genuine advantage of Git Flow's model — the cut decouples the release cycle from feature development.

**Phase 2: Stabilize**

QA runs against `release/2.3.0`. Bug fixes discovered during stabilization are committed directly to the release branch — not to `develop` first. Version bumps, changelog finalization, release notes, and any last configuration changes happen here.

```bash
# Fix found during QA
git checkout release/2.3.0
git checkout -b fix/login-redirect-bug   # some teams branch off release; others commit directly
# ... fix the bug ...
git checkout release/2.3.0
git merge fix/login-redirect-bug
git push origin release/2.3.0
```

The critical discipline here: every fix on `release/2.3.0` must also be applied to `develop`. This is a manual requirement. Git Flow does not enforce it automatically.

**Phase 3: Tag and merge**

When QA signs off, the release ships via two merges:

```bash
# Merge to main and tag
git checkout main
git pull origin main
git merge --no-ff release/2.3.0
git tag -a v2.3.0 -m "Release 2.3.0"
git push origin main
git push origin v2.3.0

# CRITICAL: merge release branch back to develop
# This captures all stabilization fixes made during Phase 2
git checkout develop
git pull origin develop
git merge --no-ff release/2.3.0
git push origin develop

# Clean up
git branch -d release/2.3.0
git push origin --delete release/2.3.0
```

The `--no-ff` flag (no fast-forward) is conventional in Git Flow. It creates a merge commit even when a fast-forward would be possible, which preserves the branch history in the DAG. Some teams use squash merges on the `develop` side to keep the history linear; the tradeoff is losing individual commit attribution.

### 1.5 The Hotfix Process

Hotfixes are the emergency path: a critical production bug must be fixed immediately, without going through the normal develop → release cycle.

```bash
# Branch from main (the released code), not develop
git checkout main
git pull origin main
git checkout -b hotfix/fix-null-pointer

# Fix and verify
# ... fix the bug, add test, verify locally ...

# Ship to production
git checkout main
git merge --no-ff hotfix/fix-null-pointer
git tag -a v2.2.1 -m "Hotfix 2.2.1 — fix null pointer in payment service"
git push origin main
git push origin v2.2.1

# CRITICAL: also merge to develop
git checkout develop
git merge --no-ff hotfix/fix-null-pointer
git push origin develop

# And if a release branch is currently active, apply there too
git checkout release/2.3.0
git merge --no-ff hotfix/fix-null-pointer
git push origin release/2.3.0

git branch -d hotfix/fix-null-pointer
git push origin --delete hotfix/fix-null-pointer
```

The pattern here is the same as the release merge-back: the fix must flow to both `main` and `develop`. If `release/2.3.0` is currently in stabilization, it needs the fix too. Three merge targets for one emergency fix.

### 1.6 Pain Points at Scale

These are the failure modes that drive teams off Git Flow. They are not theoretical — they are endemic at the 20+ developer scale. Any team that has lived with Git Flow for more than a year will recognize at least three of these.

**Integration hell (the most common)**

Long-lived feature branches accumulate divergence silently. Feature A and Feature B both branch off `develop` on sprint day 1 and run in parallel for three weeks. During those three weeks, they have zero awareness of each other's changes. When both merge at sprint end, the conflicts are resolved in isolation, by people who may not have written the original code.

The conflict rate is non-linear. With three branch types (feature, release, hotfix) all merging into `develop`, there are roughly three times the standard merge opportunities for conflict. At ten concurrent features, the tenth merge to land is resolving conflicts introduced by the previous nine. Teams at this scale routinely spend an entire day — sometimes two — merging at the end of a sprint.

The root cause is not that the code is incompatible. It is that the incompatibility was knowable weeks earlier, when the two branches were both touching the same files, but nobody knew to look.

**The "frozen develop" problem**

When a release branch is being stabilized, teams often informally freeze `develop` to avoid contaminating the release. Product managers stop approving merges. Developers hold their PRs. This creates dead time across the entire team.

This is not a Git Flow rule — it is a human response to complexity. But it is so common that it might as well be a rule. Git Flow's separation of release from develop is meant to prevent this, but the practical reality is that teams don't trust the model enough to keep developing while a release is in flight.

**The forgotten merge-back**

When a bug is fixed on `release/2.3.0` during stabilization, it must also be applied to `develop`. When a hotfix lands on `main`, it must also land on `develop`. These are mandatory steps, but they are manual, easy to forget under deadline pressure, and completely silent when they are missed.

The failure mode: the bug is fixed in v2.3.0. The release ships. The team moves on. Six weeks later, v2.4.0 is in QA and the same bug is reported. A postmortem traces back to the forgotten merge-back from the v2.3.0 release branch. Many teams have an incident report that reads exactly like this.

**Incompatibility with continuous delivery**

Git Flow assumes a release cadence of weeks to months. The model has no concept of deploying from `develop` to production — that is explicitly outside the model. Teams that want to deploy more than once a week find Git Flow's branch overhead directly in the way. Every deployment requires a branch, a merge ceremony, a tag, and a reverse merge. The overhead compounds at high frequency.

Thoughtworks assessed Git Flow as a "Hold" on their Technology Radar: "long-lived branches are the opposite of continuously integrating all changes." This is not a stylistic preference — it is a causal relationship.

**Microservices coordination failure**

In a microservices architecture with multiple repositories, each repo has its own Git Flow state. Deploying a feature that spans three services requires tracking three feature branches, three develop branches, three potential release branches, and their relative synchronization. There is no mechanism in Git Flow to coordinate this. Teams end up with YAML manifests or spreadsheets that track which version of Service A is compatible with which version of Service B. This problem scales exponentially with service count.

---

## 2. Trunk-Based Development

Trunk-based development (TBD) is the branching model used by high-performing engineering organizations. The 2024 DORA State of DevOps Report found that elite performers are 2.3x more likely to use trunk-based development. It is not the only factor separating elite from average performers, but it is consistently present in the elite cohort and consistently absent from the rest.

### 2.1 The Core Rule

The single defining rule of TBD: **every developer integrates their work to trunk (main) at least once per day.**

Everything else — short-lived branches, feature flags, CI gates, code review SLAs — is infrastructure that makes this rule safe to follow. The rule exists because the word "integration" in "continuous integration" is literal. Integrating means merging to a shared branch, not just running tests on an isolated branch.

Trunk is not `develop`. In Git Flow, `develop` accumulates unreleased work indefinitely, with no contractual guarantee that it could be deployed. In TBD, trunk is always in a state where it *could* be deployed right now. This distinction is absolute and is the load-bearing difference between the two models.

### 2.2 Short-Lived Feature Branches

TBD does not require every developer to commit directly to trunk on every change. Most TBD teams use short-lived branches as a code review artifact. The constraint is on lifetime, not on existence.

```bash
# Start work — branch off trunk
git checkout main
git pull --rebase origin main
git checkout -b feat/invoice-preview

# Commit incrementally
git add src/invoice/InvoicePreview.tsx
git commit -m "Add InvoicePreview component skeleton"

git add src/invoice/InvoicePreview.test.tsx
git commit -m "Add tests for InvoicePreview"

# Before opening a PR: rebase onto current trunk
git fetch origin main
git rebase origin/main

# Open PR — small, reviewable, targeted
# Get review same day
# Merge to main
# Branch is deleted immediately after merge
```

The branch serves one purpose: to hold work while it is reviewed. It is not a development isolation unit. It is a code-review artifact.

The lifetime limit — 1 to 2 days maximum — is what distinguishes a TBD short-lived branch from a Git Flow feature branch. At 2 days, the branch has not had time to accumulate significant divergence from trunk. The PR is small enough to be reviewed in one sitting. The merge is almost always clean.

**Contrast with Git Flow feature branches:**

| Property | Git Flow Feature Branch | TBD Feature Branch |
|----------|------------------------|-------------------|
| Maximum lifetime | Weeks to months | 1-2 days |
| Merges into | `develop` | `main` (trunk) |
| Number of developers | Multiple (team-owned feature) | One (or one pair) |
| Contains | A complete feature | One incremental slice |
| Active count at once | One per feature | One per developer |
| Primary purpose | Development isolation | Code review staging |

### 2.3 Parallel Features Without Long-Lived Branches

This is the question that trips up most teams migrating from Git Flow: "We have a feature that takes six weeks to build. How do you work on it without a six-week branch?"

The answer is that TBD reframes the question. The question is not "how do you protect a long-running feature from trunk?" The question is "how do you break a six-week feature into two-day increments that can each land on trunk safely?"

There are three mechanisms.

**Mechanism 1: Vertical decomposition**

Instead of building the entire feature in isolation and shipping it whole, decompose it into trunk-safe increments. Each increment is a unit of work that, by itself, does not break anything — either because it has no visible effect yet (a new API endpoint with no consumer, a database migration with no application code using the new columns), or because it is hidden behind a flag.

A six-week feature might decompose as:

```
Week 1, Day 1-2:  Add DB schema migration (new tables, backward-compatible)
Week 1, Day 3-4:  Add repository layer, unit tested
Week 2, Day 1-2:  Add service layer behind feature flag
Week 2, Day 3-4:  Add API endpoint, no UI consumer yet
Week 3, Day 1-2:  Add UI component, not yet wired to routing
Week 3, Day 3-4:  Wire component to routing behind flag
Week 4+:          Iterative refinement, all behind flag
Final day:        Enable flag for internal users → gradual rollout
```

Every one of these increments merges to trunk within two days. None of them breaks the build. None of them exposes users to an incomplete experience. The feature is complete when all increments are in trunk and the flag is turned on.

**Mechanism 2: Feature flags**

Code that is not ready for users is merged to trunk behind a flag. The flag gates exposure, not existence. The code compiles. Tests run. CI passes. The flag simply prevents users from reaching that code path.

```typescript
// This is trunk-safe. It can be merged today.
// Users see current behavior. The new path is tested but unreachable.
async function renderDashboard(user: User): Promise<DashboardView> {
  if (await featureFlags.isEnabled('new-analytics-dashboard', user)) {
    return renderNewDashboard(user);   // in-progress work, fully tested
  }
  return renderLegacyDashboard(user); // current production behavior, unchanged
}
```

Feature flags are covered in depth in Section 3. For the purposes of understanding parallel features: a flag is what allows six developers to each be working on six different features simultaneously, all of them landing on trunk daily, none of them interfering with any user in production.

**Mechanism 3: Branch by abstraction**

When replacing a core component — migrating from one ORM to another, replacing a HTTP client library, refactoring a shared service — the change cannot be decomposed into small increments because it is a single, interconnected structural change. For these cases, TBD uses the Branch by Abstraction technique.

The steps:

1. Create an interface (or abstract class) that both the old and new implementations satisfy.
2. Migrate all callers to use the interface, incrementally, over multiple trunk commits.
3. Build the new implementation behind the interface — in parallel with ongoing trunk activity.
4. Switch to the new implementation with a single flag or configuration change.
5. Delete the old implementation and remove the abstraction layer.

```typescript
// Step 1: Define the abstraction
interface UserRepository {
  findById(id: string): Promise<User | null>;
  save(user: User): Promise<User>;
  delete(id: string): Promise<void>;
}

// Existing implementation satisfies the interface (Step 1 continued)
class TypeOrmUserRepository implements UserRepository {
  // ... existing TypeORM implementation ...
}

// Step 3: Build new implementation alongside old one (separate trunk commits)
class PrismaUserRepository implements UserRepository {
  // ... new Prisma implementation ...
}

// Step 4: Toggle between implementations via config (one line change to ship)
const userRepository: UserRepository =
  config.useNewORM
    ? new PrismaUserRepository(prismaClient)
    : new TypeOrmUserRepository(dataSource);
```

The entire migration — including building the new implementation, migrating all callers, and testing — happens in small trunk commits over days or weeks. At no point is there a multi-week branch diverging from trunk. At no point is the system in an unrunnable state. The switch is a one-line configuration change.

### 2.4 What "Stable" Means in TBD

In TBD, there is one stability concept: trunk is always releasable. There is no secondary tier. There is no "mostly stable" branch.

This invariant is enforced by three mechanisms working in combination:

**Pre-merge CI gate.** All tests must pass before a PR can merge to trunk. Branch protection rules make this automatic and non-bypassable.

**Fast build cycle.** CI must be fast enough that a failure is caught and fixed within the same working session. A 45-minute CI pipeline that fails halfway through produces a developer who has moved on to something else before the result arrives. Fast CI means fast feedback, which means fast fixes.

**Stop the line.** When a commit reaches trunk and CI fails, it is treated as a P0 — not a "someone will fix it later" issue. Every developer on the team is effectively blocked from merging until trunk is green again. This is not punitive; it is structural. The CI gate means broken trunk immediately affects everyone, which means everyone has an incentive to fix it immediately.

Compared to Git Flow's "develop is usually stable," TBD's "trunk is always releasable" is a stronger and easier invariant to maintain precisely because it is binary and automated. Either CI is green and trunk is releasable, or CI is red and the team is actively fixing it. There is no ambiguous middle state.

### 2.5 Release Branching in TBD

TBD teams have two release models depending on their confidence level and deployment infrastructure.

**Model A: Release from trunk directly**

No release branches. Trunk is always deployable, so the release is the act of deploying trunk. When a release is needed, take the current trunk SHA and deploy it. If a post-release bug is found, fix it on trunk (subject to the normal PR process) and deploy the new trunk. This is the "fix forward" model.

Requirements for Model A:
- Very fast CI (so fixes can be deployed within minutes, not hours)
- Comprehensive monitoring and alerting (you detect bugs quickly)
- Feature flags to disable anything not ready for users
- High team trust in the trunk stability invariant

This is the model used by organizations shipping web services with continuous deployment. GitHub, Netflix, and Google all operate variants of this model.

**Model B: Cut a release branch from trunk**

For enterprise teams, regulated environments, or products with external customers who cannot be exposed to trunk directly, release branches remain appropriate. The key difference from Git Flow is how the branch is used.

```bash
# Cut the release branch just-in-time — days before shipping, not weeks
git checkout main
git pull --rebase origin main
git checkout -b release/2.3.0
git push origin release/2.3.0
```

The rules that govern TBD release branches are different from Git Flow's:

**Cut late, not early.** Git Flow cuts the release branch weeks before shipping to allow stabilization. TBD cuts it days before, because trunk is already stable. The branch is a deployment artifact, not a stabilization arena.

**No feature development on the release branch.** If a feature does not make the cut, it stays on trunk behind its flag. It will ship in the next release. There is no "just one more thing" on a release branch.

**Fixes go to trunk first, then cherry-pick.** This is the most important rule. If a bug is found on the release branch during final validation, the fix is written and merged on trunk through the normal PR process, and then cherry-picked to the release branch.

```bash
# 1. Fix on trunk (normal PR process)
git checkout main
git checkout -b fix/invoice-tax-rounding
# ... write the fix, tests pass ...
git checkout main
git merge fix/invoice-tax-rounding
git push origin main
FIX_COMMIT=$(git rev-parse --short HEAD)

# 2. Cherry-pick to release branch
git checkout release/2.3.0
git cherry-pick $FIX_COMMIT
git push origin release/2.3.0
```

**Never fix on the release branch and merge back.** This is the rule that prevents the "forgotten merge-back" failure mode from Git Flow. In Git Flow, fixes on `release/2.3.0` must be manually merged back to `develop`. If that step is forgotten, the fix is lost. In TBD, fixes always start on trunk. The release branch only ever receives cherry-picks. There is nothing to "merge back" because the fix is already on trunk.

**Delete the release branch after shipping.** Release branches are not permanent. Once the release is shipped and the tag is created, the branch is deleted. The tag is the permanent record; the branch is the delivery mechanism.

### 2.6 CI/CD Requirements

TBD is not a choice you can make in isolation. It is a system property that depends on CI/CD infrastructure that must be in place before the branching model change is made. Teams that attempt TBD without the infrastructure will observe trunk instability that erodes trust and causes reversion to Git Flow.

**The CI pipeline must have all of these properties:**

**Fast.** The target is under 10 minutes end-to-end for most changes. Under 20 minutes is acceptable. Beyond 30 minutes, developers stop waiting and start committing in batches, which defeats continuous integration. Use parallel test runners, test sharding, incremental builds (Bazel, Nx, Turborepo), and selective test execution based on changed files.

**Reliable.** Flaky tests — tests that pass and fail non-deterministically — are fatal to TBD. A flaky test creates false alarms on trunk, which trains developers to ignore CI failures, which means real failures get ignored too. Target less than 2% flaky test rate. Eliminate flaky tests aggressively; they are more expensive than they appear.

**Authoritative.** The CI result on a PR must be the final word on whether the code can merge. No manual overrides, no "CI is red but it's a known flaky test, just approve it." If CI is not authoritative, branch protection has no teeth.

**Staged correctly.** Run fast checks first so developers get early signal:

```
Stage 1: Compile / type check        (fast — fail fast if code does not compile)
Stage 2: Unit tests                  (fast — target < 5 minutes)
Stage 3: Integration tests           (medium — can run in parallel with Stage 2)
Stage 4: E2E / functional tests      (slower — run in parallel, can be selective)
Stage 5: Static analysis, linting    (fast — can run in parallel with Stage 2)
Stage 6: Security scans              (medium — SAST, dependency vulnerability scan)
```

**Branch protection rules (GitHub example):**

```
Branch: main
  Require a pull request before merging: YES
    Required approvals: 1 (minimum)
    Dismiss stale pull request approvals when new commits are pushed: YES
  Require status checks to pass before merging: YES
    Required status checks: [all CI stages listed above]
  Require branches to be up to date before merging: YES
  Restrict who can push to matching branches: [define your policy]
  Allow force pushes: NO
  Allow deletions: NO
```

**When trunk breaks: stop the line.**

When CI fails on a commit that reached trunk, it is a P0. Not a background ticket. Not a "we'll fix it in the next sprint." The team stops what it is doing and fixes the broken build. This practice is what gives TBD its force: trunk breakage is immediately visible to everyone and immediately everyone's problem. The pain of a broken trunk is distributed, which means fixes happen in minutes, not hours.

---

## 3. Feature Flags

Feature flags (also called feature toggles) are the primary mechanism that makes trunk-based development safe at scale. Without flags, "trunk is always deployable" would mean "every feature in progress must be complete before it can be merged." That is effectively Git Flow without the branch names. Flags break the coupling between when code is merged and when users see the feature.

### 3.1 What Feature Flags Are

A feature flag is a conditional in your code that selects a code path based on a runtime configuration value:

```typescript
if (featureFlags.isEnabled('new-checkout-flow', user)) {
  return renderNewCheckout(user);
}
return renderLegacyCheckout(user);
```

The key properties:

- The flag is evaluated at runtime, not at compile time
- The flag value is controlled separately from the code (config file, environment variable, flag service)
- Changing the flag value does not require a code deploy
- The code behind the flag is fully compiled, fully tested, and present in production — it is simply not reachable by the users for whom the flag evaluates to false

This separation of "code deployment" from "feature exposure" is what enables trunk-based development. Code can be merged to trunk (and deployed to production) before a feature is complete, because the flag ensures users do not encounter the incomplete feature.

### 3.2 Martin Fowler's Four Types

Martin Fowler's canonical taxonomy identifies four distinct flag types with different lifecycles, ownership models, and implementation requirements. Conflating them leads to the most common flag problems: using a release toggle as an ops toggle, or letting an experiment toggle linger indefinitely.

**Release Toggles**

*Purpose:* Allow incomplete or unreviewed code to exist in production without being exposed to users. The primary enabler of trunk-based development.

*Characteristics:*
- Short-lived: days to two weeks maximum, then cleaned up
- Evaluated statically per deploy (not dynamically per user, per request)
- Owned by the engineering team that built the feature
- Must be removed after full rollout — they are temporary by design

*Example:*

```typescript
// Feature is in progress. Code is merged to trunk. Users see legacy checkout.
if (flags.isEnabled('new-checkout-flow')) {
  return renderNewCheckout();
}
return renderLegacyCheckout();
```

A release toggle is a temporary scaffolding. Once the feature is fully rolled out and stable, the conditional is deleted and the old code path is removed. A release toggle that outlives its purpose becomes an ops toggle that nobody owns — which is dangerous.

**Ops Toggles (Kill Switches)**

*Purpose:* Allow operators to disable non-critical functionality during outages, capacity incidents, or unexpected load. Manual circuit breakers.

*Characteristics:*
- Can be long-lived — months to years as standing kill switches
- Must be changeable at runtime without a code deploy or application restart
- Owned by operations or SRE
- Require a distributed configuration system (Consul, etcd, a flag service with streaming updates) — a config file is insufficient because file reads require a restart or re-read trigger

*Example:*

Your recommendation engine queries an ML model that is expensive under load. During a database incident, you want to disable recommendations instantly without a deploy that takes 10 minutes. An ops toggle lets an on-call engineer flip a switch in the admin UI and have every instance stop calling the recommendation service within seconds.

**Experiment Toggles (A/B Flags)**

*Purpose:* Route different user cohorts to different code paths to measure behavioral outcomes. The technical implementation of A/B testing.

*Characteristics:*
- Per-request, dynamic decisions — each incoming request is independently evaluated
- Owned by product management
- Live for hours to weeks, until statistical significance is reached
- Require analytics instrumentation alongside the flag (the flag itself tells you nothing; the conversion events it produces tell you everything)

*Example:*

```typescript
// 50% of users see the primary CTA as "Get Started"
// 50% see it as "Start Free Trial"
// Analytics track which version produces more signups
const ctaText = await flags.getVariant('onboarding-cta-text', user);
return <Button>{ctaText}</Button>;
```

Experiment toggles are not controlled by engineering after implementation. The engineering team writes the branching code and instruments the analytics events. Product management controls when the experiment runs, which user percentage is in each variant, and when to end the experiment based on statistical results.

**Permission Toggles (Entitlement Flags)**

*Purpose:* Gate features for specific user segments — premium tier subscribers, beta testers, internal employees, enterprise customers.

*Characteristics:*
- Per-request and per-user — the evaluation requires user context
- Can be very long-lived (years, for standing premium feature gates)
- Owned by product or billing
- Require a data store that can be queried per-request with low latency (typically the flag service itself, backed by a fast cache)

*Example:*

```typescript
// "Export to Excel" is only available to Business tier subscribers
if (user.plan === 'business' && await flags.isEnabled('export-to-excel', user)) {
  return <ExportButton format="xlsx" />;
}
```

Permission toggles are the only flag type where long-lived is expected and appropriate. A premium feature gate is a business rule, not a temporary scaffolding. It does not have an expiry date.

### 3.3 Implementation Levels

The right implementation level depends on your team's current scale, flag volume, and targeting requirements. Start at the lowest level that meets your needs.

**Level 1: Config file**

Good for: teams just starting their TBD migration, fewer than 10 active flags, no per-user targeting needed.

```yaml
# config/feature-flags.yaml
flags:
  new-checkout-flow: false
  new-analytics-dashboard: false
  invoice-preview: true
  export-to-excel: false
```

```typescript
// flags.ts
import * as yaml from 'js-yaml';
import * as fs from 'fs';

const config = yaml.load(fs.readFileSync('config/feature-flags.yaml', 'utf8')) as any;

export function isEnabled(flagName: string): boolean {
  return config.flags[flagName] ?? false;
}
```

Limitations: Changing a flag requires a code deploy (the config file is baked into the build). No per-user targeting. No runtime changes. No UI. Entirely appropriate when these limitations are acceptable.

**Level 2: Environment variables**

Good for: teams with existing 12-factor app infrastructure, when you need per-environment flag values without a deploy, still no per-user targeting.

```typescript
// flags.ts
export function isEnabled(flagName: string): boolean {
  const envKey = `FEATURE_${flagName.toUpperCase().replace(/-/g, '_')}`;
  return process.env[envKey] === 'true';
}

// Usage
isEnabled('new-checkout-flow')
// reads FEATURE_NEW_CHECKOUT_FLOW from environment
```

Limitations: Changing a flag requires an environment variable update and an application restart (or platform-level hot injection). Still no per-user targeting. No UI. Harder to audit.

**Level 3: Database-backed with admin UI**

Good for: production flag management without external dependencies, basic percentage rollout, user allowlisting.

```sql
CREATE TABLE feature_flags (
  name         VARCHAR(100) PRIMARY KEY,
  enabled      BOOLEAN      NOT NULL DEFAULT FALSE,
  rollout_pct  INTEGER               DEFAULT 0,     -- 0-100 for percentage rollout
  user_ids     TEXT[]                DEFAULT '{}',  -- explicit allowlist
  expires_at   TIMESTAMP,
  owner        VARCHAR(100),
  flag_type    VARCHAR(20)  NOT NULL,               -- release|ops|experiment|permission
  description  TEXT,
  created_at   TIMESTAMP    NOT NULL DEFAULT NOW(),
  updated_at   TIMESTAMP    NOT NULL DEFAULT NOW()
);
```

This level allows product and operations teams to change flags at runtime through an internal admin panel without engineering involvement. Build a simple CRUD UI over this table and you have a flag service that supports most enterprise use cases.

**Level 4: Dedicated flag service**

For enterprise scale, per-request targeting, experimentation with statistical analysis, and cross-team flag management, use a purpose-built tool:

| Tool | Model | Self-Hosted | Best For |
|------|-------|-------------|---------|
| LaunchDarkly | SaaS only | No | Large enterprise; richest SDK ecosystem (50+ languages); built-in experimentation and A/B testing; streaming flag updates |
| Unleash | Open-source + SaaS | Yes | Teams needing data sovereignty, self-hosted control, or open-source alignment |
| Flagsmith | Open-source + SaaS | Yes | Budget-conscious teams; simpler requirements; strong open-source community |
| ConfigCat | SaaS | No | Simpler pricing model than LaunchDarkly; good developer experience |

All four major tools support the **OpenFeature** standard (a CNCF project), which defines a vendor-neutral SDK API for flag evaluation. Using OpenFeature in your application code means you can switch flag service providers without changing application logic — only the provider adapter changes.

### 3.4 Flag Lifecycle: Creation to Cleanup

The biggest risk with feature flags is not their introduction. It is their failure to be removed. Flags accumulate into a combinatorial test burden, a readability problem, and — in the worst cases — a dangerous operational hazard.

In 2012, Knight Capital Group lost $460 million in 45 minutes. The post-incident analysis identified a decommissioned feature toggle that had not been removed from the codebase. A new deployment activated the toggle on some servers but not others, causing the system to behave as if different code was running on different instances, routing orders through a legacy code path that executed in unintended ways. $460 million in 45 minutes is the most expensive example in software history of flag debt being realized.

**The lifecycle:**

```
1. CREATE → 2. DARK LAUNCH → 3. CONTROLLED ROLLOUT → 4. FULL RELEASE → 5. CLEANUP
```

**Stage 1: Create**

Define the flag in your flag store. Before writing any code, record:
- Flag name (use kebab-case consistently: `new-checkout-flow`, not `newCheckoutFlow`)
- Flag type (release, ops, experiment, permission)
- Owner (the team responsible for cleanup)
- Expiry date (for release toggles: 2-4 weeks from creation; for ops toggles: none; for experiments: end of test)
- Purpose (one sentence)
- Link to the cleanup ticket (create the cleanup ticket immediately, before writing code)

**Stage 2: Dark launch (optional)**

Deploy the new code with the flag off for all users. The code is in production but unreachable. Run it in the background for observability purposes — check that the new code path does not introduce latency regressions or unexpected errors in the instrumentation layer — without exposing any users.

If your flag service supports it, enable the flag for an internal allowlist (employees, the development team) to verify the feature end-to-end in the production environment before any users see it.

**Stage 3: Controlled rollout**

```
1%  → monitor for 24 hours → check error rates, latency, business metrics
5%  → monitor for 24-48 hours
25% → monitor for 48 hours
50% → monitor
100% → proceed to Stage 4 immediately
```

The key operational property of a controlled rollout: rollback is a flag toggle, not a code revert and redeployment. If error rates spike at 5%, flip the flag to 0% and the system returns to the previous behavior in seconds — no deployment required. This is why monitoring at each stage matters: you want to know quickly if something is wrong, while rollback is still cheap.

**Stage 4: Full release**

The flag is on for 100% of users. At this point, the feature is shipped and the flag has zero business value. It is now only costing maintenance overhead.

Begin cleanup immediately. Do not let the codebase sit in dual-path state. The longer the flag stays, the more context is lost about why both paths exist.

**Stage 5: Cleanup**

```typescript
// Before cleanup — dual path, flag still present
async function renderDashboard(user: User): Promise<DashboardView> {
  if (await featureFlags.isEnabled('new-analytics-dashboard', user)) {
    return renderNewDashboard(user);
  }
  return renderLegacyDashboard(user);
}

// After cleanup — single path, old code deleted, flag removed
async function renderDashboard(user: User): Promise<DashboardView> {
  return renderNewDashboard(user);
}
```

Steps:
1. Inline the "on" code path — remove the conditional, delete the "off" path (the old code)
2. Delete the flag from the flag store
3. Remove any test overrides that reference the flag
4. Close the cleanup ticket

**Enforce expiry with a test that fails the build:**

```typescript
// In your test suite — this test fails the build if a release toggle outlives its expiry
describe('feature flag hygiene', () => {
  it('release toggles must not outlive their expiry date', async () => {
    const flags = await flagRegistry.getAllFlags();
    const now = Date.now();
    const expired = flags
      .filter(f => f.type === 'release' && f.expiresAt && f.expiresAt < now);

    if (expired.length > 0) {
      const names = expired.map(f => `"${f.name}" (expired ${new Date(f.expiresAt).toISOString()})`);
      throw new Error(
        `The following release toggles are overdue for cleanup:\n  ${names.join('\n  ')}\n` +
        `Remove them from the codebase before merging.`
      );
    }
  });
});
```

This test failing the build is not optional. Flag debt, left unmanaged, becomes the Knight Capital scenario at the extreme and an unreadable codebase at the average.

### 3.5 Testing With Flags

**The combinatorial explosion problem**

With N binary flags, there are theoretically 2^N test states. At 10 active flags, that is 1,024 combinations. You cannot test all of them, and you do not need to.

The principle is: test the states that users will actually experience, not every mathematical combination.

**Test these states:**

1. **Current production state** — all flags exactly as they are in production right now. This is your regression baseline.
2. **Upcoming release state** — the flags as they will be configured when you ship the next feature. This is your forward validation.
3. **Rollback state** — the new flag turned off. This proves the legacy path still works after you've been building against the new path.
4. **Flag interactions** — only when two flags share a code path. Test the combination only if both flags being on (or off) produces different behavior than either alone.

```typescript
describe('checkout flow', () => {
  afterEach(() => {
    flags.reset();  // clear any overrides set during the test
  });

  it('renders legacy checkout when new-checkout-flow is OFF (current production)', () => {
    flags.override({ 'new-checkout-flow': false });
    render(<CheckoutPage user={mockUser} />);
    expect(screen.getByTestId('legacy-checkout')).toBeInTheDocument();
  });

  it('renders new checkout when new-checkout-flow is ON (upcoming release)', () => {
    flags.override({ 'new-checkout-flow': true });
    render(<CheckoutPage user={mockUser} />);
    expect(screen.getByTestId('new-checkout')).toBeInTheDocument();
  });

  it('falls back to legacy checkout if new-checkout-flow is toggled off mid-rollout', () => {
    // rollback state: confirms the old path is not broken by the new code
    flags.override({ 'new-checkout-flow': false });
    render(<CheckoutPage user={mockUser} />);
    expect(screen.getByTestId('legacy-checkout')).toBeInTheDocument();
  });
});
```

**Make flags injectable in tests.** If your flag implementation calls a real flag service in tests, you will have slow, non-deterministic tests and no control over flag state. Flags must be injectable or overridable in test environments:

```typescript
export class FeatureFlags {
  private overrides: Record<string, boolean> = {};

  constructor(private readonly source: FlagSource) {}

  async isEnabled(name: string, user?: User): Promise<boolean> {
    if (name in this.overrides) {
      return this.overrides[name];
    }
    return this.source.evaluate(name, user);
  }

  // Test helper — only call in test environments
  override(flags: Record<string, boolean>): void {
    this.overrides = { ...this.overrides, ...flags };
  }

  reset(): void {
    this.overrides = {};
  }
}
```

### 3.6 Who Owns Flags

Flag ownership must be explicit because the consequences of the wrong person changing a flag can range from surprising to catastrophic.

| Flag Type | Owner | Change Mechanism | Engineering Review Required? |
|-----------|-------|------------------|-------------------------------|
| Release toggle | Engineer who built the feature | PR to code or flag service config | Yes |
| Ops toggle / kill switch | SRE / Operations | Admin UI, available to on-call engineers | No (this is the point — must be changeable instantly) |
| Experiment toggle | Product Manager | Flag service admin UI | No (PM controls experiment timing) |
| Permission toggle | Product or Billing | Flag service admin UI | No (business rule, not code) |

**The critical boundary:** Product managers must not be able to modify release toggles without engineering review. Release toggles gate incomplete code. Enabling a release toggle prematurely can expose an unfinished feature, trigger code paths that have only been tested partially, or expose in-progress API changes to external consumers. Engineering owns release toggles until the feature is fully shipped.

**Maintain a flag registry.** Even a shared document works: flag name, type, owner, creation date, expiry date, cleanup ticket link, and current rollout percentage. Review it as a standing agenda item in sprint planning. Flags with no expiry date and no owner are a liability.

---

## 4. Migrating from Git Flow to Trunk

### 4.1 The Mindset Shift

The challenge of migrating from Git Flow to TBD is not technical. The git commands do not change. The challenge is a fundamental shift in how the team thinks about the relationship between code, branches, and stability.

| Git Flow Mindset | TBD Mindset |
|-----------------|-------------|
| Stability through isolation — branches protect trunk from incomplete work | Stability through automation — CI gates protect trunk from broken work |
| Features are complete before they are shared with the team | Code is always shared; features are gated by flags, not branches |
| Merging is a periodic event — sprint end, release cut | Merging is a continuous activity — multiple times per day |
| The release is when the branch is ready | The release is whenever you choose to deploy trunk |
| "Don't break develop" is a cultural norm, lightly enforced | "Don't break trunk" is a hard rule, enforced by CI and branch protection |
| A long-lived feature branch is normal | A long-lived branch is a failure mode — a sign something went wrong |

A team that adopts TBD tooling while keeping the Git Flow mindset will produce the worst of both worlds. Developers will push partial code to trunk without flags. Trunk will break constantly. The team will lose trust in the process and revert to long-lived branches "temporarily" — and the temporary branches will stay.

The mindset shift must precede the tooling change. Without it, the tooling change fails.

### 4.2 Prerequisites Before Starting

Do not begin the migration until every item on this list is in place. Attempting TBD without these prerequisites is not "starting with imperfect infrastructure and improving it" — it is producing trunk instability that will cause the team to abandon the model.

**Non-negotiable:**
- Automated test coverage that passes reliably. Flaky tests must be eliminated or quarantined before migration.
- CI pipeline that runs on every push and produces an authoritative pass/fail signal.
- Branch protection on `main` requiring CI green before any merge.
- A feature flag system — even a config-file implementation (Level 1 from Section 3.3) is sufficient to start.
- Explicit team agreement that broken trunk is treated as a P0 — work stops until trunk is green.

**Strongly recommended:**
- CI that completes in under 15 minutes end-to-end.
- Monitoring and alerting on production so post-deploy regressions are caught quickly.
- PR review SLA — for example, reviewers must respond within 4 hours. In TBD, a PR that sits for 3 days is equivalent to a 3-day feature branch: the developer is blocked from integrating their next increment.

### 4.3 Migration Phases

**Phase 0: Fix CI (weeks 1-4) — do not change branching yet**

Many teams discover when they attempt this migration that their CI is too slow, too flaky, or not comprehensive enough to support TBD. Fixing CI is the prerequisite, not the side project.

Deliverables:
- CI runs on every PR, automatically
- CI is reliable: less than 2% flaky test rate (measure this explicitly)
- CI completes in time to give meaningful feedback within a PR's review lifecycle
- `main` is protected: no direct pushes, PR and CI required to merge

Do not move to Phase 1 until these are true. Continuing with flaky or slow CI will produce a TBD migration that fails.

**Phase 1: Shorten branch lifetimes without changing the branch model (weeks 4-8)**

Keep Git Flow's branch structure. Do not introduce GitHub Flow or TBD branches yet. Simply enforce shorter lifetimes on the branches you already have.

Actions:
- Set a team policy: feature branches must merge within 5 working days (not 3 weeks)
- Set a code review SLA: reviewers respond within the same business day
- Start breaking features into smaller tickets that can be implemented in 1-3 days
- Observe where friction appears: too-large PRs, slow reviews, flaky tests

This phase has two purposes: it surfaces the real obstacles (which are almost always large PRs and slow reviews, not the branch model itself), and it begins shifting the team's habit toward smaller, more frequent integrations.

**Phase 2: Introduce feature flags (weeks 6-12)**

Pick 2-3 upcoming features and implement them behind flags from the beginning. Use the simplest flag implementation that supports your use case — Level 1 or Level 2 from Section 3.3.

Goals for this phase:
- Engineering team learns the flag lifecycle: create, dark launch, rollout, cleanup
- Product management learns to control rollout timing via flags rather than release branches
- CI learns to test both flag states
- The team builds confidence that incomplete code behind a flag is safe to ship to production

This phase is the most important one. Feature flags are what make trunk-safe development possible for any feature that spans multiple PRs. Without flags, the only trunk-safe code is code that is complete and ready for users — which forces long-lived branches.

**Phase 3: Adopt GitHub Flow (weeks 8-16)**

GitHub Flow is the intermediate state between Git Flow and TBD. Its rules:
1. Branch from `main` — not from `develop`
2. Commit incrementally to the branch
3. Open a PR against `main`
4. CI passes, reviewer approves
5. Merge to `main`
6. Delete the branch

This eliminates `develop`, which is Git Flow's largest source of merge conflict accumulation. It does not yet enforce the 1-2 day branch lifetime rule, which is what makes it a good intermediate state: it is familiar enough that teams accept it without feeling like a disruptive change.

Migration steps:
1. Stop creating new work against `develop` — new work branches from `main`
2. Let existing `develop`-based feature branches complete their lifecycle and merge as planned
3. Allow `develop` to drain naturally
4. Retire `develop` once it is empty and all active branches have merged or been rebased to `main`

Do not force-migrate existing branches. Force-migrating active feature branches creates a chaotic integration event and erodes team trust in the migration process.

**Phase 4: Enforce TBD discipline (months 4-6)**

With branches already short-lived (from Phase 1), flags in use (from Phase 2), and the `develop` layer removed (from Phase 3), you are running close to TBD already. Phase 4 is tightening the screws.

Actions:
- Enforce 1-2 day maximum branch lifetime — use branch age reporting in your CI system or code review tooling
- All release branches are cut from `main`, not from any intermediate branch
- All hotfixes are applied to `main` first, then cherry-picked to active release branches
- Remove any remaining long-lived branches (there should be none, but if any crept in, now is when they are addressed)

**Phase 5: Evaluate direct-to-trunk (ongoing)**

Once the team has high trust in CI and the daily cadence is established, some developers can commit small, low-risk changes directly to trunk without a PR — documentation fixes, single-line typo corrections, trivial configuration changes. This is optional, team-specific, and should start with senior engineers who can accurately judge what qualifies as "low-risk." It is not a requirement for TBD.

### 4.4 Handling Existing Long-Lived Branches

When the migration begins, there will likely be several active feature branches with weeks of history. Do not force-merge them.

The correct approach:
- Allow all active branches to complete their normal lifecycle and merge to `develop` (or `main`, if you've already moved to Phase 3) as planned
- Set a date after which no new long-lived branches are started (this is your first concrete policy change)
- Communicate the policy change clearly — including the rationale — before implementation
- The last cohort of long-lived branches becomes the natural end of Git Flow in your repository

Trying to forcibly terminate branches mid-work creates bad merges, frustrated developers, and a narrative that "the TBD migration broke our release" — which will haunt the effort for months.

### 4.5 Common Risks and Mitigations

**Risk: Incomplete features accidentally ship to users**

Mitigation: Feature flags (Section 3). This is the core mechanism and it is not optional. Code ships to production, but the feature is toggled off for all users until the team is ready. Every feature that spans multiple PRs must be behind a flag.

**Risk: Trunk breaks constantly during the transition period**

Mitigation: Fix CI before changing branching (Phase 0). Do not allow merges to trunk when CI is red. Treat broken trunk as P0 immediately. If trunk is breaking more than once per week, stop the migration and fix the CI reliability problem first.

**Risk: The team reverts to long-lived branches under deadline pressure**

Mitigation: Establish the policy at the engineering leadership level, not just among individual developers. The instinct under deadline pressure is to open a "stabilization branch" — which recreates Git Flow under a different name. This must be explicitly prohibited and the prohibition must have leadership backing. If it happens, address it immediately and publicly.

**Risk: PR review bottleneck replaces merge conflict bottleneck**

Mitigation: Establish and enforce PR review SLAs. In TBD, a PR that sits unreviewed for three days is exactly as harmful as a three-day feature branch. The developer cannot integrate their next increment while the current PR is in review. Reviews must happen the same day. This often requires changing team norms around asynchronous reviews versus real-time pairing, and it requires managers to treat review turnaround as a delivery metric.

**Risk: Feature flags accumulate, codebase becomes unreadable**

Mitigation: Create a cleanup ticket for every flag at flag creation time. Set expiry dates on release toggles. Add the test that fails the build when release toggles expire (Section 3.4). Review the flag registry monthly. Make cleanup a first-class engineering activity, not a "when we have time" task.

---

## 5. Decision Guide

The choice between Git Flow and trunk-based development is not a universal question of which model is "better." It is a question of fit: does this model match your release cadence, team structure, CI investment, and regulatory environment?

### When Git Flow Is the Right Tool

**Regulated environments with formal release artifacts.** If your release process requires an auditable, immutable release artifact that has been formally approved by QA, compliance, or an external auditor, Git Flow's release branch provides a clear audit trail: this exact branch, tagged as this version, was reviewed and approved on this date. The structure maps naturally to SOC 2 Type II evidence collection, FDA 21 CFR Part 11 workflows, and financial software audit requirements.

**Products with multiple major versions simultaneously in production.** If you support version 2.x and version 3.x as independent, separately-licensed products — where a customer on 2.x might have paid for a specific behavior that must not change — Git Flow's long-lived branch structure maps naturally to this reality. The `2.x` line needs its own separate fixes, its own QA cycle, and its own release process. TBD's "trunk is the only thing" model does not accommodate this naturally.

**Large teams with a distinct, separate QA phase.** If you have a QA organization that receives a build, runs it through a formal test cycle (manual regression, exploratory testing, performance testing), and signs off before release, Git Flow's release stabilization branch provides the handoff artifact. QA receives `release/2.3.0`, runs its cycle on that fixed snapshot, and releases it when ready. In TBD, trunk moves continuously, which makes "QA signs off on trunk" a moving target.

**Teams without automated test infrastructure.** TBD without reliable automated tests is dangerous. If your team does not have meaningful test coverage and reliable CI, trunk-based development will produce trunk instability continuously. Git Flow's isolation at least contains breakage to individual feature branches. Fixing CI is a prerequisite for TBD, not something you can build while also running TBD.

**Packaged software with scheduled release cycles.** Desktop applications, embedded systems, annual major versions, and any product where "upgrading" requires a deliberate customer action fit the Git Flow cadence naturally. These products release quarterly or annually by design. Git Flow's branch structure reflects that cadence.

### When Trunk-Based Development Is the Right Tool

**Web and SaaS products with continuous deployment.** If "upgrading" means nothing — users visit your URL and automatically get the current version — you have nothing to gain from Git Flow's release stabilization ceremony and everything to gain from TBD's deployment flexibility. Every hour a bug exists in production without a fix being shippable is unnecessary cost.

**Teams wanting to deploy more frequently than weekly.** The operational overhead of a Git Flow release branch — creating it, stabilizing it, merging it, tagging it, reverse-merging it — is measured in hours per release. At once per month, this is acceptable. At twice per week, it is a bottleneck. At multiple times per day, it is impossible.

**Teams experiencing merge conflict overhead exceeding 10% of engineering time.** If your sprint planning includes a known "merge day" at the end of the sprint, or if your team regularly spends 4+ hours resolving merge conflicts per release, the Git Flow model is actively costing you. The conflict is not incidental; it is structural. TBD's short-lived branches eliminate the structural cause.

**Microservices architectures.** Coordinating Git Flow across multiple repositories with inter-service dependencies is a solved problem only in the sense that teams have developed workarounds — manifest files, coordination spreadsheets, deploy scripts that check branch states across repos. None of these are solutions. TBD's "trunk is always deployable" model makes cross-service deployment straightforward: each service deploys from its own trunk, and backward-compatible API versioning handles compatibility.

**Teams with existing CI/CD investment.** If you have a fast, reliable CI pipeline, branch protection enforcing it, and deployment automation triggered by trunk merges, you already have the infrastructure TBD requires. Using Git Flow on top of this infrastructure adds ceremony without adding safety.

### The Honest Prerequisite Checklist for TBD

Before committing to a TBD migration, verify all of these. Skipping prerequisites produces trunk instability that causes reversion.

- [ ] Automated tests with meaningful coverage (not 100%, but sufficient that a broken change fails CI)
- [ ] CI pipeline that is fast (under 20 minutes) and reliable (under 2% flaky test rate)
- [ ] Branch protection on `main` requiring CI green before any merge — enforced by the platform, not by convention
- [ ] Feature flag infrastructure at any level (config file is sufficient to start)
- [ ] Explicit team agreement that broken trunk is treated as a P0, not a background ticket
- [ ] PR review cadence that supports daily integration — reviews must happen the same day a PR is opened

Attempting TBD without all six produces a predictable failure mode: trunk breaks, developers lose trust, long-lived branches return "temporarily," and the migration is quietly declared unsuccessful. The prerequisites are not a high bar — but all six must be true simultaneously.

---

## Sources

- [Feature Toggles (aka Feature Flags) — Martin Fowler](https://martinfowler.com/articles/feature-toggles.html) — Canonical taxonomy, lifecycle model, implementation patterns, Knight Capital postmortem context
- [Branch by Abstraction — Martin Fowler](https://martinfowler.com/bliki/BranchByAbstraction.html) — Technique for large structural changes without long-lived branches
- [Trunk Based Development — trunkbaseddevelopment.com](https://trunkbaseddevelopment.com/) — Core principles, short-lived branch rules, CI requirements
- [Branch for Release — trunkbaseddevelopment.com](https://trunkbaseddevelopment.com/branch-for-release/) — Release branching strategy in TBD context, cherry-pick-first policy
- [Feature Flags — trunkbaseddevelopment.com](https://trunkbaseddevelopment.com/feature-flags/) — Flags as a TBD enabling mechanism
- [Short-Lived Feature Branches — trunkbaseddevelopment.com](https://trunkbaseddevelopment.com/short-lived-feature-branches/) — Two-day rule, PR flow, CI requirements
- [Continuous Integration — trunkbaseddevelopment.com](https://trunkbaseddevelopment.com/continuous-integration/) — CI pipeline requirements for TBD
- [Long-Lived Branches with Gitflow — Thoughtworks Technology Radar](https://www.thoughtworks.com/radar/techniques/long-lived-branches-with-gitflow) — "Hold" recommendation with rationale
- [DORA Accelerate State of DevOps 2024](https://dora.dev/research/2024/dora-report/) — Elite performers 2.3x more likely to use TBD; longitudinal data across performance tiers
- [A Successful Git Branching Model — Vincent Driessen](https://nvie.com/posts/a-successful-git-branching-model/) — The original Git Flow specification (2010)
- [Please Stop Recommending Git Flow — George Stocker](https://georgestocker.com/2020/03/04/please-stop-recommending-git-flow/) — Failure modes at scale, merge conflict tripling at parallel branch count
