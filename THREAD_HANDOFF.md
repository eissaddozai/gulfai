# Gulf AI Assessment — Thread Handoff Guide
**Session:** `claude/session_01YUKpCoBcVqBgaxy98QeUKj`
**File:** `/home/user/gulf_ai_assessment_enhanced.html`
**Repo:** `https://github.com/eissaddozai/gulfai.git`
**Date written:** 2026-03-04

---

## 1. What This Document Is

A single self-contained HTML intelligence document (~801 KB, ~9,025 lines) serving as a world-class analytical artifact on Gulf state AI investment and technology transfer risks. All content, styles, and JavaScript live in one file. No build step, no bundler, no server required — open in browser.

The file has been through extensive enhancement work across multiple sessions. This guide exists so the next thread does not repeat the same investigations, does not re-introduce fixed bugs, and understands the landmines buried in the codebase.

---

## 2. File Anatomy

```
gulf_ai_assessment_enhanced.html
│
├── <head>
│   ├── Google Fonts (Source Sans 3, Playfair Display)
│   ├── D3.js v7.8.5 — INLINED (279 KB) ← CDN was unreachable; do NOT switch back to CDN
│   ├── <style block 0>  — original base styles
│   ├── <style block 1>  — enhancements (id="enhancements") — ALL new CSS goes here
│   └── <style block 2>  — editor styles
│
├── <body>
│   ├── .site-header       — classification bar + document title
│   ├── <nav>              — 10 tab buttons (.nb), one per panel
│   ├── #panel-progress    — reading progress bar (not a content panel)
│   ├── #panel-uae         — UAE Governance & AI Entities
│   ├── #panel-saudi       — Saudi Arabia AI Landscape
│   ├── #panel-qatar       — Qatar AI & Sovereign Investments
│   ├── #panel-maturity    — Capital & Technology Maturity Matrix
│   ├── #panel-dep         — Technology Dependency & Chip Analysis
│   ├── #panel-gaps        — Intelligence Gaps
│   ├── #panel-comp        — Competitive Landscape
│   ├── #panel-allied      — Five Eyes Allied Context
│   ├── #panel-glossary    — Glossary & Reference
│   ├── #panel-edit        — Editor v16 controls
│   │
│   ├── <script block 0>   — original core JS (show(), filterGaps(), tooltips, D3 drawXxx())
│   ├── <script block 1>   — Editor v16
│   ├── <script block 2>   — enhancements JS (id="enhancements-js") — ALL new JS goes here
│   ├── <script block 3>   — ambient/atmospheric effects
│   └── <script block 4>   — additional feature scripts
│
└── </body>
```

### Panels and their accent colors

| Panel ID | Tab Label | `--panel-accent` | Character |
|---|---|---|---|
| `panel-uae` | UAE | `#0A7E8C` | Teal |
| `panel-saudi` | Saudi Arabia | `#C8A84A` | Amber |
| `panel-qatar` | Qatar | `#8B3050` | Burgundy |
| `panel-maturity` | Capital | `#4682B4` | Steel blue |
| `panel-dep` | Tech Chain | `#9A5A30` | Rust |
| `panel-gaps` | Gaps | `#C0392B` | Red |
| `panel-comp` | Competitive | `#5A7A6A` | Muted green |
| `panel-allied` | Five Eyes | `#4A7A9A` | Slate blue |
| `panel-glossary` | Reference | `#6B7A8A` | Grey |
| `panel-edit` | Edit | `#6B7A8A` | Grey |

> **Note:** Panel accent tokens are defined TWICE in the CSS — once in the original block and again in the enhancements block. The enhancements block appears later and wins via cascade. Do not remove the second set.

---

## 3. The Architecture of `show()` — Critical to Understand

The `show(id, btn)` function has been **wrapped 9 times** by successive enhancement scripts. Each wrapper calls `_perfOrig(id, btn)` (or similar `_xxxOrig`) to chain down to the real function. The final wrapper at the bottom is the `contentVisibility` performance wrapper:

```js
// Wrapper N (contentVisibility performance — INNERMOST WRAPPER, runs last in chain)
(function(){
  // On page load: hide all inactive panels
  document.querySelectorAll('.panel:not(.active)').forEach(function(p){
    p.style.contentVisibility = 'hidden';
  });

  var _perfOrig = window.show;
  window.show = function(id, btn){
    // 1. Clear contentVisibility on TARGET panel FIRST
    var target = document.getElementById('panel-' + id);
    if(target) target.style.contentVisibility = '';
    // 2. Call the real show()
    _perfOrig(id, btn);
    // 3. After 400ms, hide all now-inactive panels
    setTimeout(function(){
      document.querySelectorAll('.panel:not(.active)').forEach(function(p){
        p.style.contentVisibility = 'hidden';
      });
    }, 400);
  };
})();
```

### Why `contentVisibility: hidden` is a landmine

`contentVisibility: hidden` tells the browser to **completely skip rendering, layout, and painting** of the element's subtree. This has cascading side effects:

- **IntersectionObserver does not fire** for elements inside a `contentVisibility: hidden` container. Ever. Even if the elements would geometrically be in the viewport.
- **CSS animations and transitions do not run** inside hidden subtrees.
- **JS `getBoundingClientRect()`** returns `{0,0,0,0}` for elements inside hidden containers.
- **`offsetWidth` / `offsetHeight`** return 0 for elements inside hidden containers.

This is responsible for **every single table/diagram visibility bug encountered in this session.** When you add any scroll-reveal, entrance animation, or IntersectionObserver-gated visibility to elements inside panels, those elements will be permanently invisible in all non-initial panels.

---

## 4. The D3 Diagrams — What Was Fixed and Why They Work Now

### Five diagrams, five draw functions

| SVG ID | Draw function | Height | Panel |
|---|---|---|---|
| `uae-svg` | `drawUAE()` | 720px | `panel-uae` |
| `saudi-svg` | `drawSaudi()` | 650px | `panel-saudi` |
| `qatar-svg` | `drawQatar()` | 620px | `panel-qatar` |
| `maturity-svg` | `drawMaturity()` | 690px | `panel-maturity` |
| `dep-svg` | `drawDep()` | 980px | `panel-dep` |

### The `_rendered` memoization flag

Every draw function is guarded by:
```js
var _rendered = { uae:false, saudi:false, qatar:false, maturity:false, dep:false };
```

When a draw function is called and succeeds, it sets `_rendered.xxx = true`. Subsequent calls are silently skipped. This prevents redundant re-renders on repeated panel visits.

**The W=0 bug (fixed at commit `119e1b7`):** If a draw function is called before the SVG container has settled in layout (e.g., the panel hasn't fully rendered yet), `parentElement.offsetWidth` returns 0. The old code would draw with `viewBox="0 0 0 720"` — everything invisible — then set `_rendered.xxx = true` and never draw again.

**The fix:** Every draw function now has this guard at the top:
```js
const W = document.getElementById('uae-svg').parentElement.offsetWidth, H = 720;
if(W < 10) {
  if(typeof _rendered !== 'undefined') _rendered.uae = false;
  setTimeout(drawUAE, 150);
  return;
}
svg.attr('viewBox', `0 0 ${W} ${H}`);
```

**The double opacity bug (fixed at commit `119e1b7`):** `applyD3Enhancements()` was setting `g.style.opacity = '0'` on ALL node groups after D3's own opacity transitions had completed, then using a fragile `i * 35ms` setTimeout chain to restore them. On panel switch, the setTimeout chain was cancelled, leaving all nodes at `opacity: 0` permanently. The entrance animation block was removed from `applyD3Enhancements`. D3 draw functions handle their own opacity transitions internally.

### D3 must remain inlined

D3 v7.8.5 is inlined directly in `<head>`. The CDN (`cdnjs.cloudflare.com`) is unreachable in the deployment environment. Do not remove the inline D3 or replace it with a CDN link.

---

## 5. The Tables — What Was Broken and How It Was Finally Fixed

This took the most time to resolve. There were **three separate, compounding mechanisms** all conspiring to hide tables:

### Layer 1: Clip-path wipe animation (M8 enhancement)
```css
table.dt.sr { clip-path: inset(0 100% 0 0); transition: none; }
table.dt.sr.vis { clip-path: inset(0 0% 0 0); transition: clip-path .7s ...; }
```
Tables started fully clipped to zero width. IntersectionObserver was supposed to add `.vis` and reveal them. Worked for the initial active panel. Failed for all others because of `contentVisibility: hidden`.

### Layer 2: `.sr` scroll-reveal system applied to tables
The enhancements JS added `.sr` to all `.dt` tables:
```js
document.querySelectorAll('.dt, .sc, .kf-callout, ...').forEach(el => {
  el.classList.add('sr');
  srTargets.add(el);
});
```
`.sr` applies `opacity: 0; transform: translateY(18px)`. Combined with Layer 1, tables were doubly hidden.

### Layer 3: TABLE ROW ENTRANCE ANIMATION (the final, actual culprit)
```js
// ═══ v13: TABLE ROW ENTRANCE ANIMATION ═══
document.querySelectorAll('.dt tbody tr').forEach((row, i) => {
  row.style.opacity = '0';                           // INLINE STYLE on every <tr>
  row.style.transform = 'translateX(-8px)';
  row.style.transition = `opacity .3s ease ${i * 0.03}s, ...`;
});
```
This set `style.opacity = '0'` as an **inline style** on every single table row at page load. The MutationObserver meant to restore them was unreliable (missed panels affected by `contentVisibility: hidden`).

**Critical insight:** Our CSS fix `table.dt.sr { opacity: 1 !important }` targeted the `<table>` element, but the opacity was on the `<tr>` children — a completely different DOM node. Even with `!important` on the parent table, child rows had their own inline `opacity: 0`.

### The Final Fix (current state)
Two changes at commit `6ce00cb`:

**CSS** — override both the table and every row:
```css
table.dt, table.dt.sr, table.dt.sr.vis {
  opacity: 1 !important;
  transform: none !important;
  clip-path: none !important;
  transition: none !important;
}
table.dt tbody tr {
  opacity: 1 !important;  /* overrides inline style.opacity='0' set by JS */
  transform: none !important;
}
```

**JS** — neutralize the entrance animation:
```js
document.querySelectorAll('.dt tbody tr').forEach((row, i) => {
  // Row entrance animation disabled — was hiding rows with opacity:0
  row.style.opacity = '1';
  row.style.transform = 'translateX(0)';
  row.style.transition = '';
});
```

> **Rule for future work:** `!important` in a CSS stylesheet DOES override inline `style` attributes (unless the inline style also carries `!important`). When CSS `!important` isn't working, suspect that the property is being set on a CHILD element, not the element your selector targets.

---

## 6. Definitive Rules for This Codebase

These are laws. Violate them and you will spend hours debugging.

### RULE 1: Never use IntersectionObserver to gate visibility of panel content
`contentVisibility: hidden` on inactive panels silently kills IntersectionObserver for all elements inside those panels. If an element is hidden until the observer fires, and it's in an inactive panel, it will be hidden forever. Use CSS that works unconditionally instead.

### RULE 2: Never use JS entrance animations that set `style.opacity = '0'` without a guaranteed restore
If JS sets an inline opacity of 0 on anything inside a panel, that thing will be invisible in every panel except the one that was active at page load, unless the restore mechanism is 100% synchronous and not gated on any event that `contentVisibility: hidden` might suppress.

### RULE 3: Never add `.sr` (scroll-reveal) to table rows, table cells, or anything inside `.dt`
The `.sr` system was designed for top-level block elements. It is not reliable for table internals. Use simple unconditional CSS for table visibility.

### RULE 4: Never call a D3 draw function without the W=0 guard
All five draw functions now have `if(W < 10){ _rendered.xxx = false; setTimeout(drawXxx, 150); return; }`. If you add a sixth diagram, include this guard. If you modify an existing draw function, preserve the guard.

### RULE 5: The `_rendered` flag must be reset before retrying
If you want to force a redraw (e.g., on resize), set `_rendered.xxx = false` before calling `drawXxx()`.

### RULE 6: Do not add a new wrapper around `show()` without understanding the chain
Nine wrappers already exist. Each calls the prior one as `_xxxOrig`. Adding a tenth is fine if necessary, but trace the chain before you do. The `contentVisibility` wrapper must remain the outermost (last defined).

### RULE 7: All new CSS goes in `<style id="enhancements">`, all new JS goes in `<script id="enhancements-js">`
The original `<style>` block 0 and the original script block 0 must remain untouched, except for surgical bug fixes. This is a hard constraint from the project owner.

### RULE 8: Editor v16 must remain 100% intact
Do not restructure, consolidate, or refactor the editor script block. Bug fixes only, applied as minimal inline edits to existing functions.

---

## 7. CSS Architecture Reference

### Design tokens (`:root`)

```css
/* Surfaces */
--surface-0: #0B0E14   /* deepest background */
--surface-1: #131720   /* panel background */
--surface-2: #1A1F2B   /* card / raised elements */

/* Country identity */
--uae:   #0A7E8C
--saudi: #8B7355
--qatar: #5F6B7A

/* Severity */
--c-critical: #C0392B
--c-high:     #E67E22
--c-low:      #27AE60

/* Text */
--text-sec: #9AA0B0

/* Borders */
--border:    rgba(255,255,255,0.06)
--border-hi: rgba(255,255,255,0.12)

/* Typography */
--font-s: 'Source Sans 3','Helvetica Neue',sans-serif  /* body */
--font-d: 'Playfair Display',Georgia,serif              /* display/headings */

/* Radii */
--radius-md: 6px
--r-lg:      12px

/* Shadows */
--shadow-md: 0 4px 16px rgba(0,0,0,0.4)

/* Motion */
--transition-fast: .15s cubic-bezier(.4,0,.2,1)
--transition-med:  .25s cubic-bezier(.4,0,.2,1)

/* Per-panel accent (overridden per panel) */
--panel-accent: var(--uae)
```

> There are 115 unique CSS custom properties in total. The above are the most important for design consistency.

### Key selectors

| Selector | Purpose |
|---|---|
| `.panel` | Each content section; `display:none` unless `.active` |
| `.panel.active` | The visible panel |
| `.nb` | Nav tab button |
| `.nb.active` | Active tab |
| `.pn` | Panel name (small label above title) |
| `.pt` | Panel title (large heading) |
| `.ph` | Panel header block |
| `.pf` | Panel prose/flow text block |
| `.dt` | Data table |
| `.dt-wrap` | Scrollable wrapper around wide tables |
| `.sc` | SVG diagram container |
| `.sc-label` | Diagram label |
| `.gc` | Gap card |
| `.gg` | Gap card grid |
| `.stat-card` | Statistic card |
| `.kf-callout` | Key Finding callout block |
| `.kf-pill` | Key finding pill/tag |
| `.tt` | Tooltip container |
| `.sr` | Scroll-reveal element (starts `opacity:0`) |
| `.sr.vis` | Revealed scroll-reveal element |
| `.gfbtn` | Gap filter button |

---

## 8. The Tooltip System

Three functions manage the global tooltip:

```js
showTT(html, event)  // populate and show
mvTT(event)          // follow cursor
hideTT()             // hide
```

Data is stored in `ttData[]` — an array of objects parsed from the HTML at page load by `parseTooltips()`. Each object holds the tooltip HTML and the target entity name. The editor can modify tooltip content in-memory; BUG-11 (partially fixed) means calling `on()` again overwrites in-memory edits with re-parsed HTML. Guard: `if(!ttData.length) parseTooltips();`.

---

## 9. Editor v16 — Bug Status

The plan identified 12 bugs. Here is their current fix status:

| Bug | Severity | Description | Fixed? |
|---|---|---|---|
| BUG-01 | CRITICAL | Stray `$('eCat').classList.remove('vis')` executes at parse time, not on close | ✅ Yes (fixed in prior session) |
| BUG-02 | CRITICAL | Global click handler closes catalogue on every click, including clicks inside catalogue | ✅ Yes (fixed in prior session) |
| BUG-03 | HIGH | `exportHTML` leaves editor in corrupted state after export | ❌ Not yet |
| BUG-04 | HIGH | CSS variable undo removes property entirely instead of restoring previous value | ❌ Not yet |
| BUG-05 | MEDIUM | `mSvgFront`/`mSvgBack` not tracked in undo stack | ❌ Not yet |
| BUG-06 | MEDIUM | IntersectionObserver double-observes elements (deduped in enhancements JS, but original still present) | ✅ Mitigated |
| BUG-07 | MEDIUM | `posCtx` can place context bar above viewport with no floor clamp | ❌ Not yet |
| BUG-08 | LOW | Resize undo doesn't restore `maxWidth` when it was empty string | ❌ Not yet |
| BUG-09 | LOW | `off()` doesn't remove `position:relative` injection from `on()` | ❌ Not yet |
| BUG-10 | LOW | `show()` pushes URL hash but `popstate` is never handled | ❌ Not yet |
| BUG-11 | LOW | `parseTooltips()` called every `on()`, overwrites in-memory tooltip edits | ❌ Not yet |
| BUG-12 | LOW | `rgbHex` returns `#000000` for transparent backgrounds in color picker | ❌ Not yet |

---

## 10. What the Plan Called For (Status)

Reference: `/root/.claude/plans/dapper-inventing-wombat.md`

### Phase 0 — Bug fixes
- BUG-01, BUG-02: ✅ Done
- BUG-03, BUG-04, BUG-10: ❌ Pending

### Phase 1 — CSS only
Foundational table, typography, and token refinements (B3, B4, C2, C3, C5, C7, C8, D6, D7, E4, I3, A7, A3, A4, R1, R2, R5, R6, P1, P3, P6). **Not started** — the session was consumed by stability work.

### Phase 2 — CSS token + ambient system
Panel accent tokens: ✅ Done (injected in enhancements block).
Atmospheric layering (O3, O5, O8, R10): ❌ Pending.

### Phase 3 — Table interactivity (the big one, not started)
Five signature interactions:

| Table | Interaction | Panel |
|---|---|---|
| Chip Approvals vs. Confirmed Possession | **Animated Waffle Scatter + Computed Gap Column** | `#panel-dep`, Table 1.3a |
| Integration Tier Assessment | **Live Heat-Map + FLIP-animated Sort + Click-Pin Comparison** | `#panel-comp` |
| Competitive Landscape Matrix | **Text Filter + Expandable Actor Profile** | `#panel-comp` |
| Sovereign AI Models | **Table ↔ Animated Radar Chart Toggle** | `#panel-dep`, Table 1.3c |
| Five Eyes Allied Context | **Threat-Level Gauge Overlay + Gap Highlight** | `#panel-allied` |

### Phase 4 — D3 diagram refinements (not started)
G2 (node dimming on hover), G6 (tooltip redesign), G1 (entrance animations — safe now that the double-opacity bug is fixed), G4 (node glow on priority nodes), N9 (minimap for UAE diagram), G8 (pan+zoom).

### Phase 5 — Motion & ambient polish (not started)
M1 (FLIP sort), M6 (kf-pill stagger), M9 (slot machine stat numbers), D1 (count-up animation), D2 (progress bar under stats), O2 (aurora sweep on panel activation), O6 (typing animation on subtitle), O9 (glitch badge on load).

### Phase 6 — Power features (not started)
Q1 (command palette Cmd+K), Q5 (? shortcut overlay), Q2 (side-peek drawer), Q3 (focus mode), K1 (tooltip pin), H1 (within-panel jump nav).

---

## 11. Git History (This Session)

```
6ce00cb  Fix tables hidden by JS inline opacity:0 on every tbody tr  ← current HEAD
9b9a3e0  Fix tables permanently hidden by clip-path + contentVisibility conflict
c439b45  Clean nav tabs + section accent colors — D3 diagrams and tables untouched
7862865  Fix tables hidden by clip-path scroll-reveal + contentVisibility conflict (incomplete)
1f9d744  Replace D3 diagrams with structured entity placeholders (WRONG — reverted)
119e1b7  Fix diagram rendering: W=0 guard + remove double entrance animation
8dbda9d  Visual revamp — all 8 phases implemented comprehensively
a416b65  Diagnose & fix: SVG letterboxing, nav tabs, spacing, opacity fallback
b3e8e94  Phase 4-6: panel gradient titles, D3 post-processing, power features
3a1e357  CRITICAL FIX: Inline D3 v7.8.5 — diagrams now CDN-independent
```

If anything goes catastrophically wrong, `119e1b7` is the last commit where both D3 diagrams AND tables were confirmed working (diagrams working, tables working in active panel but broken in inactive panels — the inactive-panel table bug was fixed at `6ce00cb`).

---

## 12. Lessons Learned — For Any Future Session

### On debugging invisible elements
Always check **all three layers**:
1. Is the element present in the DOM? (`document.querySelectorAll('.dt').length`)
2. Is a parent container hiding it? (`contentVisibility`, `display:none`, `overflow:hidden` with zero height)
3. Does the element have an **inline style** (`style.opacity`, `style.display`) overriding CSS?

When CSS `!important` doesn't seem to work, check whether your selector targets the right DOM node. `table.dt { opacity: 1 !important }` doesn't fix `<tr style="opacity:0">` inside it.

### On `contentVisibility: hidden`
It is powerful but brutal. It's a performance optimization that completely skips rendering of panel contents. It works perfectly for the use case (hiding inactive panels) but is **incompatible** with:
- IntersectionObserver-gated visibility
- CSS animations that haven't started yet
- JS measurements (`offsetWidth`, `getBoundingClientRect`)
- Any "lazy initialize on scroll" pattern

The solution for anything that needs to work in inactive panels: use CSS that applies unconditionally, not via observer or event.

### On entrance animations for data elements
Skip them for tables and critical content. Apply them only to decorative elements (stat cards, callout blocks, gap cards) where being briefly invisible doesn't constitute a functional failure. Tables that are invisible are data that doesn't exist from the user's perspective.

### On the `show()` wrapper chain
If something needs to happen on panel switch, add a wrapper at the TOP of the enhancements-js block so it chains correctly. Always store the previous `window.show` before overwriting it. The chain is:

```
user clicks tab
→ outermost wrapper (yours)
→ ... intermediate wrappers ...
→ contentVisibility wrapper (clears hidden on target)
→ _perfOrig (original show function — adds .active, fires animation)
→ setTimeout 400ms: hides inactive panels with contentVisibility:hidden
```

### On the D3 diagrams specifically
They are stable now. The key insight: D3 needs a non-zero parent width to draw correctly. The `show()` function triggers drawing when the panel becomes active, but the first call might happen before layout has settled (especially if the tab was never clicked before). The 150ms retry loop on W=0 handles this gracefully.

Do not touch the entrance animation in `applyD3Enhancements`. The function now only handles hover dimming (nodes fade non-adjacent nodes on hover) — leave it that way.

### On the Python scripting used to edit the file
**Never use f-strings with backslash-escaped quotes in expressions.** This was a recurring gotcha:
```python
# FAILS:
print(f"count: {content.count('class=\"dt\"')}")

# WORKS:
q = 'class="dt"'
print(f"count: {content.count(q)}")
```

Always verify the edit was applied (`if old in content:`) before saving. Always verify DOM invariants after saving (table count, SVG ID presence).

---

## 13. Current State of the File

The file at `HEAD` (`6ce00cb`) has:
- ✅ All 17 tables — visible in all panels
- ✅ All 5 D3 SVG diagrams — rendering correctly
- ✅ Nav tabs — clean (UAE / Saudi Arabia / Qatar / Capital / Tech Chain / Gaps / Competitive / Five Eyes / Reference / Edit)
- ✅ Per-panel accent colors — applied to headings and tab states
- ✅ Editor v16 — BUG-01 and BUG-02 fixed, rest pending
- ✅ D3 v7.8.5 — inlined, CDN-independent
- ❌ Phase 3–6 plan items — not started (table interactivity, D3 refinements, motion polish, power features)

The file is in a stable, functional baseline state. All the excavation work is done. The next session can focus entirely on building new features rather than fighting existing bugs.

---

## 14. Recommended Starting Point for Next Session

Read this document first. Then:

```bash
cd /home/user
git log --oneline -5           # verify you're on the right branch
git status                     # confirm clean working tree
# open gulf_ai_assessment_enhanced.html in browser
# navigate all 10 panels, verify tables and diagrams visible
# then proceed to Phase 3 table interactivity
```

The highest-value next item is **Phase 3A: Chip Approvals Waffle Scatter** in `#panel-dep`. It's the most dramatic data point in the document (58:1 chip accountability gap) and D3 is already loaded. Second priority is fixing the remaining critical editor bugs (BUG-03, BUG-04) since they affect anyone using the editor to author content.

Good luck. The hardest work is behind you.
