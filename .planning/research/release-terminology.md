# OSS Release Terminology and Versioning Conventions

**Domain:** OSS Java/JVM ecosystem — Spring Framework, Spring Boot, comparable projects
**Researched:** 2026-04-25
**Overall confidence:** HIGH for Spring conventions (training through Aug 2025 + published Spring conventions are stable and well-documented); MEDIUM for exact current version numbers without live verification
**Note:** WebFetch/WebSearch tools were unavailable during this session. All findings are drawn from training data on officially published Spring conventions, Maven specs, and community patterns. Claims marked [VERIFY] should be spot-checked against live sources.

---

## 1. SNAPSHOT

### Definition

A SNAPSHOT is a **mutable, in-development build** artifact. The version string `7.2.0-SNAPSHOT` does not refer to a fixed artifact — it refers to "whatever the latest build of the 7.2.0 development line is right now."

The `-SNAPSHOT` suffix is a **Maven convention** that the entire JVM ecosystem inherits. It carries specific semantic meaning in the build toolchain (see Section 9).

### When It Is Published

- Continuously, on every successful CI build that commits to the development branch for that version line.
- Spring Framework publishes SNAPSHOTs to `https://repo.spring.io/snapshot` (not Maven Central).
- A SNAPSHOT is never promoted to Maven Central. SNAPSHOTs live only in Spring's own artifact repositories.
- When a milestone, RC, or GA is cut, a fixed versioned artifact is published instead. SNAPSHOT publishing for that version line continues in parallel until the line is closed.

### Real Examples

```
7.2.0-SNAPSHOT   — active development of 7.2.0 before first milestone
7.2.1-SNAPSHOT   — active development of the first patch release after 7.2.0 GA
6.2.0-SNAPSHOT   — was published throughout 2024 as 6.2.x was being developed
```

Spring Boot mirrors this:
```
3.5.0-SNAPSHOT   — in-progress development of Spring Boot 3.5.0
3.3.7-SNAPSHOT   — patch line development after 3.3.6 GA
```

### Stability Contract

None. Consumers using `-SNAPSHOT` explicitly opt in to receiving breaking changes at any time. SNAPSHOTs are appropriate only for:
- Contributors testing against the bleeding edge
- Integration testing in the Spring project's own CI pipelines
- Experimentation by early adopters willing to deal with breakage

---

## 2. Milestone Releases (M1, M2, M3, ...)

### Definition

A Milestone is a **fixed, versioned pre-release** checkpoint. Unlike a SNAPSHOT, `7.2.0-M1` is immutable — once published, the artifact never changes. It represents a deliberate "stake in the ground" during development.

Milestones signal:
- "Here is a concrete artifact you can evaluate."
- "This is not production-ready, but the API shape is becoming visible."
- "We want early adopter feedback before we commit to RC."

### Naming Convention

The suffix is `-M` followed by a sequential integer: `-M1`, `-M2`, `-M3`. Numbering starts at 1 and increments. There is no predetermined maximum — Spring Framework releases have used up to M3 or M4 depending on the cycle complexity.

### Typical Milestone Count Before RC

Spring Framework and Spring Boot typically ship **2–4 milestones** before the first RC. The Spring 6.0 cycle used M1–M5. The Spring Boot 3.0 cycle also used multiple milestones. Simpler minor releases (e.g., a 6.1.x or 7.2.x feature release) may use only M1–M2. [VERIFY: exact count for Spring Framework 7.x milestones]

### Real Examples (Spring Framework 6.x cycle, HIGH confidence)

```
6.0.0-M1   — first public milestone of Spring 6 (early 2022)
6.0.0-M2
6.0.0-M3
6.0.0-M4
6.0.0-M5
6.0.0-RC1  — first release candidate (mid-2022)
6.0.0-RC2
6.0.0-RC3
6.0.0-RC4
6.0.0      — GA (November 2022)
```

Spring Boot 3.0 (which depends on Spring Framework 6.0):
```
3.0.0-M1 through 3.0.0-M5
3.0.0-RC1, RC2
3.0.0      — GA (November 2022, same day as Spring Framework 6.0)
```

### Where Milestones Are Published

Spring milestone artifacts are published to `https://repo.spring.io/milestone`. They are **not** on Maven Central. Projects that want to test against a milestone must explicitly add this repository to their build configuration.

### Stability Contract

Limited. API shape may still change between milestones, particularly M1 and M2. By the last milestone before RC, the API should be stabilizing. Milestones are appropriate for:
- Library authors who need to test compatibility early
- Teams willing to track pre-release versions in non-production environments
- Providing feedback on API ergonomics

---

## 3. Release Candidate (RC1, RC2, ...)

### Definition

A Release Candidate is a **feature-complete, API-frozen build** that the team believes could become the GA release if no blocking issues are found. The suffix is `-RC` followed by a sequential integer: `-RC1`, `-RC2`.

The key distinction from a milestone:

| Attribute | Milestone | Release Candidate |
|-----------|-----------|------------------|
| Feature completeness | Partial | Complete |
| API stability | "Stabilizing" | Frozen (no new APIs) |
| Breaking changes allowed | Yes (with notice) | No |
| Purpose | Shape feedback, checkpoints | Final validation, bug hunting |
| Typical duration before next | Weeks to months | Days to weeks |

If a blocking bug is found in RC1, it is fixed and RC2 is released. If RC2 is clean, it is promoted (or re-released as-is) to GA.

### Real Examples (HIGH confidence)

Spring Framework 6.1.x:
```
6.1.0-RC1
6.1.0-RC2
6.1.0      — GA (November 2023)
```

Spring Boot 3.2:
```
3.2.0-RC1
3.2.0-RC2
3.2.0      — GA (November 2023)
```

Spring Framework 5.3:
```
5.3.0-RC1
5.3.0-RC2
5.3.0      — GA (October 2020)
```

### Where RCs Are Published

Same repository as milestones: `https://repo.spring.io/milestone`. Not on Maven Central.

### Stability Contract

Strong. Production use of an RC is still not recommended by Spring, but library authors and framework integrators commonly pin to RC versions for final compatibility testing. An RC that ships with no blocking bugs becomes the GA binary (the same artifact hash, just re-tagged in some projects; in Maven terms a new artifact is published to Central without the `-RC1` suffix).

---

## 4. GA — General Availability

### Definition

GA (General Availability) is the **production-ready release**. In Spring's Maven versioning, a GA release carries **no suffix**:

```
7.2.0    — GA (not 7.2.0.RELEASE, not 7.2.0-GA)
```

This is a change from older Spring conventions. Spring Framework 4.x and 5.x used the `.RELEASE` suffix (e.g., `5.3.39.RELEASE`). Starting with Spring Framework 6.0, the `.RELEASE` suffix was dropped and plain `6.0.0` is the GA artifact. [VERIFY: confirm 6.0.0 dropped .RELEASE]

### Published To

Maven Central. A GA release is the only release type that lands on Maven Central (`https://repo1.maven.org/maven2/`). All SNAPSHOTs, milestones, and RCs are Spring-repo-only.

### Real Examples

```
# Spring Framework GA releases (plain version = GA)
6.0.0
6.0.1
6.0.14
6.1.0
6.1.14
7.0.0       [VERIFY: confirm 7.0.0 shipped; may still be in milestone/RC as of Aug 2025]

# Spring Boot GA releases
3.0.0
3.1.0
3.2.0
3.3.0
3.3.6
3.4.0
```

### Old-style Suffix for Context

If you encounter `.RELEASE` in dependency declarations for older Spring projects, that is the GA artifact:
```
# Maven POM — Spring 5.x style
<version>5.3.39.RELEASE</version>

# Gradle — same
implementation 'org.springframework:spring-core:5.3.39.RELEASE'
```

---

## 5. Alpha and Beta — How Spring Differs From Other Projects

### Spring Framework's Practice

Spring Framework and Spring Boot **do not use Alpha or Beta labels**. Their pre-release vocabulary is:

```
SNAPSHOT → M1 → M2 → [M3...] → RC1 → [RC2...] → GA
```

There are no `7.2.0-alpha1` or `7.2.0-beta1` artifacts in the Spring ecosystem. This is a deliberate choice: Milestone serves the role that Alpha/Beta serves elsewhere, with more granular sequential numbering.

### Projects That Use Alpha/Beta

**Quarkus:**
```
3.0.0.Alpha1
3.0.0.Alpha2
3.0.0.Beta1
3.0.0.Beta2
3.0.0.CR1       (CR = Candidate Release, analogous to RC)
3.0.0.Final     (GA)
```
Quarkus uses `.Final` instead of plain `3.0.0` for GA, and `CR` instead of `RC`. This reflects its Red Hat / JBoss heritage.

**Apache projects (e.g., Apache Kafka, Apache Camel):**
```
3.7.0-SNAPSHOT
3.7.0-alpha1    (early testing)
3.7.0-beta1     (feature complete, API may still shift)
3.7.0-rc1
3.7.0           (GA)
```
Apache projects frequently publish alpha builds to Maven Central (unlike Spring, which keeps pre-GA artifacts off Central).

**Hibernate ORM:**
```
6.5.0.Alpha1
6.5.0.Alpha2
6.5.0.Beta1
6.5.0.Beta2
6.5.0.CR1
6.5.0.Final
```
Same pattern as Quarkus — JBoss heritage, `.Final` suffix for GA.

### Translation Table

| Spring Concept | Quarkus/Hibernate Equivalent | Apache Equivalent |
|----------------|------------------------------|-------------------|
| M1 | Alpha1 | alpha1 |
| M2/M3 | Alpha2/Beta1 | beta1 |
| RC1 | CR1 | rc1 |
| GA (plain `7.2.0`) | Final (`7.2.0.Final`) | plain `3.7.0` |

### Why Spring Chose Milestone Over Alpha/Beta

The Spring team has publicly stated they prefer the Milestone label because it is neutral about quality (Alpha implies "barely works," which is often inaccurate by M1 if the team has been developing in private) and because it maps directly to their sprint/iteration planning terminology. Each milestone corresponds to a planned sprint or iteration boundary.

---

## 6. Patch Releases

### Definition

A patch release increments the **third version number** (the patch segment in MAJOR.MINOR.PATCH semver):

```
7.2.0   — initial GA
7.2.1   — first patch
7.2.2   — second patch
...
7.2.14  — fourteenth patch (realistic for an active Spring maintenance line)
```

### What Goes Into a Patch

Spring patch releases contain:
- Bug fixes only (no new features, no API additions)
- Dependency version bumps for security CVEs
- Compatibility fixes for JDK updates or ecosystem changes

Spring follows this discipline strictly. If a fix requires an API addition it is deferred to the next minor release (7.3.0), not backported as a patch.

### Release Cadence

Active maintenance lines receive patches approximately monthly. The Spring team publishes a release on a roughly 4–6 week cadence for each supported minor version. During security incidents the cadence can be much faster (days).

### Real Examples (Spring Boot 3.3.x patch series, HIGH confidence)

```
3.3.0   — GA, May 2024
3.3.1
3.3.2
3.3.3
3.3.4
3.3.5
3.3.6   — [VERIFY: approximate, patch count may differ]
```

Spring Framework 6.1.x:
```
6.1.0   — November 2023
6.1.1
6.1.2
...
6.1.14  — [VERIFY: approximate]
```

### Patch Versions Are Not Pre-Released

Unlike minor/major releases, patch releases do not have milestones or RCs. They go directly from SNAPSHOT to GA:

```
7.2.1-SNAPSHOT → 7.2.1 (GA)
```

No `7.2.1-M1` or `7.2.1-RC1` exists. This is consistent across Spring and most Java OSS projects.

---

## 7. Full Release Lifecycle Sequence

The complete sequence from first commit to stable maintenance:

```
[Development begins]
         │
         ▼
  7.2.0-SNAPSHOT        ← continuous CI builds, published to snapshot repo
         │
         ▼
    7.2.0-M1            ← planned iteration checkpoint, published to milestone repo
         │
         ▼
    7.2.0-M2            ← second checkpoint; API still in flux
         │
         ▼
    7.2.0-M3            ← (optional) third checkpoint
         │
         ▼
    7.2.0-RC1           ← feature complete, API frozen, published to milestone repo
         │               [if blocking bugs found → RC2, RC3...]
         ▼
    7.2.0-RC2           ← (if needed)
         │
         ▼
      7.2.0             ← GA, published to Maven Central; OSS support begins
         │
         ▼
      7.2.1             ← first patch (bug fixes only, no pre-releases)
         │
         ▼
      7.2.2             ← second patch
         │
         ▼
       ...
         │
         ▼
  [OSS support window ends — typically 12–18 months after GA]
         │
         ▼
  [Commercial support (VMware/Broadcom) continues for enterprise subscribers]
```

Simultaneously, while 7.2.x is being patched:
```
7.3.0-SNAPSHOT is published → 7.3.0-M1 → ... → 7.3.0 GA
```

The two lines run in parallel. Users are expected to upgrade to the new minor within the OSS support window.

---

## 8. Release Trains

### Definition

A **release train** is a coordination mechanism where multiple related projects agree to release together on a synchronized schedule, with compatible versions.

### The Spring Release Train

Spring does not release a single library — it releases an ecosystem. A "Spring Boot release" is really a train that pins compatible versions of:

- Spring Framework
- Spring Data (multiple modules: JPA, MongoDB, Redis, etc.)
- Spring Security
- Spring Batch
- Spring Integration
- Spring AMQP
- Spring for GraphQL
- Micrometer
- Reactor (Project Reactor)
- Netty
- Tomcat / Undertow / Jetty
- ... (dozens of additional libraries)

**Spring Boot is the conductor of this train.** Each Spring Boot release (e.g., `3.3.0`) defines a Bill of Materials (BOM) — a Maven POM that pins compatible versions of all member projects.

### BOM Example

```xml
<!-- Spring Boot BOM controls all downstream versions -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.0</version>
</parent>

<!-- These versions are resolved by the BOM — no need to specify them explicitly -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-core</artifactId>
    <!-- version omitted — Spring Boot BOM provides 6.1.8 -->
</dependency>
```

### Train Naming Conventions in the Spring Ecosystem

Before Spring Boot became the canonical train conductor, the **Spring Data project** used a codename-based release train system. The names were London Underground station names in alphabetical order:

```
Arabba       (very old)
Babbage
Codd
...
Hopper       (circa Spring Data 2021)
Ingalls
Jevons
Klara
Lovelace     (aligned with Spring Boot 2.6.x)
Moore        (aligned with Spring Boot 2.7.x)
Neumann      (aligned with Spring Boot 3.0.x / Spring Framework 6.0)
O'Brien      (aligned with Spring Boot 3.1.x)
Pascal       (aligned with Spring Boot 3.2.x / Spring Framework 6.1)
Quince       (aligned with Spring Boot 3.3.x)
Rl           (aligned with Spring Boot 3.4.x) [VERIFY]
```

Each train name maps to a generation. Within a train, individual Spring Data modules (Spring Data JPA, Spring Data MongoDB, etc.) release patch versions independently but must stay within the train's compatibility contract.

### What "Train" Signals to Consumers

When you see "Spring Boot 3.3" used as a qualifier, it signals the entire constellation of compatible libraries. A team saying "we're on Spring Boot 3.3" implicitly means:
- Spring Framework 6.1.x
- Spring Security 6.3.x
- Spring Data 3.3.x (Quince train)
- Micrometer 1.13.x
- etc.

This is why "upgrade to Spring Boot 3.4" is a meaningful unit of work — it means upgrading the entire train.

---

## 9. Maven/Gradle SNAPSHOT Resolution Behavior

### Why `-SNAPSHOT` Has Special Build-Tool Meaning

The `-SNAPSHOT` suffix is not just a naming convention — **Maven and Gradle treat SNAPSHOT artifacts with fundamentally different resolution logic** than fixed-version artifacts.

### Maven Behavior

**Fixed versions** (milestones, RCs, GA): Maven resolves once, caches in `~/.m2/repository`, and never re-downloads unless you explicitly delete the local cache or use `-U`.

**SNAPSHOT versions**: Maven checks the remote repository on every build (subject to the `updatePolicy` setting, default: once per day). If a newer SNAPSHOT has been published since your last build, Maven downloads it automatically.

Maven tracks SNAPSHOT resolution via timestamped metadata files:
```
~/.m2/repository/org/springframework/spring-core/7.2.0-SNAPSHOT/
    spring-core-7.2.0-20250312.143022-42.jar   ← timestamped artifact
    spring-core-7.2.0-SNAPSHOT.jar             ← local symlink/copy to latest
    maven-metadata-local.xml                   ← records last update time
```

The remote SNAPSHOT artifact is actually published as a timestamped file (e.g., `7.2.0-20250312.143022-42.jar`). Maven resolves the logical `7.2.0-SNAPSHOT` coordinate to the latest timestamped artifact per the `maven-metadata.xml` on the server.

**Force update:**
```bash
mvn clean install -U    # -U forces SNAPSHOT update regardless of update policy
```

### Gradle Behavior

Gradle has equivalent but differently-configured behavior:

```groovy
// build.gradle — explicitly declaring a SNAPSHOT dependency
dependencies {
    implementation 'org.springframework:spring-core:7.2.0-SNAPSHOT'
}

// Required: tell Gradle where to find SNAPSHOTs
repositories {
    maven { url 'https://repo.spring.io/snapshot' }
    mavenCentral()
}
```

Gradle's default caching for SNAPSHOTs is 24 hours. Override:
```groovy
configurations.all {
    resolutionStrategy.cacheChangingModulesFor 0, 'seconds'  // always re-check
}
```

Mark a dependency as "changing" (Gradle's term for SNAPSHOT semantics):
```groovy
dependencies {
    implementation('com.example:library:1.0') {
        changing = true   // treat as SNAPSHOT even without -SNAPSHOT suffix
    }
}
```

### Why SNAPSHOTs Are Banned From Maven Central

Maven Central enforces immutability: once an artifact is published at a coordinate, it cannot be modified or replaced. SNAPSHOT artifacts by definition are mutable (they are updated frequently). Maven Central's release policy explicitly prohibits SNAPSHOT uploads. This is why Spring operates a separate `repo.spring.io/snapshot` repository for SNAPSHOT artifacts.

### Practical Implications for Dependency Management

| Scenario | Recommended Practice |
|----------|---------------------|
| Production application | Only use GA versions from Maven Central |
| Library compatibility testing | Use milestone or RC from Spring milestone repo |
| Contributing to Spring | Use SNAPSHOT to test against unreleased changes |
| CI pipeline testing Spring changes | Add snapshot repo, pin to `-SNAPSHOT` |
| Security-sensitive environment | Never add snapshot/milestone repos; Maven Central only |

---

## 10. Spring Support Windows

### Two-Tier Support Model

Spring (now under Broadcom/VMware Tanzu) maintains two support tiers for each release:

**OSS (Open Source) Support:** Free. Bugs and security fixes are published publicly to Maven Central. Ends when the project team marks the version as End-of-Life.

**Commercial Support:** Paid. Via VMware Spring Runtime (part of Tanzu portfolio). Extends the maintenance window significantly. Security patches for commercially supported versions are delivered to enterprise subscribers. Not available to the general public.

### Spring Framework Support Windows (HIGH confidence for general structure, VERIFY exact dates)

| Version | Initial GA | OSS End-of-Life | Commercial EOL |
|---------|-----------|-----------------|----------------|
| 5.3.x | Oct 2020 | Dec 2024 | Dec 2027 |
| 6.0.x | Nov 2022 | Aug 2024 | Dec 2026 |
| 6.1.x | Nov 2023 | Aug 2025 | Dec 2027 |
| 6.2.x | Nov 2024 | Aug 2026 (est.) | Dec 2028 (est.) |
| 7.0.x | [2025, VERIFY] | ~12–18 mo. after GA | ~3 years after GA |

[VERIFY: Exact EOL dates from https://spring.io/projects/spring-framework#support — the table above is based on training data patterns and should be confirmed against the live support page]

### Spring Boot Support Windows (MEDIUM confidence on exact dates)

Spring Boot follows a similar structure, with each minor release receiving OSS support for approximately 12 months:

| Version | Initial GA | OSS EOL | Aligns With |
|---------|-----------|---------|-------------|
| 2.7.x | May 2022 | Nov 2023 | Spring Framework 5.3.x |
| 3.0.x | Nov 2022 | Feb 2024 | Spring Framework 6.0.x |
| 3.1.x | May 2023 | Nov 2024 | Spring Framework 6.1.x |
| 3.2.x | Nov 2023 | Nov 2024 | Spring Framework 6.1.x |
| 3.3.x | May 2024 | Nov 2025 | Spring Framework 6.1.x |
| 3.4.x | Nov 2024 | Nov 2025 | Spring Framework 6.2.x |
| 3.5.x | May 2025 (est.) | Nov 2026 (est.) | Spring Framework 6.2.x |

[VERIFY: Confirm against https://spring.io/projects/spring-boot#support]

### How Support Windows Affect Patch Releases

**During OSS support window:** All patch releases are published publicly. The version receives regular monthly patches. Anyone can consume these for free.

**After OSS EOL, during commercial window:** Patches are still produced but delivered through commercial channels. If you depend on `6.0.x` after its OSS EOL, you need a commercial support contract to get security patches. The open-source artifacts on Maven Central are frozen at the last OSS-era patch.

**After commercial EOL:** No patches of any kind. Teams on these versions are entirely on their own.

### Which Versions Actually Get Patches at Any Given Time

Spring typically maintains two to three minor versions concurrently in OSS support. The team's practice (as of 2024) is:

- One "generation" back minor version: security-only patches
- Current minor version: full bug fix + security patches
- Upcoming minor version: active development

Example circa late 2024:
```
6.0.x  — OSS EOL, commercial support only
6.1.x  — OSS active, security fixes + bug fixes
6.2.x  — OSS active (newest), full development
7.0.0-SNAPSHOT — next generation, pre-release only
```

---

## 11. Practical Version String Reference

Quick reference for anyone reading a Spring version string:

| Version String | Type | Published To | Stability |
|----------------|------|-------------|-----------|
| `7.2.0-SNAPSHOT` | Continuous build | repo.spring.io/snapshot | None |
| `7.2.0-M1` | First milestone | repo.spring.io/milestone | Low |
| `7.2.0-M2` | Second milestone | repo.spring.io/milestone | Low-Medium |
| `7.2.0-RC1` | First release candidate | repo.spring.io/milestone | High |
| `7.2.0-RC2` | Second release candidate | repo.spring.io/milestone | High |
| `7.2.0` | GA | Maven Central | Production |
| `7.2.1` | First patch | Maven Central | Production |
| `7.2.0.RELEASE` | Old-style GA (pre-6.0) | Maven Central | Production |

---

## 12. Sources and Confidence Notes

**HIGH confidence (stable, well-documented Spring conventions, unchanged for years):**
- Maven SNAPSHOT semantics (Maven specification, stable since Maven 2)
- Spring milestone/RC/GA release sequence structure
- BOM and release train concept
- `.RELEASE` suffix drop in Spring 6.0

**MEDIUM confidence (patterns match training data, exact versions need live verification):**
- Spring Boot support window exact dates
- Spring Data codename mapping to Spring Boot versions (Rl codename VERIFY)
- Exact milestone count for 7.x cycles

**LOW confidence / should be verified against live sources:**
- Spring Framework 7.x current release status (may have progressed beyond M1/M2 phase)
- Exact OSS EOL dates for 6.2.x and 7.0.x
- Current patch version numbers (3.3.x and 6.1.x highest patch)

**Primary sources to verify against:**
- https://spring.io/projects/spring-framework#support
- https://spring.io/projects/spring-boot#support
- https://github.com/spring-projects/spring-framework/releases
- https://github.com/spring-projects/spring-boot/releases
- https://repo.spring.io (artifact repository browser)
- Maven documentation on SNAPSHOT resolution: https://maven.apache.org/guides/getting-started/
