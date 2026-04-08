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

## Next Steps

- Functional critique session (interactions, data accuracy, edge cases)
- Re-run `/critique` to score improvements (target: 32+/40)
- Propagate approved changes to production template
