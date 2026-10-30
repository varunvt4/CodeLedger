# CodeLedger UI — Design System

Single source of truth for the dashboard in `codeledger/web/index.html`
(one self-contained file: `<style>` + `<script>`, no build step, no external requests).

## 1. File architecture

| Layer | Location | Order |
|---|---|---|
| CSS tokens | `:root` | 1st in `<style>` |
| Themes | `[data-theme="dark"]`, `[data-theme="light"]`, `prefers-color-scheme` pre-boot override | 2nd |
| Base / reset | element defaults, focus ring, scrollbar, sr-only | 3rd |
| Components | buttons, inputs, badges, chips, kbd, cards, stats, tables, segs, tooltip `[data-tip]`, dialog/palette/toasts, banner, skeleton, empty/error, ratio bars | 4th |
| Layouts | sidebar, topbar, crumbs, split/code viewer, side panel, chain, lessons, steps, charts, responsive `@960/@640` | 5th |
| JS | `state → utils → demo → api → router → toasts → dialogs → palette → helpers → charts → highlighter → views → theme → actions → keyboard → init` | single `<script>`, `"use strict"` |

JS conventions: ES5-style functions, `esc()`/`attr()` on **all** dynamic data
(prompts, code, filenames, commit messages are untrusted), inline SVG icons via
`ic(name, size)`, one delegated `click` + one `keydown` handler on `document`.

## 2. Tokens

### Spacing & shape
`--s1..--s12` (4/8/12/16/20/24/32/40/48) · radii `--r-sm 5 / --r-md 8 / --r-lg 12 / --r-xl 16 / --r-full`.

### Typography
System stacks only — `--font-ui` (ui-sans-serif, system-ui, Segoe UI…), `--font-mono` (ui-monospace, Menlo, Consolas…).
Sizes `--fs-xs 11 → --fs-3xl 34` (base 14px), line-heights `--lh-tight 1.25 / --lh 1.5`, weights `--fw-reg/med/semi/bold`.

### Motion
`--t-fast 120ms`, `--t 160ms`, `--ease: ease` — color/border/transform only;
all transitions disabled under `@media (prefers-reduced-motion: reduce)`.

### Elevation & layers
`--shadow-1` (cards), `--shadow-2` (overlays/sidebar) · z-index `--z-nav 40 < --z-overlay 60 < --z-toast 80`.

## 3. Themes

- `html[data-theme="dark"]` (default) and `html[data-theme="light"]`; a tiny inline
  `prefers-color-scheme` block prevents flash before the script boots, then JS adds
  `.theme-init` to hand control to the real theme rules.
- Preference `system | dark | light` stored in `localStorage["cl.theme"]`;
  button cycles system → dark → light (`t` or palette action).

### Semantic color roles (never hard-code hex in components)

| Role | Dark | Light | Meaning |
|---|---|---|---|
| `--accent` / `--accent-solid` | `#6c8cff` / `#4f6ae8` | `#4c63e6` / `#4459db` | one primary accent (AI, links, primary buttons) |
| `--human` | `#7d8596` | `#6a7280` | human-written lines |
| `--success` (+`-soft`) | `#3fb950` | `#17803d` | gate pass, tested, committed |
| `--warning` (+`-soft`) | `#d29922` | `#9a6700` | pending, untested, bypassed |
| `--danger` (+`-soft`) | `#f85149` | `#c1372f` | gate fail, errors |
| `--surface*`, `--border*`, `--ink*`, `--muted`, `--faint` | 3-step surfaces, 2-step borders, 4-step text | same | structure & hierarchy |
| `--code-bg`, `--tok-*` | code canvas + 7 syntax tokens | GitHub-ish tokens | highlighter |

Ratio bar segments: `.seg-ai` (accent) · `.seg-human` · `.seg-mixed` (warning) · `.seg-unknown` (surface-3) · `.seg-ok` / `.seg-danger`.

## 4. Components

- **Buttons** `.btn` (+ `.btn-sm`, `.btn-primary`, `.btn-ghost`, `.btn-danger`), `.icon-btn(.bare)`, `.kbd-btn`; ≥36px targets.
- **Forms** `.input`, `.select`, `.textarea` inside `.input-wrap` (leading icon); `.toolbar` for filter rows.
- **Badges** `.badge` + `.b-ok/.b-warn/.b-danger/.b-neutral/.b-ai/.b-mono`, optional `.dot` and `[data-tip]` explanation.
- **Chips** `.chip` (file paths — links when navigable), `.ec-files` wrap.
- **Cards** `.card` (+`.card-head/.card-title/.card-body/.card-pad`); `.stat-grid` → `.stat` (label/value/sub).
- **Tables** `.table`, `.codeview` (line gutter `.ln`, source `.src`, row classes `.attr-ai/.attr-human/.attr-uncommitted` + `.is-sel`).
- **Lists** `.entry-list → .entry-card` (`.is-sel` for j/k), `.alert-list → .alert-row.sev-*`, `.lesson-list → .lesson(.is-pinned)`.
- **Chain** `.chain → .chain-node` (`.cn-kind/.cn-title/.cn-body`) + `.chain-connector` (trace).
- **Steps** `.steps → .step(.done)` (setup).
- **Charts** inline SVG `.svg-chart` + `.legend`; `stackedBar()` returns bar+legend HTML.
- **Overlays** `.overlay` (palette `#palette`, help `#help`, dialog `#dialog`) — focus trap, Esc, click-outside; `.toasts`.
- **States** `.skel` skeletons, `.empty` (+art, actions, `cmd`), `.error-box` (+retry), `.banner.warn/.danger`, `.ratio-hero`.

## 5. Interaction & accessibility

- **Keys:** `Ctrl/⌘K` palette · `/` focus search (or palette) · `?` help · `j/k` list move · `Enter` open · `Esc` close · `g`+`o/t/b/r/m/c/s/a` navigate · `t` theme. Typing in inputs suppresses shortcuts.
- **Deep links:** `#/blame?file=…&line=…`, `#/trace?entry=…`, `#/entry/L-…`, `#/timeline?q=…`.
- **A11y:** AA contrast, visible `--focus-ring`, `aria-current` nav, `aria-pressed/selected` toggles, `aria-live` on counts/toasts, `<caption>`/`aria-label` on code tables, skip-link, reduced-motion.
- **Data safety:** every dynamic string through `esc()`; syntax highlighter re-escapes each fragment; no `innerHTML` of raw API text.
- **Offline:** zero external URLs; icons/charts are inline SVG; fonts are system; settings/notes/pins in `localStorage` (`cl.theme`, `cl.demo`, `cl.pins`, `cl.notes`).

## 6. API contract used by the UI

`/api/health`, `/api/entries`, `/api/entry/:id`, `/api/files`, `/api/file?path=`,
`/api/blame?file[&line=]`, `/api/capsule`, and **`/api/overview`**
(`{head, generated_at, truncated, limits, lines{ai,human,mixed,unknown}, files_scanned,
by_model[], by_tool[], tested{tested,untested}, recent[], trend[]}` + `ok, git, files`).
Dashboard fetches `/api/overview` once (in-flight dedupe, `?refresh=1` bypass);
on failure it falls back to ledger-derived cards with a Retry banner.
Demo mode (`cl.demo`) short-circuits the API layer to a built-in, labeled sample repo.
