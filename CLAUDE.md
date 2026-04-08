# Pace Dashboard Preview

## What This Is

Preview repo for LOD's weekly hotel pace reports. Visual QA happens here before changes propagate to the production Jinja2 template in `pace-pipeline`.

## Architecture

- **Single-file HTML** — no external dependencies (no CDN links, no Google Fonts)
- **Vanilla JS (ES5)** — uses `var`, not `const`/`let`, for broad browser compat
- **Inline SVG** for charts — generated server-side, re-rendered client-side on toggle changes
- **Print-first** — every design decision must work on letter-size PDF (0.35in sides, 0.45in top-bottom)

## Design Context

Brand and design guidelines are in `.impeccable.md`. Key points:

- **Font stack:** `'Inter', 'DM Sans', -apple-system, sans-serif` — Inter renders if installed, falls back gracefully. Do NOT add external font links (violates single-file constraint).
- **Colors:** `--lod-black` (#000000) for structure, `--lod-near-black` (#1A1A1A) for body text, `--lod-electric-blue` (#0A84FF) for accents, `--pos` (#167D48) / `--neg` (#d63b3b) for deltas
- **Typography:** Display headings = light weight (300). Data values = bold (700). Section labels = 12px uppercase letter-spaced.
- **Accessibility:** WCAG AA target. Deltas use arrows + color (never color alone). All pills are keyboard-accessible.

## Specs

- `docs/specs/DESIGN.md` — Approved v5 design specification
- `docs/specs/AESTHETICS-FINDINGS.md` — UX critique findings (11 P1, 9 P2, 7 P3)
- `docs/specs/FLOW-STATUS.md` — Vibe coding flow tracker

## Workflow

1. Make CSS/HTML/JS changes in `dashboard-v5.html`
2. Visual QA in browser + print preview
3. Once approved, propagate to `pace-pipeline/src/dashboard/template.html` (Jinja2 version)
4. Run integration tests in pace-pipeline to verify data rendering

## Chart Behavior

- **≤3 months:** daily data points (one bar per day)
- **≥6 months:** weekly aggregation (ISO week buckets)
- **Bar color:** green (`--pos`) when ahead of comparison, red (`--neg`) when behind, blue (`--lod-electric-blue`) when no comparison
- **ADR line:** dashed blue (`#0A84FF`), toggleable via pill (default on)
- **Right axis:** ADR scale in blue when ADR active
- **Tooltip:** Revenue OTB + comparison + delta + ADR + Occ% (ADR/Occ have no comparison shown)
- **Daily occ delta:** approximate — uses `comparison.rooms / total_rooms * 100` as proxy (daily comparison data lacks occ fields)

## Table Layout

Interleaved columns: Month | Rooms | Rev OTB | Rev Δ | ADR OTB | ADR Δ | Occ% OTB | Occ% Δ | RevPAR | Pickup Rooms | Pickup Revenue | Pickup ADR

- `.col-sep` class on group boundary cells (not nth-child selectors)
- `.rev-vs-pos` / `.rev-vs-neg` on Revenue Δ cells for background color
- `.day-expand-row td.delta-positive/negative` for daily delta text color (CSS specificity override)
- OOO indicator: group header for Occ%, second line for RevPAR, omitted from subheaders

## Next Steps

- Final aesthetic pass (font options discussion deferred)
- Re-run `/critique` to score improvements (target: 32+/40)
- Propagate approved changes to production template (`pace-pipeline/src/dashboard/template.html`)
- Hook up to email ingestion pipeline for automated generation
