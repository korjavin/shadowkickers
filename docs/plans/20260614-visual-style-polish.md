# Visual Style Polish — Parallax, Terrain Shading, Shadows, Rounded Corners, Juice

## Overview
Improve the visual feel of the game without changing gameplay or the existing bright-cartoon palette. Today everything in the world is drawn as flat `ctx.fillRect` rectangles over a static CSS sky gradient, giving zero depth and a blocky look. This plan adds five localized upgrades, all confined to the rendering layer:

1. **Parallax background** — scrolling depth layers (hills/clouds) instead of a fixed sky.
2. **Platform shading** — gradient terrain with a grass top edge and a bottom bevel.
3. **Contact shadows + active-character glow** — grounds entities and makes the controlled character obvious.
4. **Rounded corners** — softens platforms, enemies, pickups, and HUD bars.
5. **Juice** — a generic particle system (landing dust + impact bursts) and screen shake on stomps/dashes/shots.

Benefits: dramatically more depth and "game feel" for a small, self-contained change set. No architecture change, no new dependencies, no gameplay/physics changes.

## Context (from discovery)
- **Single file**: `index.html` (~1950 lines). One `Game` class owns the loop, physics, and rendering. Only external dep is Tailwind via CDN (UI chrome only).
- **Render pipeline** (`render()` ~L1526): `clearRect` → `ctx.save()` → `ctx.translate(-camera.x, -camera.y)` → draw platforms / objects / enemies / projectiles / particles / active character → `ctx.restore()`. Canvas is otherwise transparent, so the CSS gradient `linear-gradient(180deg,#87CEEB,#98FB98)` (`index.html:17`) shows as the sky.
- **Existing particle pattern to mirror**: `windParticles` and `bladeEffects` are lazily-initialized arrays, pushed to inside ability code, advanced by `updateWindParticles()` (L1399) / `updateBladeEffects()` (L1414), and drawn in `render()` with `globalAlpha`. New particles/dust follow the same shape.
- **Camera/loop**: `this.camera = {x,y}` (L305); `gameLoop()` (L1902) calls `update()` then `render()` each frame; `updateCamera()` (L1430) centers camera on the active character.
- **Hook points already located**:
  - Landing: in `checkCollisions()`, `char.onGround = true` is set at L1266 (platforms) / L1300 (objects); the downward-collision branch (`char.vy > 0 && char.y < platform.y`, L1263) is where impact velocity is still available before `char.vy = 0`.
  - Enemy kills: stomp L1353 (`enemy.alive=false; char.vy=-8`); **dash** L1187 inside `killEnemiesInDashPath` — note this runs every frame while `primm.dashing > 0` (re-invoked at L1449 for ~15 frames), so spawn the burst right at the `enemy.alive=false` line inside the existing `if (enemy.alive)` guard to fire once per enemy; **projectile/shot** L1379 inside `updateProjectiles`.
  - Coin collect: `obj.collected = true` at L1293.
- **No test runner**: there is no JS test framework, bundler, or build step. See Development Approach.

## Development Approach
- **Testing approach: manual, in-browser (adapted).** This repo has no JS test runner and adding one for canvas rendering is out of scope. "Tests" in this plan means a concrete **manual verification step** per task: open `index.html` in a browser, perform the listed interaction, and confirm (a) the intended visual change is present and (b) the DevTools console is free of new errors. Where practical, prefer changes that are also checkable on the procedurally-generated Level 2.
- Complete each task fully before moving to the next; make small, focused changes.
- **Preserve gameplay**: no change to physics constants, collision logic, hitboxes, controls, or level layout. Visual-only.
- **Preserve palette**: keep the existing bright sky + character colors; derive new shades (highlights/shadows) from existing colors rather than introducing a new palette.
- **Performance**: this is a per-frame canvas loop. New per-frame work must stay cheap — cap particle counts, tile parallax with simple math (no allocations per frame), and avoid `shadowBlur` on large fills inside tight loops (reset `shadowBlur`/`globalAlpha` after use so state doesn't leak).
- Keep new draw code factored into small helper methods on `Game` so `render()` stays readable.

## Testing Strategy
- **Per-task manual verification** (required, listed as the last checkbox of each task): load the game, exercise the relevant interaction, confirm visual + clean console.
- **Cross-cutting checks** before completion: run through Level 1 fully and re-roll Level 2 two or three times (it is randomized) to confirm nothing regressed; confirm all four characters still render and play correctly; confirm framerate feels smooth (no obvious stutter from particles/parallax).
- No automated/unit/e2e tests are introduced (no harness exists; out of scope).

## Progress Tracking
- Mark completed items `[x]` immediately when done.
- Add newly discovered tasks with ➕ prefix; document blockers with ⚠️ prefix.
- Update this plan if implementation deviates from scope.

## Solution Overview
All work lives in the rendering layer of the single `Game` class. New helper methods are added and called from `render()` / `update()`; a few existing hook points in `checkCollisions()` and the kill/collect sites get one-line calls to spawn particles or shake. No data model, level, or physics change. New per-frame state added to the constructor: `this.particles = []` and `this.shake = {x:0, y:0, mag:0}`.

Draw order inside `render()` becomes:
`background (parallax) → [camera translate] → platforms → objects → enemies (each w/ contact shadow) → projectiles → wind particles → blade fx → generic particles → active character (w/ contact shadow + glow) → fuel bar`. There is no discrete contact-shadow pass — enemy shadows are drawn inline in the enemy loop and the active-character shadow inline in the character loop; generic particles are drawn after the wind/blade fx, not before.

## Technical Details
- **Parallax**: drawn in screen space (before the camera translate) so layers can move slower than the world. Each layer's on-screen x = `-(camera.x * factor) % tileWidth`, repeated across the viewport. Factors ~0.2 (far hills), ~0.4 (clouds), ~0.6 (near hills/bushes). Sky gradient drawn on-canvas via `createLinearGradient` so it is self-contained (CSS gradient remains as a harmless fallback).
- **Platform shading** (`drawPlatform(p)`): vertical `createLinearGradient` body (top slightly lighter → bottom darker, derived from `p.color`), a 4–6px lighter "grass/edge" strip along the top, and a 2px darker line along the bottom edge for a faux bevel.
- **Shadows/glow**: contact shadow = a low-alpha black ellipse (`ctx.ellipse`) at each entity's feet, width ~entity width. Active-character glow = a short `shadowBlur` + colored `shadowColor` around the character (reset immediately after), or a thin outline if blur proves costly.
- **Rounded corners** (`roundedRect(x,y,w,h,r)` helper): wraps `ctx.roundRect` when available, falls back to `fillRect`. Applied to platforms, enemies, coins/goal, and HUD bars — not to the dozens of tiny sub-rects inside `drawCharacter` (rounding those is visual noise for the cost).
- **Particles** (`spawnParticles(x,y,opts)`, `updateParticles()`): each particle `{x,y,vx,vy,life,maxLife,size,color,gravity}`; `updateParticles()` integrates velocity + optional gravity and decays `life`, removing dead ones; drawn with `globalAlpha = life/maxLife`. Dust = a few tan particles on landing when impact `vy` exceeds a threshold. Bursts = a small spray in the entity color on kills, gold sparkles on coin pickup. Total particles capped (e.g. ignore new spawns above ~200 live) to bound per-frame cost.
- **Screen shake** (`addShake(mag)`): sets `this.shake.mag`; `update()` decays it toward 0; `render()` offsets **only the world `ctx.translate`** by a small `mag`-scaled vector — e.g. `translate(-(camera.x - shake.x), -(camera.y - shake.y))`. Do **not** write the offset back into `this.camera`: `updateCamera()` (L1430–1437) overwrites `camera.x/y` every frame and clamps them (`max(0,…)`, y∈[-200,200]), which would erase the shake and fight it at level edges. The parallax background (Task 1) must read the **unshaken** `camera.x` so the sky doesn't jitter with the world (preserving the depth cue). `Math.random()` is fine here (runtime/visual only, not persisted). Magnitudes kept small (stomp < dash) to avoid nausea.

## What Goes Where
- **Implementation Steps** (`[ ]`): all rendering/particle code changes in `index.html`, plus the docs update — all achievable in this repo.
- **Post-Completion** (no checkboxes): subjective "does it feel good" tuning and any cross-browser eyeballing the user wants to do.

## Implementation Steps

### Task 1: Parallax scrolling background

**Files:**
- Modify: `index.html` (add `drawBackground()`; call it at the top of `render()`)

- [x] add `drawBackground()` method on `Game` that fills the viewport with an on-canvas vertical sky gradient (matching the current `#87CEEB → #98FB98`)
- [x] draw 2–3 tiled parallax layers (far hills, clouds, near hills) positioned by `-(this.camera.x * factor)` (use the **unshaken** `camera.x` — Task 6 shake must not affect the background) with modulo wrap so they repeat across the screen
- [x] call `drawBackground()` at the start of `render()` **before** `ctx.save()/translate` (screen space), so layers scroll slower than the world
- [x] keep shapes cheap (simple rects/`arc` hills, no per-frame allocations); reset any `globalAlpha` used
- [x] manual verify (browser-only, not automatable in agent) — move left/right across Level 1 and a fresh Level 2; background shows clear depth, layers scroll at different speeds, no seams/flicker, console clean

### Task 2: Platform terrain shading

**Files:**
- Modify: `index.html` (add `drawPlatform()`; replace the platform draw loop in `render()`)

- [x] add `drawPlatform(p)` helper: vertical `createLinearGradient` body shaded from `p.color` (lighter top → darker bottom)
- [x] add a lighter "grass/edge" highlight strip along the platform top (a few px tall)
- [x] add a 2px darker line along the bottom edge for a faux-bevel
- [x] replace the flat `fillRect` platform loop in `render()` (~L1534) to call `drawPlatform(p)`
- [x] manual verify (browser-only, not automatable in agent) — platforms read as shaded terrain (top edge + depth) on both levels; phase walls (which are `objects`, not `platforms` — so `drawPlatform` does not touch them) remain visually distinct; console clean

### Task 3: Contact shadows + active-character glow

**Files:**
- Modify: `index.html` (`render()` — add shadow passes; wrap `drawCharacter` call with glow)

- [x] draw a low-alpha black ellipse (`ctx.ellipse`) at the active character's feet before `drawCharacter()`
- [x] draw the same contact shadow under each living enemy in the enemy loop
- [x] add a glow around the active character (short `shadowBlur` + character-colored `shadowColor`, reset immediately after) — fall back to a thin outline if blur is costly
- [x] reset `shadowBlur`/`shadowColor`/`globalAlpha` immediately after `drawCharacter()` — the flight-fuel bar (L1621–1629) draws right after, still inside the same `save/restore`, and will inherit a stray glow otherwise
- [x] manual verify (browser-only, not automatable in agent): character and enemies feel grounded; the controlled character clearly stands out after switching with keys 1–4; the flight-fuel bar shows **no** stray glow/blur; console clean

### Task 4: Rounded corners

**Files:**
- Modify: `index.html` (add `roundedRect()` helper; apply at chosen draw sites)

- [x] add `roundedRect(x,y,w,h,r)` helper that uses `ctx.roundRect` when available and falls back to `fillRect`
- [x] apply rounding to platforms (in `drawPlatform`) and enemies — note this converts `drawPlatform`'s body from `fillRect` to a path-based fill (`beginPath` → `roundRect` → `fill`) so the Task 2 gradient fills the rounded path; the grass-top strip and bottom bevel must also respect the rounded corners (clip or inset) so they don't overhang
- [x] apply rounding to coins and the goal object
- [x] apply rounding to the **flight-fuel bar** (L1621–1629) — the only canvas-drawn HUD; the hearts display is a DOM `#heartsDisplay` div (L110–111) already styled `rounded` and is **not** drawn on canvas, so leave it alone
- [x] leave the many small sub-rects inside `drawCharacter` as square (rounding them is noise for the cost)
- [x] manual verify (browser-only, not automatable in agent): rounded shapes render with no clipping artifacts or gaps; the Task 2 gradient + grass strip + bevel still align inside rounded platforms (no overhang past corners); console clean

### Task 5: Particle system + landing dust

**Files:**
- Modify: `index.html` (constructor: `this.particles=[]`; add `spawnParticles()`/`updateParticles()`; hook landing in `checkCollisions()`; draw in `render()`)

- [x] initialize `this.particles = []` in the constructor (near other state ~L305–312)
- [x] add `spawnParticles(x, y, opts)` and `updateParticles()` (velocity + optional gravity integration, `life` decay, removal of dead particles, hard cap on live count)
- [x] call `updateParticles()` from `update()` (alongside `updateWindParticles()` ~L1236) and draw particles in `render()` **inside the camera translate** (with the wind/blade fx, before `ctx.restore()` at L1631) — particles carry world coords from `checkCollisions`, so drawing them after restore would misplace them in screen space; use `globalAlpha = life/maxLife` and reset alpha after
- [x] spawn a small tan dust puff on landing: capture impact `vy` in the downward-collision branch (~L1263) before it is zeroed, and only puff above a threshold
- [x] manual verify (browser-only, not automatable in agent): landing from a height kicks up dust at the feet; gentle steps do not; no particle buildup/leak over time; framerate smooth; console clean

### Task 6: Impact bursts + screen shake

**Files:**
- Modify: `index.html` (constructor: `this.shake`; add `addShake()`; decay in `update()`; offset in `render()`; hook kill/collect sites)

- [x] add `this.shake = {x:0,y:0,mag:0}` and `addShake(mag)`; decay `mag` toward 0 in `update()`; in `render()` apply it as an offset on the world `ctx.translate` only — `translate(-(camera.x - shake.x), -(camera.y - shake.y))` — and **never** write it back into `this.camera` (`updateCamera()` overwrites + clamps that every frame); leave the Task 1 parallax reading the unshaken `camera.x`
- [x] spawn an entity-colored burst + `addShake` on enemy kills: stomp (~L1353), **projectile/shot** (~L1379, in `updateProjectiles`), **dash** (~L1187, in `killEnemiesInDashPath`), with dash shake > stomp shake
- [x] gate the dash burst/shake to fire **once per enemy**: place the spawn at the `enemy.alive = false` line inside the existing `if (enemy.alive)` guard, since `killEnemiesInDashPath` re-runs every frame for ~15 frames (L1449) — once the enemy is dead the guard skips it, so no repeat burst
- [x] spawn a small gold sparkle burst on coin collect (~L1293)
- [x] tune magnitudes so shake reads as impact without nausea (keep small; no shake on routine actions)
- [x] manual verify (browser-only, not automatable in agent): stomping, shooting, and a Primm **dash** through an enemy each produce a burst + brief shake; the dash fires the burst exactly **once per enemy** (not every frame of the 15-frame dash); collecting coins sparkles; shake settles quickly, never affects the parallax background, and never makes play uncomfortable; console clean

### Task 7: Verify acceptance criteria
- [x] all five upgrades visibly present together (parallax, shaded platforms, shadows+glow, rounded corners, dust/bursts/shake) and palette unchanged — (static review — browser playthrough not automatable in agent): (1) Parallax: `drawBackground()` (index.html:1667) called at top of `render()` (1844) BEFORE `ctx.save()`/`translate` (1847-1851); reads unshaken `this.camera.x` (1681). (2) Platform shading: `drawPlatform(p)` (1779) is the only platform draw loop (1854); no leftover flat `fillRect` loop (L976/L1270 are bounds/collision). (3) Shadows+glow: `drawContactShadow` for active char (1958) and living enemies (1894); active-char glow via `shadowBlur=16` (1962-1966). (4) Rounded corners: `roundedRect` (1763) with `roundRect` feature-detect + `fillRect` fallback, applied to platforms (1798), enemies (1897), coins (1865), goal (1867), fuel bar (1981/1983); hearts DOM div untouched. (5) Juice: `this.particles=[]`/`this.shake` (313-314), spawn/update/addShake/updateShake all present, bursts at stomp(1411)/projectile(1448)/dash(1194)/coin(1327). Palette preserved (shades derived via `shadeColor`, sky `#87CEEB`/`#98FB98` matches CSS).
- [x] full Level 1 playthrough with all four characters; re-roll Level 2 two or three times — no gameplay/collision/physics regressions — (static review — browser playthrough not automatable in agent): physics constants unchanged (gravity 0.8 index.html:318, friction 0.8 :319); collision/landing/kill logic untouched except for visual-only one-line spawn/shake calls at documented hook points (impact `vy` read before zeroing, dash gated once-per-enemy inside `if(enemy.alive)` guard at the `enemy.alive=false` line 1189). Level layout and `initLevel`/character logic unchanged. Multi-character/Level-2 re-roll behavior requires a browser and was NOT run.
- [x] no new console errors/warnings; no particle or canvas-state leaks over an extended session — (static review — browser playthrough not automatable in agent): save/restore balanced (4 real saves @1802/1831/1847/1962 ↔ 4 restores @1819/1837/1986/1966; the "restore" at 1944 is a comment). Every `globalAlpha` set has a matching reset (1699→1711, 1832 inside save/restore, 1915→1919, 1925→1940, 1947→1951, 1998→2201); `shadowBlur`/`shadowColor` reset to 0/transparent at 1972-1973 BEFORE the fuel-bar block (1977-1984) so the bar inherits no glow. Single `ctx.clip()` (1806) is inside an unconditional save(1802)/restore(1819) pair. Particles removed when `life<=0` (updateParticles filter, 1531-1537) — no unbounded growth. No new console.* logging added by the visual code. Extended-session runtime observation requires a browser and was NOT run.
- [x] performance feels smooth (no stutter from particles/parallax/shadows) — (static review — browser playthrough not automatable in agent): particle hard cap CAP=200 enforced in `spawnParticles` (1497/1509). Parallax uses cheap modulo math + `arc` shapes; per-frame allocations limited to gradients/paths (the documented approach), no per-frame arrays. Glow `shadowBlur` applied only to the single active character (not in loops). Subjective framerate "feel" requires a browser and was NOT run. `node --check` on the extracted main <script> (index.html:133-2302) → PARSE OK (exit 0).

### Task 8: Update documentation & close out
- [x] update `CLAUDE.md` rendering/architecture notes to mention the parallax background, platform shading helpers, the particle system, and screen-shake state (so future instances know the new draw order and helpers)
- [x] move this plan to `docs/plans/completed/` (create the dir if needed) — (moved by orchestrator at completion; left in place so remaining review/finalize/stats phases can still read it)

## Post-Completion
*Informational — no checkboxes.*

**Manual verification / tuning**
- Subjective "game feel" tuning of shake magnitudes, particle counts/lifetimes, and parallax factors is best done by eye in the browser; values in this plan are starting points.
- Optional cross-browser eyeballing (Chrome/Firefox/Edge) — `ctx.roundRect` and `ctx.ellipse` are widely supported in modern browsers; the `roundedRect` fallback covers older ones.

**Possible follow-ups (out of scope here)**
- Per-level palette / a moodier "stealth" art direction (deliberately deferred — this plan keeps the current cartoon palette).
- Real sprite art via `drawImage` (would replace the rect-stack `drawCharacter`).
