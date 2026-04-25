# OSS Release Strategy

Reference documentation for teams consuming or contributing to open source frameworks, using Spring Framework as the primary case study.

**Audience:** Developers working with Spring, Kafka, Quarkus, or Kubernetes who want to understand version strings, release lifecycles, and branch maintenance models.

**Scope:** This documentation covers OSS framework release conventions — versioning stages, branching models, and multi-train management. It is separate from the [enterprise strategy](../enterprise/README.md), which covers branching workflows for feature-driven application teams (Git Flow vs. trunk-based development).

---

## Contents

| File | Description |
|------|-------------|
| [`oss-release-strategy.md`](oss-release-strategy.md) | Comprehensive reference — branching model, release terminology, multi-train management, OSS comparisons, support tiers |
| [`diagrams.md`](diagrams.md) | Mermaid diagrams — lifecycle timeline, branch model, forward-merge flow, BOM hub, ecosystem heritage |
| [`slides.md`](slides.md) | Marp Markdown slide deck — 14 slides for a 20-30 min team presentation |

## Topics Covered

- **Spring's maintenance-branch model** — how `main`, `7.0.x`, `6.2.x` relate and why it's not git-flow
- **Fix propagation** — forward merge (`6.2.x → 7.0.x → main`) and backport cherry-picks
- **Release lifecycle** — SNAPSHOT → Milestone → RC → GA → patches, with real version examples
- **Release trains** — Spring Boot as coordinator, the BOM, what "upgrade Boot" really means
- **Support tiers** — OSS vs. commercial (Broadcom Tanzu), what happens at EOL
- **Ecosystem comparison** — Spring vs. Apache Kafka, Quarkus, Kubernetes, Next.js
- **Vocabulary heritage** — why Spring says "Milestone", Quarkus says "CR", Kubernetes says "alpha"
