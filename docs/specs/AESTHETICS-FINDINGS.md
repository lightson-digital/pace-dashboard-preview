# Pace Dashboard v5 — Aesthetics Findings

**Date:** 2026-04-07
**Status:** Findings complete. Awaiting implementation approval.
**File under review:** `dashboard-v5.html` (73KB, KAH 2026-03-11 snapshot)
**Design context:** `.impeccable.md` (established this session)
**Assessments run:** critique, typeset, colorize, arrange, polish

---

## Design Health Score: 27/40 (Acceptable)

| # | Heuristic | Score | Key Issue |
|---|-----------|-------|-----------|
| 1 | Visibility of System Status | 3 | No expand/collapse indicator on month rows |
| 2 | Match System / Real World | 4 | Speaks hotelier language perfectly |
| 3 | User Control and Freedom | 3 | Good toggles; no bulk expand/collapse |
| 4 | Consistency and Standards | 2 | Brand drift: system fonts, iOS colors, missing header/footer |
| 5 | Error Prevention | 3 | Budget pill disabled gracefully |
| 6 | Recognition Rather Than Recall | 3 | WoW/STLY abbreviations assume LOD vocabulary |
| 7 | Flexibility and Efficiency | 2 | No keyboard shortcuts or bulk actions |
| 8 | Aesthetic and Minimalist Design | 2 | Clean but unbranded; KPI cards lack impact |
| 9 | Error Recovery | 3 | Graceful null handling throughout |
| 10 | Help and Documentation | 2 | Footnotes explain some concepts; no contextual help |

**Anti-patterns verdict:** Not AI slop, but unbranded prototype. Two tells: iOS system color palette (#30D158, #FF453A) and generic system font stack. A designer would clock it as "functional MVP with zero brand investment."

---

## What's Working

1. **Information architecture is spot-on.** KPI banner -> hero chart -> monthly table -> expandable daily detail. The "3 seconds -> 10 seconds -> deep dive" scanning model works.

2. **Comparison toggle pattern is clean.** WoW/STLY/Budget pills with disabled state, OOO toggle, chart filter — simple, discoverable, no unnecessary complexity.

3. **Null/edge case handling is thorough.** Em dashes for missing data, graceful budget-disabled state, pickup ADR guard for diverging signs.

---

## Consolidated Findings

Deduplicated across all five assessments. Findings are grouped by severity, then by implementation domain (what skill/pass would fix them).

### P1 — Must Fix (11 findings)

#### P1-1: Font stack is not LOD brand
**Source:** typeset, critique, polish
**Current:** `--font-main: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif`
**Required:** `--font-main: 'Inter', 'DM Sans', -apple-system, sans-serif`
**Action:** Update `--font-main` to `'Inter', 'DM Sans', -apple-system, sans-serif`. Do NOT add a Google Fonts `<link>` — the DESIGN.md requires a single-file artifact with no external dependencies. Inter/DM Sans will render if installed on the machine; otherwise the system font fallback applies. For future consideration: base64-embed Inter via `@font-face` with `data:` URI to guarantee the font without an external request.
**Why:** Every text element renders in a non-brand font. This is the foundation — all other typography fixes depend on it.

#### P1-2: Display heading uses bold instead of LOD signature light weight
**Source:** typeset, critique, arrange
**Current:** `.report-header h1 { font-size: 28px; font-weight: 700; letter-spacing: -0.5px; }`
**Required:** `font-size: 36px; font-weight: 300; letter-spacing: -0.75px;`
**Why:** LOD's brand signature is light-weight display text. Bold h1 is the single biggest departure from LOD visual identity. At 36px/300, it pairs with KPI values at 36px/700 — same size, opposite weights.

#### P1-3: KPI metric values too small for "monumental" intent
**Source:** typeset, arrange, critique
**Current:** `.kpi-current { font-size: 26px; font-weight: 700; letter-spacing: -0.5px; }`
**Required:** `font-size: 36px; font-weight: 700; letter-spacing: -1px; line-height: 1.1;`
**Why:** At 26px, the revenue number is only marginally larger than body text. The KPI banner answers "am I ahead or behind?" — it needs to command attention at arm's length on a printed page.

> **Implementation note:** P1-1, P1-2, and P1-3 form the LOD typographic triad. Apply together — they're interdependent.

#### P1-4: Delta colors are iOS neon — fail WCAG AA and look garish on print
**Source:** colorize, critique, polish
**Current -> Required:**

| Token | Current | New name | New value | WCAG on white |
|-------|---------|----------|-----------|---------------|
| `--lod-green` | `#30D158` (2.02:1 FAIL) | `--pos` | `#167D48` | ~4.6:1 PASS |
| `--lod-red` | `#FF453A` (3.41:1 FAIL) | `--neg` | `#d63b3b` | 4.62:1 PASS |
| `--lod-orange` | `#FF9F0A` (2.06:1 FAIL) | `--watch` | `#d97706` | 3.19:1 (large text only) |

**Why:** Current neon colors fail WCAG AA at every combination. They wash out on laser printers. The muted palette was chosen in the DESIGN.md spec specifically for cross-medium use. Green darkened from spec's `#1a8c52` (4.27:1, borderline fail) to `#167D48` for AA clearance.

#### P1-5: `--lod-black` conflates true black and near-black
**Source:** colorize
**Current:** `--lod-black: #1A1A1A` used for everything (header borders, active pills, body text, ADR line)
**Required:** Split into two tokens:
- `--lod-black: #000000` — header/footer backgrounds, heavy borders
- `--lod-near-black: #1A1A1A` — body text, ADR line, data values

**Why:** The spec defines these as distinct tokens with different purposes. Conflating them means the header/footer can never be true black without making body text too dark.

#### P1-6: Report header missing black background per spec
**Source:** arrange, colorize, critique
**Current:** White background, plain text, 2px bottom border.
**Required:**
```css
.report-header {
  background: var(--lod-black);
  color: var(--lod-white);
  padding: 28px 32px;
  margin: -32px -40px 32px -40px; /* bleed to body edges */
  border-bottom: none;
}
.report-header .subtitle { color: rgba(255,255,255,0.7); }
.report-header .subtitle span::before { background: rgba(255,255,255,0.5); }
```
**Why:** The DESIGN.md spec says "Black background, LOD branding." The header is the first thing a hotel GM sees. It should look like a premium consulting deliverable.

#### P1-7: Report footer missing black background, confidentiality notice, and hotel name
**Source:** arrange, colorize, polish
**Current:** `text-align: center; font-size: 11px; color: gray; border-top only.` Content: "Generated by Lights On Digital | Pace Pipeline v5"
**Required:**
```css
.report-footer {
  background: var(--lod-black);
  color: rgba(255,255,255,0.7);
  padding: 20px 32px;
  margin: 0 -40px -32px -40px;
  border-top: none;
  display: flex;
  justify-content: space-between;
  font-size: 11px;
}
```
Content should be dynamically populated:
- Left: "Confidential — Prepared for The Kahala Hotel & Resort"
- Right: "Generated 2026-03-11 | Lights On Digital"
- Remove "v5" (internal nomenclature)

#### P1-8: Delta formatting lacks arrows — accessibility gap
**Source:** polish
**Current:** Deltas use `+`/`-` signs only. No directional arrows. Uses ASCII hyphen-minus (U+002D) for negatives instead of minus sign (U+2212).
**Required:** Add `\u25B2` (filled triangle up) / `\u25BC` (filled triangle down) to all delta formatters. Use proper minus sign U+2212 for negatives.
**Why:** The .impeccable.md states "Color is never the sole indicator — deltas use arrows/symbols alongside color." Currently green/red color is the primary differentiator. Arrows add a second non-color channel.

#### P1-9: Month row expansion has no visual affordance
**Source:** polish, critique
**Current:** Only cues are `cursor: pointer` on first column and blue hover color. No chevron or indicator.
**Required:** Prepend `<span class="expand-icon">&#x25B8;</span>` to month label, rotate 90deg on expand.
**Why:** Without an affordance, hotel GMs will never discover the daily drill-down. The daily breakdown is one of the dashboard's best features.

#### P1-10: Print CSS missing page-break controls
**Source:** arrange, polish
**Current:** Only `break-inside: avoid` on `.kpi-card`. No chart protection, no table row orphan control, no page size declaration.
**Required:**
```css
@media print {
  * { -webkit-print-color-adjust: exact; print-color-adjust: exact; }
  @page { size: letter; margin: 0.45in 0.35in; } /* per DESIGN.md spec */
  .chart-section { break-inside: avoid; }
  .data-table tr { break-inside: avoid; }
  .footnotes, .report-footer { break-before: avoid; }
}
```
**Why:** "Print is a first-class citizen." Without these, the chart can split across pages and table rows can orphan. Margins match DESIGN.md spec (0.35in sides / 0.45in top-bottom).

#### P1-11: Interactive elements not keyboard-accessible
**Source:** critique (persona: Sam)
**Current:** Pills are `<span>` with `onclick` — no `tabindex`, no `role="button"`, no `aria-label`. Month row toggle via `onclick` on `<tr>` — not focusable.
**Required:** Add `tabindex="0" role="button"` to all pills. Add `onkeydown` handlers for Enter/Space. Add `:focus-visible` styles.
**Why:** WCAG AA requires all interactive elements to be operable via keyboard.

---

### P2 — Should Fix (9 findings)

#### P2-1: Section headers missing LOD label treatment
**Source:** typeset
**Current:** `.chart-section h2, .table-section h2 { font-size: 16px; font-weight: 600; }`
**Required:** `font-size: 12px; font-weight: 600; text-transform: uppercase; letter-spacing: 0.1em; color: var(--lod-gray-dark);`
**Why:** LOD section labels are small, caps, letter-spaced. Current 16px sentence-case reads generic.

#### P2-2: Section spacing is uniform — no visual rhythm
**Source:** arrange
**Current:** Everything uses `margin-bottom: 28px`.
**Required:** Variable spacing by hierarchy:

| Transition | Current | Recommended |
|------------|---------|-------------|
| Header -> controls | 28px | 32px |
| Controls -> KPIs | 24px | 24px |
| KPIs -> chart | 28px | 36px |
| Chart -> table | 28px | 36px |
| Table -> footnotes | 28px | 16px |
| Footnotes -> footer | 20px | 32px |

#### P2-3: KPI banner cards need more generous spacing
**Source:** arrange
**Current:** `gap: 16px; padding: 18px 20px;`
**Required:** `gap: 20px; padding: 24px 20px;`
**Why:** Cards visually merge. More padding supports the "monumental" numbers.

#### P2-4: Table column groups lack visual separation
**Source:** arrange, polish
**Current:** Groups differentiated only by th-group headers. No separators in data rows.
**Required:** Left border on first column of each group:
```css
.data-table th:nth-child(7), .data-table td:nth-child(7) { border-left: 2px solid var(--lod-gray-mid); }
.data-table th:nth-child(11), .data-table td:nth-child(11) { border-left: 2px solid var(--lod-gray-mid); }
```

#### P2-5: Chart SVG uses different font-family than page
**Source:** typeset, polish
**Current:** SVG: `font-family:Arial,Helvetica,sans-serif` (hardcoded in HTML and JS renderHeroChart)
**Required:** Match page font stack in both locations.

#### P2-6: CSS variable naming doesn't match spec tokens
**Source:** colorize
**Action:** Rename `--lod-blue` -> `--lod-electric-blue`. Add `--ghost: #C0C0C0`. Reconcile `--lod-gray-light` (#F5F5F5) vs spec's `--lod-light-gray` (#E5E5E5).

#### P2-7: Day expansion row has CSS/JS conflict
**Source:** polish
**Current:** CSS says `padding-left: 28px`, JS inline-styles `padding-left: 24px`. JS wins. Dead CSS.
**Required:** Remove inline styles from JS. Use CSS-only with consistent 28px indent. Add subtle left border (`3px solid var(--lod-electric-blue)`) for visual nesting.

#### P2-8: Pill touch targets too small
**Source:** polish
**Current:** Compare pills: ~25px height. Chart filter pills: ~20px height. Minimum: 44px.
**Required:** `min-height: 36px; display: inline-flex; align-items: center;` on pills. Chart filter pills: `padding: 8px 14px;`.

#### P2-9: Hover/focus states incomplete
**Source:** polish
**Current:** No `:active` state on pills. No `:focus-visible` styles. Table row hover has no transition.
**Required:** Add `.pill:active { transform: scale(0.97); }`, focus-visible outlines, `tr { transition: background 0.1s ease; }`.

---

### P3 — Polish Pass (7 findings)

| # | Finding | Fix |
|---|---------|-----|
| P3-1 | KPI label `letter-spacing: 0.5px` should be `0.1em` | Align with LOD spec |
| P3-2 | Table header `letter-spacing: 0.3px` should be `0.1em` | Consistency |
| P3-3 | 13px/12px pill font-size inconsistency | Standardize to 12px |
| P3-4 | Chart hardcoded hex in JS constants | Reference CSS variables |
| P3-5 | Empty `#window-label` span shows orphaned dot separator | `.subtitle span:empty::before { display: none; }` |
| P3-6 | Disabled pill tooltip uses native `title` (no mobile) | CSS `::after` tooltip or rely on footnote |
| P3-7 | Chart legend misaligned with plot area | Add `margin-left: 80px` to match PAD_L |

---

## Active Pill Color Decision

The .impeccable.md says Electric Blue (`#0A84FF`) for "active pills." The current implementation uses black. White text on `#0A84FF` = **3.65:1** — fails AA for normal text. Options:

1. **Keep black active pills** (current) — 21:1 contrast, brand-safe, no risk
2. **Use blue active pills** — requires text >=18.67px bold for AA, or darken blue

**Recommendation:** Keep black. Flag for future design review if LOD wants blue pills specifically.

---

## Recommended Implementation Order

These map to the impeccable skills that would execute the fixes:

1. **`/typeset`** — P1-1, P1-2, P1-3, P2-1, P2-5, P3-1, P3-2, P3-3
   *Font stack, weight hierarchy, sizing. The foundation for everything else.*

2. **`/colorize`** — P1-4, P1-5, P2-6
   *Palette swap. Must happen before header/footer since those use `--lod-black`.*

3. **`/arrange`** — P1-6, P1-7, P2-2, P2-3, P2-4
   *Header/footer structure, section spacing, KPI card layout, table groups.*

4. **`/polish`** — P1-8, P1-9, P1-10, P1-11, P2-7, P2-8, P2-9, P3-4 through P3-7
   *Micro-details: affordances, delta arrows, print CSS, keyboard a11y, touch targets.*

5. **`/critique`** — Re-score after fixes. Target: 32+/40.

---

## Persona Summary

| Persona | Key Risk |
|---------|----------|
| **Alex (LOD strategist)** | No keyboard navigation for any interactive element |
| **Sam (accessibility)** | Zero keyboard accessibility on pills/rows; delta colors fail WCAG |
| **Kenji (Hotel GM, PDF)** | Header looks generic; KPI numbers don't pop; "WoW"/"STLY" jargon; footer is throwaway |

---

## Files to Modify

All changes are in **one file**: `dashboard-v5.html`
- CSS `:root` variables (lines 8-19)
- CSS component styles (lines 20-386)
- HTML header/footer structure (lines 389-542)
- JS render functions (lines 28796-29249) — delta formatting, chart font, footer content
- JS chart constants (around line 28653) — COL_CURRENT, COL_GHOST, etc.

No external dependencies added — single-file constraint preserved per DESIGN.md.

The Python template (`pace-pipeline/src/dashboard/template.html`) will need the same CSS/HTML changes propagated after the preview is approved.
