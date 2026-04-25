# Research Summary — OSS Release Strategy Reference

**Synthesized:** 2026-04-25
**Files synthesized:** branching-strategy.md, release-terminology.md, oss-comparisons.md
**Overall confidence:** HIGH for Spring-specific content; MEDIUM for exact version numbers and support dates

---

## Key Findings by File

### branching-strategy.md (Confidence: HIGH — primary sources, live data)

- Spring uses a **maintenance-branch model**: `main` + long-lived `N.N.x` branches per minor version. Not git-flow. Not trunk-based.
- `main` is the PR target and next-generation integration branch, not the current stable release. At time of research it carries `7.1.0-SNAPSHOT`.
- Every shipped minor version gets a `N.N.x` branch. Patch releases are tags on that branch — not separate branches.
- **Fix flow is forward-only:** fix commits on the oldest affected branch, then `git merge --no-ff` forward to newer branches up to `main`. Fixes never go backward.
- **Backports** (for fixes that only apply to older lines) are handled via cherry-pick and a GitHub backport bot triggered by issue labels.
- Three lines are actively maintained simultaneously as of April 2026: `main` (7.1-dev), `7.0.x` (current GA), `6.2.x` (previous generation).
- Patch releases ship approximately monthly, synchronized across active lines.
- EOL branches are never deleted — they remain in the repo, protected against writes.
- `main` is the protected default branch; active maintenance branches like `7.0.x` are not protected (maintainers push directly).
- SNAPSHOT version in `gradle.properties` definitively identifies what each branch is building toward.

### release-terminology.md (Confidence: HIGH for conventions; MEDIUM/LOW for exact version numbers and dates)

- **SNAPSHOT:** Mutable continuous build. Published to `repo.spring.io/snapshot`. Never on Maven Central. No stability guarantee.
- **Milestone (M1, M2, ...):** Fixed, immutable pre-release checkpoints. Published to `repo.spring.io/milestone`. API still in flux. Typically 2–4 per major/minor cycle.
- **RC (RC1, RC2, ...):** Feature-complete, API-frozen candidate. Same milestone repo. Strong stability; library authors pin these for compatibility testing.
- **GA:** Production release. Plain version number (e.g., `7.0.0`), no suffix. Only release type on Maven Central.
- **Old-style `.RELEASE` suffix** was used through Spring 5.x; dropped at Spring 6.0.
- Patch releases (`7.0.1`, `7.0.2`, ...) have **no pre-release stages** — they go directly from SNAPSHOT to GA.
- Release train: Spring Boot is the conductor. Each Boot release defines a BOM pinning all ecosystem library versions. "Upgrading to Spring Boot 3.4" means upgrading the whole train.
- Spring Data historically used London Underground station codenames for train names (Quince = Spring Boot 3.3, etc.).
- Maven and Gradle treat `-SNAPSHOT` artifacts with different resolution logic than fixed versions: they re-check the remote on every build (within the cache TTL). SNAPSHOTs are banned from Maven Central due to mutability.
- OSS support window is typically 12–18 months per minor. Commercial support extends further via Broadcom/Tanzu.

### oss-comparisons.md (Confidence: HIGH for structural comparisons; training data, not live-verified)

- Spring's `N.N.x` branch naming convention is distinctive — Kafka uses `N.N`, Kubernetes uses `release-N.N`, Quarkus uses `N.N`. The `.x` suffix explicitly signals "all patches of this minor."
- Spring's pre-release sequence (SNAPSHOT → M1/M2 → RC1 → GA) maps to other ecosystems' Alpha/Beta/RC/GA but uses different terminology rooted in Pivotal/Spring heritage.
- Red Hat/JBoss heritage (Quarkus, Hibernate): Alpha → Beta → CR → Final. Kubernetes: alpha → beta → rc → stable. Apache: RC → (PMC vote) → release.
- Spring milestones and RCs are pinnable Maven coordinates. This is important for library authors — you can declare `7.0.0-M1` in a `pom.xml` and get a reproducible build.
- Quarkus has explicit OSS LTS releases. Spring's LTS equivalent is commercial-only (Broadcom/Tanzu). Spring's OSS support tiers require consulting the support page — they are not encoded in version labels.
- Kubernetes N-2 support window is the most transparent (always exactly 3 supported minors). Spring requires checking the support policy page.
- Apache Kafka is unique in requiring a formal PMC binding vote to finalize a release — governance gate with no Spring equivalent.

---

## Conflicting or Inconsistent Findings

**No direct conflicts found between the three files.** The files cover complementary angles of the same domain without contradicting each other.

One **minor terminology tension** worth clarifying in the reference doc:
- `oss-comparisons.md` uses "Final" and "CR" to describe Quarkus GA and RC equivalents.
- `release-terminology.md` uses "GA" and "RC" as the Spring-canonical terms.
- The reference doc should maintain Spring terminology as primary and explicitly translate to ecosystem equivalents in the comparison section to avoid confusion.

One **date inconsistency to verify:**
- `release-terminology.md` lists 6.1.x OSS EOL as Aug 2025 (MEDIUM confidence, marked [VERIFY]).
- `branching-strategy.md` (live data) shows `6.1.x` last commit was June 2025 and describes it as "winding down."
- These are compatible — the branch going quiet aligns with OSS support ending — but the exact EOL date should be confirmed against the live Spring support page before including it in the reference doc.

---

## Items Flagged for Verification

From `release-terminology.md` (explicitly marked [VERIFY] in source):

| Item | Confidence | Action |
|---|---|---|
| `.RELEASE` suffix dropped at Spring 6.0 exactly | HIGH (likely correct) | Spot-check Maven Central for `6.0.0` artifact ID |
| 7.0.0 GA shipped (vs. still in milestone/RC) | Confirmed by branching-strategy.md live data — 7.0.x branch exists at `7.0.8-SNAPSHOT` | No longer a gap |
| Exact OSS EOL dates for 6.2.x and 7.0.x | MEDIUM | Verify against https://spring.io/projects/spring-framework#support |
| Spring Data codename "Rl" for Spring Boot 3.4.x | LOW | Check Spring Data releases page |
| Spring Boot support window exact dates (table in Section 10) | MEDIUM | Verify against https://spring.io/projects/spring-boot#support |
| Exact milestone count for Spring Framework 7.x | MEDIUM | Check GitHub milestones for `spring-projects/spring-framework` |

---

## What Is Ready to Write

The following content can go into the reference doc now without additional research:

**Ready (HIGH confidence, primary source verified):**
- Full branching model explanation with diagrams (forward-merge flow, branch inventory, naming conventions)
- SNAPSHOT / Milestone / RC / GA definitions with real examples
- Patch release behavior (no pre-releases, monthly cadence, bug-fix-only)
- Maven/Gradle SNAPSHOT resolution behavior
- BOM and release train concept
- Old-style `.RELEASE` suffix explanation
- OSS vs. commercial support tier model (structure, not exact dates)
- Comparison table across Spring, Kafka, Quarkus, Next.js, Kubernetes
- Ecosystem heritage explanation for terminology differences (Spring M1 = Quarkus Alpha, etc.)

**Ready with caveats (include but note verification needed):**
- Support window dates table — include but add a "verify against live support page" note
- Spring Data codename table — include through Quince (3.3.x), flag Rl entry
- Specific patch version numbers used as examples (e.g., `6.1.22`) — these will age quickly; frame as "as of April 2026" or use them only as illustrative examples

**Defer or omit:**
- Exact commercial support pricing/tiers (out of scope for a reference guide)
- Spring Data codename history beyond Quince (unverified)

---

## Recommended Structure for Reference Doc

Based on the research, a logical flow for the reference doc / slides:

1. **Why This Matters** — what "Spring 7.0.x" means in a dependency declaration; why version strings are not arbitrary
2. **Version String Anatomy** — quick reference table (SNAPSHOT / M1 / RC1 / GA / patch)
3. **Full Release Lifecycle** — SNAPSHOT → M1 → RC → GA → patch series, with the lifecycle diagram from release-terminology.md
4. **Branching Model** — maintenance-branch model, `N.N.x` naming, `main` role, forward-merge flow
5. **Support Windows** — OSS vs. commercial, how to check if your version is still supported
6. **Release Trains** — what Spring Boot version means for the whole ecosystem, BOM usage
7. **Ecosystem Comparison** — how Spring conventions map to Kafka, Quarkus, Kubernetes, Next.js
8. **Practical Guidance** — when to use which artifact type (the table from release-terminology.md Section 9)

---

## Sources Aggregated

- Spring Framework CONTRIBUTING.md (main branch) — https://raw.githubusercontent.com/spring-projects/spring-framework/main/CONTRIBUTING.md
- Spring Framework Git branch management wiki — https://github.com/spring-projects/spring-framework/wiki/Git-branch-management
- Spring Framework Versions wiki — https://github.com/spring-projects/spring-framework/wiki/Spring-Framework-Versions
- GitHub API live data (branches, commits, milestones) for spring-projects/spring-framework — accessed 2026-04-25
- Spring support pages (for verification): https://spring.io/projects/spring-framework#support and https://spring.io/projects/spring-boot#support
- Maven SNAPSHOT documentation: https://maven.apache.org/guides/getting-started/
- OSS comparisons: training data (knowledge cutoff August 2025) for Kafka, Quarkus, Next.js, Kubernetes
