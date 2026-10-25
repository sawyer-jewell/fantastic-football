# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository shape

This is a **single-file browser game**: `index.html` at the repo root. There is no build step, no package manager, no test suite, no linter. All HTML, CSS, and JavaScript live in that one file. The only dependency is Three.js r128, loaded from a CDN (`cdnjs.cloudflare.com`) with an SRI hash — there are no `node_modules` and no bundler. (The player model is read by a small built-in GLB reader, `parseGLB`, not Three's GLTFLoader: some Chromebooks block other CDNs.)

### The player model

Players are drawn from a Blender model: `models/player.blend` → exported to `models/player.glb` → embedded in `index.html` as base64 inside `<script id="player-model-glb" type="application/octet-stream">` (so the game still works opened straight from disk). `loadPlayerModel()` parses it at startup with `parseGLB` and `splitPlayerModel()` splits it into parts by Blender object name (`Body`, `Head`, `Helmet`, `Facemask`, `Left/RightUpperArm`, `Forearm`, `Hand`, `Thigh`...); `Player.buildFromModel()` bakes each part into one vertex-colored mesh per player look (cached) and hangs it on the same joints the game animates. Uniform colors are matched by Blender **material** names (`Jersey`, `Pants`, `Helmet`, `Skin`, ...), and the decals (`NumberFront/Back`, `NameBack`, `NumberShoulderL/R`) get per-player canvas textures. If the model or loader fails, the classic box players are used. That base64 line is ~300 KB — don't print it; grep with `cut -c1-120`.

To update the model: edit `models/player.blend`, export `player.glb` (glTF binary, +Y up, selection = the `FootballPlayer` collection), then replace the base64 inside that script tag (e.g. PowerShell `[Convert]::ToBase64String([IO.File]::ReadAllBytes('models\player.glb'))`). Keep object and material names, or update `splitPlayerModel`/`buildFromModel` to match. `models/viewer.html` previews the .glb on its own (serve the folder over HTTP).

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
   - **Saves:** three slots in localStorage (`SAVE_KEY-1..3`, `activeSlot`); the game opens on `openSaveSelect`. Each save holds its team (`setMyTeam` rewrites `MY_TEAM`/`NFL_TEAMS` in place from `ALL_TEAMS`, all 32), league, `roster`, `credits` and stats (`getSaveData`/`applySaveData`).
   - **Roster:** twelve starters (`ALL_SLOTS`: the six on offense in formation order, `K`, and `DEF_SLOTS`), each with the skills in `SLOT_RATINGS`, an age, and season/career lines (`creditPlayer` during a game, rolled up in `finalizeGameStats`). Coaching credits (`awardCredits`, `upgradeRating`) and the starting/rookie drafts (`openDraft`) change it.
   - Anything that runs at startup (before the end of the script) must only use function declarations or globals declared above it — `const`/`let` further down are still in their temporal dead zone.

2. **Three.js engine** (rest): one scene, one camera, one renderer, one `animate()` loop driven by `requestAnimationFrame`. Field/stadium/players/ball are all built imperatively in `init()` → `buildField()` / `buildStadium()` / `setupFormation()`. The ball uses a hand-rolled physics integrator in `updateBallPhysics(dt)` (gravity is the constant `30`, not 9.8 — tuned for arcade feel).

### The `GAME_STATE` state machine

A single string global, `GAME_STATE`, gates everything. Values seen in the code:

`'MENU'`, `'MATCHUP'`, `'PLAYBOOK'`, `'PRE_SNAP'`, `'PLAY'`, `'POST_CATCH'`, `'TACKLED'`, `'GETTING_UP'`, `'PAUSED'`, `'SIMULATING'`, `'GAME_OVER'`, plus the special-teams and defense states (`'XP_ANIM'`, `'KICKOFF_RETURN'`, `'FG_BLOCK'`, `'DEFENSE'`, `'DRONE_SIM'`, `'INT_RETURN'`, `'TIMEOUT'`, `'CELEBRATE'` for a defense's turnover celebration, ...).

The `animate()` loop at the bottom of the file is the master tick: it short-circuits player/ball updates whenever `GAME_STATE` is in `{PAUSED, MENU, SIMULATING, MATCHUP, GAME_OVER}`. When adding a new screen or modal, you almost always need to add its state to that exclusion set, otherwise the field/players keep updating behind it.

State transitions are scattered (search for `GAME_STATE =`) — there is no central reducer. Key transition points: `openMatchupScreen` → `MATCHUP`, `setupFormation` → `PLAYBOOK`, `selectPlay` → `PRE_SNAP`, `snapBall` → `PLAY`, the receiver-catch handler in the player update loop → `POST_CATCH`, `triggerTackle` → `TACKLED` → `GETTING_UP`, `handlePlayResult` → `MENU`.

### Pass mechanics

The trajectory preview, pass power, and actual throw all use the same kinematic equations with gravity `30`, and both go through `throwPlan` (range from pass power and the QB's arm; out-of-range throws come down short, flight time capped). If you change the gravity constant, change it in **both** the trajectory preview block inside `animate()` and `updateBallPhysics`, or the preview will lie to the player.

### Sim mode

When the user skips a game, `startSim` runs `simIntervalTimer` (a `setInterval`) that calls `processSimPlay` repeatedly. This path is entirely separate from the Three.js loop — `GAME_STATE` is `'SIMULATING'` and `animate()` skips physics. `stopSim` and `skipToNextQuarter` are the exits.

## Editing conventions to preserve

- Keep everything in the one HTML file. Splitting into modules would require introducing a build step, which the project deliberately avoids.
- The retro look depends on the `Press Start 2P` Google Font and the `#002244` / `#ffcc00` / `#e31837` palette in the CSS — don't refactor those into "design tokens" unless asked.
- Three.js is pinned to **r128** via CDN, with a Subresource Integrity (SRI) hash on the `<script>` tag. APIs from later majors (e.g. removal of `Geometry`, changes to `Lambert` materials) will silently misbehave; verify against r128 docs. **If you ever change the Three.js URL or version, you must regenerate the SRI hash** or the browser will refuse to load the script:

  ```sh
  curl -sS <new-url> | openssl dgst -sha384 -binary | openssl base64 -A
  ```
