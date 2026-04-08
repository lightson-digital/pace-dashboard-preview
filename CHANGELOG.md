# Changelog

All notable changes to the pace dashboard preview.

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/)

## [Unreleased]

### Fixed (2026-04-08)
- ADR right Y-axis no longer disappears in print/PDF export (added viewBox to SVG, overflow:visible in print CSS)

### Added (2026-04-08)
- ADR comparison line in hero chart — dashed lighter blue line shows comparison ADR when WoW/STLY/Budget is active
- Chart tooltip shows ADR delta vs comparison (e.g. "+$48 vs $516")
- Chart tooltip shows Occ% delta vs comparison where data is available (e.g. "-11.4pp vs 71.6%")
- "ADR (Comparison)" legend entry appears when comparison mode is active

### Changed (2026-04-08)
- Current ADR line color softened from #0A84FF to #7FBFFF (lighter blue, less visual weight vs bars)
- Current ADR line style changed from dashed to solid (comparison is now dashed)

### Changed — Functional Improvements (2026-04-07)

**Hero Chart**
- Chart height scaled to 460px (from 320px) for better visual presence
- Daily data points for ≤3-month filter; weekly aggregation for ≥6-month filter
- Bar color reflects ahead/behind status vs comparison period (green/red/blue fallback)
- ADR overlay changed to dashed blue (#0A84FF) line with toggle pill (default on)
- ADR right axis labels styled in blue to match line
- X-axis labels show month name only at first bar of each month
- Light dashed vertical gridlines at month boundaries
- Hover tooltip: Revenue OTB, comparison, delta amount/%, snapshot ADR and Occ%

**Monthly Breakdown Table**
- Restructured to interleaved layout: KPI + comparison columns adjacent (Revenue OTB | Rev Δ | ADR OTB | ADR Δ | Occ% OTB | Occ% Δ | RevPAR | Pickup)
- Revenue Δ cell background colored green (#b8e8c8) / red (#f5b0b0) for ahead/behind
- Comparison column headers update dynamically with toggle ("vs STLY", "vs WoW", "vs Budget")
- Occ% subheader: OOO indicator moved to group header only
- RevPAR header: OOO indicator on second line
- Pickup "Rev" column header spelled out to "Revenue"
- All headers set to nowrap to prevent wrapping at default screen width
- Column group separators use `.col-sep` class (replacing fragile nth-child selectors)

**Daily Breakdown**
- Day-of-week prefix on daily rows ("Sat Mar 1")
- Revenue Δ cell background color fix (CSS specificity override for .day-expand-row td)
- Delta text colors (ADR vs, Occ vs) now show green/red matching monthly rows
- Approximate Occ% delta computed from comparison.rooms / total_rooms
- Hover highlight on daily rows (slightly darker bg on hover)

**Color Consistency**
- All green/red aligned to --pos (#167D48) / --neg (#d63b3b) family
- Cell backgrounds: #b8e8c8 (green) / #f5b0b0 (red)
- Tooltip delta text: #b8e8c8 / #f5b0b0 (matches cell backgrounds, readable on dark bg)
- Hover states for daily rev-vs cells: #a8ddb8 / #eba0a0

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
