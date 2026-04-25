# Enterprise Release Branching: Git Flow vs Trunk-Based Development

**Researched:** 2026-04-25
**Audience:** Enterprise feature-driven software team currently on Git Flow, migrating toward TBD
**Confidence:** HIGH (primary sources: Martin Fowler, trunkbaseddevelopment.com, DORA 2024 report, Thoughtworks Radar)

---

## Table of Contents

1. [Git Flow in the Enterprise Context](#1-git-flow-in-the-enterprise-context)
2. [Trunk-Based Development](#2-trunk-based-development)
3. [The Migration Path: Git Flow to Trunk](#3-the-migration-path-git-flow-to-trunk)
4. [Feature Flags Deep Dive](#4-feature-flags-deep-dive)
5. [Decision Framework](#5-decision-framework)
6. [Sources](#6-sources)

---

## 1. Git Flow in the Enterprise Context

### 1.1 Branch Structure

Git Flow defines five branch types with explicit, non-negotiable roles:

```
main (or master)     ← production-only; every commit is a release tag
develop              ← integration branch; "next release" in progress
feature/ABC-123      ← individual feature work; branches off develop
release/1.4.0        ← release stabilization; branches off develop
hotfix/fix-payment   ← emergency fix on production; branches off main
```

The key structural rule: **main and develop are permanent; all others are temporary.**

Vincent Driessen's original model (2010) intended `develop` to be the "integration" surface and `main` to be a record of shipped software. In practice, most teams add a third permanent concept: the release branch acts as a stabilization gate between develop and main.

### 1.2 Parallel Feature Development

Git Flow handles parallel features by giving each feature its own branch off `develop`. Features are isolated from each other until they merge back.

```bash
# Start feature A
git checkout develop
git checkout -b feature/user-auth

# Start feature B (independently, same time)
git checkout develop
git checkout -b feature/payment-redesign
```

Features integrate only when they complete and merge into `develop`. This means two features that touch the same files will not conflict until merge time — which is the core source of Git Flow's later pain. The isolation is real during development but deferred conflict, not eliminated conflict.

### 1.3 What "Stable" Means in Git Flow

Git Flow uses a layered stability model:

| Branch | Stability Meaning |
|--------|------------------|
| `main` | Always production-deployable; only released, tagged code |
| `release/x.y.z` | Intended to be stable; only bug fixes, no new features |
| `develop` | Integration stable — compiles, tests pass, but may have unreleased bugs |
| `feature/*` | Developer stable — may be broken, work in progress |
| `hotfix/*` | Surgical — must be minimal and immediately stable |

The promise is: **if you're on main, you can ship it.** The problem is that the distance between develop and main can represent weeks or months of work, so "stable" on develop is loosely defined.

### 1.4 Merge Strategy and Integration Points

Git Flow has four merge choreographies, each with distinct directionality:

**1. Feature → Develop (feature complete)**
```bash
git checkout develop
git merge --no-ff feature/user-auth
git branch -d feature/user-auth
git push origin develop
```
The `--no-ff` flag is conventional to preserve branch history in the DAG. Some teams use squash merges to keep develop history linear.

**2. Develop → Release branch (release cut)**
```bash
git checkout develop
git checkout -b release/2.3.0
# bump version, update changelog
git commit -am "Bump version to 2.3.0"
```
At this point, develop is open for the next release's feature work immediately.

**3. Release → Main AND Release → Develop (release ships)**
```bash
git checkout main
git merge --no-ff release/2.3.0
git tag -a v2.3.0 -m "Release 2.3.0"

# CRITICAL: merge back to develop so bug fixes aren't lost
git checkout develop
git merge --no-ff release/2.3.0
git branch -d release/2.3.0
```
The develop merge-back is frequently forgotten. Missing it means hotfixes and release fixes are silently lost.

**4. Hotfix → Main AND Hotfix → Develop (emergency)**
```bash
git checkout main
git checkout -b hotfix/fix-null-pointer
# fix the bug
git checkout main
git merge --no-ff hotfix/fix-null-pointer
git tag -a v2.2.1
git checkout develop
git merge --no-ff hotfix/fix-null-pointer
git branch -d hotfix/fix-null-pointer
```

### 1.5 Release Process: Cut, Stabilize, Tag

The release lifecycle in Git Flow has three phases:

**Phase 1: Cut the release branch**
- When develop has the features intended for the release, cut `release/x.y.z`
- No new features land on this branch after cut
- Develop continues for the next cycle immediately

**Phase 2: Stabilize**
- QA runs on the release branch
- Bug fixes commit directly to `release/x.y.z` (not develop, then cherry-pick)
- Each fix on the release branch must also be applied to develop (manual or scripted)
- Version number bumps, changelog updates, release notes happen here

**Phase 3: Tag and merge**
- When QA signs off, merge to main
- Tag main with the version: `git tag -a v2.3.0 -m "Release 2.3.0"`
- Merge release branch back to develop (capture all stabilization fixes)
- Delete the release branch

### 1.6 Pain Points at Scale

These are the failure modes that drive teams off Git Flow. They are not theoretical — they are endemic at the 20+ developer scale.

**Integration Hell (the most common)**

Long-lived feature branches accumulate divergence. When feature/A and feature/B both branch off `develop` on day 1 and merge 3 weeks later, they have no knowledge of each other's changes. The merge conflicts are resolved in isolation, often by people who didn't write the original code.

The conflict rate is non-linear. With three branch types (feature, release, hotfix) all merging into `develop`, there are roughly 3x the standard merge opportunities for conflict. At 10 concurrent features, this becomes untenable.

**The "Frozen Develop" Problem**

When a release branch is being stabilized, teams often informally freeze `develop` to avoid contaminating the release. This creates dead time for the entire team. It's not a Git Flow rule — it's a human response to complexity.

**Merge Trains and Review Backlogs**

In a 20-developer team, merging 20 feature branches into `develop` at end-of-sprint creates a merge queue. The first merge is clean. The tenth merge is resolving conflicts introduced by the previous nine. Teams often spend an entire day merging at sprint end.

**The Forgotten Merge-Back**

When a bug is fixed on `release/2.3.0` or `hotfix/2.2.1`, it must also be applied to `develop`. This is mandatory but manual. Forgetting it means the bug reappears in the next release. Many teams have incident post-mortems that trace to this exact failure.

**Incompatibility with Continuous Delivery**

Git Flow assumes a release cadence of weeks to months. Deploying from `develop` directly to production is explicitly not in the model. Teams that want to deploy multiple times per day find Git Flow's branch overhead directly in the way. As Thoughtworks put it: "long-lived branches are the opposite of continuously integrating all changes."

**Microservices Coordination Failure**

In a microservices architecture with multiple repos, coordinating Git Flow across repos requires YAML manifest management and cross-team synchronization that scales exponentially. Each repo's develop branch has a different state. Deploying a feature that spans three services requires tracking three feature branches, three release branches, and their relative states.

---

## 2. Trunk-Based Development

### 2.1 Core Principle

The single defining rule of TBD: **every developer integrates their work to trunk (main) at least once per day.**

That's it. Everything else — short-lived branches, feature flags, CI gates — is infrastructure that makes this rule safe to follow. The purpose of the rule is continuous integration in the literal sense: integrating code continuously, not just running tests.

Trunk is not develop. In Git Flow, develop accumulates unreleased work indefinitely. In TBD, trunk is always in a state where it *could* be released. The distinction is absolute: **trunk IS the deployable artifact at all times.**

### 2.2 Short-Lived Feature Branches

TBD does not require committing directly to trunk, but it requires integrating to trunk within 1-2 days maximum.

```bash
# Start work
git checkout main
git pull
git checkout -b feat/add-invoice-preview    # branch off trunk

# Work, commit locally
git add src/invoice/preview.ts
git commit -m "Add invoice preview component"

# Integrate same day (or next day at latest)
git checkout main
git pull --rebase origin main
git checkout feat/add-invoice-preview
git rebase main
# open PR, get fast review, merge to main
# branch immediately deleted after merge
```

**Critical differences from Git Flow feature branches:**

| Property | Git Flow Feature Branch | TBD Feature Branch |
|----------|------------------------|-------------------|
| Max lifetime | Weeks to months | 1-2 days |
| Merges into | `develop` | `main` |
| Developers per branch | Multiple (team feature) | 1 (or pair) |
| Contains | Complete feature | Incremental slice |
| Number active simultaneously | One per feature | One per developer |

The branch is a code-review artifact, not a development isolation unit.

### 2.3 Parallel Features Without Long-Lived Branches

This is the question that trips up most teams migrating from Git Flow: "How do you work on a feature that takes 3 weeks without a 3-week branch?"

The answer has two components:

**Component 1: Decompose vertically, not horizontally**

Instead of "feature branch lives until feature ships," decompose work into trunk-safe increments:
- Each increment is independently deployable (or safely hidden behind a flag)
- An increment might be "add the DB schema" with no UI yet
- An increment might be "add API endpoint, no consumer yet"
- The feature is complete when all increments are in trunk and the flag is removed

**Component 2: Feature flags for in-progress work**

Code that isn't ready for users is merged to trunk behind a flag. The flag gates exposure, not existence. The code compiles, tests run, CI passes — the flag just prevents users from triggering that code path.

```typescript
// trunk-safe, mergeable immediately
async function renderDashboard(user: User) {
  if (featureFlags.isEnabled('new-analytics-dashboard', user)) {
    return renderNewDashboard(user);     // work in progress, safe
  }
  return renderLegacyDashboard(user);   // current behavior unchanged
}
```

**Component 3: Branch by Abstraction for large structural changes**

When replacing a core component (e.g., migrating from one ORM to another), use an abstraction layer:

1. Create an interface that both old and new implementations satisfy
2. Migrate all callers to use the interface (committed to trunk incrementally)
3. Build the new implementation behind the interface
4. Switch to the new implementation via a toggle
5. Delete the old implementation and the abstraction layer

This allows a multi-week architectural change with no long-lived branch, no merge conflict risk, and the system is always in a runnable state.

### 2.4 What "Stable" Means in TBD

In TBD, there is one stability concept: **trunk is always releasable.**

This is enforced by:
1. **Pre-commit local verification**: Developer runs the full build locally before pushing
2. **CI gate on PR**: All tests must pass before the PR can merge to trunk
3. **Fast CI**: Build must be fast enough (under 10 minutes ideally) that broken builds are caught and fixed before they compound

There is no "integration branch" where things are allowed to be broken. There is no "develop branch" that's allowed to have known failures. Trunk is green, or you stop everything and fix it.

This sounds strict but is actually easier to maintain than Git Flow's "develop is usually stable" because the invariant is binary and enforced by automation.

### 2.5 Release Branching in TBD

TBD teams have two release models:

**Model A: Release from trunk directly (highest-trust CD)**

No release branches. Deploy from trunk. When a release is needed, take the current trunk SHA. If a bug is found post-release, fix forward — write a fix on trunk and deploy the new trunk. This requires:
- Very fast CI (so fixes deploy within minutes)
- Comprehensive monitoring and alerting
- Confidence in the trunk stability invariant
- Feature flags to disable anything not ready for users

**Model B: Cut a release branch from trunk (common in enterprise)**

```bash
# Cut just-in-time, when ready to release
git checkout main
git checkout -b release/2.3.0     # snapshot of trunk at release time
git push origin release/2.3.0
```

Rules for release branches in TBD:
- They are cut **late** (days before release, not weeks)
- **No new feature development** on the release branch
- Bug fixes go to **trunk first**, then cherry-pick to the release branch
- Never fix on release and merge back to trunk (prevents forgotten fixes)
- Release branches are **deleted** after the release ships
- The team continues full-speed on trunk during stabilization

```bash
# Fix on trunk first
git checkout main
git checkout -b fix/invoice-tax-rounding
# fix and test
git checkout main && git merge fix/invoice-tax-rounding
git tag main-fix-invoice-tax-rounding     # optional reference

# Cherry-pick to release branch
git checkout release/2.3.0
git cherry-pick <commit-sha>
git push origin release/2.3.0
```

This unidirectional flow (trunk → release) prevents the forgotten-merge-back failure mode that plagues Git Flow.

### 2.6 CI/CD Requirements

TBD only works if the CI pipeline is trustworthy and fast. These are prerequisites, not nice-to-haves:

**Pipeline stages (in order, must all pass to land on trunk):**
1. Compile / type check
2. Unit tests (must be fast: target < 5 min)
3. Integration tests
4. Functional/E2E tests (can run in parallel, may be slower)
5. Static analysis, linting, security scans

**Speed requirements:**
- Total pipeline: ideally < 10 minutes, acceptable < 20 minutes
- If CI takes 45+ minutes, developers stop waiting for it, start batch-committing, and trunk stability erodes
- Use parallel test runners, test sharding, incremental builds (Bazel, Nx, Turborepo) for large repos

**Branch protection rules (enforce in GitHub/GitLab/Bitbucket):**
```
main branch rules:
  - Require PR before merge: YES
  - Require status checks: ALL CI stages
  - Require up-to-date before merge: YES
  - Dismiss stale reviews: YES
  - Require linear history: OPTIONAL (matters for bisect-ability)
```

**Trunk is broken: stop the line**

When CI fails on trunk, it is a P0. Not a "someone will fix it later." Every developer on the team is blocked until trunk is green again. This is the practice that gives TBD its force: the breakage is immediately visible and immediately everyone's problem.

### 2.7 DORA Metrics and TBD

The 2024 DORA State of DevOps Report (decade-long longitudinal study) found:
- Elite performers meeting reliability targets are **2.3x more likely to use trunk-based development**
- Elite teams: lead time < 1 day, deploy on demand, change failure rate ~5%, MTTR < 1 hour
- Only 19% of teams surveyed reached Elite level in 2024

TBD is not sufficient for elite performance, but it is consistently present in elite organizations. Git Flow is not present in elite organizations.

---

## 3. The Migration Path: Git Flow to Trunk

### 3.1 Why This Is Hard

The challenge is not technical. Git commands don't change. The challenge is a **fundamental mindset shift**:

| Git Flow Mindset | TBD Mindset |
|-----------------|-------------|
| Stability through isolation | Stability through automation |
| Features are complete before shared | Code is always shared, features gated by flags |
| Merging is an event (sprint end) | Merging is continuous (multiple times/day) |
| Release is when the branch is ready | Release is whenever you choose to deploy trunk |
| "Don't break develop" (soft rule) | "Don't break trunk" (hard rule, enforced by CI) |
| Long-lived feature branch = normal | Long-lived branch = failure mode |

A team can adopt TBD tooling while keeping Git Flow mindset. This produces the worst of both worlds: developers push partial code to trunk without flags, trunk breaks constantly, team loses trust in the process and reverts.

### 3.2 Prerequisites Before You Start

Do not start the migration until these are in place:

**Non-negotiable:**
- [ ] Automated test coverage that passes reliably (flaky tests kill TBD)
- [ ] CI pipeline that runs on every push and is authoritative
- [ ] Branch protection on `main` that requires CI green before merge
- [ ] A feature flag system (even a simple config-file implementation)
- [ ] Team agreement on "trunk broken = stop the line"

**Strongly recommended:**
- [ ] Fast CI (< 15 min end-to-end)
- [ ] Monitoring and alerting on production
- [ ] PR review SLA (e.g., review within 4 hours; otherwise TBD's daily cadence breaks down)

### 3.3 Migration Phases

**Phase 0: Don't change branching yet — fix CI (weeks 1-4)**

Many teams discover their CI is too slow, too flaky, or not comprehensive enough to support TBD. Fix this first. You cannot run TBD on a CI that takes 45 minutes or passes intermittently.

Deliverables:
- CI runs on every PR
- CI is reliable (< 2% flaky test rate)
- CI is fast enough to give meaningful feedback within a PR lifecycle
- Main is protected: no direct pushes, PR + CI required

**Phase 1: Shorten branch lifetimes (weeks 4-8)**

Don't change the branch model yet. Just enforce shorter branch lifetimes on the existing develop-based model.

- Set a team policy: feature branches must merge within 5 days
- Start code reviews with faster SLAs (same-day response target)
- Begin decomposing features into smaller tickets
- Observe where the friction is (too-large PRs, slow reviews, flaky tests)

**Phase 2: Introduce feature flags (weeks 6-12)**

Pick 2-3 new features and implement them behind flags. Use the simplest possible implementation (see Section 4.2). Goals:
- Team learns the flag lifecycle
- PM/product learns to manage flag rollout
- CI learns to test both flag states

This is the most important phase. Feature flags are what make trunk-safe development possible for features that span multiple PRs.

**Phase 3: Introduce GitHub Flow (the intermediate state, weeks 8-16)**

GitHub Flow: short-lived branches directly off `main` (no `develop`), merged to `main` via PR. This is TBD with training wheels.

Steps:
1. Stop creating new `develop` branches for new work — branch from `main` directly
2. Let existing long-lived feature branches complete their lifecycle and merge to develop as planned
3. Retire the `develop` branch once it's drained
4. New features: branch from `main`, target `main`, merge within 2 days

This eliminates the `develop` layer, which is the biggest source of Git Flow's merge conflict accumulation.

**Phase 4: Enforce TBD discipline (months 4-6)**

Now that branches are short-lived, flags are in use, and CI is trusted:
- Enforce 1-2 day maximum branch lifetime (use branch age tooling or linting)
- Release branches are cut from `main`, not from `develop`
- Hotfixes go to `main` first, then cherry-picked to release branches
- Remove any remaining long-lived branches

**Phase 5: Evaluate direct-to-trunk (ongoing)**

Once team trust in CI is high, some developers (start with seniors) can commit directly to trunk for small, low-risk changes (documentation, trivial fixes, single-line changes). This is optional and team-specific.

### 3.4 Handling Existing Long-Lived Branches

Do not force-merge existing feature branches during migration. This creates a chaotic integration event and erodes trust.

Instead:
- Let active branches complete their lifecycle and merge normally
- Set a date after which no new long-lived branches are started
- Communicate the policy change clearly before implementation
- The last cohort of long-lived branches becomes the natural end of Git Flow in your repo

### 3.5 Key Risks and Mitigations

**Risk: Incomplete features shipping to users**
Mitigation: Feature flags. Code ships to production, but the feature is toggled off for all users until ready. This is not optional — it's the core mechanism.

**Risk: Trunk breaks constantly during transition**
Mitigation: Fix CI first (Phase 0). Do not allow merges to trunk when CI is red. Broken trunk must be treated as P0 immediately.

**Risk: Team reverts to long-lived branches under deadline pressure**
Mitigation: Establish the policy at leadership level, not just engineering. When a deadline feels close, the instinct is to open a long-lived "stabilization branch" — this recreates Git Flow and should be explicitly prohibited.

**Risk: PRs pile up, review bottleneck replaces merge conflict bottleneck**
Mitigation: PR review SLAs. In TBD, a PR that sits for 3 days is as bad as a 3-day feature branch. Reviews must happen same day.

**Risk: Feature flags accumulate, codebase becomes unreadable**
Mitigation: Treat flag creation as ticket creation. Every flag introduced creates a cleanup ticket. Set expiry dates. See Section 4.3 for lifecycle management.

### 3.6 What GitHub Flow Is (and Why It's a Good Intermediate)

GitHub Flow is the model used by GitHub itself:
1. Branch from `main`
2. Commit to branch
3. Open PR
4. Get review + CI pass
5. Deploy (optionally from branch, before merge)
6. Merge to `main`
7. Delete branch

It differs from pure TBD in that it doesn't strictly enforce daily integration — branches can live for days or weeks. But it eliminates `develop`, `release/*`, and the Git Flow merging ceremony. It is a pragmatic middle ground.

Use GitHub Flow as the target for Phase 3. From there, tighten branch lifetime expectations until you reach TBD discipline.

---

## 4. Feature Flags Deep Dive

### 4.1 Types of Feature Flags

Martin Fowler's canonical taxonomy (from martinfowler.com/articles/feature-toggles.html):

**Release Toggles**
Purpose: Allow incomplete code to exist in production without being exposed to users. The primary enabler of trunk-based development.

Characteristics:
- Short-lived: days to 2 weeks maximum
- Decided statically per-deploy (not per-request)
- Owned by engineering
- Must be removed after full rollout

Example:
```typescript
if (flags.isEnabled('new-checkout-flow')) {
  return renderNewCheckout();
}
return renderLegacyCheckout();
```

**Ops Toggles (Kill Switches)**
Purpose: Allow operators to disable non-critical functionality during outages or high load. Manual circuit breakers.

Characteristics:
- Can be long-lived (months to years as standing kill switches)
- Require instant reconfiguration without redeployment
- Owned by operations/SRE
- Should be implemented in a distributed config system (Consul, etcd) not a config file

Example: Disable recommendation engine under database load without a deploy.

**Experiment Toggles (A/B Flags)**
Purpose: Route different user cohorts to different code paths to measure behavioral outcomes.

Characteristics:
- Per-request dynamic decisions
- Owned by product
- Live for hours to weeks (until statistical significance)
- Require analytics instrumentation alongside the flag

Example: 50% of users see button color A, 50% see color B; measure conversion.

**Permission Toggles (Entitlement Flags)**
Purpose: Gate features for specific user segments: premium users, beta testers, internal staff.

Characteristics:
- Very dynamic (per-request, per-user)
- Can be long-lived (months to years for premium feature gates)
- Owned by product or billing
- Require a data store that can be queried per-request with low latency

Example: `user.hasPremiumAccess && flags.isEnabled('export-to-excel', user)`

### 4.2 Implementation Patterns (Simplest to Most Capable)

**Level 1: Config file (immediate, zero infrastructure)**

Good for: small teams, early-stage migration, getting started.

```yaml
# config/feature-flags.yaml
flags:
  new-checkout-flow: false
  new-analytics-dashboard: false
  export-to-excel: false
```

```typescript
// flags.ts
import config from '../config/feature-flags.yaml';

export function isEnabled(flagName: string): boolean {
  return config.flags[flagName] ?? false;
}
```

Limitation: Changing a flag requires a deploy. Cannot do per-user targeting. Cannot change flags at runtime.

**Level 2: Environment variables (12-factor, no deploy required for env changes)**

```typescript
export function isEnabled(flagName: string): boolean {
  const envKey = `FEATURE_${flagName.toUpperCase().replace(/-/g, '_')}`;
  return process.env[envKey] === 'true';
}
```

Limitation: Still not per-user. Changing requires restart or platform-level env injection. No UI.

**Level 3: Database-backed with admin UI (most common production pattern)**

Store flags in your application database. Build or use an admin panel. Supports runtime changes, basic user targeting.

```sql
CREATE TABLE feature_flags (
  name        VARCHAR(100) PRIMARY KEY,
  enabled     BOOLEAN NOT NULL DEFAULT FALSE,
  rollout_pct INTEGER DEFAULT 0,     -- 0-100, for percentage rollout
  user_ids    TEXT[],                 -- explicit allowlist
  expires_at  TIMESTAMP,
  owner       VARCHAR(100),
  description TEXT
);
```

**Level 4: Dedicated feature flag service**

For enterprise scale, per-request targeting, experimentation, and cross-team management:

| Tool | Model | Self-Hosted | Best For |
|------|-------|-------------|---------|
| LaunchDarkly | SaaS only | No | Large enterprise, rich SDK ecosystem, experimentation |
| Unleash | Open-source + SaaS | Yes | Teams needing self-hosted, customization, data sovereignty |
| Flagsmith | Open-source + SaaS | Yes | Budget-conscious, simpler requirements, open-source first |
| ConfigCat | SaaS | No | Simpler pricing than LaunchDarkly, good DX |

LaunchDarkly is the market leader with the richest SDK support (50+ languages/platforms) and built-in experimentation. Its costs scale with MAUs and flag count, which can become significant. Unleash is the best self-hosted option with comparable features. Both support the OpenFeature standard (a CNCF project), which allows switching providers without changing application code.

### 4.3 Flag Lifecycle: Creation to Cleanup

The biggest risk with feature flags is not their introduction — it's their failure to be removed. Flags accumulate into a combinatorial test burden and an unreadable codebase. Knight Capital Group's 2012 trading system failure ($460M loss) was partly attributed to an unremoved feature flag that activated legacy code unexpectedly.

**Lifecycle stages:**

```
1. CREATE  →  2. DARK LAUNCH  →  3. CONTROLLED ROLLOUT  →  4. FULL RELEASE  →  5. CLEANUP
```

**Stage 1: Create**
- Define the flag in your flag store
- Record: owner, purpose, type (release/ops/experiment/permission), expiry date
- Create a cleanup ticket in your backlog immediately
- Write code gated by the flag with both paths tested

**Stage 2: Dark Launch (optional but recommended)**
- Flag is off for all users
- Code exists in production, but no users see it
- Internal team can test in production with a permission toggle allowlist

**Stage 3: Controlled Rollout**
- Enable for 1% → 5% → 25% → 50% → 100%
- Monitor error rates and performance metrics at each stage
- Rollback is a flag toggle, not a revert+deploy

**Stage 4: Full Release**
- Flag is on for 100% of users
- At this point, the flag is costing maintenance overhead with zero benefit
- **Begin cleanup immediately** — do not let the code sit in dual-path state

**Stage 5: Cleanup**
- Remove the conditional: inline the "on" path, delete the "off" path
- Delete the flag from the flag store
- Close the cleanup ticket
- This is non-optional. A release toggle that outlives its purpose becomes a permanent ops toggle nobody owns.

**Enforce expiry discipline:**
```typescript
// In test suite: fail if a release toggle is older than 30 days
it('release toggles must not outlive their expiry', () => {
  const flags = releaseToggles.getAll();
  const now = Date.now();
  flags.forEach(flag => {
    if (flag.type === 'release' && flag.expiresAt < now) {
      throw new Error(`Release toggle "${flag.name}" expired ${flag.expiresAt}. Remove it.`);
    }
  });
});
```

### 4.4 Testing With Feature Flags

**The combinatorial explosion problem**

With N binary flags, there are theoretically 2^N states. At 10 flags, that's 1,024 combinations. You cannot test all of them.

The practical solution: you don't need to test all combinations, only meaningful ones.

**Test these states:**
1. **Current production state**: all flags as they are in prod right now
2. **Intended release state**: the flags as they will be when you flip the next flag on
3. **Fallback state**: the new flag turned off (regression protection)
4. **Only test flag interactions when flags are logically related**

```typescript
describe('checkout flow', () => {
  it('works with new-checkout-flow OFF (current prod)', () => {
    flags.override({ 'new-checkout-flow': false });
    // test current behavior
  });

  it('works with new-checkout-flow ON (upcoming release)', () => {
    flags.override({ 'new-checkout-flow': true });
    // test new behavior
  });
});
```

**Flags interact when they share code paths.** If `feature-A` and `feature-B` both modify the same component, test the combination. Otherwise, treat them independently.

**Test infrastructure for flag overrides:**
```typescript
// Make flags injectable/overridable in tests
export class FeatureFlags {
  private overrides: Record<string, boolean> = {};

  isEnabled(name: string, user?: User): boolean {
    if (name in this.overrides) return this.overrides[name];
    return this.source.evaluate(name, user);
  }

  // test helper only
  override(flags: Record<string, boolean>) {
    this.overrides = flags;
  }
}
```

### 4.5 Who Owns Flags

Flag ownership follows flag type:

| Flag Type | Owner | Changes Via |
|-----------|-------|-------------|
| Release toggle | Engineer who built the feature | Code review / PR |
| Ops toggle / kill switch | SRE / Operations | Admin UI, with PagerDuty access |
| Experiment toggle | Product Manager | Flag service admin UI |
| Permission toggle | Product or Billing | Flag service admin UI |

**The critical boundary**: product managers should not be able to modify release toggles without engineering review. Release toggles gate incomplete code — enabling them prematurely can expose in-progress features or broken code paths.

**Establish a flag registry** (even if just a shared doc): flag name, type, owner, creation date, expiry date, cleanup ticket link. Review it monthly.

---

## 5. Decision Framework

### When Git Flow is still appropriate

- Teams shipping boxed software with hard release dates and long QA cycles (e.g., embedded systems, annual major versions)
- Teams maintaining multiple major versions simultaneously with independent support contracts
- Regulatory environments requiring explicit release artifact isolation and sign-off
- Teams with no automated testing (TBD without tests is dangerous; Git Flow at least isolates breakage)

### When to move to TBD

- Teams shipping web software or SaaS products
- Teams wanting to deploy more frequently than weekly
- Teams experiencing merge conflict overhead > 10% of engineering time
- Teams building on microservices (Git Flow coordination across repos is intractable)
- Teams that have invested in or plan to invest in CI/CD automation

### The honest prerequisite list for TBD

You cannot safely adopt TBD without:
1. Automated tests with meaningful coverage
2. Fast, reliable CI (< 20 min, < 2% flaky)
3. Feature flag infrastructure (any level is fine to start)
4. Team agreement that broken trunk is a P0
5. PR review cadence that supports daily integration

Attempting TBD without these produces trunk instability that will cause the team to abandon the process.

---

## 6. Sources

- [Feature Toggles (aka Feature Flags) — Martin Fowler](https://martinfowler.com/articles/feature-toggles.html) — PRIMARY: canonical taxonomy, lifecycle, implementation patterns
- [Branch by Abstraction — Martin Fowler](https://martinfowler.com/bliki/BranchByAbstraction.html) — PRIMARY: technique for large changes without long-lived branches
- [Trunk Based Development — trunkbaseddevelopment.com](https://trunkbaseddevelopment.com/) — PRIMARY: core principles, short-lived branches
- [Branch for Release — trunkbaseddevelopment.com](https://trunkbaseddevelopment.com/branch-for-release/) — PRIMARY: release branching strategy, cherry-pick policy
- [Feature Flags — trunkbaseddevelopment.com](https://trunkbaseddevelopment.com/feature-flags/) — PRIMARY: flags in TBD context
- [Short-Lived Feature Branches — trunkbaseddevelopment.com](https://trunkbaseddevelopment.com/short-lived-feature-branches/) — PRIMARY: 2-day rule, PR flow, CI requirements
- [Continuous Integration — trunkbaseddevelopment.com](https://trunkbaseddevelopment.com/continuous-integration/) — PRIMARY: CI pipeline requirements
- [Long-Lived Branches with Gitflow — Thoughtworks Technology Radar](https://www.thoughtworks.com/radar/techniques/long-lived-branches-with-gitflow) — PRIMARY: "Hold" recommendation with rationale
- [DORA Accelerate State of DevOps 2024](https://dora.dev/research/2024/dora-report/) — PRIMARY: elite performers 2.3x more likely to use TBD
- [Please Stop Recommending Git Flow — George Stocker](https://georgestocker.com/2020/03/04/please-stop-recommending-git-flow/) — MEDIUM: specific failure modes, merge conflict tripling
- [Transitioning from GitFlow to Trunk-Based Development — Aviator](https://www.aviator.co/blog/how-to-transition-from-gitflow-to-trunk-based-development/) — MEDIUM: migration phases and infrastructure requirements
- [Moving from GitFlow to Trunk-Based Development — Shipyard](https://shipyard.build/blog/gitflow-to-trunk-based-development/) — MEDIUM: people/process/testing pillars for migration
- [Elite Performance with Trunk-based Development — LaunchDarkly](https://launchdarkly.com/blog/elite-performance-with-trunk-based-development/) — MEDIUM: DORA correlation with TBD
- [Feature Flag Tools Comparison — Unleash](https://www.getunleash.io/blog/feature-flag-tools-which-should-you-use-with-pricing) — MEDIUM: LaunchDarkly vs Unleash vs alternatives
- [Top LaunchDarkly Alternatives — ConfigCat](https://configcat.com/blog/top-eight-launchdarkly-alternatives/) — LOW: vendor comparison (vendor-authored, treat with bias awareness)
