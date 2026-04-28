# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Single-file vanilla JS workout interval timer. No build step, no framework, no dependencies. The entire app — HTML, CSS, and JS — lives in `index.html`. Deployed to GitHub Pages automatically on push to `main` via `.github/workflows/deploy.yml`.

## Development

Open `index.html` directly in a browser (`file://` works fine). There is no build, no npm, no server required. To test changes, refresh the browser.

## Architecture

Everything is in `index.html` in this order: `<style>` (all CSS) → `<body>` (all HTML) → `<script>` (all JS).

### View state machine

`idleView` (string: `'templates' | 'detail' | 'config' | 'ready'`) tracks which idle screen is visible. All views share one `<main>` container; visibility is toggled with the `hidden` CSS class.

`hideAllViews(clearTimeline = true)` hides everything. Pass `false` to preserve the rendered timeline (used by `showTimerView()` so the timeline survives the transition into the active timer).

View entry points: `showTemplateView()`, `showDetailView(id)`, `showConfigView()`, `showReadyView()`.

### Timer state machine

All timer state lives in the `T` object:
- `T.status`: `'idle' | 'running' | 'paused' | 'done'`
- `T.phases`: flat array of `{ type, label, duration }` built by `buildPhases(cfg)`
- `T.idx`: current phase index
- The loop runs via `requestAnimationFrame` → `tick(ts)`

Sound deduplication: `T.lastTick` (integer seconds) prevents firing the same sound twice per second. `Math.ceil(left)` is used so each whole-second value fires exactly once.

### Audio engine

`tone(freq, dur, offset, amp, shape)` is the primitive. All sounds in `SFX` are composed from multiple `tone()` calls with time offsets. Web Audio API is lazy-initialized on first user interaction.

### Data / localStorage

| Key | Contents |
|-----|----------|
| `wt_config_v1` | Last-used config object |
| `wt_exercises_v1` | User-added custom exercises (string array) |
| `wt_saved_workouts_v1` | Saved named workouts `[{ id, name, config }]` |

Saved workouts use `u<timestamp>` IDs. Built-in template IDs are plain strings like `'tabata'`, `'hiit'`, etc.

`currentDetailId` — ID of the workout currently shown in detail view.  
`currentEditWorkoutId` — when set, the Save button says "Update" and overwrites that saved workout.

### Exercise picker

`EXERCISE_CATS` is the source of truth for categorised exercises. `PRESET_EXERCISES` is derived from it (`[...new Set(EXERCISE_CATS.flatMap(c => c.list))]`). The picker panel (`#ex-picker-panel`) is injected into the DOM inline below the triggering set button and removed on close. `activePickerSetIdx` and `activePickerCat` are module-level state.

### Phase colours

```js
const PHASE_COLORS = { warmup: '#f59e0b', set: '#22c55e', rest: '#60a5fa', cooldown: '#c084fc' };
```

Used consistently across timeline pills, phase bar segments, and `document.body.dataset.phase` (for CSS theming during active timer).

### Config object shape

```js
{
  warmup, sets, setDuration, rest, cooldown,  // seconds
  countdownN,   // final-N-seconds countdown beeps (0 = off)
  tickEvery,    // boolean — per-second tick sound
  setNames,     // string[] — one per set, may be empty strings
}
```

`readConfig()` reads the live form. `saveConfig()` persists it. `loadConfig()` restores the form on init.

## Key patterns

- **Adding a new SFX**: add a method to the `SFX` object using `tone()` calls. Wire it up inside `tick()` or `enterPhase()`.
- **Adding a new view**: add its HTML, add it to `hideAllViews()`, write a `showXxxView()` function that sets `idleView` and calls `hideAllViews()` then unhides the element.
- **Adding a new config field**: add the `<input>` in the config panel HTML, read it in `readConfig()`, persist/restore in `saveConfig()`/`loadConfig()`, and use it in `buildPhases()` or `tick()`.
