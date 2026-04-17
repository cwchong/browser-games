# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the games

No build step. Open any HTML file directly in a browser:

```bash
open game.html
open tictactoe.html
```

There are no dependencies, no package manager, no server required.

## Repository layout

| File | Description |
|------|-------------|
| `game.html` | Top-down shooter — RECON: Zero Day (~1160 lines) |
| `tictactoe.html` | Tic-tac-toe with minimax AI (~253 lines) |

## game.html architecture

All code lives in a single `<script>` block divided by `// SECTION N:` comment markers. Sections must stay in order — later sections depend on earlier ones.

| Section | Lines | Purpose |
|---------|-------|---------|
| 1 | 16 | Constants (canvas size, speeds, HP, radii, cooldowns) |
| 2 | 31 | Color palette object `C` — single source of truth for all colors |
| 3 | 43–478 | Sprite data: 2D palette-index arrays + `drawSprite()` helper |
| 4 | 479 | `Input` singleton — keyboard state + mouse position/click |
| 5 | 505 | `Particle` class + `ParticleSystem` singleton |
| 6 | 565 | `Bullet` class |
| 7 | 592 | `Enemy` base class + `GruntEnemy`, `BruteEnemy`, `ScoutEnemy` |
| 8 | 671 | `Player` class |
| 9 | 725 | `WaveManager` class |
| 10 | 797 | `LEVEL_DATA` array — 5 levels, each with wave configs |
| 11 | 834 | Game state machine: `GS` object, `setState()`, `startLevel()`, `resetGame()` |
| 12 | 894 | Screen renderers: `renderMenu`, `renderHUD`, `renderLevelComplete`, `renderGameOver`, `renderBackground` |
| 13 | 1053 | `update(dt)` and `render(ctx)` — the per-frame logic |
| 14 | 1148 | Bootstrap: floor debris generation, `Input.init()`, `requestAnimationFrame` |

### Key design patterns

**Sprite rendering** — `drawSprite(ctx, frameData, palette, cx, cy, scale, angle)` draws a 2D array of palette indices using `fillRect`. Sprites are authored facing "up" (negative Y) and rotated via canvas transform with a `+Math.PI/2` correction. All sprites use `scale=2`, making 16×16 pixel art render at 32×32 canvas pixels.

**Game state** — One plain object `GS` holds all mutable state (`state`, `score`, `level`, `player`, `enemies`, `bullets`, `waveManager`). State transitions go through `setState(STATES.*)`. The `STATES` enum has four values: `MENU`, `PLAYING`, `LEVEL_COMPLETE`, `GAME_OVER`.

**Enemy lifecycle** — Enemies are never removed mid-frame. When HP reaches zero, `dead=true` triggers a 4-frame death animation. `isFullyDead` (getter: `dead && animFrame >= 3`) is what the main loop checks before splicing from `GS.enemies`. Score and kill count are tallied at removal time in `update()`.

**WaveManager** — Each level's enemy list is flattened into a round-robin `spawnQueue` (interleaved by type). A wave advances only when the queue is empty AND no non-`isFullyDead` enemies remain. `WaveManager.done` triggers level-complete or game-over transitions.

**Input** — `Input.mouseJustDown` / `Input.mouseJustUp` are single-frame flags; `Input.flush()` is called at the end of each `gameLoop` iteration. Mouse coordinates are CSS-scale-corrected using `getBoundingClientRect`.

**dt (delta time)** — The game loop passes `dt = elapsed / 16.667`, normalized to 60fps. All movement multiplies by `dt`. It is capped at 3× to prevent tunnelling on tab-switch.

### Adding a new enemy type

1. Define palette and sprite frames (`SPR_X_WALK`, `SPR_X_DIE`) in Section 3.
2. Create `class XEnemy extends Enemy` in Section 7 — override only the constructor to set stats and assign sprite arrays.
3. Add a `'type-key'` branch in `WaveManager._spawnEnemy()`.
4. Reference it by string key in `LEVEL_DATA` wave configs.

### Adding a new level

Append an object to `LEVEL_DATA` (Section 10). Each level needs `title` (string) and `waves` (array of `{ enemies: [{type, count}], spawnInterval, spawnDelay }`).

## Git / GitHub workflow

- Remote: `https://github.com/cwchong/browser-games`
- Default branch: `main`
- Commit directly to `main` for small changes; use feature branches for larger work
- No CI, no linting, no automated tests — verify by opening the HTML file in a browser
