# Plan: Top-Down Shooter Game (browser, single HTML file)

## Context
Building a retro-style 2D top-down shooter in the browser as a single self-contained HTML file (`game.html`), consistent with the existing `tictactoe.html` pattern in this project directory. No build tools, no external dependencies. All sprites drawn programmatically via Canvas 2D API (pixel art with `fillRect`).

---

## Output File
`/Users/cheewaichong/Work/Py/cc/game.html`  
Reference for theme/style: `/Users/cheewaichong/Work/Py/cc/tictactoe.html`

---

## Architecture: Sections in `game.html`

All JS in one `<script>` block, divided by comment dividers:

```
SECTION 1:  CONSTANTS & CONFIG
SECTION 2:  COLOR PALETTE
SECTION 3:  SPRITE DEFINITIONS (pixel art 2D arrays + drawSprite helper)
SECTION 4:  INPUT MANAGER
SECTION 5:  PARTICLE SYSTEM
SECTION 6:  BULLET CLASS
SECTION 7:  ENEMY CLASSES (Enemy base + GruntEnemy, BruteEnemy, ScoutEnemy)
SECTION 8:  PLAYER CLASS
SECTION 9:  WAVE MANAGER
SECTION 10: LEVEL DATA
SECTION 11: GAME STATE MACHINE (setState, startLevel, update, render)
SECTION 12: SCREEN RENDERERS (menu, hud, levelComplete, gameOver)
SECTION 13: MAIN LOOP (init, gameLoop)
SECTION 14: BOOTSTRAP
```

Canvas: 800×600. CSS: `cursor:none` on canvas (custom crosshair drawn in-canvas).

---

## Game States

```
MENU → PLAYING → LEVEL_COMPLETE → PLAYING (next level)
                               ↘ GAME_OVER (last level or player dies)
```

`startLevel(n)` resets enemies/bullets/particles, creates/heals player, creates WaveManager.  
Between levels: player heals +30 HP (not full heal).

---

## Classes

### Player
- Position: starts at canvas center
- HP: 100, speed: 2.8px/frame
- Movement: ArrowUp/Down/Left/Right (normalize diagonals)
- Aim: angle = `atan2(mouseY - y, mouseX - x)` each frame
- Shoot: on mouseDown + cooldown=0 → spawn Bullet at gun tip, muzzle flash particles, reset cooldown (12 frames)
- Invincible for 45 frames after hit
- Sprite: 16×16, scale=2, drawn facing "up" then rotated to aim angle

### Enemy (base) + 3 subtypes
All move straight toward player each frame (Scout adds sinusoidal orbit wobble).

| Type  | Speed | HP | DMG | Score | Size  |
|-------|-------|----|-----|-------|-------|
| Grunt | 1.2   | 30 | 8   | 10    | 16×16 |
| Brute | 0.7   | 80 | 20  | 25    | 24×24 |
| Scout | 2.2   | 15 | 5   | 15    | 12×12 |

`contactCooldown=30` prevents per-frame damage. Death animation plays before removal (`isFullyDead` getter: `dead && animFrame >= DEATH_ANIM_FRAMES-1`).

### Bullet
- Speed: 9px/frame, lifetime: 90 frames, radius: 3
- Spawned at player gun tip, travels toward mouse-click angle

### ParticleSystem (singleton object)
- `spawnMuzzleFlash`, `spawnEnemyDeath`, `spawnHitSpark`, `spawnBloodTrail`
- Cap at 200 particles; each has color, size, lifetime, friction (vx/vy × 0.92/frame)

### WaveManager
- Flattens wave enemy list into a round-robin spawn queue
- Spawns from random screen edge (30px inset from corners)
- Advances wave only when: spawnQueue empty AND all enemies fully dead
- `done=true` after last wave completes

---

## Sprite System

`drawSprite(ctx, frameData, palette, cx, cy, scale, angle)`:
- Each frame = 2D array of palette indices (0 = transparent)
- Translates to (cx,cy), rotates by `angle + Math.PI/2` (sprites face "up")
- `fillRect` for each non-zero cell

Animations: array of frames per state (idle/walk/shoot/die). `animFrame` advances every 6 game ticks.

**Palette (matches tictactoe.html theme):**
- BG: `#1a1a2e`, Panel: `#16213e`, Border: `#0f3460`
- Accent red: `#e94560`, Cyan: `#a8dadc`, White: `#fff`, Yellow: `#ffe066`
- Grunt: `#4a7c59` / `#2d4a35`, Brute: `#7a3b3b` / `#4a1f1f`, Scout: `#5a4a8a` / `#2d2050`

---

## Level Data (5 levels)

| Level | Title         | Waves | Enemy types            |
|-------|---------------|-------|------------------------|
| 1     | SECTOR 1      | 3     | Grunt only             |
| 2     | SECTOR 2      | 4     | Grunt + Scout          |
| 3     | SECTOR 3      | 4     | Grunt + Scout + Brute  |
| 4     | SECTOR 4      | 5     | All types, fast spawns |
| 5     | SECTOR 5 FINAL| 5     | All types, dense       |

spawnInterval decreases each level (90→25 frames between spawns).

---

## Rendering Pipeline (back to front each frame)

1. Background: `#1a1a2e` fill + subtle `#0f3460` grid lines (40px, 30% alpha)
2. Floor debris: 40 static small rects (pre-generated at init)
3. Enemy shadows (dark ellipses, 30% alpha)
4. Enemies (walk or death animation)
5. Player shadow
6. Player
7. Bullets (cyan rect + white trailing rect for motion blur)
8. Particles
9. HUD: health bar (bottom-left), score (top-right), level+wave (top-left), wave progress bar (top-center), crosshair at mouse
10. Screen overlay (menu / level-complete / game-over) when not PLAYING

---

## Key Constants

```javascript
CANVAS_W=800, CANVAS_H=600
PLAYER_SPEED=2.8, PLAYER_MAX_HP=100
BULLET_SPEED=9, BULLET_DAMAGE=10, SHOOT_COOLDOWN=12
ANIM_SPEED=6 (frames per cel)
All sprites at scale=2 (so 16×16 → 32×32 canvas px)
```

---

## Collision Detection

Circle-based (radius comparisons), O(bullets × enemies) per frame:
- Bullet hits enemy: `dist < bullet.radius + enemy.radius`
- Enemy contacts player: `dist < enemy.radius + 12` (player radius)

---

## Screens

- **Menu**: floating title (sinusoidal Y oscillation), blinking "CLICK TO PLAY", instructions panel, retro scanline effect on title text
- **Level Complete**: "SECTOR CLEAR", wave bonus score, "+30 HP" heal note, "PRESS ANY KEY"
- **Game Over**: "GAME OVER" or "MISSION COMPLETE" (if level 5 cleared), final score, enemies defeated, "CLICK TO RETURN TO MENU"

---

## Verification

1. Open `game.html` in browser — menu screen appears, title floats
2. Click Play → Level 1 Sector 1, enemies spawn from edges, move toward player
3. Arrow keys move player, mouse aims, click shoots (cyan bullets travel to cursor)
4. Enemies die with particle burst; player takes damage on contact
5. Complete all 3 waves → Level Complete screen → continue to Level 2
6. Die → Game Over screen → click returns to menu
7. Complete Level 5 → "MISSION COMPLETE" variant of game over
8. No console errors; smooth 60fps throughout
