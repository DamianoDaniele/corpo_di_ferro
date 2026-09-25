# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

"Corpo di Ferro" is a personal home-workout tracker (training, nutrition, progress). The user runs it from Chrome on a Samsung Galaxy S24 Ultra. The whole app lives in one file, `index.html`: inline CSS plus one inline `<script>` wrapped in an IIFE. There is no framework, no build step, no package manager, no tests and no linter. The only external resource is Google Fonts.

It is deployed through **GitHub Pages from `main`**. That is why the file is named `index.html`. Anything merged to `main` and pushed goes live on the phone.

**Language:** the UI text, code comments and commit messages are in **Italian**. Keep it that way. The code has a comment on almost every line; match that density. Commits use conventional prefixes with an Italian description, e.g. `feat(workout): ...` or `fix(home): ...`. Feature branches are merged into `main` with `--no-ff`.

## Running / testing

Serve the folder statically, e.g. `npx -y http-server -p 8765 -c-1 .`, then open it at a mobile viewport (375×812).

- In the Claude desktop browser pane, `localStorage` is blocked, both under `data:` URLs and on localhost. To test with data, make a copy of `index.html` in the scratchpad and inject a `<script>` right after `<head>`. That script redefines `window.localStorage` with an in-memory object holding a seeded `cdf_state`, and stubs `navigator.share` when needed.
- Seeded sessions must include the fields that `finishSession()` really writes, such as `completedSets`. If they don't, the Progressi charts throw.
- To check a quick syntax or logic change without a browser, pull out the `<script>` contents and `eval` them in Node against stubbed `document`/`localStorage`.

## Architecture (index.html)

The script is split into sections marked with `// ─────────── NAME ───────────` headers: DATI ALLENAMENTI, STATO APP, NAVIGAZIONE, HOME, SESSIONE, TIMER, NUTRIZIONE, PROGRESSI, IMPOSTAZIONI, FEATURE SMART, VALUTAZIONE AI and INIT. The event listeners are bound in INIT; for buttons in settings, functions are exposed on `window.*` and called through `onclick`.

- **State:** there is a single global `state`, persisted as JSON under the `localStorage` key `cdf_state` by `save()`. Its main fields are `sessions` (completed-session history), `todaySession` (the session in progress), `plan` (the adaptive plan for each exercise), `fatigue` (event log), `bodyMeasurements`, `settings`, `lastSessionType` and `forceDeload`.
  - Migrations run inline right after load, in the STATO APP section. Any new field needs a default there **and** in `normalizeImportedState()`, which handles JSON backup import.
- **Program definition:** `getCurrentSessions()` rebuilds the session definitions every time it is called.
  - The sessions are A = Push (Friday), B = Pull (Saturday) and C = isometric Sunday. C is bodyweight only, with exercises using `unit: 'sec'`.
  - The function then applies cycle modifiers: peak weeks add sets, the deload week drops to 2 sets, and block ≥2 switches to harder variants and adds exercises. Never cache what it returns across cycle changes.
  - Exercise ids (`cp`, `af`, `rw`, `rd`, `pl`, `ws`, …) are the keys used in `state.plan` and `session.exercises`.
- **Cycle:** `getCycleInfo()` is the single source of truth. A cycle is `CYCLE_WEEKS = 9` weeks: 8 working weeks and week 9 as deload. It is derived only from the count of **strength** sessions (2 = 1 week). Isometric session C never advances the cycle. `PROGRESSION[phaseIdx]` holds the text for each phase.
- **Hard weight cap:** the user owns dumbbells up to `settings.maxWeight` (10 kg). Progression must never come from adding load. The levers are reps → sets → range → density (rest cuts) → harder variants.
- **Adaptive engine:** `finishSession()` snapshots the session into `state.sessions`. For every exercise it calls `computeNextPlan()`, which writes `state.plan[exId]` and logs fatigue events. In deload, adaptation is skipped on purpose.
  - `computeNextPlan()` separates a genuine rep shortfall from one caused by a skipped rest.
  - Fatigue is a weighted score over a 14-day window: `FATIGUE_WEIGHTS`, `getFatigueLevel()` → green/yellow/red.
- **Weekly schedule:** `WEEK_PLAN` (index = weekday, 0 = Sunday) decides which session is proposed on a given day and which icons appear in the week strip. The `day` field of each session must stay aligned with it.
- **AI evaluation:** the Claude Pro subscription has **no API access**, so there are no `fetch` calls to Claude.
  - `buildSessionReviewPrompt()` / `buildCycleReviewPrompt()` build an Italian coaching prompt from `state`. `shareToClaude()` sends it through `navigator.share`, the Android share sheet leading to the Claude app, with a clipboard fallback.
  - The buttons are on the "Sessione completata" overlay (the cycle button only after a deload session) and on the Progressi page.
