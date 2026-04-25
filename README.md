# OSS Release Strategy Reference

A team reference guide explaining how open source projects like Spring Framework manage release versioning and branching strategy. Uses Spring as the primary case study while covering general OSS patterns across the ecosystem.

## Contents

| File | Description |
|------|-------------|
| [`docs/oss-release-strategy.md`](docs/oss-release-strategy.md) | Comprehensive reference doc — branching model, release terminology, multi-train management, OSS comparisons |
| [`docs/diagrams.md`](docs/diagrams.md) | Mermaid diagrams — lifecycle timeline, branch model, forward-merge flow, BOM hub, ecosystem heritage |
| [`docs/slides.md`](docs/slides.md) | Marp Markdown slide deck — 14 slides for a 20-30 min team presentation |
| [`docs/presentation-outline.md`](docs/presentation-outline.md) | Slide-by-slide outline with speaker notes |

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
marp docs/slides.md --pdf

# Export to HTML
marp docs/slides.md --html

# Live preview
marp docs/slides.md --preview
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
