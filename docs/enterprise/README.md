# Enterprise Release Strategy

Reference documentation for feature-driven enterprise teams managing branching strategy and release workflows.

**Audience:** Mid-level developers on enterprise teams, particularly those currently using Git Flow and moving toward trunk-based development.

**Scope:** This documentation covers branching models and workflows for application/product teams delivering features. It is separate from the [OSS strategy](../oss/oss-release-strategy.md), which covers framework/library release lifecycles (versioning, milestones, release trains).

---

## Contents

| File | Description |
|------|-------------|
| [`release-strategy.md`](release-strategy.md) | Comprehensive reference — Git Flow vs. trunk-based development, feature flags, parallel feature management, migration guide, decision guide |
| [`diagrams.md`](diagrams.md) | Mermaid diagrams — branch structures, feature flag lifecycle, flag taxonomy, side-by-side scenario comparison, migration path |
| [`slides.md`](slides.md) | Marp Markdown slide deck — 12 slides for a 20-30 min team presentation |
| [`version-and-release-notes.md`](version-and-release-notes.md) | Version bump timing and location (Git Flow vs TBD), release notes authoring, tooling patterns: conventional commits, git-cliff, release-please, semantic-release |

## Topics Covered

- **Git Flow** — branch roles, merge choreography, when it works and where it breaks
- **Trunk-Based Development** — short-lived branches, CI as a prerequisite, how releases work
- **Feature flags** — Fowler's four types, rollout stages, cleanup discipline, flag debt
- **Parallel features** — how each model handles two features in flight simultaneously
- **Migration** — phased path from Git Flow to full TBD with prerequisites per stage
- **Decision guide** — which model fits which team context
- **Version and release notes** — when and where version bumps are committed, where changelogs are authored, tooling patterns (conventional commits, git-cliff, release-please, semantic-release)
