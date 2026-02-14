# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**GREYBOX TACTICAL** — a browser-based 3D FPS shooter, entirely in a single file (`index.html`, ~4300 lines). No build system, no bundler, no npm. Just open the HTML file in a browser.

## Tech Stack

- **Three.js r128** (UMD from cdnjs) — 3D rendering
- **Cannon.js 0.6.2** (UMD from cdnjs) — physics
- **Web Audio API** — all sounds are procedurally synthesized (no audio files)
- **All procedural** — no external model/texture/sound assets

## How to Run

Open `index.html` in a browser. No server needed (though a local server avoids CORS if needed later).

## Architecture (single file: index.html)

The file is structured top-to-bottom as: **HTML/CSS → CDN scripts → JavaScript**.

### HTML sections (~lines 1-470)
- CSS styles (menu, HUD, death/victory screens, settings, loadout, perk panels)
- Menu screen, HUD elements, death screen, victory screen
- Settings panel, loadout panel, quit confirm, perk selection UI
- CDN script tags for Three.js and Cannon.js

### JavaScript sections (~lines 473-4340)
Key sections in order:

| Section | Description |
|---|---|
| **Globals & State** (~475) | All game state: `STATE` enum, player vars, weapon/armor defs, perk defs, feature flags |
| **Init** (~600) | Sets up renderer, scene, camera, physics world, builds level, wires events |
| **Lights / Ground / Walls / Cover** (~825-1020) | Level geometry + physics bodies |
| **Feature systems** (~1020-1170) | Breakable windows, moving cover, weapon models |
| **Enemies** (~1180) | `enemyTypes` definitions, `createEnemy()` with hierarchical pivot-based skeleton |
| **Enemy AI** (~1870) | State machine: patrol → chase → attack → seekCover → wounded → surrendered |
| **Animation** (~1970) | `animateEnemy()` — walking, breathing, two-handed aiming, recoil |
| **Shooting** (~2760) | Raycasting, hit detection, wound/kill logic, bullet impacts |
| **Kill / Ragdoll** (~3150) | `killEnemy()` — single-body ragdoll + separate gun drop |
| **Waves & Perks** (~3285) | 5-wave system, perk selection between waves, endless mode |
| **Game Loop** (~3900) | `update()` — player movement, physics step, AI, HUD, rendering |
| **Settings / Loadout / Menu** (~4200) | UI event handlers |

### Enemy Model Hierarchy

Enemies use a hierarchical pivot system for procedural animation:
```
group (root, positioned in world)
  └─ bodyPivot (breathing/sway)
       ├─ pelvisMesh, belt, pouches
       ├─ torsoMesh, front/back plates, collar
       ├─ shoulder pads
       ├─ headMesh, helmet, visor (red glow), cheek/chin guards
       ├─ armLPivot (left arm — support hand)
       │    └─ upper arm, bracer, elbow pad, glove
       ├─ armRPivot (right arm — trigger hand)
       │    ├─ upper arm, bracer, elbow pad, glove
       │    └─ gunGroup (detachable: receiver, barrel, mag, stock, scope)
       ├─ legLPivot → upper leg, knee pad, shin, boot
       └─ legRPivot → upper leg, knee pad, shin, boot
```

Gun is built along Y axis (barrel = -Y) so it points forward when arm rotates. Both arms maintain two-handed grip in all states.

### Physics Conventions

- `world.defaultContactMaterial.friction = 0` — no friction, direct velocity control
- Player: `Sphere(0.4)`, mass 80, `fixedRotation`, `linearDamping = 0`
- Player movement: set `playerBody.velocity` directly BEFORE `world.step()`
- Enemies: `Cylinder` body, mass 0 (kinematic), moved via position manipulation
- Ragdolls: mass > 0 bodies created on death, synced in game loop
- Cannon.js 0.6.2 API: `body.applyImpulse(impulse, worldPoint)` takes two `CANNON.Vec3` args. No `ConeTwistConstraint` — use `PointToPointConstraint` only.

### Game Loop Order (in `update()`)

```
updatePlayer(delta) → world.step() → syncPlayerCamera() →
updateWeapon(delta) → updateEnemyAI(delta) → updateGrenades(delta) →
updateLaser() → updateEvacZone(delta) → updateHealth(delta) →
updateMinimap() → updateCompass() → updateHUD() →
updateFootsteps(delta) → updatePickups(delta) → updateMovingCovers(delta) →
ragdoll sync → FPS counter → renderer.render()
```

### Wound/Kill Mechanic

- **1st body shot** → enemy enters `wounded` state (30% chance: surrender instead)
- **2nd body shot** → kill (regardless of HP)
- **Headshot** → always instant kill
- Hit detection: `raycaster.intersectObjects([enemy.group], true)`, check if hit object `=== enemy.head`

## Key Patterns to Follow

- **All state is global** — no modules, no classes. Variables declared at top of script block.
- **Cleanup in `resetGame()`** — any new system with scene objects or physics bodies MUST be cleaned up here.
- **New globals go near line 475-560** — alongside existing declarations.
- **New update functions** go inside the `STATE.PLAYING` block in `update()`.
- **Sounds are synthesized** — use `audioCtx.createBuffer()` + `createBufferSource()` pattern. See `playGunshot()` or `playFootstep()` as examples.
- **Enemy types** are defined in `enemyTypes` object. Type-specific visuals go in the `createEnemy()` type adjustments block.
- **Wave system**: `startWave(n)` spawns enemies, `killEnemy()` decrements `waveEnemiesLeft` and checks completion.

## Common Pitfalls

- **Don't use `element.textContent = ...` on containers with child elements** — it destroys child DOM nodes. Update specific child spans instead.
- **Cannon.js 0.6.2 has no `ConeTwistConstraint`** — only `PointToPointConstraint` is available for ragdoll joints.
- **`window._slowMoFactor`** — always reference the global, not a local copy.
- **Duplicate declarations** — since everything is in one script block, verify a variable name isn't already taken before declaring it.
- **Syntax check**: `node -e "const fs=require('fs');const c=fs.readFileSync('index.html','utf8');const m=c.match(/<script>([\s\S]*)<\/script>/);try{new Function(m[1]);console.log('OK')}catch(e){console.log(e.message)}"` — run this after edits.
