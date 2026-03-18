# QR Cost Landscape Section — Implementation Plan

Add a new "QR Cost Landscape" section to the existing `index.html` single-page app. The section lets students explore how total annual cost varies across (Q, R) space via a **2D heatmap (default)** and an **optional 3D surface**, with markers for the current and optimal policies and a compact comparison panel.

## User Review Required

> [!IMPORTANT]
> **No new files are created.** All changes go into `index.html`. The prototype file `qr_cost_landscape_prototype_v2.html` is read-only reference — its design tokens and layout inform the implementation, but no code is copied verbatim.

> [!WARNING]
> Plotly.js (~3.5 MB minified) will be loaded via CDN `<script>` tag. This adds network weight on first load but avoids a build step. If bundle size is a concern, we can use the `plotly-basic` partial bundle (~1 MB) instead.

## 1. Exact File Map

| File | Role | Action |
|------|------|--------|
| [index.html](file:///c:/Users/maxba/Documents/GitHub/normal-loss-visualizer/index.html) | Main app | MODIFY — CSS, HTML section, JS |
| [qr_cost_landscape_prototype_v2.html](file:///c:/Users/maxba/Documents/GitHub/normal-loss-visualizer/qr_cost_landscape_prototype_v2.html) | Design reference | READ ONLY |

No new files are recommended. Everything stays in one HTML file to match the existing architecture.

## 2. Insertion Points in `index.html`

### CSS (lines ~10–533, inside `<style>`)
Insert new rules at **end of the `<style>` block** (before `</style>`, line 533). Scoped under a `.qr-landscape` namespace to avoid collisions with existing styles.

### HTML (line ~735, after Cost Breakdown `</section>`, before Scenario Comparison `<section>`)
Insert the entire QR Cost Landscape section here:

```html
<!-- ── QR COST LANDSCAPE ────────────────── -->
<section class="panel qr-landscape" id="qrLandscapeSection">
  ...
</section>
```

### Plotly CDN (line ~758, just before `<script>`)
```html
<script src="https://cdn.plot.ly/plotly-2.35.2.min.js" charset="utf-8"></script>
```

### JavaScript (inside existing `<script>`, after `renderApp()` definition, ~line 896)
Add new functions. Wire into `renderApp()` by adding a `renderQrLandscape(active)` call.

## 3. Proposed Changes

### CSS Additions

#### [MODIFY] [index.html](file:///c:/Users/maxba/Documents/GitHub/normal-loss-visualizer/index.html)

New CSS (appended inside `<style>`):
- `.qr-landscape` — section container, inherits `.panel`
- `.qr-landscape-header` — eyebrow tag + title + subtitle, matching existing `section-heading` weight
- `.qr-landscape-legend` — pill row for Current / Optimal markers
- `.qr-view-toggle` — 2D / 3D toggle (radio-backed, CSS-only active state)
- `.qr-plot-wrap` — container for the Plotly div, card border + radius
- `.qr-comparison-grid` — two-column grid for current vs optimal mini-cards
- `.qr-mini-card`, `.qr-mini-card.optimal` — mini stat cards
- `.qr-metric-list`, `.qr-metric`, `.qr-metric-delta` — metric rows
- `.qr-tip-box` — "How to read it" educational callout
- Responsive overrides at `@media (max-width: 980px)` and `640px`

Design language rules:
- Reuse existing CSS variables: `--surface`, `--border`, `--accent`, `--radius-md`, `--font`, `--font-heading`, `--muted`, `--ink`
- Match `--green` / `--green-soft` for optimal highlights
- Font sizes, weights, and letter-spacing match existing `.section-heading`, `.stat-card`, `.teaching-callout` patterns

---

### HTML Section

Insert after line 735 (Cost Breakdown `</section>`):

```
Section header (eyebrow, h2, subtitle)
├── Legend pills: Current policy / Optimal policy / "Default view: 2D heatmap"
├── 2D / 3D radio toggle
├── Layout grid (2 columns on desktop, 1 on mobile)
│   ├── Chart card
│   │   ├── Chart title + context-aware description (swaps with view mode)
│   │   ├── <div id="qrPlot2d"> — Plotly 2D heatmap
│   │   └── <div id="qrPlot3d" style="display:none"> — Plotly 3D surface
│   └── Side stack
│       ├── Comparison card (Current vs Optimal)
│       │   ├── Mini-card: Current policy — cost, Q, R
│       │   ├── Mini-card: Optimal policy — cost, Q*, R*
│       │   └── Metric list: total cost gap, Type 1 service, Type 2 service
│       └── Tip card ("How to read it")
└── Footer tagline
```

---

### JavaScript

New functions (added inside existing `<script>`):

| Function | Responsibility |
|----------|---------------|
| `computeCostSurface(state)` | Build a grid of ~30×30 (Q, R) points, compute total annual cost at each via the existing `computeScenario()` logic. Also compute Type 1 (cycle service level) and Type 2 (fill rate) at each point. Return `{qVals, rVals, costMatrix, optQ, optR, optCost, optType1, optType2, curCost, curType1, curType2}`. |
| `renderQrHeatmap(surface)` | Call `Plotly.react('qrPlot2d', ...)` with a heatmap trace + scatter markers for current/optimal. Color scale: cool blues → warm reds (matching prototype). |
| `renderQrSurface3d(surface)` | Call `Plotly.react('qrPlot3d', ...)` with a surface trace + scatter3d markers. |
| `renderQrComparison(surface)` | Populate the comparison panel DOM: costs, service levels, deltas. |
| `renderQrLandscape(m)` | Orchestrator: calls `computeCostSurface`, then the three renderers above. |

Key details:
- **Optimal (Q\*, R\*) search**: Iterate the grid, pick the cell with minimum total cost. Display as the optimal marker.
- **Type 1 service**: `1 - P(stockout)` = `Φ(z)` at each (Q, R).
- **Type 2 service (fill rate)**: `1 - n_R / Q` at each (Q, R).
- **Grid range**: Q from `0.3 * currentQ` to `3 * currentQ`; R from `0.3 * currentR` to `3 * currentR`, clamped to positive values.
- **Toggle**: JS listens for radio change, shows/hides the two Plotly divs and calls the appropriate render function (lazy: 3D only renders on first toggle to 3D).

Lifecycle wiring — add to the existing `renderApp()`:
```js
renderQrLandscape(active);
```

## 4. Milestone Plan

### Milestone 1 — CSS + HTML Shell
- Add all CSS rules (scoped under `.qr-landscape`)
- Add the full HTML section with placeholder divs for Plotly
- Add Plotly CDN script tag
- **Acceptance**: Section appears with correct layout, toggle works (shows/hides divs), comparison panel is styled but shows placeholder text. No JS logic yet.

### Milestone 2 — Cost Surface Computation
- Implement `computeCostSurface(state)`
- Verify grid generation and optimal search with console logs
- **Acceptance**: Console output shows a 30×30 cost matrix, identified optimal Q\* and R\*, correct Type 1/Type 2 at current and optimal.

### Milestone 3 — 2D Heatmap (Default View)
- Implement `renderQrHeatmap(surface)`
- Wire into `renderApp()`
- **Acceptance**: 2D heatmap renders on page load. Current marker (dark dot) and optimal marker (green dot) are visible. Color scale matches prototype palette. Axes labeled "Reorder point R" and "Order quantity Q".

### Milestone 4 — 3D Surface (Optional)
- Implement `renderQrSurface3d(surface)`
- Wire toggle to show 3D on radio change
- **Acceptance**: Toggling to 3D shows a rotatable surface plot with markers. Toggling back to 2D restores the heatmap.

### Milestone 5 — Comparison Panel + Polish
- Implement `renderQrComparison(surface)`
- Populate mini-cards and metric rows with live data
- Add delta badges (e.g., "25.4% lower", "+22.6 pts")
- **Acceptance**: Comparison panel shows correct live data matching the current inputs. Changing inputs updates both the chart and the panel. Responsive layout works on mobile.

## 5. Risks / Edge Cases

| Risk | Mitigation |
|------|-----------|
| **Plotly CDN fails** | Wrap render calls in `if (typeof Plotly !== 'undefined')`. Show a graceful fallback message. |
| **σ_X = 0 (known-demand scenario)** | When σ = 0, Type 1 and Type 2 are trivial (100% or 0%). Cost surface becomes flat in the R dimension. Still render but note it in the tip box. |
| **Very large Q or R values** | Clamp grid range so Plotly doesn't choke on huge arrays. Cap at 50×50 grid. |
| **Optimal = current** | If the user's inputs happen to be optimal, show "Current policy is already optimal" instead of a delta. |
| **Negative safety stock** | `computeScenario` already handles this. Cost surface will show high costs for low R, which is correct. |
| **Mobile performance** | 3D surface can be heavy on mobile. Keep grid at 30×30 max. Lazy-load 3D only when toggled. |
| **CSS namespace collisions** | All new classes prefixed with `qr-` to avoid clashing with existing `.layout`, `.toggle`, `.pill`, etc. |

## 6. Recommended New Files

**None.** All code stays in `index.html` to match the existing single-file architecture. The prototype file is reference-only.

## 7. Preserving Existing Design Language

| Element | Existing pattern | How we match |
|---------|-----------------|-------------|
| Section wrapper | `.panel` with `border`, `border-radius`, `box-shadow` | QR section uses `.panel.qr-landscape` |
| Heading | `.section-heading` (Source Serif 4, 1.04rem, 700) | Same class reused |
| Divider | `.section-divider` | Same class reused |
| Stat cards | `.stat-card` / `.cost-card` | QR mini-cards use similar padding, border, radius |
| Color tokens | `--accent`, `--green`, `--muted`, `--border`, `--ink` | All reused; optimal highlights use `--green` / `--green-soft` |
| Fonts | Inter for body, Source Serif 4 for headings, Courier New for numbers | Same |
| Responsive breakpoints | 980px, 640px | Same breakpoints |
| Concept pills | `.concept-pill` pattern | QR legend pills follow same structure |

## Verification Plan

### Browser Testing
1. Open `index.html` in a local browser (right-click → Open with Live Server, or `python -m http.server` in the repo directory)
2. Scroll to the QR Cost Landscape section
3. **Check 2D heatmap** is the default view, showing a colored grid with two markers
4. **Click the 3D toggle** — verify the surface loads and markers appear
5. **Click back to 2D** — verify it restores
6. **Check the comparison panel** shows cost, Type 1, and Type 2 for current vs optimal
7. **Change inputs** (e.g., reorder point slider) — verify the chart and panel update live
8. **Resize the window** to mobile width — verify responsive layout stacks correctly
9. **Load benchmark** button — verify the section still renders correctly with benchmark inputs

### Manual User Verification
- After implementation, the user should visually inspect the section in their browser and confirm it matches the prototype's design intent.
