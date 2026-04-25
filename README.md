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

## Enterprise Strategy

For feature-driven application teams, especially those on Git Flow and heading toward trunk-based development.

| File | Description |
|------|-------------|
| [`docs/enterprise/release-strategy.md`](docs/enterprise/release-strategy.md) | Comprehensive reference — Git Flow vs. trunk-based development, feature flags, parallel features, migration guide |
| [`docs/enterprise/diagrams.md`](docs/enterprise/diagrams.md) | Mermaid diagrams — branch structures, flag lifecycle, flag taxonomy, scenario comparison, migration path |
| [`docs/enterprise/slides.md`](docs/enterprise/slides.md) | Marp Markdown slide deck — 12 slides for a 20-30 min team presentation |

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

