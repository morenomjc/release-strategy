# Phase 2 Plan: Diagrams and Presentation Slides

**Phase:** 2
**Goal:** Produce a complete, deliverable Marp slide deck and Mermaid diagram set that a team member can present directly — no additional authoring required
**Status:** IN PROGRESS

---

## Scope

Expanding from the existing `docs/presentation-outline.md` (14-slide outline) to actual deliverable files:

1. **`docs/diagrams.md`** — Mermaid diagrams for all three diagram callouts in the outline:
   - Branch model diagram (main → 7.0.x → 6.2.x → 6.1.x with forward-merge arrows)
   - Release lifecycle timeline (SNAPSHOT → M1 → M2 → RC1 → RC2 → GA → patch)
   - OSS project comparison table (rendered as a Mermaid or Markdown table)

2. **`docs/slides.md`** — Marp Markdown slide deck (14 slides) implementing the presentation outline with:
   - Marp frontmatter (`marp: true`, theme, paginate)
   - Each slide from the outline with key points as bullet lists
   - Mermaid diagrams embedded where the outline calls them out
   - Speaker notes as HTML comments (`<!-- speaker note -->`)

## Tasks

- [ ] **Task 1**: Write `docs/diagrams.md` with all three Mermaid diagrams
- [ ] **Task 2**: Write `docs/slides.md` as a complete Marp deck implementing all 14 slides

## References

- `docs/presentation-outline.md` — slide-by-slide outline to implement
- `docs/oss-release-strategy.md` — source content for all facts and version examples
- `.planning/research/branching-strategy.md` — branch data for diagrams

## Verification

1. `docs/slides.md` has valid Marp frontmatter and 14 slide separators (`---`)
2. Every diagram callout in the outline is implemented in `docs/diagrams.md`
3. Diagrams are embedded in the slide deck at the correct slides
4. All version numbers match `docs/oss-release-strategy.md`

---
*Plan updated: 2026-04-25*
