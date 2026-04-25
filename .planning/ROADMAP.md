# Roadmap: OSS Release Strategy Reference

**Project:** OSS Release Strategy Reference
**Core Value:** A developer on the team can read this and immediately understand why a version is labeled `7.2.0-M3` vs `7.2.0-RC1` vs `7.2.0` — and how the project's git branches map to that lifecycle.
**Granularity:** Coarse (documentation project — 2 phases)

---

## Phases

- [x] **Phase 1: Core Reference Doc** — Write the comprehensive Markdown reference doc covering branching strategy, release terminology, multi-train management, OSS comparisons, and support tiers
- [x] **Phase 2: Presentation Outline** — Write the slide-by-slide presentation outline for team delivery

---

## Phase Details

### Phase 1: Core Reference Doc
**Goal**: A developer can open one document and get the complete mental model for Spring's release strategy — branch naming, fix flow, lifecycle stages, concurrent trains, and how Spring compares to other major OSS projects
**Depends on**: Nothing (first phase)
**Requirements**: BRANCH-01, BRANCH-02, BRANCH-03, BRANCH-04, TERM-01, TERM-02, TERM-03, TERM-04, TERM-05, TERM-06, TRAIN-01, TRAIN-02, TRAIN-03, COMP-01, COMP-02, COMP-03, COMP-04, OUT-01
**Success Criteria** (what must be TRUE):
  1. A developer reading the doc can correctly identify what `7.2.0-M3`, `7.2.0-RC1`, and `7.2.0` mean without consulting any external source
  2. A developer can explain what branch a bug fix should target and how it reaches other supported versions
  3. The doc contains a quick-reference table mapping version string patterns to stability level and artifact repository
  4. The comparison section lets a developer recognize Spring's patterns when they encounter Quarkus, Kafka, or Kubernetes versioning
**Plans**: `.planning/phases/01-core-reference-doc/01-PLAN.md`

### Phase 2: Presentation Outline
**Goal**: A team member can pick up the presentation outline and deliver a 20-30 minute talk that builds shared vocabulary and mental model across the team
**Depends on**: Phase 1
**Requirements**: OUT-02
**Success Criteria** (what must be TRUE):
  1. The outline has 12-15 numbered slides each with a title, 3-5 key points, and speaker note suggestions for key slides
  2. Diagrams to draw are called out explicitly (branch diagram, lifecycle flow, comparison table)
  3. The arc of the talk moves from "why does this matter" to "here is how it works" to "how we compare to the ecosystem"
**Plans**: `.planning/phases/02-presentation-outline/02-PLAN.md`
**UI hint**: no

---

## Progress Table

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Core Reference Doc | 1/1 | Complete | 2026-04-25 |
| 2. Presentation Outline | 1/1 | Complete | 2026-04-25 |

---

## Coverage

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

**Coverage:** 19/19 v1 requirements mapped. No orphans. ✓

---
*Roadmap created: 2026-04-25*
