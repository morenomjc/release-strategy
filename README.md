# Release Strategy Reference

A team reference guide covering two distinct release strategy contexts:

1. **OSS Strategy** — how open source frameworks and libraries (Spring, Kafka, Quarkus, Kubernetes) manage versioning, release lifecycles, and branch maintenance
2. **Enterprise Strategy** — how feature-driven application teams manage branching workflows (Git Flow vs. trunk-based development), feature flags, and migration between models

---

## OSS Strategy

For teams consuming or contributing to open source frameworks. Uses Spring Framework as the primary case study.

| File | Description |
|------|-------------|
| [`docs/oss/oss-release-strategy.md`](docs/oss/oss-release-strategy.md) | Comprehensive reference doc — branching model, release terminology, multi-train management, OSS comparisons |
| [`docs/oss/diagrams.md`](docs/oss/diagrams.md) | Mermaid diagrams — lifecycle timeline, branch model, forward-merge flow, BOM hub, ecosystem heritage |
| [`docs/oss/slides.md`](docs/oss/slides.md) | Marp Markdown slide deck — 14 slides for a 20-30 min team presentation |
| [`docs/oss/presentation-outline.md`](docs/oss/presentation-outline.md) | Slide-by-slide outline with speaker notes |

## Enterprise Strategy

For feature-driven application teams, especially those on Git Flow and heading toward trunk-based development.

| File | Description |
|------|-------------|
| [`docs/enterprise/release-strategy.md`](docs/enterprise/release-strategy.md) | Comprehensive reference — Git Flow vs. trunk-based development, feature flags, parallel features, migration guide |
| [`docs/enterprise/diagrams.md`](docs/enterprise/diagrams.md) | Mermaid diagrams — branch structures, flag lifecycle, flag taxonomy, scenario comparison, migration path |
| [`docs/enterprise/slides.md`](docs/enterprise/slides.md) | Marp Markdown slide deck — 12 slides for a 20-30 min team presentation |

## Quick Reference

| Version string | Type | Repository | Use in production? |
|---|---|---|---|
| `7.2.0-SNAPSHOT` | Mutable CI build | repo.spring.io/snapshot | No |
| `7.2.0-M1` | Milestone — API in flux | repo.spring.io/milestone | No |
| `7.2.0-RC1` | Release candidate — API frozen | repo.spring.io/milestone | No |
| `7.2.0` | GA | Maven Central | Yes |
| `7.2.1` | Patch — bugfix only | Maven Central | Yes |

## Presenting the Slides

Install [Marp CLI](https://github.com/marp-team/marp-cli) or the [Marp VS Code extension](https://marketplace.visualstudio.com/items?itemName=marp-team.marp-vscode), then:

```bash
# Export to PDF
marp docs/oss/slides.md --pdf

# Export to HTML
marp docs/oss/slides.md --html

# Live preview
marp docs/oss/slides.md --preview

# Enterprise deck
marp docs/enterprise/slides.md --pdf
```

## Topics Covered

- Spring's **maintenance-branch model** — how `main`, `7.0.x`, `6.2.x` relate and why it's not git-flow
- **Fix propagation** — forward merge (`6.2.x → 7.0.x → main`) and backport cherry-picks
- **Release lifecycle** — SNAPSHOT → Milestone → RC → GA → patches, with real version examples
- **Release trains** — Spring Boot as coordinator, the BOM, what "upgrade Boot" really means
- **Support tiers** — OSS vs. commercial (Broadcom Tanzu), what happens at EOL
- **Ecosystem comparison** — Spring vs. Apache Kafka, Quarkus, Kubernetes, Next.js
- **Vocabulary heritage** — why Spring says "Milestone", Quarkus says "CR", Kubernetes says "alpha"

## Verification

Some support EOL dates in the reference doc are approximate. Verify current dates at:
https://spring.io/projects/spring-framework#support
