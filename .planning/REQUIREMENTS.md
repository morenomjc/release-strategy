# Requirements: OSS Release Strategy Reference

**Defined:** 2026-04-25
**Core Value:** A developer on the team can read this and immediately understand why a version is labeled `7.2.0-M3` vs `7.2.0-RC1` vs `7.2.0` — and how the project's git branches map to that lifecycle.

## v1 Requirements

### Branching Strategy

- [x] **BRANCH-01**: Explain Spring's maintenance-branch model (not git-flow, not trunk-based) with real branch names
- [x] **BRANCH-02**: Explain branch naming convention (`N.N.x` pattern, `main` for next gen)
- [x] **BRANCH-03**: Explain how fixes propagate across branches (forward merge + backport)
- [x] **BRANCH-04**: Explain how branch lifetimes map to the support policy

### Release Terminology

- [x] **TERM-01**: Define SNAPSHOT — mutable, CI artifact, never on Maven Central, resolves to "latest build"
- [x] **TERM-02**: Define Milestone (M1, M2...) — fixed Maven coordinate, API-feedback stage, not API-stable
- [x] **TERM-03**: Define Release Candidate (RC1, RC2) — feature-complete, API-frozen, final verification
- [x] **TERM-04**: Define GA — plain version string, the only artifact on Maven Central
- [x] **TERM-05**: Explain the full lifecycle sequence: SNAPSHOT → M1 → [M2] → RC1 → [RC2] → GA → patches → EOL
- [x] **TERM-06**: Explain patch releases — bugfix-only, no milestones/RCs, go straight to GA

### Multi-Train Release Management

- [x] **TRAIN-01**: Explain what a "release train" is and how Spring Boot orchestrates it via BOM
- [x] **TRAIN-02**: Explain concurrent version support — active dev vs. current GA vs. maintenance vs. EOL
- [x] **TRAIN-03**: Explain OSS vs. commercial support tiers (Broadcom Tanzu) and what changes at EOL

### OSS Comparisons

- [x] **COMP-01**: Compare Spring's approach with Apache projects (Kafka) — RC-only model, ASF vote
- [x] **COMP-02**: Compare with Quarkus — Alpha/Beta/CR vs. Milestone/RC, explicit LTS
- [x] **COMP-03**: Compare with Kubernetes — strict N-2 support, time-boxed releases, alpha→beta→rc→GA
- [x] **COMP-04**: Produce a comparison table covering all projects

### Outputs

- [x] **OUT-01**: Produce `docs/oss-release-strategy.md` — comprehensive Markdown reference doc
- [x] **OUT-02**: Produce `docs/presentation-outline.md` — slide-by-slide outline for team presentation

## v2 Requirements

### Deeper Tooling

- **TOOL-01**: Show how to configure Maven/Gradle to resolve snapshots and milestones (repo config)
- **TOOL-02**: Explain GitHub Actions release automation patterns used by Spring

### Extended Comparisons

- **EXT-01**: Cover Python ecosystem (PEP 440 versioning, PyPI pre-release conventions)
- **EXT-02**: Cover Go module versioning and its major-version branching convention

## v1 Enterprise Requirements

### Enterprise Branching Models

- [x] **ENT-01**: Compare Git Flow and trunk-based development side-by-side across: stable branch identity, parallel feature handling, release process, hotfix process, CI requirements
- [x] **ENT-02**: Explain feature flags — Fowler's four types (release, ops, experiment, permission), lifecycle stages, and cleanup discipline
- [x] **ENT-03**: Document how to run parallel features safely in both models
- [x] **ENT-04**: Provide a migration path from Git Flow to trunk-based development with prerequisites per stage
- [x] **ENT-05**: Produce `docs/enterprise/release-strategy.md` and `docs/enterprise/diagrams.md` as standalone enterprise reference docs

## Out of Scope

| Feature | Reason |
|---------|--------|
| CI/CD pipeline setup | Focus is concepts, not tooling implementation |
| Private/enterprise release strategies (implementation) | Strategy concepts only, not implementation details |
| Full SemVer spec explanation | Assume basic familiarity |
| Spring Boot auto-configuration deep dive | Release strategy only, not internals |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| BRANCH-01 | Phase 1 | Complete |
| BRANCH-02 | Phase 1 | Complete |
| BRANCH-03 | Phase 1 | Complete |
| BRANCH-04 | Phase 1 | Complete |
| TERM-01 | Phase 1 | Complete |
| TERM-02 | Phase 1 | Complete |
| TERM-03 | Phase 1 | Complete |
| TERM-04 | Phase 1 | Complete |
| TERM-05 | Phase 1 | Complete |
| TERM-06 | Phase 1 | Complete |
| TRAIN-01 | Phase 1 | Complete |
| TRAIN-02 | Phase 1 | Complete |
| TRAIN-03 | Phase 1 | Complete |
| COMP-01 | Phase 1 | Complete |
| COMP-02 | Phase 1 | Complete |
| COMP-03 | Phase 1 | Complete |
| COMP-04 | Phase 1 | Complete |
| OUT-01 | Phase 1 | Complete |
| OUT-02 | Phase 2 | Complete |

**Coverage:**
- v1 requirements: 19 total
- Mapped to phases: 19
- Unmapped: 0 ✓

---
*Requirements defined: 2026-04-25*
