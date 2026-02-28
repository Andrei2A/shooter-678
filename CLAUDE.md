# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Breach & Clear** — a single-file tactical FPS browser game. The entire game lives in `game.html` (~6700 lines of inline HTML/CSS/JS). No build tools, no package manager, no bundler.

## Running

Open `game.html` directly in a modern browser (Chrome recommended for best WebGL/Web Audio support). No server needed.

## Syntax Checking

```bash
node -e "const fs=require('fs'); const c=fs.readFileSync('game.html','utf8').match(/<script type=\"module\">([\s\S]*)<\/script>/); const s=c[1].replace(/^import\s+.*$/gm,''); try{new Function(s);console.log('OK')}catch(e){console.log(e.message)}"
```

The `import` error is expected (ES modules); only real syntax errors matter. Always run this after edits.

## Architecture

### Single-File Structure (`game.html`)

All code is inline in one HTML file. Order matters — classes reference each other sequentially:

1. **CSS** (~590 lines) — UI overlays, HUD elements, menus, scope/NV/flash/slow-mo overlays
2. **HTML** — Canvas, UI overlay divs, menu/death/victory screens, HUD elements
3. **JS** — Three.js r160 via `<script type="importmap">` from unpkg CDN

### Class Dependency Chain

```
Game (master controller, game loop)
├── InputManager (keyboard/mouse, pointer lock)
├── Player (camera, WASD, sprint/crouch/lean, collision)
├── Weapon (4 types: shotgun/pistol/knife/sniper, 5-finger hand meshes via buildHandGroup)
├── Level (corridor + 10 apartments, procedural textures, lighting, shootable lamps)
│   └── AABB (collision boxes for all walls)
├── Door[] (kick/shoot to open, bullet holes, handle shot, destruction)
├── EnemyManager
│   └── Enemy[] (13-state AI, 4 types, ragdoll physics, playing dead)
│       └── Ragdoll (VerletParticle + DistConstraint + MinDistConstraint)
├── TacticalAI (DeepSeek API coordination + fallback heuristics)
├── SoundManager (all audio procedural via Web Audio API)
├── UIManager (screens, crosshair, blood vignette, screen shake)
├── ShellCasing[] (pooled brass cylinders, 30 max)
├── ParticleSystem (reusable for breaches, explosions)
├── Grenade[] (frag type only)
├── Tripwire[] (laser mine placement)
└── Gas canisters (shootable explosive objects)
```

### Weapon System

4 weapon types switchable via keys 1, 2, 4, V:
- **Shotgun** (1): 6 shells, 8 pellets, pump action, shell casings on pump
- **Pistol** (2): 12 rounds, single wield only
- **Sniper** (4): 5 rounds, scope zoom (FOV 30), 200 damage, wall penetration
- **Knife** (V): silent melee, 1.5m range, instant kill, returns to previous weapon

All weapons have 5-finger hand meshes built via `buildHandGroup()` helper. Hands are positioned below weapons, rotated `Math.PI` on Z so fingers curl upward around grips/barrels.

### Enemy System

**13 AI states**: IDLE, ALERT, ATTACKING, RUSHING, WOUNDED, DEAD, TAKE_COVER, FLANKING, RETREATING, AMBUSH, COORDINATED_RUSH, SURRENDERING, PLAYING_DEAD

**4 enemy types**: regular (2HP), armored (3HP, slow), shotgunner (2HP, fast rush), sniper (1HP, long range)

**Special behaviors**: Playing dead (~30% chance, gets up behind player), surrender (low intelligence, last alive)

**Line-of-sight**: `canSeePlayer()` uses 2D ray-AABB intersection against all wall colliders. Also blocked when apartment door is closed or apartment lamp is destroyed.

### Equipment

- **Frag grenade** (G): 3m kill radius, 5m damage, self-damage
- **Tripwire mines** (C): laser line across doorway, frag explosion on trigger
- **Night vision** (N): 15s battery, boosts lights 1.5x, enemies glow green, recharges when off

### Shootable Lamps

Each apartment has a ceiling lamp (`level.lamps[]`). When shot, the lamp is destroyed (`lamp.destroyed = true`), the apartment light turns off, and enemies inside can no longer see the player. Lamp hit detection is in `handleShooting()` before gas canister checks.

### Key Design Patterns

- **Procedural everything**: Textures via canvas, audio via Web Audio API oscillators/noise buffers, no external assets except Three.js + Google Fonts
- **Verlet ragdoll physics**: 12-particle skeleton, DistConstraint for bones, MinDistConstraint for self-collision, 10 solver iterations
- **Sub-stepped movement**: Player/enemy movement in 0.08m steps to prevent wall tunneling
- **Time scale system**: `this.timeScale` multiplies dt for slow-mo (door breach, kill cam)
- **Kill camera**: Final enemy kill triggers bullet tracer + orbit camera cinematic
- **damagePlayer(fromDir)**: All callers must pass direction vector
- **NV-aware lighting**: Flickering/non-flickering light update loops check `this.nvActive` to adjust clamp limits and multipliers

### TacticalAI / DeepSeek Integration

- API: `https://api.deepseek.com/v1/chat/completions`, model `deepseek-chat`
- Key sources: URL param `?deepseek_key=xxx`, `localStorage` key `deepseek_api_key`, `.env` file
- Sends game state JSON every ~3s; returns `{ strategy, orders: [{ enemyId, action, targetX?, targetZ? }] }`
- Fallback heuristic AI activates without API key or after 3 failures
- Intelligence slider (0-100) on menu affects enemy behavior tier

### Building Layout

```
Z=0 (spawn)              Z=30 (end)
  | [0][2][4][6][8]  |   ← Left apartments (X negative)
  |   Corridor 3m    |
  | [1][3][5][7][9]  |   ← Right apartments (X positive)
```

Corridor: 3m wide × 30m long × 3m tall. Apartments: 5×5m each. Doors at Z=3,9,15,21,27 per side. Random furniture, gas canisters (50% chance per apartment).

### External Dependencies (CDN only)

- Three.js r160 via unpkg importmap
- Google Fonts: Space Grotesk

### Design Tokens

- Primary accent: `#ec7f13` (orange)
- Background dark: `#221910`
- Font: Space Grotesk
- Style: dark/gritty tactical with cinematic vignette and film grain

## Controls Reference

| Key | Action | Key | Action |
|-----|--------|-----|--------|
| WASD | Move | 1 | Shotgun |
| Mouse | Look | 2 | Pistol |
| Left Click | Shoot | 4 | Sniper |
| Right Click | ADS | V | Knife |
| Shift | Sprint | R | Reload |
| Ctrl | Crouch | G | Frag grenade |
| Q/E | Lean | C | Place tripwire |
| F | Kick door | N | Night vision |

## Files

- `game.html` — The complete game (~6700 lines)
- `1.html`, `2.html`, `3.html` — Visual style reference mockups
- `deepseek.md` — DeepSeek API reference notes
- `.env` — API key (gitignored, never commit)
