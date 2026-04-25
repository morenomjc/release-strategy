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
    font-size: 0.85rem;
  }
  table {
    width: 100%;
    font-size: 0.9rem;
  }
  th {
    background: #1a5276;
    color: white;
  }
  .tag-snapshot { background:#e8e8e8; padding:0.2em 0.6em; border-radius:4px; }
  .tag-milestone { background:#fff3cd; padding:0.2em 0.6em; border-radius:4px; }
  .tag-rc { background:#cce5ff; padding:0.2em 0.6em; border-radius:4px; }
  .tag-ga { background:#d4edda; padding:0.2em 0.6em; border-radius:4px; font-weight:bold; }
  footer {
    font-size: 0.7rem;
    color: #aaa;
  }
---

<!-- _paginate: false -->
<!-- _footer: "" -->

# How Open Source Projects Release

### Branching, versioning, and the mental model behind Spring's release process

&nbsp;

- This talk gives you a **vocabulary and mental model**, not a how-to guide
- After this, you will know exactly what `7.2.0-M3` vs `7.2.0-RC1` vs `7.2.0` means
- You will understand why `7.0.x` and `6.2.x` both exist in the repo right now
- We use **Spring** as the primary case study, then compare to the broader ecosystem

<!--
SPEAKER NOTE: Open with a quick question to the audience: "Has anyone looked at a Spring changelog and wondered why there are M1, M2, M3, RC1, RC2 before the final release? Or wondered what branch to look at for the current stable code?" The goal is shared vocabulary — everyone on the team should be able to have the same conversation when they see a version string.
-->

---

# What Does This Version Mean?

Five version strings from the same project — what is the difference?

```
7.2.0-SNAPSHOT    7.2.0-M1    7.2.0-RC1    7.2.0    7.2.1
```

&nbsp;

- All five exist in the same project at different points in time
- **Two** of them are on Maven Central — which ones?
- **Three** require special repository configuration to even resolve
- Understanding the answer requires knowing the release lifecycle

&nbsp;

> **Answer:** Only `7.2.0` (GA) and `7.2.1` (patch) are on Maven Central.
> Everything else lives at `repo.spring.io` only.

<!--
SPEAKER NOTE: Let this slide breathe. Most developers have vague intuitions about these suffixes. Wait for answers to the Maven Central question before revealing. The "two on Maven Central" hook is a good engagement device.
-->

---

# Three Ways to Manage Code

**Git Flow** — feature branches + develop + release branches + hotfixes
- Complex, designed for teams with long QA cycles
- Not what Spring uses

**Trunk-based development** — everyone commits to one branch; short-lived feature flags
- Fast deployment cadence
- Not what Spring uses either

**Maintenance-branch model** — one primary branch for the next generation + long-lived `N.N.x` branches per released minor
- Designed for supporting multiple versions in production simultaneously
- **This is Spring's model**

<!--
SPEAKER NOTE: Don't spend long here — this is context, not the destination. The key point is that Spring's model is its own thing, not git-flow (a common misconception).
-->

---

# Spring's Actual Branches Right Now

```
main (7.1.0-SNAPSHOT)       ← next unreleased generation
  ↑ git merge --no-ff
7.0.x (7.0.8-SNAPSHOT)      ← current GA production line
  ↑ git merge --no-ff
6.2.x (6.2.19-SNAPSHOT)     ← previous generation, active OSS maintenance
  ↑ cherry-pick (backport)
6.1.x (6.1.22-SNAPSHOT)     ← winding down (last OSS commit June 2025)

6.0.x · 5.3.x · 5.2.x ...  ← EOL, preserved read-only forever
```

&nbsp;

- The `.x` in `7.0.x` is a **literal string** — there is no `7.0.1` branch; patches are tags
- **`main` is not current stable** — it is the next unreleased generation
- Branches are **never deleted** — historical branches exist going back to Spring 3.x

<!--
SPEAKER NOTE: The key insight: if you checkout `main` right now you get unreleased features that may change. The current stable release lives on `7.0.x`. Also: the SNAPSHOT version in each branch tells you exactly what the next release will be called.
-->

---

# Fix Once, Propagate Forward

Fixes go in at the **oldest affected branch**, then merge forward. Never backward.

```
6.2.x ──── commit "Fix unsafe static resource" ──── →
             ↓ git merge --no-ff
7.0.x ─────────────────── Merge branch '6.2.x' ──── →
                            ↓ git merge --no-ff
main  ──────────────────────────── Merge branch '7.0.x' ──── →
```

&nbsp;

| | |
|---|---|
| All active lines | Fix on oldest branch, merge forward |
| Older-only fix | Cherry-pick via backport bot (`for: backport-to-6.1.x` label) |
| Fixes never go backward | Merging `7.0.x` → `6.2.x` would risk introducing unreleased APIs |

<!--
SPEAKER NOTE: Evidence is visible in the live commit history on `main` — a continuous stream of `Merge branch '7.0.x'` commits. The same individual fix commits appear in the same order on all three branches. This is the forward merge pattern in action.
-->

---

# From First Commit to End of Life

```
SNAPSHOT  →  M1  →  M2  →  [M3]  →  RC1  →  [RC2]  →  GA  →  patches  →  EOL
```

| Stage | Example | Stability | Where |
|---|---|---|---|
| SNAPSHOT | `7.2.0-SNAPSHOT` | Mutable, CI build | repo.spring.io/snapshot |
| Milestone | `7.2.0-M1` | Fixed, API in flux | repo.spring.io/milestone |
| Release Candidate | `7.2.0-RC1` | Feature-complete, API frozen | repo.spring.io/milestone |
| GA | `7.2.0` | Production-ready | **Maven Central** |
| Patch | `7.2.1` | Bugfix only, no pre-releases | **Maven Central** |

&nbsp;

**Rule:** Patches skip all pre-release stages → `7.2.1-SNAPSHOT` becomes `7.2.1` directly.
There is no such thing as `7.2.1-M1`.

---

# SNAPSHOT: Always the Latest Build

`7.2.0-SNAPSHOT` is **not a fixed artifact** — it means "latest successful CI build of 7.2.0"

&nbsp;

- Resolves to **different bytecode** tomorrow than it does today
- Maven re-checks the remote once per day by default; Gradle has a configurable TTL
- **Never on Maven Central** — Central enforces immutability; SNAPSHOTs are mutable by definition
- Requires explicit repository config to use:

```xml
<repositories>
  <repository>
    <id>spring-snapshots</id>
    <url>https://repo.spring.io/snapshot</url>
    <snapshots><enabled>true</enabled></snapshots>
  </repository>
</repositories>
```

&nbsp;

> **Who should use SNAPSHOTs:** Contributors testing unreleased changes. Not production applications.

<!--
SPEAKER NOTE: The "mutable" point is the key concept. If a build using a SNAPSHOT starts failing with no code changes, the SNAPSHOT may have been updated upstream. This is a real debugging scenario that trips up developers who don't know about SNAPSHOT semantics.
-->

---

# Milestones and Release Candidates

**Milestone (M1, M2...):** Fixed immutable artifact. API still in flux.
*"Here is a stake in the ground for early adopter feedback."*

**RC (RC1, RC2...):** Feature-complete, API frozen.
*"We believe this could be GA if no blocking bugs are found."*

| | Milestone | Release Candidate |
|---|---|---|
| Features complete? | No | **Yes** |
| API stable? | Stabilizing | **Frozen** |
| Breaking changes? | Possible | No |
| Purpose | Shape API feedback | Bug hunting |
| Typical cycle time | Weeks–months | Days–weeks |
| Repository | repo.spring.io/milestone | repo.spring.io/milestone |

&nbsp;

Spring 6.0 used **M1 through M5** before RC1 — more complex cycles take more milestones.

<!--
SPEAKER NOTE: Practical implication for library authors: start compatibility testing at M2 or M3 (not M1 — too early). RC1 is your last chance to find blockers before GA ships.
-->

---

# GA and Patches: The Stable World

**GA — General Availability**
- Plain version string: `7.2.0` (no suffix)
- The **only** artifact type published to Maven Central
- Immutable forever — a bug in `7.2.0` is addressed in `7.2.1`, not by replacing `7.2.0`
- Historical note: Spring 5.x used `.RELEASE` suffix (`5.3.39.RELEASE`). Spring 6.0 dropped it.

&nbsp;

**Patch releases**
- Increment the third number: `7.2.1`, `7.2.2`...
- **Bugfixes and security patches only** — no new APIs, no feature additions
- Go **directly** from SNAPSHOT to GA — no milestones, no RCs:
  ```
  7.2.1-SNAPSHOT  ──────────────────────►  7.2.1 (GA)
  ```
- Active lines release patches approximately monthly — coordinated across all active branches

---

# Release Trains and the BOM

Spring releases an **ecosystem**, not a single library. Spring Boot is the coordinator.

```
Spring Boot 3.3.0 (the conductor)
    │
    ├── Spring Framework  6.1.8
    ├── Spring Security   6.3.0
    ├── Spring Data       3.3.0 (Quince)
    ├── Spring Batch      5.1.0
    ├── Micrometer        1.13.0
    ├── Reactor Core      3.6.6
    └── Tomcat            10.1.x
```

&nbsp;

- **BOM** (Bill of Materials): a Maven POM that pins compatible versions across the ecosystem
- `"We're on Spring Boot 3.3"` = the entire constellation above, coordinated
- `"Upgrade to Spring Boot 3.4"` = upgrading the **whole train**, not one artifact

<!--
SPEAKER NOTE: This is often the "aha" moment. The BOM is doing a lot of invisible work. Teams that manually manage individual Spring versions often hit mysterious compatibility issues that the BOM prevents.
-->

---

# Same Concept, Different Vocabulary

All major OSS projects have the same underlying stages — the labels differ by heritage.

| Stage | Spring | Quarkus | Kafka | Kubernetes |
|---|---|---|---|---|
| Mutable CI | SNAPSHOT | — | SNAPSHOT | dev build |
| Early feedback | M1 | Alpha1 | — *(skips to RC)* | alpha.1 |
| API stabilizing | M2/M3 | Beta1 | — | beta.0 |
| Feature-complete | RC1 | **CR1** | rc1 | rc.0 |
| Stable | `7.2.0` | `3.27.0` *(was `.Final`)* | `3.8.0` | `v1.36.0` |

&nbsp;

`CR` = Candidate Release (JBoss/Red Hat term for RC)
Kafka skips milestone stages entirely — RC-only pre-release
Kubernetes uses semver pre-release identifiers, not Maven version qualifiers

---

# What Makes Each Project Distinctive

**Apache Kafka**
- Only RC stage — no milestones, no alpha/beta
- Requires a **PMC community vote** to publish — any committer can veto with a -1
- Uses `trunk` instead of `main` (SVN heritage)

**Quarkus**
- Explicit **LTS designations** every ~6 months; non-LTS minors supported for ~4 weeks
- Ships ~one new minor per month (much faster cadence than Spring)
- Emergency branches for critical CVEs: `3.20.2-emergency`

**Kubernetes**
- **N-2 support**: exactly 3 minors simultaneously — when 1.36 ships, 1.33 is EOL immediately
- **Time-boxed**: 3 minors per year on a calendar. Features that miss the window wait.
- Patches on the 28th of every month across all 3 supported minors

<!--
SPEAKER NOTE: The Kubernetes N-2 policy is worth dwelling on — it's the clearest support window model here. You always know exactly which 3 versions are supported and exactly when your version reaches EOL.
-->

---

# The Vocabulary Reflects the Heritage

| Project | Heritage | Vocabulary |
|---|---|---|
| Spring | Pivotal/VMware · JVM · Maven | SNAPSHOT → M1 → RC1 → GA |
| Quarkus / Hibernate | Red Hat · JBoss AS (acquired 2006) | Alpha → Beta → **CR** → **Final** |
| Apache Kafka / Camel | Apache Software Foundation · SVN era | RC1 → GA (via community vote) |
| Kubernetes | Linux kernel · Go · CNCF | alpha.1 → beta.0 → **rc.0** → stable |

&nbsp;

> When you see `CR1` and `Final` → you are in the JBoss/Red Hat lineage
> When you see `trunk` and a PMC vote → you are in the ASF lineage
> When you see `alpha.1` as semver pre-release → you are in the systems/Kubernetes lineage

**The mental model:** understand the project's heritage and you can predict its conventions.

---

<!-- _paginate: false -->

# What You Now Know

| You see... | It means... | Where |
|---|---|---|
| `7.2.0-SNAPSHOT` | Mutable CI build — changes daily | snapshot repo |
| `7.2.0-M1` | Fixed pre-release, API in flux | milestone repo |
| `7.2.0-RC1` | Feature-complete, API frozen | milestone repo |
| `7.2.0` | Production-ready | **Maven Central** |
| `7.2.1` | Bugfix patch, no pre-release stages | **Maven Central** |

&nbsp;

- **Branches:** `N.N.x` owns all patches for that minor · `main` = next gen, not current stable
- **Fixes:** commit to oldest branch, merge forward · never backward
- **Boot:** one Boot version = the whole ecosystem · BOM pins all the rest
- **Other projects:** same concepts, different vocabulary — heritage predicts conventions

&nbsp;

📄 Reference doc: `docs/oss-release-strategy.md`
📊 Diagrams: `docs/diagrams.md`
🔗 Support dates: https://spring.io/projects/spring-framework#support

<!--
SPEAKER NOTE: Leave 5 minutes for questions. Common questions: (1) "When should we upgrade?" — before the previous minor reaches OSS EOL; one minor behind current GA is fine. (2) "Can we use a milestone in production?" — Spring says no; library authors sometimes do for final compat testing. (3) "How do we know when our version reaches EOL?" — spring.io/projects/spring-framework#support.
-->
