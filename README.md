# Pace Dashboard Preview

Preview repository for Lights On Digital's weekly hotel pace reports. Contains generated HTML dashboards for visual QA before changes land in the production template (`pace-pipeline/src/dashboard/template.html`).

## Current Version

**v5** — Rolling 12-month pace report with comparison toggles (WoW/STLY/Budget), OOO occupancy toggle, hero chart with time filter and ADR overlay, interleaved monthly breakdown table with expandable daily detail, and print-to-PDF support.

## Files

- `dashboard-v5.html` — Latest generated dashboard (KAH, snapshot 2026-03-11)
- `dashboard-v4.html` — Previous version (pre-feedback revision)
- `dashboard-demo.html` — Demo with fabricated data
- `index.html` — Passcode-gated access page
- `.impeccable.md` — Design context for impeccable skills (LOD brand, typography, color)

## Design Specs

- `docs/specs/DESIGN.md` — v5 design specification
- `docs/specs/AESTHETICS-FINDINGS.md` — UX critique and aesthetics assessment (27 findings)
- `docs/specs/FLOW-STATUS.md` — Vibe coding flow tracker

## Architecture

Single self-contained HTML file per hotel per snapshot. No external dependencies.

- All data pre-computed and embedded as JSON
- Vanilla CSS + JS — no frameworks
- Inline SVG charts — no chart libraries
- `@media print` optimized for letter-size PDF

## Brand

LOD-branded but light theme: black header/footer, white data area, Inter font family (fallback), Electric Blue (#0A84FF) accent, muted delta colors (#167D48 green, #d63b3b red).

## Related

- Production template: `lightson-digital/pace-reporting` → `src/dashboard/template.html`
- Data pipeline: `lightson-digital/pace-reporting` → `src/dashboard/generate.py`
