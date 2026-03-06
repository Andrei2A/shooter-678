# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Breach & Clear** — a single-file tactical FPS browser game. The entire game lives in `game.html` (~9600 lines of inline HTML/CSS/JS). No build tools, no package manager, no bundler.

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

1. **CSS** (lines 8–585) — UI overlays, HUD, menus, scope/NV/flash/slow-mo/suppression overlays
2. **HTML** (lines 586–691) — Canvas, UI overlay divs, menu/death/victory screens, HUD elements
3. **JS** (lines 692–9625) — Three.js r160 via `<script type="importmap">` from unpkg CDN

### Class Dependency Chain

```
Game (master controller, game loop, blood/particle systems)
├── InputManager (keyboard/mouse, pointer lock)
├── Player (camera, WASD, sprint/crouch/lean/slide, collision, breathing sway, suppression)
├── Weapon (4 types: shotgun/pistol/knife/sniper, tactical reload, 5-finger hands)
├── Level (corridor + 10 apartments with 3 rooms each, procedural textures, lighting)
│   └── AABB (collision boxes for all walls)
├── Door[] (kick/shoot to open, bullet holes, handle shot, destruction)
├── EnemyManager
│   └── Enemy[] (18-state AI, 4 types, ragdoll, playing dead, wounded crawl, suppression)
│       └── Ragdoll (VerletParticle + DistConstraint + MinDistConstraint)
├── TacticalAI (DeepSeek API coordination + fallback heuristics)
├── SoundManager (all audio procedural via Web Audio API)
├── UIManager (screens, crosshair, blood vignette, screen shake)
├── ShellCasing[] (pooled brass cylinders, 30 max)
├── ParticleSystem (reusable for breaches, explosions)
├── Grenade[] (frag type only)
├── Tripwire[] (laser mine placement)
├── Gas canisters (shootable explosive objects)
└── Furniture[] (destructible, HP-based, debris on destroy)
```

### Class Line Index

Quick reference for navigating the 9600-line file:

| Line | Class | Line | Class |
|------|-------|------|-------|
| 698 | AABB | 4175 | Enemy |
| 751 | VerletParticle | 5806 | EnemyManager |
| 815 | DistConstraint | 5920 | TacticalAI |
| 841 | MinDistConstraint | 6299 | Player |
| 866 | Ragdoll | 6704 | UIManager |
| 1087 | SoundManager | 6802 | Game |
| 2283 | InputManager | 3876 | ShellCasing |
| 2501 | Level | 3947 | ParticleSystem |
| 2923 | Door | 4018 | Grenade |
| 3271 | Weapon | 4098 | Tripwire |

### Weapon System

4 weapon types switchable via keys 1, 2, 4, V:
- **Shotgun** (1): 6 shells, 8 pellets, pump action, shell casings on pump, bayonet (B key)
- **Pistol** (2): 12 rounds, tactical reload, dual-wield toggle (press 2 again), suppressor (U), laser sight (J)
- **Sniper** (4): 5 rounds, scope zoom (FOV 30), 200 damage, wall penetration 0.5m, suppressor (U)
- **Knife** (V): silent melee, 1.5m range, instant kill, returns to previous weapon

**Weapon attachments**: Suppressor (U key, pistol/sniper), flashlight (L key), laser sight (J key, pistol)

**Weapon mechanics**: Overheat system, 2% jam chance (R to unjam), weapon inspection (T key), ammo types (3 key: standard/AP/hollow-point/incendiary), magazine drop on reload

**Tactical reload**: When `ammo > 0`, reload is faster and grants +1 round in chamber (pistol, sniper).

**Wall penetration**: Weapon-dependent — sniper 0.5m, pistol 0.25m, shotgun 0.

### Enemy System

**18 AI states**: IDLE, ALERT, ATTACKING, RUSHING, WOUNDED, DEAD, TAKE_COVER, FLANKING, RETREATING, AMBUSH, COORDINATED_RUSH, SURRENDERING, PLAYING_DEAD, PANIC, DRAGGING, THROWING_BACK, FLIPPING_TABLE, HEALING

**4 enemy types**: regular (2HP), armored (3HP, slow), shotgunner (2HP, fast rush), sniper (1HP, long range)

**3-4 enemies per apartment** (~30-40 total). Enemies shoot immediately when door opens.

**Special behaviors**:
- Playing dead (~30% chance, gets up behind player)
- Surrender (low intelligence, last alive)
- Wounded crawl + medic healing (10% regular enemies are medics)
- Drag wounded allies to safety
- Panic state (low intel, few allies, drops weapon, runs randomly)
- Throw grenades back (15% chance near grenade)
- Radio chatter, turn off lights, sniper laser visibility
- Armored enemies throw flashbangs (10% on first spot)
- Snipers shoot through thin walls with reduced accuracy
- Suppression: near-miss bullets increase `enemy.suppression`

**Line-of-sight**: `canSeePlayer()` uses 2D ray-AABB intersection against all wall colliders. Blocked when door closed or apartment lamp destroyed.

**NPCs**: K9 dogs (2-3 per game, charge at 5.5 speed), civilian hostages (1-2, penalty for killing)

### Player Mechanics

- **Movement inertia**: Velocity-based with acceleration/deceleration curves
- **Stamina**: Drains when sprinting (25/s), recovers at 15/s, blocks sprint when depleted
- **Jump**: Space key, velocity 4.5, gravity 12, costs 10 stamina
- **Prone**: Z key, 0.3x speed, lower camera (0.5y)
- **Vault**: Space near low furniture to jump over
- **Slide** (Sprint + Ctrl while moving): 0.5s duration, 2.4x speed
- **Bandaging**: B key, 3s duration, heals 1HP, clears injuries
- **Injury system**: 25% chance on non-fatal hit (leg = 0.6x speed, arm = extra sway)
- **Suppression**: Enemy near-misses add suppression (camera flinch, spread penalty)
- **Heavy breathing**: After stamina drain, extra sway and breathing sounds
- **Trip over bodies**: Camera stumble when walking over dead enemies

### Equipment

- **Frag grenade** (G): 3m kill radius, 5m damage, self-damage, tinnitus/muffled audio, fire propagation, ceiling debris
- **Tripwire mines** (C): laser line across doorway, frag explosion on trigger
- **Night vision** (N): 15s battery, boosts lights 1.5x, enemies glow green, recharges when off
- **Door barricading**: Ctrl+F on open door to barricade it shut

### Blood & Gore System

- **Blood splats**: Pre-generated texture pool (6 textures at init), headshot vs body shot variants, placed on walls via 2D ray-AABB detection
- **Blood particles**: 3D spheres with gravity physics, shared `_bloodParticleGeo`
- **Floor blood drops**: Shared `_bloodDropGeo` + pre-generated textures (6 at init)
- **Blood pools**: Organic bezier shapes, multi-layer fill (coagulated edges, wet center, specular), spawns under dead enemies after 1.5s
- **Wounded blood trail**: Floor drops every 0.4s during crawl

### Destructible Environment

- **Furniture**: Tracked in `this.furniture[]` with HP (3-5). Hit detection via `Box3.intersectsBox`. On destroy: removes mesh + collider, spawns 2-3 debris chunks + 3 dust particles
- **Shootable lamps**: 3 per apartment. Destroyed lamp darkens room, blocks enemy LOS
- **Wall hits**: Decal + 2-3 dust particles + 1 debris chunk, ricochet (15% chance), metal sparks
- **Gas canisters**: 50% chance per apartment, explode when shot
- **Windows**: Glass panes on outer walls, crack on first hit (#43), shatter on second (#44)
- **Water pipes**: 30% chance per apartment kitchen, burst when shot
- **Sparking wires**: 20% chance per apartment kitchen, electric damage
- **Fire system**: Explosions have 40% fire chance, fire damages nearby player, produces smoke

### Building Layout

```
Z=0 (spawn)                    Z=50 (end)
  | [0]  [2]  [4]  [6]  [8] |   <- Left apartments (X negative)
  |      Corridor 3m wide    |
  | [1]  [3]  [5]  [7]  [9] |   <- Right apartments (X positive)
```

Corridor: 3m wide x 50m long x 3m tall. Apartments: 8x8m each with 3 rooms (entry room, kitchen, bedroom). Internal doorways 2.0m wide. Doors at Z intervals of 10. Random furniture, gas canisters (50% chance per apartment).

### Key Design Patterns

- **Procedural everything**: Textures via canvas, audio via Web Audio API oscillators/noise buffers, no external assets except Three.js + Google Fonts
- **Verlet ragdoll physics**: 12-particle skeleton, DistConstraint for bones, MinDistConstraint for self-collision, 10 solver iterations
- **Sub-stepped movement**: Player/enemy movement in 0.08m steps to prevent wall tunneling
- **Time scale system**: `this.timeScale` multiplies dt for slow-mo (door breach, kill cam)
- **Kill camera**: Final enemy kill triggers bullet tracer + orbit camera cinematic
- **damagePlayer(fromDir)**: All callers must pass direction vector
- **NV-aware lighting**: Flickering/non-flickering light update loops check `this.nvActive` to adjust clamp limits and multipliers
- **Weapon state sync**: `Game.update()` syncs `weapon.isADS` and `weapon.weaponType` to `player._isADS` / `player._weaponType` before `player.update()`

### Performance Patterns

Critical for maintaining 60fps with 30-40 enemies and particle effects:

- **Shared geometries**: `_dustGeo`, `_debrisGeo`, `_hitMarkGeo`, `_bloodDropGeo`, `_bloodParticleGeo` — created once in Game constructor, reused for all particles
- **Pre-generated textures**: 6 blood splat + 6 blood drop textures generated at init, randomly selected per hit (avoids per-hit canvas creation)
- **2D ray-AABB slab test**: Wall hit detection uses fast math against `level.colliders[]` instead of `intersectObjects(scene.children, true)` — critical bottleneck fix
- **Furniture hit detection**: `Box3.intersectsBox` instead of scene raycast
- **Suppression optimization**: Squared distance comparison, skip enemies >15m, no Vector3 allocations in hot loop
- **Mesh cleanup**: Blood/particle meshes with `shared` flag skip geometry/texture disposal (only dispose material)
- **Particle caps**: hitMarks 40, bloodSplats 40, reduced spawn counts (2-3 dust, 1 debris per wall hit)

### TacticalAI / DeepSeek Integration

- API: `https://api.deepseek.com/v1/chat/completions`, model `deepseek-chat`
- Key sources: URL param `?deepseek_key=xxx`, `localStorage` key `deepseek_api_key`, `.env` file
- Sends game state JSON every ~3s; returns `{ strategy, orders: [{ enemyId, action, targetX?, targetZ? }] }`
- Fallback heuristic AI activates without API key or after 3 failures
- Intelligence slider (0-100) on menu affects enemy behavior tier

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
| Mouse | Look | 2 | Pistol (2x = dual) |
| Left Click | Shoot | 3 | Cycle ammo type |
| Right Click | ADS | 4 | Sniper |
| Shift | Sprint | V | Knife |
| Ctrl | Crouch | R | Reload / unjam |
| Sprint+Ctrl | Slide | G | Frag grenade |
| Q/E | Lean | C | Place tripwire |
| F | Kick door | N | Night vision |
| Ctrl+F | Barricade door | L | Flashlight |
| Space | Jump / vault | U | Suppressor toggle |
| Z | Prone | J | Laser sight |
| B | Bandage / bayonet | T | Weapon inspection |

## Files

- `game.html` — The complete game (~9600 lines)
- `1.html`, `2.html`, `3.html` — Visual style reference mockups
- `deepseek.md` — DeepSeek API reference notes
- `.env` — API key (gitignored, never commit)
