# Tucked-ball parenting — design

**Date:** 2026-04-25
**Branch:** `fix/tucked-ball-parenting`
**Scope:** Fix the ball's appearance while a receiver (or scrambling QB) is carrying it. Today the ball jitters, snaps to a wrong angle on catch, and rubber-bands when the carrier turns. The fix is to make the ball a **child of the carrier's group** for the duration of `POST_CATCH`, so Three.js's scene graph carries it automatically.

## Problem

In `index.html`, while `GAME_STATE === 'POST_CATCH'`, `updateBallPhysics` recomputes the ball's world position every frame from the carrier's world transform (lines 966–968):

```js
let offset = new THREE.Vector3(0.5, 1.2, -0.3); offset.applyEuler(activeReceiver.group.rotation);
ball.position.copy(activeReceiver.group.position).add(offset);
ball.rotation.copy(activeReceiver.group.rotation); ball.rotateX(Math.PI / 3);
```

Symptoms:
- Position pops at the moment of the catch (no smoothing into the tuck).
- Orientation looks wrong — ball points up/sideways rather than tucked.
- On hard cuts the ball appears to lag/swing because the offset is recomputed from a Euler that was just changed by `lookAt`.

## Solution: scene-graph parenting

Make the ball a child of the carrier on catch. Set its **local** position and rotation once. Three.js then composes the carrier's world transform with the ball's local transform every frame for free — no per-frame ball math, no drift, no rubber-band.

### Helpers

```js
const TUCKED_POSITION = new THREE.Vector3(0.45, 1.55, 0.15);   // local to carrier.group
const TUCKED_EULER    = new THREE.Euler(Math.PI / 2.5, 0, 0);  // tilt long axis forward

function attachBallToCarrier(carrier) {
    carrier.group.add(ball);
    ball.position.copy(TUCKED_POSITION);
    ball.rotation.copy(TUCKED_EULER);
    ball.velocity = null;
}

function detachBallToScene() {
    if (ball.parent && ball.parent !== scene) {
        scene.attach(ball);   // preserves world transform — no visual snap
    }
}
```

`scene.attach()` (vs `scene.add()`) preserves the ball's world transform when re-parenting, so detaching mid-play causes no visible jump.

### Wire-up

**Entry points** (call `attachBallToCarrier`):
1. Receiver catches pass — `updateBallPhysics`, around line 984.
2. QB scramble crosses LOS — player update QB branch, around line 689.

**Exit points** (call `detachBallToScene`):
1. `triggerTackle` — detach **before** the carrier rotation flip so the ball stays vertical, the runner falls cleanly.
2. `setupFormation` — universal reset before placing the ball at LOS.
3. `startGame` — defensive reset at game start.

**Delete** the world-space carry block (lines 966–968 in `updateBallPhysics`).

## State-machine impact

`POST_CATCH` already gates everything else; no new state needed.

| Transition | Code path | Action |
|---|---|---|
| `PLAY` → `POST_CATCH` (catch) | `updateBallPhysics` line 984 | `attachBallToCarrier(p)` |
| `PLAY` → `POST_CATCH` (QB scramble) | player.update line 689 | `attachBallToCarrier(this)` |
| `POST_CATCH` → `TACKLED` | `triggerTackle` | `detachBallToScene()` first |
| `*` → `MENU` | `handlePlayResult` → `setupFormation` | `detachBallToScene()` |
| game start | `startGame` → `setupFormation` | (covered by above) |
| `*` → `PAUSED` | `checkInputs` | nothing — frozen carrier freezes ball |

## Edge cases

- **Tackle animation.** `triggerTackle` rotates the carrier's group flat. By detaching first with `scene.attach`, the ball stays at its world position and the rotation flip only affects the player mesh. Looks like the runner falls past the ball.
- **Pause/resume.** `animate()` already skips updates while `GAME_STATE === 'PAUSED'`. Parented ball freezes naturally.
- **Sim mode.** Sim path does not touch the ball mesh's transform; `setupFormation` detaches on next play.
- **Interception.** Existing INT path (line 994) calls `handlePlayResult("INTERCEPTED", …)` directly without entering `POST_CATCH`. No attach happens, so no detach needed. Confirm by inspection during testing.

## Test plan

1. Throw to a receiver. At catch, ball should appear in the tuck **immediately**, no jitter or snap-frame.
2. Have receiver cut hard. Ball should rotate with the body, no swing/lag.
3. Get tackled. Ball detaches; tackle animation looks normal.
4. Call a new play. Ball is back at LOS, no orphan parent.
5. QB scramble across LOS. Same tuck pose applies.
6. Pause mid-`POST_CATCH` then resume. Ball stays glued.
7. Throw incomplete (no catch). No regression — the attach path never fires.
8. Interception. Ball goes to ground / play ends as before.

## Risks

Low. The only realistic failure mode is forgetting a detach path, which manifests as "ball stuck floating in the air where the carrier was." Easy to spot, easy to fix by adding `detachBallToScene()` to whatever transition was missed. The full list of carrier-clearing transitions is enumerated in the table above.

## Estimated diff

~25 lines added (two helpers, ~5 call sites), ~3 deleted (the world-space carry block). Single-file change to `index.html`.
