# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository shape

This is a **single-file browser game**: `index.html` at the repo root. There is no build step, no package manager, no test suite, no linter. All HTML, CSS, and JavaScript live in that one file (~1100 lines). The only dependency is Three.js r128, loaded from a CDN (`cdnjs.cloudflare.com`) inside a `<script>` tag — there are no `node_modules` and no bundler.

## Running the game

Just open the HTML file in a browser:

```sh
open index.html
```

For local dev with proper CDN/CORS behavior, serve the directory and load `http://localhost:8000/`:

```sh
python3 -m http.server 8000
```

There is nothing to build, install, or test. Edits are live on browser refresh.

## Code architecture

The single `<script>` block at the bottom of the HTML is organized into two halves:

1. **Franchise / scheduling layer** (top): pure data + DOM. Tracks `MY_TEAM`, `NFL_TEAMS`, `seasonSchedule`, `myRecord`, `currentWeek`, `currentOpponent`, and the quarter-by-quarter `qtrScores`. Functions like `generateSchedule`, `updateMenuUI`, `openSchedule`, `openMatchupScreen`, `advanceWeek` only touch DOM and franchise state — they do not touch Three.js.

2. **Three.js engine** (rest): one scene, one camera, one renderer, one `animate()` loop driven by `requestAnimationFrame`. Field/stadium/players/ball are all built imperatively in `init()` → `buildField()` / `buildStadium()` / `setupFormation()`. The ball uses a hand-rolled physics integrator in `updateBallPhysics(dt)` (gravity is the constant `30`, not 9.8 — tuned for arcade feel).

### The `GAME_STATE` state machine

A single string global, `GAME_STATE`, gates everything. Values seen in the code:

`'MENU'`, `'MATCHUP'`, `'PLAYBOOK'`, `'PRE_SNAP'`, `'PLAY'`, `'POST_CATCH'`, `'TACKLED'`, `'GETTING_UP'`, `'PAUSED'`, `'SIMULATING'`, `'GAME_OVER'`.

The `animate()` loop at the bottom of the file is the master tick: it short-circuits player/ball updates whenever `GAME_STATE` is in `{PAUSED, MENU, SIMULATING, MATCHUP, GAME_OVER}`. When adding a new screen or modal, you almost always need to add its state to that exclusion set, otherwise the field/players keep updating behind it.

State transitions are scattered (search for `GAME_STATE =`) — there is no central reducer. Key transition points: `openMatchupScreen` → `MATCHUP`, `setupFormation` → `PLAYBOOK`, `selectPlay` → `PRE_SNAP`, `snapBall` → `PLAY`, the receiver-catch handler in the player update loop → `POST_CATCH`, `triggerTackle` → `TACKLED` → `GETTING_UP`, `handlePlayResult` → `MENU`.

### Pass mechanics

The trajectory preview, pass power, and actual throw all use the same kinematic equations with gravity `30`. If you change the gravity constant, change it in **both** the trajectory preview block inside `animate()` and `updateBallPhysics`, or the preview will lie to the player.

### Sim mode

When the user skips a game, `startSim` runs `simIntervalTimer` (a `setInterval`) that calls `processSimPlay` repeatedly. This path is entirely separate from the Three.js loop — `GAME_STATE` is `'SIMULATING'` and `animate()` skips physics. `stopSim` and `skipToNextQuarter` are the exits.

## Editing conventions to preserve

- Keep everything in the one HTML file. Splitting into modules would require introducing a build step, which the project deliberately avoids.
- The retro look depends on the `Press Start 2P` Google Font and the `#002244` / `#ffcc00` / `#e31837` palette in the CSS — don't refactor those into "design tokens" unless asked.
- Three.js is pinned to **r128** via CDN. APIs from later majors (e.g. removal of `Geometry`, changes to `Lambert` materials) will silently misbehave; verify against r128 docs.
