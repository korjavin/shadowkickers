# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

"Crime Kickers vs The Bad Guys" is a single-file, browser-based 2D stealth-puzzle platformer inspired by real-time tactics games (Shadow Tactics, Commandos). The core mechanic is switching between four characters with distinct abilities to solve environmental puzzles that no single character can pass alone.

## Architecture

### Single-File Game
The entire game — HTML, CSS, and vanilla JavaScript — lives in **`index.html`** (~1950 lines). All rendering is done on an HTML `<canvas>`. The only external dependency is **Tailwind CSS loaded via CDN** (`https://cdn.tailwindcss.com`) for the UI chrome (menus, character selector, message boxes). There is no build step, bundler, or framework.

> Note: the `README.md` and `agent.md` still refer to the original filename `the_unlikely_squad.html`. The file was renamed to `index.html` (so it serves as an nginx index — see Deployment). Treat `index.html` as the source of truth.

`agent.md` is the original task spec used to generate the game; it documents the intended Level 1 design and character specs but is not authoritative for current behavior.

### Core Engine (`Game` class)
A single `Game` class owns the loop, physics, rendering, and collision. State lives in arrays on the instance: `platforms`, `enemies`, `projectiles`, `objects`, plus particle/effect arrays. Key methods:

- `initCharacters()` — builds the four character objects (each with `speed: 5`, `jumpPower: 15`, ability flags).
- `initLevel(levelNumber)` — dispatches to `initLevel1()` or `generateLevel2()`.
- `switchCharacter(index)` — swaps the active character; the new character inherits the old one's position (teleport mechanic). Only one character is active at a time.
- `useAbility()` — `switch(this.currentCharacter)` to run the active character's primary ability.
- `update()` → `checkCollisions()`, `updateEnemies()`, `updateProjectiles()`, `updateObjects()`, `updateCamera()`, `updateCooldowns()`, `checkWinCondition()`, `checkFallDeath()`.
- `render()` / `drawCharacter()` — `drawCharacter` uses `switch(index)` to draw each character's distinct look.

Global physics: `gravity = 0.8`, `friction = 0.8`.

### Level System (5 levels: one hand-authored + four procedural)
There are **5 levels** (`maxLevel = 5`), selectable on the start screen and chained by `nextLevel()` on completion. `initLevel(levelNumber)` dispatches: level 1 → `initLevel1()`, levels 2–5 → `generateLevel(levelNumber)`.
- **Level 1 — hand-authored** (`initLevel1()`): platforms, enemies, and puzzle objects are pushed in by hardcoded coordinates. Linear left-to-right progression that gates on each character's ability in sequence (Teibi crawl gap → Mister Underpants flight chasm → Primm phase wall → Windman lift platform → goal). Serves as the guided tutorial.
- **Levels 2–5 — procedurally generated** (`generateLevel(level)`): `buildSectionList(level)` produces the section list — it always includes all four character sections (`crawl` / `fly` / `phase` / `wind`) so every character stays required, then pads with random extra sections (counts: L2=4, L3=5, L4=6, L5=8) and `shuffleArray()`s the result. Each section is built by `generateSection(..., level)`, advancing a `currentX` cursor. **Complexity scales with `level`** via a `diff = level - 2` factor: wider fly gaps, higher wind climbs, an extra phase wall at L4+, and more/faster enemies (`generateEnemies(level)`). `generateCoins()` and `generateEnemies()` spread their objects across the actual generated level width. When editing the procedural levels, work through `generateSection()`'s `switch(section.type)` and the per-section `diff` scaling rather than fixed coordinates.

### Characters & Abilities
1. **Mister Underpants** (red): glide/flight consuming regenerating `flightFuel` (held Up in air); Spacebar shoots a projectile.
2. **Windman** (blue): Spacebar raises wind-receptive lift platforms; wind particles are visual feedback.
3. **Teibi** (green): Spacebar toggles shrink — small form fits through narrow gaps.
4. **Primm** (purple): Spacebar dashes (Katana Strike, instant enemy kill); **hold C** to phase through "phase walls" (solid for everyone else).

### Difficulty
Selected on the start screen (`startGame(difficulty, level)`): **Easy** (3 hearts, respawn), **Normal** (no extra lives), **Hardcore** (more enemies). Hearts UI is hidden outside Easy mode.

### Audio
Two background tracks (`Verse 1.mp3`, `Verse 1 (2).mp3`) driven by `playMusic()`, with rotation/mute controls. Browser autoplay policy means audio starts only after user interaction.

### Rendering Layer (visual polish)
The rendering layer in `index.html` adds depth and "juice" on top of the flat-rect drawing. All of this is visual-only — no physics, hitboxes, or level data are affected.

- **`drawBackground()`**: On-canvas sky gradient plus 3 tiled parallax layers (factors 0.2 / 0.4 / 0.6). Called at the **top of `render()` before the camera translate** (screen space), and reads the **unshaken** `camera.x` so the sky doesn't jitter with screen shake.
- **`drawPlatform(p)`** + color helpers **`parseHexColor()`** / **`shadeColor()`**: Renders each platform as a gradient terrain body with a grass-top strip and a bottom bevel. The platform loop calls this instead of a flat `fillRect`. For a non-hex `p.color` (where `parseHexColor` returns `null`) it falls back to the original flat `fillRect`.
- **`roundedRect(x, y, w, h, r)`**: `roundRect`-with-`fillRect`-fallback helper used for platforms, enemies, coins, the goal, and the flight-fuel bar.
- **`drawContactShadow(ent)`**: Ground-contact ellipse drawn under the active character and each living enemy. The active character also gets a `shadowBlur` glow, which is reset before the fuel-bar is drawn (so the HUD bar inherits no stray glow).
- **Particle system**: `this.particles = []` with **`spawnParticles(x, y, opts)`** / **`updateParticles()`** (hard cap ~200 live particles). Drawn **inside the camera translate** (particles carry world coords). Used for landing dust, entity-colored impact bursts on kills, and gold coin-collect sparkles.
- **Screen shake**: `this.shake` with **`addShake(mag)`** / **`updateShake()`**. Applied **only** as an offset on the world `ctx.translate` (never written into `this.camera`, which `updateCamera()` overwrites/clamps every frame). Magnitudes scale stomp < projectile < dash.

**Draw order in `render()`**: background (parallax) → `[camera translate]` → platforms → objects → enemies (each with its contact shadow) → projectiles → wind particles → blade fx → generic particles → active character (with contact shadow + glow) → fuel bar. There is no separate "contact shadows" pass: enemy shadows are drawn inline in the enemy loop and the active-character shadow inline in the character loop (gated on `onGround`). Generic particles are drawn after the wind/blade fx, not before.

## Running the Game

This is a static site with no build system.

- **Local dev**: open `index.html` directly in a modern browser (Chrome/Firefox/Edge). Edit the file and refresh. Debug via the browser console.
- **Container**: `docker compose up --build` serves the static files via nginx on **port 8080** (the Dockerfile runs nginx as a non-root user on a non-privileged port, copying `index.html` and both `.mp3` files into the image).

## Deployment (CI/CD)

Push to `main` triggers `.github/workflows/deploy.yml`:
1. Builds the Docker image and pushes it to `ghcr.io/<owner>/<repo>:<sha>`.
2. Force-pushes a `deploy` branch whose `docker-compose.yml` has the image tag rewritten to the new SHA (via `sed`).
3. Calls the Portainer redeploy webhook (`secrets.PORTAINER_REDEPLOY_HOOK`) to pull and restart.

`docker-compose.yml` carries Traefik labels (TLS via `myresolver`, host from `${HOSTNAME}`, backend port 8080) and attaches to an external `traefik_network`. The `deploy` branch is machine-generated — do not edit it by hand; change `docker-compose.yml` on `main` instead.

## Testing

No automated tests. Verify manually in-browser:
1. All four abilities work (shoot/fly, wind lift, shrink, dash/phase).
2. Each level can be completed and genuinely requires every character — re-run the procedural levels (2–5) a few times each since they are randomized, and confirm difficulty actually ramps from L2 to L5.
3. Enemy interactions: stomp/dash/projectile kills vs. side/below damage; wind stun.
4. Win (reach goal) and lose (fall death / out of hearts) conditions, across all three difficulties.
