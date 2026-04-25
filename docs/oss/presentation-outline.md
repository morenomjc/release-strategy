# Presentation Outline: OSS Release Strategy

**Title:** How Open Source Projects Release: Branching, Versioning, and the Spring Model
**Format:** 20-30 minute team presentation
**Slide count:** 14 slides
**Audience:** Development team — varying familiarity with OSS processes, comfortable with git and Maven

---

## Slide 1 — Title and Framing

**Title:** How Open Source Projects Release
**Subtitle:** Branching, versioning, and the mental model behind Spring's release process

**Key points:**
- This talk gives you a vocabulary and mental model, not a how-to guide
- After this, you will know exactly what `7.2.0-M3` vs `7.2.0-RC1` vs `7.2.0` means
- You will understand why `7.0.x` and `6.2.x` both exist in the repo right now
- We use Spring as the primary case study, then see how other projects compare

**Speaker notes:**
Open with a quick question to the audience: "Has anyone looked at a Spring changelog and wondered why there are M1, M2, M3, RC1, RC2 before the final release? Or wondered what branch to look at for the current stable code?" This grounds the talk in a real problem rather than abstract theory. The goal is shared vocabulary — everyone on the team should be able to have the same conversation when they see a version string.

---

## Slide 2 — The Problem: What Does This Version Mean?

**Title:** What Does This Version Mean?

**Key points:**
- Show five version strings on screen: `7.2.0-SNAPSHOT`, `7.2.0-M1`, `7.2.0-RC1`, `7.2.0`, `7.2.1`
- All five exist in the same project — what is the difference?
- Two of them are on Maven Central. Three are not. Which ones?
- Understanding the answer requires knowing the release lifecycle

**Diagram to draw:** A simple horizontal timeline with these five version strings placed left to right in order: SNAPSHOT → M1 → RC1 → GA → patch. No detail yet — just establish the sequence.

**Speaker notes:**
Let this slide breathe. Most developers have seen these suffixes and developed vague intuitions. This talk makes those intuitions precise. The "two on Maven Central" hook is a good engagement device — wait for answers before revealing: only GA (7.2.0) and patches (7.2.1) go to Maven Central. Everything else is Spring-repo-only.

---

## Slide 3 — Three Branching Models (Context)

**Title:** How Projects Manage Code: Three Common Approaches

**Key points:**
- **Git Flow:** feature branches + develop branch + release branches + hotfix branches. Good for teams with long QA cycles. Complex.
- **Trunk-based development:** Everyone commits to one branch (trunk/main). Short-lived feature flags. Good for rapid deployment.
- **Maintenance-branch model:** One primary branch for next generation + long-lived N.N.x branches per released minor. Spring's model.

**Diagram to draw:** Three small branch diagrams side by side. Git flow: a complex web of branches. Trunk-based: a single horizontal line with tiny feature branches. Maintenance-branch: main + parallel N.N.x lines running horizontally below it.

**Speaker notes:**
Don't spend too long here — this slide is context, not the destination. The point is that Spring's model is not git-flow (a common misconception) and not strict trunk-based development. It is its own thing with a specific purpose: supporting multiple concurrent versions in production at the same time.

---

## Slide 4 — Spring's Branching Model

**Title:** Spring's Actual Branches Right Now

**Key points:**
- `main` = next unreleased generation (7.1.0-SNAPSHOT today)
- `7.0.x` = current GA production line (7.0.8-SNAPSHOT — next patch)
- `6.2.x` = previous generation, still in active OSS maintenance
- `6.1.x` = winding down (last commit June 2025)
- `6.0.x`, `5.3.x`, `5.2.x`... = EOL, preserved read-only forever
- The `.x` in `7.0.x` means "all patch releases of 7.0" — there is no `7.0.1` branch

**Diagram to draw:**
```
main  (7.1.0-SNAPSHOT)     -- next generation
  |
7.0.x (7.0.8-SNAPSHOT)     -- current GA
  |
6.2.x (6.2.19-SNAPSHOT)    -- previous generation
  |
6.1.x (6.1.22-SNAPSHOT)    -- winding down
  |
6.0.x, 5.3.x ...           -- EOL (read-only)
```
Draw as a vertical stack with arrows pointing downward to show generation order.

**Speaker notes:**
The key insight here is that `main` is not the stable code. If you checkout `main` right now, you are getting unreleased features that may change. The current stable release lives on `7.0.x`. Also worth noting: the SNAPSHOT version in each branch tells you exactly what the next release from that branch will be called. `7.0.x` at `7.0.8-SNAPSHOT` means the next patch will be `7.0.8`.

---

## Slide 5 — How Fixes Reach All Versions

**Title:** Fix Once, Propagate Forward

**Key points:**
- Fixes go in at the **oldest affected branch**, then merge forward
- Flow: `6.2.x` → merge → `7.0.x` → merge → `main`
- Uses `git merge --no-ff` (creates explicit merge commit, never fast-forward)
- Fixes never flow backward — no merging `7.0.x` into `6.2.x`
- For older-only fixes: cherry-pick via backport bot (label `for: backport-to-6.1.x`)

**Diagram to draw:** Three horizontal branch lines (6.2.x, 7.0.x, main). A fix commit goes in on 6.2.x. Draw a merge arrow pointing right and upward to 7.0.x, then another merge arrow right and upward to main. Show the same commit appearing in all three lines.

**Speaker notes:**
The evidence for this pattern is visible in the live commit history. If you look at recent commits on `main`, you see a stream of `Merge branch '7.0.x'` commits. If you look at `7.0.x`, you see `Merge branch '6.2.x'`. The same individual fix commits — same message, same change — appear in the same order on all three branches. This is the forward merge pattern in action.

---

## Slide 6 — Release Terminology: The Full Lifecycle

**Title:** From First Commit to End of Life

**Key points:**
- SNAPSHOT → M1 → [M2, M3] → RC1 → [RC2] → GA → patch releases → EOL
- Each stage serves a different audience and signals a different stability contract
- Only GA and patches land on Maven Central
- SNAPSHOT, milestones, and RCs are on repo.spring.io only (requires explicit repository config)

**Diagram to draw:** A vertical flowchart (or horizontal pipeline) showing the full sequence. Use boxes with the version labels. Add small annotations:
- Next to SNAPSHOT: "mutable, CI builds"
- Next to M1: "fixed artifact, API in flux"
- Next to RC1: "feature-complete, API frozen"
- Next to GA: "Maven Central"
- Next to patches: "straight to GA, no pre-releases"

---

## Slide 7 — SNAPSHOT: The Mutable Build

**Title:** SNAPSHOT: Always the Latest Build

**Key points:**
- `7.2.0-SNAPSHOT` is not a fixed artifact — it means "latest successful CI build of 7.2.0"
- Resolves to a different jar tomorrow than it does today
- Maven and Gradle re-check the remote repository (default: once per day)
- Never on Maven Central — Maven Central requires immutability; SNAPSHOTs are mutable by definition
- Lives at `https://repo.spring.io/snapshot`
- Appropriate users: contributors testing unreleased changes, Spring's own CI pipelines

**Key code snippet:**
```xml
<!-- To use a SNAPSHOT, you must add the repo explicitly -->
<repositories>
  <repository>
    <id>spring-snapshots</id>
    <url>https://repo.spring.io/snapshot</url>
    <snapshots><enabled>true</enabled></snapshots>
  </repository>
</repositories>
```

**Speaker notes:**
The "mutable" point is the key concept. Most developers assume a version string maps to an immutable artifact. SNAPSHOT breaks that assumption intentionally. The Maven/Gradle tooling knows this and has special resolution logic for it — it re-fetches SNAPSHOTs on a schedule. One consequence: if you have a SNAPSHOT in your `pom.xml` and your build starts failing with no code changes, it may be because the SNAPSHOT was updated upstream.

---

## Slide 8 — Milestones and RCs

**Title:** Milestones and Release Candidates: Fixed but Not Final

**Key points:**
- **Milestone (M1, M2...):** Fixed immutable artifact, API still in flux. "Here is a stake in the ground for early adopter feedback." Lives at repo.spring.io/milestone.
- **RC (RC1, RC2):** Feature-complete, API frozen. "We believe this could become GA if no blocking bugs are found." Same repo as milestones.
- Spring 6.0 used M1 through M5 before RC1 — more complex cycles use more milestones
- Key distinction: milestones allow API changes; RCs do not

**Comparison table to show:**
| | Milestone | RC |
|---|---|---|
| Features complete? | No | Yes |
| API stable? | Stabilizing | Frozen |
| Breaking changes? | Possible | No |
| Purpose | Shape feedback | Bug hunting |
| Time to next | Weeks-months | Days-weeks |

**Speaker notes:**
The practical implication for library authors: if you want to test compatibility with an upcoming Spring version, start at M2 or M3 (not M1 — too early, may change significantly). If you want to confirm your library is compatible before the GA ships, test against RC1. The RC is the last chance to find blockers.

---

## Slide 9 — GA and Patches

**Title:** GA and Patches: The Stable World

**Key points:**
- **GA:** Plain version string (`7.2.0`), on Maven Central, immutable forever
- Historical note: Spring 5.x used `.RELEASE` suffix (`5.3.39.RELEASE`). Spring 6.0 dropped it.
- **Patches:** Increment third number (`7.2.1`, `7.2.2`). Bugfixes and security only. Go straight from SNAPSHOT to GA — no milestones, no RCs.
- Active maintenance lines ship patches approximately monthly
- April 2026 example: `7.0.7` and `6.2.18` released on the same day (coordinated)

**Key rule to emphasize:** There is no such thing as `7.2.1-M1` or `7.2.1-RC1`. Patch releases have no pre-release stages. SNAPSHOT → GA.

---

## Slide 10 — Release Trains and the BOM

**Title:** Release Trains: Spring Is an Ecosystem, Not a Library

**Key points:**
- Spring does not release a single library — it releases a coordinated ecosystem of dozens of interdependent projects
- **Spring Boot is the conductor** — each Boot release pins compatible versions of all member projects via a BOM
- "We're on Spring Boot 3.3" implicitly means: Spring Framework 6.1.x, Spring Security 6.3.x, Spring Data 3.3.x (Quince), Micrometer 1.13.x, and more
- BOM = Bill of Materials — a Maven POM that resolves version conflicts across the ecosystem
- "Upgrade to Spring Boot 3.4" is upgrading the entire train, not just one artifact

**Diagram to draw:** Spring Boot version at the top (e.g., 3.3.0) with arrows pointing down to: Spring Framework 6.1.8, Spring Security 6.3.0, Spring Data 3.3.0 (Quince), Spring Batch 5.1.0, Micrometer 1.13.0, Reactor 3.6.6. Emphasize that the Boot version is the handle for the whole group.

**Speaker notes:**
This is often the "aha" moment for developers who have used Spring Boot but not thought explicitly about why version management feels easier than it should. The BOM is doing a lot of work. The Spring Data codename system (Lovelace, Moore, Neumann, Pascal, Quince...) is historical context — less relevant now that Spring Boot is the primary coordination mechanism, but worth mentioning since you see these names in older docs.

---

## Slide 11 — Ecosystem Comparison Overview

**Title:** Same Concept, Different Vocabulary: Four Projects Compared

**Key points:**
- All major OSS projects have the same underlying stages: "early feedback" → "feature complete" → "stable"
- The labels differ by ecosystem heritage
- Spring: SNAPSHOT → M1/M2 → RC1 → GA (plain version)
- Quarkus/Hibernate: — → Alpha1/Beta1 → CR1 → Final
- Apache Kafka: SNAPSHOT → — → RC1 → GA (no milestone stage)
- Kubernetes: dev build → alpha → beta → rc → stable

**Translation table to show:**
| Spring | Quarkus | Kafka | Kubernetes |
|---|---|---|---|
| SNAPSHOT | — | SNAPSHOT | dev build |
| M1 | Alpha1 | — (skips to RC) | alpha.1 |
| M2/M3 | Beta1 | — | beta.0 |
| RC1 | CR1 | rc1 | rc.0 |
| GA (`7.2.0`) | Final | `3.8.0` | `v1.36.0` |

---

## Slide 12 — Key Differences in Three Projects

**Title:** What Makes Each Project's Model Distinctive

**Key points (one per project):**

**Apache Kafka:**
- No milestone stage — goes straight from SNAPSHOT to RC
- ASF governance: a PMC community vote is required before any release is official; any committer can veto with a -1
- Uses `trunk` (not `main`) as primary branch — SVN heritage showing through git surface

**Quarkus:**
- Explicit LTS designation (every ~6 months) — Spring's equivalent requires checking the support policy page
- Ships roughly one new minor per month — much faster cadence than Spring
- Out-of-band emergency branches for critical CVEs: `3.20.2-emergency`

**Kubernetes:**
- N-2 support policy: exactly three versions simultaneously, no exceptions. When 1.36 ships, 1.33 is EOL.
- Time-boxed releases: three minors per year on a calendar schedule. Features that miss the window wait.
- Semver pre-release identifiers (v1.36.0-alpha.1) vs Maven version qualifiers (7.2.0-M1) — different tooling ecosystems

**Speaker notes:**
The Kubernetes N-2 policy is worth dwelling on briefly. It is the clearest support window model here — you always know exactly which three versions are supported and when your version will reach EOL. Spring's model is more flexible but requires reading the support policy page to know where you stand.

---

## Slide 13 — Why the Terminology Differs: Ecosystem Heritage

**Title:** The Vocabulary Reflects the Heritage

**Key points:**
- **Spring/Pivotal:** "Milestone" comes from sprint/iteration planning — neutral about quality, maps to planned iteration boundaries. Maven SNAPSHOT semantics inherited from the broader JVM ecosystem (Maven 2, circa 2004).
- **Red Hat/JBoss (Quarkus, Hibernate, WildFly):** `CR` = Candidate Release (instead of RC), `Final` (instead of plain version). Red Hat/JBoss vocabulary traces to JBoss AS, acquired by Red Hat in 2006.
- **Apache Software Foundation (Kafka, Camel):** The governance vote IS the release mechanism. Pre-release label complexity matters less than the community process.
- **Linux/Systems (Kubernetes):** Alpha/beta/rc/stable come from Linux kernel conventions and the systems world. Semver pre-release format reflects Go/binary distribution rather than Maven coordinates.

**Key takeaway:** When you see `CR1` and `Final`, you are in the JBoss/Red Hat lineage. When you see `trunk` and a community vote, you are in the ASF lineage. When you see `alpha.1` as a semver pre-release, you are in the systems/Kubernetes lineage. The vocabulary predicts the organizational culture.

**Diagram to draw:** A simple heritage tree or table showing: Spring → Pivotal/VMware heritage. Quarkus/Hibernate/WildFly → Red Hat/JBoss heritage. Kafka/Camel → ASF heritage. Kubernetes → Linux/systems heritage.

---

## Slide 14 — Summary and Takeaways

**Title:** What You Now Know

**Key points:**
- A version string tells you: what stage it is in, where to find the artifact, and what stability you can expect
- SNAPSHOT = mutable CI build (snapshot repo, not Central). Milestone = fixed pre-release, API in flux. RC = feature-complete, API frozen. GA = production, Maven Central only.
- Patches skip pre-release stages entirely: SNAPSHOT → GA.
- Spring's `N.N.x` branches each own all patch releases for that minor. `main` is the next unreleased generation, not current stable.
- Fixes flow forward only: commit to oldest affected branch, merge up. Never backward.
- Spring Boot is the conductor of a release train — one Boot version pins the compatible versions of the whole ecosystem.
- Different project vocabularies (Alpha/CR/Final, trunk/vote, N-2) reflect organizational heritage, not different underlying concepts.

**Call to action:**
- Bookmark the reference doc in this repo: `docs/oss-release-strategy.md`
- When you encounter a version string you do not recognize in any OSS project, the quick-reference table at the top of the doc is your starting point
- Check https://spring.io/projects/spring-framework#support to know the current support status of any Spring version you are running

**Speaker notes:**
Leave 5 minutes for questions. Common questions that tend to come up:
1. "When should we upgrade from one minor to the next?" — generally before the previous minor reaches OSS EOL. Being one minor behind current GA is acceptable; being two or more increases upgrade risk.
2. "Can we use a milestone in production?" — Spring says no, but library authors sometimes do for final compat testing before a GA ships. For application code, stick to GA.
3. "How do we know when our Spring version reaches EOL?" — https://spring.io/projects/spring-framework#support has the official dates. The commercial support option exists if your organization cannot upgrade on the OSS schedule.

---

## Presentation Notes

**Total time budget (20-30 minutes):**
- Slides 1-2: 3 minutes (framing, problem statement)
- Slides 3-5: 5 minutes (branching model)
- Slides 6-9: 7 minutes (release terminology — the core of the talk)
- Slides 10: 3 minutes (release trains)
- Slides 11-13: 5 minutes (comparisons and heritage)
- Slide 14: 2 minutes (summary)
- Questions: 5 minutes

**Diagrams that need to be drawn (as slide visuals):**
1. Slide 2: SNAPSHOT → M1 → RC1 → GA → patch timeline
2. Slide 3: Three branch model sketches side by side
3. Slide 4: Vertical branch stack (main, 7.0.x, 6.2.x, 6.1.x, EOL lines)
4. Slide 5: Forward merge flow showing a fix traveling from 6.2.x up through 7.0.x to main
5. Slide 6: Full lifecycle pipeline with annotations
6. Slide 10: Spring Boot BOM hub with downstream library versions as spokes
7. Slide 13: Heritage tree

**Suggested visual style:** Diagrams work well as simple hand-drawn-style flows (Excalidraw or Mermaid). Tables should be visible from the back of a room — limit to 4-5 columns maximum.

**Recommended demo if time allows (Slide 4 or 5):**
Open https://github.com/spring-projects/spring-framework/branches in a browser during the talk. The branch list is the most convincing visualization of the maintenance-branch model — live, real, immediate. Filter by `*.x` to show only the maintenance branches. Then show a recent commit on `main` to confirm the `Merge branch '7.0.x'` pattern.
