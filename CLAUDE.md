# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**GREYBOX TACTICAL** — a browser-based 3D FPS shooter, entirely in a single file (`index.html`, ~6600 lines). No build system, no bundler, no npm. Just open the HTML file in a browser.

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
- CSS styles (menu, HUD, death/victory screens, settings, loadout, perk panels, Windows error popups, night vision overlay)
- Menu screen with mode buttons (Standard, Last Stand, Snow, Error, Night, Matrix)
- HUD elements, death screen, victory screen
- Settings panel (including weather toggle), loadout panel, quit confirm, perk selection UI
- CDN script tags for Three.js and Cannon.js

### JavaScript sections (~lines 473-6600)
Key sections in order:

| Section | Description |
|---|---|
| **Globals & State** (~475-640) | All game state: `STATE` enum, player vars, weapon/armor defs, perk defs, feature flags, mode flags (`snowMode`, `errorMode`, `nightMode`, `matrixMode`), weather/music/tracer globals |
| **Init** (~650) | Sets up renderer, scene, camera, physics world, builds level, wires events |
| **Mode Start Functions** (~780-870) | `startSnowMode()`, `startErrorMode()`, `startNightMode()`, `startMatrixMode()`, `toggleNightVision()` |
| **Lights / Ground / Walls / Cover** (~900-1100) | Level geometry + physics bodies, mode-specific lighting (night/matrix/error) |
| **Feature systems** (~1100-1250) | Breakable windows, moving cover, weapon models |
| **Enemies** (~1260) | `enemyTypes` definitions, `createEnemy()` with mode-specific materials (ice/snow/glitch/normal) |
| **Enemy AI** (~1950) | State machine: patrol → chase → attack → seekCover → wounded → surrendered. Error mode: zombie behavior (no shooting, melee only) |
| **Animation** (~2060) | `animateEnemy()` — walking, breathing, two-handed aiming, recoil |
| **Cyberpunk Music** (~2820-3080) | `startCyberMusic()` / `stopCyberMusic()` — 5-layer procedural synthwave (bass, pad, arpeggio, drums, atmospheric FX) |
| **Weather System** (~3200) | `startRain()`, `stopRain()`, `updateWeather()` — 2000 rain particles, lightning flashes, thunder |
| **Shooting** (~3300) | Raycasting, hit detection, wound/kill logic, bullet impacts, tracers |
| **Detach/Dismember** (~3520) | `detachLimb()` — head/arm/leg separation with ice/snow/normal particle variants |
| **Kill / Ragdoll** (~3700) | `killEnemy()` — ragdoll + error popup spawn in error mode |
| **Waves & Perks** (~3850) | 5-wave system, perk selection between waves, endless mode |
| **Game Loop** (~4500) | `update()` — player movement, physics, AI, tracers, matrix bullets, weather, HUD, render |
| **Melee Weapons** (~5400) | `meleeAttack()` (knife + hand model, stab animation), `katanaDash()` (katana + two hands, horizontal head-cut slash) |
| **Key Bindings** (~5230) | All keyboard/mouse input handlers |
| **Settings / Loadout / Menu** (~6370) | UI event handlers, settings apply/open |

### Player Controls

| Key | Action |
|---|---|
| W/A/S/D | Movement |
| Space | Jump |
| Shift | Sprint |
| Ctrl/C | Crouch |
| Q/E | Lean left/right |
| LMB | Shoot |
| RMB (hold) | Aim |
| R | Reload |
| G | Grenade |
| V | Knife melee (backstab = instant kill, NO head dismemberment) |
| X | Katana dash — teleport to nearest enemy, horizontal slash, ALWAYS cuts head off (2s cooldown) |
| F | Bullet Time (slow motion) |
| N | Night vision toggle (Night Mode only) |

### Game Modes

- **Standard** — 5 waves of enemies, extraction after final wave
- **Last Stand / Endless** — infinite waves, high score tracking
- **Snow Mode** — ice enemies (50% chance, translucent, shatter on death) and snow enemies (opaque white, dissolve)
- **Error Mode** — green Matrix-like environment, zombie robots (no shooting, melee only), Windows XP error popups on kill, glitch VFX
- **Night Mode** — total darkness, flashlight on camera, red enemy visors, N key for night vision (green overlay + boosted lighting)
- **Matrix Mode** — green-tinted world, enemy bullets become visible projectiles, auto-slomo when bullet < 3m from player

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

Gun is built along Y axis (barrel = -Y) so it points forward when arm rotates. Both arms maintain two-handed grip in all states. In error mode: gun hidden, arms posed as zombie.

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
updateTracers(delta) → updateMatrixBullets(delta) → updateWeather(delta) →
ragdoll sync → FPS counter → renderer.render()
```

### Wound/Kill Mechanic

- **1st body shot** → enemy enters `wounded` state (30% chance: surrender instead)
- **2nd body shot** → kill (regardless of HP)
- **Headshot** → always instant kill
- **Knife (V)** → kills in melee range, backstab dismembers limbs (no head)
- **Katana (X)** → dash + always decapitate + kill
- Hit detection: `raycaster.intersectObjects([enemy.group], true)`, check if hit object `=== enemy.head`

## Key Patterns to Follow

- **All state is global** — no modules, no classes. Variables declared at top of script block.
- **Cleanup in `resetGame()`** — any new system with scene objects or physics bodies MUST be cleaned up here. Also reset mode flags.
- **New globals go near line 600-640** — alongside existing mode/feature declarations.
- **New update functions** go inside the `STATE.PLAYING` block in `update()`.
- **Sounds are synthesized** — use `audioCtx.createBuffer()` + `createBufferSource()` pattern. See `playGunshot()` or `playFootstep()` as examples. Always wrap in try/catch and check `audioCtx` exists.
- **Enemy types** are defined in `enemyTypes` object. Type-specific visuals go in the `createEnemy()` type adjustments block.
- **Wave system**: `startWave(n)` spawns enemies, `killEnemy()` decrements `waveEnemiesLeft` and checks completion.
- **Mode-specific enemy materials**: branch in `createEnemy()` on `snowMode`/`errorMode`/`nightMode`/`matrixMode` flags.
- **Melee weapon visuals**: create mesh group on weaponGroup, hide gun children, animate, then dispose meshes and restore gun. Always dispose geometries and materials.
- **Music**: `startCyberMusic()` on game start, `stopCyberMusic()` on death/victory/menu. Uses setInterval-based sequencing.
- **Weather**: toggled via Settings panel checkbox, not keyboard. Uses `startRain()`/`stopRain()` and `updateWeather(delta)` in game loop.

## Common Pitfalls

- **Don't use `element.textContent = ...` on containers with child elements** — it destroys child DOM nodes. Update specific child spans instead.
- **Cannon.js 0.6.2 has no `ConeTwistConstraint`** — only `PointToPointConstraint` is available for ragdoll joints.
- **`window._slowMoFactor`** — always reference the global, not a local copy. Reset to 1 after temporary slomo effects.
- **Duplicate declarations** — since everything is in one script block, verify a variable name isn't already taken before declaring it.
- **Mode cleanup** — all mode flags must be reset to `false` in `resetGame()`. Night vision overlay must be hidden. Weather must be stopped. Music must be stopped.
- **Dispose Three.js resources** — when creating temporary meshes (knife, katana, VFX), always `.dispose()` geometries and materials on cleanup to prevent memory leaks.
- **Syntax check**: `node -e "const fs=require('fs');const c=fs.readFileSync('index.html','utf8');const m=c.match(/<script>([\s\S]*)<\/script>/);try{new Function(m[1]);console.log('OK')}catch(e){console.log(e.message)}"` — run this after edits.
