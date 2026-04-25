# OSS Release Strategy Comparisons

Research synthesized from training data (knowledge cutoff August 2025) + live GitHub data where available.

---

## Apache Kafka

**Branch model:** `trunk` (not `main`) + `N.N` maintenance branches (e.g., `3.7`, `3.8`, `3.9`). No `.x` suffix.

**Release labeling:**
- `X.Y.Z-rc1`, `X.Y.Z-rc2` → `X.Y.Z` (GA)
- Only RC stage — no alpha, no beta, no milestones
- Governed by ASF PMC vote: a release is official only after a binding community vote passes

**Concurrent support:** ~2-3 concurrent minor lines. No formal LTS designation.

**Notable:** The ASF release vote is a governance gate unique to Apache Foundation projects. Any PMC member can block a release with a veto (-1 vote). This makes the release process more community-governed than Spring's maintainer-driven model.

---

## Quarkus

**Branch model:** `main` + `N.N` maintenance branches (e.g., `3.15`, `3.20`). No `.x` suffix.

**Release labeling:**
- `X.Y.Z.Alpha1` → `X.Y.Z.Beta1` → `X.Y.Z.CR1` (CR = Candidate Release, equivalent to RC — JBoss/Red Hat naming heritage) → `X.Y.Z` (GA / "Final")
- Uses Alpha/Beta explicitly, unlike Spring

**Concurrent support:** Explicit LTS releases every ~6 months. As of April 2026, active LTS lines: 3.15, 3.20, 3.27. Non-LTS minors are maintained for ~4 weeks until the next minor ships (~1 new minor/month).

**Emergency branches:** Uses out-of-band `3.20.2-emergency` branches for critical CVE patches.

**Notable differences from Spring:**
- Quarkus explicitly labels LTS vs. non-LTS in its release metadata — Spring's support tiers are determined by the support policy page, not version labels
- Alpha/Beta terminology instead of Milestones — both serve the same purpose (early feedback, not API-stable) but Quarkus stages are more granular

---

## Next.js (frontend/Node.js ecosystem)

**Branch model:** `canary` is the primary development branch (not `main`). No long-lived maintenance branches per minor version — "maintenance" means "upgrade to the latest."

**Release labeling / distribution channels:**
- `canary` — continuous pre-release builds (analogous to SNAPSHOT)
- `rc` — release candidate channel
- `latest` — stable (analogous to GA)
- Version numbers follow semver strictly: `16.2.4`, `16.3.0-canary.x`

**Concurrent support:** Essentially one supported version. Older majors receive critical security patches only. No LTS model.

**Notable differences from Spring:**
- No per-minor maintenance branches — the expectation is forward migration
- Canary vs. stable is a distribution channel, not a branch — users opt in via `npm install next@canary`
- No concept of pinnable pre-release Maven coordinates (the npm/semver model handles pre-release differently)

---

## Kubernetes

**Branch model:** `main` + `release-X.Y` maintenance branches (e.g., `release-1.33`, `release-1.34`).

**Release labeling:**
- `v1.36.0-alpha.1` → `v1.36.0-beta.0` → `v1.36.0-rc.0` → `v1.36.0` (GA)
- Explicit alpha → beta → RC → GA sequence, unlike Spring which goes SNAPSHOT → Milestone → RC → GA
- All pre-release labels are part of the version tag (semver pre-release syntax)

**Concurrent support:** N-2 policy — 3 minor versions supported simultaneously (~14 months per minor). As of April 2026: 1.33, 1.34, 1.35 active, 1.36 in development.

**Release cadence:** Time-boxed — 3 minors per year. Monthly patch releases on the 28th of each month.

**Notable differences from Spring:**
- Strictly time-boxed releases (features that miss the window wait for the next minor)
- N-2 support window is hard and predictable — Spring's support windows vary by generation
- Alpha/beta/rc are semver pre-release identifiers; Spring's M1/RC1 suffixes are Maven version qualifiers (different ecosystems, different tooling implications)

---

## Comparison Table

| Project | Primary Dev Branch | Maintenance Branch Pattern | Pre-release Labels | GA Label | LTS? | Concurrent Supported Versions |
|---|---|---|---|---|---|---|
| **Spring Framework** | `main` | `N.N.x` (e.g. `7.0.x`) | SNAPSHOT → M1/M2 → RC1 | `7.0.0` (plain) | Via Broadcom commercial | ~2-3 minor lines |
| **Apache Kafka** | `trunk` | `N.N` (e.g. `3.8`) | RC1/RC2 only | `3.8.0` | No | ~2-3 minor lines |
| **Quarkus** | `main` | `N.N` (e.g. `3.20`) | Alpha1 → Beta1 → CR1 | `3.27.0` | Yes (explicit OSS LTS) | LTS lines + current minor |
| **Next.js** | `canary` | None (one active version) | canary channel | `16.2.4` (latest channel) | No | 1 (current major) |
| **Kubernetes** | `main` | `release-N.N` | alpha → beta → rc | `v1.35.0` | No (N-2 policy) | 3 minors (N-2) |

---

## Key Observations

**Spring's approach is library-centric.** Milestones and RCs are published as real Maven/Gradle coordinates — consumers can pin `7.2.0-M1` in their `pom.xml` and get a reproducible build. Kubernetes alpha/beta tags are deployment artifacts; Quarkus pre-releases are also coordinates, but fewer of them (no milestones, just alpha/beta/CR).

**The `.x` in Spring branch names is distinctive.** `7.0.x` means "all patch releases of the 7.0 minor." Kafka uses `3.8`, Kubernetes uses `release-1.35`. Spring's convention makes the intent explicit in the branch name.

**Alpha/Beta vs. Milestone terminology is ecosystem heritage.**
- Spring/Pivotal heritage: SNAPSHOT → Milestone (M1, M2) → RC → GA
- Red Hat/JBoss heritage (Quarkus, Hibernate): Alpha → Beta → CR (Candidate Release) → Final
- Linux/systems heritage (Kubernetes): alpha → beta → rc → stable
- Apache heritage: (no pre-release or) alpha → beta → RC → (ASF vote) → release

**Support window transparency varies.** Kubernetes N-2 is the clearest — you always know exactly which 3 versions are supported. Spring requires checking the support policy page. Quarkus LTS designation is in the release name but the policy requires reading docs.
