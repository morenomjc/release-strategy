# OSS Release Strategy: A Developer's Reference

**Primary case study:** Spring Framework (`spring-projects/spring-framework`)
**Audience:** Mid-level developers familiar with git and Maven, new to OSS release processes
**Last updated:** 2026-04-25

---

## Quick Reference

If you are reading a Spring version string right now and want to know what it means:

| Version String | Type | Where to Find It | Use in Production? |
|----------------|------|------------------|--------------------|
| `7.2.0-SNAPSHOT` | Continuous CI build — changes daily | repo.spring.io/snapshot | No |
| `7.2.0-M1` | First milestone — fixed artifact, API in flux | repo.spring.io/milestone | No |
| `7.2.0-M2` | Second milestone — API stabilizing | repo.spring.io/milestone | No |
| `7.2.0-RC1` | Release candidate — feature-complete, API frozen | repo.spring.io/milestone | Only for final compat testing |
| `7.2.0-RC2` | Second RC — only if RC1 had blocking bugs | repo.spring.io/milestone | Only for final compat testing |
| `7.2.0` | GA (General Availability) — production release | Maven Central | Yes |
| `7.2.1` | First patch — bugfixes only, no pre-releases | Maven Central | Yes |
| `7.2.0.RELEASE` | Old-style GA (Spring 5.x and earlier) | Maven Central | Yes (legacy) |

**Key rule:** If a version has any suffix other than a plain `.RELEASE`, it is a pre-release artifact and does not live on Maven Central.

---

## Table of Contents

1. [Branching Strategy](#1-branching-strategy)
2. [How Fixes Propagate](#2-how-fixes-propagate)
3. [Release Terminology and Lifecycle](#3-release-terminology-and-lifecycle)
4. [Release Trains and the BOM](#4-release-trains-and-the-bom)
5. [Concurrent Version Support](#5-concurrent-version-support)
6. [OSS vs. Commercial Support Tiers](#6-oss-vs-commercial-support-tiers)
7. [OSS Ecosystem Comparison](#7-oss-ecosystem-comparison)
8. [Ecosystem Heritage: Why the Terminology Differs](#8-ecosystem-heritage-why-the-terminology-differs)

---

## 1. Branching Strategy

### The Model: Maintenance Branches

Spring Framework does not use git-flow. It does not use strict trunk-based development. It uses a **maintenance-branch model** — a pattern also used by the Linux kernel stable branch and many long-lived Apache projects.

The rules are simple:

- `main` is always the integration point for the *next* unreleased generation.
- Every shipped minor version gets a permanent `N.N.x` maintenance branch.
- Fixes flow **forward only** — older branches merge into newer ones, never backward.
- Fixes for older-only lines are cherry-picked via backport.

### What the Branches Look Like Right Now

As of April 2026, the Spring Framework repository contains these active branches:

```
main (7.1.0-SNAPSHOT)        — next generation feature development
  ^
  | forward merge (git merge --no-ff)
  |
7.0.x (7.0.8-SNAPSHOT)       — current GA production line
  ^
  | forward merge
  |
6.2.x (6.2.19-SNAPSHOT)      — previous generation, active OSS maintenance
  ^
  | cherry-pick (backport)
  |
6.1.x (6.1.22-SNAPSHOT)      — winding down (last OSS commit June 2025)
```

Historical branches (`6.0.x`, `5.3.x`, `5.2.x`, `4.3.x`, and earlier) still exist in the repository and are preserved permanently as read-only references. They never get deleted.

### Branch Naming Convention

The pattern is intentionally minimal:

| Pattern | Meaning |
|---------|---------|
| `main` | Next unreleased generation (or current generation if nothing new is in development) |
| `N.N.x` | All patch releases for minor version N.N — e.g., `7.0.x`, `6.2.x`, `6.1.x` |

The `.x` in `7.0.x` is a **literal string**, not a glob pattern. It is a convention that says "this branch owns all patch-level releases of 7.0." There is no `7.0.1` branch or `7.0.2` branch — those are tags on the `7.0.x` branch.

There are no `feature/` branches, no `develop` branch, no `hotfix/` branches, no `release/` branch prefix. Contributors fork the repository and do their work in their own fork. All public branches are long-lived maintenance branches or infrastructure branches (`docs-build`, `gh-pages`).

### What `main` Is and Is Not

`main` is **not** the current stable release. The official CONTRIBUTING.md is direct about this:

> "Always check out the `main` branch and submit pull requests against it. Backports to prior versions will be considered on a case-by-case basis."

At the time of writing:
- `main` contains `version=7.1.0-SNAPSHOT` in `gradle.properties`
- `7.0.x` contains `version=7.0.8-SNAPSHOT`
- `main` has a stream of commits that are literally `Merge branch '7.0.x'`

This means `main` is always **ahead** of the latest released version. The most recently released stable code lives on the `N.N.x` branch, not on `main`.

### PR Submission Target

All contributor pull requests target `main`. A small number of PRs from maintainers target maintenance branches directly for targeted fixes. Backports from main to older lines are handled by the maintainer team after the original PR merges, not by contributors.

---

## 2. How Fixes Propagate

### Forward Merge (for fixes that apply to all active lines)

When a bug exists in all active versions, the fix is committed to the **oldest affected branch** and then merged forward:

```bash
# Step 1: Fix the bug on the oldest affected branch
git checkout 6.2.x && git pull
# ... make the fix ...
git add . && git commit -m "Fix unsafe static resource locations"
git push origin 6.2.x

# Step 2: Merge forward into the next branch
git checkout 7.0.x && git pull
git merge --no-ff 6.2.x
git push origin 7.0.x

# Step 3: Merge forward into main
git checkout main && git pull
git merge --no-ff 7.0.x
git push origin main
```

The `--no-ff` flag (no fast-forward) is important — it creates an explicit merge commit even when fast-forward would be possible. This preserves the history of which commits came from which branch.

**Evidence from the live repository.** Sampling commits on `main` in April 2026 shows a continuous stream of `Merge branch '7.0.x'` commits. The same fix commits (e.g., "Warn against unsafe static resource locations", "Add doOnDiscard in MultipartHttpMessageReader") appear in the same order on `6.2.x`, `7.0.x`, and `main` — confirming the flow.

### Backport (for fixes that apply only to older lines)

When a fix is relevant to an older line but not to newer ones — for example, a regression that was introduced and removed in the same minor version — the team uses cherry-picks.

Spring maintains a **backport bot**. A maintainer adds a label like `for: backport-to-6.1.x` to an issue, which triggers the bot to open a backport issue. A maintainer then performs the cherry-pick:

```bash
git checkout 6.1.x && git pull
git cherry-pick c0ffee456 --edit
# Commit message should reference:
# Closes gh-4567 (the backport issue)
# See gh-1234 (the original issue)
git push origin 6.1.x
```

### Why Fixes Never Flow Backward

Merging a newer branch into an older one would risk introducing unreleased API changes or new features into a maintenance line. The forward-only policy is a hard rule: `6.2.x` fixes flow into `7.0.x`, never the reverse. This keeps maintenance branches stable and predictable.

---

## 3. Release Terminology and Lifecycle

### The Full Lifecycle Sequence

A new minor version of Spring follows this path from first commit to end-of-life:

```
[Development begins on main or N.N.x branch]
         |
         v
  7.2.0-SNAPSHOT       -- continuous CI builds, published to snapshot repo, mutable
         |
         v
    7.2.0-M1           -- first milestone: planned iteration checkpoint
         |
         v
    7.2.0-M2           -- second milestone: API still in flux
         |
         v
    7.2.0-M3           -- (optional) third milestone
         |
         v
    7.2.0-RC1          -- feature-complete, API frozen
         |               [if blocking bugs found: RC2, RC3...]
         v
    7.2.0-RC2          -- (if needed)
         |
         v
      7.2.0            -- GA: published to Maven Central, OSS support begins
         |
         v
      7.2.1            -- first patch: bugfixes only, goes straight from SNAPSHOT to GA
         |
         v
      7.2.2            -- second patch
         |
         v
       ...
         |
         v
  [OSS support window ends -- typically 12-18 months after GA]
         |
         v
  [Commercial support available for enterprise subscribers]
         |
         v
  [Commercial support ends -- branch preserved read-only forever]
```

While `7.2.x` is being patched, the next generation runs in parallel:
```
7.3.0-SNAPSHOT --> 7.3.0-M1 --> ... --> 7.3.0 GA
```

### SNAPSHOT

A SNAPSHOT is a **mutable, continuous CI build**. The version string `7.2.0-SNAPSHOT` does not refer to a fixed artifact. It resolves to "the latest successful build of the 7.2.0 development line."

**Key facts:**
- Published on every successful CI build to `https://repo.spring.io/snapshot`
- Never published to Maven Central (Maven Central enforces immutability; SNAPSHOTs are mutable by definition)
- When you resolve `7.2.0-SNAPSHOT` today and resolve it again tomorrow, you may get different bytecode
- Maven and Gradle treat SNAPSHOT coordinates with special re-check logic: by default, the build tool checks the remote repository once per day for a newer SNAPSHOT (Maven) or after a configurable TTL (Gradle)

**Real examples from the live repository:**
```
7.1.0-SNAPSHOT   -- what main is building toward right now
7.0.8-SNAPSHOT   -- the next patch of the current GA line (7.0.x branch)
6.2.19-SNAPSHOT  -- the next patch of the maintenance line (6.2.x branch)
```

**Who should use SNAPSHOTs:** Contributors testing their changes against unreleased code, or Spring's own CI pipelines doing integration testing. Production applications should never depend on a SNAPSHOT.

**Stability contract:** None. Breaking changes can appear at any time. You opt in to instability when you add the snapshot repository.

### Milestone (M1, M2, M3...)

A Milestone is a **fixed, immutable pre-release checkpoint**. Unlike a SNAPSHOT, `7.2.0-M1` refers to exactly one artifact that never changes after publication.

Milestones signal: "Here is a concrete artifact you can evaluate. The API shape is becoming visible. We want early-adopter feedback before we commit to RC."

**Key facts:**
- Published to `https://repo.spring.io/milestone` (not Maven Central)
- Immutable once published — the artifact at `7.2.0-M1` never changes
- Sequential numbering starting at 1: `-M1`, `-M2`, `-M3`
- No predetermined maximum — Spring 6.0 used M1 through M5; simpler cycles use M1-M2
- API may still change between milestones, especially early ones

**Real examples from the Spring 6.0 release cycle (HIGH confidence):**
```
6.0.0-M1   -- first public milestone of Spring 6 (early 2022)
6.0.0-M2
6.0.0-M3
6.0.0-M4
6.0.0-M5
6.0.0-RC1  -- first release candidate
6.0.0-RC2
6.0.0-RC3
6.0.0-RC4
6.0.0      -- GA (November 2022)
```

Spring Boot 3.0 (which depends on Spring Framework 6.0) followed the same structure:
```
3.0.0-M1 through 3.0.0-M5
3.0.0-RC1, RC2
3.0.0      -- GA (November 2022, same day as Spring Framework 6.0)
```

**Who should use Milestones:** Library authors who need to test compatibility early; teams tracking pre-release versions in non-production environments; anyone providing API feedback. Explicitly requires adding the milestone repository to your build config.

**Stability contract:** Limited. API surface is emerging but subject to change. Do not ship milestones in production dependencies.

### Release Candidate (RC1, RC2...)

A Release Candidate is **feature-complete and API-frozen**. The team believes this artifact could become the GA release if no blocking issues are found.

| Attribute | Milestone | Release Candidate |
|-----------|-----------|------------------|
| Feature completeness | Partial | Complete |
| API stability | Stabilizing, may change | Frozen — no new APIs |
| Breaking changes allowed | Yes (with notice) | No |
| Purpose | Shape feedback, iteration checkpoints | Final validation, bug hunting |
| Typical time to next stage | Weeks to months | Days to weeks |

If a blocking bug is found in RC1, it is fixed and RC2 is released. If RC2 is clean, it becomes GA.

**Key facts:**
- Published to `https://repo.spring.io/milestone` (same repo as milestones)
- Suffix is `-RC` followed by a sequential integer: `-RC1`, `-RC2`
- Not on Maven Central
- The same artifact hash from a clean RC may become the GA binary (in Maven terms, a new artifact is published to Central without the RC suffix)

**Real examples:**
```
6.1.0-RC1
6.1.0-RC2
6.1.0      -- GA (November 2023)

5.3.0-RC1
5.3.0-RC2
5.3.0      -- GA (October 2020)
```

**Who should use RCs:** Library authors and framework integrators doing final compatibility testing. Production use is still not recommended by Spring but is lower-risk than milestones.

### GA (General Availability)

GA is the **production-ready release**. In Spring's versioning, a GA release carries no suffix:

```
7.2.0    -- GA
```

This is the only artifact type that lands on Maven Central. It is also immutable (Maven Central enforces this globally). Once `7.2.0` is on Maven Central, it never changes. A critical bug in `7.2.0` is addressed in `7.2.1`, not by replacing `7.2.0`.

**Historical note:** Spring 5.x and earlier used a `.RELEASE` suffix for GA artifacts (e.g., `5.3.39.RELEASE`). Starting with Spring Framework 6.0, the suffix was dropped. If you see `.RELEASE` in a dependency declaration, it is a GA artifact from an older generation.

**Published to:** Maven Central (`https://repo1.maven.org/maven2/`)

### Patch Releases

A patch release increments the third number (the PATCH in MAJOR.MINOR.PATCH):

```
7.2.0   -- initial GA
7.2.1   -- first patch (bugfixes only)
7.2.2   -- second patch
...
7.2.14  -- realistic for a long-lived Spring maintenance line
```

**Key facts:**
- Bugfixes and security patches only — no new features, no API additions
- No milestones or RCs — patches go directly from SNAPSHOT to GA:
  ```
  7.2.1-SNAPSHOT  -->  7.2.1 (GA)
  ```
  There is no such thing as `7.2.1-M1` or `7.2.1-RC1`
- Active maintenance lines receive patches approximately monthly (roughly 4-6 week cadence)
- Security CVEs can trigger out-of-cycle patches within days

**Real cadence from the live repository (April 2026):**
```
2026-04-17 | 7.0.7 released | 6.2.18 released
2026-03-13 | 7.0.6 released | 6.2.17 released
2026-02-12 | 7.0.4 released | 6.2.16 released
2026-01-15 | 7.0.3 released
2025-12-11 | 7.0.2 released | 6.2.15 released
```

Note that multiple lines ship patches on the same day — this is intentional coordination.

---

## 4. Release Trains and the BOM

### What a Release Train Is

Spring does not release a single library. It releases an ecosystem of dozens of interdependent libraries. A **release train** is a coordination mechanism: multiple related projects agree to release together on a synchronized schedule with pinned compatible versions.

**Spring Boot is the conductor of the Spring release train.** Each Spring Boot release defines a **BOM** (Bill of Materials) — a Maven POM that pins the compatible versions of all member projects.

When Spring Boot `3.3.0` ships, it pins:
- Spring Framework `6.1.8`
- Spring Data `3.3.0` (Quince train)
- Spring Security `6.3.0`
- Spring Batch `5.1.0`
- Micrometer `1.13.0`
- Reactor Core `3.6.6`
- Tomcat `10.1.x`
- ... and dozens more

### Using the BOM in Practice

```xml
<!-- Maven: import the Spring Boot BOM via parent -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.0</version>
</parent>

<!-- Now you can declare Spring Framework without specifying a version -->
<!-- The BOM provides spring-core:6.1.8 automatically -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-core</artifactId>
    <!-- version omitted -- Spring Boot BOM manages this -->
</dependency>
```

```groovy
// Gradle: import via platform dependency
dependencies {
    implementation platform('org.springframework.boot:spring-boot-dependencies:3.3.0')
    implementation 'org.springframework:spring-core'  // version resolved by BOM
}
```

### What "We're on Spring Boot 3.3" Really Means

When a team says "we're on Spring Boot 3.3," they implicitly mean the entire constellation of compatible libraries — Spring Framework 6.1.x, Spring Security 6.3.x, Spring Data 3.3.x, and so on. "Upgrade to Spring Boot 3.4" is not just changing one version number. It is upgrading the entire train.

### Spring Data Codename Trains (Historical Context)

Before Spring Boot became the canonical train conductor, **Spring Data** used codename-based release trains. The names were London Underground stations in alphabetical order: Lovelace (aligned with Spring Boot 2.6.x), Moore (2.7.x), Neumann (3.0.x), O'Brien (3.1.x), Pascal (3.2.x / Spring Framework 6.1), Quince (3.3.x). You may encounter these codenames in older documentation or blog posts.

### The `7.0.x` and `6.2.x` Lines Coexisting

At any given time, multiple minor versions of Spring Framework are in active use. Spring Boot coordinates this by maintaining multiple concurrent release lines:

```
Spring Boot 3.4.x  -- uses Spring Framework 6.2.x
Spring Boot 3.3.x  -- uses Spring Framework 6.1.x (OSS EOL Nov 2025)
Spring Boot 3.5.x  -- in development, uses Spring Framework 6.2.x
```

Teams on older Spring Boot versions continue receiving patch releases on that line until its OSS EOL, without being forced to upgrade their Spring Framework generation.

---

## 5. Concurrent Version Support

### The Four States of a Branch

At any given time, every `N.N.x` branch is in one of four states:

| State | Description | Example (April 2026) | What You Get |
|-------|-------------|---------------------|--------------|
| **Active development** | Next unreleased generation | `main` (7.1.0-SNAPSHOT) | No releases yet — only SNAPSHOTs |
| **Current GA** | Latest released generation, fully active | `7.0.x` | Full bugfix + security patches, monthly |
| **Active maintenance** | Previous generation, OSS support | `6.2.x` | Bugfix + security patches, monthly |
| **Winding down** | Support ending soon | `6.1.x` | Last OSS commit June 2025; branch frozen |
| **EOL / archived** | Support ended | `6.0.x`, `5.3.x` | No new commits; read-only historical reference |

### Branch Lifetime

1. A maintenance branch is cut from `main` when a new minor release enters stabilization (usually around the RC stage)
2. The branch is active for the OSS support window — typically 12-18 months after GA
3. After OSS EOL, commercial support is available (see next section)
4. The branch is **never deleted** — it remains permanently in the repository as a historical reference

### Snapshot Versions as Intent Markers

The `version` property in `gradle.properties` always carries the next planned release as a SNAPSHOT. This makes it easy to verify what a branch is working toward:

```
main          gradle.properties: version=7.1.0-SNAPSHOT   -> building toward 7.1.0
7.0.x         gradle.properties: version=7.0.8-SNAPSHOT   -> next patch will be 7.0.8
6.2.x         gradle.properties: version=6.2.19-SNAPSHOT  -> next patch will be 6.2.19
```

If you checkout a maintenance branch and look at `gradle.properties`, the version tells you exactly what the next release will be called.

---

## 6. OSS vs. Commercial Support Tiers

### Two-Tier Support Model

Spring operates a two-tier support structure:

**OSS (Open Source) Support — Free**
- Bugs and security fixes are published publicly to Maven Central
- All patch releases are freely available to anyone
- Support ends when the Spring team marks the version as End-of-Life
- EOL dates are published on https://spring.io/projects/spring-framework#support

**Commercial Support — Paid (VMware Spring Runtime / Broadcom Tanzu)**
- Extends the maintenance window significantly beyond OSS EOL
- Security patches continue to be produced and delivered to enterprise subscribers
- Not publicly available — requires a commercial support contract
- Relevant for organizations that cannot upgrade on the OSS schedule

### Support Windows

| Spring Framework Version | Initial GA | OSS EOL | Commercial EOL |
|--------------------------|-----------|---------|----------------|
| 5.3.x | Oct 2020 | Dec 2024 | Dec 2027 |
| 6.0.x | Nov 2022 | Aug 2024 | Dec 2026 |
| 6.1.x | Nov 2023 | ~Aug 2025 | ~Dec 2027 |
| 6.2.x | Nov 2024 | ~Aug 2026 | ~Dec 2028 |
| 7.0.x | Nov 2025 | ~12-18 mo. after GA | ~3 years after GA |

*Note: Exact dates should be verified against https://spring.io/projects/spring-framework#support*

### What Changes at OSS EOL

Before OSS EOL: Patch releases are published publicly on Maven Central. Anyone running `mvn dependency:get` gets the latest security fix for free.

After OSS EOL, during commercial window: Patches continue to be produced but are delivered through commercial channels. The artifact on Maven Central is frozen at the last OSS-era patch version. Organizations on an unsupported OSS version are getting no security patches.

After commercial EOL: No patches of any kind. Teams on these versions are entirely self-reliant.

### Which Versions Are Patched Simultaneously

Spring typically patches two to three minor versions concurrently during the OSS window. From the release cadence data:

```
April 2026 patches:
  7.0.7 (current GA -- full bugfix + security)
  6.2.18 (previous generation -- full bugfix + security)
  [6.1.x frozen -- last OSS patch was ~June 2025]
```

The practical implication: being one minor version behind the current GA is acceptable. Being two or more minor versions behind puts you outside the free patch window and increases upgrade debt.

---

## 7. OSS Ecosystem Comparison

Spring is one approach. Here is how it maps to other major OSS projects that developers commonly encounter.

### Apache Kafka

**Branch model:** `trunk` as the primary development branch (not `main`). Maintenance branches use `N.N` (e.g., `3.7`, `3.8`) — no `.x` suffix.

**Release stages:** Only one pre-release stage — RC. There are no milestones, no alpha, no beta.
```
3.8.0-SNAPSHOT  -->  3.8.0-rc1  -->  3.8.0-rc2  -->  3.8.0 (GA)
```

**Governance gate:** Apache Software Foundation (ASF) projects require a **PMC vote** before any release is official. A release candidate is voted on by project committers. Any PMC member can block the release with a formal -1 veto. This community governance step has no equivalent in Spring's maintainer-driven model.

**Support:** Approximately 2-3 concurrent minor lines, no formal LTS. No commercial support tier built into the project (third-party vendors like Confluent offer commercial support separately).

### Quarkus

**Branch model:** `main` + `N.N` maintenance branches (e.g., `3.15`, `3.20`). No `.x` suffix.

**Release stages:** More granular than Spring's Milestone/RC, less granular than Kubernetes:
```
3.27.0.Alpha1  -->  3.27.0.Alpha2  -->  3.27.0.Beta1  -->  3.27.0.CR1  -->  3.27.0 (Final)
```

Note: `CR` means "Candidate Release" — the Red Hat/JBoss term for RC. `Final` is the GA label instead of a plain version number. This is JBoss heritage.

**LTS:** Quarkus explicitly designates LTS releases approximately every 6 months. As of April 2026, active LTS lines are 3.15, 3.20, and 3.27. Non-LTS minor releases are maintained for approximately 4 weeks (until the next minor ships). This is a much faster cadence than Spring — Quarkus ships roughly one new minor per month.

**Emergency branches:** Quarkus creates out-of-band branches like `3.20.2-emergency` for critical CVE patches that cannot wait for the next scheduled minor.

**Key difference from Spring:** Quarkus's LTS designation is explicit in the project documentation. Spring's effective "LTS" is the OSS support window, but it is not labeled as such in version strings — you must check the support policy page.

### Kubernetes

**Branch model:** `main` + `release-X.Y` maintenance branches (e.g., `release-1.33`, `release-1.34`).

**Release stages:**
```
v1.36.0-alpha.1  -->  v1.36.0-beta.0  -->  v1.36.0-rc.0  -->  v1.36.0 (GA)
```

These are proper semver pre-release identifiers (the part after the hyphen in semver), not Maven version qualifiers. The difference matters: Maven/Gradle sort `7.2.0-M1` differently than semver would; Kubernetes tags are consumed by Kubernetes tooling which understands semver natively.

**Strict N-2 support policy:** Exactly three minor versions are supported simultaneously. As of April 2026: 1.33, 1.34, 1.35. When 1.36 ships, 1.33 reaches EOL immediately. This is the clearest support window policy of any project covered here — you always know exactly which three versions are supported.

**Time-boxed releases:** Kubernetes ships exactly three minor versions per year. Features that miss a window wait for the next minor. This is different from Spring, where minor releases happen when they are ready (roughly once or twice per year but not on a strict calendar).

**Monthly patch cadence:** Patch releases on approximately the 28th of each month for all three supported minors simultaneously.

### Next.js (Frontend Reference Point)

**Branch model:** `canary` is the primary development branch — not `main`. There are no per-minor maintenance branches. "Maintenance" for Next.js means "we will accept critical security PRs" for older majors, but the expectation is that teams upgrade to the current version.

**Release stages:** Not pre-release versions with pinnable coordinates — distribution channels:
- `npm install next@canary` — continuous pre-release builds (analogous to SNAPSHOT)
- `npm install next@rc` — release candidate channel
- `npm install next` — stable (latest channel, analogous to GA)

This is a fundamentally different model than Maven coordinates. You opt in to a channel at install time rather than pinning an exact pre-release version string in your dependency file.

**Support:** One current stable major. Older majors get critical security patches only. No LTS model.

### Full Comparison Table

| Project | Primary Dev Branch | Maintenance Branch Pattern | Pre-release Labels | GA Label | LTS? | Concurrent Supported Versions |
|---|---|---|---|---|---|---|
| **Spring Framework** | `main` | `N.N.x` (e.g. `7.0.x`) | SNAPSHOT -> M1/M2 -> RC1 | `7.0.0` (plain) | Via Broadcom commercial | ~2-3 minor lines |
| **Apache Kafka** | `trunk` | `N.N` (e.g. `3.8`) | RC1/RC2 only | `3.8.0` | No | ~2-3 minor lines |
| **Quarkus** | `main` | `N.N` (e.g. `3.20`) | Alpha1 -> Beta1 -> CR1 | `3.27.0` (no `.Final` in newer releases) | Yes (explicit OSS LTS) | LTS lines + current minor |
| **Kubernetes** | `main` | `release-N.N` | alpha -> beta -> rc | `v1.35.0` | No (N-2 policy) | 3 minors (N-2) |
| **Next.js** | `canary` | None | canary channel | `16.2.4` (latest channel) | No | 1 current major |

### Equivalent Pre-release Stages Across Projects

| Spring Concept | Quarkus / Hibernate | Apache Kafka | Kubernetes |
|----------------|--------------------|--------------|-----------:|
| `7.2.0-SNAPSHOT` | — (no equivalent) | `3.8.0-SNAPSHOT` | `v1.36.0-dev` builds |
| `7.2.0-M1` | `3.27.0.Alpha1` | — (no milestone stage) | `v1.36.0-alpha.1` |
| `7.2.0-M2` / `M3` | `3.27.0.Alpha2` / `Beta1` | — | `v1.36.0-beta.0` |
| `7.2.0-RC1` | `3.27.0.CR1` | `3.8.0-rc1` | `v1.36.0-rc.0` |
| `7.2.0` | `3.27.0.Final` | `3.8.0` | `v1.36.0` |

---

## 8. Ecosystem Heritage: Why the Terminology Differs

The terminology differences across OSS projects are not arbitrary. They are the sediment of organizational heritage.

### Spring / Pivotal / VMware Heritage

Spring's vocabulary — SNAPSHOT, Milestone, Release Candidate, General Availability — comes from the Java enterprise world and specifically from the organizational culture of Pivotal (now VMware/Broadcom). "Milestone" maps to the team's sprint/iteration planning language. Each milestone corresponds to a planned sprint boundary — which is why they chose a neutral term ("milestone") rather than a quality-implying term ("alpha"). A milestone might be high quality but incomplete; calling it "alpha" would imply otherwise.

The Maven ecosystem's SNAPSHOT convention predates Spring's use of it. Maven 2 introduced `-SNAPSHOT` semantics circa 2004, and the entire JVM ecosystem inherited it. Spring's choice to build its CI artifact strategy around Maven conventions made SNAPSHOTs a natural fit.

### Red Hat / JBoss Heritage (Quarkus, Hibernate, WildFly)

Red Hat's projects use `Alpha`, `Beta`, `CR` (Candidate Release), and `Final`. This vocabulary traces to JBoss AS (now WildFly), which Red Hat acquired in 2006. `CR` instead of `RC`, `Final` instead of a plain version number — these are signals of Red Hat/JBoss organizational origin. When you see `CR1` and `Final`, you are in the JBoss heritage lineage regardless of the project name.

### Apache Software Foundation Heritage (Kafka, Camel, Commons)

ASF projects tend to use minimal pre-release stages because the governance model is the differentiator, not the version label. The ASF release vote is the "release candidate" — once a community vote passes, that artifact becomes the official release. Some ASF projects do publish alpha and beta artifacts to Maven Central (unlike Spring, which keeps all pre-GA artifacts off Central), but the release management energy is spent on the governance process rather than the pre-release labeling.

The use of `trunk` as the primary branch name (instead of `main`) in some ASF projects also reflects older SVN conventions that predate git's `main` convention. The Apache Kafka project uses `trunk` precisely because of this heritage.

### Linux / Systems Heritage (Kubernetes)

Kubernetes inherits vocabulary from the Linux kernel development model and the broader systems programming world where `alpha`, `beta`, `rc`, and `stable` are conventional. Using semver pre-release identifiers (`v1.36.0-alpha.1`) rather than Maven version qualifiers reflects that Kubernetes is a Go project distributed as binaries and Helm charts, not a library consumed via Maven Central. The tooling consuming these version strings is different, so the version string format is different.

The `release-N.N` branch naming convention also reflects systems project conventions more than Java library conventions.

### Why This Matters for Your Team

When you encounter `3.27.0.CR1` in a Quarkus changelog, you now know it means the same thing as `7.2.0-RC1` in Spring. When you see a Kafka `3.8.0-rc1`, you know there were no "milestone" stages — Kafka goes straight to RC. When Kubernetes ships `v1.36.0-alpha.1`, you know it is much earlier in the cycle than what Spring would call a milestone, and that the alpha/beta/rc stages will each get multiple iterations.

The mental model is not "learn each project's terms in isolation." The mental model is: "understand your project's heritage, and you can predict its conventions."

---

## Sources

All Spring-specific claims are verifiable from these primary sources:

- **CONTRIBUTING.md (Spring Framework main branch):** https://raw.githubusercontent.com/spring-projects/spring-framework/main/CONTRIBUTING.md
- **Git branch management wiki:** https://github.com/spring-projects/spring-framework/wiki/Git-branch-management
- **Spring Framework Versions wiki (support policy):** https://github.com/spring-projects/spring-framework/wiki/Spring-Framework-Versions
- **Spring Framework support page:** https://spring.io/projects/spring-framework#support
- **Spring Boot support page:** https://spring.io/projects/spring-boot#support
- **Spring artifact repositories:** https://repo.spring.io
- **Maven SNAPSHOT documentation:** https://maven.apache.org/guides/getting-started/
- **Apache Kafka releases:** https://github.com/apache/kafka/releases
- **Quarkus releases:** https://github.com/quarkusio/quarkus/releases
- **Kubernetes release cadence:** https://kubernetes.io/releases/release/
