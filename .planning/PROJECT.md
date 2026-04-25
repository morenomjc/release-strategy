# OSS Release Strategy Reference

## What This Is

A team reference guide explaining how open source projects like Spring Framework manage release versioning and branching strategy. Uses Spring as the primary case study while covering general OSS patterns applicable across the ecosystem. Output is Markdown docs + presentation slides.

## Core Value

A developer on the team can read this and immediately understand why a version is labeled `7.2.0-M3` vs `7.2.0-RC1` vs `7.2.0` — and how the project's git branches map to that lifecycle.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] Explain OSS git branching strategies (git flow, trunk-based, maintenance branches) with Spring as the primary example
- [ ] Explain how projects manage multiple concurrent release trains (e.g. 7.x active + 6.x maintained)
- [ ] Define and differentiate release terminology: SNAPSHOT, Milestone (M1/M2), Release Candidate (RC), GA, alpha, beta
- [ ] Show how version numbers map to branches and release lifecycle stages
- [ ] Compare Spring's approach to other notable OSS projects (Quarkus, Apache Commons, etc.)
- [ ] Produce Markdown reference doc(s) in this repo
- [ ] Produce a presentation outline / slides for team delivery

### Out of Scope

- CI/CD pipeline implementation details — focus is strategy/concepts, not tooling configuration
- Private/enterprise release strategies — scope is public OSS patterns
- Semantic versioning spec itself (SemVer) — assume basic familiarity, only cover OSS-specific conventions layered on top

## Context

- Audience: development team — onboarding devs and aligning team on release conventions
- Motivation: team needs shared vocabulary and mental model before contributing to or adopting OSS-style release patterns
- Primary case study: Spring Framework (spring-projects/spring-framework on GitHub)
- Output formats: Markdown doc(s) for repo reference + slides/presentation deck

## Constraints

- **Accuracy**: All claims about Spring's process should be verifiable from their public GitHub, Wiki, or release notes
- **Clarity**: Terminology section must be concrete — definitions with real version number examples, not abstract descriptions

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Spring as primary case study | Well-documented, widely known, has clear multi-train history | — Pending |
| General OSS patterns scope | Team will encounter multiple projects, not just Spring | — Pending |

---
*Last updated: 2026-04-25 — project initialized*
