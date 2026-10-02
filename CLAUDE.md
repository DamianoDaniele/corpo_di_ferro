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

The script is split into sections marked with `// ─────────── NAME ───────────` headers: DATI ALLENAMENTI, STATO APP, NAVIGAZIONE, HOME, SESSIONE, TIMER, NUTRIZIONE, CIBO, PROGRESSI, IMPOSTAZIONI, FEATURE SMART, VALUTAZIONE AI and INIT. The event listeners are bound in INIT; for buttons in settings, functions are exposed on `window.*` and called through `onclick`.

- **Cibo tab (`page-food`, nav "Cibo"):** a food diary separate from the fixed meal plan of the Dieta tab (`MEALS`, `mealsChecked`), which it never touches.
  - Data lives in `state.foodLog` (`days`, `saved`, `custom`, `products`, `prefs`), normalized by `normalizeFoodLog()` both in STATO APP and in `normalizeImportedState()`.
  - `FOOD_DB` (in DATI ALLENAMENTI) holds base foods per 100 g from the CREA tables, with the CREA code in the source field and LARN portions. Diary entries store a snapshot of the per-100 g values, so later edits to a food don't rewrite history.
  - Barcodes: `BarcodeDetector` + camera, then Open Food Facts `/api/v2/product/{code}.json` (CORS works). OFF name search (`/cgi/search.pl`) is often down or blocked by CORS, so it is optional and its failure is expected. Chosen products are cached in `foodLog.products` for offline use.
  - `foodTargets()` uses Mifflin-St Jeor with activity 1.4 (no training) / 1.55 (training days from `WEEK_PLAN` or a logged session), protein minimum 1.6 g/kg. `evaluateFoodDay()` / `evaluateFoodWeek()` produce the traffic-light verdict and tips. All events go through `data-fd` attributes and `fdHandle()`, bound once by `bindFood()`.

- **State:** there is a single global `state`, persisted as JSON under the `localStorage` key `cdf_state` by `save()`. Its main fields are `sessions` (completed-session history), `todaySession` (the session in progress), `plan` (the adaptive plan for each exercise), `fatigue` (event log), `bodyMeasurements`, `settings`, `lastSessionType` and `forceDeload`.
  - Migrations run inline right after load, in the STATO APP section. Any new field needs a default there **and** in `normalizeImportedState()`, which handles JSON backup import.
- **Program definition:** `getCurrentSessions()` rebuilds the session definitions every time it is called.
  - The sessions are A = Push (Friday), B = Pull (Saturday) and C = isometric Sunday. C is bodyweight only, with exercises using `unit: 'sec'`.
  - The function then applies cycle modifiers: peak weeks add sets, the deload week drops to 2 sets, and block ≥2 switches to harder variants and adds exercises. Never cache what it returns across cycle changes.
  - Exercise ids (`cp`, `af`, `rw`, `rd`, `pl`, `ws`, …) are the keys used in `state.plan` and `session.exercises`.
- **Cycle:** `getCycleInfo()` is the single source of truth. A cycle is `CYCLE_WEEKS = 9` weeks: 8 working weeks and week 9 as deload. It is derived only from the count of **strength** sessions (2 = 1 week): `weekNum = floor(done / 2) + 1` is the week of the *next* session, so the deload covers sessions 17-18. Isometric session C never advances the cycle. `PROGRESSION[phaseIdx]` holds the text for each phase.
- **Real plates, not a free number:** the user has adjustable dumbbells. `settings.handleWeight` is the empty dumbbell (2 kg, weighed; `settings.handleWeighed` marks that the old 2.5 kg estimate was migrated) and `settings.plates` is the plate inventory (`{5: 8, 2: 3, 1: 4}`). The two 20 kg plates don't fit the handles and are left out.
  - `loadsFor(ex)` lists only the loads that can really be mounted. The plates must split into sides of equal weight: 4 sides for a pair of dumbbells, 2 for one dumbbell (`load: 'single'`, used by `rw`, the curls and `cp` from block 2).
  - `snapLoad` / `nextLoad` / `prevLoad` are the only way weights move. The +/− buttons, the plan, the deload and the engine never produce a load that can't be mounted. `syncPlanToLoads()` re-snaps `state.plan` and an unfinished `todaySession` at startup, after import and after a handle/plates change.
  - `loadShort` / `loadHowTo` / `platesLabel` are the user-facing text: dumbbell count, weight of each dumbbell, and the plates for every side.
  - The pair has a big gap (6 → 12 kg). A fourth 2 kg plate would add 8, 18 and 28.
- **Progression levers, in order:** reps up to `repMax` → reps beyond the range until the next mountable load is reachable → load → extra sets → density (rest cuts, never below 60 s on dynamic exercises) → harder variants. Reaching `repMax` on every set jumps straight to the next lever even if the target was lower.
  - The plates make jumps of +11% to +33% (ACSM 2009 suggests +2–10%), so a load jump is allowed only when `estRepsAt()` (Epley, matching the NSCA %1RM chart) predicts at least `landingReps(ex)` = repMin − 3 (min 6) reps at the new load. After the jump the target restarts from that estimate and climbs back. Without this the engine jumped 12 → 14 kg, reps fell to ~7 and two sessions later it dropped back to 12.
  - Missing the target by 1 rep (5 s on holds) counts as normal set-to-set decline, not a hard session: no fatigue event and no `holds` increment.
  - **Whenever the load goes up** (engine, the + button during a session, or a plan snapped upward) extra sets and rest cuts reset: back to the base sets with full rests. The goal is a lean, hard physique in a deficit, so volume is rebuilt on the new load.
- **Adaptive engine:** `finishSession()` snapshots the session into `state.sessions`. For every exercise it calls `computeNextPlan()`, which writes `state.plan[exId]` and logs fatigue events. In deload, adaptation is skipped on purpose.
  - `computeNextPlan()` separates a genuine rep shortfall from one caused by a skipped rest.
  - Fatigue is a weighted score over a 14-day window: `FATIGUE_WEIGHTS`, `getFatigueLevel()` → green/yellow/red.
- **Weekly schedule:** `WEEK_PLAN` (index = weekday, 0 = Sunday) decides which session is proposed on a given day and which icons appear in the week strip. The `day` field of each session must stay aligned with it.
- **Progressi:** `renderLoadProgress()` shows, for every dumbbell exercise, the current load, history and record; `nextStepFor()` describes the next lever with the same rules as `computeNextPlan()`, so keep the two in sync. `sessionVolume()` feeds both the volume chart and `getVolumeTrend()`, which marks a drop as `programmato` (not fatigue) when recent sessions still hit ≥ 90% of the planned volume.
- **Timer:** rest and holds follow the clock (`timerEndAt`), not the tick count, because Chrome throttles `setInterval` with the screen off. `timerAlert()` vibrates and beeps according to settings; the `AudioContext` is unlocked by the tap that starts the timer.
- **AI evaluation:** the Claude Pro subscription has **no API access**, so there are no `fetch` calls to Claude.
  - `buildSessionReviewPrompt()` / `buildCycleReviewPrompt()` build an Italian coaching prompt from `state`. `shareToClaude()` sends it through `navigator.share`, the Android share sheet leading to the Claude app, with a clipboard fallback.
  - The buttons are on the "Sessione completata" overlay (the cycle button only after a deload session) and on the Progressi page.
