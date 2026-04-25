# Phase 1 Plan: Core Reference Doc

**Phase:** 1 — Core Reference Doc
**Goal:** A developer can open one document and get the complete mental model for Spring's release strategy — branch naming, fix flow, lifecycle stages, concurrent trains, and how Spring compares to other major OSS projects
**Status:** COMPLETE — delivered during project initialization

---

## Context

All research was conducted during `/gsd-new-project` initialization:
- `.planning/research/branching-strategy.md` — live GitHub data on Spring's branch model
- `.planning/research/release-terminology.md` — full lifecycle and Maven semantics
- `.planning/research/oss-comparisons.md` — Spring vs Kafka vs Quarkus vs Kubernetes vs Next.js

## Tasks

- [x] **Task 1**: Write `docs/oss-release-strategy.md` covering all BRANCH-*, TERM-*, TRAIN-*, COMP-* requirements
  - Quick-reference table (version string → type → repo → production-safe?)
  - Branching model section with live branch inventory and forward-merge diagram
  - Release terminology section with real version examples for each stage
  - Release trains and BOM section
  - Concurrent version support table
  - OSS vs. commercial support tiers
  - OSS ecosystem comparison table (5 projects)
  - Ecosystem heritage section

## Verification

**Success criteria check:**
1. ✓ Doc contains quick-reference table mapping version strings to stability and repo
2. ✓ Branch diagram shows `6.2.x → 7.0.x → main` forward-merge flow
3. ✓ All 5 version types (SNAPSHOT/M/RC/GA/patch) defined with real examples
4. ✓ Comparison section covers Spring, Kafka, Quarkus, Kubernetes, Next.js

**Output:** `docs/oss-release-strategy.md`

---
*Plan created: 2026-04-25 | Status: Complete*
