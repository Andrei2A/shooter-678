# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Breach & Clear** — a single-file tactical FPS browser game. The entire game lives in `game.html` (~3300 lines of inline HTML/CSS/JS). No build tools, no package manager, no bundler.

## Running

Open `game.html` directly in a modern browser (Chrome recommended for best WebGL/Web Audio support). No server needed.

## Architecture

### Single-File Structure (`game.html`)

All code is inline in one HTML file. The order matters — classes are defined sequentially and reference each other:

1. **CSS** — UI overlay styles (crosshair, vignette, blood effects, film grain, menus)
2. **HTML** — Canvas element + UI overlay divs (menu screen, death screen, victory screen, HUD elements)
3. **JS** — Three.js imported via `<script type="importmap">` from unpkg CDN

### Class Dependency Chain

```
Game (master controller, game loop)
├── InputManager (keyboard/mouse, pointer lock)
├── Player (camera, WASD movement, collision, 2 HP)
├── Weapon (shotgun mesh on camera, shooting, reload, ADS)
├── Level (corridor + 10 apartments, procedural textures, lighting)
│   └── AABB (collision boxes for all walls)
├── Door[] (kick to open, collider removal)
├── EnemyManager
│   └── Enemy[] (AI state machine, tactical mesh, ragdoll physics)
│       └── Ragdoll (VerletParticle + DistConstraint)
├── SoundManager (all audio is procedural via Web Audio API)
└── UIManager (screen transitions, crosshair, blood vignette)
```

### Key Design Decisions

- **No HUD**: Health shown via blood vignette + heartbeat sound; ammo must be mentally tracked
- **Procedural everything**: Textures generated via canvas, audio synthesized via Web Audio API, no external assets except Three.js and Google Fonts
- **Verlet ragdoll physics**: Custom physics engine (not Cannon.js) — `VerletParticle` for points, `DistConstraint` for bones, `Ragdoll` for the full 12-particle skeleton
- **Enemy AI states**: IDLE → ALERT → ATTACKING/RUSHING → WOUNDED → DEAD, with sound-reactive triggers

### Building Layout

```
Z=0 (spawn)              Z=30 (end)
  | [0][2][4][6][8]  |   ← Left apartments (X negative)
  |   Corridor 3m    |
  | [1][3][5][7][9]  |   ← Right apartments (X positive)
```

Corridor is 3m wide × 30m long × 3m tall. Each apartment is 5×5m. Player spawns at Z=1 facing +Z.

### External Dependencies (CDN only)

- Three.js r160 via unpkg importmap
- Google Fonts: Space Grotesk

### Design Tokens (from mockups 1.html, 2.html, 3.html)

- Primary accent: `#ec7f13` (orange)
- Background dark: `#221910`
- Font: Space Grotesk
- Style: dark/gritty tactical with cinematic vignette and film grain

## Files

- `game.html` — The complete game
- `1.html` — Gameplay view mockup (reference for visual style)
- `2.html` — Wounded state mockup (blood vignette reference)
- `3.html` — Menu screen mockup (UI layout reference)
