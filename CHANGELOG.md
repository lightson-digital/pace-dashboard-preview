# Changelog

All notable changes to the pace dashboard preview.

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/)

## [Unreleased]

### Changed — Aesthetics Refresh (2026-04-07)

**Typography**
- Font stack changed from system fonts to Inter/DM Sans with system fallback
- Hotel name (h1) changed from bold 28px to light 36px (LOD signature weight)
- KPI metric values increased from 26px to 36px for "monumental" readability
- Section headers converted to LOD label style (12px uppercase, letter-spaced)
- Chart SVG font aligned with page font stack
- All label letter-spacing standardized to 0.1em

**Colors**
- Split `--lod-black` (#000000) from `--lod-near-black` (#1A1A1A)
- Replaced iOS neon delta colors with WCAG AA-compliant muted palette:
  - Green: #30D158 -> #167D48 (4.6:1 contrast)
  - Red: #FF453A -> #d63b3b (4.6:1 contrast)
  - Orange: #FF9F0A -> #d97706
- Renamed `--lod-blue` to `--lod-electric-blue`, added `--ghost` token

**Layout**
- Added black header with white text per DESIGN.md spec
- Added black footer with confidentiality notice and hotel name
- Variable section spacing (24-36px) replacing uniform 28px
- KPI card padding increased, gap widened to 20px
- Table column group separators (left borders at Delta and Pickup groups)

**Polish**
- Delta formatting: added directional arrows (filled triangles), proper minus sign (U+2212)
- Month rows: added expand chevron with rotation animation
- Print CSS: page break controls, spec-compliant margins, chart/KPI break-inside protection
- Keyboard accessibility: tabindex, role="button", onkeydown on all interactive pills
- Focus-visible outline styles for keyboard navigation
- Day expansion rows: removed inline styles, added blue left accent border
- Chart legend aligned with plot area left edge
- Footer dynamically populated with hotel name and snapshot date

### Added
- `.impeccable.md` — Design context for impeccable skills
- `docs/specs/AESTHETICS-FINDINGS.md` — Full UX critique and assessment findings

## [1.0.0] - 2026-04-07

### Added
- v5 dashboard with rolling 12-month view, comparison toggles, OOO toggle, hero chart with time filter
- Monthly breakdown table with expandable daily detail
- Print/PDF optimization with color preservation
- Budget pill (disabled — no data yet)
- Design spec, implementation plan, and flow status docs

## [0.2.0] - 2026-03-15

### Added
- Non-gated demo dashboard with fabricated data

## [0.1.0] - 2026-03-10

### Added
- Initial passcode-gated preview page
