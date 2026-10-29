# CodeLedger — UX Audit (before redesign)

_Date of audit: 2026-10-29 · Scope: full product UI, current `codeledger/web/index.html` (902 lines) served by `ledger ui`._

---

## 1. Stack & architecture (what we're working with)

| Layer | Reality |
| --- | --- |
| Backend | Python 3.10+ package `codeledger/` — Typer CLI + Rich. No database, no network, no LLM calls. |
| Frontend | **One static file**: `codeledger/web/index.html` — inline CSS (~313 lines) + inline vanilla JS (~590 lines). No framework, no build step, no `package.json`, no Tailwind, no npm. |
| Server | `codeledger/webui.py` — stdlib `http.server`, binds `127.0.0.1` only. Serves the single HTML + a read-only JSON API. POST → `405` by contract. |
| Packaging | `pyproject.toml` packages only `codeledger/web/*.html` — **any split into .js/.css assets would not ship in the wheel** and would change the documented security posture ("serves one HTML file with no CDN or external assets"). |
| Tests | `tests/test_webui.py` — live-server tests over every endpoint + path-traversal guards. `test_index_served` asserts the index contains `"CodeLedger"` and `"Timeline"`. |

**Hard constraints a redesign must respect:**

1. Single self-contained `index.html` (no CDN, no external assets, no build step).
2. Read-only API — every "write" feature (add/edit/pin lessons) must either be client-local (localStorage) or generate a copyable CLI command (`ledger log …`), never a POST.
3. Existing API surface is fixed unless we *add* new read-only endpoints (additive, low-risk):
   `/api/health`, `/api/entries`, `/api/entry/<id>`, `/api/blame?file=&line=`, `/api/capsule`, `/api/files`, `/api/file?path=`.
4. Existing tests must keep passing (string checks on the index; endpoint payload shapes).

**Data model** (`.ledger/decisions.jsonl`, one JSON object per line): `id (L-NNNN)`, `timestamp`, `prompt`, `model`, `tool`, `files[]`, `decision`, `rejected_alternatives[{approach, reason}]`, `status (pending|committed)`, `commit` (sha), `tests_included`, `gate_result (pass|fail|bypassed|n/a)`, `redactions`.

---

## 2. Current screens & flows

| # | Screen (sidebar tab) | What it shows | Primary flow |
| --- | --- | --- | --- |
| 1 | **Timeline** | All ledger entries newest-first as cards; search box; model + status filters | Scan entries → expand prompt inline |
| 2 | **Blame** | File picker → code table with AI gutter bar → click a line → right panel with commit + entry (prompt, decision, files) | Pick file → load → click line → read provenance |
| 3 | **Capsule** | Rendered `CAPSULE.md` markdown + "Copy for any AI tool" | Read snapshot → copy → paste into another AI session |
| 4 | **Health** | 3 stat cards (entries, % with tests, redactions) + gate-outcome stacked bar + per-model horizontal bars | Glance at stats |

Global: theme toggle (light/dark, system-aware via `prefers-color-scheme` + localStorage), init banner when `.ledger/` is missing.

---

## 3. What's missing vs the product brief

| Requested screen / capability | Status today |
| --- | --- |
| Dashboard / overview (AI-vs-human ratio, recent changes, untested warnings, top models/tools, bug-to-prompt alerts) | **Partial** — Health has only 3 stats + 2 charts. No AI/human ratio, no tool chart, no untested list, no alerts, no recent-activity feed. |
| Ledger / timeline with filters for model, tool, author, date, tested/untested, file path | **Partial** — only model + status filters exist. No tool/author/date/tested/file filters. |
| Command palette (Cmd/Ctrl+K), `/` search, j/k navigation, `?` help | **Missing** entirely. |
| Blame: syntax highlighting, "why this line exists" summary, diff view, linked tests/bugs | **Partial** — plain `<code>` text, no highlight, no summary line, no diff, no test/bug links. |
| Entry detail page (full prompt, params, decision trail, files, test status, revision history, "copy context") | **Missing** — `/api/entry/<id>` exists but **the UI never calls it**. Entries are only cards; prompt hides in a `<details>`. |
| Memory / lessons (aggregate "don't do this again", add/edit/pin) | **Missing as a view** — rejected alternatives are buried per-card; Capsule renders a read-only section. |
| Bug trace (bug → code → prompt → model → decision chain) | **Missing** — no view exists; data is joinable client-side via `entry.commit` + `/api/blame`. |
| Settings / integrations (repos, AI tools, team, API keys) | **Missing** — nothing exists. (Honest scope: this is a local read-only tool, so settings = repo info, hooks status, integrations how-to, theme, shortcuts.) |
| Onboarding / 3-step setup, illustrated empty states | **Missing** — one dismissible-forever banner line: "run `ledger init`". Empty states are plain text boxes, no illustration, no next-step guidance beyond one command. |
| Deep links (share `file:line` or entry id) | **Missing** — hash routing only picks a tab (`#timeline`). Blame cannot be deep-linked, which kills the #1 sharing flow for a "blame" product. |
| Toasts, breadcrumbs, skip-link, reduced-motion | **Missing** |

---

## 4. What's confusing / inconsistent / broken

### Bugs found (functional)

1. **Blame auto-load is broken.** `blPickBest()` references `sel` which is scoped to `populateFiles()` — under `"use strict"` this throws a `ReferenceError` on screen load, so the file picker never auto-selects/loads a file and the viewer shows a swallowed error until the user manually clicks *Load blame* (`index.html` ~L602–610).
2. **Invalid HTML in the code viewer.** When a file exceeds 2500 lines, a `<div>` is injected inside `<table>` (`renderCode`, ~L637) — browsers reparent it and can blank the table.
3. **Fragile markdown renderer.** `mdLite()` is a regex soup; nested lists / code fences in `CAPSULE.md` render unpredictably.
4. **`init` banner logic is opaque** — it toggles off when counts exist, but the condition `counts.total >= 0` is always true when counts exist, making the second clause dead code (~L813). Users on a git repo without `.ledger/` see the banner forever with no richer guidance.

### UX / IA problems

5. **Timeline cards are overloaded** — id, model, tool, status, gate, tests, commit, time, decision, prompt, files, rejected alternatives, redactions all in one card. Dense-but-readable failed into dense-and-noisy; there's no progressive disclosure path *to* a focused entry view.
6. **No place for the hero story** — the "why this line exists" answer is scattered across an aside panel with raw chips; no one-line natural-language summary.
7. **Capsule is the only "memory" surface**, but it's a generated artifact (read-only, line-capped). There's no browsable, searchable lessons list.
8. **Health's charts don't add up to decisions** — you can't answer "what should I do next?" (e.g. "3 AI changes are untested", "entry L-0004 bypassed the gate").
9. **Terminology drift**: "Timeline" vs "entries" vs "ledger"; "Capsule" is jargon with no tooltip; "gate" is unexplained anywhere in the UI.
10. **No keyboard model** for power users (the target audience lives in keyboards).
11. **Mobile**: sidebar becomes a horizontal tab strip that can overflow; no drawer; blame aside stacks but is always rendered below a very tall code table.
12. **Copy affordances are one-off** (button label swap on Capsule only); entry IDs/commits aren't copyable; no "copy context for another AI" on an entry.
13. **Theme default is light** in the HTML attribute; brief asks dark-by-default with system-aware behavior (JS fixes it after first paint — causes a flash).
14. **Typography**: system UI stack; no Inter/Geist or JetBrains Mono preference (must stay offline-safe: prefer locally-installed fonts with solid fallbacks — no CDN allowed).
15. **Accessibility gaps**: no skip-to-content link, no focus trap in the (nonexistent) dialogs, skeleton `pulse` animation ignores `prefers-reduced-motion`, blame rows use `role="button"` on `<tr>` (works but noisy), color is the only AI/human signal in the gutter (no legend, no shape/text cue).

### Performance

16. Whole lists render as one giant `innerHTML` string — fine for tens of entries, unusable at hundreds (no windowing/virtualization, no pagination).
17. No request de-duplication; Timeline and Health each refetch `/api/entries`.
18. No `loading` semantics (`aria-busy`), no layout reservation beyond skeleton boxes.

---

## 5. What's good (keep and build on)

- **Solid security posture**: loopback-only, read-only, traversal-guarded, no CDN. The redesign must not regress this.
- **Honest empty/error states** with copyable commands (`ledger demo`, `ledger log …`).
- **Hash-based routing + per-view lazy loading** — right pattern, just too shallow.
- **CSS custom properties already in use** — a clean pivot point for a real token system.
- **Focus-visible rings and aria-labels already present** on interactive elements.
- **API is genuinely good** — entry detail, full-file blame with per-line annotations, files list, file content. The UI simply underuses it.

---

## 6. Summary verdict

The current UI is a competent **4-tab internal dashboard** (~900 lines of one file) that proves the data model but doesn't yet deliver the product story: no overview that answers "what needs attention", no entry detail, no memory/lessons surface, no trace, no keyboard-first navigation, one load-time bug in the hero (blame) screen, and no onboarding beyond a banner. The backend API already supports ~80% of the requested redesign **read-only**; the remaining gaps (lessons CRUD, bug entities, integrations) must be handled client-side or honestly scoped as read-only "how to" surfaces.

**Recommended path:** keep the single-file, stdlib-served, zero-dependency architecture; rebuild the frontend as a token-driven design system with 8 client-routed screens, fix the blame bug, add palette/keyboard/a11y, and derive Memory/Trace/Dashboard views from existing API data + localStorage. Touch `webui.py` only if we add a new read-only aggregate endpoint (optional).
