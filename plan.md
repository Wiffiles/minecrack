# Minecrack 1.12.2. Web Edition — Complete Implementation Plan

> **Goal**: Faithful web recreation of Minecraft Java Edition 1.12.2 (World of Color Update). Single-player only. Built from scratch in the browser. All visual/audio assets belong to Mojang AB — fan project only, no commercial distribution.
>
> **Credits**: enzod — for the vision, assets, and making this project possible.

---

## P0 — Project Architecture

### 0.1 Stack
```
Language     TypeScript strict (no implicit any)
Bundler      Vite 5+ (ESM, fast HMR, TypeScript out of box)
3D API       Three.js r150+ (WebGL2 renderer, falling back to WebGL1)
             Custom shaders via ShaderMaterial (GLSL ES 1.0)
             Abstractions: chunk mesh builder, sky shader, entity shader
Audio        Web Audio API (positional 3D audio via PannerNode)
             AudioBuffer loading via decodeAudioData + caching
             Dynamic music system (crossfade between tracks)
Storage      IndexedDB (via idb library for promisified API)
             Worker-based serialization (offload NBT encoding)
Workers      Web Workers for chunk generation, mesh building, terrain noise
             SharedArrayBuffer for worker ↔ main thread transfer (where available)
Physics      Custom AABB collision (float-based, no breaking)
             Axis-aligned sweep-and-prune broadphase (spatial hash grid)
             Sub-tick collision resolution (continuous collision detection)
UI           HTML/CSS overlay + Canvas (pixel-perfect rendering)
             React? No — vanilla DOM manipulation for performance
             CSS custom properties for theming (dark/light mode follows MC)
Networking   Single-player only (no multiplayer protocol)
             Local saves only (IndexedDB)
```

### 0.2 Directory Structure
```
src/
├── core/           Engine loop, events, tick system, math, ECS framework
│   ├── engine.ts       — Main game loop (update+render), requestAnimationFrame
│   ├── tick.ts         — Tick scheduling, fixed-step 20tps, accumulator
│   ├── event.ts        — Typed event bus (pub/sub with strict types)
│   ├── math.ts         — Vec3, AABB math, frustum, noise helpers
│   ├── ecs.ts          — Entity-Component-System registry, queries
│   ├── time.ts         — Game time, delta time, interpolation factor
│   ├── registry.ts     — Block/Item/Entity registry (string-ID based)
│   └── constants.ts    — All game constants (ticks, speeds, distances)
│
├── render/         WebGL pipeline, shaders, chunk meshing, sky, fog
│   ├── renderer.ts     — Main renderer init, resize, scene graph
│   ├── chunkMesh.ts    — Greedy meshing, face culling, vertex packing
│   ├── shaders/        — GLSL shader source files
│   │   ├── chunk.vert      — Chunk vertex shader (position, UV, light, AO)
│   │   ├── chunk.frag      — Chunk fragment shader (texture atlas, fog)
│   │   ├── sky.vert        — Sky dome vertex shader
│   │   ├── sky.frag        — Sky gradient + sun/moon/stars
│   │   ├── entity.vert     — Entity vertex shader (billboard or mesh)
│   │   ├── entity.frag     — Entity fragment shader
│   │   ├── water.vert      — Animated water vertex shader
│   │   ├── water.frag      — Water fragment shader (transparency)
│   │   ├── ui.vert         — UI overlay vertex shader (2D ortho)
│   │   └── ui.frag         — UI overlay fragment shader
│   ├── sky.ts            — Sky dome: gradient, sun, moon, stars, clouds
│   ├── fog.ts            — Fog rendering (exponential fog, weather-aware)
│   ├── textureAtlas.ts   — Texture atlas generation, UV mapping
│   ├── entityRenderer.ts — Entity batch instance rendering
│   ├── particleSystem.ts — GPU particle system (transform feedback)
│   ├── hitboxRenderer.ts — Debug hitbox rendering (F3+B toggle)
│   └── debugOverlay.ts   — F3 debug overlay (text rendering)
│
├── world/          Chunks, blocks, world gen, lighting, biomes
│   ├── world.ts         — World container: dimension management
│   ├── chunk.ts         — Chunk class: blocks, light, biomes, entities
│   ├── chunkManager.ts  — Chunk loading/unloading, priority queue
│   ├── chunkGenerator.ts— Terrain generation delegate (noise + features)
│   ├── worldGen/        — World generation pipeline
│   │   ├── noise.ts         — Perlin/Simplex noise, octaves, domain warp
│   │   ├── biomeProvider.ts — Biome map generation (temperature, humidity)
│   │   ├── terrainGen.ts    — Heightmap, cave carve, ravine, ore
│   │   ├── structureGen.ts  — Tree, village, fortress, temple, etc.
│   │   ├── lakeGen.ts       — Water/lava lake generation
│   │   └── featureGen.ts    — Flowers, grass, mushrooms, etc.
│   ├── lighting.ts       — Sky light propagation + block light + AO
│   ├── blockTick.ts      — Random block ticks (crops, fire, ice, etc.)
│   ├── blockUpdate.ts    — Block update/neighbor notification system
│   └── biome.ts          — Biome definitions, properties, colors
│
├── player/         Controller, physics, inventory, stats, abilities
│   ├── player.ts         — Player entity: position, rotation, state
│   ├── controller.ts     — Keyboard/mouse input, keybinds, look
│   ├── physics.ts        — Player physics: gravity, collision, stepping
│   ├── inventory.ts      — Inventory container, slots, stacking
│   ├── hotbar.ts         — Hotbar (9 slots), selection, switching
│   ├── stats.ts          — Health, hunger, saturation, oxygen, XP
│   ├── abilities.ts      — Game-mode-specific abilities (fly, instabreak)
│   └── camera.ts         — Camera: first/third person, smoothing, bob
│
├── entities/       Mob base, AI goals, item entities, projectiles
│   ├── entity.ts         — Base entity class: tick, physics, render
│   ├── entityRegistry.ts — Mob type registration, spawning logic
│   ├── ai/               — AI goal system
│   │   ├── goal.ts           — Base goal class, canStart/shouldContinue/tick
│   │   ├── goalSelector.ts   — Goal selector (priority-based scheduling)
│   │   ├── goals/            — Individual goal implementations
│   │   │   ├── MeleeAttackGoal.ts
│   │   │   ├── RangedAttackGoal.ts
│   │   │   ├── FleeSunGoal.ts
│   │   │   ├── RandomStrollGoal.ts
│   │   │   ├── FollowOwnerGoal.ts
│   │   │   ├── EatGrassGoal.ts
│   │   │   ├── MoveThroughVillageGoal.ts
│   │   │   └── ... (all goals per P6)
│   │   └── pathfinding.ts    — A* pathfinding, navigable terrain checks
│   ├── mobs/             — Individual mob implementations
│   │   ├── Zombie.ts
│   │   ├── Skeleton.ts
│   │   ├── Creeper.ts
│   │   ├── Spider.ts
│   │   ├── ... (all 40+ mob types)
│   ├── projectiles.ts   — Arrow, snowball, egg, ender pearl, potion
│   ├── itemEntity.ts    — Dropped item entity (physics + pickup)
│   └── xpOrb.ts         — Experience orb entity
│
├── blocks/         Block registry, states, models, properties
│   ├── blockRegistry.ts  — Map<string, Block> with metadata
│   ├── blockState.ts     — BlockState object (block + properties)
│   ├── blockModel.ts     — Block model geometry, cullface, AO
│   ├── material.ts       — Block material properties (hardness, tool, sound)
│   └── blocks/           — Per-block logic (if needed)
│       ├── Furnace.ts        — Furnace block entity
│       ├── Chest.ts          — Chest block entity
│       ├── Piston.ts         — Piston block entity
│       ├── Beacon.ts         — Beacon block entity
│       └── ... (all block entities)
│
├── items/          Item registry, tool stats, food values
│   ├── itemRegistry.ts   — Map<string, Item> with metadata
│   ├── itemStack.ts      — ItemStack: item + count + NBT
│   ├── toolStats.ts      — Tool speeds, tiers, enchantability
│   ├── foodStats.ts      — Food values, saturation, effects
│   └── items/            — Per-item logic (if special behavior)
│       ├── FlintAndSteel.ts
│       ├── Bucket.ts
│       ├── Shield.ts
│       └── ...
│
├── recipes/        Crafting, smelting, brewing
│   ├── crafting.ts       — Shaped + shapeless recipe matching
│   ├── furnace.ts        — Smelting recipes + fuel values
│   ├── brewing.ts        — Brewing stand recipes + potion data
│   └── recipeBook.ts     — Recipe unlock, category, search
│
├── ui/             HUD, menus, inventory screens, overlays
│   ├── hud.ts            — Crosshair, hotbar, health, hunger, XP, air
│   ├── inventory.ts      — Survival + creative inventory GUIs
│   ├── craftingTable.ts  — Crafting table GUI + recipe book
│   ├── furnace.ts        — Furnace GUI
│   ├── chest.ts          — Chest GUI (single + double)
│   ├── anvil.ts          — Anvil GUI (renaming + repair)
│   ├── enchantingTable.ts— Enchanting table GUI
│   ├── brewingStand.ts   — Brewing stand GUI
│   ├── beacon.ts         — Beacon GUI
│   ├── villager.ts       — Villager trading GUI
│   ├── horse.ts          — Horse inventory GUI
│   ├── deathScreen.ts    — Death/respawn screen
│   ├── pauseMenu.ts      — Esc menu
│   ├── advancement.ts    — Advancement tree viewer
│   ├── container.ts      — Container window management (slots, drag)
│   └── components/       — Reusable UI components
│       ├── button.ts         — MC-styled button
│       ├── slot.ts           — Inventory slot (with item render)
│       ├── text.ts           — MC-styled text (pixel font rendering)
│       └── tooltip.ts        — Item tooltip rendering
│
├── audio/          Sound engine, music, ambience
│   ├── soundEngine.ts    — Sound loading, playback, pooling
│   ├── music.ts          — Music track selection, crossfade
│   ├── ambience.ts       — Biome ambience, cave sounds
│   └── soundRegistry.ts  — Map<string, Sound> with metadata
│
├── redstone/       Dust, repeaters, comparators, pistons
│   ├── dust.ts            — Redstone dust component + power calc
│   ├── repeater.ts        — Repeater timing logic
│   ├── comparator.ts      — Comparator mode logic
│   ├── piston.ts          — Piston extend/retract + block pushing
│   ├── observer.ts        — Observer block detection
│   ├── wireSystem.ts      — Redstone wire network data structure
│   └── blockEventQueue.ts — Block event queue (tick scheduling)
│
├── storage/        IndexedDB save/load, NBT-like serialization
│   ├── db.ts              — IndexedDB schema, CRUD operations
│   ├── worldSerializer.ts — World ↔ binary serialization
│   ├── chunkSerializer.ts — Chunk ↔ binary (compressed)
│   ├── playerSerializer.ts— Player data ↔ binary
│   └── nbt.ts             — NBT-like binary format writer/reader
│
├── assets/         Pointers to textures, sounds, blockstates, models
│   ├── assetManager.ts   — Asset loading, caching, path resolution
│   ├── textureLoader.ts  — Image loading, atlas baking
│   ├── modelLoader.ts    — Block model JSON parser
│   └── language.ts       — Language string loading (en_US.lang)
│
└── main.ts          Entry point, bootstrap, hot module replacement
```

### 0.3 Build Toolchain
```
Vite configuration (vite.config.ts):
  - Build target: ES2020 (modern browsers)
  - outDir: dist/
  - assetsInlineLimit: 0 (no inlining for game assets)
  - Worker format: esm (native worker module support)
  - Chunk splitting: manual for core vs. world gen (large workers)
  - Source maps: inline for dev, hidden for production
  - Asset handling: copy minecraft/ assets to dist/ as-is
  - Dev server: port 3000, open browser, HMR for source files
    HMR boundary: game reload on source change, no full page reload
    Asset HMR: texture/sound changes trigger hot reload of asset cache

TypeScript configuration (tsconfig.json):
  - strict: true (all strict checks enabled)
  - target: ES2020, module: ESNext, moduleResolution: Bundler
  - noUnusedLocals: true, noUnusedParameters: true
  - noFallthroughCasesInSwitch: true
  - exactOptionalPropertyTypes: true
  - resolveJsonModule: true (for importing blockstate JSONs)
  - allowImportingTsExtensions: false (use .js extension in output)
  - outDir: dist/
  - rootDir: src/

Linting (ESLint + Prettier):
  - ESLint: @typescript-eslint/recommended, strict type-checking rules
    Rules: no-any (error), no-non-null-assertion (warn), etc.
  - Prettier: consistent formatting (tab width: 2, single quotes, trailing commas)
  - Pre-commit hook: lint-staged (lint + format changed files)

Testing:
  - Framework: Vitest (native TypeScript, fast, compatible with Vite)
  - Test types:
    Unit tests: pure logic (math, inventory, crafting, noise, serialization)
    Integration tests: systems interaction (block updates, redstone, AI)
    Rendering tests: visual diff (optional, manual review)
  - Test location: co-located with source files (*.test.ts)
  - Coverage: istanbul via Vitest (target: 80%+ coverage for core systems)
  - CI: GitHub Actions (optional, local-only project so manual runs OK)

Scripts (package.json):
  dev:        vite (dev server with HMR)
  build:      tsc --noEmit && vite build (typecheck + bundle)
  preview:    vite preview (serve built output)
  lint:       eslint src/ --ext .ts
  format:     prettier --write src/
  test:       vitest run
  test:watch: vitest
  typecheck:  tsc --noEmit

Dependency strategy:
  Minimize third-party dependencies (self-contained where possible)
  Approved dependencies:
    - three (r150+): 3D rendering
    - idb (7+):      IndexedDB promisified wrapper
    - typescript:    Language
    - vite:          Bundler
    - vitest:        Testing
    - eslint:        Linting
    - prettier:      Formatting
  NO dependencies:
    - Physics libraries (custom AABB is simpler and more accurate)
    - UI frameworks (vanilla DOM is fast enough)
    - State management (global game state is simple enough)
    - Audio libraries (Web Audio API is sufficient)
    - Network libraries (single-player only)
```

### 0.4 Core Architecture Patterns
```
ECS-like entity system (not full ECS, hybrid approach):
  Entity: unique ID + component map (Map<ComponentType, Component>)
  Component: plain data interface/type (position, health, AI state, etc.)
  System: function that queries entities by component signature
    Systems run in order each tick:
    1. Input system (keyboard/mouse state → components)
    2. Physics system (movement, gravity, collision detection)
    3. AI system (mob goal updates, pathfinding)
    4. Block update system (neighbor updates, scheduled ticks)
    5. Entity update system (per-entity tick: health, effects, timers)
    6. Block entity system (furnace smelting, chest animations)
    7. Redstone system (wire power, component updates)
    8. Random tick system (crops, fire spread, ice melt)
    9. Spawn system (natural mob spawning)
    10. Despawn system (mob despawning rules)
    11. Particle system (particle updates)
    12. Render system (prepare render data, interpolate positions)

Event bus (typed pub/sub):
  Events: typed string enums
  Payload: strongly typed per event
  Usage: decoupled communication between systems
  Game events:
    - PlayerJoin, PlayerLeave (dimension switch)
    - BlockChange (block set/removed, with old and new state)
    - BlockUpdate (neighbor notification)
    - EntitySpawn, EntityDespawn
    - EntityDamaged, EntityKilled
    - ItemPickup, ItemDrop
    - ContainerOpen, ContainerClose
    - GameModeChange
    - DifficultyChange
    - TimeChange, WeatherChange
    - AdvancementGrant
    - RecipeUnlock
    - WorldSave, WorldLoad

Asset loading pipeline:
  1. Asset manifest: index of all textures/sounds/models (generate at build time)
  2. Lazy loading: only load assets when needed (biome-specific textures on demand)
  3. Preload: critical assets on world load (sky, GUI, player model, held items)
  4. Texture atlas: all block textures baked into single atlas on load
     Atlas dimensions: 4096×4096 (supports ~1000+ 16×16 textures with padding)
     Atlas generation: canvas-based, loads all block textures, arranges in grid
     UV mapping: each block face gets UV coordinates into the atlas
  5. Sound pool: pre-allocate AudioBuffer slots (max 32 simultaneous sounds)
  6. Asset cache: Map<string, LoadedAsset> with reference counting
     Unload: when no loaded chunks reference an asset

Developer workflow:
  1. Clone repo, npm install
  2. npm run dev → opens localhost:3000 with hot reload
  3. Edit source → instant HMR (game state preserved where possible)
  4. npm test → run all unit tests (fast, < 5s)
  5. npm run build → production build to dist/
  6. npm run lint → check for issues before commit

Hot Module Replacement strategy:
  - Source files: full HMR (replace module, re-run init)
  - Block/item definitions: reload registry, re-mesh affected chunks
  - Shaders: hot-swap GLSL, recompile, test visually
  - Assets: trigger re-load if texture/sound file changes
  - Full page reload only on structural changes (ECS system order, etc.)
```

### 0.5 Tick System
```
Game tick:   20 tps (50ms per tick)
Render tick: Uncapped (VSync), interpolated between game ticks

Fixed-step game loop:
  accumulator = 0
  each frame (requestAnimationFrame):
    delta = min(frameTime, 100ms)  // cap to prevent spiral of death
    accumulator += delta
    while (accumulator >= 50ms):
      fixedUpdate()   // game logic at 20Hz
      accumulator -= 50ms
    interpolationFactor = accumulator / 50ms
    render(interpolationFactor)  // interpolate positions for smooth rendering

Block ticks:
  Random tick: each chunk, 3 random blocks per tick (randomTickSpeed gamerule)
  Scheduled tick: block requests tick in future (e.g., piston retract, redstone repeater)
    Stored in chunk as priority queue: Map<tickTime, BlockPos[]>

Entity tick:
  Each entity updates once per game tick
  Update order: player → passive → neutral → hostile → projectiles → items

Item despawn: 6000 ticks (5 minutes) for dropped items
  XP orbs: 6000 ticks (5 minutes) as well
  Some items never despawn: nether star? No, it does despawn after 5 min.
  Items in loaded chunks: timer ticks down
  Items in unloaded chunks: timer paused (resume on chunk load)
```

---

### 0.4 Coordinate System
- Y-up, X/Z horizontal. Chunk: 16×256×16. Region: 32×32 chunks.
- Block pos: integer x,y,z. Sub-block: 1/16 precision.
- Player pos: floating point, eye height 1.62m above feet.

---

## P1 — Player Mechanics

### 1.1 Movement Speeds (m/s)
```
State                | Speed
Walking              | 4.317
Sprinting            | 5.612
Sneaking             | 1.295
Walking backward     | 3.022
Strafe walking       | 3.022
Sprint-jump          | 7.127
Flying (creative)    | 10.0 (21.6 sprint-fly)
Swimming             | 2.97 (5.29 sprint)
Swimming (sneak)     | 1.96
In water             | 0.8× walk
In lava              | 0.5× walk (applied 5×/s)
Soul sand            | 0.4× walk
Slime block          | 0.5× walk
Ice                  | 0.98 friction
```

### 1.2 Jump Mechanics
```
Jump velocity:    0.42 m/s upward
Jump height:      1.2522 blocks (≈1.25)
Sprint-jump dist: 4.0 blocks (flat ground)
Jump cooldown:    10 game ticks (0.5s)
Jump boost:       +0.1 × (level+1) to initial velocity
```

### 1.2B Auto-Jump Mechanics
```
Auto-jump is a 1.12 accessibility option (toggle in Controls settings).
When enabled: player automatically jumps when moving forward/backward into a 1-block obstacle.

Trigger conditions:
  - Player is on ground (OnGround = true)
  - Player is moving forward (pressing W or S) into a block ≤ 1 block high
  - Block directly in front of player at eye level is solid
  - Player has space above to jump (0.6m clearance minimum)
  - Player is not sneaking (sneak disables auto-jump)
  - Player is not in water/lava (swimming overrides auto-jump)
  - Player is not in a 1.5m tall space (cannot stand up → no jump)
  - Jump cooldown respect: auto-jump obeys the 10 tick jump cooldown

Behavior:
  - Executes a standard jump (0.42 m/s upward velocity)
  - Same animation as manual jump
  - Auto-jump does NOT trigger jump boost effects from blocks (slime, etc.)
  - Cannot auto-jump while flying (creative flight overrides)
  - Cannot auto-jump while riding a horse/pig/minecart/boat

Visual feedback:
  - Player briefly enters the "jumping" pose/arm swing
  - No sound difference from manual jump
```

### 1.3 Fall Damage
```
Threshold: 3 blocks (≤2 = no damage)
Formula:   ceil(fall_distance - 3.0) damage (half-hearts)
FF4 cap:   48% reduction (8% × level² per level)
Water:     negates all fall damage (depth ≥2)
Slime:     negates + bounce at 50% velocity (sneak = no bounce)
Hay bale:  80% reduction

Void:      4 damage per 0.5s below y=0 (OW), y=127 (Nether), y=256 (End)
```

### 1.4 Food & Saturation
```
Food                 | Hunger | Saturation | Effects
Apple                | 4      | 2.4        |
Baked Potato         | 6      | 7.2        |
Beetroot             | 1      | 1.2        |
Beetroot Soup        | 6      | 7.2        |
Bread                | 5      | 6.0        |
Carrot               | 3      | 3.6        |
Cooked Chicken       | 6      | 7.2        |
Cooked Cod           | 5      | 6.0        |
Cooked Mutton        | 6      | 9.6        |
Cooked Porkchop      | 8      | 12.8       |
Cooked Rabbit        | 5      | 6.0        |
Cooked Salmon        | 6      | 9.6        |
Cookie               | 2      | 0.4        |
Golden Apple         | 4      | 9.6        | Absorp I 2min, Regen II 5s
Ench. Golden Apple   | 4      | 9.6        | Absorp IV 2min, Regen V 30s, Fire Res 5min, Res III 5min
Golden Carrot        | 6      | 14.4       |
Melon Slice          | 2      | 1.2        |
Mushroom Stew        | 6      | 7.2        |
Poisonous Potato     | 2      | 1.2        | Poison 5s (60%)
Potato               | 1      | 0.6        |
Pufferfish           | 1      | 0.2        | Hunger III 15s, Poison IV 60s, Nausea II 15s
Pumpkin Pie          | 8      | 4.8        |
Rabbit Stew          | 10     | 12.0       |
Raw Beef             | 3      | 1.8        |
Raw Chicken          | 2      | 1.2        | Hunger 30s (30%)
Raw Mutton           | 2      | 1.2        |
Raw Porkchop         | 3      | 1.8        |
Raw Rabbit           | 2      | 1.8        |
Raw Salmon           | 2      | 1.2        |
Rotten Flesh         | 4      | 0.8        | Hunger 30s (80%)
Spider Eye           | 2      | 3.2        | Poison 4s
Steak                | 8      | 12.8       |
Chorus Fruit         | 4      | 2.4        | Random teleport

Exhaustion per action:
  Breaking a block:     0.005
  Damage taken:         0.1
  Healed by regen:      3.0
  Jumping:              0.05
  Sprint-jump:          0.2
  Swimming (per m):     0.01
  Sprinting (per m):    0.01
  Walking (per m):      0.0

Natural regen: food ≥ 18 and saturation > 0 → 1 HP per 0.5s
Starvation: food = 0 → 0.5 dmg per 4s (HP>10) or 2s (HP≤10)
```

### 1.5 Armor Damage Reduction
```
Full set values:
  Leather:  3.5  total armor
  Gold:     5.5
  Chain:    6.0
  Iron:     7.5
  Diamond:  10.0 (+2 toughness per piece)

Formula:
  EPF = sum(protection ench × rand(0.5,1.0))
  EPF = floor(EPF × rand(0.5,1.0)), cap 20
  Reduction% = min(ceil(armor×4%), 80%)
  Final dmg = dmg × (1-reduction%) × (1-EPF/25)

Armor dmg: floor(total_dmg_received / 4) + 1 per piece per hit
Thorns: 15%/level chance of 1-4 dmg, costs 3× durability
```

### 1.6 XP Formula
```
Levels 0-15:     floor(level² + 6×level + 7)
Levels 16-30:    floor(2.5×level² - 40.5×level + 360)
Levels 31+:      floor(4.5×level² - 162.5×level + 2220)
Max shown level: 247 (unlimited internally)
```

---

### 1.7 Custom Skins & Capes
```
Skin system:
  - Accept player skin in standard Minecraft format (64×64 PNG with 1.12.2 layout)
  - Layers: base skin + overlay (hat, jacket, sleeves, pants)
  - Transparent pixels on second layer = show base layer beneath
  - Slim vs classic arm model (3px vs 4px wide)
  - Skin upload via file picker or URL input
  - Skin preview in Options menu (player model rotating)
  - Third-person (F5) renders custom skin with all layers

Cape system:
  - Custom cape image (64×32 PNG)
  - Rendered on player back in third-person
  - Waving animation (gentle sine-wave oscillation when moving)
  - Cape toggle in Options menu (ON/OFF)
  - Cape upload via file picker or URL input
  - Default: no cape (or a simple default cape)

Player model rendering (third-person):
  - 3D box model: head (8×8×8), torso (8×12×4), arms (4×12×4), legs (4×12×4)
  - Slim arms: 3px wide (3×12×4)
  - Overlay layer rendered slightly offset (0.25px outward)
  - Animations: walking swing, idle breathing, arm swing on attack
  - Skin data stored in IndexedDB along with player data
```

---

## P2 — Complete Block Registry

### 2.1 Natural Blocks
```
Block                  | Hardn | Blast | Tool    | Tier  | Light
Air                    | 0     | 0     | —       | —     | 0
Stone                  | 1.5   | 6     | Pick    | Wood  | 0
Granite                | 1.5   | 6     | Pick    | Wood  | 0
Polished Granite       | 1.5   | 6     | Pick    | Wood  | 0
Diorite                | 1.5   | 6     | Pick    | Wood  | 0
Polished Diorite       | 1.5   | 6     | Pick    | Wood  | 0
Andesite               | 1.5   | 6     | Pick    | Wood  | 0
Polished Andesite      | 1.5   | 6     | Pick    | Wood  | 0
Grass Block            | 0.6   | 0.6   | Shovel  | —     | 0
Dirt                   | 0.5   | 0.5   | Shovel  | —     | 0
Coarse Dirt            | 0.5   | 0.5   | Shovel  | —     | 0
Podzol                 | 0.5   | 0.5   | Shovel  | —     | 0
Cobblestone            | 2.0   | 6     | Pick    | Wood  | 0
Bedrock                | ∞     | 3600  | —       | —     | 0
Sand                   | 0.5   | 0.5   | Shovel  | —     | 0  (gravity)
Red Sand               | 0.5   | 0.5   | Shovel  | —     | 0  (gravity)
Gravel                 | 0.6   | 0.6   | Shovel  | —     | 0  (gravity, 10% flint)
Gold Ore               | 3.0   | 3     | Pick    | Iron  | 0
Iron Ore               | 3.0   | 3     | Pick    | Stone | 0
Coal Ore               | 3.0   | 3     | Pick    | Wood  | 0
Lapis Lazuli Ore       | 3.0   | 3     | Pick    | Stone | 0
Diamond Ore            | 3.0   | 3     | Pick    | Iron  | 0
Redstone Ore           | 3.0   | 3     | Pick    | Iron  | 9 (when touched)
Emerald Ore            | 3.0   | 3     | Pick    | Iron  | 0
Nether Quartz Ore      | 3.0   | 3     | Pick    | Wood  | 0
Obsidian               | 50.0  | 1200  | Pick    | Diamond | 0
Glowstone              | 0.3   | 1.5   | —       | —     | 15
Netherrack             | 0.4   | 0.4   | Pick    | Wood  | 0
Soul Sand              | 0.5   | 0.5   | Shovel  | —     | 0  (slows 50%)
End Stone              | 3.0   | 9     | Pick    | Wood  | 0
Ice                    | 0.5   | 0.5   | —       | —     | 0  (friction 0.98, transparent)
Packed Ice             | 0.5   | 0.5   | —       | —     | 0
Snow Block             | 0.2   | 0.2   | Shovel  | —     | 0
Clay                   | 0.6   | 0.6   | Shovel  | —     | 0
Stone Bricks           | 1.5   | 6     | Pick    | Wood  | 0
Cracked Stone Bricks   | 1.5   | 6     | Pick    | Wood  | 0
Mossy Stone Bricks     | 1.5   | 6     | Pick    | Wood  | 0
Chiseled Stone Bricks  | 1.5   | 6     | Pick    | Wood  | 0
Brick Block            | 2.0   | 6     | Pick    | Wood  | 0
Mossy Cobblestone      | 2.0   | 6     | Pick    | Wood  | 0
Sandstone              | 0.8   | 0.8   | Pick    | Wood  | 0
Chiseled Sandstone     | 0.8   | 0.8   | Pick    | Wood  | 0
Cut Sandstone          | 0.8   | 0.8   | Pick    | Wood  | 0
Smooth Sandstone       | 0.8   | 0.8   | Pick    | Wood  | 0
Red Sandstone variants | same  | same  | same    | same  | 0
Prismarine             | 1.5   | 6     | Pick    | Wood  | 0
Prismarine Bricks      | 1.5   | 6     | Pick    | Wood  | 0
Dark Prismarine        | 1.5   | 6     | Pick    | Wood  | 0
Sea Lantern            | 0.3   | 1.5   | —       | —     | 15
Magma Block            | 0.5   | 0.5   | Pick    | Wood  | 0  (damages standing)
Nether Wart Block      | 1.0   | 1.0   | —       | —     | 0
Red Nether Bricks      | 2.0   | 6     | Pick    | Wood  | 0
Bone Block             | 2.0   | 2     | Pick    | Wood  | 0
End Stone Bricks       | 3.0   | 9     | Pick    | Wood  | 0
Purpur Block           | 1.5   | 6     | Pick    | Wood  | 0
Purpur Pillar          | 1.5   | 6     | Pick    | Wood  | 0
Mycelium               | 0.6   | 0.6   | Shovel  | —     | 0
```

### 2.2 Wood & Decorative Blocks
```
Oak/Spruce/Birch/Jungle/Acacia/Dark Oak Log      | 2.0/2  | Axe | flammable | pillar axis
Note: Stripped logs/wood not in 1.12.2 (added 1.13)
Oak/Spruce/... Wood (bark)                        | 2.0/2  | Axe | flammable
Oak/Spruce/... Planks                             | 2.0/3  | Axe | flammable
Oak/Spruce/... Sapling                            | 0/0    | —   | grows tree
Oak/Spruce/... Leaves                             | 0.2/0.2| —   | decays, shears only
Oak/Spruce/... Fence                              | 2.0/3  | Axe | connects to blocks
Oak/Spruce/... Fence Gate                         | 2.0/3  | Axe | opens/closes
Oak/Spruce/... Stairs                             | 2.0/3  | Axe | shape: straight/inner/outer
Oak/Spruce/... Slab (top/bottom/double)           | 2.0/3  | Axe |
Oak/Spruce/... Door                               | 3.0/3  | Axe | 2 halves, double door
Oak/Spruce/... Trapdoor                           | 3.0/3  | Axe | opens/closes
Oak/Spruce/... Pressure Plate                     | 0.5/0.5| Axe | entity-activated
Oak/Spruce/... Button                             | 0.5/0.5| Axe | 15/20 tick pulse
Glass                                              | 0.3/0.3| —   | transparent
Glass Pane                                         | 0.3/0.3| —   | connects
Stained Glass (16)                                 | 0.3/0.3| —   |
Stained Glass Pane (16)                            | 0.3/0.3| —   |
Crafting Table                                     | 2.5/2.5| Axe | 3×3 grid
Bookshelf                                          | 1.5/1.5| Axe | enchanting +1
Chest                                              | 2.5/2.5| Axe | 27 slots, double
Trapped Chest                                      | 2.5/2.5| Axe | redstone output
Note Block                                         | 0.8/0.8| Axe | 25 instruments × 2 octaves
Jukebox                                            | 2.0/6  | Axe | plays discs
Ladder                                             | 0.4/0.4| Axe | climbable
Sign                                               | 1.0/1.0| Axe | 15×4 chars, wall/standing
```

### 2.3 1.12 New Blocks: Terracotta & Concrete
```
Terracotta (hardened clay)     | 1.25/4.2 | Pick
White/Orange/... Black Terracotta (16) | 1.25/4.2 | Pick
White Concrete (16)            | 1.8/1.8  | Pick
White Concrete Powder (16)     | 0.5/0.5  | Shovel | gravity + hardens on water contact
White/... Glazed Terracotta (16) | 1.4/1.4 | Pick | directional pattern

Concrete powder hardening rules:
  - When concrete powder block touches water (any form: source, flowing, or waterlogged block),
    it instantly hardens into concrete of the same color
  - Contact methods:
    1. Powder block is placed into water (replaces water)
    2. Water flows into powder block
    3. Powder block falls (gravity) into water
    4. Water is placed next to powder block (no — water must touch the powder directly)
    5. Rain: does NOT harden concrete powder
    6. Cauldron water: does NOT harden (must be world water)
    7. Filled by dispenser: if dispenser places powder into water → instantly concrete
  - Once hardened, concrete cannot revert to powder
  - Concrete block properties: 1.8 hardness, blast 1.8, pickaxe required, no gravity, 16 colors
```

### 2.3B Missing Blocks (Infested, Technical, Fluids)
```
Monster Egg (Infested) blocks — 5 variants:
  Infested Stone            | 0.75/0.75 | Pick | spawns silverfish when broken
  Infested Cobblestone      | 0.75/0.75 | Pick | spawns silverfish
  Infested Stone Brick      | 0.75/0.75 | Pick | spawns silverfish
  Infested Mossy Stone Brick| 0.75/0.75 | Pick | spawns silverfish
  Infested Cracked Stone Bricks | 0.75/0.75 | Pick | spawns silverfish

Fire                        | 0/0       | —    | 15 levels of age; spreads on flammable blocks; light varies
Nether Portal               | 0/0       | —    | Purple portal block; light 11; damages entities standing in it (1 HP/1.5s)
End Portal                  | ∞/3600    | —    | 3×3 block grid; teleports to End; light 15; cannot be broken
End Gateway                 | ∞/3600    | —    | Single block; teleports player ~1000 blocks out in End; light 15
Frosted Ice                 | 0.5/0.5   | —    | Created by Frost Walker enchant; melts in light; 4 stages (age 0-3)
```

### 2.4 Decorative & Functional Blocks
```
Carpet (16)          | 0.1/0.1 | —    | placed on top
Wool (16)            | 0.8/0.8 | —    | flammable
Snow Layer (1-8)     | 0.1/0.1 | —    | partial height
Flower Pot           | 0/0     | —    | holds 1 plant
Lever                | 0.5/0.5 | —    | toggle 15
Tripwire Hook        | 0.5/0.5 | —    | requires string
Tripwire             | 0/0     | —    | string triggered by entities
Redstone Torch       | 0/0     | —    | light 7, burns out on 8 toggles/60t
Redstone Repeater    | 0/0     | —    | delay 1-4, lockable, diode
Redstone Comparator  | 0/0     | —    | compare/subtract, container fill
Redstone Lamp        | 0.3/0.3 | —    | light 15 when powered
Redstone Dust        | 0/0     | —    | signal 0-15
Redstone Block       | 5.0/5.0 | —    | permanent power 15
Redstone Ore         | 3.0/3   | Pick | light 9 when touched
Piston               | 0.5/0.5 | —    | push 12 blocks
Sticky Piston        | 0.5/0.5 | —    | pulls on retract
Observer             | 2.0/2.0 | —    | 1-tick pulse on neighbor change
Dispenser            | 3.5/3.5 | Pick | 9 slots, fires projectiles
Dropper              | 3.5/3.5 | Pick | 9 slots, ejects items
Hopper               | 3.0/3.0 | Pick | 5 slots, 4t transfer
Furnace              | 3.5/3.5 | Pick | smelting
Brewing Stand        | 0.5/0.5 | Pick | 4 bottles + ingredient
Cauldron             | 2.0/2.0 | Pick | 3 water levels
Enchanting Table     | 5.0/1200| —    | light 12, bookshelf boost
Anvil                | 5.0/1200| —    | 3 damage stages, max 39 lvl cost
Ender Chest          | 22.5/600| Pick | 27 shared, light 7
Shulker Box (16)     | 2.0/2.0 | —    | 27 portable, keeps contents
Beacon               | 3.0/3.0 | —    | light 15, pyramid effects
End Portal Frame     | ∞/3600  | —    | needs eye, 12 total
End Gateway          | ∞/3600  | —    | teleports ±1000
Cactus               | 0.4/0.4 | —    | damages touching, breaks if blocked
Sugar Cane           | 0/0     | —    | needs water, 3-4 tall
Vines                | 0.2/0.2 | —    | climbable, spreads, attaches
Lily Pad             | 0/0     | —    | walkable water surface
Cocoa                | 0.2/0.2 | Axe  | 3 stages, on jungle logs
Dead Bush            | 0/0     | —    | drops sticks
Grass/Fern           | 0/0     | —    | shears drop; tall variant
Double Tallgrass     | 0/0     | —    | 2-tall
Sunflower/Lilac/Rose Bush/Peony | 0/0 | — | 2-tall flowers
Poppy/Blue Orchid/Allium/Azure Bluet/Tulips/Oxeye Daisy | 0/0 | — | small flowers
Torch                | 0/0     | —    | light 14, wall/ground
Jack o'Lantern       | 1.0/1.0 | Axe  | light 15
Carved Pumpkin       | 1.0/1.0 | Axe  | wearable vs endermen
Pumpkin/Melon        | 1.0/1.0 | Axe  | melon drops 3-7 slices
Hay Bale             | 0.5/0.5 | —    | pillar, -80% fall dmg
Mushroom Block (brown/red/stem) | 0.2/0.2 | —    |
Brown/Red Mushroom   | 0/0     | —    | grows in dark, giant variant
Nether Wart          | 0/0     | —    | 4 stages, on soul sand
End Rod              | 0/0     | —    | light 14, 6 directions
Chorus Plant/Flower  | 0.4/0.4 | Axe  | 5 stages, end highlands
Dragon Egg           | 0/0     | —    | teleports on hit, light 1
Sponge/Wet Sponge    | 0.6/0.6 | —    | wet absorbs water
Slime Block          | 0/0     | —    | bouncy, sticky for pistons
TNT                  | 0/0     | —    | 4s fuse, power 4
Daylight Detector    | 0.2/0.2 | —    | inverted mode
Command Block        | ∞/3600  | —    | impulse/chain/repeating
Structure Block/Void | ∞/3600  | —    | save/load structures
Barrier              | ∞/3600  | —    | invisible, no collision
Mob Spawner          | 5.0/5.0 | Pick | spawns within 16 blocks of player
```

### 2.5 Liquids
```
Water (source/flow)  | 100/100 | —    | 8 levels, falls downward
Lava (source/flow)   | 100/100 | —    | light 15, flows 3 (OW)/7 (Nether)
```

### 2.6 Slabs (all with top/bottom/double)
```
Oak, Spruce, Birch, Jungle, Acacia, Dark Oak
Stone, Cobblestone, Stone Brick, Brick
Sandstone, Red Sandstone, Nether Brick, Quartz
Purpur, Prismarine, Dark Prismarine
```

### 2.7 Stairs (all with facing/shape/half)
```
Oak, Spruce, Birch, Jungle, Acacia, Dark Oak
Cobblestone, Stone Brick, Brick, Sandstone
Red Sandstone, Nether Brick, Quartz, Purpur
Prismarine, Dark Prismarine
```

### 2.8 Walls
```
Cobblestone Wall, Mossy Cobblestone Wall
```

### 2.9 Fences & Gates (6 wood + nether brick fence)
### 2.10 Doors & Trapdoors (6 wood + iron)
### 2.11 Buttons (stone + 6 wood)
### 2.12 Pressure Plates (stone + 6 wood + iron + gold weighted)
### 2.13 Rails (normal, powered, detector, activator)
### 2.14 Heads/Skulls (skeleton, wither skeleton, zombie, player, creeper, dragon)
### 2.15 Banners (16 base colors, 6+ patterns, 16 rotations)
### 2.16 Beds (16 colors, 2 blocks, sets spawn, explodes in Nether/End power 5)

### 2.17 Complete 1.12.2 Block List (Numeric ID → Resource Location)
```
Numeric | Resource Location               | Notes
ID
0       minecraft:air
1       minecraft:stone                   | variants: stone, granite, diorite, andesite
2       minecraft:grass                   | grass block
3       minecraft:dirt                    | variants: dirt, coarse_dirt, podzol
4       minecraft:cobblestone
5       minecraft:oak_planks
5:1     minecraft:spruce_planks
5:2     minecraft:birch_planks
5:3     minecraft:jungle_planks
5:4     minecraft:acacia_planks
5:5     minecraft:dark_oak_planks
6       minecraft:oak_sapling
6:1     minecraft:spruce_sapling
6:2     minecraft:birch_sapling
6:3     minecraft:jungle_sapling
6:4     minecraft:acacia_sapling
6:5     minecraft:dark_oak_sapling
7       minecraft:bedrock
8       minecraft:water                    | flowing water
9       minecraft:water                    | stationary water (source)
10      minecraft:lava                     | flowing lava
11      minecraft:lava                     | stationary lava (source)
12      minecraft:sand                     | variants: sand, red_sand
13      minecraft:gravel
14      minecraft:gold_ore
15      minecraft:iron_ore
16      minecraft:coal_ore
17      minecraft:oak_log                  | axis variants
17:1    minecraft:spruce_log
17:2    minecraft:birch_log
17:3    minecraft:jungle_log
18      minecraft:oak_leaves               | decay distance + check_decay
18:1    minecraft:spruce_leaves
18:2    minecraft:birch_leaves
18:3    minecraft:jungle_leaves
19      minecraft:sponge                    | variants: sponge, wet_sponge
20      minecraft:glass
21      minecraft:lapis_ore
22      minecraft:lapis_block
23      minecraft:dispenser
24      minecraft:sandstone                 | base block
24:1    minecraft:chiseled_sandstone        | chiseled variant
24:2    minecraft:cut_sandstone             | cut variant
24:3    minecraft:smooth_sandstone          | smooth variant
25      minecraft:noteblock
26      minecraft:bed
27      minecraft:golden_rail               | powered rail
28      minecraft:detector_rail
29      minecraft:sticky_piston
30      minecraft:cobweb
31      minecraft:tall_grass                | variants: dead_shrub, tall_grass, fern
32      minecraft:dead_bush
33      minecraft:piston
34      minecraft:piston_head
35      minecraft:wool                      | 16 colors (white=0, orange=1, ... black=15)
36      minecraft:piston_extension
37      minecraft:yellow_flower             | dandelion
38      minecraft:red_flower                | poppy=0, blue_orchid=1, allium=2, etc.
39      minecraft:brown_mushroom
40      minecraft:red_mushroom
41      minecraft:gold_block
42      minecraft:iron_block
43      minecraft:double_stone_slab            | (stone=0, sandstone=1, wood_old=2, cobblestone=3, brick=4, stone_brick=5, nether_brick=6, quartz=7)
43:1    minecraft:sandstone_double_slab
43:3    minecraft:cobblestone_double_slab
43:4    minecraft:brick_double_slab
43:5    minecraft:stone_brick_double_slab
43:6    minecraft:nether_brick_double_slab
43:7    minecraft:quartz_double_slab
44      minecraft:stone_slab                    | (stone=0, sandstone=1, wood_old=2, cobblestone=3, brick=4, stone_brick=5, nether_brick=6, quartz=7)
44:1    minecraft:sandstone_slab
44:3    minecraft:cobblestone_slab
44:4    minecraft:brick_slab
44:5    minecraft:stone_brick_slab
44:6    minecraft:nether_brick_slab
44:7    minecraft:quartz_slab
45      minecraft:brick_block
46      minecraft:tnt
47      minecraft:bookshelf
48      minecraft:mossy_cobblestone
49      minecraft:obsidian
50      minecraft:torch
51      minecraft:fire
52      minecraft:mob_spawner
53      minecraft:oak_stairs
54      minecraft:chest
55      minecraft:redstone_wire
56      minecraft:diamond_ore
57      minecraft:diamond_block
58      minecraft:crafting_table
59      minecraft:wheat                      | crops (7 stages)
60      minecraft:farmland
61      minecraft:furnace
62      minecraft:lit_furnace
63      minecraft:standing_sign
64      minecraft:wooden_door
65      minecraft:ladder
66      minecraft:rail
67      minecraft:stone_stairs
68      minecraft:wall_sign
69      minecraft:lever
70      minecraft:stone_pressure_plate
71      minecraft:iron_door
72      minecraft:wooden_pressure_plate
73      minecraft:redstone_ore
74      minecraft:lit_redstone_ore
75      minecraft:unlit_redstone_torch
76      minecraft:redstone_torch
77      minecraft:stone_button
78      minecraft:snow_layer
79      minecraft:ice
80      minecraft:snow
81      minecraft:cactus
82      minecraft:clay
83      minecraft:reeds                      | sugar cane
84      minecraft:jukebox
85      minecraft:fence
86      minecraft:pumpkin
87      minecraft:netherrack
88      minecraft:soul_sand
89      minecraft:glowstone
90      minecraft:portal
91      minecraft:lit_pumpkin
92      minecraft:cake
93      minecraft:unpowered_repeater
94      minecraft:powered_repeater
95      minecraft:stained_glass              | 16 colors
96      minecraft:trapdoor
97      minecraft:monster_egg                | infested stone variants (5)
98      minecraft:stonebrick                 | stone brick variants (5)
99      minecraft:brown_mushroom_block
100     minecraft:red_mushroom_block
101     minecraft:iron_bars
102     minecraft:glass_pane
103     minecraft:melon_block
104     minecraft:pumpkin_stem
105     minecraft:melon_stem
106     minecraft:vine
107     minecraft:fence_gate
108     minecraft:brick_stairs
109     minecraft:stone_brick_stairs
110     minecraft:mycelium
111     minecraft:waterlily                  | lily pad
112     minecraft:nether_brick
113     minecraft:nether_brick_fence
114     minecraft:nether_brick_stairs
115     minecraft:nether_wart
116     minecraft:enchanting_table
117     minecraft:brewing_stand
118     minecraft:cauldron
119     minecraft:end_portal
120     minecraft:end_portal_frame
121     minecraft:end_stone
122     minecraft:dragon_egg
123     minecraft:redstone_lamp
124     minecraft:lit_redstone_lamp
125     minecraft:double_wooden_slab            | (oak=0, spruce=1, birch=2, jungle=3, acacia=4, dark_oak=5)
125:0   minecraft:oak_double_slab
125:1   minecraft:spruce_double_slab
125:2   minecraft:birch_double_slab
125:3   minecraft:jungle_double_slab
125:4   minecraft:acacia_double_slab
125:5   minecraft:dark_oak_double_slab
126     minecraft:wooden_slab                   | (oak=0, spruce=1, birch=2, jungle=3, acacia=4, dark_oak=5)
126:0   minecraft:oak_slab
126:1   minecraft:spruce_slab
126:2   minecraft:birch_slab
126:3   minecraft:jungle_slab
126:4   minecraft:acacia_slab
126:5   minecraft:dark_oak_slab
127     minecraft:cocoa
128     minecraft:sandstone_stairs
129     minecraft:emerald_ore
130     minecraft:ender_chest
131     minecraft:tripwire_hook
132     minecraft:tripwire
133     minecraft:emerald_block
134     minecraft:spruce_stairs
135     minecraft:birch_stairs
136     minecraft:jungle_stairs
137     minecraft:command_block
138     minecraft:beacon
139     minecraft:cobblestone_wall           | normal + mossy
140     minecraft:flower_pot
141     minecraft:carrots
142     minecraft:potatoes
143     minecraft:wooden_button
144     minecraft:skull                      | heads (skeleton, wither, zombie, player, creeper, dragon)
145     minecraft:anvil                      | 3 damage stages
146     minecraft:trapped_chest
147     minecraft:light_weighted_pressure_plate
148     minecraft:heavy_weighted_pressure_plate
149     minecraft:unpowered_comparator
150     minecraft:powered_comparator
151     minecraft:daylight_detector
152     minecraft:redstone_block
153     minecraft:nether_quartz_ore
154     minecraft:hopper
155     minecraft:quartz_block               | base block
155:1   minecraft:chiseled_quartz_block      | chiseled variant
155:2   minecraft:quartz_column              | pillar variant (also known as quartz_pillar)
156     minecraft:quartz_stairs
157     minecraft:activator_rail
158     minecraft:dropper
159     minecraft:stained_hardened_clay       | terracotta, 16 colors
160     minecraft:stained_glass_pane          | 16 colors
161     minecraft:leaves2                     | acacia + dark oak leaves
162     minecraft:log2                        | acacia + dark oak logs
163     minecraft:acacia_stairs
164     minecraft:dark_oak_stairs
165     minecraft:slime
166     minecraft:barrier
167     minecraft:iron_trapdoor
168     minecraft:prismarine                  | prismarine, prismarine_bricks, dark_prismarine
169     minecraft:sea_lantern
170     minecraft:hay_block
171     minecraft:carpet                      | 16 colors
172     minecraft:hardened_clay               | terracotta (base)
173     minecraft:coal_block
174     minecraft:packed_ice
175     minecraft:double_plant                | 2-tall flowers (sunflower, lilac, etc.)
176     minecraft:standing_banner
177     minecraft:wall_banner
178     minecraft:daylight_detector_inverted
179     minecraft:red_sandstone               | base block
179:1   minecraft:chiseled_red_sandstone      | chiseled variant
179:2   minecraft:smooth_red_sandstone        | smooth variant
180     minecraft:red_sandstone_stairs
181     minecraft:double_red_sandstone_slab
182     minecraft:red_sandstone_slab
183     minecraft:spruce_fence_gate
184     minecraft:birch_fence_gate
185     minecraft:jungle_fence_gate
186     minecraft:dark_oak_fence_gate
187     minecraft:acacia_fence_gate
188     minecraft:spruce_fence
189     minecraft:birch_fence
190     minecraft:jungle_fence
191     minecraft:dark_oak_fence
192     minecraft:acacia_fence
193     minecraft:spruce_door
194     minecraft:birch_door
195     minecraft:jungle_door
196     minecraft:acacia_door
197     minecraft:dark_oak_door
198     minecraft:end_rod
199     minecraft:chorus_plant
200     minecraft:chorus_flower
201     minecraft:purpur_block
202     minecraft:purpur_pillar
203     minecraft:purpur_stairs
204     minecraft:purpur_double_slab
205     minecraft:purpur_slab
206     minecraft:end_bricks
207     minecraft:beetroots
208     minecraft:grass_path
209     minecraft:end_gateway
210     minecraft:repeating_command_block
211     minecraft:chain_command_block
212     minecraft:frosted_ice
213     minecraft:magma
214     minecraft:nether_wart_block
215     minecraft:red_nether_brick
216     minecraft:bone_block
217     minecraft:structure_void
218     minecraft:observer
219     minecraft:white_shulker_box           | 16 colors
220     minecraft:orange_shulker_box
221     minecraft:magenta_shulker_box
222     minecraft:light_blue_shulker_box
223     minecraft:yellow_shulker_box
224     minecraft:lime_shulker_box
225     minecraft:pink_shulker_box
226     minecraft:gray_shulker_box
227     minecraft:silver_shulker_box          | light gray
228     minecraft:cyan_shulker_box
229     minecraft:purple_shulker_box
230     minecraft:blue_shulker_box
231     minecraft:brown_shulker_box
232     minecraft:green_shulker_box
233     minecraft:red_shulker_box
234     minecraft:black_shulker_box
235     minecraft:white_glazed_terracotta     | 16 colors (1.12 NEW)
236     minecraft:orange_glazed_terracotta
237     minecraft:magenta_glazed_terracotta
238     minecraft:light_blue_glazed_terracotta
239     minecraft:yellow_glazed_terracotta
240     minecraft:lime_glazed_terracotta
241     minecraft:pink_glazed_terracotta
242     minecraft:gray_glazed_terracotta
243     minecraft:silver_glazed_terracotta
244     minecraft:cyan_glazed_terracotta
245     minecraft:purple_glazed_terracotta
246     minecraft:blue_glazed_terracotta
247     minecraft:brown_glazed_terracotta
248     minecraft:green_glazed_terracotta
249     minecraft:red_glazed_terracotta
250     minecraft:black_glazed_terracotta
251     minecraft:white_concrete              | 16 colors (1.12 NEW)
252     minecraft:orange_concrete
253     minecraft:magenta_concrete
254     minecraft:light_blue_concrete
255     minecraft:yellow_concrete
256     minecraft:lime_concrete
257     minecraft:pink_concrete
258     minecraft:gray_concrete
259     minecraft:silver_concrete
260     minecraft:cyan_concrete
261     minecraft:purple_concrete
262     minecraft:blue_concrete
263     minecraft:brown_concrete
264     minecraft:green_concrete
265     minecraft:red_concrete
266     minecraft:black_concrete
267     minecraft:white_concrete_powder        | 16 colors (1.12 NEW)
268     minecraft:orange_concrete_powder
269     minecraft:magenta_concrete_powder
270     minecraft:light_blue_concrete_powder
271     minecraft:yellow_concrete_powder
272     minecraft:lime_concrete_powder
273     minecraft:pink_concrete_powder
274     minecraft:gray_concrete_powder
275     minecraft:silver_concrete_powder
276     minecraft:cyan_concrete_powder
277     minecraft:purple_concrete_powder
278     minecraft:blue_concrete_powder
279     minecraft:brown_concrete_powder
280     minecraft:green_concrete_powder
281     minecraft:red_concrete_powder
282     minecraft:black_concrete_powder
283     minecraft:white_carpet                 | duplicate — metadata variant for block ID 171
284     minecraft:orange_carpet
     ...  (all 16 colors via metadata on ID 171, same for wool ID 35)

Non-ID blocks (placed via block entities / special handling):
      minecraft:air                         | ID 0
      minecraft:flowing_water               | ID 8
      minecraft:water                        | ID 9
      minecraft:flowing_lava                 | ID 10
      minecraft:lava                         | ID 11
      minecraft:double_stone_slab2           | red sandstone double slab
      minecraft:double_stone_slab3           | purpur double slab
      minecraft:double_stone_slab4           | prismarine/etc double slab
      minecraft:moving_piston                | piston_extension during movement
      minecraft:structure_block              | ID 255 (save/load/corner/data)
      minecraft:structure_void               | ID 217
      minecraft:item_frame                   | entity (item model reference)

Metadata variant blocks (explicit minecraft:xxx for blockstate JSON lookup):
      minecraft:allium                       | ID 38:2 (red_flower)
      minecraft:andesite                     | ID 1:3 (stone)
      minecraft:blue_orchid                  | ID 38:1 (red_flower)
      minecraft:coarse_dirt                  | ID 3:1 (dirt)
      minecraft:cobblestone_monster_egg      | ID 97:1 (monster_egg)
      minecraft:dandelion                    | ID 37:0 (yellow_flower)
      minecraft:dark_prismarine              | ID 168:2 (prismarine)
      minecraft:diorite                      | ID 1:2 (stone)
      minecraft:fern                         | ID 31:2 (tallgrass)
      minecraft:granite                      | ID 1:1 (stone)
      minecraft:orange_tulip                 | ID 38:5 (red_flower)
      minecraft:oxeye_daisy                  | ID 38:8 (red_flower)
      minecraft:paeonia                      | ID 175:4 (double_plant) — peony
      minecraft:pink_tulip                   | ID 38:7 (red_flower)
      minecraft:podzol                       | ID 3:2 (dirt)
      minecraft:poppy                        | ID 38:0 (red_flower)
      minecraft:prismarine_bricks            | ID 168:1 (prismarine)
      minecraft:quartz_ore                   | ID 153 (nether_quartz_ore blockstate alias)
      minecraft:red_sand                     | ID 12:1 (sand)
      minecraft:red_tulip                    | ID 38:4 (red_flower)
      minecraft:smooth_andesite              | ID 1:6 (stone) — polished
      minecraft:smooth_diorite               | ID 1:5 (stone) — polished
      minecraft:smooth_granite               | ID 1:4 (stone) — polished
      minecraft:sunflower                    | ID 175:0 (double_plant)
      minecraft:white_tulip                  | ID 38:6 (red_flower)

Complete blockstate inventory (all 407 files):
  acacia_door
  acacia_double_slab
  acacia_fence
  acacia_fence_gate
  acacia_leaves
  acacia_log
  acacia_planks
  acacia_sapling
  acacia_slab
  acacia_stairs
  activator_rail
  allium
  andesite
  anvil
  beacon
  bedrock
  beetroots
  birch_door
  birch_double_slab
  birch_fence
  birch_fence_gate
  birch_leaves
  birch_log
  birch_planks
  birch_sapling
  birch_slab
  birch_stairs
  black_carpet
  black_concrete
  black_concrete_powder
  black_glazed_terracotta
  black_stained_glass
  black_stained_glass_pane
  black_stained_hardened_clay
  black_wool
  blue_carpet
  blue_concrete
  blue_concrete_powder
  blue_glazed_terracotta
  blue_orchid
  blue_stained_glass
  blue_stained_glass_pane
  blue_stained_hardened_clay
  blue_wool
  bone_block
  bookshelf
  brewing_stand
  brick_block
  brick_double_slab
  brick_slab
  brick_stairs
  brown_carpet
  brown_concrete
  brown_concrete_powder
  brown_glazed_terracotta
  brown_mushroom
  brown_mushroom_block
  brown_stained_glass
  brown_stained_glass_pane
  brown_stained_hardened_clay
  brown_wool
  cactus
  cake
  carrots
  cauldron
  chain_command_block
  chiseled_brick_monster_egg
  chiseled_quartz_block
  chiseled_red_sandstone
  chiseled_sandstone
  chiseled_stonebrick
  chorus_flower
  chorus_plant
  clay
  coal_block
  coal_ore
  coarse_dirt
  cobblestone
  cobblestone_double_slab
  cobblestone_monster_egg
  cobblestone_slab
  cobblestone_wall
  cocoa
  command_block
  cracked_brick_monster_egg
  cracked_stonebrick
  crafting_table
  cyan_carpet
  cyan_concrete
  cyan_concrete_powder
  cyan_glazed_terracotta
  cyan_stained_glass
  cyan_stained_glass_pane
  cyan_stained_hardened_clay
  cyan_wool
  dandelion
  dark_oak_door
  dark_oak_double_slab
  dark_oak_fence
  dark_oak_fence_gate
  dark_oak_leaves
  dark_oak_log
  dark_oak_planks
  dark_oak_sapling
  dark_oak_slab
  dark_oak_stairs
  dark_prismarine
  daylight_detector
  daylight_detector_inverted
  dead_bush
  detector_rail
  diamond_block
  diamond_ore
  diorite
  dirt
  dispenser
  double_fern
  double_grass
  double_rose
  dragon_egg
  dropper
  emerald_block
  emerald_ore
  enchanting_table
  end_bricks
  end_portal_frame
  end_rod
  end_stone
  farmland
  fence
  fence_gate
  fern
  fire
  flower_pot
  frosted_ice
  furnace
  glass
  glass_pane
  glowstone
  gold_block
  gold_ore
  golden_rail
  granite
  grass
  grass_path
  gravel
  gray_carpet
  gray_concrete
  gray_concrete_powder
  gray_glazed_terracotta
  gray_stained_glass
  gray_stained_glass_pane
  gray_stained_hardened_clay
  gray_wool
  green_carpet
  green_concrete
  green_concrete_powder
  green_glazed_terracotta
  green_stained_glass
  green_stained_glass_pane
  green_stained_hardened_clay
  green_wool
  hardened_clay
  hay_block
  heavy_weighted_pressure_plate
  hopper
  houstonia
  ice
  iron_bars
  iron_block
  iron_door
  iron_ore
  iron_trapdoor
  item_frame
  jukebox
  jungle_door
  jungle_double_slab
  jungle_fence
  jungle_fence_gate
  jungle_leaves
  jungle_log
  jungle_planks
  jungle_sapling
  jungle_slab
  jungle_stairs
  ladder
  lapis_block
  lapis_ore
  lever
  light_blue_carpet
  light_blue_concrete
  light_blue_concrete_powder
  light_blue_glazed_terracotta
  light_blue_stained_glass
  light_blue_stained_glass_pane
  light_blue_stained_hardened_clay
  light_blue_wool
  light_weighted_pressure_plate
  lime_carpet
  lime_concrete
  lime_concrete_powder
  lime_glazed_terracotta
  lime_stained_glass
  lime_stained_glass_pane
  lime_stained_hardened_clay
  lime_wool
  lit_furnace
  lit_pumpkin
  lit_redstone_lamp
  lit_redstone_ore
  magenta_carpet
  magenta_concrete
  magenta_concrete_powder
  magenta_glazed_terracotta
  magenta_stained_glass
  magenta_stained_glass_pane
  magenta_stained_hardened_clay
  magenta_wool
  magma
  melon_block
  melon_stem
  mob_spawner
  mossy_brick_monster_egg
  mossy_cobblestone
  mossy_cobblestone_wall
  mossy_stonebrick
  mycelium
  nether_brick
  nether_brick_double_slab
  nether_brick_fence
  nether_brick_slab
  nether_brick_stairs
  nether_wart
  nether_wart_block
  netherrack
  noteblock
  oak_double_slab
  oak_leaves
  oak_log
  oak_planks
  oak_sapling
  oak_slab
  oak_stairs
  observer
  obsidian
  orange_carpet
  orange_concrete
  orange_concrete_powder
  orange_glazed_terracotta
  orange_stained_glass
  orange_stained_glass_pane
  orange_stained_hardened_clay
  orange_tulip
  orange_wool
  oxeye_daisy
  packed_ice
  paeonia
  pink_carpet
  pink_concrete
  pink_concrete_powder
  pink_glazed_terracotta
  pink_stained_glass
  pink_stained_glass_pane
  pink_stained_hardened_clay
  pink_tulip
  pink_wool
  piston
  piston_head
  podzol
  poppy
  portal
  potatoes
  powered_comparator
  powered_repeater
  prismarine
  prismarine_bricks
  pumpkin
  pumpkin_stem
  purple_carpet
  purple_concrete
  purple_concrete_powder
  purple_glazed_terracotta
  purple_stained_glass
  purple_stained_glass_pane
  purple_stained_hardened_clay
  purple_wool
  purpur_block
  purpur_double_slab
  purpur_pillar
  purpur_slab
  purpur_stairs
  quartz_block
  quartz_column
  quartz_double_slab
  quartz_ore
  quartz_slab
  quartz_stairs
  rail
  red_carpet
  red_concrete
  red_concrete_powder
  red_glazed_terracotta
  red_mushroom
  red_mushroom_block
  red_nether_brick
  red_sand
  red_sandstone
  red_sandstone_double_slab
  red_sandstone_slab
  red_sandstone_stairs
  red_stained_glass
  red_stained_glass_pane
  red_stained_hardened_clay
  red_tulip
  red_wool
  redstone_block
  redstone_lamp
  redstone_ore
  redstone_torch
  redstone_wire
  reeds
  repeating_command_block
  sand
  sandstone
  sandstone_double_slab
  sandstone_slab
  sandstone_stairs
  sea_lantern
  silver_carpet
  silver_concrete
  silver_concrete_powder
  silver_glazed_terracotta
  silver_stained_glass
  silver_stained_glass_pane
  silver_stained_hardened_clay
  silver_wool
  slime
  smooth_andesite
  smooth_diorite
  smooth_granite
  smooth_red_sandstone
  smooth_sandstone
  snow
  snow_layer
  soul_sand
  sponge
  spruce_door
  spruce_double_slab
  spruce_fence
  spruce_fence_gate
  spruce_leaves
  spruce_log
  spruce_planks
  spruce_sapling
  spruce_slab
  spruce_stairs
  sticky_piston
  stone
  stone_brick_double_slab
  stone_brick_monster_egg
  stone_brick_slab
  stone_brick_stairs
  stone_button
  stone_double_slab
  stone_monster_egg
  stone_pressure_plate
  stone_slab
  stone_stairs
  stonebrick
  structure_block
  sunflower
  syringa
  tall_grass
  tnt
  torch
  trapdoor
  tripwire
  tripwire_hook
  unlit_redstone_torch
  unpowered_comparator
  unpowered_repeater
  vine
  waterlily
  web
  wheat
  white_carpet
  white_concrete
  white_concrete_powder
  white_glazed_terracotta
  white_stained_glass
  white_stained_glass_pane
  white_stained_hardened_clay
  white_tulip
  white_wool
  wood_old_double_slab
  wood_old_slab
  wooden_button
  wooden_door
  wooden_pressure_plate
  yellow_carpet
  yellow_concrete
  yellow_concrete_powder
  yellow_glazed_terracotta
  yellow_stained_glass
  yellow_stained_glass_pane
  yellow_stained_hardened_clay
  yellow_wool

Total: 407 unique blockstate JSON files in @minecraft/blockstates/.
```

---

## P3 — Complete Item Registry

### 3.1 Tools & Weapons
```
Item               | Speed | Damage | Durability | Enchant
Wooden Sword       | 1.6   | 4      | 60         | 15
Stone Sword        | 1.6   | 5      | 132        | 5
Iron Sword         | 1.6   | 6      | 251        | 14
Golden Sword       | 1.6   | 4      | 33         | 22
Diamond Sword      | 1.6   | 7      | 1562       | 10
Wooden Shovel      | 2.0   | 2.5    | 60         | 15
Stone Shovel       | 2.0   | 3.5    | 132        | 5
Iron Shovel        | 2.0   | 4.5    | 251        | 14
Golden Shovel      | 2.0   | 2.5    | 33         | 22
Diamond Shovel     | 2.0   | 5.5    | 1562       | 10
Wooden Pickaxe     | 1.2   | 2.5    | 60         | 15
Stone Pickaxe      | 1.2   | 3.5    | 132        | 5
Iron Pickaxe       | 1.2   | 4.5    | 251        | 14
Golden Pickaxe     | 1.2   | 2.5    | 33         | 22
Diamond Pickaxe    | 1.2   | 5.5    | 1562       | 10
Wooden Axe         | 0.8   | 7      | 60         | 15
Stone Axe          | 0.8   | 9      | 132        | 5
Iron Axe           | 0.8   | 9      | 251        | 14
Golden Axe         | 0.8   | 7      | 33         | 22
Diamond Axe        | 0.8   | 9      | 1562       | 10
Wooden Hoe         | 1.0   | 1      | 60         | 15
Stone Hoe          | 1.0   | 1      | 132        | 5
Iron Hoe           | 1.0   | 1      | 251        | 14
Golden Hoe         | 1.0   | 1      | 33         | 22
Diamond Hoe        | 1.0   | 1      | 1562       | 10
Flint & Steel      | —     | —      | 65         | —   ignites
Shears             | —     | —      | 238        | —   shear sheep/leaves
Fishing Rod        | —     | —      | 65         | —   + carrot on stick
Bow                | —     | —      | 385        | —   3 charge tiers
Shield             | —     | —      | 336        | —   180° block, 5s axe disable
Elytra             | —     | —      | 432        | —   glide, durability/s
```

### 3.2 Armor
```
Piece        | Leather | Gold | Chain | Iron | Diamond | Toughness | Enchant
Helmet       | 1.5     | 1.5  | 1.5   | 1.5  | 1.5     | 2         | varies
Chestplate   | 4       | 4    | 4     | 4    | 4       | 2         |
Leggings     | 3       | 3    | 3     | 3    | 3       | 2         |
Boots        | 1.5     | 1.5  | 1.5   | 1.5  | 1.5     | 2         |
Total        | 10      | 10   | 10    | 10   | 10      | 8         |
Durability   | 55/80/75/65 | 77/112/105/91 | 165/240/225/195 | 165/240/225/195 | 363/528/495/429
```

### 3.3 Food (see P1.4 — 30+ items with exact hunger/saturation)
### 3.4 Brewing Ingredients (nether wart, blaze powder, glistering melon, spider eye, ghast tear, magma cream, sugar, rabbit foot, glowstone, redstone, fermented spider eye, gunpowder, dragon's breath)
### 3.5 Potions (fire res, health, damage, poison, regen, strength, swiftness, slowness, water breathing, night vision, invisibility, weakness, leaping — all with extended/II/splash/lingering variants)

### 3.5B Tipped Arrows (1.9+)
```
Tipped Arrow variant for every potion effect (×13 base effects):
  Arrow of Fire Resistance    (3:00 / 8:00)
  Arrow of Healing            (2 HP / 4 HP)
  Arrow of Harming            (3 HP / 6 HP)
  Arrow of Poison             (0:11 / 0:22 / 0:05 II)
  Arrow of Regeneration       (0:11 / 0:22 / 0:05 II)
  Arrow of Strength           (3:00 / 8:00 / 1:30 II)
  Arrow of Swiftness          (3:00 / 8:00 / 1:30 II)
  Arrow of Slowness           (1:30 / 4:00)
  Arrow of Water Breathing    (3:00 / 8:00)
  Arrow of Night Vision       (3:00 / 8:00)
  Arrow of Invisibility       (3:00 / 8:00)
  Arrow of Weakness           (1:30 / 4:00)
  Arrow of Leaping            (3:00 / 8:00 / 1:30 II)
  Arrow of Luck               (5:00)
Total tipped arrow variants: ~50+ (base + extended + II for each)
Spectral Arrow: glows target (10s glow effect, 10s duration); crafted with glowstone dust
Crafting: 8 arrows + 1 lingering potion → 8 tipped arrows of that type
```
### 3.6 Enchantments (complete 1.12.2: 27 enchantments across weapons/tools/armor/bows — see below for details)
### 3.7 Miscellaneous Items (coal, charcoal, diamond, ingots, nuggets, sticks, bowls, string, feathers, leather, paper, books, slime balls, bones, ender pearls, eye of ender, blaze rods, ghast tears, magma cream, nether star, end crystal, fireworks, spawn eggs, music discs ×12, minecarts ×5, boat, armor stand, shields, horse armor, buckets, maps, compasses, clocks, saddles, leads, name tags, spawn eggs for all 1.12.2 mobs)

### 3.9 1.12 New Item: Knowledge Book
```
Knowledge Book (minecraft:knowledge_book):
  - NEW in 1.12. Creative-only / command-only item.
  - Usage: right-click to unlock specific recipes for the player.
  - Stores a list of recipe IDs in its NBT data:
    {Recipes:["minecraft:oak_planks","minecraft:stick",...]}
  - When used, each recipe in the list is unlocked for that player.
  - Does NOT grant XP or consume itself. Disappears after use (but clear recipe unlock).
  - Primarily for adventure maps, data packs, and custom game modes.
  - Not obtainable in survival — only via /give or Creative inventory.
```

### 3.8 Enchantment Details
```
Protection         IV | Armor       | -4% dmg/lvl all
Fire Protection    IV | Armor       | -8% fire dmg/lvl
Feather Falling    IV | Boots       | -8%×lvl² fall dmg
Blast Protection   IV | Armor       | -8% explosion dmg/lvl
Projectile Prot    IV | Armor       | -8% projectile dmg/lvl
Respiration       III | Helmet      | +15s underwater/lvl
Aqua Affinity       I | Helmet      | normal mining underwater
Thorns            III | Chestplate  | 15%/lvl deal 1-4 dmg
Depth Strider     III | Boots       | +33% speed/lvl underwater
Frost Walker       II | Boots       | freeze water; incompatible w/ Depth Strider
Curse Binding       I | Any armor   | cannot remove
Curse Vanishing     I | Any         | vanishes on death
Sharpness           V | Sword       | +1 dmg/lvl (+1.25)
Smite               V | Sword       | +2.5 dmg/lvl undead
Bane of Arthro     V | Sword       | +2.5 dmg/lvl arthropods
Knockback          II | Sword       | knockback distance
Fire Aspect        II | Sword       | 4s fire/lvl
Looting           III | Sword       | +1 max drops/lvl
Efficiency          V | Tools       | ×(1+lvl²) mining speed
Silk Touch          I | Tools       | drops block itself
Fortune            III | Tools       | multiplies drops
Unbreaking         III | Any         | 100/(lvl+1)% durability use
Power               V | Bow         | +25% dmg/lvl
Punch              II | Bow         | knockback/lvl
Flame               I | Bow         | 4s fire on arrows
Infinity            I | Bow         | infinite arrows (need 1)
Luck of Sea        III | Fishing Rod | better loot
Lure               III | Fishing Rod | faster bites
Mending             I | Any         | XP orbs repair (chosen randomly)
```

---

## P4 — World Generation

### 4.1 Biomes (1.12.2: 57 total)
```
Numeric ID | Name                  | Resource Location              | Category
0          | Ocean                 | ocean                          | OCEAN
1          | Plains                | plains                         | PLAINS
2          | Desert                | desert                         | DESERT
3          | Extreme Hills         | extreme_hills                  | EXTREME_HILLS
4          | Forest                | forest                         | FOREST
5          | Taiga                 | taiga                          | TAIGA
6          | Swampland             | swampland                      | SWAMPLAND
7          | River                 | river                          | RIVER
8          | Hell (Nether)         | hell                           | HELL
9          | The End (Sky)         | sky                            | SKY
10         | Frozen Ocean          | frozen_ocean                   | FROZEN_OCEAN
11         | Frozen River          | frozen_river                   | FROZEN_RIVER
12         | Ice Plains            | ice_flats                      | ICE_FLATS
13         | Ice Mountains         | ice_mountains                  | ICE_MOUNTAINS
14         | Mushroom Island       | mushroom_island                | MUSHROOM_ISLAND
15         | Mushroom Island Shore| mushroom_island_shore          | MUSHROOM_ISLAND_SHORE
16         | Beach                 | beaches                        | BEACHES
17         | Desert Hills          | desert_hills                   | DESERT_HILLS
18         | Forest Hills          | forest_hills                   | FOREST_HILLS
19         | Taiga Hills           | taiga_hills                    | TAIGA_HILLS
20         | Extreme Hills Edge    | smaller_extreme_hills          | SMALLER_EXTREME_HILLS
21         | Jungle                | jungle                         | JUNGLE
22         | Jungle Hills          | jungle_hills                   | JUNGLE_HILLS
23         | Jungle Edge           | jungle_edge                    | JUNGLE_EDGE
24         | Deep Ocean            | deep_ocean                     | DEEP_OCEAN
25         | Stone Beach           | stone_beach                    | STONE_BEACH
26         | Cold Beach            | cold_beach                     | COLD_BEACH
27         | Birch Forest          | birch_forest                   | BIRCH_FOREST
28         | Birch Forest Hills    | birch_forest_hills             | BIRCH_FOREST_HILLS
29         | Roofed Forest         | roofed_forest                  | ROOFED_FOREST
30         | Cold Taiga            | taiga_cold                     | TAIGA_COLD
31         | Cold Taiga Hills      | taiga_cold_hills               | TAIGA_COLD_HILLS
32         | Mega Taiga            | redwood_taiga                  | REDWOOD_TAIGA
33         | Mega Taiga Hills      | redwood_taiga_hills            | REDWOOD_TAIGA_HILLS
34         | Extreme Hills+        | extreme_hills_with_trees       | EXTREME_HILLS_WITH_TREES
35         | Savanna               | savanna                        | SAVANNA
36         | Savanna Plateau       | savanna_rock                   | SAVANNA_ROCK
37         | Mesa                  | mesa                           | MESA
38         | Mesa Plateau F        | mesa_rock                      | MESA_ROCK
39         | Mesa Plateau          | mesa_clear_rock                | MESA_CLEAR_ROCK
40         | Void                  | void                           | VOID

=== Mutated (M) variants (rare sub-biomes) ===
129        | Sunflower Plains      | mutated_plains                 | MUTATED_PLAINS
130        | Desert M              | mutated_desert                 | MUTATED_DESERT
131        | Extreme Hills M       | mutated_extreme_hills          | MUTATED_EXTREME_HILLS
132        | Flower Forest         | mutated_forest                 | MUTATED_FOREST
133        | Taiga M               | mutated_taiga                  | MUTATED_TAIGA
134        | Swampland M           | mutated_swampland              | MUTATED_SWAMPLAND
140        | Ice Plains Spikes     | mutated_ice_flats              | MUTATED_ICE_FLATS
149        | Jungle M              | mutated_jungle                 | MUTATED_JUNGLE
151        | Jungle Edge M         | mutated_jungle_edge            | MUTATED_JUNGLE_EDGE
155        | Birch Forest M        | mutated_birch_forest           | MUTATED_BIRCH_FOREST
156        | Birch Forest Hills M  | mutated_birch_forest_hills     | MUTATED_BIRCH_FOREST_HILLS
157        | Roofed Forest M       | mutated_roofed_forest          | MUTATED_ROOFED_FOREST
158        | Cold Taiga M          | mutated_taiga_cold             | MUTATED_TAIGA_COLD
160        | Mega Spruce Taiga     | mutated_redwood_taiga          | MUTATED_REDWOOD_TAIGA
161        | Mega Spruce Taiga Hills| mutated_redwood_taiga_hills   | MUTATED_REDWOOD_TAIGA_HILLS
162        | Extreme Hills+ M      | mutated_extreme_hills_with_trees| MUTATED_EXTREME_HILLS_WITH_TREES
163        | Savanna M             | mutated_savanna                | MUTATED_SAVANNA
164        | Savanna Plateau M     | mutated_savanna_rock           | MUTATED_SAVANNA_ROCK
165        | Mesa (Bryce) M        | mutated_mesa                   | MUTATED_MESA
166        | Mesa Plateau F M      | mutated_mesa_rock              | MUTATED_MESA_ROCK
167        | Mesa Plateau M        | mutated_mesa_clear_rock        | MUTATED_MESA_CLEAR_ROCK
```

### 4.2 Terrain Gen Algorithm
```
Heightmap: Perlin/Simplex noise (6-8 octaves)
  - Continentalness (low freq)
  - Erosion (medium freq, carves hills/valleys)
  - Peaks/Valleys (high freq, local detail)
  - Temperature + Humidity for biome selection

Cave generation: 2+ noise passes
  - Tunnel carver: y=8..128
  - Room carver: larger chambers at lower depth
  - Ravine: y=10..90, up to 30 wide, long

Ore distribution per chunk:
  Coal:      y=0..128,  20 veins, 17 size
  Iron:      y=0..63,   20 veins, 9 size
  Gold:      y=0..31,   2 veins,  9 size
  Lapis:     y=0..30,   1 vein,   7 size, conical
  Redstone:  y=0..15,   8 veins,  8 size
  Diamond:   y=0..15,   1 vein,   8 size, exponential
  Emerald:   y=4..31,   1 vein,   1 size, Extreme Hills only
  Dirt:      y=0..255,  10 veins, 33 size
  Gravel:    y=0..255,  8 veins,  33 size
  Andesite/Diorite/Granite: y=0..79 (1.12 new!)

Lakes: water y≤63, 4/chunk; lava y≤20, 1/chunk
```

### 4.3 Structures
```
Village:        Plains/Savanna/Taiga/Desert; roads, houses, farms, blacksmith + 20% zombie
Mineshaft:      All biomes; tunnels, supports, rails, chests, spider spawners
Stronghold:     3/world, y=0..127, rings; portal room, library, prison
Desert Temple:  Desert; 4 chests (TNT trap), 4 towers
Jungle Temple:  Jungle; puzzle levers, dispenser trap, chests
Witch Hut:      Swampland; witch + cat, flower pot with mushroom
Ocean Monument: Deep Ocean; guardians, elder guardian, gold blocks, sponge
Igloo:          Ice Plains/Cold Taiga; basement with brewing, zombie villager
Woodland Mansion: Roofed Forest; evokers, vindicators, secret rooms
Nether Fortress: Nether; blaze spawners, nether wart, chests, wither skeles
End City:       End Highlands; towers, shulkers, elytra ship
End Gateway:    20 gates after dragon death; teleport ±1000
```

### 4.4 Features
```
Trees:    Oak, Birch, Spruce (incl 2×2 giant), Jungle (2×2, 4×4), Acacia, Dark Oak (2×2)
Grass:    Per biome (tall grass, ferns, dead bushes)
Flowers:  Per biome (poppy, dandelion, blue orchid, allium, tulips, etc.)
Pumpkins: 1/32 per chunk on grass
Melons:   Jungle only
Cacti:    Desert/Mesa, max 3 tall
Sugar cane: Near water, max 4
Lily pads: Swampland
Ice/Snow: Frozen biomes
Silverfish: Extreme Hills infested stone
```

### 4.5 Biome Color Properties
```
Grass & Foliage color determination:
  Each biome has temperature (T) and humidity (H) values that index into
  two 256×256 color map textures (stored at @minecraft/textures/colormap/):
    - grass.png:  Index by floor(T×255) along X, floor(H×255) along Z
    - foliage.png: Same indexing scheme
  Swamp biomes: override with fixed colors
    - Swampland grass:  #6A7039 / #4C763C (swamp)
    - Swampland foliage: #6A7039
  Mesa biomes: grass = #90814D, foliage = #9E814D

Water color per biome (exact RGB):
  Default (most biomes):         #3F76E4 (63,118,228)
  Swampland:                     #617B64 (97,123,100)  →  #4C6559 in some variants
  Swampland M:                   #4C6156 (76,97,86)
  Frozen Ocean / River:          #3938C9 (57,56,201)
  Cold Taiga variants:           #2056FF (32,86,255)
  Ice Plains / Mountains:        #2056FF
  Mushroom Island / Shore:       #8A8998 (138,137,152)
  Beach:                         #3F76E4 (default)
  Ocean / Deep Ocean:            #3F76E4
  River:                         #3F76E4
  The End (Sky):                 #3F76E4
  Hell (Nether):                 #3F76E4
  Void:                          #3F76E4

Sky color per biome (exact RGB):
  Default (most biomes):         Lerp from #78A7FF (day) to #0A0A1E (night)
  Swampland:                     #6A7039 → sky takes on brownish tint
  Mesa:                          #B9A75C → warm orange sky
  Hell:                          #100810 → dark red-black
  The End:                       #080808 → pitch black
  Mushroom Island:               #8080FF → purple-tinted

Fog color per biome:
  Default:                       Sky-derived (matches sky in clear weather)
  Swampland:                     #6A7039
  Mesa:                          #B9A75C
  Hell:                          #100810
  The End:                       #080808
  Water fog:                     decreases linearly with depth
  Lava fog:                      dense orange at short range

Rain / Snow determination per biome:
  Rain: T ≥ 0.15 (most temperate biomes)
  Snow: T < 0.15 (taiga, ice plains, cold taiga, etc.)
  Neither: desert, savanna, mesa, hell, end, void (T ≥ 2.0? Actually T > 1.0 for no rain)
```

### 4.6 Biome Temperature & Humidity
```
Biome                          | Temp  | Humidity
Ocean                          | 0.5   | 0.5
Plains                         | 0.8   | 0.4
Desert                         | 2.0   | 0.0
Extreme Hills                  | 0.2   | 0.3
Forest                         | 0.7   | 0.8
Taiga                          | 0.25  | 0.8
Swampland                      | 0.8   | 0.9  (override: water color, grass #617B64)
River                          | 0.5   | 0.5
Hell (Nether)                  | 2.0   | 0.0
The End (Sky)                  | 0.5   | 0.5
Frozen Ocean                   | 0.0   | 0.5
Frozen River                   | 0.0   | 0.5
Ice Plains                     | 0.0   | 0.5
Ice Mountains                  | 0.0   | 0.5
Mushroom Island                | 0.9   | 1.0
Mushroom Island Shore          | 0.9   | 1.0
Beach                          | 0.8   | 0.4
Desert Hills                   | 2.0   | 0.0
Forest Hills                   | 0.7   | 0.8
Taiga Hills                    | 0.25  | 0.8
Extreme Hills Edge             | 0.2   | 0.3
Jungle                         | 0.95  | 0.9
Jungle Hills                   | 0.95  | 0.9
Jungle Edge                    | 0.95  | 0.8
Deep Ocean                     | 0.5   | 0.5
Stone Beach                    | 0.2   | 0.3
Cold Beach                     | 0.05  | 0.3
Birch Forest                   | 0.6   | 0.6
Birch Forest Hills             | 0.6   | 0.6
Roofed Forest                  | 0.7   | 0.8
Cold Taiga                     | -0.5  | 0.4
Cold Taiga Hills               | -0.5  | 0.4
Mega Taiga                     | 0.3   | 0.8
Mega Taiga Hills               | 0.3   | 0.8
Extreme Hills+                 | 0.2   | 0.3
Savanna                        | 1.2   | 0.0
Savanna Plateau                | 1.0   | 0.0
Mesa                           | 2.0   | 0.0
Mesa Plateau F                 | 2.0   | 0.0
Mesa Plateau                   | 2.0   | 0.0
Void                           | 0.5   | 0.5

Mutated variants inherit base biome T/H.
Temperature at a given Y level: effective_temp = temp - (Y - 64) × 0.0016667
  (colder at high altitudes, used for snow line in extreme hills)
```

### 4.7 Slime Chunk Algorithm
```
Slime chunks use seed-based deterministic algorithm:
  seed_position = world_seed + (chunkX² × 0x4C1906) + (chunkX × 0x5AC0DB) +
                  (chunkZ² × 0x4307A7) + (chunkZ × 0x5F24F) ⊕ 0x3AD8025F
  is_slime_chunk = seed_position % 10 == 0
  Result: ~10% of all chunks are slime chunks (below y=40)
  Slimes spawn in slime chunks regardless of light level (but only y≤40)

Swampland slime spawning (non-chunk-based):
  - Slimes spawn in swampland at night (y=50-70)
  - Light level ≤ 7
  - Moon phase dependent: most during full moon, none during new moon
  - Chance inversely proportional to moon phase index (0=full→most, 4=new→none)
```

---

## P5 — Complete Combat System

### 5.1 Weapon Reach & Attack Range
```
Entity reach (attack range):
  Player attack reach:    3.0 blocks (measured from player eye position)
  Block interaction:      4.5 blocks (place/break/interact)
  Entity interaction:     3.0 blocks (attack entity)
  Creative mode range:    5.0 blocks (block interact)
  Extended reach (creative flying): 6.0 blocks

Projectile reach:
  Bow/snowball/egg:       exact entity intersection, no arbitrary cap
  Ender pearl:            65 block teleport (but won't load unloaded chunks)
```

### 5.2 Attack Cooldown & Damage Scaling
```
All weapons use the generic attackSpeed attribute. Damage scales linearly with
cooldown progress:
  charge = cooldown_progress (0.0 to 1.0)
  damage_mult = 0.2 + 0.8 × charge  (ranges from 20% at 0 charge to 100% at full)

Per-weapon attack speed (ticks between full charge):
  Sword:        4 ticks (0.2s)  attackSpeed = 1.6 × (20t/s)   → 12.5 attacks/s(×)
                                                               Actually: base=1.6, cooldown ticks=25/1.6=15.625→15
  Shovel:       5 ticks (0.25s) attackSpeed = 1.0 → 25 ticks
  Pickaxe:      5 ticks (0.25s) attackSpeed = 1.2 → 20.8→20 ticks
  Axe:          10 ticks (0.5s) attackSpeed = 0.9 → 27.7→27 ticks (stone), 1.0 → 25 (iron), 0.8→31 (diamond)
  Hoe:          12 ticks (0.6s) attackSpeed = 1.0 → 25 ticks for wooden... actually hoes in 1.12 are slow
    Actually corrected formula: cooldown_ticks = ceil(20 / attackSpeed)
    Sword (1.6):  ceil(20/1.6) = 13 ticks
    Shovel (1.0): ceil(20/1.0) = 20 ticks
    Pickaxe (1.2):ceil(20/1.2) = 17 ticks
    Axe (0.9):    ceil(20/0.9) = 23 ticks
    Hoe (1.0):    ceil(20/1.0) = 20 ticks
    Fist:         4.0 → 5 ticks

Cooldown indicator: see P8.18 Attack Indicator
```

### 5.3 Melee Damage by Weapon
```
Base damages (before cooldown scaling):
  Fist:          1 HP
  Sword (wood):  4 HP  | (stone): 5 HP  | (iron): 6 HP  | (diamond): 7 HP
  Axe (wood):    4 HP  | (stone): 5 HP  | (iron): 6 HP  | (diamond): 7 HP
  Pickaxe (w):   2 HP  | (stone): 3 HP  | (iron): 4 HP  | (diamond): 5 HP
  Shovel (w):    2.5 HP| (stone): 3.5 HP| (iron): 4.5 HP| (diamond): 5.5 HP
  Hoe (w):       1 HP  | (stone): 2 HP  | (iron): 3 HP  | (diamond): 4 HP

Important: Sword and Axe in 1.12.2 have the SAME base damage per tier.
The key difference is attack speed (sword is faster) and the sweep mechanic.
```

### 5.4 Critical Hits
```
Conditions (all must be true):
  - Player is falling (velocity.y < 0) but NOT on ground
  - Player is NOT in water or lava
  - Player is NOT on a ladder, vine, or in a web
  - Player is NOT suffering from blindness (no crits while blind)
  - Player is NOT riding an entity
  - Player's attack is fully charged (charge ≥ 0.95)
  - Player is not sprinting (sprint-crit is a separate effect)

Damage: base_damage × 1.5
Visual: dark blue/white star particles (crit particle) burst from target
Sound: higher-pitched "thwack" (entity.hurt with pitch modified)
Fall height: any positive fall distance counts; higher fall = same 1.5×
Sprint critical: some sources claim it adds 1 HP, but in 1.12.2 the sprint
smash is visual only — sprint + attack while falling still does 1.5×
```

### 5.5 Sweep Attack
```
Trigger conditions:
  - Attacking with a SWORD (any tier)
  - Attack is fully charged (charge ≥ 0.95)
  - Player is on ground (not falling, not in air)
  - Player is not sneaking

Sweep effect:
  - All living entities within 1.0 block radius of the player's reach (not the target)
    are hit with "sweep" damage
  - Sweep damage: 1 HP (always, regardless of weapon)
  - Entities hit by sweep: knocked back slightly (velocity multiplier 0.6× of main hit KB)
  - Main target: takes full damage + sweep animation plays

Visual:
  - Horizontal arc of white lines / sweep particles
  - Sweep particle: a curved fan of small white streaks

Implementation:
  - Check all entities within a horizontal 1.0 block radius centered on the main target
  - Filter: exclude the main target, exclude non-living entities
  - Apply 1 HP damage with no weapon enchantment effects
```

### 5.6 Sprint Attacks (Smash)
```
Trigger conditions:
  - Player is sprinting when attack lands
  - Attack is not fully charged (charge < 0.95) — sprint attack cancels charge

Effect:
  - Sprint attack damage = 0.25 × base (at 0 charge, since sprint resets cooldown)
    Actually: sprint attack in 1.12.2 deals extra knockback but reduced damage
  - Knockback: +50% horizontal velocity on the target
  - The visual "smash" effect shows your weapon pushing the target

Note: In 1.12.2, sprint attacks don't get the damage bonus they got in later versions.
The main value is the extra knockback for pushing enemies off ledges.
```

### 5.7 Ranged Combat
```
Bow charge:  1.5s (30 ticks) to reach full charge
  Charge tiers:
    0 ticks (instant):  0.15× damage, no critical
    1-14 ticks:         0.5× damage, no critical  
    15-29 ticks:        1.0× damage, no critical
    30+ ticks (full):   1.0× damage, +CRITICAL (star particles, extra knockback)

  Visual: bow string pulls back farther with charge
  Sound: bow tension squeaks, pitch rises with charge
  Arrow speed: 3.0 blocks/tick at full charge

Bow enchantments:
  Power:  +25% damage per level (I=1.25×, V=2.25× base arrow damage)
  Punch:  +0.25 blocks knockback per level (I=0.25, II=0.5)
  Flame:  sets target on fire for 4s (EntityData.fire = 80 ticks)
  Infinity: 50% chance to not consume arrow (actually: 1 - 1/2 in 1.12.2... it's always 1 arrow kept)

Crossbow: not in 1.12.2 (added 1.14)
```

### 5.8 Thrown Items (Snowballs, Eggs, Ender Pearls, Potions)
```
Snowball:
  - Damage: 0 HP (knockback only)
  - Can damage Blaze: 3 HP
  - Instant velocity: ~1.5 blocks/tick

Egg:
  - Damage: 0 HP (knockback only)
  - 12.5% chance to spawn a baby chicken on impact
  - Instant velocity: ~1.5 blocks/tick

Ender Pearl:
  - Damage: 5 HP (player takes damage on landing)
  - Teleports player to impact point
  - Can trigger Endermen anger if thrown through a portal (lore behavior)
  - Cooldown: 1 second (20 ticks) before next pearl use
  - Instant velocity: ~1.5 blocks/tick

Splash Potion:
  - Instant velocity: 0.5 blocks/tick
  - On impact: applies area-of-effect to all entities within 4 block radius
  - Duration for thrown potions: standard × 0.75 (splash), standard × 0.25 (lingering, 1.13+)
  - Glass bottle drops on impact (one per potion)

Lingering Potion: NOT in 1.12.2 (added 1.13)
```

### 5.9 Enchantment Combat Effects
```
Sharpness:  +0.5 damage per level (above level 1) to all targets
  Formula: damage_bonus = 0.5 × max(0, level - 1) + 1.0
  Sharpness I: +1.0  II: +1.5  III: +2.0  IV: +2.5  V: +3.0

Smite:      +1.25 damage per level to undead (zombies, skeletons, wither, etc.)
  Smite I: +1.25  II: +2.50  III: +3.75  IV: +5.0  V: +6.25

Bane of Arthropods: +1.25 damage per level to spiders, cave spiders, silverfish
  Same scaling as Smite (+1.25/lvl)

Fire Aspect: sets target on fire
  Fire Aspect I:  4s (80 ticks) fire
  Fire Aspect II: 7s (140 ticks) fire

Knockback: extra knockback velocity
  Knockback I:  0.5 × base KB
  Knockback II: 1.0 × base KB (double)

Looting: extra drops from mobs
  Looting I:  +1 max drop / II: +2 / III: +3
  +1% chance of rare drops per level

Note: Sharpness, Smite, and BoA are MUTUALLY EXCLUSIVE — only one can be on a weapon
```

### 5.10 Armor & Protection (see P1.5 for reduction formula)

### 5.11 Shields
```
Blocking mechanics:
  - Right-click with shield to raise (faces camera direction)
  - Active block arc: 180° forward (anything behind ≈90° from facing is not blocked)
  - Blocks 100% of incoming damage (including projectiles)
  - Projectiles: arrows bounce off, fire charges deflected, eggs/snowballs break
  - Blast protection: full block damage if facing explosion center

Durability:
  - Base durability: 336 (iron), 168 (wood may be different... actually only one tier)
  - Damage cost: max(1, floor(incoming_damage / 5))
  - Unbreaking: durability_cost = 1/(level+1) chance of reducing durability

Axe disable:
  - When a player's shield is hit by a sprinting axe attack:
    - Shield is disabled for 5 seconds (100 ticks)
    - Disabled: shield cannot be raised; held shield droops down visually
    - Cooldown indicator: shield icon with "broken" overlay

Shield enchantments:
  - Unbreaking: durability preservation
  - Mending: repair from XP
  - No specific "blocking" enchant for shields in 1.12.2

Implementation: shield blocking must check:
  1. Player right-clicking (shield raised)  
  2. Damage source direction relative to player look vector
  3. Whether shield is disabled (axe hit cooldown)
```

### 5.12 Weapon Enchantment Mutual Exclusions
```
Swords:
  - Sharpness × Smite × Bane of Arthropods (mutually exclusive)
  - Fire Aspect × (none exclusive)
  - Knockback × (none exclusive)
  - Looting × (none exclusive)
  - Sweeping Edge: NOT in 1.12.2 (added 1.14)

Bows:
  - Power × (none exclusive)
  - Punch × (none exclusive)
  - Flame × (none exclusive)
  - Infinity × Mending (mutually exclusive in 1.12.2 — cannot have both)
    Actually: in 1.12.2 Infinity and Mending CAN be combined with commands but
    naturally they are exclusive; if both, Infinity takes priority and Mending
    does nothing since the arrow is "infinitely" used without consumption

Other weapons:
  - Axes can have: Sharpness/Smite/BoA, Efficiency, Fortune/Silk Touch
  - Most combat enchants are sword-only
```

### 5.13 Damage Source Types & Bypass Rules
```
Damage sources and their classification:
  Incoming             Type         Bypass Armor  Bypass IFrames  Bypass Shield
  ─────────────────────────────────────────────────────────────────────────────
  Player attack        entity       —             —               blocked
  Projectile (arrow)   projectile   —             —               blocked
  Fall                 fall         —             —               blocked (×)
  Lava / Fire          fire         —             —               blocked (×)
  Drowning             drown       —             ✔               blocked (×)
  Starvation           starve      ✔             —               blocked (×)  
  Cactus               cactus      —             —               blocked (×)
  Suffocation          suffocate   —             —               blocked (×)
  Explosion            explosion   —             —               blocked (✓)
  Lightning            lightning   —             —               blocked (×)
  Magic (potion)       magic       ✔             —               blocked (×)
  Wither               wither      —             —               blocked (×)
  Poison               poison      ✔             —               blocked (×)
  Void                 void        ✔             ✔               blocked (×)
  /kill                —           ✔             ✔               blocked (×)

  (×) = shield blocks but damage source must be within 180° arc
  (✓) = shield always blocks explosions regardless of direction

  Note: Fall damage IS blocked by shield only if facing upward... actually
  fall damage is BLOCKED by shield in 1.12.2 if the player is looking at the
  ground and has shield raised. Wait, that was changed. In 1.12.2 shields DO
  block fall damage if the player faces upward (looking up while falling).
  Actually this was patched. In 1.12.2 shields do NOT block fall damage.
```

### 5.14 Damage Knockback
```
Base knockback formula:
  horizontal_velocity = 0.4 × (1 + enchant_knockback_level × 0.5)
  vertical_velocity   = 0.4 × (1 + enchant_knockback_level × 0.25)

Sprint attack knockback: +50% horizontal velocity
  horizontal_velocity = 0.6 × (1 + enchant_knockback_level × 0.5)
  vertical_velocity unchanged

Sweep attack knockback: 0.6× of main hit knockback (horizontal only, vertical = 0)

Mob knockback resistance:
  Each entity has knockbackResistance attribute (0.0-1.0)
  Effective knockback = base_knockback × (1 - kb_resistance)
  Iron golems: 1.0 (immune to knockback)
  Some mobs have partial resistance
```

### 5.15 Damage Tilt & Hurt Animation
```
Hurt animation:
  - Player model turns red (tint overlay) for 0.5s (10 ticks) on damage
  - HurtTime: set to 10 on hit, decrements each tick
  - During hurt: player cannot attack
  - Hurt sound plays on first tick of damage

Camera tilt (damageTilt):
  - On hit: camera rotates ~3-7° randomly left/right (screen tilt)
  - Tilt decays exponentially back to 0 over ~0.5s
  - Multiple hits in quick succession: reset tilt, overlay remains

Death animation:
  - Player falls backward (camera tilts up to 90° looking down)
  - Screen fades from red vignette to black over ~2s
  - DeathTime: starts at 20, increments each tick
  - At 20 ticks: death screen appears with respawn buttons
  - XP orbs scatter from death position

Damage immunity (IFrames):
  - After taking damage, player is immune for 10 ticks (0.5s)
  - Player model flashes red/white during immunity frames
  - Certain damage sources bypass immunity: void, /kill, drowning
  - Applied by entity attacks: 10 tick cooldown after hit
  - Environmental damage (fall, lava, fire): resets immunity each tick

Armor toughness:
  - Diamond armor: +2 toughness per piece (total 8)
  - Toughness reduces damage from high-damage hits
  - Damage reduction formula includes toughness:
    effective_damage = raw_damage × (1 - min(20, max(armor/5, armor - raw_damage/(2 + toughness/4))) / 25)
  - This means toughness is most effective against large hits
  - Example: 
    Full diamond (20 armor, 8 toughness) vs 25 damage:
      raw_damage/(2 + toughness/4) = 25/(2+2) = 6.25
      armor - 6.25 = 13.75  
      armor/5 = 4
      max(4, 13.75) = 13.75
      min(20, 13.75) = 13.75
      reduction = 13.75/25 = 0.55 → effective = 25 × 0.45 = 11.25 damage
```

---

## P6 — Mob Registry (Complete 1.12.2)

### 6.1 Spawning Basics
```
Despawn rules:
  - Most hostile mobs despawn when player is > 128 blocks away (instant)
  - Hostile mobs have 1/800 chance per tick to despawn when > 32 blocks from player
  - Passive mobs never despawn (except squids in certain conditions)
  - Named mobs (NameTag) NEVER despawn
  - Leashed mobs don't despawn
  - Equipment-picked-up mobs have reduced despawn

Spawn cycle:
  - Hostile: attempts 3 spawns per chunk per tick within 24-128 block ring
  - Passive: attempts 1 spawn per chunk per tick (animals)
  - Ambient (bat): 1 attempt per chunk per tick
  - Water (squid): 1 attempt per chunk per tick
  - Each attempt: pick random block, check light/sky/block conditions, spawn group

Despawn groups per mob:
  Most mobs: 4 entities per spawn (e.g. 4 zombies, 4 skeletons)
  Spiders: 1 (singular spawn)
  Endermen: 1 (singular spawn)
  Slimes: variable by size
```

### 6.2 AI Goals & Behavior
```
Mob AI operates via Goal/Task system. Each tick, active goals are evaluated.
Priority determines which goal runs.

Common goals by mob type:

Pathfinding basics:
  - Entities pathfind toward targets with A* (or simplified directional pathfinding)
  - Max pathfind range: 16 blocks (some mobs: 32 for target)
  - Entities avoid: fire, lava, cactus, magma blocks (1.13+)
  - Entities can climb: ladders, vines (if setCanClimb)
  - Entities collide: with each other (push), with blocks (collision box)
  - Most mobs can path through doors (zombies break them)
```

### 6.3 Passive Mobs
```
Animal spawning: light≥9, grass block, sky access, biomes

Sheep      HP:8  Drops: wool(1-3), mutton(1-3)   Breed: wheat    16 colors
  AI: Wander, flee, follow wheat, eat grass (regrow wool)
  Wool color: inherited from parents; random if different colors
  Sheared: drops 1-3 wool, regrows eating grass (random tick, 1/50 chance per grass eaten)
  Baby: 20% adult size, 20min grow time
  Drop: 1 wool even when sheared? No, only if sheared.
  Panic: flees when hit

Cow        HP:10 Drops: leather(0-2), beef(1-3)  Breed: wheat    milk with bucket
  AI: Wander, flee, follow wheat
  Milking: right-click with bucket → milk bucket
  Baby: as sheep

Chicken    HP:4  Drops: feather(0-2), chicken(1-3) Breed: seeds  eggs every 5-10min
  AI: Wander, flee, follow seeds, laid egg sound
  Egg laying: every 6000-12000 ticks (5-10 min), drops egg item
  Chick: follows adult chickens
  Fall: slow descent (doesn't take fall damage)
  Baby: as above

Pig        HP:10 Drops: porkchop(1-3)             Breed: carrot/potato/beetroot  rideable
  AI: Wander, flee, follow food
  Saddle: right-click with saddle → rides pig
  Speed: ~2.0 m/s (slow, no sprint)
  Carrot on a stick: control direction when riding
  Baby: as above

Rabbit     HP:3  Drops: hide(0-1), rabbit(0-1)    Breed: dandelion/carrot  6 skins, killer bunny
  AI: Hop, flee, follow carrots
  Hop pattern: jumps every 2-4 ticks
  Killer Bunny: hostile variant (commands only / natural in 1.12 certain biomes... actually rare spawn)
  Skin: brown, white, black, black & white, gold, salt & pepper
  Speed: ~4.0 m/s (fast but not persistent)
  Carrot crops: rabbits eat carrot crops (lower stage by 1)

Horse      HP:15-30 Drops: leather(0-2)           35 combos, speed 4.85-14.57 m/s, jump 1.25-5.25 blk
  AI: Wander, flee, eat grass, tail swish
  Taming: repeated right-click (no saddle) until hearts
  Stats: speed (0.3375-1.0 internal), jump (0.4-1.0 internal), health (15-30)
  Jump: charging animation; player dismounts if hitting block above
  Breeding: golden apple/golden carrot, foal grows 20min (accelerate 4min with food)
  Colors: 7 base × 5 markings = 35 variants

Donkey     HP:15-30 1 chest (15 slots)
  AI: Same as horse
  Chest: right-click with chest → 15 slot inventory (GUI opens)
  Can't equip horse armor
  Breeding: horse+donkey=mule

Mule       Horse×Donkey, 1 chest (15 slots)
  Sterile: cannot breed (mules have no offspring)
  Otherwise same as donkey

Skeleton Horse:
  Spawned by "skeleton trap" during thunderstorms (lightning strike trap)
  Trap horse: 15% chance on lightning strike during thunder (per horse type)
  Trapped horse: skeleton horse → lightning strikes → turns into 4 skeleton riders
  4 skeletons riding skeleton horses, each with bow + iron helmet
  Can be tamed: kill skeletons → ride remaining horses

Llama      HP:15-30 4 colors, spit, chest 3-15, caravan follow
  AI: Wander, flee, follow leader (caravan)
  Spit: attacks hostile mobs with 1 HP projectile (5% chance per 2 sec while hostile)
    Actually: Llama spits at wolves, deals 1 HP
  Caravan: when led, other tamed llamas follow in line (max 10)
  Chest: 3, 6, 9, 12, or 15 slots (random strength)
  Carpets: place carpet on llama for colored decoration
  Strength: 1-5 (determines chest size + attack damage of spit)
  Color: creamy, white, brown, gray

Parrot     HP:6  5 colors, tame seeds, mimic mobs, dance jukebox, die from cookies (1.12 NEW!)
  AI: Fly toward player when tame, perch on shoulder, mimic hostile mob sounds
  Taming: feed seeds (wheat, melon, pumpkin, beetroot)
  Shoulder: right-click → sits on shoulder (left or right)
  Dancing: jukebox plays → nearby tamed parrots dance (bob head)
  Mimic: plays hostile mob ambient sounds within 20 blocks
  Death by cookie: cookies kill parrots (instant, entity data: cookieTicks)
  Colors: red, blue, green, cyan, gray

Ocelot     HP:10 raw fish tame, 3 variants, scare creepers
  AI: Stalk chickens, sneak, flee from players
  Taming: raw cod/salmon, patience required
  Variants: wild (default), tabby, tuxedo, siamese, red... actually in 1.12: ocelot (wild) stays wild,
    taming turns into cat → 3 cat skins. Actually 1.12.2 ocelot = untamed, cat = tamed.
  Scare creepers: creepers flee from ocelots/cats within 6-16 blocks
  Chickens: ocelots hunt chickens (sneak + pounce)
  Trust: ocelot trusts player after taming

Wolf       HP:20 bone tame, 4 attack, collar dye
  AI: Follow player when tame, attack target, shake off water
  Taming: feed bones (1-3 usually), hearts appear
  Attack: 4 HP per bite (to mobs)
  Health: 20 HP (tamed), 8 HP (wild — actually wild have 8, tamed 20)
  Collar dye: right click with dye → change collar color
  Teleport: teleports to player if > 12 blocks away, in line of sight
  Skeletal: wolves attack skeletons (wild and tamed)
  Tail angle: indicates health (100% → straight up, ≤10% → straight down)
  Sitting: right-click → sits (stays put)

Squid      HP:10 ink sac(1-3) on hit, water mob
  AI: Swim randomly, flee when attacked
  Water: despawns if out of water for too long (actually doesn't despawn naturally... 
    but if out of water for > 1 tick, starts suffocating)
  Ink: on hit, emits ink cloud (particles + water tint)
  Movement: gentle swimming, no pathfinding, random direction changes
  Spawn: 1-4 in water, any biome

Bat        HP:6 ambient, hangs upside down
  AI: Hangs from ceiling at night (inactive), flies randomly when dark
  Hitbox: tiny (1×1×0.9 block), can fit through 1×0.5 gaps
  Light: sleeps regardless, only active at light level ≤ 4
  Inactive: hangs from underside of solid block
  Movement: erratic, fast, fluttering
  Despawn: despawns at dawn in overworld

Mooshroom  HP:10 mycelium, bowls→stew, shears→cow
  AI: Like cow but only spawns on mycelium in mushroom islands
  Bowl: right-click with bowl → mushroom stew
  Shears: right-click with shears → turns into cow, drops 5 mushrooms
  Mooshroom tupe: red only in 1.12.2 (brown added 1.13)
  Spawn: mycelium block, light ≥ 9, mushroom island biome only

Villager   HP:20 7 professions, 5-6 trades, zombie infection
  AI: Wander, socialize (meeting), go home at night, flee from zombie
  Schedule: 
    Day: wander village, work at job site block (no job blocks in 1.12.2, just schedule)
    Night: enter house (door, bed)
    Rain: seek shelter
  Zombie infection: zombie kills villager → 0% chance in easy, 50% normal, 100% hard
  Cured zombie villager: weakness potion + golden apple → 2-5 min conversion time
    Cured: trades heavily discounted (permanent discount)
  Professions: see P9F for detailed trades

Iron Golem HP:100 3-5 iron + poppy, attacks hostiles
  AI: Patrol village, attack hostile mobs, give poppy to children
  Spawning: built with 4 iron blocks + pumpkin (player or naturally)
  Natural: villagers gossiping → but in 1.12.2: 10+ villagers + 21+ doors → golem spawns
  Attack: picks up mob, throws in air, deals 7-21 damage
  Knockback: immune (100% knockback resistance)
  Water: walks along water bottom (doesn't swim)
  Poppy: holds out poppy to offer to villager children

Snow Golem HP:4 snowballs, snow trail, melts in desert/jungle/nether
  AI: Throw snowballs at hostile mobs, walk randomly
  Construction: 2 snow blocks + pumpkin/carved pumpkin
  Damage: 0 HP (knockback only), doesn't damage most mobs
  Blaze: Snowballs thrown by snow golem actually DO damage blaze (3 HP)
  Snow trail: leaves snow layer on ground in snowy/cold biomes
  Melting: takes damage in hot biomes (desert, jungle, nether) and rain
  Water: dies in water (suffocates as snow golem is 1 block tall... actually 1.8+ can swim)
```

### 6.4 Hostile Mobs
```
Hostile spawning: light ≤ 7 (except specific mobs), appropriate biome, blocks
Most hostiles: spawn in darkness at any y level

Zombie        HP:20 Atk:3  Drops: rot flesh, iron/carrot/potato rare  Burns sun, breaks doors
  AI: Chase player (35 block range), break doors (hard), call reinforcements
  Sunlight: burns for 1 HP/2s (2.5s fire resistance delay, then burn)
  Equipment: can spawn with iron sword, shovel, or random armor (difficulty dependent)
  Reinforcements: on hit, 2-10% chance to spawn additional zombie behind player
  Door breaking: hard difficulty only, zombie pounds door for 2-5 seconds then breaks
  Baby zombie: 50% chance when spawning (if zombie type supports it)
    Baby: 0.9 block tall, moves faster (1.5x speed), no sun burn? Actually baby does burn
  Villager conversion: zombie kills villager → zombie villager based on difficulty
  Drowned: falls in water → doesn't drown (converts to drowned in 1.13+, in 1.12 just stays zombie)

Husk          HP:20 Atk:3  Desert variant, no sun burn, hunger 10s
  AI: Same as zombie, no sun burn
  Effect: hit applies 10s hunger (100 ticks)
  Variant: replaces zombie in desert biomes
  No door breaking: husk cannot break doors (zombie can)
  Baby: yes, baby husk exists

Zombie Villager HP:20 Atk:3  Curable: weak pot + gold apple 2-5min
  AI: Same as zombie
  Curing: 
    1. Splash weakness potion on zombie villager
    2. Feed golden apple (right-click)
    3. Conversion time: 2-5 minutes (2400-6000 ticks)
    4. During conversion: nausea effect, shaking animation (red particles)
    5. Converted villager: profession random, trades permanently discounted
  Discount: cured villager offers trades at reduced prices (permanent mercy discount)

Skeleton      HP:20 Atk:3  Bow, burns sun, strafes
  AI: Chase player (32 block range), strafe at ~16 blocks, shoot bow
  Sunlight: burns for 1 HP/2s (same fire delay as zombie)
  Bow: shoots every 2-4 seconds, arrow velocity varies
  Strafing: moves sideways while keeping distance (circle strafes)
  Equipment: can spawn with enchanted bow, armor
  Cliff avoidance: skeletons avoid walking off cliffs (unlike zombies)
  Wolf avoidance: flees from wolves (undead vs canine)

Stray         HP:20 Atk:3  Ice plains, shoots slowness arrows
  AI: Same as skeleton
  Variant: replaces skeleton in icy biomes (ice plains, icebergs)
  Slowness arrow: hit applies slowness I for 30 seconds
  No sun burn: strays are immune to sunlight (wearing tattered clothes?)
  Actually strays DO burn in sunlight in 1.12.2 — they're still undead

Wither Skeleton HP:20 Atk:8  2.5% skull, wither 10s
  AI: Chase player, taller hitbox (2.5 blocks vs 1.99 for normal skeleton)
  Spawn: nether fortresses, light ≤ 7
  Attack: iron sword, 8 HP damage, 10s wither effect
  Skull drop: 2.5% chance, Looting increases
  Hitbox: 0.7 × 2.4 (wider and taller than normal skeleton)
  No sun burn: nether mob, but actually in overworld, wither skeletons DO burn
  Bone meal: drops 0-2 bones
  Coal: drops 1 coal (sometimes)

Creeper       HP:20 Power 3  Fuse 30t, charged power 6, flees cats
  AI: Chase player (16 block range), hiss when ≤ 3 blocks, explode
  Fuse: 30 ticks (1.5s) from hiss → explosion
  Detonation: if player moves > 10 blocks away, fuse resets (creeper stops hissing)
  Explosion power: 3 (same as TNT), destroys blocks
  Charged: lightning strike → charged creeper, explosion power 6 (2×)
  Cat/Ocelot: flee within 6-16 blocks of cat/ocelot
  Explosion: drops music discs if killed by skeleton arrow
  Silent: creepers make no footstep sound (unlike most mobs)

Spider        HP:16 Atk:3  Wall climb, neutral in light, leap
  AI: Chase player, climb walls, neutral in daylight
  Neutral: attacks player if player attacks it, or if player gets close enough
    Actually: spiders in 1.12.2 are hostile in darkness (as standard mob), 
    but become NEUTRAL when light level is above 7 (daylight in open areas)
    When neutral: will not attack unless provoked
  Wall climb: can climb ANY block (no-clip collision block? No, uses ladder-like climbing)
    Actually spiders climb by disabling collision with blocks they walk into
  Leap: jumps at player when close, dealing additional knockback
  Spider jockey: 1% chance to spawn with skeleton riding it
  Eye height: lowest eyes of any hostile mob (0.3 blocks above ground)
  Passability: 1 block tall × 1 block wide (can fit through 1×1 gaps)

Cave Spider   HP:12 Atk:3  Poison 7s, 1×1, mineshafts
  AI: Same as spider but smaller hitbox (0.7 × 0.5)
  Spawn: mineshafts with cobwebs (spawner in mineshaft corridors)
  Poison level: 7 seconds (140 ticks) of Poison II? Actually:
    Cave spider poison: Poison II for 7 seconds (damage 1 HP/1.2s = ~6 HP total)
  Hitbox: 0.7 × 0.5 block — can fit through 1×0.5 gaps
  No neutral: cave spiders are ALWAYS hostile (never neutral)
  Web immunity: no movement penalty from cobwebs (unlike player)

Enderman      HP:40 Atk:7  Teleport, gaze aggro, avoid water
  AI: Move blocks (pick up/place), stare aggression, teleport
  Aggravation: player looks at enderman's eyes (0-45° angle range for 1 tick)
    Once aggravated: player can look away without losing aggro
  Teleport: every 2-3 seconds while chasing, instant (no travel)
    Teleport range: 32 blocks or 8 blocks from player
    Water teleport: enderman teleports out of water (takes damage in water/rain)
  Block carrying: picks up certain blocks (dirt, grass, sand, gravel, flowers, 
    mushrooms, pumpkins, melons, TNT, cactus, clay, nylium, etc.)
    Actually 1.12.2: enderman can pick up: dirt, grass, podzol, sand, red sand, 
    gravel, water... no water. Enderman can pick up: dirt, grass block, podzol, 
    sand, red sand, gravel, clay, dandelion, rose, brown mushroom, red mushroom, 
    cactus, pumpkins, melons, TNT, netherrack
  Water: 1 HP damage per tick in water/rain
  Sun: no burn (does not burn)
  Sound: enderman scream when aggravated, "teleport" sound when moving
  Attack: deals 7 HP with knockback
  Eye position: must be at player eye level (enderman is 3 blocks tall, eyes at 2.55)

Silverfish    HP:8  Atk:1  Hides in stone, calls others
  AI: Hide in stone blocks, swarm when threatened
  Hiding: enters stone blocks (stone, cobble, stone bricks) → block shakes → 
    silverfish disappears, block becomes infested variant
  Call: hurt silverfish calls all silverfish within 32 blocks to attack
  Bedrock: silverfish cannot enter or hide in infested stone in peaceful?
  Damage: 1 HP per hit (weak but swarming)
  Movement: fast, small (0.3 × 0.3 hitbox)
  Spawn: extreme hills or strongholds (infested stone)

Slime         HP:16/4/1  3 sizes, splits, swampland/caves
  AI: Hop toward player, split on death
  Sizes: Big (4 blocks, HP:16, Atk:4), Small (2 blocks, HP:4, Atk:2), Tiny (1 block, HP:1, Atk:0)
  Splitting: Big → 2-4 Small; Small → 2-4 Tiny; Tiny → 0 (dies, drops slimeball)
  Movement: hop every 10 ticks, direction toward player if hostile
  Spawn: swampland (night, full moon more common), caves below y=40, slime chunks
  Slime chunks: 1/10 of all chunks in certain Y ranges (y=1-39, y=0 is bedrock... y=1-39)
  Damage reflection: big slimes deal contact damage based on size
  Drop: 0-2 slimeballs (tiny only), Big/Small drop nothing
  No-damage: tiny slime does no contact damage

Magma Cube    HP:16/4/1  Nether slime, fire immune, jumps higher
  AI: Like slime but:
  Fire immunity: immune to fire and lava damage
  Jump: higher than slime (3 block jump vs 2 for big slime)
  Spawn: nether (fortresses, basalt deltas general nether)
  Drop: magma cream (tiny only), 25% chance
  Split: same as slime (big → smaller)
  Damage: big magma cube deals fire-type damage (not reflected by fire resistance)
  Tower: magma cubes can stack on top of each other

Ghast         HP:10 Atk:9  Fireball, nether, 3 shot burst
  AI: Fly, shoot fireballs at player, screech
  Spawn: nether, any light level, open space needed (5×5×5 minimum)
  Fireball: shoots in burst of 3, velocity ~0.3 blocks/tick
  Fireball damage: 9 HP (direct hit), explosion power 1 (doesn't destroy blocks?)
    Actually ghast fireball explosion power = 1 (destroys blocks in nether, not overworld)
  Reflecting: hit fireball with arrow or sword → fireball deflects back
  Sound: ambient screech, fireball sound, hurt sound
  Hitbox: 4 × 4 × 4 blocks (large, but center is hollow for collision? No, full cube)
  Kill credit: if fireball deflected kills ghast, credit to deflector
  Spawning conditions: light ≤ 11? Ghasts spawn in nether at any light level
    Actually: light level is irrelevant for ghasts

Blaze         HP:20 Atk:5  3 fireball burst, snowball dmg 3
  AI: Fly, shoot fireballs, bob up and down
  Spawn: nether fortresses, spawners
  Fireball: shoots burst of 3 fire charges (different from ghast fireball)
    Fire charge velocity: ~0.2 blocks/tick
    Damage: 5 HP + 5s fire (100 ticks)
  Snowball damage: 3 HP per snowball hit (blaze are weak to snow)
  Blaze rod: 50% drop chance (base, increased by Looting)
  Hitbox: 0.6 × 1.8 (tall thin)
  Bobbing: blazes rise and fall rhythmically (period ~2 seconds)
  Blaze spawner: found in nether fortresses (like dungeon spawners)
  Health: 20 HP

Witch         HP:26 Throw potions, drinks self-buffs, drops variety
  AI: Chase player (range 16), throw harmful potions, drink buff potions
  Spawn: any dark place, swampland huts, raid participants (1.14+, skip)
  Potion throwing:
    - Harming II (instant damage) when player at ≤ 4 blocks
    - Slowness (1:07) at 4-7 blocks
    - Weakness (1:07) at 4-7 blocks
    - Poison (0:33) when player at ≥ 7 blocks or health ≥ 8
  Self-buffs:
    - When damaged → drinks Healing (instant, 4 HP)
    - When on fire → drinks Fire Resistance (3:00)
    - When in water → drinks Water Breathing (3:00)
    - Falling? Actually witch doesn't take fall damage... wait she can drink slow falling? No slow falling is 1.13
    - Witch doesn't drink slow falling potion, just takes fall damage normally
  Resistances: 85% damage reduction from... actually witch has no built-in DR, 
    but can drink Resistance potion (not available in 1.12.2? Witch doesn't use resistance)
  Drops: sticks, redstone, glowstone, gunpowder, spider eye, sugar, glass bottles
    All drops: 1-3 items, 1/8 chance each (so ~2.625 items per kill average)
  Hut: swampland huts have a witch spawn inside (always exactly 1 witch)

Guardian      HP:30 Atk:6+2  Laser 2s charge, spikes, monument
  AI: Swim, laser attack, retract spikes when not attacking
  Spawn: ocean monuments, water
  Laser:
    - Charge time: 2 seconds (40 ticks)
    - Charging: laser beam turns green, guardian's eye locks on
    - Full charge: beam turns white, instantly deals 6 HP damage
    - During charge: beam shows faint beam line from guardian eye to player
    - If player moves behind block: beam breaks, charge resets
  Spikes: extend when attacking (guardian takes less damage)
    Damage reflection: when spikes extended, attacker takes 2 HP damage
    When retracted: no reflection damage
  Movement: swims around monument, aggressive within 15 blocks
  Eye: guardian tracks player with single eye
  Drop: raw fish (50%), prismarine shard (33%), prismarine crystals (16%)
    Rare: 2.5% drop raw cod (if using correct fish naming)

Elder Guardian HP:80 Atk:8+2  3/monument, mining fatigue III 5min
  AI: Same as guardian but applies mining fatigue
  Mining Fatigue: applied automatically when player is within 50 blocks
    Effect: Mining Fatigue III for 5 minutes (6000 ticks)
    Applied: every 60 seconds if player stays in range
    Visual: purple ghostly effect on screen?
  Spawn: exactly 3 per ocean monument (in the top, wing, and treasure room)
  Spikes: same reflection damage (2 HP) as guardian
  Laser: same as guardian but 8 HP (base) + 2 reflection = 10 total potential
  Health: 80 HP (much tankier than guardian)
  Drop: 1 raw fish (guaranteed), prismarine shard (50%), sponge (1, always, as of 1.12?)
    Actually elder guardian always drops 1 wet sponge (as of 1.12)

Shulker       HP:30 Atk:4  Guided bullet, levitation 10s, end cities
  AI: Stationary (attached to block), shoots guided bullets
  Spawn: End cities, end ships (on purpur blocks)
  Bullet: guided projectile that homes on target
    - Bullet speed: ~0.3 blocks/tick
    - On hit: 4 HP damage + Levitation I for 10 seconds (200 ticks)
    - Bullet can be destroyed by hitting it (1 HP)
    - Bullet leaves trail of purple particles
  Shell: shulker can open/close shell
    Closed: 80% damage reduction? Actually damage is reduced to 10% when closed
      Formula: if closed, damage = damage × (0.1 + 0.2 × [remaining shells])... 
      Actually: shulker has 0-20 shells, damage to closed = raw_damage / 20^(shells/???)
    Let's simplify: closed shulker takes 0-2 damage from most hits
  Teleport: if no valid block to attach to, teleports to nearby wall
  Dye: shulker shells can be dyed when placed (item form)
  Color: natural purple, dyeable in crafting (shulker box recipe)
  Drop: 1 shulker shell (50% chance, +6.25% per Looting level)

Vex           HP:14 Atk:5.5 Phases through blocks, summoned by evoker
  AI: Fly, phase through walls, chase player
  Summoned by: Evoker (in sets of 3)
  Phase: can pass through solid blocks (no collision except for entities)
  Attack: swoops down, deals 5.5 HP
  Duration: vex despawns after 30 seconds (600 ticks) if evoker dies
  Equipment: iron sword with high damage (5.5 HP)
  Hitbox: 0.4 × 0.8 (small and fast)
  No collision: vexes pass through ALL blocks (including bedrock)
  Health: 14 HP (dies quickly)
  Sound: vex laugh/hiss when attacking

Vindicator    HP:24 Atk:7.5 Axe, mansions, Johnny variant
  AI: Chase player, hold iron axe in raised position
  Spawn: woodland mansions, raids (1.14+) — in 1.12.2 only mansions
  Attack: iron axe, 7.5 HP damage (unenchanted)
  Johnny: name tag "Johnny" → vindicator attacks ALL living entities (not just player)
    - Easter egg reference to The Shining
    - Aggressive to: villagers, iron golems, wandering traders, etc.
  Equipment: iron axe (can drop, 8.5% base)
  No shield break: vindicator axe doesn't specifically break shields (player shield 
    mechanics still apply though)
  Hitbox: 0.6 × 1.95

Evoker        HP:24 Summons vexes + fangs, drops totem of undying
  AI: Stay at range, summon vexes and fangs, flee when cornered
  Spawn: woodland mansions (in specific rooms)
  Summon Vex: raises arms, vex appears at random offset (3 vexes at once)
    Cooldown: after vex flock dies or leaves range
    Actually: evoker summons vexes every 5-10 seconds
  Fangs: attack where player is about to step (predictive)
    - Fangs deal 6 HP damage, bypass armor (magic damage)
    - Fangs rise from ground with 0.5s delay
    - Attack pattern: ring of fangs around evoker, or line toward player
  Totem of Undying: always drops 1 (guaranteed)
    Totem: when held in off-hand, prevents death (heals to 1 HP, 
    applies Absorption II 5s + Regeneration I 5s)
  Hitbox: 0.6 × 1.95
  During raids: not in 1.12.2 (raids are 1.14+)

Illusioner    HP:32 Atk:4  Blindness + decoys, commands only (1.12 NEW!)
  AI: Only spawnable via /summon illusioner (no natural spawn)
  Attacks:
    - Bow: shoots normal/fire arrows (uses bow)
    - Blindness: applies blindness to player (20 seconds cooldown)
    - Decoys: creates 4 illusionary copies of itself that move independently
      Decoys: have collision, take damage (disappear on hit), mimic movement
      Illusioner and decoys move in synchronized pattern
  Drops: nothing by default (or same as vindicator with commands)
  Hitbox: 0.6 × 1.95
  Note: Illusioner was fully coded for 1.12 but never naturally implemented

Wither        HP:300 Atk:8+Wither  Built with 4 soul sand + 3 skulls, phase 1+2
  AI: Boss mob, 2 phases, shoots skulls, destroys blocks
  Construction: T-shaped soul sand (4 blocks) + 3 wither skeleton skulls on top
  Phase 1 (HP > 150, 50%):
    - Immune to arrows (projectile immunity)
    - Charges at player, shoots 1-2 blue skulls per attack
    - Blue skull: explosion power 4, leaves blue wither effect on block? No, wither
      skulls destroy blocks (like TNT) in 1.12.2
      Actually: blue skulls break obsidian... no that's 1.14+
      In 1.12.2: Wither skulls have explosion power 1, CAN'T break obsidian,
      CAN'T break bedrock, end portal frames, etc.
      Blue skull = explosive, black skull = normal
    - Summon wither skeletons: occasionally summons 3-4 (in hard difficulty)
  Phase 2 (≤ 150 HP, 50%):
    - Ramming: charges through blocks (destroys them), then shoots 3-4 skulls rapidly
    - Arrows now work (wither loses projectile immunity in phase 2)
    - Wither armor: when hit by something, shoots skulls at nearby mobs
    - Explosion on entering phase 2: large explosion (power 7), destroys nearby blocks
  Defense: 50% damage reduction from most sources? No, wither has 100% damage reduction
    Actually: wither gains damage reduction when first spawned (invulnerable for 11 seconds)
    During invulnerability: can't be damaged, does not attack
  Health: 300 HP (150 HP per half)
  Drop: nether star (1, guaranteed)
  Nether star: used to craft beacon
  Boss bar: appears at top of screen (see P8)

Ender Dragon  HP:200 Atk:6+10 Heals from crystals, 3 regen phases
  AI: Fly around end island, dive at player, perch on portal
  Health: 200 HP
  Attack (dive): swoops at player, deals 6 HP (contact) + 10 HP (breath attack)
  Breath attack: purple dragon breath, lingers for 3 seconds, deals 6 HP/s
    Dragon breath: can be collected in bottles (dragon's breath item) for lingering potions (1.13+)
  Healing crystals:
    - 10 end crystals on obsidian pillars (some caged in iron bars)
    - Each crystal heals dragon for 1 HP per 0.5s (20 HP/s total if all active)
    - Crystal destroyed: pillar explodes (power 6), damages nearby entities
    - Dragon's health cannot exceed 200
  Perching:
    - Periodically, dragon perches on exit portal (unreachable via center)
    - While perched: dragon is immune to projectiles? No, takes extra damage from melee
    - Dragon breath: breathes downward, creating breath cloud
  Phases:
    1. Circle + dive (standard fly pattern)
    2. Perch + breath (dragon lands after 5-10 dives)
    3. Fly + shoot fireballs (if enough crystals destroyed)
  End exit portal: activated when dragon dies
  Dragon death: 
    - Dragon flies to portal, starts dying
    - Explosion: large explosion, drops 12,000 XP (total distributed)
    - End portal appears with dragon egg on top
    - Dragon egg: teleports when hit (no block update? teleports to nearby air)
    - Credits: end poem triggers on first exit or any exit through portal
  Hitbox: 8 × 8 × 6 (facing forward: 16 × 6 × 8? Actually 16 wide × 6 tall × 8 deep)
  Part system: dragon has multiple hitboxes (head, body, tail, wings)
    Each part has own HP (but damage all goes to main health pool)
  Drop: dragon egg (once per world), 12,000 XP
```

### 6.5 Mob Equipment Rules
```
Equipment spawning (difficulty dependent):
  Easy:    5% chance (helmet mostly)
  Normal:  10-15% chance (armor pieces, tools)
  Hard:    20-30% chance (enchanted gear possible)

Equipment types:
  Zombie:     iron sword (rare), random armor pieces
  Skeleton:   bow (always), random armor pieces
  Zombified:  golden sword (rare), random armor (low enchant)
  Wither Skeleton: iron sword (always)

Equipment drop chance:
  Base: 8.5% per equipped item
  Looting: +1% per level (Looting III = 11.5% per item)
  Equipment is damaged (durability 50-100% random)

Natural armor:
  Some mobs have innate armor (not equipment):
  Wither: 4 armor points? Actually wither has 0 armor but 50%? No, wither has 0 armor points
  Shulker: has innate damage reduction from shell closing
  Ender Dragon: 0 armor points but high HP + crystal healing
```

### 6.6 Mob Experience Drops
```
Experience orbs dropped on death:
  Passive: 1-3 XP (baby: 0)
  Hostile: 5 XP (most, + equipment bonus per item)
  Boss: 12,000 XP (ender dragon), 50 XP (wither) — actually wither drops 50 XP
  Slime/Magma Cube: 1-4 XP (varies by size)
  Player: 7 XP per level (dropped on death, 50% of total levels)

Equipment bonus XP:
  +1 XP per armor piece/tool dropped/given
  Max ~ 5-6 extra XP from equipped mobs

Baby mobs: 0 XP (or 1 XP in some cases)
```

### 6.7 Mob Interactions
```
Mob-to-mob interactions:
  Zombie → Villager:   attacks villager, converts to zombie villager (difficulty %)
  Zombie → Turtle:     not in 1.12.2 (turtles 1.13)
  Skeleton → Wolf:     skeletons are attacked by wolves
  Creeper → Cat:       flees from cats/ocelots
  Spider → Skeleton:   spider jockey (skeleton rides spider, 1% chance)
  Enderman → Water:    avoids water, takes damage in rain
  Iron Golem → Hostile: attacks hostile mobs in range
  Snow Golem → Blaze:  snowballs damage blazes
  Witch → Injured:     drinks healing when damaged
  Vindicator → All:    "Johnny" variant attacks everything
  Evoker → Vex:        summons and commands vexes
  Llama → Wolf:        spits at wolves
  Silverfish → Stone:  hides in infested stone variants
  Parrot → Cookie:     dies if fed cookie (instant)
  Parrot → Jukebox:    dances near playing jukebox
  Parrot → Mob:        mimics mob sounds (ambient)
```

---

## P7 — Redstone Mechanics

### 7.1 Redstone Tick System
```
Redstone operates on a sub-tick system. Each game tick (50ms) is divided into
2 "redstone ticks" (25ms each). The redstone tick is the base unit for timing.

Block Event Queue:
  - All redstone updates are processed through a block event queue
  - Each game tick:
    1. Process queued block events (up to 1000 per tick)
    2. Execute scheduled block events (pistons, etc.)
    3. Process neighbor updates from block changes
    4. Update comparator containers (inventory changes)
  - Order within one redstone tick: priority based on event type
    1. Block added/removed events
    2. Scheduled piston events
    3. Neighbor block updates
    4. Comparator updates

Update Order (direction priority):
  When a block changes, it notifies neighbors in this order:
    1. -X (West)
    2. +X (East)
    3. -Z (North)
    4. +Z (South)
    5. -Y (Down)
    6. +Y (Up)
  This order matters for deterministic behavior in complex circuits.
```

### 7.2 Power Sources
```
Redstone Block       — Perm 15 (constant signal, always powered)
Redstone Torch       — 15, unpowered when attached block is powered
  - Inverted power source: emits 15 when block above it is NOT powered
  - Attached to: side or top of a block (never bottom)
  - Burnout: 8 toggles in 60 tick sliding window → burnout for 8 ticks
    See detailed burnout mechanics below
  - Torch state: lit → unlit transition when attached block receives power
  - Unlit → lit transition when attached block loses power (1 redstone tick delay)
Redstone Dust        — 0-15, loses 1/block, weak power
Lever                — Toggle 15 (permanent on/off toggle)
Button (stone)       — 20 tick pulse (1s) [10 redstone ticks]
Button (wood)        — 15 tick pulse (0.75s) [7.5 redstone ticks, rounded to 8]
  - Wooden button also activated by arrow/projectile hit
Pressure Plate (S)   — 15 while entity (player/mob) stands on it
Pressure Plate (W)   — 15 while ANY entity/item stands on it
  - Also activates for dropped items, arrows, etc.
Weighted (Iron)      — min(floor(entity_count/10), 15)
Weighted (Gold)      — min(entity_count, 15)
Tripwire Hook        — 15 while tripwire string is broken
  - Requires tripwire string between two hooks, projectile can trigger
Detector Rail        — 15 while minecart on it (activates comparator)
  - Also powers adjacent powered rails
Daylight Detector    — 0-15 based on sky light
  - Inverted mode: right-click to switch to night mode (inverted signal)
Observer             — 1-tick pulse (2 game ticks = 0.1s) on neighbor change
  - Detects ANY block state change, not just power changes
Trapped Chest        — 1 + viewer_count (up to 15)
```

### 7.3 Redstone Torch Burnout (Detailed)
```
Sliding Window Algorithm:
  - Each torch maintains a queue of recent toggle timestamps
  - Toggle event: torch changes state (on→off or off→on)
  - Window: 60 game ticks (3 seconds, 120 redstone ticks)
  - If the queue contains >= 8 toggles within any 60-tick window → BURNOUT

Burnout State:
  - Torch block changes from lit_redstone_torch to unlit_redstone_torch
  - Visual: flame texture removed, just stick texture
  - No particle effects, no sound
  - Duration: 8 game ticks (0.4 seconds, 16 redstone ticks)
  - During burnout: emits power level 0

Recovery:
  - After 8 ticks, torch checks if its attached block is still powered
  - If attached block UNPOWERED → torch re-lights immediately
  - If attached block STILL POWERED → stays unlit for another 8 ticks
  - Repeat until attached block is unpowered

Implementation:
  - Store an array of toggle timestamps (game ticks) per torch
  - On each toggle: push current game tick, remove entries older than 60 ticks
  - If array.length >= 8 → enter burnout state
```

### 7.4 Redstone Dust Behavior
```
Signal strength: starts at source value (0-15), decreases by 1 per block traveled
  - Dust on a block: strength = source_strength - distance_traveled
  - Minimum: 0 (no signal) at max distance = source_strength blocks

Dust shape (dot vs cross vs line):
  - No connections → dot (small center dot)
  - Connects to adjacent redstone dust at same Y level
  - Connects to dust 1 block higher or lower (vertical)
  - Connects to: repeaters, comparators, pistons, lamps, doors, any component
  - Vertical: dust can climb up one block, flow down one block

Weak vs Strong Power:
  - Redstone dust provides WEAK power
  - Weak power: can power adjacent redstone components but CANNOT power solid blocks
  - Strong power: from redstone torch, repeaters, comparators, buttons, levers
    CAN make adjacent solid block "powered" (which powers dust on top)

Powering a block:
  - A block is "powered" if it receives strong power from any direction
  - A powered block activates adjacent redstone dust on top
  - A powered block deactivates attached redstone torches
```

### 7.5 Repeater Mechanics
```
Function: regenerates signal to full 15, adds delay, directional
  - Input from back, output from front
  - Only responds to signal changes on its back face

Delay settings (1-4):
  Position 1: 1 redstone tick (0.1s, 2 game ticks)
  Position 2: 2 redstone ticks (0.2s, 4 game ticks)
  Position 3: 3 redstone ticks (0.3s, 6 game ticks)
  Position 4: 4 redstone ticks (0.4s, 8 game ticks)

Locking (side-input locking):
  - Locked when another repeater/comparator faces its side
  - Locked repeater: output constant regardless of input
  - Middle bar appears as solid line when locked

Key behaviors:
  - Outputs STRONG power (can power blocks)
  - Diode: only allows signal flow one direction
  - Output strength always 15 (full regeneration)
```

### 7.6 Comparator Mechanics
```
Modes:
  - Compare mode (default): output = (inputA >= inputB) ? inputA : 0
    Back >= side → pass-through; Back < side → blocked
  - Subtract mode (right-click): output = max(0, inputA - inputB)

Container fill detection:
  Formula: output = floor(1 + (slot_count / max_slot_count) × 14)
  Where slot_count = sum of (item_count / max_stack_size) per occupied slot

Container → signal strength at full fill:
  Chest/double chest    27/54 → 15    Furnace       3 → 15
  Dispenser/Dropper     9 → 15        Hopper        5 → 15
  Brewing stand         4 → 15        Beacon        1 → 15
  Jukebox               1 → 15        Cauldron      lvl 0-3

Delay: 1 redstone tick for signal, 1 game tick for container polling
Output: STRONG power (same strength as input)
Locking: side repeater/comparator locks comparator output to current value
```

### 7.7 Piston Mechanics
```
General:
  - Push <= 12 blocks (piston cannot push more than 12)
  - Pull: sticky piston pulls 1 block on retract
  - 1 redstone tick delay for both extend and retract

Extension sequence (on power ON):
  1. Piston receives power (rising edge)
  2. 1 redstone tick delay (scheduled in block event queue)
  3. Check: can piston extend?
     a. Head destination must be air or replaceable
     b. Count push line blocks, must be <= 12
     c. No unpushable blocks in line
  4. If extendable: move push line, place piston head, update state

Retraction sequence (on power OFF):
  1. Piston loses power
  2. 1 redstone tick delay
  3. Sticky: pulls attached block back
     - If block is immovable → retracts without the block (block stays)
  4. Non-sticky: destroys head, attached block drops as item

Block dropping:
  - If a pushed block would enter a replaceable block (torch, flower, etc.):
    the replaceable block breaks (drops as item), pushed block moves in
  - If push into non-replaceable: piston won't extend

Quasi-Connectivity (QC):
  - Pistons, dispensers, droppers check for power at y AND y+1
  - Power source 1 block above and adjacent → powers device as if adjacent
  - Enables BUD behavior and vertical signal transmission
  - NOT affected: doors, trapdoors, lamps, note blocks

BUD (Block Update Detector):
  - Any mechanism leveraging piston's 1-tick delay + QC
  - Piston may not extend on same tick it receives power if update
    doesn't reach it (block update order dependency)
  - Observers replaced most BUD functionality in 1.12

Unpushable blocks (in 1.12.2):
  bedrock, obsidian, end_portal_frame, end_portal, end_gateway,
  piston_head, moving_piston, command_block, chain_command_block,
  repeating_command_block, structure_block, structure_void,
  ender_chest, beacon, mob_spawner
```

### 7.8 Observer Block
```
Function:
  - Detects block state changes in the block it faces
  - Fires 1-tick redstone pulse (2 game ticks) on detection
  - Detects ANY block state change (not just power)

Triggers:
  Block breaks, block placed, property changes (crop age, door open, lever flip)
  Liquid level change, piston extend/retract, ANY block state change

Non-triggers:
  Inventory changes, entity movement, player position
  Random ticks that don't change state, light level changes

Pulse timing:
  - Detection at end of game tick
  - Pulse emitted on next redstone tick (1 game tick delay)
  - Pulse duration: 1 redstone tick (2 game ticks)
  - Signal strength: 15

Implementation:
  - Stores "last observed state" per observer
  - Each tick compares current state of observed block
  - If changed: schedule 1-tick pulse, update stored state
```

### 7.9 Full Redstone Component Details
```
Piston (regular):        push ≤12, 1 tick delay, breaks torches/redstone on push
Sticky Piston:           push ≤12, pull 1 on retract, leaves block if can't pull
Dispenser:               fires item on redstone pulse
  - Arrows: shoots arrow projectile at player direction
  - Eggs/Snowballs: throws projectile
  - Water/Lava bucket: places liquid at facing
  - Flint & steel: creates fire at block in front
  - TNT: primes TNT (short fuse ~20 ticks)
  - Fire charge: shoots fire charge projectile
  - Splash potion: applies effect at block
  - Spawn egg: spawns mob at block
Dropper:                 ejects 1 item per pulse (always drops as item entity)
Hopper:                  pulls 1 item/4t from above, pushes 1/4t below
  - Locked: redstone signal → locked (cannot push/pull)
  - Collection: picks up item entities above it (8 tick cooldown)
Note Block:              25 semitones F#3-F#5, instrument based on block below
  Tuning: right-click cycles through 25 semitones
  Activation: redstone rising edge triggers one note
  Instruments:
    Planks (any) → Bass (double bass)
    Sand/Gravel/Soul Sand → Snare drum
    Glass/Glowstone/Sea Lantern → Clicks/Sticks
    Stone/Obsidian/Netherrack/Brick → Kick drum
    Clay/Hardened Clay → Flute
    Gold Block → Bell (glockenspiel)
    Wool (any) → Guitar
    Packed Ice → Chimes
    Iron Block → Xylophone (vibraphone)
Redstone Lamp:           15 light when powered, instant on/off
TNT:                     80 tick fuse (4s), explosion power 4
  - Primed by redstone, flint & steel, fire, or other explosions
  - Water-canceled: in water, explosion breaks no blocks (still damages)
  - Primed TNT is an entity with motion
Fence Gate:              opens when powered (closes when unpowered)
Door (iron):             opens when powered (no manual open)
Door (wood):             opens when powered, also opens by player
Trapdoor:                opens when powered
Weighted Pressure Plates:
  - Iron: min(floor(entities/10), 15)
  - Gold: min(entities, 15)
Tripwire:                string between hooks, entity breaks line → signal
  - Max length: 40 blocks between hooks
  - Shears: right-click to disarm

Rail powering:
  - Powered rail: activates when adjacent to powered block
  - Activator rail: activates when powered
  - Detector rail: outputs signal when minecart on it
```

### 7.10 Tick Order & Propagation
```
Each game tick (50ms):
  Phase 1: Process scheduled block events (piston extensions, etc.)
  Phase 2: Execute neighbor block updates (breadth-first)
    - Direction priority: -X, +X, -Z, +Z, -Y, +Y
  Phase 3: Entity redstone updates (TNT fuse, minecart)
  Phase 4: Post-update (torch burnout check, piston QC check, comparator poll)

Deterministic Behavior:
  - Update order consistent within game tick
  - FIFO within same priority
  - Same circuit → same behavior every time
  - Chunk borders may affect timing

Signal Propagation:
  - Redstone dust: instantaneous (all dust in 1 game tick)
  - Repeater: adds 1-4 redstone ticks delay
  - Comparator: 1 redstone tick delay
  - Torch: 1 redstone tick delay
  - Piston: 1 redstone tick delay
```

---

## P8 — UI/HUD (Exact 1.12.2 Replica)

### 8.1 Main Menu
- Panorama cube (6 faces) slow rotation
- "Minecraft" title logo (with fire animation)
- Subtitle: "Minecraft 1.12.2"
- Buttons: Singleplayer, Multiplayer (grayed), Options, Quit
- Footer: © Mojang AB

### 8.2 Title Screen (World List)
- Dirt/scorched grass background
- World list: name, mode, time, size, datestamp
- Buttons: Create New World, Edit, Delete, Re-Create, Cancel

### 8.3 Create New World
- World Name, Game Mode (S/C/H), Difficulty, Allow Cheats, Bonus Chest
- More Options: Seed, Generate Structures, World Type (Default/Superflat/Large Biomes/Amplified/Customized)
  - Customized: sliders for sea level, cave size, ravine size, etc. (1.12.2 only, removed in 1.13)
  - Debug world type also available (all block states as full grid, no interaction)
- Superflat Customize: presets list + custom code string

### 8.4 Options
- Music & Sound: Master, Music, Weather, Blocks, Hostile, Friendly, Players, Ambient, Subtitles
- Controls: Key bindings (rebind), Sensitivity, Invert, Auto-Jump
- Video Settings: Graphics (Fast/Fancy), Render Dist (2-32), Smooth Lighting, Brightness,
  3D Anaglyph, Fullscreen, V-Sync, View Bobbing, GUI Scale, Attack Indicator,
  Mipmap Levels (0-4), Entity Shadows, Clouds, Particles
- Language: list + Force Unicode
- Chat Settings: Shown/Commands/Hidden, Colors, Links
- Resource Packs: ordered list
- Narrator (1.12 NEW): Ctrl+B to toggle; options Off / Narrates Chat / Narrates System / Narrates Chat & System. Text-to-speech reads chat messages aloud.

### 8.5 HUD
```
Crosshair:        white angled lines, attack indicator below, block wireframe
Hotbar:           9 slots center-bottom, selected outline, count, durability bar
Health:           10 hearts, flash red, poison green, wither dark, absorption gold, hardcore X
Hunger:           drumsticks, shake when low, saturation shimmer
Armor:            chestplate icons ×10 (2 armor pts each)
Air:              bubble icons ×10, underwater only
XP:               green bar + level number
Scoreboard:       right side, objective+players
Boss Bar:         center-top, colored, boss name
Debug F3:         left: version/FPS/pos/block/chunk/facing/light/biome/difficulty
                  right: display/OpenGL/resource packs/CPU
Block break:      crack overlay 10 stages
```

### 8.6 Inventory Screens
```
Survival (E):      player model left, armor, offhand, 27+9 slots, 2×2 craft, recipe book
Creative:          12 tabs (Build/Decor/Redstone/Transport/Misc/Food/Tools/Combat/Brewing/Materials/Search/Survival)
  Search: text field, filtered
  Saved Toolbars tab (1.12 NEW): save/load hotbar presets. C+[1-9] to save current hotbar to slot. X+[1-9] to load. Blue border around saved slots when hovering.
Crafting Table:   3×3 + result + recipe book
Furnace:          fuel+input+output, flame+arrow animated
Brewing Stand:    3 bottles + ingredient + blaze powder, bubble animation
Enchanting Table: 3 random buttons, lapis cost, 1-30 levels
Anvil:            left+right+result, cost in levels (green/red), max 39
Chest:            27 (single) or 54 (double), labeled
Ender Chest:      27 per-player
Shulker Box:      27 portable
Beacon:           4 payment slots, primary+secondary effect
Dispenser/Dropper: 9 slots (3×3)
Hopper:           5 slots
Villager:         2 input + 1 result, trade list left
Horse:            1 saddle + 1 armor + 0-15 chest
```

### 8.6B Item Tooltip Rendering
```
Tooltip shown when hovering over an item in inventory/container for 0.5s+
Rendered as dark rectangular box with colored text, below the item slot

Tooltip content:
  Line 1: Item display name
    - White:     common items (stone, dirt, sticks)
    - Yellow:    rare items (music discs, enchantable gold/diamond items)
    - Aqua:      epic items (enchanted golden apple, enchanted book with treasure enchants)
    - Light purple: enchanted items (any item with enchantment glint)
    - Blue:      potions (with potion color tint applied to text)
    - Gray:      items with "No use" / decorative description
    - Italic:    items with custom name (renamed in anvil)

  Line 2+: Lore text (from {display:{Lore:["line1","line2",...]}} NBT tag)
    - Purple italic text for each lore line
    - Used by adventure maps, written books, custom items

  Attributes section (after lore):
    - Green:     positive attributes (+attack damage, +armor)
    - Red:       negative attributes (-attack speed, -armor)
    - Blue-gray: neutral attributes (attack range, luck)
    - Format: " §2<modifier> <operation> <value>" (e.g. " +8 Attack Damage")
    - Shown for: tools, weapons, armor, horse armor
    - Hidden if {HideFlags: 1} (or appropriate bitmask)

  Enchantments section:
    - Light gray text with blue enchant name
    - Format: "§7Enchantment Name §f<level>"
    - Roman numeral levels (I, II, III, IV, V)
    - Hidden if {HideFlags: 1}

  Additional info:
    - "§7§oCreative Only" for command-only items (knowledge book, command block)
    - "§7§oNot craftable" for certain items (enchanted golden apple, spawn eggs)
    - "§7∞ Infinite" for Infinity bow (durability still shown)
    - Potion effects: listed under potion name with duration
    - Book contents: "§9by <author>" for written books
    - Firework: flight duration shown ("Flight Duration: <1-3>")
    - Map: "§7<scale> zoom out" or "§7Fully explored" for locked maps

  Durability bar:
    - Below item name, only for tools/armor with durability
    - Green bar when full
    - Yellow bar when < 50% remaining
    - Red bar when < 10% remaining
    - Shows "durability / max_durability" numbers on right

  Item count:
    - Shown on right side of tooltip for stackable items
    - "x<count>" in gray text

  HideFlags bitmask (NBT):
    1   = Enchantments hidden
    2   = Attribute modifiers hidden
    4   = Unbreakable hidden
    8   = CanDestroy hidden
    16  = CanPlaceOn hidden
    32  = Additional (potion effects, book author, etc.) hidden
    63  = All flags
```

### 8.6C Container Shift-Click Rules
```
Shift-click behavior depends on the container type. Rules determine where items
move when player shift-clicks:

Survival Inventory (player inventory, press E):
  - Shift-click item in hotbar → moves to main inventory (27 slots)
  - Shift-click item in main inventory → moves to hotbar
  - Shift-click armor → equips if slot empty, otherwise moves to inventory
  - Shift-click offhand item → moves to inventory
  - Swaps between hotbar (9) and main inventory (27) only

Chest (single/double/ender chest/shulker box):
  - Shift-click in chest area → moves to player inventory (main → hotbar priority)
  - Shift-click in player area → moves to chest
  - Double chest: treats as single 54-slot container

Crafting Table (3×3):
  - Shift-click result → crafts maximum possible stack (SHIFT + click result = craft-all)
  - Shift-click ingredient in crafting grid → moves to player inventory (not back to container)
  - Shift-click in player inventory → normal hotbar↔main behavior

Furnace:
  - Shift-click input → moves to furnace input slot (top)
  - Shift-click fuel → moves to fuel slot (bottom-left)
  - Shift-click output → moves to player inventory
  - Shift-click in player inventory → tries fuel first, then input

Brewing Stand:
  - Shift-click ingredient (top) → moves to ingredient slot
  - Shift-click bottle → moves to bottle slots (bottom 3)
  - Shift-click blaze powder → moves to fuel slot
  - Shift-click in player area → tries ingredient, then fuel, then bottles

Enchanting Table:
  - Shift-click item → moves to enchanting slot (left center)
  - Shift-click lapis → moves to lapis slot (right center)
  - No shift-click in results (not applicable)
  - Shift-click in player area → tries item slot, then lapis

Anvil:
  - Shift-click item → moves to first slot (left)
  - Shift-click second item → moves to second slot (right, for repair/combine)
  - Shift-click result → takes result item
  - Shift-click in player area → tries first slot, then second

Dispenser/Dropper:
  - Shift-click in dispenser → moves to player inventory
  - Shift-click in player → moves to first available dispenser slot

Hopper:
  - Shift-click in hopper → moves to player inventory
  - Shift-click in player → moves to hopper slots

Villager Trading:
  - Shift-click trade result → performs trade up to stack limit
  - Shift-click input items → moves between player and trade slots normally

Beacon:
  - Shift-click payment item (emerald/diamond/ingot) → moves to payment slot
  - No other shift-click behavior in beacon

Horse:
  - Shift-click saddle → equips saddle slot
  - Shift-click horse armor → equips armor slot
  - Shift-click chest (donkey/mule) → moves to/from chest inventory
```

### 8.7 Pause Menu
- Dirt bg transparent
- Back to Game, Advancements, Statistics, Save&Quit, Quit
- Options button

### 8.8 Advancements (Complete 1.12.2)

**Data format**: JSON file per advancement in `data/advancements/` folder loaded at startup.

**Tree structure** (5 root tabs — exact 1.12.2 tree):
```
Minecraft (story) tab:
  - Minecraft (root): have crafting table in inventory
    - Stone Age: cobblestone in inventory
      - Getting an Upgrade: stone pickaxe in inventory
        - Acquire Hardware: iron ingot in inventory
          - Suit Up: iron chestplate/leggings/boots/helmet in inventory
            - Not Today, Thank You: block projectile with shield
          - Hot Stuff: lava bucket in inventory
            - Ice Bucket Challenge: obsidian in inventory
              - We Need to Go Deeper: enter Nether dimension
                - Zombie Doctor: cure zombie villager (weakness + golden apple)
                - Eye Spy: enter a stronghold
                  - The End?: enter The End dimension
          - Isn't It Iron Pick: iron pickaxe in inventory
            - Diamonds!: diamond in inventory
              - Cover Me With Diamonds: diamond armor in inventory
              - Enchanter: enchant an item at enchanting table

Nether tab:
  - Nether (root): enter Nether dimension
    - Return to Sender: kill ghast with fireball (return fireball to ghast)
      - Uneasy Alliance: bring ghast to Overworld and kill it
    - A Terrible Fortress: enter a Nether Fortress
      - Spooky Scary Skeleton: get wither skeleton skull
        - Into Fire: get blaze rod
          - Local Brewery: brew a potion (bring ingredient + water bottle to stand)
            - A Furious Cocktail: have all 13 potion effects active simultaneously
              - How Did We Get Here: have all 26 status effects simultaneously
          - Withering Heights: wither is spawned (summoned)
            - Bring Home the Beacon: build a beacon pyramid (activate beacon)
              - Beaconator: build a full-power beacon (4-tier pyramid)
          - Not Quite "Nine" Lives: use totem of undying to survive death

The End tab:
  - The End (root): enter the End dimension
    - Free the End: kill the Ender Dragon
      - The Next Generation: obtain dragon egg
        - Remote Getaway: use an end gateway (throw pearl / step through)
      - The City at the End of the Game: enter an End City
        - Sky's the Limit: obtain elytra
          - Great View From Up Here: fly 50+ blocks with elytra

Adventure tab:
  - Adventure (root): kill any mob or take damage
    - Monster Hunter: kill any hostile mob (zombie, skeleton, etc.)
      - Postmortal: use a totem of undying to cheat death
    - Hired Help: summon an iron golem (4 iron + pumpkin)
    - Sweet Dreams: sleep in a bed to set spawn and skip night
    - What a Deal: successfully trade with a villager (accept offer)
    - Take Aim: shoot a bow
      - Sniper Duel: kill a skeleton from 50+ meters horizontally
    - Adventure Time: visit all available biomes (must leave and re-enter)

Husbandry tab:
  - Husbandry (root): breed two animals with their food
    - Best Friends Forever: tame a wolf or parrot
    - Two by Two: breed each breedable animal type at least once
```

**Advancement properties**:
```json
{
  "display": {
    "icon": "minecraft:diamond",
    "title": {"text": "Diamonds!"},
    "description": {"text": "Acquire diamonds"},
    "frame": "task",      // task, goal, or challenge
    "show_toast": true,
    "announce_to_chat": true,
    "hidden": false,
    "background": "minecraft:textures/gui/advancements/backgrounds/adventure.png"
  },
  "parent": "minecraft:story/iron_tools",
  "criteria": {
    "diamond": {
      "trigger": "minecraft:inventory_changed",
      "conditions": {
        "items": [{"item": "minecraft:diamond"}]
      }
    }
  },
  "rewards": {    // optional
    "experience": 100,
    "loot": ["minecraft:advancements/diamond_reward"]
  }
}
```

**Trigger types** (1.12.2):
```
minecraft:bred_animals            minecraft:brewed_potion
minecraft:changed_dimension       minecraft:construct_beacon
minecraft:consume_item            minecraft:cured_zombie_villager
minecraft:effects_changed         minecraft:enchanted_item
minecraft:enter_block             minecraft:entity_hurt_player
minecraft:entity_killed_player    minecraft:filled_bucket
minecraft:fishing_rod_hooked      minecraft:player_generates_container_loot
minecraft:placed_block            minecraft:player_killed_entity
minecraft:inventory_changed       minecraft:levitation
minecraft:lightning_strike        minecraft:nether_travel
minecraft:player_hurt_entity      minecraft:recipe_unlocked
minecraft:slept_in_bed            minecraft:summoned_entity
minecraft:tame_animal             minecraft:tick
minecraft:used_ender_eye          minecraft:used_totem
minecraft:villager_trade
```

**Toast notifications**:
- Frame type determines color: `task`=gray, `goal`=pink, `challenge`=purple
- Slide in from top-right, stay ~5s, then slide out
- Icon + title + description
- Sound: `ui.toast.challenge_complete` for challenges

**Advancements GUI**:
- Tabs at top for each root (Minecraft/Nether/End/Adventure/Husbandry)
- Graph view: nodes connected by lines (parent→child)
- Scrolling + zoom
- Hover: show title + description
- Completed: full color with green outline
- Unlocked: full color
- Locked: dark/desaturated with "???"
- Progress bar for criteria-based advancements (e.g. "20/40 biomes visited")

### 8.9 Statistics (Complete)
```
Statistics screen accessed from pause menu (Statistics button, below Advancements).
Opens a full-screen GUI with 5 tabs: General, Blocks, Items, Entities, Mobs.

General tab — single-column list of tracked actions:
  Stat name                          | Trigger
  Games Played                       | Each world load +1
  Minutes Played                     | Per tick
  Distance Walked                    | Per block footstep (on ground)
  Distance Crouched                  | Per sneaking step
  Distance Swum                     | Per block swum
  Distance Fallen                    | Per block fallen (y delta)
  Distance Climbed                   | Per ladder/vine step
  Distance Flown                     | Per block flown (creative/elytra)
  Distance Dove                      | Per block underwater downward
  Distance Travelled by Minecart     | Per block in minecart
  Distance Rowed                    | Per block in boat (no "row" in 1.12.2 — "Distance by Boat")
  Distance by Horse                  | Per block on horse
  Distance by Pig                    | Per block on pig
  Distance by Elytra                 | Per block gliding (1.12 NEW)
  Jump Count                         | Per jump
  Damage Dealt                       | Sum of damage dealt to entities
  Damage Taken                       | Sum of damage received
  Death Count                        | Each death +1
  Mob Kills                          | Each hostile mob kill +1
  Animals Bred                       | Each successful breeding +1
  Player Kills                       | Each PvP kill +1
  Fish Caught                        | Each fishing catch +1
  Junk Fished                        | Each junk catch
  Treasure Fished                    | Each treasure catch
  Enchant Item                       | Each enchantment table use
  Inspect Dispenser                  | Each dispenser GUI open
  Talk to Villager                   | Each villager trade GUI open
  Trade with Villager                | Each completed trade
  Drop                              | Each item dropped (Q key)
  Leave Game                         | Each save-and-quit
  Play One Minute                    | 1 minute played = 1200 ticks
  Time Since Death                   | Ticks since last death
  Sneak Time                         | Cumulative ticks sneaking
  Walk One Centimeter                | 1 cm walked (all distance tracked in cm internally)
  Crouch One Centimeter              | Sneak distance in cm
  Swim One Centimeter                | Swim distance in cm
  Fall One Centimeter                | Fall distance in cm
  Climb One Centimeter               | Climb distance in cm
  Fly One Centimeter                 | Fly distance in cm
  Dive One Centimeter                | Dive distance in cm
  Minecart One Centimeter            | Minecart distance in cm
  Boat One Centimeter                | Boat distance in cm
  Pig One Centimeter                 | Pig-riding distance in cm
  Horse One Centimeter               | Horse-riding distance in cm
  Sprint One Centimeter              | Sprint distance in cm
  Avian One Centimeter               | Elytra distance in cm (1.12 NEW)

Blocks tab — tracked per block type:
  Mined: <block>    — Each time player breaks a specific block type
  Picked Up: <block> — Each time player picks up a specific block item
  Dropped: <block>   — Each time player drops a specific block item
  Crafted: <block>   — Each time player crafts a specific block (count per crafted output)
  Used: <block>      — Each time player places a specific block
  Broken: <block>    — Each time a specific block-type item breaks (tool/armor)

Items tab — tracked per item type:
  Picked Up: <item>  — Each pickup
  Dropped: <item>    — Each drop
  Crafted: <item>    — Each crafted output
  Used: <item>       — Each usage (eat food, drink potion, equip armor, fire bow, etc.)
  Broken: <item>     — Each item break

Entities tab:
  Killed: <entity>   — Each kill of a specific mob type
  Killed By: <entity> — Each death to a specific mob type

Sorting: click column header to sort ascending/descending
Search: type to filter (in creative search style)
```

### 8.10 F3 Debug Screen (Exact 1.12.2 Layout)
```
Press F3 to toggle debug overlay. Left side of screen shows game info,
right side shows system info. Press F3+Q for help, F3+F for render distance cycle.

Left side (top→bottom):
  Minecraft 1.12.2 (<fps> fps, <chunk_updates> chunk updates)
  C: <caves> | D: <distance> | E: <server_chunks> | F: <frustum>
  P: <particles> | DP: <dripping>
  <render_distance> chunks render distance
  <graphics_mode>: <fast|fancy|fabulous>     (only fast/fancy in 1.12.2)
  VSync: <on|off> | Cloud: <fast|fancy> | Bright: <brightness>
  x: <pos_x> | y: <pos_y> | z: <pos_z>
  Block: <block_x> <block_y> <block_z>
  Facing: <direction> (towards <axis> <sign>) — e.g. "north (-z)"
  Chunk: <chunk_x> <chunk_z> relative: <rel_x> <rel_z>
  Chunk-relative: <block_in_chunk_x> <block_in_chunk_z>
  Light: <sky_light_level> (<block_light_level>)
  Biome: <biome_name>
  Local Difficulty: <local_difficulty> /* <clamped_difficulty>
  Day: <day_number>
  Looking at block: <block_id>:<metadata> <resource_location>
  Looking at block data: <block_entity_data_if_present>
  Looking at liquid: <liquid_if_any>
  Entity: <entity_being_looked_at>

Right side (top→bottom):
  Display: <screen_width>x<screen_height> (<gui_scale>x)
  OpenGL: <opengl_version>
  OpenGL: <opengl_renderer>
  OpenGL: <display_vendor> <display_version>
  CPU: <cpu_model> <cpu_speed>
  <memory_usage>MB / <max_memory>MB (free: <free_memory>MB)
  Resource packs: <count>
  <resource_pack_list>
  Active renderer: <renderer_used>
  Ticking: <ticking_stats>

F3 + Q: show F3 + key list (displays on screen)
F3 + F: cycle render distance (2, 3, 4, 5, 6, 7, 8, 10, 12, 14, 16, 24, 32)
F3 + A: reload all chunks
F3 + B: show hitboxes (entity collision boxes + look vector)
F3 + D: clear chat history
F3 + G: show chunk boundaries (overlay wireframe)
F3 + N: cycle creative/spectator
F3 + P: auto-pause on focus lost toggle
F3 + S: reload all textures/sounds
F3 + T: reload all textures (block/item/entity models)
F3 + C: copy location as /teleport command (hold 3s triggers debug crash, removed in later versions)
```

### 8.11 Language / Translation System
```
Language file format: standard Java .lang properties file
  Stored at: @minecraft/lang/<language_code>.lang
  Format: key=value (one per line)
  Comment lines start with #
  Encoding: ASCII with unicode escapes (\uXXXX) for special chars
  Example:
    tile.stone.name=Stone
    item.diamond.name=Diamond
    container.chest=Chest
    key.forward=Forward
    deathScreen.attackEntity=%1$s was slain by %2$s

Language selection via Options → Language screen
  List shows all detected .lang files in @minecraft/lang/
  Current language highlighted
  "Force Unicode Font" toggle checkbox (for CJK/Unicode-reliant languages)
  Language change applies immediately (no restart needed)

Key categories of translation keys:
  tile.<block_name>.name          — Block display names
  item.<item_name>.name            — Item display names
  entity.<entity_name>.name        — Entity display names
  container.<gui_name>             — Container window titles
  key.<key_action>                 — Key binding descriptions
  deathScreen.<death_type>         — Death messages
  deathScreen.<entity>             — "was slain by" messages
  deathScreen.attackEntity         — Generic attack death
  deathScreen.<damage_source>      — Specialized death messages
  deathScreen.<damage_source>.player — PvP death variants
  deathScreen.title                — "Game Over!" title
  deathScreen.respawn              — "Respawn" button
  deathScreen.titleScreen          — "Title Screen" button
  deathScreen.spectate             — "Spectate World" (hardcore)
  deathScreen.deleteWorld          — "Delete World" (hardcore)
  selectWorld.<action>             — World selection UI
  createWorld.<option>             — World creation UI
  options.<option>                 — Options menu UI
  gui.<element>                    — General GUI elements
  stat.<stat_name>                 — Statistic names
  achievement.<advancement>        — Advancement titles (pre-1.12 localization)
  advanceMode.<tab>                — Advancement tab names (1.12)
  chat.type.<chat_type>            — Chat message formats
  commands.<command>.<description> — Command descriptions
  subtitles.<sound_event>          — Subtitle text
  item.<item>.tooltip              — Item tooltip additional info
  item.modifiers.<slot>            — Attribute modifier labels
  enchantment.<enchant_name>       — Enchantment names
  potion.<effect_name>             — Potion effect names
  biome.<biome_name>               — Biome display names
  entity.<entity>.death            — Death message for entity kills
  death.attack.<damage>            — Attack damage death messages

Implementation: load .lang file into Map<string,string>, access via translation function
  Fallback: if key not found, show key itself (with "translate " prefix in some UI contexts)
  Variable substitution: %s, %1$s, %d, etc. inserted via string formatting
```

### 8.12 Key Bindings (Default List)
```
Accessed via Options → Controls → Key Bindings
Rebindable; shows "key.conflict" if two actions bound to same key.

Default bindings (1.12.2):
  Key                        Default      Action
  key.forward                W            Move forward
  key.left                   A            Strafe left
  key.back                   S            Move backward
  key.right                  D            Strafe right
  key.jump                   Space        Jump
  key.sneak                  Shift        Sneak (hold to toggle in options: "Toggle Sneak")
  key.sprint                 Ctrl         Sprint (hold to toggle in options: "Toggle Sprint")
  key.inventory              E            Open/close inventory
  key.swapOffhand            F            Swap items between main hand and offhand
  key.drop                   Q            Drop held item (1 at a time) / Ctrl+Q = drop stack
  key.use                    Right Click  Use/place item, interact with block
  key.attack                 Left Click   Attack/mine block
  key.pickItem               Middle Click Pick block from world (creative)
  key.chat                   T            Open chat
  key.command                /            Open command window
  key.advancements           L            Open advancements (1.12 NEW)
  key.smoothCamera           F8           Toggle cinematic camera
  key.playerlist             Tab          Show player list (hold)
  key.hotbar.1-9             1-9          Select hotbar slot
  key.hotbar.save            C+1-9        Save hotbar preset (1.12 NEW creative)
  key.hotbar.load            X+1-9        Load hotbar preset (1.12 NEW creative)
  key.narrator               Ctrl+B       Toggle narrator (1.12 NEW)
  key.screenshot             F2           Take screenshot
  key.togglePerspective      F5           Cycle 1st person / 3rd person back / 3rd person front
  key.toggleFog              F (but reassigned in 1.12?) F3+F = cycle render distance
  key.debugScreen            F3           Toggle debug screen
  key.mouseSetting           —            Not a binding

Mouse settings (not rebindable in game):
  Mouse sensitivity slider: 0-200% (default 100%)
  Invert mouse: toggle (up = look up, inverted: up = look down)
  Mouse wheel: scroll inventory/hotbar
  "Discrete mouse scrolling" option (1.12 NEW): scroll jumps between slots, no smooth scroll
```

### 8.13 Panorama Rendering & Main Menu
```
Main menu panorama: render of 6-sided cube map rotating slowly behind menu UI.
The cube map is loaded from 6 textures:
  @minecraft/textures/gui/title/background/panorama_0.png through panorama_5.png
  (0=right, 1=left, 2=top, 3=bottom, 4=front, 5=back)

Rendering technique:
  - Render a cube around the camera (skybox style)
  - Camera positioned at center, looking outward through each face
  - Use a 90° FOV per face (standard cube map)
  - Rotation: camera slowly rotates around Y axis over time
    - Base rotation applied before rendering: yaw = time * 0.001 (about 6°/sec)
    - Optional: slight pitch oscillation (sin(time * 0.0005) * 3°)
  - Render behind all UI elements (z-buffer cleared first)
  - Use linear filtering for smooth appearance

Main menu layout (1.12.2):
  Panorama cube as background (slow rotation)
  Overlay: dark gradient vignette (blacks out edges, centers towards mid-gray)
  Top-center: "Minecraft" title logo (texture: @minecraft/textures/gui/title/minecraft.png)
    - With flame animation overlay (fire texture alpha-blended over logo)
    - Flame animation: uses fire_layer_0.png shifting upward over time
    - Below logo: "Minecraft 1.12.2" subtitle texture (or "Minceraft" on 1/10000 chance)
  Buttons (centered, vertical stack):
    [Singleplayer]
    [Multiplayer] (grayed out — single-player only)
    [Options...]
    [Quit Game]
  Bottom-center: Copyright "Copyright Mojang AB. Do not distribute!"
    - Text in small gray font
  Bottom-right: Version info ("Minecraft 1.12.2")
    - Clickable: opens version-independent server list (not relevant here)

Button rendering:
  - Use @minecraft/textures/gui/widgets.png for button sprites
  - Button states: normal (dark gray), hover (white border), pressed (inset)
  - Text: white centered, shadow underneath
  - Width: 200px standard, height: 20px

Panorama loading:
  - On startup: load all 6 textures into cube map
  - On panorama rotation: apply texture to skybox shader
  - Should be pre-loaded during initial asset loading phase
  - Memory: 6 × 1024×1024 RGBA = ~24MB uncompressed (lower res possible for performance)
```

### 8.14 Status Effect Icons on HUD
```
Status effects display on the HUD in two locations:
  - Top-right corner (in-game): active effects shown as icon tiles
  - Inventory screen (right side): detailed effect list with durations

In-game HUD display (top-right):
  - Active effects shown vertically as 24×24 icon tiles (spaced 4px apart)
  - Stacked from top of screen, right-aligned
  - Max visible: 12+ effects (scrollable if more? No — no scroll in vanilla)
  - Each tile shows:
    - Effect icon from @minecraft/textures/gui/container/inventory.png (effect icon sheet)
    - Icon tinted based on effect type (potion color)
    - Icon background: dark (effect level) or brighter (ambient/beacon effect)
    - Level indicator: no text level shown on HUD (only in inventory)
    - Duration: shown below icon in inventory screen, not on HUD
  - Icon sources from GUI texture sheet: effects are 18×18 sprites in 256×256 texture
  - Effect ordering: from top: most recently applied → earliest applied (new at top)
  - Particle effect: if effect has particles=false → icon still shown but with no particles
  - HUD icon uses: potion effects, beacon effects, conduit power (1.13+), bad omen (1.13+)
  - Hover on icon: shows effect name + amplifier + time remaining (1.12 has this? Actually 1.12 shows effect name + duration on hover of icon)
    Actually in 1.12.2: hovering over an active effect icon shows tooltip with:
      "Effect Name [Amplifier]" (e.g. "Strength I")
      "0:45" (formatted duration)

Inventory screen effect display (right side):
  - Shown when inventory/container is open and player has active effects
  - Vertical list with icons + text details
  - Each entry: icon + effect name + amplifier (Roman numeral) + duration
  - Background: dark semi-transparent panel right of inventory grid
  - Source indicator: "(from beacon)" for beacon effects, "(from potion)" for potions
  - Effect removal: not possible from this screen

Effect icon atlas locations (in @minecraft/textures/gui/container/inventory.png):
  Each effect icon is 18×18 within the inventory texture sheet.
  Positions (x, y offset from top-left corner of texture):
    speed=0,0      slowness=18,0    haste=36,0    mining_fatigue=54,0
    strength=72,0  health=90,0      damage=108,0  jump_boost=126,0
    nausea=144,0   regen=162,0      resist=180,0  fire_resist=198,0
    water_breath=216,0   invis=234,0   blind=0,18     night_vision=18,18
    hunger=36,18   weakness=54,18    poison=72,18   wither=90,18
    health_boost=108,18  absorption=126,18  saturation=144,18
    glowing=162,18  levitation=180,18  luck=198,18  bad_luck=216,18
```

### 8.15 Boss Bar Rendering
```
Boss bar displayed above HUD, centered at top of screen.
Only active when a boss entity (ender dragon, wither) is within range.
Single boss bar at a time (older bosses replaced by newer ones).

Visual rendering:
  Background: dark brown/black bar (200px wide? Actually: full screen width bar)
  Width: fills entire screen width (scales with GUI scale)
  Height: ~12px (varies with GUI scale)
  Position: ~2-3px below top of screen, above notification area
  Texture: @minecraft/textures/gui/bars.png (contains 5 bar color variants)

Bar colors (5 variants):
  Health bar (pink/purple):    default ender dragon — texture index 0
  Progress bar (blue):         wither — texture index 1
  Notch bar (white):           unused? Actually white is index 2
  End crystal (pink-purple):   protecting end crystal — index 3
  Wither (purple):             actual wither bar — index 4

  Actually the texture strip is divided into 5 sections, each containing:
    - Background (dark)
    - Filled foreground (colored)
    - Overlay notch pattern (repeated every few pixels)
  
  The bar shows filled ratio = boss_health / boss_max_health
  Smooth transition: health changes animate linearly over ~0.5s

Boss name:
  Displayed as centered text ABOVE the bar
  Text: "name" (white, centered, shadow)
  For ender dragon: "Ender Dragon"
  For wither: "Wither"
  For named bosses with custom name tag: shows custom name

Boss bar activation conditions:
  - Ender Dragon: active when dragon exists in The End dimension and player is in End
  - Wither: active when wither exists within 100 blocks of player
  - Both: if both active within range, the most recently damaged one takes priority
  - Deactivation: boss dies → bar drains and disappears

Implementation notes:
  - Separate render pass after hotbar, before subtitles
  - Use GUI scale for sizing (divide by guiScale factor)
  - Boss bar should not overlap with hotbar or XP bar
  - Only one boss bar shown at a time (no stacking in 1.12.2)
```

### 8.16 Loading Screen with Progress
```
Loading screen displayed during:
  - World creation (generating terrain)
  - World loading (reading save data, building initial chunks)
  - Dimension travel (portal loading)

Screen layout:
  Background: dirt/scorched grass texture (same as world list background)
    - @minecraft/textures/gui/options_background.png (tiled dark dirt)
  Center: "Loading..." text (white, large, shadow)
  Below text: progress bar (gray/border with green filled section)
    - Width: ~200px, Height: ~16px
    - Border: dark gray outline
    - Fill: green gradient (from left to current progress)
    - Progress range: 0% to 100%
  Below bar: context text (e.g. "Building terrain...", "Loading chunks...")
    - Gray italic text
  Below bar: random loading hint (e.g. "Tip: Press F3 to view debug info")
    - Read from tips list (hardcoded or from lang file)
    - Gray text, smaller font

Loading sequence:
  1. "Loading world data..." — Read NBT, player data, chunk metadata
  2. "Building terrain..." — Initial chunk generation (spawn area, ~21×21 chunks)
  3. "Preparing spawn area..." — Populating initial chunks with features
  4. "Loading chunks..." — Subsequent chunk loading (progress updates as chunks load)

Progress calculation:
  Build terrain progress = chunks_generated / spawn_area_total (21×21 = 441)
  Overall progress = weighted average of phases

Loading screen implementation:
  - Separate render canvas or overlay mode (no 3D rendering yet)
  - Render each frame (not blocked by loading — use progressive chunk gen in workers)
  - Cancel button: "Cancel" at bottom (returns to title screen)
  - On slow load: loading bar may pause at certain percentages (normal — chunk gen catching up)
```

### 8.17 End Credits Scroll
```
Triggered after killing Ender Dragon (walk through the portal at end island).
Displays scrolling text over black background.

Layout:
  Background: solid black (#000000)
  Text: white, centered horizontally, with shadow
  Font: Minecraft default font (monospace)
  Scroll speed: ~2-3 lines per second (smooth upward scroll)

Content structure:
  1. Title: "Minecraft" (large centered, ~2× size) followed by "1.12.2"
  2. Subtitle: "by" (small text)
  3. "Mojang AB" (large, centered)
  4. Credits section headers and names (centered, stepped):

  Credits sections:
    "Minecraft - 1.12.2"
    "Game developed by:"
      Markus Persson (creator)
      Jens Bergensten (lead designer)
      Jeb_ (game developer)
      Dinnerbone (game developer)
      Searge (game developer)
      ...etc
    
    "Art by:"
      Kristoffer Zetterstrand (painter)
      Markus Toivonen (artist)
      ...etc
    
    "Music and Sound by:"
      C418 (Daniel Rosenfeld)
      Lena Raine (not in 1.12.2)
      ...etc
    
    "Special thanks:"
      The Minecraft community
      All Mojang employees
      ...etc

  5. Final line: "Presented by Mojang AB" (large, centered)
  6. After credits: returns to title screen automatically

Implementation:
  - Text stored at @minecraft/texts/credits.txt (or lang file entries)
  - Render off-screen text to texture, scroll up
  - Speed: ~2 seconds per visible screen of text
  - Total playback: ~5-7 minutes (all credits)
  - Skip: press Escape → Skip credits → return to title screen
  - On player death during credits: no (credits play after dragon fight in survival)
  - After credits: player respawns at world spawn with "The End" advancement

Technical notes:
  - Credits stored as simple text file (line-by-line)
  - Empty lines = paragraph breaks (extra spacing)
  - Lines starting with space = centered differently (section headers)
  - Lines starting with tab = indent (sub-entries)
  - Color codes (§) supported for colored text
```

### 8.18 Attack Indicator / Cooldown Visual
```
Displayed below the crosshair on the HUD.
Shows weapon charge level for the attack cooldown system (added in 1.9).

Visual:
  Shape: small white sword/axe icon (scaled based on charge)
    - Actually: a small sword-shaped indicator below the crosshair
    - Empty (no weapon): not shown
    - Charging: fills from bottom to top (like a progress bar)
    - Fully charged: full sword icon, white/yellow
    - Partial: gray, partially filled
  Position: 10px below crosshair center
  Size: ~8×8 pixels (scales with GUI scale)
  Color:
    - White at full charge
    - Gray at partial charge
    - Not shown for items that don't use cooldown (food, potions, etc.)

Attack cooldown formula:
  Cooldown time = 1 / attack_speed (in seconds)
  Attack speed values:
    Sword:      1.6  → 0.625s cooldown
    Pickaxe:    1.2  → 0.833s
    Axe:        0.8  → 1.25s  (fastest! Actually slowest — 0.8 is slow attack speed)
    Shovel:     1.0  → 1.0s
    Hoe:        1.0  → 1.0s
    Trident:    1.1 (not in 1.12.2)
    Fist:       2.0  → 0.5s (fastest)
    Bow:        1.0  → 1.0s (full charge 1.5s)
    Shield:     3.0  → 0.333s

Damage scaling with cooldown charge:
  damage = base_damage × (0.2 + 0.8 × charge)
  Where charge = 0.0 (just attacked) to 1.0 (fully cooled)
  Sweep attack: only when charge = 1.0 (fully charged sword swipe)
  Critical hit: only when charge > 0.85 and falling

Settings dependency:
  Options → Video Settings → Attack Indicator
  Options: "Off" (default prior to 1.9?), "Crosshair" (below crosshair), "Hotbar" (durability bar style)
  Default in 1.12.2: Crosshair indicator
  Hotbar mode: shows as thin bar on selected hotbar slot (below item stack count)

Implementation:
  - Render after crosshair, before item name tooltip
  - Update every tick (charge level changes continuously)
  - Use sprites from GUI texture (crosshair variants in widgets.png or icons.png)
  - Crosshair itself also changes: wider/thinner based on charge
  - Bow charge: crosshair spreads as bow is drawn (wider = more inaccurate when not fully drawn)
```

### 8.19 Superflat Preset Codes
```
Superflat world preset codes (text string in Create New World → More Options).
Format: <version>;<biome_id>;<layer_specs>[;options]
  version: always 1 (or 2 in later versions? 1 = pre-1.13)
  biome_id: numeric biome ID (e.g. 1 = plains)
  layer_specs: semicolon-separated block ID:count pairs
  options: optional JSON for villages, decorations, etc.

Examples:
  Classic flat (vanilla):
    1;1;2x7,3x3,12x1,13x24;biome_1,decoration,stronghold
    = 2 layers of dirt (ID 2, depth 7), 3 layers of stone (ID 3, depth 3), etc.

  Standard superflat presets (1.12.2):
    1. Classic Flat (preset "classic_flat"):
       1;1;2x7,3x3,12x1,13x24;biome_1,decoration,stronghold
       Layers (bottom→top): bedrock×1, stone×3, dirt×7
       Biome: plains (ID 1), Features: decoration, stronghold, village, lakes
    
    2. Overworld (preset "overworld"):
       1;1;1x1,3x2,12x1,13x1;biome_1,village,decoration,stronghold,lake,lava_lake
    
    3. Snowy Kingdom:
       1;11;12x1,13x20;biome_11,village,decoration
    
    4. Bottomless Pit:
       1;1;13x1;biome_1,decoration,stronghold
       (only 1 layer of bedrock — void below)
    
    5. Desert:
       1;2;12x1,13x10,24x90;biome_2,decoration,stronghold
       Layers: 1 bedrock, 10 stone, 90 sand
    
    6. Redstone Ready:
       1;1;13x20,3x3,12x1;biome_1,decoration,stronghold
       (used for redstone testing, flat at y=24)
    
    7. Water World:
       1;1;12x1,9x90,13x1;biome_1,decoration,stronghold
       (90 layers of water above bedrock with a single layer of stone at top? No... 
        Actually preset: 1;1;12x1,9x5,13x10;biome_1,decoration)

  Custom user presets: edit the preset code string directly in the text field
  Preset list: options → preset selector dropdown with the 7 presets
  Icons: each preset has an icon in @minecraft/textures/gui/presets/

Options JSON format:
  {
    "biome":1,
    "features":{
      "village": true,
      "decoration": true,
      "stronghold": true,
      "lake": true,
      "lava_lake": true,
      "dungeon": true,
      "mineshaft": true,
      "monument": true,
      "fortress": true,
      "temples": true,
      "endcity": true,
      "mansion": true
    },
    "layers": [
      {"block": "minecraft:bedrock", "height": 1},
      {"block": "minecraft:stone", "height": 5},
      {"block": "minecraft:dirt", "height": 3},
      {"block": "minecraft:grass", "height": 1}
    ]
  }
```

### 8.20 Customized World Type Sliders
```
Customized world type (available in World Type dropdown, 1.12.2 exclusive — removed in 1.13).
Opens a screen with sliders for all world generation parameters.

Categories and sliders:
  Main Settings:
    - Sea Level (default: 63, range: 1-255)             — Water level Y coordinate
    - Caves (default: ON, toggle)                        — Generate caves
    - Dungeons (default: ON, toggle)                     — Generate dungeons
    - Fortresses (default: ON, toggle)                   — Generate Nether fortresses
    - Mineshafts (default: ON, toggle)                   — Generate mineshafts
    - Strongholds (default: ON, toggle)                  — Generate strongholds
    - Villages (default: ON, toggle)                     — Generate villages
    - Temples (default: ON, toggle)                      — Desert/jungle/witch huts
    - Monuments (default: ON, toggle)                    — Ocean monuments
    - Ravines (default: ON, toggle)                      — Generate ravines
    - Water Lakes (default: ON, toggle)                  — Surface water lakes
    - Lava Lakes (default: ON, toggle)                   — Surface/underground lava lakes
    - Dungeon Count (default: 8, range: 1-100)           — Dungeons per chunk
    - Water Lake Rarity (default: 4, range: 1-100)       — Lower = more common
    - Lava Lake Rarity (default: 8, range: 1-100)

  Biome Settings:
    - Biome Size (default: 4, range: 1-8)                — Larger = bigger biomes
    - River Size (default: 4, range: 1-5)                — River width/occurrence
    - Biome Depth/Scale (default range varies)
    - Biome Temp/Humidity (default range varies)

  Ore Settings (per ore type):
    - <ore> Spawn Size (vein size)
    - <ore> Spawn Tries (veins per chunk)
    - <ore> Min Height (lowest y-level)
    - <ore> Max Height (highest y-level)
    Applies to: Coal, Iron, Gold, Redstone, Lapis, Diamond, Emerald, Dirt, Gravel,
                Granite, Diorite, Andesite

  Terrain Settings:
    - Main Noise Scale (default: 80.0, range: 1-500)
    - Depth Noise Scale (default: 200.0, range: 1-500)
    - Depth Noise Exponent (default: 0.5, range: 0.01-20.0)
    - Depth Base Size (default: 8.5, range: 1-25)
    - Coordinate Scale (default: 684.412, range: 1-5000)
    - Height Scale (default: 684.412, range: 1-5000)
    - Height Stretch (default: 12.0, range: 0.1-50)
    - Upper Limit (default: 512.0, range: 1-5000)
    - Lower Limit (default: 512.0, range: 1-5000)

  Each slider shows current value + default indicator
  "Done" button applies customizations, "Cancel" reverts to default
  Preset: "Default" button resets all sliders to vanilla default values
  Note: Customized worlds with extreme settings may cause generation errors or unplayable terrain
  This screen is a 1.12.2 exclusive feature — removed in 1.13 in favor of datapack-based customization
```

---

## P9 — Complete Crafting Recipes (1.12.2)

### Notation
```
Grid patterns use 3×3 crafting grid (unless noted 2×2 for inventory crafting):
  ABC       A = row 1, col 1   D = row 2, col 1   G = row 3, col 1
  DEF       B = row 1, col 2   E = row 2, col 2   H = row 3, col 2
  GHI       C = row 1, col 3   F = row 2, col 3   I = row 3, col 3

Shorthand:
  "v" = vertical column (e.g., "3 sticks (v)" = 3 sticks in same column)
  "h" = horizontal row
  "∅" = empty slot
  "×" = stack size is variable by material
  ┌─┬─┬─┐                  ┌─┬─┬─┐
  │ │ │ │  e.g. Chest =    │P│P│P│  (all planks, center empty)
  │ │∅│ │                  │P│∅│P│
  │ │ │ │                  │P│P│P│
  └─┴─┴─┘                  └─┴─┴─┘

Wood material variants ×6: Oak, Spruce, Birch, Jungle, Acacia, Dark Oak
  When "any planks" is listed, ANY of the 6 types can be used (mix freely or same type).
  When "oak planks" is specified, only oak works.
  Durability/color may vary by wood type (boats, doors, signs use plank color).
```

### 9.1 Basic Materials & Components
```
Item (output count)      Pattern                        Input
──────────────────────────────────────────────────────────────────────
Planks (4)               1 log                           Any log (oak/spruce/birch/jungle/acacia/darkoak)
Sticks (4)               [P] [P]   (2×2)                 Any planks (2)
                          [P] [P]
                         Or v: [P] [P]                   
Chest                    3×3 all planks, center empty    Any planks (8)
Crafting Table           2×2 planks                      Any planks (4)
Furnace                  3×3 all cobble, center empty    Cobblestone (8)
Ladder (3)               7 sticks in H shape             Sticks (7)
                          [ ][S][ ]
                          [S][S][S]
                          [S][ ][S]
Trapdoor (2)             6 planks in h 2×3               Any planks (6)
                          [P][P][P]
                          [P][P][P]
Door (3)                 6 planks in v 3×2               Any planks (6)
                          [P][P]
                          [P][P]
                          [P][P]
Fence (3)                4 sticks + 2 planks             Sticks (4), Any planks (2)
                          [S][P][S]
                          [S][P][S]
Fence Gate               4 sticks + 2 planks             Sticks (4), Any planks (2)
                          [S][P][S]
                          [S][P][S]
                         (same shape as fence but outputs 1 gate)
Slab (6)                 3 blocks horizontal             Any slabs material (3)
                          [ ][ ][ ]
                          [ ][ ][ ]
                          [B][B][B]
Stairs (4)               6 blocks in L shape             Stairs material (6)
                          [B][ ][ ]
                          [B][B][ ]
                          [B][B][B]
Glass Pane (16)          6 glass h 2×3                  Glass blocks (6)
                          [G][G][G]
                          [G][G][G]
Iron Bars (16)           6 iron ingots h 2×3             Iron ingots (6)
Bookshelf                6 planks + 3 books              Any planks (6), Books (3)
                          [P][B][P]
                          [P][B][P]
                          [P][B][P]
Sandstone                4 sand (2×2)                    Sand (4)
Red Sandstone            4 red sand (2×2)                Red sand (4)
Bricks (block)           4 brick items (2×2)             Brick items (4)
Stone Bricks (4)         4 stone (2×2)                   Stone (4)
Iron Nugget (9)          1 iron ingot (shapeless)        Iron ingot (1)
Gold Nugget (9)          1 gold ingot (shapeless)        Gold ingot (1)
Glass Bottle (3)         3 glass (V)                     Glass (3)
                          [G][ ][ ]
                          [G][ ][ ]
                          [G][ ][ ]
Bowl (4)                 3 planks (V)                    Any planks (3)
                          [P][ ][ ]
                          [ ][P][ ]
                          [ ][P][ ]
Blaze Powder (2)         1 blaze rod (shapeless)         Blaze rod (1)
Magma Cream              1 slimeball + 1 blaze powder    Slimeball (1), Blaze powder (1)
                         (shapeless)
Glistering Melon         1 melon + 8 gold nuggets       Melon (1), Gold nuggets (8)
                         (shapeless)
Fermented Spider Eye     1 spider eye + 1 brown mush +   Spider eye (1), Brown mushroom (1),
                         1 sugar (shapeless)              Sugar (1)
Book                     3 paper + 1 leather             Paper (3), Leather (1)
                          [P][P][P]   (2×2 or shapeless)
                          [ ][L][ ]
Paper (3)                3 sugar cane (h)                Sugar cane (3)
                          [S][S][S]
Map (empty)              8 paper full, center empty      Paper (8)
                          [P][P][P]
                          [P][ ][P]
                          [P][P][P]
Clock                    4 gold ingots + 1 redstone      Gold ingots (4), Redstone (1)
                          [ ][G][ ]
                          [G][R][G]
                          [ ][G][ ]
Compass                  4 iron ingots + 1 redstone      Iron ingots (4), Redstone (1)
                          [ ][I][ ]
                          [I][R][I]
                          [ ][I][ ]
Lead                     4 string + 1 slimeball          String (4), Slimeball (1)
                          [S][S][S]
                          [S][L][S]
                          [ ][S][ ]
Fire Charge (3)          1 blaze powder + 1 coal +      Blaze powder (1), Coal/Charcoal (1),
                         1 gunpowder (shapeless)          Gunpowder (1)
Bucket                   3 iron ingots (V)               Iron ingots (3)
                          [I][ ][ ]
                          [I][ ][ ]
                          [I][ ][ ]
Water/Lava Bucket        Bucket + source block           Bucket (1), Water/Lava source (1)
                         (right-click on source)
Shield                   6 planks + 1 iron ingot         Any planks (6), Iron ingot (1)
                          [P][I][P]
                          [P][P][P]
                          [ ][P][ ]
```

### 9.2 Wood Variants ×6
```
Each wood type (Oak, Spruce, Birch, Jungle, Acacia, Dark Oak) has its own set of blocks.
Recipes are identical per type — only the plank type changes.

Per wood type:
  Item (output count)     Recipe                          Input
  ──────────────────────────────────────────────────────────────────────
  Planks (4)              1 log → 4 planks                Log of that type
  Slab (6)                3 planks horizontal             Planks of that type
  Stairs (4)              6 planks in L shape             Planks of that type
  Fence (3)               4 sticks + 2 planks             Planks of that type
  Fence Gate              4 sticks + 2 planks             Planks of that type
  Door (3)                6 planks (v 3×2)                Planks of that type
  Trapdoor (2)            6 planks (h 2×3)                Planks of that type
  Pressure Plate (1)      2 planks (h 1×2)                Planks of that type
  Button (1)              1 plank (placed on surface)     Plank of that type (1)
  Sign (3)                6 planks + 1 stick              Planks (6), Stick (1)
                           [P][P][P]
                           [P][P][P]
                           [ ][S][ ]
  Boat (1)                5 planks (U shape)              Planks of that type (5)
                           [P][ ][P]
                           [P][P][P]

Wood-to-plank conversion:
  1 log → 4 planks → 8 slabs or 4 stairs or 2 doors or 3 trapdoors

Cross-wood mixing:
  Sticks, chests, crafting tables, ladders, bowls, bookshelves:
    can use any planks mixed freely
  Doors, trapdoors, fences, fence gates, slabs, stairs, boats, signs, buttons:
    must use SAME plank type (recipe gives that wood's variant)
```

### 9.3 Tools & Weapons
```
Axe, Pickaxe, Shovel, Hoe — all follow same pattern by material tier:
  ┌─┬─┬─┐          ┌─┬─┬─┐      ┌─┬─┬─┐         ┌─┬─┬─┐
  │M│M│M│  Axe     │M│ │ │  Pick │M│ │ │  Shovel  │M│M│ │  Hoe
  │M│S│ │          │M│S│S│      │S│ │ │          │ │S│ │
  │ │S│ │          │ │S│ │      │S│ │ │          │ │S│ │
  └─┴─┴─┘          └─┴─┴─┘      └─┴─┴─┘         └─┴─┴─┘
  M = material (planks/cobble/iron/diamond/gold)
  S = stick

Sword:                        [M]
  [M]                         [M]     Other tools: see above
  [M]                         [S]
  [S]                         [S]

Material tiers (all tools/weapons):
  Wood (W):  planks (any)      Stone (S): cobblestone
  Iron (I):  iron ingot        Gold (G): gold ingot
  Diamond (D): diamond         (leather: not used for tools)

Bow:                          3 sticks (diag) + 3 string (vertical)
  [ ][S][ ]                    [S][ ][ ] or [ ][ ][S]
  [S][ ][ ]                    But in 3×3: [S][ ][ ]   [S][ ][ ]
  [ ][S][ ]                                  [ ][S][ ]  [S][ ][ ]
                                            Actually correct bow pattern:
  [S][ ][ ]  with strings
  [S][ ][ ]
  [S][ ][ ]  and sticks on the other side:
  [ ][S][ ]
  [ ][ ][S] — this works too
  Canonical: sticks in L pattern + string in L pattern (mirror)
  [ ][S][S]   [S][S][ ]  [S][ ][ ]  [ ][ ][S]
  [S][ ][ ]   [ ][ ][S]  [S][ ][ ]  [S][ ][ ]
  [ ][S][S]   [S][S][ ]  [ ][S][ ]  [ ][S][ ]
  Actually the simplest: [S][ ][ ]   with strings on the right vertical
                         [ ][S][ ]
                         [S][ ][ ]
  Let me just state: 3 sticks (L-shape or mirrored) + 3 string (L-shape mirrored opposite)
  The bow recipe requires sticks in one corner diagonal and string in the opposite diagonal.

Arrow (4):                    Flint (1), Stick (1), Feather (1)
  [F][ ][ ]
  [S][ ][ ]    or any vertical    Actually:
  [E][ ][ ]                       [F]
                                  [S]
                                  [E]  (vertical in any column)

Fishing Rod:                  3 sticks + 2 string
  [ ][ ][S]                    [ ][ ][S]
  [ ][S][ ]                    [ ][S][ ]
  [S][ ][ ]  with strings on   [S][E][ ]
  Actually: [ ][ ][S]
            [ ][S][E]
            [S][ ][E]
  Canonical: sticks in diagonal + string on top-right and mid-right

Flint & Steel:                1 iron ingot + 1 flint (shapeless)
Shears:                       2 iron ingots (diagonal)
  [ ][I]
  [I][ ]

Shield:                       6 planks + 1 iron ingot
  [P][I][P]
  [P][P][P]
  [ ][P][ ]
```

### 9.4 Armor
```
Helmet (1):                   Material ×5
  [M][M][M]
  [M][ ][M]

Chestplate (1):               Material ×8
  [M][ ][M]
  [M][M][M]
  [M][M][M]

Leggings (1):                 Material ×7
  [M][M][M]
  [M][ ][M]
  [M][ ][M]

Boots (1):                    Material ×4
  [M][ ][M]
  [M][ ][M]

Materials that can be used for armor:
  Leather (from leather, crafted into leather armor)
  Iron ingot (iron armor)
  Gold ingot (gold armor)
  Diamond (diamond armor)
  Fire: chainmail armor is NOT craftable (only from villager trading or mob drops)

Horse Armor: NOT craftable in 1.12.2 (dungeon/mansion loot only)
```

### 9.5 Food
```
Item (output)               Recipe
──────────────────────────────────────────────────────────────────────
Bowl (4)                    3 planks (V)                    See 9.1
Mushroom Stew (1)           1 bowl + 1 red mushroom +       Shapeless
                             1 brown mushroom
Beetroot Soup (1)           6 beetroot + 1 bowl              Shapeless
Rabbit Stew (1)             1 cooked rabbit + 1 carrot +     Shapeless
                             1 baked potato + 1 brown mush +
                             1 bowl
                             OR: raw rabbit (cooks in stew
                             process? Actually must use cooked)
Bread (1)                   3 wheat (h)
Cookie (8)                  2 wheat + 1 cocoa beans          Shapeless
Cake (1)                    3 milk buckets + 2 sugar +       Shaped
                             1 egg + 3 wheat
                             [M][M][M]
                             [S][E][S]
                             [W][W][W]
Pumpkin Pie (1)             1 pumpkin + 1 sugar + 1 egg      Shapeless
Golden Apple (1)            8 gold ingots + 1 apple          Shaped
                             [G][G][G]
                             [G][A][G]
                             [G][G][G]
Enchanted Golden Apple:     NOT craftable in 1.12.2
                             (loot only — dungeons, temples, etc.)
                             Recipe removed in 1.9

Food smelting (furnace):    See 9.14 Furnace Recipes
  Raw beef       → Steak
  Raw porkchop   → Cooked porkchop
  Raw chicken    → Cooked chicken
  Raw cod        → Cooked cod
  Raw salmon     → Cooked salmon
  Potato         → Baked potato
  Kelp           → Dried kelp (1.13+)
```

### 9.6 Redstone Components
```
Item (output)               Pattern                        Input
──────────────────────────────────────────────────────────────────────
Piston (1)                   4 cobble + 3 planks +         Cobble (4), Planks (3),
                              1 redstone + 1 iron ingot     Redstone (1), Iron ingot (1)
                             [C][C][C]
                             [I][R][I]  — wait, piston uses:
                             Actually: [P][P][P]  top row planks
                                       [C][I][C]  middle: cobble-iron-cobble
                                       [C][R][C]  bottom: cobble-redstone-cobble
                             Where I = iron, R = redstone, C = cobble, P = planks
Sticky Piston (1)           1 slimeball + 1 piston         Shapeless
Repeater (1)                2 redstone torches + 1         Stone (3), Redstone torches (2),
                             1 redstone + 3 stone           Redstone (1)
                             [R][R][R] ... wait:
Actually repeaters:          [S][R][S]  where S=stone, R=redstone torch? No.
  Repeater pattern:          [ ][T][ ]  T = redstone torch, R = redstone
                             Actually: [S][T][S]
                                       [ ][R][ ]
                                       [S][T][S]  Wait that's not right either.
  Correct repeater:          [ ][T][ ]  where each column: stone (S), redstone torch (T), stone (S)
                             [S][R][S]  middle row: stone, redstone, stone
                             [ ][T][ ]  bottom row: same as top
  But the actual 3×3 of a comparator uses 3 stone in a row... Let me just give the canonical.
  Repeater crafting:
    [S][T][S]    (3×3)       S=stone, T=redstone_torch, R=redstone
    [ ][R][ ]
    [S][T][S]
  This gives 1 repeater.

Comparator (1)              3 redstone torches + 1         Stone (3), Redstone torches (3),
                             1 nether quartz + 3 stone     Nether quartz (1)
                             [ ][T][ ]
                             [T][Q][T]
                             [S][S][S]
                             S=stone, T=redstone torch, Q=quartz

Dispenser (1)               7 cobblestone + 1 bow +       Cobble (7), Bow (1), Redstone (1)
                             1 redstone
                             [C][C][C]
                             [C][B][C]
                             [C][R][C]
                             B=bow, R=redstone

Dropper (1)                 7 cobblestone + 1 redstone     Cobble (7), Redstone (1)
                             Same as dispenser but without bow
                             [C][C][C]
                             [C][ ][C]
                             [C][R][C]

Hopper (1)                  5 iron ingots + 1 chest        Iron ingots (5), Chest (1)
                             [I][ ][I]
                             [I][C][I]
                             [ ][I][ ]

Observer (1)                6 cobblestone + 2 redstone +   Cobble (6), Redstone (2), Quartz (1)
                             1 nether quartz
                             [C][C][C]
                             [R][Q][R]
                             [C][C][C]

Redstone Lamp (1)           1 redstone + 4 glowstone       Redstone (1), Glowstone (4)
                             [ ][G][ ]
                             [G][R][G]
                             [ ][G][ ]

TNT (1)                     4 sand + 5 gunpowder           Sand (4), Gunpowder (5)
                             [G][S][G]
                             [S][G][S]
                             [G][S][G]

Daylight Sensor (1)         3 glass + 3 nether quartz +    Glass (3), Quartz (3), 
                             3 wood slabs (any)             Wood slabs (3)
                             [G][G][G]
                             [Q][Q][Q]
                             [S][S][S]

Lever (1)                   1 cobblestone + 1 stick        Shapeless
Button (Stone) (1)          1 stone
Pressure Plate (Stone) (1)  2 stone (h)
Pressure Plate (Wood) (1)   2 planks (h) (any wood)
Weighted Pressure Plate:
  Iron (1)                  2 iron ingots (h)
  Gold (1)                  2 gold ingots (h)

Tripwire Hook (2)           1 iron ingot + 1 stick +      Iron ingot (1), Stick (1), Planks (1)
                             1 plank (any)
                             [I]
                             [S]
                             [P]

Trapped Chest (1)           1 chest + 1 tripwire hook     Shapeless

Note Block (1)              8 planks (any) + 1 redstone    Planks (8), Redstone (1)
                             [P][P][P]
                             [P][R][P]
                             [P][P][P]

Jukebox (1)                 8 planks (any) + 1 diamond     Planks (8), Diamond (1)
                             [P][P][P]
                             [P][D][P]
                             [P][P][P]

Command Block:              NOT craftable (creative/command only)
Structure Block:            NOT craftable (creative/command only)
```

### 9.7 Transportation
```
Item (output)               Pattern                        Input
──────────────────────────────────────────────────────────────────────
Minecart (1)                5 iron ingots (U)               Iron ingots (5)
                             [I][ ][I]
                             [I][I][I]

Chest Minecart (1)          1 chest + 1 minecart           Shapeless
Furnace Minecart (1)        1 furnace + 1 minecart         Shapeless
TNT Minecart (1)            1 TNT + 1 minecart             Shapeless
Hopper Minecart (1)         1 hopper + 1 minecart          Shapeless

Boat (1) (×6 wood types)   5 planks (U)                    Planks of that type (5)
                             [P][ ][P]
                             [P][P][P]

Rail (16)                   6 iron ingots + 1 stick        Iron ingots (6), Stick (1)
                             [I][ ][I]
                             [I][S][I]
                             [I][ ][I]

Powered Rail (6)            6 gold ingots + 1 stick +      Gold ingots (6), Stick (1), Redstone (1)
                             1 redstone
                             [G][ ][G]
                             [G][S][G]
                             [G][R][G]

Detector Rail (6)           6 iron ingots + 1 stone        Iron ingots (6), Stone pressure plate (1),
                             pressure plate + 1 redstone   Redstone (1)
                             [I][ ][I]
                             [I][P][I]
                             [I][R][I]

Activator Rail (6)          6 iron ingots + 2 sticks +     Iron ingots (6), Sticks (2),
                             1 redstone torch              Redstone torch (1)
                             [I][T][I]
                             [I][S][I]
                             [I][S][I]
```

### 9.8 Decorative Blocks
```
Item (output)               Pattern                        Input
──────────────────────────────────────────────────────────────────────
Brick Stairs (4)            6 brick blocks (L)             Brick blocks (6)
Stone Brick Stairs (4)      6 stone bricks (L)             Stone bricks (6)
Sandstone Stairs (4)        6 sandstone blocks (L)         Sandstone (6)
Red Sandstone Stairs (4)    6 red sandstone (L)            Red sandstone (6)
Nether Brick Stairs (4)     6 nether bricks (L)            Nether bricks (6)
Quartz Stairs (4)           6 quartz blocks (L)            Quartz blocks (6)
Purpur Stairs (4)           6 purpur blocks (L)            Purpur blocks (6)

Cobblestone Wall (6)       4 cobblestone (square 2×2)      Cobblestone (4)
                             [C][C][ ]
                             [C][C][ ]
Mossy Cobblestone Wall (6) 4 mossy cobblestone (same)     Mossy cobblestone (4)

Stone Brick Wall (6)       Same as cobble wall            Stone bricks (4)
                             (1.12.2 has... actually stone brick wall is 1.16+)

Nether Brick Fence (6)     6 nether bricks + 2 nether     Nether bricks (6), Nether brick (2)
                             bricks... wait
                             [N][N][N]
                             [N][N][N]
  Actually nether brick fence: 6 nether brick blocks (h 2×3)

Fence Gate (×6 wood types) See 9.2

Quartz Block:               4 nether quartz (2×2)          Nether quartz (4)
  Chiseled Quartz:          2 quartz slabs (v)            Quartz slab (2)
                             actually: [Q][Q]
                                       [ ][ ] → chiseled quartz (1)
  Quartz Pillar:            2 quartz blocks (v)           Quartz blocks (2)

Purpur Block:               4 popped chorus fruit (2×2)   Popped chorus fruit (4)
  Purpur Pillar:            2 purpur blocks (v)           Purpur blocks (2)
  Purpur Stairs (4):        6 purpur blocks (L)           Purpur blocks (6)
  Purpur Slab (6):          3 purpur blocks (h)           Purpur blocks (3)

End Stone Bricks (4):      4 end stone (2×2)              End stone (4)

Prismarine (block):         4 prismarine shards (2×2)     Prismarine shards (4)
  Prismarine Bricks (1):    9 prismarine shards (full)    Prismarine shards (9)
  Dark Prismarine (1):      8 prismarine shards +          Prismarine shards (8), Ink sac (1)
                             1 ink sac (shapeless)

Sea Lantern:                4 prismarine shards +          Prismarine shards (4),
                             5 prismarine crystals         Prismarine crystals (5)
                             [S][C][S]
                             [C][S][C]
                             [S][C][S]

Chiseled Stone Bricks (1): 2 stone brick slabs (v)        Stone brick slabs (2)

Mossy Stone Bricks (1):    1 stone bricks + 1 vine        Shapeless

Cracked Stone Bricks (1):  1 stone bricks (furnace)       Stone bricks (1) — actually
                            Furnace: 1 stone brick → 1 cracked stone brick
                            Wait — cracked stone brick is NOT furnace-crafted in survival.
                            It's only naturally generated in strongholds/ruins.
                            So ignore this recipe.

Enchanting Table:           4 obsidian + 2 diamonds +      Obsidian (4), Diamond (2),
                             1 book                        Book (1)
                             [ ][B][ ]
                             [D][ ][D]
                             [O][O][O]

Anvil:                      3 iron blocks + 4 iron         Iron blocks (3), Iron ingots (4)
                             ingots
                             [B][B][B]
                             [ ][I][ ]
                             [I][I][I]
                             B=iron block, I=iron ingot

Flower Pot:                 3 bricks (V)                   Brick items (3)
                             [B][ ][ ]
                             [B][ ][ ]
                             [B][ ][ ]

Item Frame:                 8 sticks + 1 leather           Sticks (8), Leather (1)
                             [S][S][S]
                             [S][L][S]
                             [S][S][S]

Painting:                   8 sticks + 1 wool (any color)  Sticks (8), Wool (1)
                             [S][S][S]
                             [S][W][S]
                             [S][S][S]

Armor Stand:                6 sticks + 1 stone slab        Sticks (6), Stone slab (1)
                             [S][S][S]
                             [ ][S][ ]
                             [ ][S][ ]

End Crystal:                1 ghast tear + 1 eye of       Ghast tear (1), Eye of ender (1),
                             ender + 7 glass blocks         Glass (7)
                             [G][G][G]
                             [G][E][G]
                             [G][T][G]
                             E=eye of ender, T=tear

Beacon:                     5 glass blocks + 3 obsidian +  Glass (5), Obsidian (3), Nether star (1)
                             1 nether star
                             [G][G][G]
                             [G][N][G]
                             [O][O][O]

Conduit (1.13+): skip

Moss Stone:                 1 cobblestone + 1 vine         Shapeless

Crafting Table:             See 9.1
```

### 9.9 Dyes & Coloring (16 Colors)
```
Base dyes (obtained from natural sources):
  White Dye (1):            Bone meal (1) from bone (shapeless)
  Red Dye (1):              Poppy (1), or Rose bush (1), or Beetroot (1)
  Orange Dye (1):           Orange tulip (1)
  Yellow Dye (1):           Dandelion (1), or Sunflower (1)
  Green Dye (1):            Cactus (furnace, 0.2xp)
  Lime Dye (1):             1 green dye + 1 white dye (shapeless)
  Light Blue Dye (1):       1 blue dye + 1 white dye, or blue orchid (1)
  Cyan Dye (1):             1 lapis + 1 green dye (shapeless)
  Light Gray Dye (1):       1 gray + 1 white, or 2 white + 1 black (shapeless) 
                             Actually: 1 azure bluet (flower) = Light Gray, or
                                       2 white dye + 1 ink sac (shapeless)
  Gray Dye (1):             1 ink sac + 1 white dye (shapeless), or
                             1 black + 2 white (also gives 2)
                             Actually gray: 1 ink sac + 1 bone meal (shapeless)
  Pink Dye (1):             1 red + 1 white, or pink tulip (1), or peony (1)
  Magenta Dye (2):          1 lilac (flower) = 2 magenta, or
                             1 purple + 1 pink, or
                             1 blue + 1 red + 2 white + 1 red (various combos)
                             Simplest: 1 purple + 1 pink (shapeless)
  Purple Dye (1):           1 lapis + 1 red dye (shapeless)
  Blue Dye (1):             Lapis lazuli (1), or cornflower (1)
  Brown Dye (1):            Cocoa beans (1)
  Black Dye (1):            Ink sac (1), or wither rose (1)

Dye mixing (shapeless):
  All intermediate dyes can be mixed per the formulas above.
  Maximum 8 dyes can be crafted at once (8 flowers → 8 dye).

Wool coloring:
  1 wool (white) + 1 dye → 1 colored wool (shapeless)
  8 white wool around dye → 8 colored wool (shaped)

Carpet coloring:
  Carpet can't be dyed directly; must craft from colored wool.
  Carpet (3):               2 colored wool (h 1×2)        Colored wool (2)
                             [W][W]

Stained glass coloring:
  Glass cannot be re-dyed; must craft from colored glass or dye directly.
  8 glass around dye → 8 stained glass (shaped)
  8 glass panes around dye → 8 stained glass panes (shaped)

Terracotta:
  Hardened clay: smelted from clay block (furnace)
  Stained terracotta: 8 hardened clay around dye → 8 stained terracotta (shaped)
  Re-dyeing: not possible for glazed terracotta — must re-craft.

Concrete Powder:
  4 sand + 4 gravel + 1 dye → 8 concrete powder (shapeless or shaped)
  [S][S][S]  ... actually shaped: [S][S][D]  for 8... wait
  Concrete Powder (8):      4 sand + 4 gravel + 1 dye     Sand (4), Gravel (4), Dye (1)
                             [S][S][G][G]
                             [S][S][G][G]
                             [D][ ][ ][ ]
  In 2×2: 2 sand + 2 gravel = 4 concrete powder. But the 3×3 recipe:
  [S][S][G][G]  top row: 2 sand, 2 gravel (4 wide) — no, crafting grid is 3×3 max.
  For 3×3: [S][S][G]  +  [S][G][G]  +  [D][ ][ ] → gives 8 concrete powder
           Actually 8 concrete powder uses 4 sand, 4 gravel, 1 dye in 3×3:
  [S][S][G]
  [S][G][G]
  [D][ ][ ]
  This gives... 5 sand + 0 gravel = not right.
  The actual recipe: sand at positions 1,2,4,5; gravel at 3,6,7,8; dye at position 9
  [S][S][G]
  [S][S][G]  → gives 8 concrete powder of the dye's color
  [G][G][D]
  OR: shapeless: 4 sand + 4 gravel + 1 dye → 8 concrete powder

Bed (all 16 colors):
  Bed color depends on the wool used. White bed requires white wool; other colors use matching wool.
  Bed (1):                  3 wool (same color) + 3 planks (h 1×3)  Any planks (3)
                             [ ][ ][ ]
                             [W][W][W]  → actually:
                             [P][P][P]
  The pattern: 3 wool on top row, 3 planks on bottom row.
  Output: bed of the wool's color.
  A bed cannot be re-dyed; crafting with dyed wool gives that color bed.

Shulker Box (all 16 colors):
  1 shulker shell + 1 chest → 1 unbounded shulker box
  Actually: 2 shulker shells + 1 chest = 1 shulker box (undyed, purple)
  [S][ ][S]  or shapeless.
  [ ][C][ ]
  Dyeing: 1 shulker box + 1 dye = 1 dyed shulker box (shapeless)

Banner (all 16 colors):
  6 wool + 1 stick (vertical) = 1 banner of that color
  [W][W][W]
  [W][W][W]
  [ ][S][ ]
  Banner cannot be re-dyed. Patterns can be layered with dyes.
```

### 9.10 Fireworks
```
Firework Star (1):
  Base: 1 gunpowder + 1 dye → star of that color
  Shape modifiers (optional, shapeless):
    + Fire charge      → Star-shaped (large)
    + Feather          → Burst (spherical)
    + Gold nugget      → Star (5-point)
    + Firework star    → Twinkle (fade to white)
    + Diamond          → Trail (glimmer trail)
    + Any dye          → Fade color (fade to this color)
  Multiple modifiers can be combined (max: 1 shape + 1 twinkle + 1 trail + multiple fade colors)
  Maximum 8 ingredients in total + gunpowder + dye + modifiers.

Firework Rocket (1-3):
  1 paper + 1-3 gunpowder + optional firework star
  No star → just a burst of noise (no color)
  With star → rocket launches and explodes into the star
  More gunpowder → higher flight duration:
    1 gunpowder: 1 second flight
    2 gunpowder: 2 seconds flight
    3 gunpowder: 3 seconds flight
  1-3 gunpowder + 0-7 firework stars + 1 paper = rocket with those stars

Firework flight in elytra:
  Rocket provides boost:
    1 gunpowder: 1 second boost (7 blocks)
    2 gunpowder: 1.5 second boost (11 blocks)
    3 gunpowder: 1.75 second boost (13 blocks)
  Rocket with firework star: boost + explosion damages player if too close
```

### 9.11 Banner Patterns
```
Banner patterns are crafted by combining a banner with additional materials:
  All recipes require 1 banner (any color) + ingredient → 1 banner with pattern applied.

Pattern                   Ingredient(s)                      Pattern name
──────────────────────────────────────────────────────────────────────────
Bordered                  Vine                               border
Field (masoned)           1 brick block + 1 banner            field_masoned
Base (indented)           Cobblestone wall                   field_masoned? No...
Actually let me list all:

  Base:       1 banner + 1 cobblestone wall (any)         → pattern.base
  Border:     1 banner + 1 vine                            → pattern.border
  Bricks:     1 banner + 1 brick block                     → pattern.bricks
  Creeper:    1 banner + 1 creeper head                    → pattern.creeper
  Cross:      1 banner + 1 cobblestone                     → pattern.cross
  Curly:      1 banner + 1 vine + 1 brick block? No...
  
Actually banner patterns in 1.12.2 use specific items:
  - Enchanted golden apple (creeper charge): NOT craftable
  - Wither skeleton skull: skull charge
  - Oxeye daisy: flower charge
  - Cobblestone wall: field masoned
  - Brick block: bordure indented? Actually

Let me just list the accurate 1.12.2 patterns:
  1. Base (bottom third):      1 banner + 1 cobblestone wall (any)   → "things" category? No
  2. Border:                   1 banner + 1 vine                      → border
  3. Bricks:                   1 banner + 1 brick block               → field_masoned
  4. Creeper:                  1 banner + 1 creeper head              → creeper
  5. Cross:                    1 banner + 1 cobblestone               → saltire? Actually cross
  6. Curly:                    1 banner + 1 vine + 1 brick block? ... no.
  7. Diagonal Left:            1 banner + 1 cobblestone?              No.

I'll simplify to the accurate list from 1.12.2:
  Pattern                       Ingredient                      Result name
  ──────────────────────────────────────────────────────────────────────────
  Thing / Base (bottom third)  1 banner + 1 cobblestone wall   pattern.thing
  Border                       1 banner + 1 vine               pattern.border  
  Bricks                       1 banner + 1 brick block         pattern.bricks
  Creeper Charge               1 banner + 1 creeper head        pattern.creeper
  Cross                        1 banner + 1 cobblestone         pattern.cross
  Curly Border                 1 banner + 1 vine + 1 brick      bordure_indented?
                                (that's 2 ingredients)          pattern.curly_border
  Diagonal Left                1 banner + 1 cobblestone         pattern.diagonal_left? No
  Diagonal Right               1 banner + 1 cobblestone         pattern.diagonal_right? No
  Diagonal Up Left             1 banner + 1 cobblestone         ... 
  
This is getting messy. Let me just state:
  Banner patterns in 1.12.2 allow applying 5 basic patterns by default +
  special patterns unlocked by specific items. The key special patterns:
    - Creeper charge (creeper head): rare
    - Skull charge (wither skeleton skull): rare
    - Flower charge (oxeye daisy): common
    - Globe (1.14+): skip
    - Thing (cobblestone wall): common
    - Bordure/border (vine): common
    - Field Masoned (brick block): common

  Maximum number of patterns per banner: 6 layers
  Each additional pattern costs 1 more dye of that pattern color.
  Patterns stack: first pattern layer is always the "base color" from the banner itself.
```

### 9.12 Map Crafting
```
Empty Map (1):              8 paper (full, center empty)   Paper (8)
                             [P][P][P]
                             [P][ ][P]
                             [P][P][P]

Empty Locator Map (1):      1 compass + 8 paper            Compass (1), Paper (8)
                             [P][P][P]
                             [P][C][P]
                             [P][P][P]
  Note: in 1.12.2, the simple empty map and locator map are both craftable.
  The locator map shows player position markers.

Map cloning:                Two maps of same scale +       Map (1), Empty map (1)
                             1 empty map (shapeless)
                             Or the "clone" cartography table recipe (1.14+).
                             In 1.12.2: 2 filled maps of same scale → 2... actually
                             Map cloning in 1.12.2: craft map + empty map = 2 maps (both filled)
                             This uses the crafting grid:
                             [M][E]  → M + E = M + M (both are the filled map)
                             2 filled maps can be crafted with 2 empty maps = 4 copies total

Map zooming out:            1 base map + 8 paper           Map (1), Paper (8)
                             [P][P][P]
                             [P][M][P]
                             [P][P][P]
                             Results in map at 2× zoom (larger area)
                             Can zoom up to 4 levels (0=original, 1=2×, 2=4×, 3=8×)
                             Each zoom level: 8 paper

Cartography Table:          1.14+, skip
```

### 9.13 Enchanting & Potion Ingredients
```
Enchanting Table:           See 9.8 Decorative Blocks
Brewing Stand:              1 blaze rod + 3 cobblestone   Blaze rod (1), Cobblestone (3)
                             [ ][B][ ]
                             [C][C][C]
Cauldron:                   7 iron ingots (U shape)        Iron ingots (7)
                             [I][ ][I]
                             [I][ ][I]
                             [I][I][I]

Eye of Ender:               1 ender pearl + 1 blaze       Ender pearl (1), Blaze powder (1)
                             powder (shapeless)

Golden Carrot:              8 gold nuggets + 1 carrot     Gold nuggets (8), Carrot (1)
                             (shapeless)

Bottle o' Enchanting:       NOT craftable in survival
Splash Potion:              Potion + gunpowder (brewing stand)
Lingering Potion:           1.13+, skip
Tipped Arrow:               1.13+, skip... WAIT — tipped arrows ARE in 1.12.2.
                             Added in snapshot 15w33a (1.9). 
                             Tipped Arrow (8): 8 arrows + 1 lingering potion... no wait,
                             In 1.12.2, tipped arrows are crafted with:
                             8 arrows + 1 lingering potion → 8 tipped arrows (with that effect)
                             But lingering potions were added in 1.9, so they exist.
                             HOWEVER lingering potions require dragon's breath, which requires
                             the End, and collecting dragon breath with glass bottles.
                             [A][A][A]  A=Arrow, P=Lingering potion
                             [A][P][A]
                             [A][A][A]
                             → 8 tipped arrows with the potion's effect
                             Duration for tipped arrows: 1/8 of potion duration
                             Splash potion: 1/8 of potion? Actually tipped arrow uses:
                             effect = lingering_potion_effect / 4 = potion_effect / 8
                             In 1.12.2, tipped arrows exist and work.
```

### 9.14 Furnace Recipes (Complete)
```
Input                → Output                XP
──────────────────────────────────────────────────────────
Raw Iron             → Iron Ingot           0.7
Raw Gold             → Gold Ingot           1.0
Diamond Ore          → Diamond              — (NOT smeltable, drops directly)
Emerald Ore          → Emerald              — (NOT smeltable)
Cobblestone          → Stone                0.1
Stone                → Smooth Stone         0.1 (for stone slabs)
Stone Bricks         → Cracked Stone Bricks — (NOT in survival furnace — generated only)
Sand                 → Glass                0.1
Red Sand             → Glass                0.1
Clay Ball            → Brick Item           0.3
Clay Block           → Hardened Clay (Terracotta)  0.35
Cactus               → Green Dye           0.2
Netherrack           → Nether Brick         0.1
Log (any)            → Charcoal             0.15
Wet Sponge           → Sponge               0.15
Chorus Fruit         → Popped Chorus Fruit  0.1
Sandstone            → Smooth Sandstone     0.1
Red Sandstone        → Smooth Red Sandstone 0.1
Quartz Ore? No, quartz ore drops quartz directly.
Nether Quartz Ore    → Nether Quartz        — (drops directly, not smelted)

Stained clay/Hardened clay is NOT re-smeltable for glazed terracotta.
Glazed terracotta: smelt colored terracotta → glazed terracotta of that color (0.1xp)

RAW MEATS (0.35 XP each):
  Raw Beef           → Steak
  Raw Porkchop       → Cooked Porkchop
  Raw Chicken        → Cooked Chicken
  Raw Cod            → Cooked Cod
  Raw Salmon         → Cooked Salmon
  Raw Mutton         → Cooked Mutton
  Raw Rabbit         → Cooked Rabbit

OTHER FOOD:
  Kelp               → Dried Kelp (1.13+)
  Potato             → Baked Potato (0.35xp)

TOOL SMELTING (iron/gold tools return nuggets):
  Iron Pickaxe/Sword/etc  → Iron Nugget(s)
  Gold Pickaxe/Sword/etc  → Gold Nugget(s)
  Amount: 1 nugget per item (not per ingot)
  Chainmail armor smelted to iron nuggets (but chainmail isn't craftable anyway)
  Iron armor → iron nuggets (1 per item? Actually 1 nugget per piece)

  Actually in 1.12.2, tool/armor smelting does NOT exist.
  Wait — it WAS added in 1.13 (village & pillage update).
  In 1.12.2: tools/armor cannot be smelted back into nuggets.
  Cross out tool smelting for 1.12.2.
```

### 9.15 Fuel Values
```
Fuel item                 Items smelted  Total time
────────────────────────────────────────────────────────
Lava Bucket               100            1000s (16000 ticks)
Coal Block                80             800s (12800 ticks)
Dried Kelp Block          20 (1.13+)
Blaze Rod                 12             120s (1920 ticks)
Coal/Charcoal             8              80s (1280 ticks)  
Boat / Chest              6              60s (960 ticks)
Log / Wood / Stripped     1.5            15s (240 ticks)
Plank / Fence / Gate      1.5            15s (240 ticks)
Mushroom Block (big)      1.5            15s (240 ticks)
Bow / Fishing Rod         1.5            15s (240 ticks)
Wood Stairs               1.5            15s (240 ticks)
Wood Slab                 0.75           7.5s (120 ticks) — but slabs stack to 6
Wood Tool / Sword         1              10s (160 ticks)
Stick / Sapling           0.5            5s (80 ticks)
Bowl                      0.5            5s (80 ticks)
Wood Button               0.5? Shapeless... button is 1 plank = 1.5 items
                           
Cactus: NOT a fuel (0 items smelted)
Bamboo: 1.13+, skip
Lava bucket: doesn't consume bucket (returns empty bucket)

A furnace can smelt one item at a time. Fuel consumption:
  Each operation takes 200 ticks (10 seconds)
  Fuel per operation: 200 ticks / fuel_total_ticks item(s) consumed
  
  Example: Coal (80 ticks) smelts 8 items: 80/200 = 0.4 items per tick? No.
  Actually: 1 coal provides 1600 ticks of burn time... wait.
  Let me recalculate: one coal (200 ticks burn time)... ?
  Wait: I said Coal/Charcoal = 80 ticks. But each item takes 200 ticks to smelt.
  That can't be right. Let me correct:
  
Corrected fuel burn times:
  Lava bucket:     20,000 ticks (1000s) = 100 items  (correct = ×10 my earlier number)
  Coal block:      16,000 ticks (800s) = 80 items
  Blaze rod:       2,400 ticks (120s) = 12 items
  Coal/Charcoal:   1,600 ticks (80s) = 8 items
  Boat:            1,200 ticks (60s) = 6 items
  Chest:           1,200 ticks (60s) = 6 items
  Log/Wood:         300 ticks (15s) = 1.5 items
  Plank/etc:        300 ticks (15s) = 1.5 items
  Stick/Sapling:    100 ticks (5s) = 0.5 items
  Wood tool:        200 ticks (10s) = 1 item
  Wood slab:        150 ticks (7.5s) = 0.75 items
  
  Each furnace slot smelts 1 item per 200 ticks (10 seconds) regardless of what it is.
  So 1 coal (1600 ticks) ÷ 200 ticks/item = 8 items smelted. ✓
```

### 9.16 Brewing (Complete)
```
Base step:
  Water Bottle + Nether Wart → Awkward Potion (3:00)
  Water Bottle + Glistering Melon → Mundane Potion (3:00)  [no effect, skip]
  Water Bottle + Sugar → Thick Potion (3:00)  [no effect, skip]
  Water Bottle + Redstone → Mundane Potion extended (8:00)  [no effect, skip]
  Water Bottle + Glowstone → Thick Potion (no duration)  [no effect, skip]
  Only Awkward Potion is useful for further brewing.

Effect potions from Awkward:
  Awkward + Golden Carrot  → Potion of Night Vision (3:00)
  Awkward + Blaze Powder   → Potion of Strength (3:00)
  Awkward + Ghast Tear     → Potion of Regeneration (0:45)
  Awkward + Magma Cream    → Potion of Fire Resistance (3:00)
  Awkward + Sugar          → Potion of Swiftness (3:00)
  Awkward + Rabbit Foot    → Potion of Leaping (3:00)
  Awkward + Spider Eye     → Potion of Poison (0:45)
  Awkward + Pufferfish     → Potion of Water Breathing (3:00)

Secondary potions:
  Night Vision + Fermented Spider Eye → Potion of Invisibility (3:00)
  Swiftness + Fermented Spider Eye   → Potion of Slowness (1:30)
  Leaping + Fermented Spider Eye     → Potion of Slowness (1:30)
  Poison + Fermented Spider Eye      → Potion of Harming (instant)
  Water Breathing + Fermented Eye?   → Potion of Harming? No...
  Actually: Fermented Eye inverts positive effects:
    Night Vision → Invisibility
    Strength → Weakness
    Regeneration → Harming
    Poison → Harming
    Water Breathing → Harming? No... Water Breathing has no inversion.
    Fire Resistance → Slowness
    Swiftness → Slowness
    Leaping → Slowness
    Health (instant health) → Harming (instant damage)
  So any positive potion + fermented eye = its negative counterpart (or harming/slowness/weakness)

  Weakness: Water Bottle + Fermented Spider Eye (3:00)
  Weakness extended: Water Bottle + Fermented Eye + Redstone (8:00)
  Extended duration default for Weakness: no glowstone variant for weakness.

Modifier steps (applied AFTER base potion, can stack):
  + Redstone     → Extends duration to 8:00 (×1.67 for most, ×2.67 for some)
  + Glowstone    → Amplifies to level II, reduces duration to 1/3
  + Gunpowder    → Splash potion (area effect, multiplied by 0.75)
  
  Order: base → (redstone/glowstone) → (gunpowder) → (fermented eye)
  Redstone and glowstone are mutually exclusive (can't have both)

Extended vs Amplified duration:
  Base potion   → +Redstone (ext)    → +Glowstone (II)
  ─────────────────────────────────────────────────────────
  Night Vision  3:00   8:00          1:30 II
  Strength      3:00   8:00          1:30 II
  Regeneration  0:45   1:30          0:22 II
  Fire Res      3:00   8:00          — (no II variant)
  Swiftness     3:00   8:00          1:30 II
  Leaping       3:00   8:00          1:30 II
  Poison        0:45   1:30          0:22 II
  Water Br      3:00   8:00          — (no II variant)
  Slowness      1:30   4:00          — (no II variant, IV exists from stray)
  Weakness      1:30   4:00          — (no II variant)
  Harming       instant —            Harming II = 6 HP (from glowstone)
  Healing       instant —            Healing II = 8 HP (from glowstone)
  Invisibility  3:00   8:00          — (no II variant)
```

### 9.17 Brewing Ingredients & Substitutions
```
Nether Wart        → Awkward base (required for most potions)
Redstone           → Extended duration (1.5× base)
Glowstone Dust     → Amplify to II (1/3 duration, stronger effect)
Gunpowder          → Splash potion (area of effect)
Fermented Eye      → Invert effect (positive → negative)
Dragon's Breath    → Lingering potion (1.13+, skip)
Sugar              → Swiftness
Blaze Powder       → Strength (also fuel for brewing stand)
Ghast Tear         → Regeneration
Magma Cream        → Fire Resistance
Glistering Melon   → Instant Health
Spider Eye         → Poison
Pufferfish         → Water Breathing
Golden Carrot      → Night Vision
Rabbit's Foot      → Jump Boost
Fermented Eye      → Weakness (base: water bottle + fermented eye)
              
Brewing stand fuel: Blaze powder
  Each piece of blaze powder fuels 20 operations
  No fuel cost if infinite fuel enabled (creative)

Brewing time:
  Each step: 20 seconds (400 ticks)
  Up to 3 bottles per operation (3 water bottles + ingredient = 3 potions)
```

---

## P9B — 1.12 Exclusive Systems (World of Color Update)

### 9B.1 Recipe Book System
```
The Recipe Book is a 1.12 new feature for discovering, viewing, and crafting recipes.

Unlocking recipes:
  - Recipes are unlocked when player picks up required ingredients
    (e.g., picking up planks unlocks all plank-based recipes)
  - Some recipes are unlocked by obtaining the output item
    (e.g., obtaining a crafting table by crafting one)
  - Achievements no longer unlock recipes (advancements replaced them in 1.12)
  - Recipe unlock is per-player, saved to player NBT
  - Default state: all recipes unlocked in Creative mode
  - /gamerule doLimitedCrafting false (default): all recipes available
    doLimitedCrafting true: only unlocked recipes appear in book
  - /recipe <give|take> <player> <namespace:path|*>
    give * : unlocks all recipes
    take * : locks all recipes (re-lock)
    give namespace:path : unlock specific recipe
  - Recipe unlock event: toast notification appears
    "New Recipes Unlocked!" toast in top-right of screen
    Toast: icon of the first unlocked item + recipe count
    Toast duration: ~3 seconds, slides in/out
  - Recipes are identified by namespace:path (e.g., minecraft:stick)
  - Recipe categories: crafting, smelting, blasting, smoking, campfire_cooking,
    stonecutting, smithing (only crafting in 1.12; others added in later versions)

Recipe book GUI:
  Access: click the green book icon in the crafting table GUI (or press 'R' key)
    Also accessible from player inventory crafting grid (2×2)
  GUI layout:
    - Left sidebar: 6 recipe category tabs
      1. All recipes (star icon)
      2. Tools & Weapons (sword icon)
      3. Building Blocks (brick icon)
      4. Food & Drinks (apple icon)
      5. Redstone & Transport (redstone dust icon)
      6. Miscellaneous (chest icon)
    - Main area: scrollable grid of recipe icons
      Each recipe shown as a small icon of the output item
      Grid: 7 columns wide, height variable (scrollable)
      Items sorted alphabetically within each category
      Locked recipes shown as dark silhouette (grayed out, no tooltip)
    - Search bar: at top of recipe list
      Searches by item name (localized, matched against display names)
      Filters visible recipes to matching results
      Search box: text input, responds to keyboard input
      Clear button: (X) to reset search
    - Recipe detail: click a recipe to view it
      Shows the 3×3 crafting grid with ingredient layout
      Ingredients shown as item icons with quantity labels
      Output shown as large item icon on the right
      Click the output or the recipe to craft (if ingredients in inventory)
      Shift-click: crafts as many as possible

Crafting interaction:
  - Clicking a recipe book entry: fills the crafting grid with the correct pattern
    If player has materials: grid fills automatically
    If materials missing: cannot craft (ingredient slots remain empty)
    Clicking again after materials depleted: clears grid
  - Quick crafting: click recipe output to craft one
    Shift-click recipe output: craft maximum amount (limited by materials)
  - Recipe book respects shapeless vs shaped recipes
    Shapeless: ingredients placed in any order in grid
    Shaped: exact pattern required, book fills grid correctly
  - Recipe book remembers last-selected recipe (persists while GUI is open)
  - Recipe search: filters by item name or hover text
    Searches recipe outputs, not ingredients
    Searches all categories simultaneously when text entered
    Searching while a category is selected: searches within that category

Recipe book data structure:
  Recipe key: string = "namespace:path"
  Recipe data:
    - output: ItemStack (item + count)
    - ingredients: Ingredient[] (each Ingredient = ItemStack[] of alternatives)
    - pattern: string[] (rows) for shaped, null for shapeless
    - group: string (recipe group for grouping similar recipes in book)
    - category: enum (building, redstone, food, tools, misc, all)
    - width: number (shaped recipe width in grid)
    - height: number (shaped recipe height in grid)
  Recipe categorization:
    building: blocks, slabs, stairs, fences, doors, etc.
    redstone: redstone-related items, pistons, rails, etc.
    food: edible items, cakes, cookies, etc.
    tools: tools, weapons, armor, etc.
    misc: dyes, banners, fireworks, etc.
  Custom recipes (data-driven, loaded from JSON):
    Recipes loaded from data/<namespace>/recipes/<path>.json
    Format: { "type": "minecraft:crafting_shaped"|"shapeless", ... }
    Can override vanilla recipes (use same namespace:path)
    Custom recipes appear in recipe book if unlocked

Saved Toolbars (1.12 NEW):
  - Creative mode only: save/load hotbar presets
  - Save: press C + number key (1-9) to save current hotbar to slot
  - Load: press X + number key (1-9) to load hotbar from slot
  - Saved toolbars persist per-player, stored in player NBT
  - Visual: tooltip shows "Saved toolbar <n>" on load/save
  - Inventory GUI: saved toolbar slots shown at bottom of creative inventory
    9 slots (1-9), click to load, right-click to overwrite
  - Up to 9 saved toolbars (C+1 through C+9)
```

### 9B.2 Creative Inventory — Full 12-Tab Layout
```
Creative mode inventory is organized into 12 searchable tabs.
Each tab is ordered precisely as in vanilla 1.12.2.

Tab 1 — Building Blocks:
  Stone, Granite, Polished Granite, Diorite, Polished Diorite, Andesite, Polished Andesite,
  Grass Block, Dirt, Coarse Dirt, Podzol, Mycelium, Stone Bricks (all variants), Mossy Stone Bricks,
  Cracked Stone Bricks, Chiseled Stone Bricks, Cobblestone, Mossy Cobblestone,
  Wood Planks (Oak, Spruce, Birch, Jungle, Acacia, Dark Oak),
  Saplings (6 types), Dead Bush, Leaves (6 types), Vines,
  Sponge, Wet Sponge, Sand, Red Sand, Sandstone (all variants), Red Sandstone (all variants),
  Gravel, Clay, Block of Gold, Block of Iron, Block of Diamond, Block of Emerald,
  Block of Lapis Lazuli, Block of Coal, Block of Redstone, Block of Quartz,
  Bookshelf, Obsidian, Netherrack, Nether Brick, Nether Quartz Ore,
  Soul Sand, Glowstone, End Stone, End Stone Bricks, Purpur Block (all variants),
  Bone Block, Coal Ore, Iron Ore, Gold Ore, Diamond Ore, Emerald Ore, Lapis Ore, Redstone Ore,
  Prismarine, Prismarine Bricks, Dark Prismarine, Sea Lantern,
  Ice, Packed Ice, Frosted Ice, Slime Block, Hay Bale, Smooth Sandstone, Smooth Red Sandstone,
  Melon Block, Pumpkin, Carved Pumpkin, Jack o'Lantern, Crafting Table, Furnace,
  Dispenser, Dropper, Observer, Hopper, Piston, Sticky Piston,
  Note Block, Jukebox, Enchanting Table, Ender Chest, Anvil (3 damage states),
  End Portal Frame, Dragon Egg, Beacon, Spawner, End Rod, Chorus Plant, Chorus Flower,
  Brewing Stand, Cauldron, Flower Pot, Cobweb, Tripwire Hook, Tripwire,
  Lever, Button (Stone + Wood), Redstone Torch, Redstone Repeater, Redstone Comparator,
  TNT, Iron Bars, Glass Pane, Stained Glass Pane (16 colors),
  Glass, Stained Glass (16 colors), Sea Pickle — wait, no; no sea pickles in 1.12.

Tab 2 — Decorative Blocks:
  Flowering Azalea? No. 1.12 decorative: All flowers, all saplings, all plants.
  Dandelion, Poppy, Blue Orchid, Allium, Azure Bluet, Red Tulip, Orange Tulip,
  White Tulip, Pink Tulip, Oxeye Daisy, Sunflower, Lilac, Tall Grass, Large Fern,
  Rose Bush, Peony, Tall Seagrass? No seagrass in 1.12.
  Oak/Wooden slabs (6 types × upper/lower), Stone variants slabs, Sandstone slabs,
  Cobblestone slabs, Brick slabs, Stone Brick slabs, Nether Brick slabs, Quartz slabs,
  Oak/Wooden stairs (6 types), Stone variants stairs, Sandstone stairs, Cobblestone stairs,
  Brick stairs, Stone Brick stairs, Nether Brick stairs, Quartz stairs, Purpur stairs,
  Oak/Wooden fences (6 types), Oak/Wooden gates (6 types),
  Cobblestone Wall, Mossy Cobblestone Wall,
  Oak/Wooden doors (6 types), Oak/Wooden trapdoors (6 types),
  Oak/Wooden pressure plates (6 types), Stone pressure plate, Weighted Pressure Plates (Light + Heavy),
  Oak/Wooden buttons (6 types), Stone button,
  Signs (Oak + 5 others, but in 1.12 all are "Sign" item), Item Frame, Painting,
  Banners (16 base colors + patterns), Bed (16 colors),
  Carpet (16 colors), Wool (16 colors),
  Hardened Clay (Terracotta, 16 colors), Glazed Terracotta (16 colors),
  Concrete Powder (16 colors), Concrete (16 colors),
  Shulker Box (undyed + 16 colors),
  Snow, Snow Layer, Ice, Packed Ice, Frosted Ice,
  Ladders, Scaffolding? No, scaffolding is 1.14. Instead: Ladders, Vines, Lily Pad,
  Chest, Trapped Chest, Ender Chest,
  End Rod, Chorus Plant, Chorus Flower,
  Brewing Stand, Cauldron, Flower Pot,
  Skull (Skeleton, Wither Skeleton, Zombie, Player, Creeper, Dragon Head),
  Cobweb, Sponge, Wet Sponge, Spawner, Dragon Egg, End Portal Frame,
  Anvil (3 damage states), Grindstone? No, added later. Enchanting Table, Bookshelf,
  Beacon, Note Block, Jukebox.

Tab 3 — Redstone:
  Dispenser, Dropper, Observer, Hopper, Piston, Sticky Piston,
  Redstone Block, Redstone Dust, Redstone Torch, Redstone Repeater, Redstone Comparator,
  Lever, Button (Wood + Stone), Pressure Plates (Wood, Stone, Light, Heavy),
  Tripwire Hook, String,
  Trapdoor (Wood + Iron), Door (Wood + Iron), Fence Gate,
  Chest, Trapped Chest, Ender Chest,
  Furnace, Dispenser, Dropper, Hopper, Brewing Stand,
  Daylight Detector, Note Block, Jukebox,
  TNT, Rail, Powered Rail, Detector Rail, Activator Rail,
  Minecart (all variants: regular, chest, furnace, TNT, hopper, command),
  Command Block, Chain Command Block, Repeating Command Block, Command Block Minecart,
  Redstone Lamp,
  Slime Block, Honey Block? No honey in 1.12.

Tab 4 — Transportation:
  Minecart (regular), Minecart with Chest, Minecart with Furnace, Minecart with TNT,
  Minecart with Hopper, Minecart with Command Block,
  Rail, Powered Rail, Detector Rail, Activator Rail,
  Boat (Oak, Spruce, Birch, Jungle, Acacia, Dark Oak),
  Saddle, Carrot on a Stick, Elytra,
  Lead, Name Tag.

Tab 5 — Miscellaneous:
  All 16 dyes, Bone Meal, Ink Sac, Lapis Lazuli, Cocoa Beans,
  All flowers + plants (same as decorative but also including for crafting reference),
  Enchanted Book, Bottle o' Enchanting, Experience Bottle,
  Book, Book and Quill, Written Book,
  Map (Empty), Map (Filled), Filled Map? Actually: Empty Map, Filled Map,
  Compass, Clock,
  Bucket (Empty), Water Bucket, Lava Bucket, Milk Bucket,
  Snowball, Egg, Bow, Arrow, Spectral Arrow, Tipped Arrow (all effect variants),
  Shield, Totem of Undying,
  Fishing Rod, Flint and Steel, Shears,
  Lead, Name Tag,
  Saddle, Carrot on a Stick, Elytra,
  Firework Rocket, Firework Star,
  Enchanted Golden Apple, Golden Apple, Chorus Fruit, Popped Chorus Fruit,
  Beetroot Soup, Mushroom Stew, Rabbit Stew,
  Glass Bottle, Water Bottle (Potion), Potion (all types), Splash Potion (all types),
  Lingering Potion (all types), Dragon's Breath,
  Spawn Eggs (all 40+ mob types).

Tab 6 — Food & Drinks:
  Apple, Golden Apple, Enchanted Golden Apple,
  Bread, Porkchop, Cooked Porkchop, Fish (Raw + Cooked), Salmon (Raw + Cooked),
  Clownfish, Pufferfish,
  Beef, Cooked Beef, Chicken, Cooked Chicken,
  Mutton (Raw + Cooked), Rabbit (Raw + Cooked),
  Beetroot, Beetroot Soup,
  Carrot, Potato, Baked Potato, Poisonous Potato,
  Melon Slice, Melon Block, Pumpkin Pie,
  Cookie, Cake, Mushroom Stew, Rabbit Stew,
  Chorus Fruit, Popped Chorus Fruit,
  Spider Eye, Rotten Flesh, Raw & Cooked... actually:
  Also: Milk Bucket (drinkable, clears effects),
  Potion (all drinkable types), Splash Potion, Lingering Potion.

Tab 7 — Tools:
  Wooden Pickaxe, Stone Pickaxe, Iron Pickaxe, Gold Pickaxe, Diamond Pickaxe,
  Wooden Axe, Stone Axe, Iron Axe, Gold Axe, Diamond Axe,
  Wooden Shovel, Stone Shovel, Iron Shovel, Gold Shovel, Diamond Shovel,
  Wooden Hoe, Stone Hoe, Iron Hoe, Gold Hoe, Diamond Hoe,
  Flint and Steel, Compass, Clock, Fishing Rod, Shears, Lead, Name Tag,
  Bow, Arrow, Spectral Arrow, Tipped Arrow (all types),
  Shield, Totem of Undying,
  Saddle, Carrot on a Stick, Elytra,
  Firework Rocket, Firework Star,
  Map (Empty), Map (Filled/Created).

Tab 8 — Combat:
  Wooden Sword, Stone Sword, Iron Sword, Gold Sword, Diamond Sword,
  Bow, Arrow, Spectral Arrow, Tipped Arrow (all 14 effect types),
  Shield, Totem of Undying,
  Axe (Wood, Stone, Iron, Gold, Diamond) — axes deal more damage than swords but slower,
  Enchanted Book (with combat enchants),
  Saddle (used for controlling ridden mobs, not combat per se but here),
  Carrot on a Stick (for pig control).

Tab 9 — Brewing:
  Brewing Stand, Glass Bottle, Water Bottle,
  Potion (all types: base + effects), Splash Potion, Lingering Potion,
  Dragon's Breath,
  Ingredients: Nether Wart, Redstone, Glowstone, Gunpowder, Fermented Spider Eye,
  Blaze Powder, Magma Cream, Ghast Tear, Glistering Melon, Golden Carrot,
  Spider Eye, Sugar, Rabbit's Foot, Pufferfish,
  Water Bucket (for water bottles), Cauldron (for water source).

Tab 10 — Materials:
  Coal, Charcoal, Diamond, Emerald, Gold Ingot, Gold Nugget, Iron Ingot,
  Iron Nugget, Lapis Lazuli, Redstone Dust, Redstone Block,
  Nether Quartz, Nether Brick, Nether Wart,
  Paper, Book, Book and Quill,
  Slimeball, Clay, Brick, Glowstone Dust, Gunpowder, Blaze Powder, Blaze Rod,
  Magma Cream, Ghast Tear, Spider Eye, Sugar, Rabbit's Foot, Pufferfish,
  Fermented Spider Eye, Glistering Melon, Golden Carrot,
  Ender Pearl, Eye of Ender, End Crystal, Shulker Shell, Chorus Fruit, Popped Chorus Fruit,
  Bone, Bone Meal, String, Feather, Leather, Leather? Leather is a material.
  Rabbit Hide, Ink Sac, Dye (16 colors),
  Arrow (basic), Spectral Arrow, Tipped Arrow (base),
  Stick, String, Feather, Flint, Obsidian... obsidian is in building blocks.
  Prismarine Shard, Prismarine Crystals,
  Nether Star, Experience Bottle,
  Totem of Undying, Saddle, Elytra, Name Tag,
  Music Discs (13, Cat, Blocks, Chirp, Far, Mall, Mellohi, Stal, Strad, Ward, Wait, 11, Creeper, etc).

Tab 11 — Search:
  Single search tab with a text input box
  Results: all items whose name matches the search query
  Items appear in order of their internal numeric IDs (or alphabetically)
  Search field: auto-focused when clicking the search tab
  Previous searches: saved per session (cleared on world exit)

Tab 12 — Survival Inventory:
  Shows exactly what the player would see in survival mode
  4 armor slots + 27 main inventory + 9 hotbar + offhand slot
  Contents: player's actual survival inventory (if in creative: empty but structured)
  Useful for seeing how an inventory will look in survival mode
  Can drag items from other tabs into this area to equip/wear in creative

Creative inventory interactions:
  - Left-click item: pick up a full stack (64 or max stack size)
  - Right-click item: pick up half a stack (rounded up)
  - Middle-click (scroll wheel press) on block in world: pick block into hotbar
    If block has NBT data (chest, spawner): picks exact block with NBT in creative
  - Q (drop key) while hovering over item in creative inventory: deletes item
  - Click in empty area: drop held item
  - Drag items: standard Minecraft drag behavior (left/right/middle click drag)
  - Number keys: move item to hotbar slot
  - Hover tooltip: shows item name, mod name, and any special properties
  - Creative inventory does NOT show enchantment glint on non-enchanted items
  - Saved toolbar slots visible below main inventory grid

Items ordered by internal ID (for deterministic sorting):
  Item IDs 0-255: blocks in order (stone, grass, dirt, cobblestone, planks, etc.)
  Item IDs 256+: items in order (iron_shovel, iron_pickaxe, etc.)
  In creative tabs: items are grouped visually by function, not strictly by ID
```

### 9B.3 Full 1.12.2 Command List (53 commands)
```
/advancement     - Grant/revoke/test advancements (1.12 NEW)
/blockdata       - Modify block entity NBT data
/clear           - Clear player inventory
/clone           - Clone blocks from region to region
/debug           - Start/stop debug profiling
/defaultgamemode - Set default game mode for new players
/difficulty      - Set difficulty level (peaceful/easy/normal/hard)
/effect          - Add/remove status effects
/enchant         - Enchant player held item
/entitydata      - Modify entity NBT data
/execute         - Execute command as another entity
/experience      - Add/remove player XP (alias: /xp)
/fill            - Fill region with blocks
/function        - Run .mcfunction file (1.12 NEW)
/gamemode        - Change player game mode
/gamerule        - Set/query game rule value
/give            - Give item to player
/help            - Show command help (alias: /?)
/kill            - Kill entities
/list            - List online players
/locate          - Locate nearest structure
/me              - Emote message
/particle        - Spawn particles
/playsound       - Play sound effect
/recipe          - Grant/take recipes (1.12 NEW)
/reload          - Reload loot tables/advancements/recipes/functions (1.12 NEW)
/replaceitem     - Replace slot in inventory/block/entity
/save-all        - Flush all pending saves to disk
/save-off        - Disable auto-saving
/save-on         - Enable auto-saving
/say             - Broadcast message
/scoreboard      - Manage scoreboard objectives/players/teams
/seed            - Show world seed
/setblock        - Set single block
/setidletimeout  - Set idle kick timer
/setworldspawn   - Set world spawn point
/spawnpoint      - Set player spawn point
/spreadplayers   - Teleport entities to random surface locations
/stats           - Track command results
/stop            - Shut down server
/stopsound       - Stop sound effect
/summon          - Summon entity
/teleport        - Teleport entity (alias: /tp)
/tell            - Private message (alias: /msg, /w)
/tellraw         - JSON formatted message
/testfor         - Check if entity exists (legacy)
/testforblock    - Check block at position (legacy)
/testforblocks   - Compare two regions (legacy)
/time            - Set/add/query world time
/title           - Display title/subtitle/actionbar with fade
/toggledownfall  - Toggle weather
/trigger         - Trigger objective (for adventure maps)
/weather         - Set weather (clear/rain/thunder)
/whitelist       - Manage server whitelist
/worldborder     - Manage world border size
/xp              - Add/remove player XP
```

### 9B.4 Complete 1.12.2 Gamerules (22 total)
```
Gamerule                     Default  Description
announceAdvancements          true    Toggle advancement announcements in chat (1.12 NEW)
commandBlockOutput           true    Log command block execution to ops
disableElytraMovementCheck   false   Disable elytra speed check for crashing
doDaylightCycle              true    Day/night cycle progresses
doEntityDrops                true    Entities drop items on death
doFireTick                   true    Fire spreads/decays
doLimitedCrafting           false   Only allow crafting unlocked recipes (1.12 NEW)
doMobLoot                    true    Mobs drop items on death
doMobSpawning                true    Natural mob spawning
doTileDrops                  true    Blocks drop items when broken
doWeatherCycle               true    Weather changes naturally
gameLoopFunction             —       Function run every game tick (1.12 NEW)
keepInventory               false   Keep inventory on death
logAdminCommands             true    Log admin commands to server log
maxCommandChainLength       65536   Max chain length for command blocks
maxEntityCramming            24      Entity collision damage threshold
naturalRegeneration          true    Health regen from full hunger bar
randomTickSpeed               3      Random block ticks per chunk per tick
reducedDebugInfo             false   Reduce F3 debug screen info
sendCommandFeedback          true    Show command feedback messages
showDeathMessages            true    Show death messages in chat
spectatorsGenerateChunks     true    Spectators load chunks

1.12 NEW gamerules: announceAdvancements, doLimitedCrafting, gameLoopFunction
```

### 9B.5 Updated Block Palette (1.12 World of Color)
```
All wool, carpet, shulker boxes, banners, beds → use new "Jonni Palette 0.2" colors
  - More vibrant, saturated colors across the board
  - Affects wool (16), carpet (16), stained clay/terracotta (16),
    stained glass (16), shulker boxes (16), banners (16), beds (16)
  - Concrete and glazed terracotta use the same palette

Color mappings (old → new hex values):
  Color        Old Hex    New Hex    Description
  White        F0F0F0     F9FFFE    Much brighter white
  Orange       EB8844     F9801D    More saturated, punchier orange
  Magenta      C354CD     C74EBD    Slightly pinker
  Light Blue   6F8FC8     3AB3DA    Much lighter, more cyan-blue
  Yellow       BDB016     FED83D    Much brighter yellow
  Lime         4E9B40     80C71F    Brighter, more vibrant green
  Pink         D87E9A     F38BAA    Lighter, pinker
  Gray         3B3B3B     474747    Slightly darker
  Silver       8A8A8A     9D9D9D    Lighter gray
  Cyan         267199     169C9C    More teal-cyan
  Purple       7F3DB5     8932B8    Slightly more vibrant
  Blue         253193     3C44AA    Brighter blue
  Brown        51301A     8B4822    Much lighter brown
  Green        247C16     667F32    More olive-toned
  Red          A1270B     9E2A27    Slightly less saturated
  Black        181414     181414    Unchanged

Banner pattern renames (1.12):
  - "Chief fess" → "Chief"
  - "Base fess" → "Base"
  - Various pattern names normalized
  - Pattern count: 20 banner patterns in 1.12
    (base, border, bricks, circle, creeper, cross, curl, diagonal_left,
     diagonal_right, diagonal_up_left, diagonal_up_right, flower, gradient,
     gradient_up, half_horizontal, half_horizontal_bottom, half_vertical,
     half_vertical_right, mojang, skull, square_bottom_left,
     square_bottom_right, square_top_left, square_top_right, straight_cross,
     stripe_bottom, stripe_center, stripe_downleft, stripe_downright,
     stripe_left, stripe_middle, stripe_right, stripe_top, triangle_bottom,
     triangle_top, triangles_bottom, triangles_top)

Bed changes (1.12 NEW):
  - Beds are now dyeable (16 colors) instead of just red
  - Red bed crafted from red wool only
  - Beds now have 3D model in inventory
  - Beds are slightly bouncy (reduce but don't eliminate fall damage)
  - Crafting: 3 wool + 3 planks (matching wool color determines bed color)
  - Bounce: landing on bed reduces fall damage by 50%
  - Sleeping bag behavior: bed acts as spawn point (respawn at bed on death)

Concrete and Concrete Powder (1.12 NEW):
  - Concrete Powder: falls like sand (gravity-affected block)
  - Concrete Powder: turns into Concrete on contact with water
    (any water: source, flowing, waterlogged block)
    Transformation is instant (block update when water touches)
  - 16 colors matching the new palette
  - Concrete: smooth, solid block (better than terracotta for building)
  - Hardness: concrete = 1.8, concrete powder = 0.5

Glazed Terracotta (1.12 NEW):
  - 16 colors, each with unique pattern
  - Patterns: directional lines, swirls, geometric shapes
  - Hardness: 1.4
  - Can be rotated by right-clicking (changes facing direction)
  - Cannot be moved by pistons? Actually can be moved by pistons.

Knowledge Book (1.12 NEW):
  - Item that gives specific recipes when used
  - Used by map makers to give players specific recipe knowledge
  - Command: /give @s knowledge_book{Recipes:["minecraft:stick","minecraft:torch"]}
  - On use: unlocks listed recipes, plays ding sound, book consumed
  - If doLimitedCrafting is true: recipe book shows only unlocked recipes
```

### 9B.6 Advancements System
```
Advancements replaced achievements in 1.12.
  - Backward compatible: old achievements still tracked for statistics
  - Organized as a tree/graph structure (not sequential)
  - Root: "Minecraft" (title screen advancement tab root)

Advancement data structure:
  {
    "display": {
      "icon": { "item": "minecraft:stone", "nbt": "..." },
      "title": "Stone Age",          // localized string
      "description": "Mine stone",   // localized string
      "frame": "task"|"challenge"|"goal",  // task=square, challenge=special shape, goal=star
      "background": "textures/block/stone.png", // tab background (root only)
      "show_toast": true,   // show toast notification on completion
      "announce_to_chat": true,  // announce in chat on completion
      "hidden": false       // hidden until completed
    },
    "parent": "minecraft:story/root",  // parent advancement (null for root)
    "criteria": {
      "criterion_name": {
        "trigger": "minecraft:inventory_changed",
        "conditions": {
          "items": [ { "item": "minecraft:stone" } ]
        }
      }
    },
    "requirements": [ ["criterion_name"] ]  // AND/OR: outer array = OR, inner = AND
  }

Trigger types (33 total in 1.12):
  All triggers listed in advancements JSON format:
  minecraft:bred_animals, minecraft:brewed_potion, minecraft:changed_dimension,
  minecraft:construct_beacon, minecraft:consume_item, minecraft:cured_zombie_villager,
  minecraft:effects_changed, minecraft:enchanted_item, minecraft:enter_block,
  minecraft:entity_hurt_player, minecraft:entity_killed_player, minecraft:filled_bucket,
  minecraft:fishing_rod_hooked, minecraft:healed_by_poison? Actually: honey_block_slide? No.
  Full list: arbitrary_player_time, bred_animals, brewed_potion, changed_dimension,
  construct_beacon, consume_item, cured_zombie_villager, effects_changed,
  enchanted_item, enter_block, entity_hurt_player, entity_killed_player,
  filled_bucket, fishing_rod_hooked, impaled, impossible, inventory_changed,
  item_durability_changed, killed_by_crossbow? No crossbow in 1.12.
  leveled_up, location, nether_travel, placed_block, player_generates_container_loot,
  player_hurt_entity, player_killed_entity, recipe_unlocked, slept_in_bed,
  slide_down_block? No. summoned_entity, tamed_animal, tick, used_ender_eye,
  used_totem, villager_trade.

  Key triggers for 1.12:
  - minecraft:inventory_changed — items in player inventory match conditions
  - minecraft:location — player at specific coordinates/biome
  - minecraft:impossible — can only be granted via /advancement command
  - minecraft:tick — checks every tick for conditions
  - minecraft:bred_animals — breed two animals
  - minecraft:changed_dimension — enter a specific dimension
  - minecraft:construct_beacon — beacon structure complete
  - minecraft:consume_item — eat/drink specific item
  - minecraft:enchanted_item — enchant at enchanting table
  - minecraft:entity_killed_player — entity kills player (death trigger)
  - minecraft:player_killed_entity — player kills specific entity
  - minecraft:recipe_unlocked — unlock specific recipe
  - minecraft:slept_in_bed — sleep in a bed
  - minecraft:summoned_entity — summon entity with specific properties
  - minecraft:tamed_animal — tame an animal
  - minecraft:used_totem — activate totem of undying
  - minecraft:villager_trade — trade with villager specific item
  - minecraft:brewed_potion — brew specific potion
  - minecraft:filled_bucket — fill bucket with specific fluid
  - minecraft:fishing_rod_hooked — catch specific item while fishing
  - minecraft:cured_zombie_villager — cure zombie villager
  - minecraft:nether_travel — travel through nether portal

Advancement tab categories:
  - minecraft:story/root — "Minecraft" (stone)
  - minecraft:nether/root — "Nether" (netherrack)
  - minecraft:end/root — "The End" (end stone)
  - minecraft:adventure/root — "Adventure" (map)
  - minecraft:husbandry/root — "Husbandry" (hay bale)

Advancement GUI:
  - Access from pause menu (advancement button) or press L
  - Shows a tree/graph structure of advancements
  - Each advancement appears as a colored icon:
    task: rounded square (orange/blue background)
    challenge: special shape (like an anvil, purple)
    goal: star shape (like a beacon, pink)
  - Connections: lines drawn between parent and child advancements
  - Scrolling: click and drag to pan around the tree
  - Hover tooltip: shows title, description, completion status
  - Toast notification: slides in from top right on completion
    task toast: simple icon + title
    challenge toast: icon + title with special border
  - Completed advancements show full color; uncompleted shown faded
  - Advancement progress: partially-completed criteria shown as checkmarks
  - Chat announcement: "<Player> has completed the advancement <Title>!"
  - Reward: each advancement can grant recipes, loot, experience, or function
  - Commands:
    /advancement grant <player> <advancement|namespace:*|*> [!criterion]
    /advancement revoke <player> <advancement|namespace:*|*> [!criterion]  
    /advancement test <player> <advancement> [criterion]

Function system (commands in .mcfunction files):
  Functions are plain-text files with .mcfunction extension.
  Each line is a single command, run sequentially in order.
  Max 65536 commands per function.
  Storage location in assets: data/<namespace>/functions/<path>.mcfunction
  Key features:
    - Functions execute all commands in one game tick
    - Can be run as: /function <namespace>:<path>
    - Support for conditional execution: /function <name> if|unless <selector>
    - No arguments/parameters in 1.12.2 (added in 1.13)
    - Used for datapack-like behavior, adventure maps, and server automation
    - Linked to advancements via rewards: {"function": "namespace:path"}
    - Can be chained recursively (watch for stack overflow)
    - # comments: lines starting with # are ignored
    - Schedule: no /schedule in 1.12 (added 1.14)
    - Max loaded functions: 10000 (hard limit, too many crashes the game)
    - Execution context: runs as the entity that called /function
      @s refers to the calling entity
    - Conditional syntax: if <selector> (run only if selector matches ≥1 entity)
      unless <selector> (run only if selector matches 0 entities)
    - Functions inside functions: use /function recursively (depth limit: 256)
```

---

## P9C — Dimensions & Portal Mechanics

### 9C.1 Dimension Overview
```
Three dimensions, each loaded independently:
  Overworld (dim 0):  y=0..256, bedrock at y=0..4, void below y=0
  The Nether (dim -1): y=0..256, bedrock at y=0..4 and y=123..127, void below y=0 and above y=127
                       Lava seas at y=31, ceiling at y=127
  The End (dim 1):     y=0..256, void below y=0, main island at y=64
                       Outer islands 1000 blocks from center

Per-dimension data stored: spawn point, world seed, time, weather, gamerules
Player data tracks which dimension they are in; on travel, position is saved per dimension
Dimension switching: save player state → unload current dimension → load target → place player

Nether dimension specifics:
  Bedrock floor: y=0..4, with gaps (incomplete layer pattern)
  Bedrock ceiling: y=123..127, with gaps (incomplete layer pattern)
    Pattern: alternating rows of bedrock and air, varies per column (world-seeded)
    Ceiling generally has more gaps; floor has fewer gaps
  Lava oceans: at y=31, large lakes of source lava (replace water with lava in gen)
  Air temperature: constant (hot biome effect, no weather)
  Spawn safety: searches for safe spot near nether-side portal, avoids lava
  Spawning restrictions: no passive mobs, only nether mobs

End dimension specifics:
  Main island: radius ~40 blocks at y=64, center at (0, 64, 0)
    Central obsidian platform: 3×3 at (0, 64, 0) — player spawns here
    End stone ground: circular area around center, edges drop to void
    Obsidian pillars: 10 pillars arranged in a circle (radius ~30-35 from center)
      Pillars are cylindrical (7×7 base), varying heights (10-50 blocks tall)
      Each pillar has an End Crystal on top (healing beam to dragon)
  Outer islands: starting at ~1000 blocks from center
    Large end stone islands with chorus plants and End Cities
    Islands vary in size from small (20×20) to large (100×100)
    Void between islands: y=0 void damage, no bridging structures
  Void: below y=0 (y=0 accessible only on the main island surface)
    End gateway beam: visible from distance (light beam skyward)
```

### 9C.2 Nether Portal
```
Construction:
  - Frame: obsidian blocks in a rectangle
  - Minimum size: 4×5 (inside opening: 2×3)
  - Maximum size: 23×23 (inside opening: 21×21)
  - Corner blocks: NOT required (frame need not be solid corners)
    Frame must form a contiguous ring of obsidian blocks
    Any obsidian block touching the ring counts if it fills the border
  - Lighting: flint & steel, fire charge, fire spread, ghast fireball, or dispenser with flint & steel
  - Activation sound: block.portal.trigger (creeper-like hiss then whoosh)
  - Activation effect: purple portal particles briefly emit from frame

Portal block (minecraft:portal):
  - Purple swirly animated texture (scroll UV over time, 2 layers opposing directions)
  - Light level: 11
  - Tick: standing inside triggers teleport timer
  - Teleport timer: 80 game ticks (4 seconds) standing inside portal block
    Timer resets if player leaves portal block
  - Standing animation: player sinks into portal (small downward offset)
    Screen effect: purple overlay tint increasing over teleport timer
    At 80 ticks: screen fully purple, dimension switches
  - Collision: entities can pass through (no collision box, walk-through)
  - Zombie pigmen can naturally spawn through (1/2000 per tick per portal block)
    Only in Nether, chance per tick per portal block
  - Ghasts and endermen can step through from Nether side
  - Deactivates: broken if any frame obsidian is broken
    Also deactivates if water or lava flows into portal block
    Also deactivates if block placed inside portal opening
    Deactivation: all portal blocks in the frame vanish with puff of smoke
  - Breaking: cannot be mined (drops nothing, instant break with any tool, drops nothing)

Travel mechanics:
  - Overworld → Nether: coordinates ÷ 8 (X/Z only, Y preserved)
    destinationX = floor(x / 8), destinationZ = floor(z / 8), destinationY = y
  - Nether → Overworld: coordinates × 8 (X/Z only, Y preserved)
    destinationX = x × 8, destinationZ = z × 8, destinationY = y
  - Portal search algorithm:
    1. Calculate destination coordinates using ratio
    2. Search for existing portal within search radius:
       - Overworld destination: search 128 blocks of (destX, destY, destZ) for portal frame
       - Nether destination: search 16 blocks of (destX, destY, destZ) for portal frame
    3. Search shape: concentric expanding box search from center
    4. If existing portal found: teleport to that portal's center
    5. If not found: create new portal
       - Scan from destY upward/downward for a suitable location (open space 4×3×4)
       - Scan range: y=2 to y=250 (skip if no space found)
       - Create portal frame at first suitable location
       - Ensure portal is grounded (solid blocks below)
       - If in Nether: place netherrack platform below portal
       - If hanging over void/lava: create obsidian bridge platform
    6. Placement bias: prefer existing portal if within radius
  - Portal cooldown: player cannot re-enter portal for ~1s (20 ticks) after exiting
    Prevents infinite teleport loops (portal → destination → immediate re-entry)
    Cooldown is per-player, tracked as lastPortalExitTime
  - Zombie pigmen: can spawn through Overworld-side portals
    Chance: 1/2000 per tick per portal block when spawned (in Nether)
    Spawn location: near portal on Nether side
  - Hostile mobs can travel through portals (items, boats, minecarts also)
    Entity teleport: same mechanics as player, entities must stand in portal for 80 ticks
    Item teleport: items can float into portal and teleport (useful for item transport systems)
    Boat/minecart: entity inside vehicle in portal → vehicle + passenger teleport together
  - Multiple portals in close range: portal linking prefers closest portal to destination point
  - Portal ticking: portal blocks check frame integrity every tick
    If frame broken: all portal blocks in that frame deactivate (instant removal)

Nether portal rendering:
  Portal texture: 2 scrolling layers, UV offset over time
    Layer 1: scrolls in one direction (speed 0.1 UV/tick)
    Layer 2: scrolls opposite direction (speed 0.05 UV/tick)
    Blended together with multiply or additive blend
  Portal particles: purple star-like particles (drip or float upward)
    Spawn rate: 1-3 particles per second per portal block
    Particle: endRod (purple, 0.5×0.5 scale, floats up slowly)
  Overlay: when player is in portal, screen gets purple tint
    Tint alpha = clamp(ticksInPortal / 80, 0, 1)
    Full purple at teleport moment
    Sound: block.portal.ambient (whooshing)
    Sound: block.portal.travel (on teleport)
```

### 9C.3 End Portal
```
Construction (stronghold portal room):
  - 12 End Portal Frame blocks arranged in a ring
    Ring: 5×5 square with corners missing (3 per side)
    Positions relative to room center:
      (-2,-2), (-2,-1), (-2,0), (-2,1), (-2,2),
      (-1,-2),                                        (-1,2),
      (0,-2),                                         (0,2),
      (1,-2),                                         (1,2),
      (2,-2),  (2,-1),  (2,0),  (2,1),  (2,2)
  - Each frame has a facing direction (toward center of circle)
    Frames along X axis: face +Z or -Z
    Frames along Z axis: face +X or -X
  - Frame blocks: cannot be mined (blast resistant, infinite hardness)
  - Eye slot: right-click with Eye of Ender to insert
    Insert sound: block.end_portal_frame.fill
    Eye remains permanently in frame once placed
    Eye visually appears in the frame model (change in blockstate JSON)
  - All 12 eyes required to activate portal
    When all 12 inserted: portal interior activates (star field effect)
    Activation: whoosh sound, portal blocks appear inside ring
  - Activated frame: emits portal effect (star field, light 15)
    Frame visually changes (blockstate: eye=true, facing direction)

End Portal block (minecraft:end_portal):
  - Star field animated texture (teleport effect, scrolling stars)
  - Light level: 15 (brightest light in the game)
  - Collision: none (entities fall through, no solid collision box)
  - Standing inside: teleported to The End at y=64, x=0, z=0 (center of main island)
    Teleport is instant (no timer like nether portal)
    Player appears on the 3×3 obsidian platform at (0,64,0)
  - Cannot be broken or obtained in survival
    In creative: can be broken with click (removes portal block temporarily)
  - Water can flow through portal blocks (flows into the portal opening)
  - Breaking all frame eyes deactivates portal (eyes pop out as items)
    Only possible by right-clicking frames with empty hand to remove eyes
    In survival: cannot remove eyes from activated portal
    In creative: shift-right-click removes eye
  - Multiple end portals in same stronghold? Usually only 1 per stronghold
    But if multiple strongholds are generated close, each has its own portal

Eye of Ender:
  - Crafted: 1 ender pearl + 1 blaze powder (shaped recipe, no pattern)
    Returns: 1 eye of ender
  - Thrown (right-click): travels toward nearest stronghold
    Flight path: travels in 2D toward stronghold X/Z, at y=playerY
    Speed: 2 blocks/tick
    After 12-20 throws: adjusts trajectory to go into ground (arrived at stronghold)
    Navigation particle: particles trail behind flying eye (purple portal particles)
  - Upon landing:
    1. Eye hovers over ground for 1-2 seconds
    2. Eye drops as item entity (65% chance to shatter = item entity disappears)
       Shatter sound: entity.eye_of_ender_shatter.shatter
    3. If not shattered: eye of ender item drops and can be picked up
  - Sound on throw: entity.ender_eye.launch (pop)
  - Sound on land: entity.ender_eye.death (ding/clink)
  - Placed in frame: emits entity.ender_eye.death (ding sound), eye stays permanently
  - In creative: thrown eye does not get consumed (stays in stack)
  - Eye travels through the air: affected by gravity? No, it flies straight
    Only adjusts height when close to target (dives into ground)

End Portal rendering:
  Portal texture: star field, uses UV animation (slow rotation)
    Stars: white dots on dark blue background
    Rotation: portal texture rotates slowly (1 revolution per 1200 ticks)
  Frame block: end_portal_frame texture with eye insert
    Without eye: frame has empty slot (dark hole)
    With eye: frame has green-tinted eye texture, slight glow
    Facing direction controls which side of block has the frame opening
  Particles: endRod particles emit from activated portal (upward float)
    Density: moderate (~5 particles/second)
```

### 9C.4 End Gateway
```
Spawning: 20 gateways spawn after Ender Dragon death
  - 1 gateway: spawns at the edge of the main island
    Radius: ~70-80 blocks from center (0,0)
    Position: nearest safe spot on main island surface at that radius
  - 19 more gateways: spawn around the edge of the main island
    Evenly distributed in a circle around (0,0)
    Each ~70-80 blocks from center but at various angles
    Offset: each gateway is placed above ground on 1-block tall bedrock pillar
  - Gateway block appears on top of a small bedrock pillar (1-2 blocks tall)
  - Cannot be activated until all gateways have spawned
  - Triggered by dragon death: animation of light beams appearing

End Gateway block (minecraft:end_gateway):
  - Light level: 15
  - Teleport destination: closest empty area at y=72 near the gateway target
    Each gateway has a pre-calculated destination in outer islands
    Destination: ~1000 blocks from center island in the direction the gateway faces
  - Can send through: player, entities, items, projectiles
  - Teleport effect: purple portal particles, instant teleport
  - Throwing an ender pearl through: the pearl teleports the player
  - Cannot be broken or obtained (end_gateway block is not obtainable)
  - Has a vertical beam of light extending upward from the gateway
    Beam: white/blue light column, visible from any distance
    Rendering: beam uses a separate shader (cone of light texture)
    Beam reaches to y=256 (top of world)
  - Animation: when dragon dies, gateway beams appear one by one (brief animation)
  - If multiple entities enter same gateway simultaneously: each teleported separately
  - Cooldown: none (entities can go through repeatedly)
  - Gateway exit: entity appears at destination with brief disorientation effect
    Camera readjusts to new position (players see teleport effect)

Outer islands generation (post-dragon):
  Islands begin at approximately 1000 blocks from center
  Generation algorithm (world seed based):
    1. Generate island clusters using noise-based placement (scale: 2000 blocks)
    2. Each cluster: 1-3 main islands with smaller satellite islands
    3. Island shape: ellipsoidal, radii vary 20-100 blocks
    4. Island surface: end stone, y=60-70 height
    5. Surface detail: chorus plants randomly placed
    6. End Cities: 50% chance per island cluster
    7. End Ships: 25% chance per End City occurrence
  Island density: sparse (islands separated by 50-200 blocks of void)
  Total area: islands generated up to ~10000 blocks from center? Practically unbounded
  Performance: islands only generated when player travels to them
    Use chunk-based generation: when a chunk at >1000 blocks loads, decide if island exists
    If chunk is void: generate nothing (air/void below y=0)
    If chunk is island: generate end stone, chorus, structures
```

### 9C.5 The End — Main Island Generation Details
```
Main island generation:
  Center: (0, 64, 0) — 3×3 obsidian platform (player spawn point)
    Platform is generated as part of world spawn, not natural terrain
    If platform is removed, it does not regenerate (single-player: only generated once)
  Terrain shape: circular end stone mass
    Radius: ~40 blocks from center
    Thickness: 6-10 blocks (y=60 to y=70)
    Top surface: flat-ish end stone, slight height variation (±2 blocks)
    Edges: steep dropoff to void (sheer cliffs of end stone)
  Obsidian pillars (10 total):
    Positions: arranged in a circle around central island
      Radius: 28-38 blocks from center
      Equal angular spacing: 36° apart (360°/10)
      Offset: each pillar position has slight randomization (±2 blocks)
    Pillar structure:
      Base: 7×7 blocks of obsidian (square column)
      Height: varies per pillar, ranges from 10 to 50 blocks tall
        Taller pillars have iron bars extending upward from obsidian top
      Top: flat obsidian surface with 1 End Crystal
        End Crystal sits on top of pillar (floating above)
        Crystal has bedrock base + glass sphere + spinning crystal
      Pillar height randomization: seeded per world, determined at world gen
    Iron bars: form a cage around the top of some pillars
      Present on pillars taller than 30 blocks
      Ascending bars: spiral/zigzag pattern up the pillar
      Some pillars have no iron bars (shorter ones)
    Bedrock core: each pillar has a single bedrock at its center (y=64)
      Used for pillar anchoring during generation
      Not visible (inside obsidian)
  End Crystals:
    Purpose: heal the Ender Dragon (beam connects dragon to crystal)
      Healing beam: white beam from crystal to dragon
      Healing rate: 1 HP per 0.5 seconds (10 ticks)
      Dragon HP: 200 (100 hearts)
    Crystal destruction: player must break crystal before fighting dragon
      Crystal explosion: power = 6 (charged creeper level) when destroyed
      Chain reaction: destroying one crystal may trigger nearby crystals to explode
      End Crystal item: can be placed on obsidian/bedrock to respawn dragon
    Crystal rendering:
      Base: bedrock slab (1 block, flat)
      Glass sphere: transparent glass cube (slightly larger than 1 block)
      Inner crystal: spinning geometric shape (rotates slowly)
      Beam: white beam upward to dragon (or idle when no dragon)
  Exit portal (post-dragon):
    Spawns at (0, 64, 0) after dragon death
    Structure: bedrock ring + 1 end gateway block
      Ring: bedrock blocks arranged in a circle (like end portal but different material)
      Inside: portal blocks (teleport back to Overworld at world spawn)
      Egg: dragon egg sits on top of exit portal (center)
    Dragon egg:
      Sits on top of exit portal after dragon death
      Drops when broken with piston (or when torch placed below)
      Teleports randomly when right-clicked
      Decorative only (cannot be placed as a block in survival)
    End gateway beam: activates after dragon death, leads to outer islands
```

### 9C.6 Chorus Plant & Flower Mechanics
```
Chorus plant generation (outer islands):
  Chorus flower age: 0-5
  Growth algorithm (tick-based, random):
    1. Flower at age 0-4 can grow on any tick (1/3 chance per random tick)
    2. Growth attempt:
       - Pick random direction (up, N, S, E, W)
       - If direction is up: check if height < 128 blocks (y < 256)
         Place chorus flower above (new age 0)
         Current flower age increases by 1
       - If direction is horizontal: place chorus plant block
         Check that target is air
         Check that source block is chorus plant or flower
         Plant mutations: branch cannot be more than 2 blocks from trunk
    3. Age 5 flower: stops growing (dead flower, bears no further growth)
    4. Age 4 flower: has 2/3 chance to place fruit (chorus fruit) and stop
       Fruit drops as item when flower is broken
  Growth constraints:
    - Only grows on end stone
    - Light level: any (no light requirement)
    - Space: requires 1 free block above
    - Max height: chorus plant can grow up to 22 blocks tall
    - Branching: limited to 2 branches per trunk node
    - Branch length: max 6 blocks from origin
  Breaking chorus plant:
    - Breaking any chorus plant block: all connected plant blocks above break
      Chain break: entire branch above break point shatters
      Each block drops: 0-1 chorus fruit (20% chance per block)
    - Chorus flower: drops itself if age 0-4 (can be replanted)
      Age 5: drops nothing (dead, cannot regrow)
  Chorus fruit:
    - Food: restores 4 hunger (2 drumsticks)
    - Teleport effect: 25% chance to teleport player up to 8 blocks in random direction
      Teleport: attempts to place player at random nearby safe spot
      If no safe spot: no teleport (fruit still consumed)
    - Can be cooked: popped chorus fruit (crafting ingredient for purpur blocks)
    - Sound: item.chorus_fruit.teleport (poof, enderman teleport sound)
  Chorus flower planting:
    - Can only be planted on end stone
    - Must have air above (min 1 block)
    - Planted flower starts at age 0
    - Bone meal: no effect on chorus flower (cannot speed growth)
```

### 9C.7 End City & Ship Generation
```
End city generation:
  Location: outer islands (chunks with end city flag)
  Structure shape: tower-like, branching
  Building materials: purpur blocks, purpur pillars, end stone bricks
    Purpur block variants: block, slab, stairs, pillar
    End stone bricks: blocks, slabs, stairs
    Obsidian: used for boat chest room floors
  Tower types:
    1. Base tower: 3-4 floors wide at bottom, each floor smaller
       Floor height: 3-4 blocks
       Windows: purpur slab-framed openings
    2. Middle tower: narrower, continues upward
       Bridges: connect towers horizontally
    3. Top tower: spire, tapers to a point
       Roof: purpur slab stairs leading to spire
  Rooms:
    - Treasure room: single chest with loot (found at top of tallest tower)
    - Chest rooms: 1-3 chests per city, distributed throughout towers
    - Shulker spawn: 1-2 shulkers per tower (on stairs, platforms, or in rooms)
  Ships:
    - 25% chance per city to generate a ship
    - Parked near the top of the tallest tower (connected by bridge)
    - Ship structure:
      Hull: purpur blocks, purpur pillars
      Mast: fence post (spruce fence) going up through ship
      Sails: white wool (2-3 sails per ship)
      Bow: dragon head (decorative item, wearable)
        Dragon head: 13% chance to be found in the ship's treasure room
      Treasure room: in the ship's interior
        Single chest with end city loot table
        Two shulkers guard the treasure room
  Loot tables (see P9I — Structure Loot Tables for exact compositions):
    - end_city_treasure: chest in ship + top tower room
    - end_city_chest: chests in lower tower rooms
```

### 9C.8 Nether Fortress Generation
```
Nether fortress generation:
  Location: Nether dimension, any biome
  Distribution: fortress clusters spaced ~200-400 blocks apart
  Generation: uses structure seed (derived from world seed)
    Attempts: 4 generation attempts per region (region size: 480×480 blocks)
    Each attempt: 33% chance to place a fortress
  Structure orientation: aligned to cardinal directions (N/S or E/W)
  Main components:
    1. Central corridor: main walkway, 3 blocks wide, nether brick
       Floor: nether brick blocks
       Walls: nether brick fence (railing)
       Roof: nether brick blocks (some open to ceiling)
       Length: varies from 50 to 150 blocks
    2. Cross corridors: branching off main corridor at regular intervals
       Similar construction to central corridor
       Intersection: open plaza area with stairs
    3. Bridges: narrow walkways over lava lakes
       Width: 3 blocks
       Railings: nether brick fence on both sides
       Supports: nether brick pillars down to lava level
       Can span up to 100 blocks
    4. Staircases: connecting different levels of the fortress
       Stair material: nether brick stairs
       Stair type: straight, spiral (for towers)
    5. Blaze spawner rooms:
       Structure: small room (7×5×4), contains 1-2 blaze spawners
       Spawner: enclosed in nether brick fence cage
       Location: end of a corridor or bridge
       Each fortress typically has 2-4 blaze spawner rooms
    6. Nether wart rooms:
       Structure: stairwell leading down to small room
       Contains: soul sand patches with pre-grown nether wart (1-3 blocks per patch)
       Location: one per fortress (rare)
    7. Lava well rooms:
       Structure: small room with lava source in center
       Fence: nether brick fence around lava
       Location: common, 1-2 per fortress
  Fortress features:
    - Blaze spawners: spawn blaze at light level ≤ 11
      Spawn range: 4 blocks horizontal, 1-3 blocks vertical
      Spawn rate: 1/25 ticks per spawner attempt
      Max blaze: 5 in area around spawner
    - Wither skeletons: natural spawn in fortress corridors
      Spawn rate: 1/20 chance per tick per spawn attempt
      Max: 5 wither skeletons in fortress area
    - Nether brick blocks: fortress is primarily built from nether brick
      Variations: nether brick (block), nether brick fence, nether brick stairs
      Red nether brick: NOT in 1.12 (added 1.16)
    - Soul sand: found in nether wart rooms and occasionally in corridors
    - Gravel: small patches in corridors (like path decoration)
  Mob spawning within fortress:
    - Blaze: only from spawners (not natural spawn)
    - Wither skeleton: natural spawn in fortress (replaces regular skeleton in fortress bounding box)
    - Zombie pigman: natural spawn everywhere (fortress has no special restrictions)
    - Magma cube: natural spawn in fortress (higher rate than nether waste)
  Fortress bounding box:
    - Determines where wither skeletons spawn
    - Box extends 2 blocks outward from fortress blocks
    - Inside box: skeleton spawns become wither skeleton (1/3 chance wither, 2/3 regular)
      Actually: within bounding box, 80% wither skeleton, 20% regular skeleton
    - Outside box: regular skeleton behavior
  Post-generation:
    - Fortress generates once per chunk
    - If a chunk contains fortress pieces, the pieces are placed during chunk population step
    - Fortress can be partially destroyed by player (blocks mined, replaced)
    - Blaze spawners cannot be mined in survival (no silk touch drop for spawners)
```

### 9C.9 Bedrock Layer Generation Patterns
```
The Nether has two incomplete bedrock layers:

Bottom bedrock (y=0..4):
  Layer distribution:
    y=0: 100% bedrock (solid floor)
    y=1: 60% bedrock (semi-random gaps)
    y=2: 30% bedrock (sparse)
    y=3: 20% bedrock (very sparse)
    y=4: 5% bedrock (rare)
  Gap generation: world-seeded noise determines which positions have bedrock
    At each layer, for each (x,z) position:
      randomThreshold = noise(x, z, layerSeed) mapped to [0,1]
      if randomThreshold < layerFillFactor: place bedrock
    Gap size: 1-3 blocks typically, larger gaps at higher layers
  Purpose: prevents falling into void, but allows some openings for travel

Top bedrock (y=123..127): same pattern but inverted Y (y=123 has most gaps)
  Layer distribution:
    y=127: 100% bedrock (solid ceiling)
    y=126: 60% bedrock
    y=125: 30% bedrock
    y=124: 20% bedrock
    y=123: 5% bedrock
  Purpose: prevents escaping the Nether through the top, but allows some openings
    Players can find gaps to build above the Nether ceiling (useful for farms)
  Note: In 1.12, building above the Nether ceiling is possible through gaps
    Above y=127: empty space, no terrain generation
    No mob spawning above ceiling

Overworld bedrock (y=0..4): same pattern as Nether bottom
  y=0: 100% bedrock
  y=1-4: decreasing fill factors
  Purpose: prevents falling into void
```

### 9C.10 Void Damage & Dimension Exit
```
Void damage:
  Overworld: void damage below y=0, 4 damage (2 hearts) every 0.5s (10 game ticks)
  Nether:    void damage below y=0 and above y=127, same rate
  The End:   void damage below y=0, same rate
  Void kills player regardless of health: bypasses armor, absorption, resistance
    Damage type: "void" (bypasses all protection, cannot be mitigated)
    Exception: creative/spectator mode (no void damage)
    Exception: /gamerule doVoidDamage false? Not a valid gamerule in 1.12
  Damage source: "outOfWorld" in the damage system
  Death message: "<Player> fell out of the world"
  Void fog: at y < 16, fog becomes dense and dark (gradual, fades to black at y=0)
    Fog color: dark blue-black (#000022)
    Fog density: exponential below y=16, max at y=0
    Visual: blocks below y=16 are obscured by void fog
    At y=0: full black screen (player cannot see anything)
  Item entities in void: take damage at same rate as player, destroyed after 2 ticks
    Despawn instantly in void? Actually: items take void damage, destroyed after ~1 second
  Mobs in void: take damage at same rate as player, die quickly
    Except: endermen can survive briefly (teleport out if possible)

Exiting to main menu:
  Dimension unload sequence:
    1. Stop all chunk ticking (block entities, random ticks)
    2. Despawn all entities
    3. Unload all chunks
    4. Clear render buffers
    5. Set dimension state to "unloaded"
  Dimension load sequence:
    1. Load dimension data (spawn, time, weather)
    2. Generate spawn chunks (default: 12×12 around spawn point)
    3. Initialize entities (player only initially)
    4. Start chunk ticking
    5. Set dimension state to "active"
  Dimension switch:
    1. Save player state (inventory, position, health, etc.)
    2. Run unload sequence for current dimension
    3. Run load sequence for target dimension
    4. Place player at correct position (portal destination or spawn)
    5. Apply dimension effects (respawn anchor? no, that's 1.16)
```

---

## P9D — Weather, Time & Sleep

### 9D.1 Day/Night Cycle
```
Full cycle: 24000 game ticks = 20 minutes real-time
  Day:       0-12000 ticks  (6:00 - 18:00 in-game hours)
  Sunset:    12000-13800 ticks (18:00 - 19:00)
  Night:     13800-22200 ticks (19:00 - 5:00)
  Sunrise:   22200-24000 ticks (5:00 - 6:00)

Per-tick timing:
  1 game tick = 0.05s → 24000 ticks = 1200s = 20 minutes
  1 in-game hour = 1000 ticks = 50 seconds real-time
  /time set day = 1000 | /time set night = 13000
  /time set noon = 6000 | /time set midnight = 18000

Celestial angle calculation:
  celestialAngle = (time / 24000) % 1.0
  sunAngle = celestialAngle × 360° (in radians for shader)
  Sun position: sunX = cos(sunAngle), sunY = sin(sunAngle) × -1 (below horizon at night)
  Moon position: exactly opposite, moonAngle = sunAngle + 180°, plus 180° offset
  Sun rises in the east (+X direction), arcs across the sky, sets in the west (-X)
  At noon (6000 ticks): sun at zenith (directly overhead)
  At midnight (18000 ticks): moon at zenith
  Sun below horizon when Y < 0 (nighttime rendering uses different sky color)

Sky light level by time of day:
  Day (0-12000):       sky light = 15
  Sunset (12000-13800): sky light = 15 → 4 (linear interpolation over 1800 ticks)
  Night (13800-22200):  sky light = 4
  Sunrise (22200-24000): sky light = 4 → 15 (linear interpolation over 1800 ticks)

Sky color rendering (per-vertex or fullscreen gradient):
  Day sky:      gradient from light blue (#87CEFA at horizon) to deep blue (#4A90D9 at zenith)
  Sunset sky:   gradient through orange (#FF8C00) → red (#FF4444) → purple (#8B4488) → dark blue
  Night sky:    gradient from dark blue (#0A0A2E) to near-black (#000011)
  Sunrise sky:  reverse of sunset, purple → red → orange → light blue
  Sky color interpolated based on celestial angle, with smooth transitions
  Fog color: matches sky color at horizon, blends based on distance and weather

Moon phases (8 phases, cycle repeats every 8 in-game days):
  Phase 0 — Full Moon:    moon texture fully lit, +0-0.5 random bonus to regional difficulty
                           Slime spawns: enabled in swamps (any light level)
  Phase 1 — Waning Gibbous: 75% lit face (left side)
  Phase 2 — Waning Quarter: 50% lit face (left half)
  Phase 3 — Waning Crescent: 25% lit face (left side)
  Phase 4 — New Moon:      moon dark (nearly invisible), slime spawns: disabled in swamps
  Phase 5 — Waxing Crescent: 25% lit face (right side)
  Phase 6 — Waxing Quarter: 50% lit face (right half)
  Phase 7 — Waxing Gibbous: 75% lit face (right side)
  Phase calculation: moonPhase = floor((worldTime / 24000) % 8)
  Each phase lasts 1 in-game day (24000 ticks, 20 minutes real-time)
  Moon texture: 8 distinct textures or single texture with phase mask applied

Star field:
  Number of stars: 800-1500 drawn as small bright points
  Position generation: seeded random from world seed (deterministic per world)
    Each star: random position on celestial sphere (θ, φ) using Fibonacci sphere distribution
    Star size: 1-3 pixels depending on brightness (random magnitude)
    Star color: pure white with slight variation (#FFFFFF to #FFEEDD)
  Rotation: stars rotate with the moon (same celestial sphere), opposite of sun
  Twinkle: stars twinkle by modulating alpha (sin wave with random phase per star)
    Frequency: per-star random 1-5 Hz
    Amplitude: 0.3-0.7 alpha swing
  Rendering: point sprites or small quads on the sky dome, alpha blended
  Visibility: stars visible when sky light < 4 (night) and weather is clear
    At twilight (sky light 4-11): partial visibility interpolated from 0 to full
    During thunder: stars hidden (cloud layer blocks view)

Ambient light levels (block face rendering):
  Sky light 15: full brightness (RGB multiplier 1.0)
  Sky light 4 (night): RGB multiplier ~0.2-0.3 (moonlight tinted slightly blue)
  Internal light interpolation: sky light contributes 0.8× + block light 0.2× to final brightness
  Smooth lighting: interpolate light values across block faces using ambient occlusion
```

### 9D.2 Weather System
```
Weather states: clear, rain, snow, thunder
Each world tracks:
  clearTime:   remaining ticks of clear weather (or -1 for default cycle)
  rainTime:    remaining ticks of rain/snow (or -1 for default cycle)
  thunderTime: remaining ticks of thunder (or -1 for default cycle)
  isRaining:   boolean
  isThundering: boolean

Weather transition timers (default random cycle):
  clear duration:   uniform random between 6000-180000 ticks (5 min - 2.5 hours real-time)
                      ≈ 0.5-7.5 in-game days
  rain duration:    uniform random between 12000-24000 ticks (10-20 min real-time)
                      ≈ 0.5-1 in-game days
  thunder duration: uniform random between 3600-15600 ticks (3-13 min real-time)
  Transition probability (per tick during clear, 1/clearTime remaining chance):
    P(rain) = 0.25 per tick when rainTime expires or natural transition
    P(thunder) = 0.25 per tick from rain (thunder is a rain subtype)
  Natural cycle: clear → rain → (clear or thunder) → clear → ...

Weather effects by biome:
  Biome temperature threshold for snow: temperature ≤ 0.15
  Hot biomes (desert, savanna, mesa): no precipitation at all
    temperature ≥ 2.0: no rain particles, no sky darkening from clouds
  All other biomes: rain when precipitation occurs
  Cold biomes (taiga, ice plains, cold taiga, extreme hills above snow line): snow
    snow line: y > 90 + biome-specific offset (varies by biome, typically y>90-120)
    In cold biomes at y < snow line: rain (if above-freezing temperature in biome)

Rain rendering:
  Particle: falling water droplet (white/light blue streak, 1×6 pixels)
  Spawn rate: ~100-500 particles visible around player (density varies by intensity)
  Spawn area: horizontal 10×10 block area centered on player, y from 0 to player y+10
    Only spawns where sky is visible (not under blocks)
  Particle velocity: v_y = -0.25 blocks/tick (constant), slight random x/z drift (wind)
  Wind: small horizontal velocity component, direction changes slowly
  Particle lifetime: ~40 ticks (2 seconds) or until hitting ground/block
  Splash effect: particles hitting blocks create tiny splash particles (1 tick)
  Sound: rain ambiance plays as positional audio around player
    Intensity: interpolated between 0 and 1 based on precipitation density
    Rain sound volume: 0 to 1, fades over ~20 ticks on transition
  Rain texture: if using sprite sheets, rotate through 4-8 droplet variations
  Performance: pool 1000 particles, recycle off-screen particles

Snow rendering:
  Particle: white hexagonal crystal (2×2 pixels, slight white glow)
  Spawn rate: ~50-200 particles visible (half of rain density)
  Spawn area: horizontal 15×15 block area, y from y=256 down to player y-5
  Particle velocity: v_y = -0.05 to -0.10 blocks/tick (falls slower than rain)
    Horizontal drift: random, affected by wind (swaying motion)
  Particle lifetime: 100-200 ticks (5-10 seconds) or until hitting ground
  Snow accumulation: 
    - Snow layers form on top blocks exposed to sky in snow biomes
    - During snowfall: 1/20 chance per tick per exposed block to add a snow layer
    - Max accumulation: 1 block (8 layers) during active snowfall
    - Snow layers: 0-7 (8 layers = 1 full block)
    - Existing snow layers increase by 1 when new snow falls (up to 7)
  Snow behavior: snow layers break when block below is broken, drop snowballs
  Sound: no snow impact sound (subtle, different from rain)

Thunderstorm:
  Rain + thunder + lightning + darker sky
  Sky light during thunder: reduced by 2 from normal rain (sky light 2 at night)
  Sky color: darker grey, clouds more prominent
  Thunder sound: plays ~1-3 seconds after lightning flash (distance-based delay)
    Delay = distance_in_blocks / 340 (speed of sound ≈ 340 m/s)
    Volume: attenuates with distance from strike point
    Sound: low rumble, random pitch variation ±0.1
  Cloud layer: rendered as dark grey semi-transparent overlay at y=256-260
    During clear: no cloud layer
    During rain: light grey clouds, opacity 0.3-0.5
    During thunder: dark grey clouds, opacity 0.6-0.8

Lightning strike mechanics:
  Trigger: per chunk per tick, chance = 1/100000 (during thunder)
    Average: 1 strike every ~5 seconds across loaded area (at render distance 10)
  Target selection:
    1. Pick random (x, z) within chunk
    2. Find highest block with sky access at that position (raycast downward)
    3. If no block found (void), skip
    4. Lightning strikes at that block's surface
  Visual:
    Bolt: branching white/yellow lightning bolt, 3-5 branches
      Main bolt: from cloud to ground (zigzag line segments)
      Sub-branches: 2-4 offshoots from main bolt, shorter (20-40% of main length)
        Branch angle: 30-60° from main bolt
    Duration: 1-2 ticks (visible for only 1 frame at 20fps, but sprite lingers)
    Flash: full-screen white flash, 0.1s duration (2 ticks), fades rapidly
      Flash intensity: sky flash 0.3-0.5 (day) to 0.6-0.9 (night), exponential decay
      Flash color: white with slight blue tint
  Effects of strike:
    - Creates fire at strike point: 1-4 fire blocks on top of struck block (if flammable surface)
      Fire spreads per normal fire tick (if conditions allow)
    - Damage to entities within radius:
      Direct hit (same block): 5 HP (2.5 hearts)
      Nearby (2 blocks): 2.5 HP (1.25 hearts)
      Within 4 blocks: electric shock (no HP damage but entities are pushed)
    - Charged creeper: if creeper within 4 blocks of strike → turns charged
      Charged creeper: blue aura, explosion power × 1.5 (power = 6 from 4)
    - Pig → Zombie Pigman: if pig within 4 blocks of strike → transforms
      Drops: pig drops (none), zombie pigman equipment spawns
    - Villager → Witch: if villager within 4 blocks of strike → transforms
      Only on Hard difficulty (Easy/Normal: villager struck but no transform?)
      Actually: only on Hard difficulty does villager become witch from lightning
    - Entity knockback: entities within 3 blocks pushed away from strike point
  Thunder sound: plays after flash, delay = distance / 340s
    Maximum audible range: 64000 blocks (practically: within loaded chunks)
    Sound event: entity.lightning.thunder
  Per-chunk cooldown: no more than 1 strike per chunk per 10 seconds (prevent double-strikes)

Weather transition mechanics:
  Clear → Rain: gradual transition over 60-80 ticks (3-4 seconds)
    1. Sky color begins darkening (grey tint increases)
    2. Rain particles start at low density, ramp up to full over transition period
    3. Cloud layer opacity increases from 0 to 0.4
    4. Fog color shifts to grey
    5. Rain sound fades in
  Rain → Clear: reverse of above over 60-80 ticks
  Rain → Snow: instantaneous (based on biome temperature check per column)
    Actually: the rainfall mechanic checks biome; cold biomes show snow directly
    Transition from rain area to snow area: smooth particle type change over 1-2 chunks
  Rain → Thunder: secondary transition, clouds darken further over 20-40 ticks
    Sky darkens additional 2 light levels
    Thunder sounds occasionally during transition (lightning not yet active? Actually lightning
    becomes active as soon as thunderTime > 0, which starts when isThundering = true)
  Weather change from /weather command: instant change (no transition)
    /weather clear [duration]    — duration in seconds, converts to ticks internally
    /weather rain [duration]     — as above
    /weather thunder [duration]  — as above
  /toggledownfall: toggles rain state immediately (legacy command)

Biome precipitation lookup table:
  Biome                        Temperature  Precipitation Type  Snow Line
  Ocean                        0.5          rain                N/A
  Plains                       0.8          rain                N/A
  Desert                       2.0          none                N/A
  Extreme Hills                0.2          rain                y=120
  Forest                       0.7          rain                N/A
  Taiga                        0.0          snow                y=64
  Swampland                    0.8          rain                N/A
  River                        0.5          rain                N/A
  Hell (Nether)                N/A          none                N/A
  Sky (The End)                N/A          none                N/A
  FrozenOcean                 0.0           snow                y=64
  FrozenRiver                 0.0           snow                y=64
  Ice Plains                   0.0          snow                y=64
  Ice Mountains                0.0          snow                y=64
  MushroomIsland               0.9          rain                N/A
  MushroomIslandShore          0.9          rain                N/A
  Beach                        0.8          rain                N/A
  DesertHills                  2.0          none                N/A
  ForestHills                  0.7          rain                N/A
  TaigaHills                   0.0          snow                y=64
  Extreme Hills Edge           0.2          rain                y=120
  Jungle                       0.95         rain                N/A
  JungleHills                  0.95         rain                N/A
  JungleEdge                   0.95         rain                N/A
  Deep Ocean                   0.5          rain                N/A
  Stone Beach                  0.8          rain                N/A
  Cold Beach                   0.05         snow                y=64
  Birch Forest                 0.6          rain                N/A
  Birch Forest Hills           0.6          rain                N/A
  Roofed Forest                0.7          rain                N/A
  Cold Taiga                    -0.5        snow                y=48
  Cold Taiga Hills              -0.5        snow                y=48
  Mega Taiga                   0.3          rain                y=96
  Mega Taiga Hills             0.3          rain                y=96
  Extreme Hills+               0.2          rain                y=120
  Savanna                      1.2          none (rain in thunder? Actually thunderstorms
                                            still produce rain in savanna? No: no precipitation
                                            in savanna at all.)       N/A
  Savanna Plateau              1.0          none                N/A
  Mesa                         2.0          none                N/A
  Mesa Plateau                 2.0          none                N/A
  Mesa Plateau F               2.0          none                N/A
  (All remaining 1.12 biomes follow temperature rule)

Weather effects on gameplay:
  Rain extinguishes fire: each tick, each fire block exposed to sky has 20% chance to be extinguished
    During thunder: 50% chance per tick (higher)
  Rain fills cauldrons: 1/20 chance per tick to increase water level by 1 (if exposed to sky)
  Rain speeds fishing: ~15% faster catch rate during rain (reduces wait time)
  Rain extinguishes burning entities: burning entities in rain take 1 less fire damage per tick
    (extends burn time but entity survives longer)
  Snow destroys torches: if snow layer forms on torch block, torch pops off as item
  Snow on farmland: if snow layer on farmland, farmland becomes hydrated (moisture increases)
  Thunder darkness: sky light capped at 10 during thunder (even daytime is dimmer)
  Weather affects mob spawning:
    - Rain: no effect on spawn rates directly
    - Thunder: extra 50% chance for hostile mob spawns (regional difficulty modifier)
  Weather affects slime spawning:
    - Swamp slime spawns: only during night AND moon phase affects rate
      - Full moon: maximum rate (increased by moon phase factor)
      - New moon: no slime spawns
  Weather affects nether portal zombie pigman spawns: more frequent during thunder? No, no effect

Fog mechanics by weather:
  Clear: fog distance = render distance (default 16 chunks, 256 blocks)
    Fog color: sky color at horizon
    Fog density: low (0.001-0.01)
  Rain: fog distance = 75% of render distance
    Fog color: grey-blue (#A0A0B0)
    Fog density: medium (0.02)
  Snow: fog distance = 50% of render distance
    Fog color: white-grey (#C0C0D0)
    Fog density: medium-high (0.03)
  Thunder: fog distance = 50% of render distance
    Fog color: dark grey (#404050)
    Fog density: high (0.04)
  Fog rendering: exponential fog, blended with sky color
    fogFactor = 1 - exp(-density × distance²)
    finalColor = lerp(sceneColor, fogColor, fogFactor)
```

### 9D.3 Sleep Mechanics
```
Requirements to initiate sleep:
  - Must be between tick 12542 and 23459 (nighttime, sky light ≤ 4)
    OR thunderstorm is active (regardless of time)
  - Player must be in Overworld dimension
  - Player must be within 2 blocks of the bed (any side, including diagonally)
  - No monsters within 8 blocks of bed (euclidean distance)
    Monsters: hostile mobs (zombies, skeletons, spiders, creepers, etc.)
    Neutral mobs that become hostile (enderman, spider at night, etc.) count as monsters
    Passive mobs (cows, pigs, sheep, etc.) do NOT prevent sleep
    Bats count as ambient and do NOT prevent sleep
  - Bed must not be obstructed (block above bed is free space)
  - Player must not be on fire, poisoned, or withering (messages vary)
  - If unobstructed: player gets in bed (lying down pose)

What sleeping does:
  - Immediately sets time to 0 ticks (dawn of next day)
  - Sets player's spawn point to the bed's location
    spawnX = bedX + 0.5, spawnY = bedY + 0.5, spawnZ = bedZ + 0.5
    Player spawns 1 block above bed on respawn
  - Clears weather if thunderstorm is active (sets rainTime = 0, thunderTime = 0)
  - Resets phantoms? No, phantoms are 1.13+
  - Player wakes up lying in bed: after ~0.5s the player stands up (10 ticks animation)
  - If right-clicking bed during day and it's not thundering:
    "You can only sleep at night and during thunderstorms" message
  - If monsters nearby: "You may not rest now; there are monsters nearby" message
  - If obstructed at wake time: "Your bed was obstructed" but time still passes

Bed obstruction checking:
  On attempting sleep: checks the air space above the bed (head + foot halves)
    - If blocked: "This bed is obstructed" message
    - Player does not enter sleeping pose
  On respawn: if bed block is missing or space above is obstructed:
    - Player respawns at original world spawn point
    - Message: "Your home bed was missing or obstructed"
    - Bed spawn point is cleared (next death goes to world spawn)

Bed explosion (Nether/End):
  Explosion properties:
    power = 5 (same as TNT, slightly less than charged creeper)
    Explosion type: fiery (creates fire in affected blocks)
    Damage: up to (5 × 2) = 10 damage at center (5 hearts), linear falloff
    Knockback: explosion knockback like TNT
  Block destruction: blocks with blast resistance < 5 × 1.3 = 6.5 are destroyed
    Obsidian (resistance 6000): survives
    Stone (resistance 6): may survive (borderline)
    Dirt (resistance 2.5): destroyed
    Planks (resistance 3): destroyed
    Average destruction radius: ~2-3 blocks
  Player effects:
    - Set on fire (lasting ~5 seconds, 100 ticks)
    - Launched upward (knockback from explosion center)
    - Damage: full explosion damage (no armor reduction for explosion? armor reduces)
  Trigger: right-clicking bed in any non-Overworld dimension
  Also triggers in Overworld at daytime? No, just "You can only sleep at night" message
  Bed explosion will not destroy the bed itself (bed drops as item)

Blocked spawn details:
  World spawn: original spawn point (x=0,z=0, or world spawn set by /setworldspawn)
  Bed spawn priority:
    1. Check bed block at last slept location
    2. If bed exists and clear space above: spawn there
    3. If bed missing or obstructed: fallback to world spawn
    4. If world spawn is obstructed: search within 10 blocks for safe space
    5. If still no safe space: search outward up to 100 blocks
  Message on fallback: "Your home bed was missing or obstructed"

Sleep animation:
  Player model lies flat on bed (rotation -90° on X axis for pose)
  Third-person: player visible lying on bed
  First-person: camera lowers to bed level (slight bob downward)
  Screen darkens slightly ("fade to sleep") over ~20 ticks
    Black overlay alpha increases from 0 to 1
  Wake animation: reverse, screen brightens over ~10 ticks
  If interrupted by monster proximity: player jolts upright, screen flash
```

---

## P9E — Difficulty & Game Modes

### 9E.1 Difficulty Settings
```
Four difficulty levels, set per-world and stored in level data:

Peaceful (0):
  - No hostile mobs spawn naturally (spawners still work but spawn passive variants)
  - No damage from hunger/starvation (food bar never depletes below 18)
  - Health regenerates at 1 HP per 0.5s regardless of food level
  - Hostile mobs despawn instantly if spawned via commands/spawn eggs
  - Spiders, endermen, zombie pigmen are neutral but never aggressive
  - Nether portal zombie pigman spawns: disabled
  - Witch huts: no witches spawn
  - Guardian spawning: disabled
  - Slimes: only tiny slimes spawn (non-hostile)
  - Silverfish: do not spawn from stone
  - Damage from non-mob sources: still applies (fall, lava, drowning, fire, etc.)
  - The Wither: can be summoned but does not attack (neutral)
  - Ender Dragon: still hostile (boss fight required to beat the game)
  - Zombie villagers: 0% conversion rate on villager death

Easy (1):
  - Hostile mob damage: × 0.5 (rounded down, minimum 1 damage)
    Example: zombie deals 3 damage → 1 damage on Easy
  - Cave spider poison duration: 3 seconds (instead of 7)
  - Wither effect duration: 5 seconds (from wither skeleton attack)
  - Zombie reinforcement chance: 0% (never spawn reinforcements)
  - Zombie door breaking: never (0% chance)
  - Starvation damage: 0.5 HP (0.25 hearts) when food ≤ 0, stops at 5 HP
  - Villager → zombie villager conversion rate: 0% (on Easy)
  - Fire damage: entity on fire takes 1 damage per tick (instead of normal rate)
  - Fall damage: unchanged (full fall damage calculation applies)

Normal (2):
  - Hostile mob damage: × 1.0 (full damage)
  - Cave spider poison duration: 7 seconds
  - Wither effect duration: 10 seconds
  - Zombie reinforcement chance: 0-10% (depends on regional difficulty)
  - Zombie door breaking: never (0% chance on Normal — only Hard)
  - Starvation damage: 0.5 HP when food ≤ 0, can kill (down to 0 HP)
  - Villager → zombie villager conversion rate: 50% on Normal
  - Fire damage: 1 damage per tick
  - Witch potion effectiveness: full duration

Hard (3):
  - Hostile mob damage: × 1.5 (rounded down, minimum 1 damage)
    Example: zombie deals 3 damage → 4 damage on Hard
  - Cave spider poison duration: 15 seconds
  - Wither effect duration: 40 seconds
  - Zombie reinforcement chance: 0-10% (depends on regional difficulty)
  - Zombie door breaking: 10% chance per tick when zombie attacks door (Hard only)
    Zombie can break wooden doors (not iron doors)
    Breaking: zombie hits door repeatedly until door breaks (takes several seconds)
    Door drops as item when broken
  - Husks: inflict Hunger effect for 10 seconds on hit (Hard only)
  - Strays: fire Slowness arrows instead of normal arrows (Hard only)
  - Starvation damage: 0.5 HP when food ≤ 0, can kill (like Normal)
  - Villager → zombie villager conversion rate: 100% on Hard
    Zombie villagers: spawned when villager dies to zombie
  - Vindicators: can break doors (like zombies, on Hard only)
  - Fire damage: 2 damage per tick (increased fire damage)

Regional difficulty formula (exact 1.12.2 calculation):
  chunk_inhabited_time = total time players have spent in this chunk (tracked per chunk in ticks)
  total_world_time = total play time of this world (in ticks)
  regional_difficulty = clamp((chunk_inhabited_time + total_world_time × 2) / 144000, 0, 1)
    Note: denominator = 144000 ticks (2 hours). Both terms measured in ticks.

  months_since_creation = total_world_time / 1728000 (1728000 ticks = 24 hours = 1 "month")
  clamp_target = 0.75 if months_since_creation > 1 else (months_since_creation × 0.75)
  clamped_difficulty = regional_difficulty × clamp_target + 0.5

  Moon phase modifier:
    phase 0 (full moon): final = clamped_difficulty + (rand(0,0.5) × (1 - clamped_difficulty))
    phase 4 (new moon): no modifier
    other phases: linear interpolation between full and new
  Final clamped_difficulty = clamp(final_value, 0, 1)

  Effects of clamped_difficulty (0.0 - 1.0):
    - Mob equipment chance: zombies/skeletons more likely to spawn with armor/weapons
      armor chance = clamped × 0.15 (Normal) or clamped × 1.0 (Hard)
      weapon chance = clamped × 0.05 (Normal) or clamped × 1.0 (Hard)
    - Mob enchantment level: existing equipment more likely to be enchanted
      enchant_chance = clamp(clamped × 2, 0, 1) × 0.5
      enchant_level = floor(5 + clamped × 18) (levels 5-23)
    - Zombie reinforcement: higher chance of calling reinforcements
      base_chance = 0.04 (Normal) or 0.06 (Hard)
      final_chance = base_chance × (1 + clamped)
    - Zombie door breaking: only in Hard, chance per tick = clamped × 0.1
    - Villager → zombie villager: conversion probability scales with clamped
      Easy: 0%, Normal: 50-100%, Hard: 100%
    - Baby zombie chance: ~5% base, unaffected by regional difficulty
    - Skeleton trap horses: no (that's 1.13+)
    - Mob health bonus: zombies get health bonus = floor(clamped × 4) extra HP
    - Mob damage bonus: some mobs get damage bonus on higher regional difficulty? No, damage is per-difficulty-level

Spawn equipment chances by difficulty:
  Easy:                  0% armor, 0% enchanted, 0% weapon
  Normal:                0-15% per armor slot, 0-50% enchanted (scales with clamped)
  Hard:                  0-100% per armor slot, 0-100% enchanted (scales with clamped)
  Zombie has 0-15% (Normal) or 0-100% (Hard) chance for each armor slot
  Skeleton has arrow type chance: 0-100% tipped arrows on Hard (scales with clamped)
  Zombie held item: 0-1% chance of holding iron shovel/sword (scales)

Difficulty lock:
  - /difficulty command: sets current difficulty immediately
  - /difficulty <peaceful|easy|normal|hard>
  - World option: "Lock Difficulty" in world settings (prevents changes)
  - When locked: /difficulty command returns error
  - Lock state stored in level.dat: DifficultyLocked byte (0|1)
  - Hardcore mode: always locked to Hard
```

### 9E.2 Game Modes
```
Four game modes, stored per-player:

Survival (0, game_type=0):
  - Core gameplay: health, hunger, oxygen, experience, inventory
  - Block breaking: takes time based on tool + block hardness
    (see P9G — Mining & Block Breaking for exact formula)
  - Can place and break blocks
  - Inventory: 41 slots total
    9 hotbar + 27 main inventory + 4 armor + 1 offhand
  - Damage: takes damage from all sources, can die
  - Natural mob spawning: enabled
  - Item drops: drop inventory on death (unless keepInventory=true)
  - XP: earned through mining, smelting, killing mobs
  - Food: hunger depletes over time, must eat
  - Abilities:
    fly = false, mayBuild = true, instabuild = false, invulnerable = false
  - Can use all GUIs (crafting table, chest, furnace, etc.)
  - Can trade with villagers
  - Can breed animals

Creative (1, game_type=1):
  - Infinite health: no damage taken (except void, /kill, and Creative-only sources)
  - Flight: double-tap space to toggle fly; hold space to ascend
    Fly speed: ~10 m/s (sprinting while flying: ~21.6 m/s)
    Flight mechanics: no gravity, no fall damage, precise control
  - Instant block breaking: 0 ticks (breaks instantly on click)
  - All blocks/items: accessible through creative inventory tabs (12 tabs)
  - No block drops: blocks mined do not drop items
  - Damage immunity: no damage from mobs, fall, fire, drowning, etc.
    Exceptions: void damage (below y=0), /kill, drowning in Creative? no
    Creative players CAN drown if underwater long enough? No, infinite air.
    Creative players CAN take damage from slipping on ice? No.
  - Mob aggro: mobs ignore creative players (no attack)
  - Abilities:
    fly = true, mayBuild = true, instabuild = true, invulnerable = true
  - Saved Toolbars (1.12 NEW): C+1-9 save, X+1-9 load
    Persist per-player across sessions
  - Can still interact with GUIs, levers, doors, etc.
  - Can still trade with villagers
  - Pick block (middle click): copies block to hotbar with exact NBT data
  - Spectator mode NOT accessible from F3+N (that's 1.15+? Actually F3+N is 1.8+...)
    Actually F3+N switches between creative and spectator in 1.12.2.

Adventure (2, game_type=2):
  - Block breaking: only blocks with matching CanDestroy NBT tag
    CanDestroy: list of block IDs player is allowed to break
    Tool must have CanDestroy tag: e.g., /give @s diamond_pickaxe{CanDestroy:["minecraft:stone"]}
  - Block placing: only blocks with matching CanPlaceOn NBT tag
    CanPlaceOn: list of block IDs player is allowed to place on
  - Otherwise identical to Survival: hunger, health, damage, etc.
  - Used for adventure maps and custom game experiences
  - Default interaction (always allowed): chests, levers, buttons, doors,
    trapdoors, crafting tables, furnaces, enchanting tables, anvils,
    brewing stands, beds, jukeboxes, note blocks, redstone components
  - Cannot use /gamemode to change (unless operator)
  - Can still eat food, drink potions, use tools with correct tags
  - Can attack mobs (PvE is allowed)
  - Can trade with villagers

Spectator (3, game_type=3):
  - Flying: enabled by default, no flight toggle needed
  - No-clip: pass through all blocks and entities (no collision)
    Movement: fly through blocks at will, camera passes through terrain
  - No interaction: cannot place/break blocks, open GUIs, use items
    Cannot attack mobs, cannot trade with villagers
    Cannot take damage from any source (including void)
    Cannot be affected by status effects
    Cannot trigger pressure plates, tripwire, etc.
  - Entity perspective: left-click entity to see from its point of view
    Camera: switches to entity's eyes, follows entity movement
    Exit entity view: press sneak (Shift) or scroll to self
  - Invisibility: other players cannot see spectator (invisible, no nametag)
    Mobs: ignore spectator players (no aggro, no detection)
  - HUD: hidden entirely (no crosshair, hotbar, health, hunger, XP)
    F3 debug: still visible (toggle with F3)
    Chat: accessible (press T)
  - Speed control: scroll wheel to change fly speed
    Speed levels: 1-10 (default 4)
    Very fast: level 10 can traverse entire map in seconds
  - Teleport to entities: click on entity in spectator to teleport to them
    Entity list: press F3+N to cycle? No, that's game mode switching.
  - Chunk loading: spectatorsGenerateChunks gamerule (default true)
    If false: spectator sees only already-loaded chunks
  - Block viewing: can see through blocks (no-clip gives full visibility)
    However: block overlays, outlines, and hitboxes are still rendered
  - Advancements: spectators cannot earn advancements
  - Recipe book: inaccessible in spectator
  - F3 menu: shows "Spectator" as gamemode
  - Can still see the world normally (sky, weather, lighting all render)
  - Left-click in spectator: if not on entity, does nothing

Gamemode commands:
  /gamemode <survival|creative|adventure|spectator> [player]
  /defaultgamemode <survival|creative|adventure|spectator>
  F3+F4: opens game mode switcher? No, that's 1.13+
  F3+N: cycles between creative and spectator (1.9+)
  /gamemode 0 = survival, /gamemode 1 = creative, /gamemode 2 = adventure, /gamemode 3 = spectator
```

### 9E.3 Death & Respawn
```
Death causes:
  - Health reaches 0 (lethal damage from any source)
  - Void damage (below y=0 or above nether ceiling)
  - /kill command (instant death, no damage source)
  - Starvation (food bar depleted and health drains to 0)

On death:
  - Screen fades immediately to red tint (blood overlay)
  - Fade duration: ~1 second (20 ticks) to full red
  - Transition to "You Died!" screen
  - Death message broadcast: "<Player> was slain by Zombie" (localized)
    Messages vary by damage source:
    - "<Player> was slain by <Mob>"
    - "<Player> was shot by <Mob>"
    - "<Player> fell from a high place"
    - "<Player> drowned"
    - "<Player> burned to death"
    - "<Player> blew up"
    - "<Player> fell out of the world"
    - "<Player> was pricked to death"
    - "<Player> starved to death"
    - "<Player> hit the ground too hard"
    - "<Player> suffocated in a wall"
    - "<Player> died" (generic)
    - "<Player> tried to swim in lava"
    - "<Player> was fireballed by <Mob>"
    - "<Player> withered away"
    - "<Player> was toasted in dragon breath"
  - Respawn screen options:
    - "Respawn" button: respawns at world spawn or bed
    - "Title Screen" button: quits to main menu (disconnects)
  - Score penalty: decrease score by level × 7
    Score = total XP points (not levels)
    Points dropped: level × 7 worth of XP orbs (max 100 orbs)
    XP orbs drop at death location (scatter around)
  - XP orbs on ground:
    Up to 100 orbs spawn at death location
    XP amount: all XP points dropped (capped at 100 orbs worth)
    Each orb holds varying XP amounts based on orb size:
      Small orb (green/yellow): 2-8 XP (most common)
      Medium orb (yellow): 8-16 XP
      Large orb (yellow/blue): 16-32 XP
    Pickup: player regains XP by walking near orbs
    Despawn: 6000 ticks (5 minutes)
  - Item drop behavior:
    All inventory items drop at death location (unless keepInventory)
    Items scatter in random directions on ground
    Despawn: 6000 ticks (5 minutes) from time of death
    Armor: each piece drops separately (not equipped on corpse)
    Offhand item drops
  - Game mode: stays in Survival (doesn't change to Spectator)
    Hardcore: changes to Spectator on death

Keep Inventory gamerule:
  /gamerule keepInventory true
  - Player retains all items on death (inventory not dropped)
  - Player retains all XP/levels (no XP orbs dropped)
  - Items in inventory remain exactly as they were before death
  - No "scatter" effect (no items dropped)
  - Keep inventory applies to all game modes (including Hardcore)
  Keep Levels gamerule: does NOT exist in 1.12 (added in later versions)
  Keep inventory is a world gamerule, not per-player

Respawn mechanics:
  - Respawn location priority:
    1. Last-slept bed (if bed still exists and space is clear)
    2. World spawn point (set by /setworldspawn or default (0,0))
    3. Search: if spawn point obstructed, search within 10 blocks for safe space
    4. Expand: if still obstructed, expand search up to 100 blocks
    5. Fallback: if all obstructed, force-place player at spawn (may suffocate)
  - Respawn effect: player appears with brief "spawn" animation (fade in)
    invulnerability: none on respawn (immediately vulnerable)
  - Health on respawn: full 20 HP
  - Hunger on respawn: full 20 hunger
  - Inventory: empty (unless keepInventory true)
  - Position: at bed or world spawn
  - Camera: resets to first-person if was in third-person? No, preserves setting
  - Respawn screen interaction: click respawn immediately (no delay)

Hardcore mode:
  - Selected on world creation (toggle in world settings)
  - Always Hard difficulty, permanently locked (cannot change)
  - On death:
    - "Game Over!" screen instead of "You Died!"
    - Options: "Delete World" and "Spectate World"
    - Player set to Spectator mode on death
    - Can explore world but cannot interact
    - Can press Esc → "Delete World" to delete the save
    - Can press Esc → "Title Screen" to quit
  - No respawn button (hardcore = permanent death)
  - Spectating: can fly around and view world but cannot interact
  - Hardcore worlds: heart icons on health bar have X marks (unique design)
  - Death message prefix: none (same messages as normal, but red color)
  - Hardcore mode: can still be opened to LAN with cheats to change game mode
    (allows recovering a hardcore world by enabling cheats)
  - Player list: hardcore players shown in red in tab list

Death statistics:
  - Stats tracked per player:
    deaths: number of times player died
    deathsBy<cause>: tracked per damage type
    timeSinceDeath: ticks since last death
  - Statistics available via /scoreboard stats command
  - Displayed on death screen: "Score: <level>"
```

---

## P9F — Villager Trading, Breeding & Fishing

### 9F.1 Villager Professions & Trade Tiers
```
Each villager has up to 5 career levels (Novice → Apprentice → Journeyman → Expert → Master)
New trades unlock when enough XP is earned from trading with that villager.

Career level thresholds (XP earned from trading):
  Novice:    0 XP
  Apprentice: 10 XP
  Journeyman: 70 XP
  Expert:    150 XP
  Master:    250 XP

Trade availability:
  - Each trade has a max uses before it locks (2-12 uses)
  - Locked trade: villager stays in "unhappy" state, needs restock
  - Restock: villager gets access to work station (1.14+, in 1.12.2 no restock)
  - In 1.12.2: trades lock permanently after their max uses
  - Villagers start with 2-4 random trades from their career level
  - Each career level adds 2 new trades

XP per trade:
  - Player earns 1-6 XP per trade
  - First trade with a villager: extra XP (bonus)
  - Villager XP: trading XP accumulates, unlocking new tiers at thresholds above

Profession IDs (career ID):
  0: Farmer   1: Librarian   2: Cleric   3: Blacksmith (Armorer/Tool/Weapon)
  4: Butcher   5: Fisherman (same as 4 in some sources... actually 5=butcher, 4=fisher)
  Actually in 1.12.2 professions by metadata:
    0: Farmer    1: Librarian   2: Cleric   3: Blacksmith
    4: Butcher   5: Fisherman   6: Leatherworker (added later? Leatherworker was always there?)
  Wait — 1.12.2 has 5 profession metadata values from 0-4... Villager metadata=profession
  Farmer(0), Librarian(1), Cleric(2), Blacksmith(3), Butcher(4)
  Fisherman and Leatherworker are part of Butcher? Let me check...
  In 1.12.2: Profession 0=Farmer, 1=Librarian, 2=Cleric, 3=Blacksmith, 4=Butcher
    But profession subtypes exist via career:
    Farmer has subtypes: Farmer, Fisherman, Shepherd, Fletcher
    Actually in 1.12.2 the system uses career levels per profession:
    Profession 0 (Farmer): Farmer, Fisherman, Shepherd, Fletcher (careers 0-3)
    Profession 1 (Librarian): Librarian, Cartographer (careers 0-1)
    Profession 2 (Cleric): Cleric (career 0)
    Profession 3 (Blacksmith): Armorer, WeaponSmith, ToolSmith (careers 0-2)
    Profession 4 (Butcher): Butcher, Leatherworker (careers 0-1)
    But in vanilla 1.12.2 without mods, villagers use a SINGLE profession value.
    The career naming system was added in later versions.
  
  Let's simplify: the exact career/subtype system in 1.12.2 uses villager metadata:
    profession (0-4) + career (internal)
    The villager's robe color indicates the trade category:
    Brown: Farmer-type trades (grain, crops, food)
    White: Librarian-type (books, paper, enchants)
    Purple: Cleric-type (potions, nether items)
    Black apron: Blacksmith-type (tools, armor, weapons)
    White apron: Butcher-type (meat, leather)
    Brown hat: Fisherman-type (fish, string)
    
    Total distinct "trade types": Farmer, Fisherman, Shepherd, Fletcher, Librarian, 
    Cartographer, Cleric, Armorer, Weapon Smith, Tool Smith, Butcher, Leatherworker
```

### 9F.2 Exact Trade Tables (1.12.2)
```
Each trade shows: input item → output item, with counts and max uses.

FARMER (brown robe, profession=0)
  Tier  | Buying (input → emerald)     | Selling (emerald → output)
  ─────────────────────────────────────────────────────────────────────
  Novice    wheat x18-22 → emerald         emerald → bread x2-4
  Novice    potato x15-19 → emerald        emerald → bread x2-4
  Novice    carrot x15-19 → emerald
             
  Apprentice  pumpkin x8-13 → emerald       emerald → apple x2-4
              
  Journeyman   melon x3-6 → emerald         emerald → cake x1
              
  Expert      beetroot x6-10 → emerald      emerald → cookies x4-7
              
  Master                                    emerald → golden apple x1 (rare, 1.12.2... wait golden apple is rare? 
                                            Actually some sources say master farmers sell golden apples, 
                                            but that might be modded. In vanilla 1.12.2, master farmers
                                            may sell golden carrots or nothing special. Let's keep it
                                            to known trades.)

  Note: Each trade is randomly selected for the villager. A villager may have
  2-4 of the available trades, not all at once.

FISHERMAN (brown hat, subtype of farmer or butcher)
  Tier  | Buying                             | Selling
  ─────────────────────────────────────────────────────────────────────
  Novice    string x15-20 → emerald           emerald → cooked cod x4-6
  Novice    coal x16-24 → emerald             emerald → cooked cod x4-6
             
  Apprentice                                emerald → cooked salmon x1
             
  Journeyman   raw cod x12-16 → emerald? No fisherman buys raw fish too
              Actually: fisherman buys raw cod x12-16 → emerald
              emerald + raw salmon → cooked salmon
              Also sells: fishing rod (enchanted?) plain fishing rod
              
  Journeyman  raw cod x12-16 → emerald
              
  Expert      raw salmon x12-16 → emerald    emerald → fishing rod x1
              
  Master                                    emerald + raw cod → cooked cod x6-10?

  This is getting convoluted. Let me simplify to the standard trades:

  Fisherman known trades:
    Buy: string(15-20)→1em, coal(16-24)→1em, raw_cod(12-16)→1em, raw_salmon(12-16)→1em
    Sell: 1em→cooked_cod(4-6), 1em→cooked_salmon(1), 1em→fishing_rod(1)

SHEPHERD (brown robe subtype)
  Buy: wool(12-22)→1em
  Sell: 1em→shears(1), 1em→wool(1-2), emerald + dye → dyed wool
  
FLETCHER (brown robe subtype)
  Buy: string(15-20)→1em, feather(14-20)→1em
  Sell: 1em→flint(4-7), 1em→bow(1), 1em→arrow(5-13), emerald + flint → arrow(100-200)

LIBRARIAN (white robe, profession=1)
  Tier | Buying                             | Selling
  ─────────────────────────────────────────────────────────────────────
  Novice   paper x24-36 → emerald            emerald → bookshelf x1
  Novice   paper x24-36 → emerald            emerald → glass block x6-9
  Novice                                     1em → enchanted book (random)
             
  Apprentice   book x8-10 → emerald          emerald + lapis → name tag? No, 
              Actually apprentice librarian:
              book x8-10 → emerald           1em + book&quill → enchanting table? No...
  Apprentice   book x8-10 → emerald          emerald → clock x1
              
  Journeyman  book & quill x2? No            emerald → compass x1
  Journeyman  bookshelf x?                   emerald → name tag x1
              
  Expert                                      emerald → enchantment table? No...
  Expert                                     Actually all librarian special: enchant books
  
  Known librarian trades:
    Buy: paper(24-36)→1em, book(8-10)→1em, written_book(2)→1em
    Buy: bookshelf(8-10)→1em
    Sell: 1em→bookshelf(1), 1em→glass(6-9), 1em→clock(1), 1em→compass(1)
    Sell: emerald + paper → name tag(1)? No, name tag is sold for emeralds: 20-22em→nametag
    Actually: name tag costs 20-22 emeralds from librarian (very expensive)
    Sell: emerald → enchanted book (random enchant, random level)
    Sell: 20-22em → name tag (expert/master)
    Sell: 2-4em → lapis lazuli... no that's cleric
    Sell: 1em → book (basic)
  
  Enchanted book pricing:
    Level 1: 5-19 emeralds
    Level 2: 8-24 emeralds  
    Level 3: 11-29 emeralds
    Treasure enchants (mending, frost walker): much more expensive

CLERIC (purple robe, profession=2)
  Tier | Buying                             | Selling
  ─────────────────────────────────────────────────────────────────────
  Novice   rotten flesh x38-44 → emerald     emerald → redstone x2-5
  Novice   rotten flesh x38 → emerald        emerald → lapis lazuli x1-2
             
  Apprentice   gold ingot x7-9 → emerald     emerald → glowstone x1-3
  Apprentice   gold ingot x8-10 → emerald    (actually same slot different price)
              
  Journeyman   spider eye x6-8 → emerald     emerald → ender pearl x1-2
              
  Expert      gunpowder x3-6 → emerald       emerald → bottle o' enchanting x1-3
              
  Master                                     emerald → bottle o' enchanting (more)
              
  Known cleric trades:
    Buy: rotten_flesh(38-44)→1em, gold_ingot(7-9)→1em, spider_eye(6-8)→1em
    Buy: gunpowder(3-6)→1em
    Sell: 1em→redstone(2-5), 1em→lapis(1-2), 1em→glowstone(1-3), 
          2-4em→ender_pearl(1), 3-11em→bottle_o_enchanting(1)

BLACKSMITH (apron, profession=3, career subtypes: Armorer/WeaponSmith/ToolSmith)

  ARMORER (black apron, subtype 0):
    Tier | Buying                             | Selling
    ─────────────────────────────────────────────────────────────────────
    Novice   coal x15-18 → emerald            emerald → iron helmet x1
    Novice   coal x15-18 → emerald            emerald → iron boots x1
                
    Apprentice   iron ingot x7-9 → emerald    emerald → iron leggings x1
    Apprentice   iron ingot x7-9 → emerald    emerald → iron chestplate x1
                
    Journeyman   diamond x1 → emerald         emerald → diamond boots x1
    Journeyman   diamond x1 → emerald+         emerald → diamond helmet x1
                
    Expert                                      emerald + diamond → diamond leggings x1
                
    Master                                      emerald + diamond → diamond chestplate x1

  WEAPON SMITH (black apron, subtype 1):
    Tier | Buying                             | Selling
    ─────────────────────────────────────────────────────────────────────
    Novice   coal x14-16 → emerald            emerald → iron sword x1
    Novice   coal x14-16 → emerald            emerald → iron axe x1
                
    Apprentice   iron ingot x8-10 → emerald   emerald → diamond? No, 
                Actually: iron ingot x7-9 → emerald
                Selling: 1em→iron sword (with random enchants)
                
    Journeyman                                emerald → iron pickaxe
                
    Expert      diamond x1 → emerald           emerald → diamond sword
                
    Master                                     emerald → diamond axe

  TOOL SMITH (black apron, subtype 2):
    Tier | Buying                             | Selling
    ─────────────────────────────────────────────────────────────────────
    Novice   coal x16-24 → emerald            emerald → iron shovel x1
    Novice   coal x16-24 → emerald            emerald → iron pickaxe x1
                
    Apprentice   iron ingot x8-10 → emerald   emerald → iron hoe x1
                
    Journeyman                                emerald → iron axe... wait, toolsmith
                
    Expert      diamond x1 → emerald           emerald → diamond pickaxe
                
    Master                                     emerald → diamond shovel

BUTCHER (white apron, profession=4, career: Butcher/Leatherworker)

  BUTCHER (white apron, career 0):
    Tier | Buying                             | Selling
    ─────────────────────────────────────────────────────────────────────
    Novice   raw porkchop x16-24 → emerald    emerald → cooked porkchop x5-9
    Novice   raw chicken x16-24 → emerald     emerald → cooked chicken x5-9
                
    Apprentice   coal x14-18 → emerald        emerald → cooked beef x6-10
                
    Journeyman   raw beef x10-16 → emerald    emerald → cooked mutton? Actually...
                Butcher doesn't buy mutton, that's weird.
    
    Known butcher trades:
      Buy: raw_porkchop(16-24)→1em, raw_chicken(16-24)→1em 
      Buy: coal(14-18)→1em, raw_beef(10-16)→1em
      Sell: 1em→cooked_porkchop(5-9), 1em→cooked_chicken(5-9), 
            1em→cooked_beef(6-10)

  LEATHERWORKER (brown hat... actually white apron but career=1):
    Tier | Buying                             | Selling
    ─────────────────────────────────────────────────────────────────────
    Novice   leather x9-12 → emerald          emerald → leather pants x1
    Novice   leather x9-12 → emerald          emerald → leather tunic x1
                
    Apprentice   leather x9-12 → emerald      emerald → leather helmet x1
                
    Journeyman                                emerald → leather boots x1
                
    Expert      leather x5-8 → emerald (reduced) 
    Actually: leatherworker buys leather for less at higher tiers
                
    Known leatherworker trades:
      Buy: leather(9-12)→1em (later tiers: 5-8→1em, 3-6→1em)
      Sell: 1em→leather_helmet(1), 1em→leather_chestplate(1),
            1em→leather_leggings(1), 1em→leather_boots(1)
      Sell: 3-4em→saddle(1) ... wait, does leatherworker sell saddle in 1.12.2?
            Some sources say yes, some say no (master leatherworker)
            Actually saddle is only from dungeon loot or fishing, not villager trading
      Sell: 10-12em→name tag(1) ... again, librarian sells name tag, not leatherworker
      
    Keep it simple: leatherworker sells leather armor only
```

### 9F.3 Trading GUI & Economy
```
Trading GUI layout:
  - Left side: scrollable trade list showing available trades
    Each trade shows: input slot(s) + output slot with price
    Button: click to select a trade
    Lock icon: trade has been fully used (max uses exhausted)
  - Right side: selected trade detail
    2 input slots (buy items)
    1 output slot (sell item)
    Arrow indicating trade direction
    "Max uses" counter (e.g. "Uses: 3/12")
    XP bar showing career progress
  - Bottom: villager name + career title (e.g. "Master Librarian")

Trade counts:
  - First trade in a tier: max 12 uses
  - Subsequent trades in same tier: max 2-6 uses
  - Each use: outputs 1 item (or item count shown)
  
Pricing adjustments:
  - Cured zombie villager: permanent 20-50% discount on all trades
  - Popularity system: trading increases popularity → better prices
  - Negative popularity (hurting villager, killing golems): prices increase

Villager inventory:
  - Villagers don't have physical inventory
  - Trades are generated on villager spawn (or first interaction)
  - Trades are tied to villager's UUID
  - Trading does NOT deplete villager inventory (magical infinite supply)
```

### 9F.4 Breeding
```
Breeding mechanics:
  - Two animals of same type + both fed their breeding food → enter love mode
  - Love mode: hearts particles; after 0.5-2.5s, baby spawns near parents
  - Baby: 20% of adult size; grows up over 20 minutes (24000 ticks)
  - Growth can be accelerated with food (each feeding reduces growth time by 10%)
  - Cooldown: 5 minutes (6000 ticks) before parents can breed again
  - Baby inherits random variant/skin from parents

Breeding food per animal:
  Sheep:  wheat               Cow:    wheat              Chicken: seeds (wheat, melon, pumpkin, beetroot)
  Pig:    carrot/potato/beetroot  Rabbit: dandelion/carrot/golden carrot
  Horse:  golden apple/golden carrot  Llama: hay bale
  Wolf:   any meat (raw/cooked, not fish)  Cat/ocelot: raw cod/salmon
  Mooshroom: wheat           Villager: 12 carrot/potato/beetroot (or 3 bread)

Breeding rewards: 1-7 XP orbs per successful breeding

Villager breeding:
  - Requires: two villagers, 12 carrots/potatoes/beetroot OR 3 bread
  - Requires: at least 3 doors in village (houses)
  - Villagers must "willing" (inventory full from food)
  - Baby villager: inherits profession of parent (random in 1.12.2)
```

### 9F.5 Fishing
```
Fishing mechanics:
  - Cast line with fishing rod (right-click on water)
  - Bobber floats on water surface; waits for bite
  - Bite time: 5-45 seconds (reduced by Lure enchantment)
  - On bite: bobber splashes, right-click to reel in
  - Three loot categories: fish, treasure, junk

Fish (85% chance):
  Raw Cod (60%), Raw Salmon (25%), Tropical Fish (2%), Pufferfish (13%)

Junk (10% chance):
  Stick (20%), Bowl (10%), String (5%), Bone (5%), Ink Sac (1%),
  Lily Pad (2%), Tripwire Hook (2%), Leather (4%), Rotten Flesh (5%),
  Water Bottle (10%), Bone Meal (2%), Fishing Rod (damaged, 2%), 
  Leather Boots (damaged, 10%), Leather (4%)

Treasure (5% chance):
  Bow (enchant random, 16.6%), Enchanted Book (random, 16.6%), 
  Fishing Rod (enchant random, 16.6%), Name Tag (16.6%), Saddle (16.6%),
  Water Lily (1.13+, skip)

Luck of the Sea enchantment:
  - Reduces junk chance by 2% per level
  - Increases treasure chance by 1% per level
  - At Luck III: fish 66.5%, junk 17.5%, treasure 16%

Lure enchantment:
  - Reduces wait time by 5 seconds per level
  - Lure III: minimum wait is 1 second (from 5s min)

Implementation:
  - Bobber entity: float on water, detect when underwater block changes
  - Bite detection: random time based on Lure level
  - Particle effect: bubble particles rising from bobber when fish is about to bite
  - Sound: splash sound on bite, reel sound when reeling
  - Reel handling: right-click removes bobber, generates loot
  - Loot generation: weighted random from categories, enchant items based on player's rod
```

---

## P9G — Mining & Block Breaking Mechanics

### 9G.1 Breaking Speed Formula (Exact 1.12.2)
```
Core formula:
  speed = base_speed / block_hardness  [in blocks/second]
  base_speed = 1.0 (hand or wrong tool, no tool bonus)
  If correct tool: base_speed = tool_speed + efficiency_bonus
  If wrong tool:  speed *= 0.2 (or base_speed << see below)
  break_time_ticks = ceil(30 / speed)  [each tick = 0.05s]

Tool speed values (blocks/second, higher = faster):
  Hand:         1.0
  Wood:         2.0
  Stone:        4.0
  Iron:         6.0
  Diamond:      8.0
  Gold:         12.0
  Shears:       5.0 (leaves), 15.0 (wool)
  Sword:        1.5 (for cobwebs, leaves — actually sword breaks faster)
  Axe:          3.0 (for planks, logs)  — included as "correct tool" for wood

Efficiency enchantment bonus:
  efficiency_bonus = enchant_level² + 1  (if tool is correct type)
  Level I:   2
  Level II:  5
  Level III: 10
  Level IV:  17
  Level V:   26

Wrong tool penalty:
  If using an incorrect tool type (e.g. pickaxe on wood):
    - Add 3.33× to break time (multiply by 3.33)
    - OR: multiply speed by 0.2 (equivalent: divide speed by 5)
  The penalty applies BEFORE rounding
  If using correct tool type but insufficient tier (e.g. wood pick on iron ore):
    - No drop penalty (block drops nothing)
    - Break time is still calculated normally (no extra slow)
    - Block will eventually break but yield no item

Minimum break time:
  - Cannot be less than 1 tick (0.05s)
  - Instant break: if speed ≥ 30 (break in 1 tick = effectively instant)
  - Blocks with hardness ≤ 0.033 break instantly with hand

Mining fatigue (multiplier):
  Level I:   0.3×  (speed × 0.3)
  Level II:  0.09× (speed × 0.09)
  Level III: 0.027× (speed × 0.027)
  Level IV:  0.0081× (speed × 0.0081)

Haste (multiplier):
  Level I:   1.4×  (speed × 1.4)
  Level II:  1.9×  (speed × 1.9)

In water (no Aqua Affinity): speed × 0.2 (÷5)
On ground: normal speed
On ladder/vine: speed depends on climbing

Breaking animation stages:
  - Block visually cracks (overlay texture) at 10% intervals
  - Stage 0: 0% (no damage) → Stage 9: 90% (about to break)
  - Each stage is (break_time / 10) ticks of mining
  - Sound: periodic "tock" sounds while mining (per block type)
  - Particles: block-breaking particles appear at mining location
```

### 9G.2 Tool Tier Requirements (What Breaks What)
```
Block                                Correct Tool    Min Tier      Drops Item?
───────────────────────────────────────────────────────────────────────────────
Stone / Cobblestone / Bricks         Pickaxe         Wood          ✓
Iron Ore / Gold Ore                  Pickaxe         Stone         ✓ (drops ore)
Diamond Ore / Emerald Ore            Pickaxe         Iron          ✓
Redstone Ore / Lapis Ore             Pickaxe         Iron          ✓
Obsidian                             Pickaxe         Diamond       ✓
Nether Quartz Ore                    Pickaxe         Wood          ✓
End Stone                            Pickaxe         Wood          ✓
All ores (general)                   Pickaxe         Varies        ✓

Wood / Logs / Planks                 Axe             None          ✓ (drops itself)
Wooden Door / Trapdoor / Fence       Axe             None          ✓
Bookshelf / Crafting Table           Axe             None          ✓
Note Block / Jukebox                 Axe             None          ✓
Pumpkin / Jack o'Lantern             Axe             None          ✓

Dirt / Grass / Mycelium              Shovel          None          ✓ (drops dirt)
Sand / Gravel / Clay                 Shovel          None          ✓
Snow / Snow Block                    Shovel          None          ✓
Soul Sand                            Shovel          None          ✓

Wool                                 Shears          None          ✓ (drops wool block)
                                     Any (hand)      —             drops 1 string
Leaves                               Shears          None          ✓ (drops leaf block)
                                     Any (hand)      —             drops stick, sapling
Cobweb                               Shears          None          ✓ (drops cobweb)
                                     Sword           None          ✓ (drops string)

Glass / Glass Pane / Sea Lantern     None            None          ✗ (drops nothing)
                                     Silk Touch      —             ✓ (drops block)
Ice                                  None            None          ✗ (drops nothing in 1.12.2... wait ice drops nothing)
                                     Silk Touch      —             ✓ (drops packed ice? No — drops ice block)

Crops (wheat, carrots, potatoes)     None            None          ✓ (drops crop + seeds)
Nether Wart                          None            None          ✓ (drops wart)
Cactus / Sugar Cane                  None            None          ✓ (drops itself at bottom)
Flowers / Grass / Fern               None            None          ✓ (see shears for block drops)
Torch / Redstone / Rails             None            None          ✓ (drops itself instantly)
Button / Lever / Pressure Plate      None            None          ✓ (drops itself)
Bed                                  None            None          ✓ (drops bed item)
Door / Trapdoor                      None            None          ✓ (drops door item)
Ladder                               None            None          ✓ (drops ladder)
Sign                                 None            None          ✓ (drops sign)
Painting / Item Frame                None            None          ✓ (drops itself)
Furnace / Chest / Crafting Table     Pickaxe?        None          ✓ (drops itself)
                                     Actually furnace drops: pickaxe → furnace, hand → nothing
                                     Chest drops: axe? No, chest drops with any tool... 
                                     Actually in 1.12.2: chest drops itself regardless of tool.
                                     Furnace drops: pickaxe needed. Without pickaxe: nothing.
                                     Dispenser/Dropper: pickaxe needed.
Brewing Stand                        Pickaxe         None          ✓
Enchanting Table                     Pickaxe         None          ✓ (drops itself)
Anvil                                Pickaxe         None          ✓ (drops itself, damaged)
Ender Chest                          Pickaxe         Wood? No, pickaxe with silk touch = drops 8 obsidian
                                     (ender chest itself requires silk touch)
Monster Egg (Silverfish)             None            None          ✗ (silverfish spawns on break)
Spawner                              Pickaxe         Iron          ✗ (drops XP, no spawner item)

Blocks that always drop nothing:
  Bedrock, End Portal, End Portal Frame, End Gateway, Portal (nether/end),
  Fire, Piston Head, Piston Extension, Moving Piston

Key rule: if incorrect tool type is used, the block may still break but drops nothing.
  - Wood pickaxe on stone → breaks but drops nothing
  - Hand on furnace → breaks but drops nothing  
  - Proper tool = proper drops + faster speed
```

### 9G.3 Block Hardness Table (All 1.12.2 Blocks, Sorted by Category)
```
STONE & MINERALS:
  Block                       Hardness   Correct Tool  Tier Req    Notes
  ─────────────────────────────────────────────────────────────────────────
  Bedrock                      -1 (∞)    —             —           Unbreakable
  Stone                        1.5       Pickaxe       Wood
  Granite/Diorite/Andesite     1.5       Pickaxe       Wood
  Cobblestone                  2.0       Pickaxe       Wood
  Stone Bricks                 1.5       Pickaxe       Wood
  Mossy Stone Bricks           1.5       Pickaxe       Wood
  Cracked Stone Bricks         1.5       Pickaxe       Wood
  Chiseled Stone Bricks        1.5       Pickaxe       Wood
  Gravel                       0.6       Shovel        None
  Sand                         0.5       Shovel        None
  Red Sand                     0.5       Shovel        None
  Sandstone                    0.8       Pickaxe       Wood
  Red Sandstone                0.8       Pickaxe       Wood
  Chiseled Sandstone           0.8       Pickaxe       Wood
  Cut Sandstone                0.8       Pickaxe       Wood
  Smooth Sandstone             0.8       Pickaxe       Wood
  End Stone                    3.0       Pickaxe       Wood
  End Stone Bricks             3.0       Pickaxe       Wood
  Nether Quartz Ore            3.0       Pickaxe       Wood
  Netherrack                   0.4       Pickaxe       Wood
  Nether Brick                 2.0       Pickaxe       Wood
  Nether Brick Fence           2.0       Pickaxe       Wood
  Nether Brick Stairs          2.0       Pickaxe       Wood
  Magma Block                  0.5       Pickaxe       Wood  (1.13+)
  Obsidian                     50.0      Pickaxe       Diamond

ORES:
  Coal Ore                     3.0       Pickaxe       Wood
  Iron Ore                     3.0       Pickaxe       Stone
  Gold Ore                     3.0       Pickaxe       Stone
  Diamond Ore                  3.0       Pickaxe       Iron
  Emerald Ore                  3.0       Pickaxe       Iron
  Lapis Ore                    3.0       Pickaxe       Stone
  Redstone Ore                 3.0       Pickaxe       Iron
  Nether Quartz Ore            3.0       Pickaxe       Wood

WOOD & PLANTS:
  Log (any)                    2.0       Axe           None
  Stripped Log                 2.0       Axe           None
  Planks (any)                 2.0       Axe           None
  Wood Stairs                  2.0       Axe           None
  Wood Slab                    2.0       Axe           None
  Fence/Gate                   2.0       Axe           None
  Door/Trapdoor                3.0       Axe           None
  Sign                         1.0       Axe           None
  Bookshelf                    1.5       Axe           None
  Crafting Table               2.5       Axe           None
  Chest                        2.5       Axe           None
  Trapped Chest                2.5       Axe           None
  Note Block                   0.8       Axe           None
  Jukebox                      2.0       Axe           None
  Leaves                       0.2       Shears/Axe    None
  Sapling                      0.0       None          None    Instant break
  Tall Grass / Fern            0.0       None/Shears   None    Instant break
  Dead Bush                    0.0       None/Shears   None    Instant break
  Vine                         0.2       Shears        None
  Lily Pad                     0.0       None          None    Instant break
  Cactus                       0.4       None          None
  Sugar Cane                   0.0       None          None    Instant break
  Vines                        0.2       Shears/Axe    None
  Hay Bale                     0.5       Hoe           None    (1.13+)

EARTH & GROUND:
  Dirt                         0.5       Shovel        None
  Grass Block                  0.6       Shovel        None
  Grass Path                   0.65      Shovel        None
  Mycelium                     0.6       Shovel        None
  Podzol                       0.5       Shovel        None
  Coarse Dirt                  0.5       Shovel        None
  Farmland                     0.6       Shovel        None
  Clay                         0.6       Shovel        None
  Snow (layer)                 0.1       Shovel        None    Instant break
  Snow Block                   0.2       Shovel        None
  Ice                          0.5       Pickaxe       None    Drops nothing unless silk touch
  Packed Ice                   0.5       Pickaxe       None
  Frosted Ice                  0.5       None          None

SAND & GRAVEL:
  Sand                         0.5       Shovel        None
  Red Sand                     0.5       Shovel        None
  Gravel                       0.6       Shovel        None
  Concrete Powder              0.5       Shovel        None    (falls like sand)

STONE BRICKS / DECORATIVE:
  Brick Block                  2.0       Pickaxe       Wood
  Brick Stairs                 2.0       Pickaxe       Wood
  Stone Brick Stairs           1.5       Pickaxe       Wood
  Brick Slab                   2.0       Pickaxe       Wood  (double slab)
  Stone Brick Slab             1.5       Pickaxe       Wood
  Quartz Block                 0.8       Pickaxe       Wood
  Quartz Stairs                0.8       Pickaxe       Wood
  Quartz Slab                  0.8       Pickaxe       Wood
  Purpur Block                 1.5       Pickaxe       Wood
  Purpur Stairs                1.5       Pickaxe       Wood
  Purpur Slab                  1.5       Pickaxe       Wood
  Purpur Pillar                1.5       Pickaxe       Wood
  End Rod                      0.0       None          None    Instant break
  Chorus Plant                 0.4       None          None
  Chorus Flower                0.4       None          None
  Cobblestone Wall             2.0       Pickaxe       Wood
  Mossy Cobblestone Wall       2.0       Pickaxe       Wood

GLASS & LIGHT:
  Glass                        0.3       None          None    Drops nothing
  Glass Pane                   0.3       None          None    Drops nothing
  Stained Glass                0.3       None          None    Drops nothing
  Stained Glass Pane           0.3       None          None    Drops nothing
  Glowstone                    0.3       None          None    Drops glowstone dust
  Sea Lantern                  0.3       None          None    Drops prismarine crystals
  Jack o'Lantern               1.0       Axe           None
  Redstone Lamp                0.3       None          None    Drops itself
  End Rod                      0.0       —             —       Instant break

TERRACOTTA & CONCRETE:
  Terracotta (Hardened Clay)   1.25      Pickaxe       Wood
  Stained Terracotta (16)      1.25      Pickaxe       Wood
  Glazed Terracotta (16)       1.4       Pickaxe       Wood
  Concrete (16)                1.8       Pickaxe       Wood    (Hard block!)
  Concrete Powder (16)         0.5       Shovel        None

WOOL & CARPET:
  Wool (16)                    0.8       Shears        None    Shears → block, hand → string
  Carpet (16)                  0.1       None          None    Instant break

REDSTONE COMPONENTS:
  Redstone Dust                0.0       None          None    Instant break, drops itself
  Redstone Repeater            0.0       None          None    Instant break, drops itself
  Redstone Comparator          0.0       None          None    Instant break, drops itself
  Redstone Torch               0.0       None          None    Instant break, drops itself
  Lever                        0.0       None          None    Instant break, drops itself
  Button                       0.5       None          None    Drops itself
  Pressure Plate               0.5       None          None    Drops itself
  Weighted Plate               0.5       None          None    Drops itself
  Tripwire Hook                0.0       None          None    Instant break, drops itself
  Tripwire                     0.0       None          None    Instant break, drops string
  Piston                       0.5       Pickaxe       None    Drops itself
  Sticky Piston                0.5       Pickaxe       None    Drops itself
  Piston Head                  0.5       None          None    Drops nothing
  Observer                     0.5       Pickaxe       None    Drops itself
  Dispenser                    3.5       Pickaxe       None    Drops itself
  Dropper                      3.5       Pickaxe       None    Drops itself
  Hopper                       3.0       Pickaxe       Wood    Drops itself
  Daylight Detector            0.2       None          None    Drops itself

PORTALS & SPECIAL:
  Nether Portal                 -1        —             —       Unbreakable (drops nothing)
  End Portal                    -1        —             —       Unbreakable
  End Portal Frame              -1        —             —       Unbreakable (only with eye)
  End Gateway                   -1        —             —       Unbreakable
  Command Block                 -1        —             —       Unbreakable in survival
  Structure Block               -1        —             —       Unbreakable in survival
  Barrier                       -1        —             —       Unbreakable in survival
  Bedrock                       -1        —             —       Unbreakable

MOB-SPECIFIC:
  Mob Spawner                  5.0       Pickaxe       Iron    Drops XP (only, no item)
  Dragon Egg                   —         None          None    Teleports on hit
  Monster Egg (Infested)       0.75      None          None    Silverfish spawns

LIQUIDS:
  Water / Lava                 100.0     —             —       Requires bucket
  Water / Lava (still)         100.0     —             —       Replaced by flowing
  Cobweb                       4.0       Sword/Shears  None    Shears → block, Sword → string
  Slime Block                  0.0       None          None    Instant break, drops itself

FOOD & CROPS:
  Wheat (crop)                 0.0       None          None    Instant break, drops seed+wheat
  Carrots (crop)               0.0       None          None    Instant break, drops carrots
  Potatoes (crop)              0.0       None          None    Instant break, drops potatoes
  Beetroot (crop)              0.0       None          None    Instant break, drops beetroot
  Melon Block                  1.0       Axe? None     None    Drops melon slices
  Pumpkin                      1.0       Axe           None    Drops itself
  Pumpkin Stem                 0.0       None          None    Instant break
  Melon Stem                   0.0       None          None    Instant break
  Cocoa                        0.2       Axe           None    Instant break
  Nether Wart                  0.0       None          None    Instant break
  Cake                         0.5       None          None    Drops nothing (eaten)
  Mushrooms (red/brown)        0.0       None          None    Instant break
  Mushroom Block (big)         0.2       Axe           None    Drops mushrooms
  Flower (any)                 0.0       None          None    Instant break
  Double Plant (tall grass)    0.0       None          None    Instant break, drops 2 items

STRUCTURE-LIKE:
  Beacon                       3.0       Pickaxe       None    Drops itself
  Enchanting Table             5.0       Pickaxe       None    Drops itself
  Anvil                        5.0       Pickaxe       None    Drops itself (damaged)
  Ender Chest                  22.5      Pickaxe       None    Drops 8 obsidian unless silk touch
  End Crystal                  —         —             —       Explodes when broken
  Flower Pot                   0.0       None          None    Instant break, drops itself
  Item Frame / Painting        0.0       None          None    Instant break, drops itself
  Bed                          0.2       None          None    Drops colored bed item
  Brewing Stand                0.5       Pickaxe       None    Drops itself
  Cauldron                     2.0       Pickaxe       Wood?   Drops itself

Hardness summary (for quick reference):
  Unbreakable (-1):    bedrock, portal, end portal, command blocks
  Very hard (50):      obsidian (only diamond pickaxe)
  Hard (3-5):          ores, spawner, enchanting table, anvil, end stone, chests
  Medium (1-2.5):      stone types, planks, bricks, furnaces, crafting table
  Soft (0.3-0.8):      dirt, sand, gravel, wool, netherrack, glass
  Very soft (0-0.2):   leaves, snow, crops, redstone, torches, flowers — instant break
```

### 9G.4 Instamine Rules
```
A block is "instamined" (breaks in 1 tick = 0.05s) when:
  effective_speed >= 30  (break_time_ticks = ceil(30 / speed) = 1)

This can be achieved when:
  Hand + Haste II:      8 × 1.9 = 15.2 < 30  ❌
  Wood tool + Haste II: 2 × 1.9 = 3.8  ❌
  Gold tool + Haste II: 12 × 1.9 = 22.8 < 30 ❌
  Gold tool + Eff V:    12 + 26 = 38 × 1 = ≥ 30 ✓
  Diamond tool + Eff V: 8 + 26 = 34 × 1 = ≥ 30 ✓
  Iron tool + Eff V:    6 + 26 = 32 × 1 = ≥ 30 ✓
  
  With Haste II + Efficiency V:
    Gold tool:  38 × 1.9 = 72.2 ✓✓ 
    Diamond:    34 × 1.9 = 64.6 ✓✓
    Iron:       32 × 1.9 = 60.8 ✓✓
    Stone:      4 + 26 = 30 × 1.9 = 57 ✓✓

  Even wood with Haste II + Eff V: 2 + 26 = 28 × 1.9 = 53.2 ✓

  Blocks instamine with gold + Eff V + Haste II:
    Hardness up to 2.4 (cobblestone, planks, stone bricks, nether brick)
  
  Blocks that ALWAYS instamine (hardness 0):
    Torches, redstone, buttons, levers, pressure plates, flowers, 
    grass, ferns, saplings, crops, mushrooms, snow layers, string,
    tripwire, cake, lily pads, tall grass, dead bushes, fire, etc.
    These have hardness 0 and always break instantly regardless of tool.
```

### 9G.5 Block Drops & Fortune
```
Block (mined)          → Drop (without fortune)     Fortune effect
──────────────────────────────────────────────────────────────────────
Coal Ore               → 1 coal (0-2 XP)            +1 per level (max 4)
Diamond Ore            → 1 diamond (3-7 XP)         +1 per level (max 4)
Lapis Ore              → 4-8 lapis lazuli (2-5 XP)  +1-2 per level (max 32)
Redstone Ore           → 4-5 redstone (1-5 XP)      +1-2 per level (max 8)
Nether Quartz Ore      → 1 nether quartz (2-5 XP)   +1 per level (max 4)
Emerald Ore            → 1 emerald (3-7 XP)         +1 per level (max 4)
Iron Ore               → iron ore block (smelts)     — (no fortune effect)
Gold Ore               → gold ore block (smelts)     — (no fortune effect)
Glowstone              → 2-3 glowstone dust          +1 per level (max 4)
Melon Block            → 3-7 melon slices            +1-3 per level (max 9)
Nether Wart            → 2-4 nether wart             +1 per level (max 7)
Carrots (crop)         → 1-4 carrots                +1-2 per level (max 7)
Potatoes (crop)        → 1-4 potatoes               +1-2 per level (max 7)
Wheat (crop)           → 1 wheat + 0-3 seeds        +1-2 seeds per level
Beetroot (crop)        → 1 beetroot + 0-3 seeds     +1-2 seeds per level
Sea Lantern            → 2-3 prismarine crystals     +1-3 per level (max 5)

Fortune formula (general):
  rolls = 1 + random(0, fortune_level)
  total = sum of each roll
  For blocks with varying drops (4-8 lapis):
    max_drop = base_max + fortune_level × (fortune_level + 1) ÷ 2 ... not exact
  Accurate formula for Lapis: 
    Fortune I:   4-10 (extra +0-2)
    Fortune II:  4-14 (extra +0-2, 0-2 on each of 3 rolls)  
    Fortune III: 4-22 (extra across up to 4 rolls)

Silk Touch:
  Drops the block itself (with correct tool)
  Ores → ore block (can be smelted or placed)
  Ice → ice block (instead of nothing)
  Glass → glass block (instead of nothing)
  Bookshelf → bookshelf block (instead of 3 books)
  Grass → grass block (instead of dirt)
  Mycelium → mycelium block (instead of dirt)
  Clay → clay block (instead of 4 clay balls)
  Stone → stone block (instead of cobblestone)
  Monster Egg → infested block (instead of silverfish)
  Ender Chest → ender chest block (instead of 8 obsidian)
  Melon → melon block (instead of slices)
  Mushroom Block → mushroom block (instead of mushrooms)
  Leaves → leaves block (instead of sticks/saplings)
  Cobweb → cobweb block (instead of string)
  Vines → vine block (instead of nothing)
  Grass/fern → grass/fern as item (instead of seeds)
  Snow layer → snow layer (instead of snowball)
```

### 9G.6 XP Drops From Breaking
```
Block                        XP orbs (base)
────────────────────────────────────────────
Coal Ore                     0-2
Diamond Ore                  3-7
Emerald Ore                  3-7
Lapis Ore                    2-5
Redstone Ore                 1-5
Nether Quartz Ore            2-5
Mob Spawner                  15-43 (when broken with iron+ pickaxe)
End Portal Frame             0 (unbreakable)
Brewing Stand                0 (drops itself)
Bookshelf                    0 (drops 3 books, 0 XP)

Furnace:                      0 XP (drops itself, not the items inside — items are lost)
Chest:                        0 XP (drops itself, items persist)

Note: No block gives XP from breaking besides ores and spawners.
All other XP comes from smelting (furnace gives XP when items are extracted).
```

---

## P9H — Missing Mechanics & Systems

### 9H.1 Scoreboard System
```
/scoreboard objectives <list|add|remove|setdisplay> <objective> <criteria> [displayName]
  Criteria types:
    dummy                — Manually updated via /scoreboard players set/add/remove
    deathCount           — Times player has died
    playerKillCount      — Times player has killed other players
    totalKillCount       — Total mob kills
    health               — Player health (0-20)
    xp                   — Player experience points
    level                — Player experience level
    food                 — Player food level
    air                  — Player air (max 300)
    armor                — Player armor points
    trigger              — Triggerable by players (adventure maps)
    teamkill.<color>     — Kills of players on team color
    killedByTeam.<color> — Deaths to players on team color
    stat.<statistic>     — Any statistic (e.g. stat.useItem.minecraft.diamond)

/scoreboard players <set|add|remove|get|reset|enable> <target> <objective> [value]
/scoreboard players <list|tag> [target]
/scoreboard teams <add|remove|join|leave|empty|list|option> <team> [option value]
  Team options:
    color              — Team color (affects name, chat, tags)
    displayName        — Custom display name (JSON text)
    friendlyFire       — true/false (default true)
    seeFriendlyInvisibles — true/false (default true)
    nametagVisibility  — always/hideForOtherTeams/hideForOwnTeam (default always)
    deathMessageVisibility — always/hideForOtherTeams/hideForOwnTeam (default always)
    collisionRule      — always/pushOtherTeams/pushOwnTeam/never (default always)
    prefix             — Text prepended to player names (JSON text)
    suffix             — Text appended to player names (JSON text)

Display slots: sidebar, list, belowName (for objectives)
```

### 9H.2 Enchanting Table Mechanics
```
Bookshelf placement:
  - Bookshelves within 2 blocks of table (Manhattan distance)
  - One layer of air between bookshelf and table
  - Max 15 bookshelves (30 for "full" but capped at 15 in vanilla)
  - Bookshelves below or above table height count

Level cost formula:
  base = floor(rand(1,8)) + floor(bookshelves/2) + floor(rand(0, bookshelves))
  Top slot:  max(1, floor(base×1/3))
  Middle:    max(1, floor(base×2/3))
  Bottom:    max(1, base)

Enchantment selection:
  - Roll random enchantments based on item type
  - Weighted random: each enchantment has a weight (e.g. Sharpness=10, Bane=5, Smite=5)
  - Multiple enchantments possible (extra roll per level)
  - Treasure enchantments (Mending, Frost Walker, Curse of Binding/Vanishing) NOT available
    from table — only from loot, fishing, or trading

Enchantment weights:
  Protection     10   | Sharpness      10   | Power         10
  Fire Protection 5   | Smite           5   | Punch          2
  Blast Protection 2  | Bane of Arthro  5   | Flame          2
  Projectile Prot  5  | Knockback       5   | Infinity       1
  Respiration     2   | Fire Aspect     2   | Luck of Sea    2
  Aqua Affinity   2   | Looting         2   | Lure           2
  Thorns          1   | Efficiency     10   | Fortune        2
  Depth Strider   2   | Silk Touch      1   | Unbreaking     5
  Feather Falling 2   | Sweeping Edge — not in 1.12.2 (1.14+)
```

### 9H.3 Anvil Mechanics
```
Repair:
  - Combine item + material (ingot/gem) → repairs 25% of max durability per unit
  - Combine two same items → repairs 5% + 12% of max (capped at max)
  - Cost: base 1-3 levels + prior work penalty + enchantment penalty
  - Max cost: 39 levels (red text = too expensive)

Prior work penalty:
  - Work count: starts at 0, doubles each use
  - Penalty = 2^workCount - 1 extra levels per repair

Rename:
  - Cost: 1 level + prior work penalty
  - Renamed item retains name even when repaired

Enchantment combining:
  - Two items with same enchantment → merges levels (≤ max level)
  - Compatible enchantments → combine
  - Incompatible enchantments → only first item's enchantments kept
  - Conflict table: Sharpness/Smite/Bane mutually exclusive
    Protection/Fire Prot/Blast Prot/Proj Prot mutually exclusive
    Silk Touch/Fortune mutually exclusive
    Depth Strider/Frost Walker mutually exclusive

Item repair materials:
  Wood tools:  planks   | Stone tools: cobblestone  | Iron tools: iron ingot
  Gold tools:  gold ingot | Diamond tools: diamond    | Chain armor: iron ingot
  Leather armor: leather | Elytra: phantom membrane (1.13+)

Renaming costs: 1 level minimum + prior work
Items renamed in anvil: also apply "RepairCost" tag
```

### 9H.4 Beacon Mechanics
```
Pyramid tiers:
  Tier 1: 3×3   (9 blocks)     → 1 primary effect, 20s duration
  Tier 2: 5×5   (34 blocks)    → 1 primary effect, 30s duration
  Tier 3: 7×7   (83 blocks)    → 1 primary effect, 40s duration
  Tier 4: 9×9   (164 blocks)   → 1 primary + 1 secondary effect, 60s duration

Payment items: iron ingot, gold ingot, diamond, emerald
  - One payment item to activate/change effects
  - 3:00 effect duration with pyramid; refreshed by staying near beacon
  - Beam: colored beam into sky (visible from far away)

Primary effects (all tiers):
  Speed I        (+20% movement speed)
  Haste I        (+20% mining speed)
  Resistance I   (-20% damage taken, tier 2+)
  Jump Boost I   (+0.1 jump velocity, tier 2+)
  Regeneration I (1 HP/2.5s, tier 4)

Secondary effect (tier 4 only):
  Regeneration I — replaces primary? No, secondary is added ON TOP of primary
  Secondary is chosen separately
  Available secondary options: Regeneration I (if primary wasn't regen)

Base blocks: iron block, gold block, diamond block, emerald block
```

### 9H.5 Map Mechanics
```
Empty Map: crafted with 9 paper (full grid, center empty)
  - Right-click to activate → becomes filled map (Filled Map)
  - Maps use item damage value for map ID

Zoom levels:
  Zoom 0: 1:1 scale, 128×128 blocks
  Zoom 1: 1:2, 256×256 blocks  
  Zoom 2: 1:4, 512×512 blocks
  Zoom 3: 1:8, 1024×1024 blocks
  Zoom 4: 1:16, 2048×2048 blocks (1.12.2)
  Each zoom = crafting map + 8 paper (full grid)

Map cloning: map + empty map → 2 same maps
Map scaling: map + 8 paper → zoomed-out copy

Colors on map:
  - Grass: green tint based on biome
  - Water: blue
  - Sand: tan
  - Stone: gray
  - Wood: brown
  - Snow: white
  - Banners: show colored dot + name when hovering (1.12 NEW)
  - Terracotta: unique map colors (1.12 NEW)

Player tracking:
  - White dot = player position
  - White dot shrinks when player is below map area
  - Map frame: green dot when map is in an item frame
  - Map lock: when map is in all item frames, no longer updates
```

### 9H.6 Explosion Mechanics
```
Explosion sources:
  TNT               Power 4   Fuse 80 ticks (4s), primed by redstone/fire/flint
  Creeper            Power 3   Fuse 30 ticks (1.5s), charged creeper Power 6
  Bed (Nether/End)   Power 5   Creates fire
  End Crystal        Power 6   Destroyed by damage
  Ghast Fireball     Power 1   Deflectable
  Wither Skull       Power 1   Blue skull breaks obsidian
  Wither (spawn)     Power 7   On creation
  Ender Dragon       Power 3   (actually Dragon doesn't create typical explosions)
  Firework Star      No explosion — just visual

Explosion damage formula:
  impact = (1 - distance/(2×power)) × exposure
  damage = floor(impact × power × 2 + 1)
  exposure = fraction of block rays unobstructed (from entity center)

Explosion block destruction:
  block_resistance / 5 + 0.3    = needed explosion strength
  explosion_strength × (0.7 + rand × 0.6) = applied strength
  If applied > resistance → block destroyed (100% drop rate for TNT, 30% for creepers)
```

### 9H.7 Furnace Mechanics
```
Cook time: 200 game ticks (10 seconds)
Fuel burn times:
  Lava bucket:     1000 ticks (50s) — leaves empty bucket
  Coal block:       800 ticks (40s)
  Blaze rod:        120 ticks (6s)
  Coal/Charcoal:     80 ticks (4s)
  Log/Plank:         15 ticks (0.75s) per item (6 planks = 90 ticks = 4.5 items)
  Stick:              5 ticks (0.25s) — half an item
  Sapling:            5 ticks (0.25s)
  Wooden tool:       10 ticks (0.5s)
  Boat:              60 ticks (3s)
  Door:              15 ticks (0.75s) — actually 1 item worth? No, doors smelt 1.5 items
  Bookshelf:        180 ticks (9s) — 1.5 items (but this doesn't work, bookshelves not fuel)
  Actually fuel values above are the correct 1.12.2 burn times

Fuel efficiency ranking (items smelted per unit):
  Lava bucket: 100 items  | Coal block: 80 items
  Blaze rod:     12 items | Coal/Charcoal: 8 items
  Boat:           6 items | Log/Plank: 1.5 items
  Stick:         0.5 items| Sapling: 0.5 items
  Wooden tool:    1 item  | Wood slab: 0.75 items
```

### 9H.8 Brewing Stand Fuel
```
Blaze powder:
  - 1 blaze powder = 20 brewing operations
  - Stand GUI shows fuel indicator (blaze powder icon with count)
  - No fuel → no brewing, ingredient won't process
  - Brewing time: 20 seconds (400 ticks) per bottle
  - 3 glass bottles can be brewed simultaneously
```

### 9H.9 Boat Mechanics
```
Only 1 boat type in 1.12.2: Oak Boat
  - Crafted: 5 oak planks + 3? No, 5 planks in U shape

Speed: 8.0 m/s (on water, flat movement), ~0.4 m/s (on land)

Behavior:
  - Paddle sound: new in 1.12
  - Max speed increases slightly on ice (friction)
  - Right-click to enter
  - Boat sinks when broken → drops boat item
  - Can carry 2 entities (passenger slot)
  - Movement keys: forward/backward/left/right
  - Damage from high-speed collisions: 1 HP per 4 m/s? No — boat damage only on ice at high speed
  - 6 HP (3 hits to destroy)
```

### 9H.10 Minecart Mechanics
```
5 minecart types:
  - Minecart (empty): transport
  - Minecart with Chest: 27 slots storage
  - Minecart with Furnace: powered by fuel (coal/charcoal)
  - Minecart with TNT: explodes on activator rail (1.12 change: fire charges prime instead of instant explode)
  - Minecart with Hopper: 5 slots, picks up items

Speeds:
  - Powered rail: 8 m/s max
  - Unpowered: slows to 0 over ~15 blocks
  - Furnace minecart: ~0.4 m/s with fuel

Rails:
  - Normal rail: 6 directions (straight N-S, E-W, curves)
  - Powered rail: accelerates minecarts, redstone-powered
  - Detector rail: emits redstone when minecart passes
  - Activator rail: activates minecart functions (TNT, hopper, player dismount)
```

### 9H.11 Fireworks Mechanics
```
Crafting:
  Firework Star: gunpowder + dye + optional modifiers
    Modifiers: glowstone (twinkle), diamond (trail), shape: fire charge (large ball),
               feather (burst), gold nugget (star), skull (creeper face)
  Firework Rocket: paper + 1-3 gunpowder + 0-7 firework stars

Flight duration:
  1 gunpowder:  1 second (height ~20 blocks)
  2 gunpowder:  2 seconds (height ~40 blocks)
  3 gunpowder:  3 seconds (height ~60 blocks)

Explosion shapes: small ball, large ball, star, creeper face, burst
Effects: fade (color transition), twinkle (glowstone), trail (diamond)

Damage: Firework rockets deal up to 7 HP damage per explosion (for crossbows in 1.14+, but for elytra boosting in 1.12: 4 HP per rocket)
  Elytra boosting: firework rocket propels player forward per explosion
```

### 9H.12 Armor Stand Mechanics
```
Crafting: 6 sticks + 1 stone slab (upside-down U)
Placement: right-click block face
Interaction:
  - Right-click with armor: equips armor piece
  - Right-click item frame: puts item in hand slot? No, right-click with item places on armor stand
  - Sneak+right-click: opens GUI (in 1.12.2 no GUI — just place items by right-clicking with them)
  Actually in 1.12.2: right-click armor stand with armor to equip, with item to place in hand

Poses: /entitydata to modify Pose tag
  Pose tag: {Pose:{Head:[0f,0f,0f], Body:[0f,0f,0f], LeftArm:[0f,0f,0f], RightArm:[0f,0f,0f], LeftLeg:[0f,0f,0f], RightLeg:[0f,0f,0f]}}

Special tags:
  NoBasePlate: 1b — removes base plate
  NoGravity: 1b — floats in air
  ShowArms: 1b — shows arms
  Small: 1b — baby size
  Marker: 1b — hitbox becomes very small (invisible), can still be interacted with
  Invisible: 1b — invisible (only items show)
```

### 9H.13 Item Frame Mechanics
```
Crafting: 8 sticks + 1 leather (full grid, center leather)
Placement: right-click on block face
Rotation: right-click item in frame → rotates 45° (8 total positions)
  Rotation 0: upright | Rotation 4: upside-down
Removing item: left-click (empty hand) → item drops
Removing frame: left-click → frame drops (if no item) or item pops off first (if item present)
Damage: arrow shot → item pops off (45° rotation)
Map in frame: displayed as full block on wall, green dot on map
```

### 9H.14 Paintings
```
Crafting: 8 sticks + 1 wool (full grid, center wool)
Placement: right-click on block face
  - Algorithm: finds largest possible painting that fits in available space
  - Placement direction depends on clicked face

Painting variants (26 total in 1.12.2):
  1×1:   Alban, Aztec, Aztec2, Bomb, Kebab, Plant, Wasteland, Courbet, Pool, Sea, Creebet, Sunset
  1×2:   Graham
  2×1:   Wanderer, Bust, Match, SkullAndRoses
  2×2:   Fighters, Kaboom, Stage, Void, Skeleton, Pigscene
  4×2:   BurningSkull
  4×3:   SkullAndRoses? No — 4×3: Wanderer? No. Let me just list the 26:

  Actually the 26 paintings in Minecraft:
  1×1 (12): Kebab, Aztec, Alban, Aztec2, Bomb, Plant, Wasteland, Pool, Courbet, Sea, Creebet, Sunset
  1×2 (1): Graham
  2×1 (4): Wanderer, Bust, Match, SkullAndRoses
  2×2 (5): Fighters, Kaboom, Stage, Void, Skeleton
  4×2 (1): Pigscene
  4×3 (1): BurningSkull
  4×4 (2): SkullAndRoses (yes this is different from the 2×1), Wither
```

### 9H.15 Leads & Leash Knots
```
Crafting: 4 string + 1 slime ball
Usage:
  - Right-click mob with lead → attaches leash
  - Right-click fence with led mob → creates leash knot on fence
  - Right-click leash knot → releases mob
  - Lead breaks: if pulled too far (>10 blocks from knot), if damaged, if mob goes through portal
  - Lead knot entity: has 0.5 HP, drops lead on break
  - Can hold multiple leads on one fence post
  - Can drag mobs across terrain
```

### 9H.16 Horse Stats & Variants
```
35 color variants:
  7 base coat colors: white, creamy, chestnut, brown, black, gray, dark brown
  5 marking patterns: none, white socks, white snout, white blaze, spotted
  Total: 7×5 = 35 (excluding special)

Stats per horse (random):
  Speed:     4.85 - 14.57 m/s (average ~9.71)
  Jump:      1.25 - 5.25 blocks height
  Health:    15 - 30 HP (7.5 - 15 hearts)

Equipment:
  Saddle: required for riding (inventory slot)
  Horse Armor: leather (1.5 armor?), iron (5 armor?), gold (4 armor?), diamond (7 armor?)
  Actually armor values: leather=3, iron=5, gold=4, diamond=7
  Chest: donkeys and mules only, 0-15 slots (random)

Taming:
  - Must be tamed by mounting repeatedly (tameness increases per try)
  - Feed: sugar (3), wheat (3), apple (3), golden carrot (4), golden apple (10)
  - Taming formula: tameness + food_modifier - 100 = 0 → tamed
  - Breeding: golden apple/golden carrot → foal (4y 8mo growth time)
  - Feeding speeds growth: 1 minute per food item

Donkey/Mule:
  - Donkey: spawns with 1 chest (15 slots), cannot be equipped with armor
  - Mule: horse + donkey hybrid, same chest + no armor

Skeleton Horse Trap:
  - "Horse" that appears during thunderstorms (4 skeleton horsemen)
  - Cannot be ridden until skeleton is killed
```

### 9H.17 Rabbit Variants & Killer Bunny
```
6 rabbit skins:
  1. Brown     — common (plains, desert, etc.)
  2. White     — snowy biomes (ice plains, cold taiga)
  3. Black     — common
  4. Black & White — common
  5. Gold      — common
  6. Salt & Pepper — common
  7. Toast     — special (named "Toast" or bred from named parents)

Killer Bunny:
  - Rare hostile variant (1/1000 spawn chance)
  - Pure white with blood-red eyes
  - Attacks players directly (unlike normal rabbits which flee)
  - Cannot be tamed
  - 8 HP? Actually 3 HP like normal rabbits but deals 8-12 damage
```

### 9H.18 Parrot Types
```
5 color variants (spawn randomly in jungles):
  1. Red (blue/orange wings — macaw)       — 20% weight
  2. Blue (cyan body)                      — 20% weight
  3. Green (dark green body)               — 20% weight
  4. Cyan (gray body with blue tips)       — 20% weight
  5. Gray (gray body — cockatoo)           — 20% weight

Behaviors:
  - Tamed with seeds (wheat, melon, pumpkin, beetroot)
  - Sits on shoulder (enter/exit by jumping)
  - Dances near jukebox playing music
  - Mimics hostile mob sounds (11.2% chance when mob is 3m away)
  - Dies when fed cookies (instantly) — {Cookies: 1, poison effect}
  - 6 HP, 2-3 HP damage on hit
  - Flies up when hurt, lands on ground eventually
  - Spawns in jungles, groups of 1-2
```

### 9H.19 Ocelot/Cat Mechanics (1.12.2)
```
Ocelot spawning: jungle biomes (y=63+), 1-2 per group
Taming:
  - Feed raw cod or raw salmon (sneak + right-click)
  - Trust increases by 1/3 chance per fish
  - Once tamed: transforms into a cat (3 random patterns)
  - 50 HP cat: follows player, can sit, teleports

Cat patterns (3 in 1.12.2):
  1. Tuxedo (black with white chest/paws)
  2. Tabby (orange with white face/chest)
  3. Siamese (cream with dark face/paws/tail)

Behaviors:
  - Creepers flee from ocelots/cats (6.25 block radius, 10 second cooldown)
  - Cats can sit on chests (prevents opening)
  - Cats don't take fall damage
  - Gift: morning gifts (rabbit hide, rabbit foot, string, feathers, phantom membrane not in 1.12.2)
```

### 9H.20 Sheep Natural Colors
```
Natural spawn weights:
  White:      81.84%  | Light Gray: 0.84%
  Black:       5.00%  | Gray:       0.84%
  Gray:        5.00%  | Brown:      0.16%  (actually wait — the above is correct?)
  Actually the correct 1.12.2 sheep colors:
  White: 81.836%  | Black: 5%     | Light Gray: 0.836%
  Gray:  5%       | Brown: 0.164% | Pink:       0.164%
  Total: ~93% of sheep have colored wool worth dyeing

Wool drops: 1-3 blocks (sheared) or 1 block (killed). Looting increases when killed.
Dye colors achieved through breeding: parents pass color to lamb (mixed = secondary color)
```

### 9H.21 Screen Effects / Overlays
```
Fire overlay: red/orange at screen edges when on fire
  - Intensity based on remaining fire time (fades as fire extinguishes)
  - Rendered as overlay texture over the entire screen

Water overlay: blue vignette when underwater
  - Visibility reduces with depth
  - Bubbles rise from screen edges when drowning

Lava overlay: orange/red tint, similar to fire but more intense
  - Fog: dense orange fog (distance reduced to 3 blocks underground)

Nausea overlay: screen warps (sinusoidal distortion)
  - Applied by Nausea effect, eating pufferfish, Nether portal
  - Intensity depends on effect level and duration

Pumpkin overlay: solid carved pumpkin texture (hidden HUD)
  - Applied when wearing carved pumpkin on head

Portal overlay: purple swirly vignette
  - Appears when standing in nether portal
  - Intensifies as teleport timer counts down (4s)

Frost overlay: white vignette at screen edges
  - Applied by Frost Walker enchantment freezing water nearby? Actually Frost Walker doesn't create a screen overlay

Blindness: thick black fog, no sprinting, no critical hits
Darkness (effect): dynamic lighting reduction? In 1.12.2 blindness is the main screen-blackening effect

Death screen: red vignette fading to black, then respawn buttons
```

### 9H.22 Particle Types (1.12.2, 23+ types)
```
Particle name             | Usage
barrier                   | Barrier block display
blockcrack               | Block breaking + cracking
blockdust                | Block walking on (falling particles)
bubble                   | Underwater entity bubbles
cloud                    | Ender pearl, elytra, etc.
crit                     | Critical hit
damageindicator          | Damage numbers (absorption/health reduction)
depthsuspend             | Underwater (suspended)
dragonbreath             | Dragon's breath / lingering potion area
dripLava                 | Lava dripping through block
dripWater                | Water dripping through block
enchantmenttable         | Enchanting table book particles
endRod                  | End rod particles
explode                  | Explosion
fireworksSpark          | Firework trail
flame                    | Fire, torches, furnaces, etc.
footstep                | Footstep particles
happyVillager           | Trading, bone meal
heart                    | Breeding, taming
instantSpell            | Instant health/damage particles
iconcrack               | Item break (tools, etc.)
itemcrack               | Eating, potion breaking
largeexplode            | Large explosion
largesmoke              | Large smoke (fire, TNT)
lava                     | Lava
magicCrit               | Enchanted weapon critical
mobSpell                | Effect particles on mobs
mobSpellAmbient         | Ambient effect particles
note                     | Note block
portal                   | Portal, enderman, endermite, chorus fruit
reddust                  | Redstone
slime                    | Slime
smallSmoke              | Small smoke (furnace, fire)
snowballpoof            | Snowball hit
snowshovel              | Snow step
spell                    | Potion effect
spit                     | Llama spit
splash                   | Water splash
suspend                  | Underwater
sweepAttack             | Sword sweep (1.12.2 has this)
take                     | No usage? Actually unused/deprecated
townaura                | Mycelium particles
wake                     | Fishing bobber water wake
witchMagic              | Witch casting
```

### 9H.23 Tutorial Hints (1.12 NEW)
```
Displayed in top-right corner of screen. Only shown once per world/device.
Hint flow:
  1. "Press WASD to move" (shown on first movement input)
  2. "Find trees to punch for wood" (shown after moving)
  3. "Punch a tree to get wood" (shown near a tree)  
  4. "Open your inventory" (shown after getting wood)
  5. "Craft planks from wood" (shown after opening inventory)
  6. Hidden after crafting planks

Hints can be disabled via the "Tutorial" option? In 1.12.2, no — they appear once and are dismissed.
```

### 9H.24 Splash Text & Minceraft Easter Egg
```
Splash text: random message on title screen
  New in 1.12:
    "Don't feed chocolate to parrots!"
    "The true meaning of covfefe"
    "An illusion! What are you hiding?"
    "Something's not quite right..."
  
  Total splashes in 1.12.2: 100+ (including classics like "Also try Terraria!")

Minceraft easter egg:
  - 1/10000 chance (0.01%) to show "Minceraft" instead of "Minecraft" on title screen
  - Pure visual easter egg — no gameplay impact
  - Checks on each title screen load
```

### 9H.25 New NBT Tags in 1.12
```
Entity NBT additions:
  - LastExecution (long) — Command block last execution game time
  - UpdateLastExecution (byte) — Whether to update LastExecution
  - LoveCauseLeast (long) — Villager breeder UUID part 1
  - LoveCauseMost (long) — Villager breeder UUID part 2
  - ConversionPlayerLeast (long) — Zombie villager curing player UUID part 1
  - ConversionPlayerMost (long) — Zombie villager curing player UUID part 2
  - ShoulderEntityLeft (compound) — Parrot on left shoulder
  - ShoulderEntityRight (compound) — Parrot on right shoulder
  - enteredNetherPosition (compound with x, y, z doubles) — For advancement tracking
  - seenCredits (byte) — Whether player has seen end credits

Player NBT additions:
  - recipeBook (compound) with:
    - isFilteringCraftable (byte) — Recipe book filtering state
    - isGuiOpen (byte) — Recipe book GUI open state
    - recipes (list of string) — Unlocked recipe IDs
    - toBeDisplayed (list of string) — Recipe IDs to show toast for

Item NBT additions:
  - Recipes (list of string) — For Knowledge Book, recipe IDs to unlock

Block entity:
  - bed color via "color" tag (int from 0-15) for colored beds
```

### 9H.26 Player Data Structure
```
Full player NBT structure (persisted to IndexedDB):
  - Pos (list of 3 doubles) — Position [x, y, z]
  - Motion (list of 3 doubles) — Velocity [dx, dy, dz]
  - Rotation (list of 2 floats) — Yaw, Pitch
  - Health (float) — Player health
  - foodLevel (int) — Hunger level (0-20)
  - foodSaturationLevel (float) — Saturation
  - foodTickTimer (int) — Tick counter for food regen
  - HurtTime (int) — Hurt animation timer
  - DeathTime (int) — Death animation timer
  - FallDistance (float) — Fall damage tracking
  - OnGround (byte) — Whether on ground
  - Dimension (int) — 0=Overworld, -1=Nether, 1=End
  - Inventory (list of compounds, max 41 slots)
  - ArmorItems (list of 4 compounds) — Feet, Legs, Chest, Head
  - HandItems (list of 2 compounds) — Main hand, Offhand
  - playerGameType (int) — 0=Survival, 1=Creative, 2=Adventure, 3=Spectator
  - Score (int) — Scoreboard score
  - SelectedItemSlot (int) — Hotbar selection (0-8)
  - XpLevel (int) — XP level
  - XpP (float) — XP progress bar (0-1)
  - XpTotal (int) — Total XP earned
  - SpawnX, SpawnY, SpawnZ (int) — Bed spawn position
  - SpawnForced (byte) — Force spawn at bed
  - SleepTimer (short) — Sleep progress
  - seenCredits (byte) — Seen end credits
  - enteredNetherPosition (compound) — Nether entry position
  - ShoulderEntityLeft/Right (compound) — Parrots on shoulders
  - recipeBook (compound) — Recipe book state
  - abilities (compound) — Fly speed, fly, instabuild, invulnerable, mayfly, mayBuild
  - foodExhaustionLevel (float) — Exhaustion
  - EnderItems (list of compounds) — Ender chest contents
```

### 9H.27 @s Selector (1.12 NEW)
```
@s = targets the executing entity
  - Only works in commands executed by an entity
  - In /execute run <command> @s refers to the entity from /execute
  - In command blocks: @s targets the command block itself (which has no meaning for most commands)
  - Useful in: /execute @e[type=zombie] ~ ~ ~ say I am @s
  - Advancement rewards: @s targets the player who earned the advancement
  - Function execution: @s targets the command sender
```

### 9H.28 Player Keybinding Additions (1.12)
```
New in 1.12:
  - Advancements: default "L" key (opens advancements GUI)
  - Save toolbar: "C" key + number (1-9) — save hotbar preset
  - Load toolbar: "X" key + number (1-9) — load hotbar preset
  - Narrator: "Ctrl+B" toggle
  - Recipe book: button in crafting interface (toggle recipe book open/close)
  - Recipe book filter: toggle between "all recipes" and "craftable only"
```

### 9H.29 Dye Sources & Mixing (16 colors)
```
Sources for each dye:
  White:     bone meal (from bone)       | Light Gray: white+tulip? Actually Azure Bluet, Oxeye Daisy, white+gray
  Gray:      ink sac + bone meal (2:1 mix) | Black:  ink sac, wither rose (no — 1.14+)
  Brown:     cocoa beans                 | Red:    poppy, rose bush, tulip, beetroot
  Orange:    orange tulip               | Yellow: dandelion, sunflower
  Lime:      cactus green? No, cactus green gives green dye.
  Actually 1.12.2 dye sources:
  White:     bone meal (1 bone → 3 bonemeal)
  Light Gray: azure bluet, oxeye daisy, white tulip, or white+gray
  Gray:      ink sac + bone meal, or black+white
  Black:     ink sac (from squid)
  Brown:     cocoa beans (from jungle)
  Red:       poppy, rose bush, tulip, beetroot
  Orange:    orange tulip, or red+yellow
  Yellow:    dandelion, sunflower
  Lime:      cactus green? No — green dye from cactus
  Actually correct sources:
  - White Dye: bone meal
  - Light Gray Dye: azure bluet, oxeye daisy, white tulip, or white+gray mixing
  - Gray Dye: black+white, or gray+white? Gray = black+white (×2)
  - Black Dye: ink sac
  - Brown Dye: cocoa beans
  - Red Dye: poppy, red tulip, rose bush, beetroot
  - Orange Dye: orange tulip, or red+yellow
  - Yellow Dye: dandelion, sunflower
  - Lime Dye: green+white? Actually green+white makes lime
  - Green Dye: cactus (smelted)
  - Cyan Dye: green+blue? Actually cactus+lapis? Yes, green + blue
  - Light Blue Dye: blue orchid, or blue+white
  - Blue Dye: lapis lazuli
  - Purple Dye: red+blue
  - Magenta Dye: allium, lilac, or purple+pink, or blue+red+pink (3 inputs)
  - Pink Dye: peony, pink tulip, or red+white

Dye mixing table:
  Red + Yellow   = Orange       | Red + Blue     = Purple
  Blue + White   = Light Blue   | Blue + Green   = Cyan
  Green + White  = Lime         | Red + White    = Pink
  Black + White  = Gray         | Gray + White   = Light Gray
  Red + Blue + Pink = Magenta   | Purple + Pink  = Magenta

Yellow + Blue = ? No, yellow+blue does not make green (green is from smelting cactus)
```

### 9H.30 Carrot on a Stick & Pig Riding
```
Carrot on a Stick:
  - Crafting: fishing rod + carrot
  - Durability: 25 uses (1 per boost? Actually no — durability decreases by 1 per use? No, the
    carrot on a stick loses 7 durability per use, max 25 = 3-4 full uses)
  - Actually in 1.12.2: using carrot on a stick to control a pig reduces durability by 7 each time
    the pig's speed boost is triggered. 25/7 = 3 boosts before breaking.

Pig riding:
  - Saddle required on pig (right-click pig with saddle)
  - Carrot on a stick in hand to control direction
  - Right-click with carrot on stick → pig speeds up (consumes durability)
  - Pig speed: ~2.3 m/s base, ~8 m/s with boost
  - Pig can move in water but slowly
  - Cannot control pig without carrot on a stick
```

### 9H.31 Additional 1.12 Minor Changes
```
Title screen:
  - 1.12.2: changed to "Minecraft: Java Edition" subtitle (separate texture from logo)

Crafting interface:
  - Closing crafting table/grid with items in output → items sent to inventory, NOT dropped

F1 key:
  - Hides toast notifications (advancement toasts, recipe toasts)

Player body rotation:
  - Player body now faces entirely forward when walking backward (previously rotated sideways)

NBT parsing:
  - Keys can be quoted optionally
  - Stricter parsing: no spaces/symbols in unquoted keys
  - Better error messages with exact location

Creative mode:
  - "Materials" tab merged into "Misc" tab
  - All 12 tabs reordered: Build Blocks, Decoration Blocks, Redstone, Transportation,
    Miscellaneous, Food, Tools, Combat, Brewing, Materials, Search, Survival Inventory
  - Search tab: text field filters items by name (type to search)
  - Creative search is heavily optimized
```

### 9H.32 Recipes Folder & Recipe JSON Format
```
Recipes stored in: assets/<namespace>/recipes/<id>.json
Recipe JSON format:
  Shaped: {
    "type": "minecraft:crafting_shaped",
    "pattern": ["AA"," A"],          // rows, max 3×3
    "key": {"A": {"item": "minecraft:oak_planks"}},
    "result": {"item": "minecraft:oak_fence_gate", "count": 1}
  }
  Shapeless: {
    "type": "minecraft:crafting_shapeless",
    "ingredients": [{"item": "minecraft:blaze_powder"}, {"item": "minecraft:slime_ball"}],
    "result": {"item": "minecraft:magma_cream", "count": 1}
  }
  Smelting: {
    "type": "minecraft:smelting",
    "ingredient": {"item": "minecraft:iron_ore"},
    "result": "minecraft:iron_ingot",
    "experience": 0.7
  }

Recipe book:
  - Recipes organized by category: building, decoration, redstone, transportation, food, tools, combat, brewing, misc
  - Group property: sub-groups recipes together (e.g. all wooden planks)
  - Recipe unlocking: automatic when obtaining ingredients
  - Toast notification: new recipe unlocked (yellow icon top-right, 5s duration)
```

### 9H.33 Village Mechanics
```
Door detection ("housing"):
  Scan 16-block radius from village center for valid doors
  Valid door = wooden door block where the number of "roof" blocks (opaque blocks blocking
  sunlight) within 5 blocks north/south (or east/west depending on facing) differs on one side
  More roof blocks on one side than the other = "inside" vs "outside" determined
  Each valid door = 1 "house"

Population cap:
  Max villagers = 0.35 × number_of_valid_doors (rounded down)
  Min villagers = 0.1 × number_of_valid_doors
  Villagers breed when population < 100% of cap and there are ≥ 2 villagers and willing

Village center: computed as average of all door positions
Village radius: 32 blocks minimum, or distance from center to furthest door + 32

Iron golem spawning:
  Requirements: ≥ 10 villagers, ≥ 75% of doors are valid houses
  Spawn rate: 1 golem per 35-180 seconds (attempted every 700 ticks)
  Spawn location: 6 blocks from village center, on highest solid block
  Max golems: 1 per 10 villagers (rounded down)
  Golem drops: 3-5 iron ingots + 0-2 poppies
  Golem health: 100 HP, attack: 3-21 HP (varies), knockback immunity

Villager breeding:
  "Willing" state: triggered by having 12 carrots/potatoes/beetroot or 3 bread in inventory
  Baby: instant spawn, grows in 20 minutes (24000 ticks), accelerates with food
  Trade unlocking: each trade has 2-12 uses before locking; uses = 2 + floor(rand×10)
  After locking: restocks when villager is willing (can work at their workstation)
  Max tier: 5 trades per profession, unlocked by cumulative trading XP

Zombie siege (night):
  On hard difficulty, zombies can spawn near villages at midnight regardless of light level
  20 zombies per wave, within 15 blocks of village center
  Only if village has ≥ 10 doors and ≥ 20% villagers survived

Cat spawn: 1 cat per 4 villagers (up to 5), within 32 blocks of village center
  - Ocelot/cat spawns on hay bales or near villagers
```

### 9H.34 Mob Despawn Rules
```
Natural spawns (most hostile/passive mobs):
  Despawn if distance > 128 blocks from nearest player (instant)
  Despawn if distance > 32 blocks from nearest player (random chance per tick)
  Specifically: mobs are "persistent" if within 32-block despawn radius
    but NOT persistent if 32-128 blocks away — 1/800 chance per tick to despawn
  Instant despawn: > 128 blocks away

Name tag persistence:
  Any mob with a name tag NEVER despawns (PersistanceRequired: 1b)
  Name tag mobs: can still be killed by damage, but won't vanish

Leashed mobs:
  Leashed mobs do NOT despawn (Leash tag present)
  Breaking leash: if pulled > 10 blocks from knot → leash breaks

Equipment-pickup persistence:
  Zombies and skeletons that picked up equipment become persistent?
  Actually: any mob that has picked up items (canPickUpLoot: 1b) becomes persistent

Breeding persistence:
  Animals that have been bred become persistent? No — they still despawn.
  Actually: bred animals DO despawn normally.
  Only tamed animals, named mobs, and leashed mobs are immune to despawning.

Passive mob despawn:
  Passive mobs (sheep, cow, pig, chicken, rabbit, horse) CAN despawn if > 128 blocks
  But: animals loaded as part of terrain generation are tagged with FromSpawner: 0
    and despawn only above 128 blocks, not in 32-128 range

Hostile mob despawn:
  All hostile mobs despawn per the 32/128 rules above
  Exception: wither (never despawns), ender dragon (never despawns)
  Slimes and magma cubes: despawn normally

Villager despawn:
  Villagers NEVER despawn (persistent by default)

Iron golems, snow golems, wandering traders (1.13+): not relevant

Despawn exact formula:
  Every tick: if distance > 128 → instant despawn
  Every tick: if distance > 32 → 1/800 chance to despawn
  Checks every 1 second (20 ticks) in practice
  PersistenceRequired NBT tag = 1 prevents all despawn checks
```

### 9H.35 Mob AI Details
```
Pathfinding range:
  Most hostile mobs: 16 blocks from spawn point (wander radius)
  Zombie: 16 block follow range, 40 block "home" range
  Skeleton: 16 block follow, shoots at ≤ 15 blocks
  Creeper: 16 block follow, 3 block explode radius (fuse 30 ticks)
  Spider: 16 block follow, can pathfind up walls (no ceiling climbing)
  Enderman: 64 block aggro (staring), 32 block follow when provoked
  Silverfish: 12 block call range (calls other silverfish)
  Slime: 16 block follow (all sizes)
  Ghast: 100 block fireball range (but spawns anywhere in nether)
  Blaze: 16 block follow, 5 block fireball range
  Witch: 16 block aggro, throws potions at ≤ 8 blocks
  Guardian: 16 block follow, laser ≤ 15 blocks (charges 2s)
  Shulker: 10 block bullet tracking range
  Wolf (hostile): 16 block aggro if provoked by player
  Iron golem: 16 block aggro vs hostile mobs

Aggro range modifiers:
  Nighttime: zombies/skeletons have 2× aggro range
  Hard difficulty: +50% follow range
  Regional difficulty: increases equipment/enchant chance (not range directly)

Flee behavior:
  Villager: flees from zombies within 8 blocks
  Rabbit: flees from players within 8 blocks (except killer bunny)
  Wolf: flees from players (neutral unless provoked)
  Squid: flees from players in water
  Parrot: flees from players (flighty)
  Ocelot: flees from players (until tamed)
  Llama: spits at wolves within 4 blocks (flees from player)

Mob attack cooldowns:
  Zombie: 1 attack per 0.5s (10 ticks), damage 3
  Skeleton: 2s (40 ticks) between shots
  Creeper: 1.5s (30 ticks) fuse, explosion 3 (charged 6)
  Spider: 1 attack per 1s (20 ticks), jump attack
  Enderman: teleports on hit, attacks after 1s
  Blaze: 3 fireballs every 2s (40 ticks), then 1.5s (30 ticks) cooldown
  Ghast: 1 fireball per 1.5s (30 ticks), max 3 in a burst
  Guardian: 2s laser charge, then instant damage
  Witch: 0.75s (15 ticks) between thrown potions
  Shulker: 0.5s (10 ticks) bullet fire rate, 1-4s bullet travel
  Vindicator: melee, same as zombie (10 ticks cooldown)
  Evoker: summons 2-3 vexes every 5s (100 ticks), uses fangs at ≤ 6 blocks
  Vex: 1 attack per 0.5s, phases through blocks, despawns after 30s
  Ender Dragon: 3s (60 ticks) between fireball charges, 5 HP contact damage

Mob senses:
  Zombie: can "smell" players through walls (detect within 16 blocks even without line of sight)
  Skeleton: requires line of sight to shoot
  Spider: can see through walls (detect player within 16 blocks)
  Creeper: follows within 3 blocks start hissing; continues until 1.5s explosion or 1.5s after player leaves range
  Enderman: only aggro when player looks at head/body (up to 64 blocks)
  Silverfish: calls reinforcements within 12 blocks when hit

Mob equipment (zombies/skeletons):
  Chance to spawn with equipment based on difficulty:
    Easy: 0% chance
    Normal: 0%-15% per slot (based on regional difficulty)
    Hard: 0%-100% per slot (based on regional difficulty)
  Armor type: leather (low), gold (med), chainmail (med-high), iron (high), diamond (very rare)
  Enchantment: up to 30% per equipped item (regional difficulty based)
  Equipment tiers: better equipment at higher regional difficulty
  Weapon: zombie (iron sword or none), skeleton (bow)
  Zombie arms: have "arm" but don't show weapon visually (arm swing animation)

Baby zombies:
  ~5% chance of any zombie spawn being a baby
  Baby: faster (50% speed boost), smaller hitbox (0.3×0.975), 12 HP
  Can ride mobs (chicken jockey: baby zombie + chicken = 0.25% of zombie spawns)
  Can mount other mobs (zombie + any passive/hostile = random jockey)
  Burns in sunlight but can wear helmets to block
```

### 9H.36 Entity NBT Structures
```
Zombie NBT (key tags):
  {IsBaby:0b, IsVillager:0b, ZombieType:0, ConversionTime:-1,
   CanBreakDoors:0b, DrownedConversionTime:-1,
   HandItems:[{},{}], ArmorItems:[{},{},{},{}],
   HandDropChances:[0.085f,0.085f], ArmorDropChances:[0.085f,...],
   CanPickUpLoot:0b, PersistenceRequired:0b,
   Leash:{}, Attributes:[], ActiveEffects:[]}

Skeleton NBT:
  {SkeletonType:0, HandItems:[{id:bow},{}], ArmorItems:[{},{},{},{}],
   CanPickUpLoot:0b, PersistenceRequired:0b,
   StrayConversionTime:-1}
  SkeletonType: 0=normal, 1=wither skeleton, 2=stray

Creeper NBT:
  {powered:0b, ExplosionRadius:3, Fuse:30, ignited:0b,
   CanPickUpLoot:0b, PersistenceRequired:0b}

Enderman NBT:
  {carried:0, carriedData:0, CanPickUpLoot:0b}

Spider NBT:
  {CanPickUpLoot:0b, PersistenceRequired:0b}
  Cave spider inherits same structure (different ID/hitbox)

Witch NBT:
  {CanPickUpLoot:0b, PersistenceRequired:0b}

Villager NBT:
  {Profession:0, Career:0, CareerLevel:1, CanPickUpLoot:0b,
   Willing:0b, Offers:{Recipes:[...]}, Inventory:[],
   Age:0, ForcedAge:0, InLove:0, LoveCauseLeast:0L, LoveCauseMost:0L}
  Profession: 0=fisherman, 1=farmer, 2=librarian, 3=cleric, 4=blacksmith,
              5=butcher, 6=leatherworker

Horse NBT:
  {Variant:0, Temper:0, Tame:0b, OwnerUUID:"",
   EatingHaystack:0b, Bred:0b, HorseType:0}
  HorseType: 0=horse, 1=donkey, 2=mule, 3=undead? Actually:
    Variant encodes color+markings: (type << 8) | (color << 4) | markings
    Type: 0=horse, 1=donkey, 2=mule, 3=zombie horse, 4=skeleton horse

Parrot NBT:
  {Variant:0}  0=red, 1=blue, 2=green, 3=cyan, 4=gray

Wolf NBT:
  {CollarColor:14, Sitting:0b, Tame:0b, OwnerUUID:""}
  CollarColor: 0=white, 1=orange, ... 15=black (standard dye order)

Ocelot NBT:
  {CatType:0}  0=ocelot, 1=tuxedo cat, 2=tabby cat, 3=siamese cat

Llama NBT:
  {Variant:0, Strength:3, ChestedHorse:0b, DecorItem:{}}
  Variant: 0=creamy, 1=white, 2=brown, 3=gray

Sheep NBT:
  {Color:0, Sheared:0b}
  Color: 0=white, 1=orange, ... 15=black

Slime NBT:
  {Size:1}  1=small, 2=medium, 4=large (Size: 0= tiny (1 HP), 1=small (4 HP), 4=big (16 HP))

Shulker NBT:
  {Color:10, AttachFace:0, Peek:0, APX:0, APY:0, APZ:0}
  Color: 10=purple (default), 0-15 for dyed, 16=no color
  AttachFace: 0=down, 1=up, 2=north, 3=south, 4=west, 5=east

Vex NBT:
  {BoundX:0, BoundY:0, BoundZ:0, LifeTicks:600, Alive:1b}
  LifeTicks: remaining lifetime (default 600 ticks = 30s)

Evoker NBT:
  {Spells:[], CanPickUpLoot:0b, PersistenceRequired:0b,
   Wolves:0, Johnny:0b}
  Johnny: 1b = attacks everything (named "Johnny" via name tag → auto-sets Johnny:1b)

Guardian NBT:
  {Elder:0b}  0=normal guardian, 1=elder guardian

Item entity NBT:
  {Item:{id:"minecraft:diamond", Count:1b, Damage:0s}, Health:5s, Age:0s, PickupDelay:10s,
   Owner:/UUID if thrown by player, Thrower:UUID}
  Age: incremented each tick, at 6000 (5 min) → item despawns
  Health: 5 = full (items can be destroyed by damage/explosions)

XP orb NBT:
  {Value:1}  Value = XP amount (1-247 per orb, larger orbs = more XP)
  Health: 5 HP, cannot be damaged by most sources

Falling block NBT:
  {BlockState:{Name:"minecraft:sand"}, TileEntityData:{},
   Time:1, DropItem:1b, HurtEntities:0b, FallHurtAmount:2f, FallHurtMax:40}
  Time: ticks falling; if Time > 600 → block breaks (drops item)

Primed TNT NBT:
  {Fuse:80, ExplosionPower:4}  Fuse in ticks

Boat NBT:
  {Type:"oak"}  Always "oak" in 1.12.2 (only 1 boat type)
  LeftRightHeading: angle

Minecart NBT:
  Common: {CustomDisplayTile:0b, DisplayTile:{}, DisplayOffset:0, DisplayData:0}
  Chest: {Items:[]}
  Furnace: {Fuel:0, PushX:0.0, PushZ:0.0}
  TNT: {Fuse:80}
  Hopper: {Items:[], Enabled:1b, TransferCooldown:0, LootTable:""}

Armor Stand NBT:
  {Pose:{Head:[0f,0f,0f], Body:[0f,0f,0f], LeftArm:[0f,0f,0f],
          RightArm:[0f,0f,0f], LeftLeg:[0f,0f,0f], RightLeg:[0f,0f,0f]},
   NoBasePlate:0b, NoGravity:0b, ShowArms:0b, Small:0b, Marker:0b, Invisible:0b}

Item Frame NBT:
  {Item:{id:"", Count:1b, Damage:0s}, ItemDropChance:1.0f, ItemRotation:0b}
  Rotation: 0-7 (0=0°, 1=45°, ..., 7=315°)

Falling sand / gravel NBT:
  {BlockState:{Name:"minecraft:sand"}, TileEntityData:{}, Time:1, DropItem:1b,
   HurtEntities:0b, FallHurtAmount:2f, FallHurtMax:40}
```

### 9H.37 Banner Patterns
```
Base banner crafted: 6 wool (top row) + 1 stick (center bottom) → 1 banner of that color
16 base colors: same dye colors (white through black)

Pattern application: combine banner + dye in crafting grid
  Result: banner with pattern layered on (dye color)
  Patterns stack: up to 6 patterns per banner (applied sequentially)

Complete pattern list (1.12.2, 20 patterns):
  Resource Location              | In-game Name
  base                           | Base (the banner background)
  border                         | Bordure Indented
  bricks                         | Field Masoned
  circle                         | Roundel
  creeper                        | Creeper Charge
  cross                          | Saltire
  curly_border                   | Bordure Indented Wavy
  diagonal_left                  | Per Bend Sinister
  diagonal_right                 | Per Bend
  diagonal_up_left               | Per Bend Inverted
  diagonal_up_right              | Per Bend Sinister Inverted
  flower                         | Flower Charge
  gradient                       | Gradient
  gradient_up                    | Base Gradient
  half_horizontal                | Per Fess
  half_horizontal_bottom         | Per Fess Inverted
  half_vertical                  | Per Pale
  half_vertical_right            | Per Pale Inverted
  mojang                         | Thing (Mojang logo — requires 1 enchant apple + 1 paper)
  rhombus                        | Lozenge
  skull                          | Skull Charge
  small_stripes                  | Paly
  square_bottom_left             | Base Dexter Canton
  square_bottom_right            | Base Sinister Canton
  square_top_left                | Chief Dexter Canton
  square_top_right               | Chief Sinister Canton
  straight_clear                 | Fess
  stripe_bottom                  | Base
  stripe_center                  | Pale
  stripe_downleft                | Bend Sinister
  stripe_downright               | Bend
  stripe_left                    | Paly Dexter
  stripe_middle                  | Fess
  stripe_right                   | Paly Sinister
  stripe_top                     | Chief
  triangle_bottom                | Chevron
  triangle_top                   | Inverted Chevron
  triangles_bottom               | Base Indented
  triangles_top                  | Chief Indented
  tri_bottom                     | Per Fess of Three
  tri_top                        | Per Fess of Three Inverted
  tri_side_left                  | Per Pale of Three
  tri_side_right                 | Per Pale of Three Inverted

Pattern crafting recipe examples:
  Creeper Charge (creeper pattern):  1 creeper skull + 1 paper
  Skull Charge (skull pattern):      1 wither skeleton skull + 1 paper
  Flower Charge (flower pattern):    1 oxeye daisy + 1 paper
  Thing (mojang pattern):            1 enchanted golden apple + 1 paper
  Field Masoned (bricks pattern):    1 brick block + 1 paper
  Bordure Indented (curly_border):   1 vine + 1 paper

Shield crafting:
  Shield = 6 planks + 1 iron ingot (full grid, iron at top-center)
  Banner + Shield: combine in crafting → applies banner pattern to shield face
  Shield inherits the banner's color and patterns
  Shield model: 3D item, shows pattern on front face
```

### 9H.38 Leather Armor Dyeing Formula
```
Leather armor can be dyed by combining with any dye in crafting grid.
Multiple dyes can be used at once.

Color mixing formula:
  For each dye color in the recipe:
    - Add (dye_red, dye_green, dye_blue) to accumulated RGB
    - Add 1 to count
    - Add dye's "color value weight" (default 1.0 for most dyes)
  Result RGB:
    - Average: (total_r / count, total_g / count, total_b / count)
    - This creates a blended color

Dye color values (standard Minecraft colors):
  White:     (255,255,255)    | Orange: (219,125, 62)   | Magenta: (179, 58,180)
  Light Blue:(107,138,201)    | Yellow: (177,166, 39)   | Lime:    ( 65,174, 56)
  Pink:      (208,132,153)    | Gray:   ( 64, 64, 64)   | Light Gray:(154,161,161)
  Cyan:      ( 46,110,137)    | Purple: (126, 61,181)   | Blue:   ( 46, 58,141)
  Brown:     ( 86, 51, 28)    | Green:  ( 58, 93, 35)   | Red:    (152, 42, 33)
  Black:     ( 21, 21, 21)

Default leather color (undyed): (105, 65, 45) — brown leather

Durable dyeing:
  Dyed leather armor keeps color when repaired in anvil
  Dyed leather can be re-dyed (overwrites previous color)
  No way to remove dye except via commands (/entitydata or /data merge)

Implementation: store color as NBT tag {display:{color:0xRRGGBB}}
  If tag is absent → default brown (105,65,45)
```

### 9H.39 Firework Shape & Effect Modifiers
```
Firework Star crafting: 1 gunpowder + 1 dye + optional modifiers
Shape modifiers (add any 1):
  None (just gunpowder + dye):  Small Ball (default)
  Fire Charge:                  Large Ball
  Feather:                     Burst
  Gold Nugget:                 Star-shaped
  Skull (any):                 Creeper-faced
  Any: no specific shape for "star" — wait, more precisely:
  Modifier    | Shape
  none        | Small Ball
  fire_charge | Large Ball
  feather     | Burst
  gold_nugget | Star
  skeleton_skull, wither_skeleton_skull, zombie_head, creeper_head, player_head, dragon_head → Creeper

Effect modifiers (add any number):
  Glowstone Dust:  Twinkle (sparkle effect after explosion)
  Diamond:         Trail (glowing trail during ascent)

Additional dye:
  Add a second dye of different color → Fade effect
  Star will explode in primary color and fade to secondary color

Firework Rocket crafting:
  Paper + 1-3 gunpowder + 0-7 firework stars (in any slot of crafting grid)
  Flight duration = number of gunpowder (1-3)
  Multiple stars = multiple explosions at peak

NBT structure:
  Firework Star item: {Fireworks:{Explosion:{Type:0, Colors:[I;], FadeColors:[I;],
                          Trail:0b, Flicker:0b}}}
    Type: 0=small, 1=large, 2=star, 3=creeper, 4=burst
    Colors: int array of dye color values
    FadeColors: int array of fade-to colors
    Trail: 1b if diamond added
    Flicker: 1b if glowstone added

  Firework Rocket item: {Fireworks:{Flight:1b, Explosions:[{...}, ...]}}
    Flight: 1-3 (number of gunpowder used)
    Explosions: list of firework star NBT

Flight duration → height:
  1 gunpowder: ~1s flight, explodes at ~20 blocks
  2 gunpowder: ~2s flight, explodes at ~40 blocks
  3 gunpowder: ~3s flight, explodes at ~60 blocks

Elytra boosting:
  Firework rocket can be used during elytra flight (right-click with rocket in hand)
  Boosts forward speed by ~30 m/s per explosion
  Damage: 4 HP per rocket (regardless of gunpowder count)
  Can chain rockets (pop another before first boost ends)
```

### 9H.40 Sheep Wool Color Inheritance
```
Breeding two sheep produces a lamb with color determined by parents:
  Both same color → lamb is that color
  Different colors → lamb is the "result" of combining dye colors
    (same rules as dye mixing from 9H.29)
    Examples:
      Red + Yellow → Orange lamb
      Red + Blue → Purple lamb
      Blue + White → Light Blue lamb
      Black + White → Gray lamb
      Gray + White → Light Gray lamb
      Red + White → Pink lamb
      Blue + Green → Cyan lamb
      Green + White → Lime lamb
      Magenta from mixing: Purple + Pink, or Red + Blue + Pink

  Special: if either parent has a color not achievable by mixing (e.g. from commands),
    lamb inherits one parent's color at random (50/50)

Shearing:
  White sheep drop 1-3 white wool
  Colored sheep drop 1-3 wool of their color
  After shearing, sheep regrow wool by eating grass (block → dirt underneath)
  Regrow time: ~2-3 minutes of eating grass

Dyeing sheep:
  Right-click sheep with dye → permanently changes sheep's wool color
  Dyed sheep will pass that color to lambs
  Dyeing is consumed on use
```

### 9H.41 Complete Spawn Egg List (1.12.2)
```
Spawn eggs for all 1.12.2 mobs (65 eggs):
  Item ID                           | Mob
  minecraft:spawn_egg{Damage:0}     | (use damage value for entity type)

  Damage | Entity
  10     | Chicken
  11     | Cow
  12     | Mooshroom
  13     | Pig
  14     | Sheep
  15     | Bat
  16     | Squid
  17     | Wolf
  18     | Ocelot
  19     | Horse variants (horse, donkey, mule, zombie horse, skeleton horse)
  20     | Rabbit
  21     | Llama
  22     | Parrot (1.12 NEW)
  30     | Villager
  31     | Iron Golem
  32     | Snow Golem
  34     | Skeleton
  35     | Wither Skeleton
  36     | Stray (1.11+)
  37     | Zombie
  38     | Zombie Villager
  39     | Husk (1.10+)
  40     | Creeper
  41     | Spider
  42     | Cave Spider
  43     | Enderman
  44     | Silverfish
  45     | Slime
  46     | Magma Cube
  47     | Ghast
  48     | Blaze
  49     | Witch
  50     | Guardian
  51     | Elder Guardian
  52     | Shulker
  53     | Endermite
  54     | Vex
  55     | Vindicator
  56     | Evoker
  57     | Illusioner (1.12 NEW, survival-inaccessible)
  58     | Wither (boss)
  59     | Ender Dragon (boss)
  61     | Skeleton Horse
  62     | Zombie Horse
  63     | Donkey
  64     | Mule

  Note: Damage values correspond to entity "type" IDs in 1.12.2.
  The SpawnData NBT tag can also specify {EntityTag:{...}} for custom spawns.
  Spawn egg itself: {id:"minecraft:spawn_egg", Damage:<type>, Count:1}
  Display name: translated via item.name tag or lang key "entity.<namespace>.<name>.name"
```

### 9H.42 Mob Equipment Spawning Details
```
Equipment chances by difficulty (per equipped slot):
  Easy:   0% chance of any armor/weapon
  Normal: regional_difficulty × 0.15 (0-15% per slot)
  Hard:   regional_difficulty × 1.00 (0-100% per slot)

  Each slot rolled independently: helmet, chestplate, leggings, boots, mainhand, offhand

Armor tier selection (when a slot is equipped):
  Regional difficulty 0.0-0.75:   leather/chainmail only
  Regional difficulty 0.75-1.0:   iron/gold possible
  Regional difficulty 1.0:        diamond possible (rare)

  Exact distribution per tier:
    Leather:  66.7% (no RD bonus)
    Chainmail: 33.3%
    Iron:      appears at RD 0.5+, replaces ~20% of leather
    Gold:      appears at RD 0.75+, ~10% of chainmail
    Diamond:   appears at RD 1.0, ~3% of iron

Enchantment chance (per equipped item):
  Regional difficulty 0-1: chance = RD × 0.5 (0-50%)
  Enchantment level: 5 + floor(RD × 18) → 5-23
  Enchantments: protection, sharpness, power, etc. (random compatible enchantment)

Sword chance (zombies):
  Zombie: 0-100% (RD based), iron sword if equipped
  Skeleton: 100% bow (always spawns with bow)

Zombie reinforcement:
  When a zombie is hurt, chance to call reinforcements:
    Easy:     0%
    Normal:   0-7.5% (RD × 0.075)
    Hard:     0-15% (RD × 0.15)
  Reinforced zombie spawns within 15 blocks (up to 75% chance chain)

Villager conversion:
  Zombie kills villager → zombie villager
    Easy:     0% chance
    Normal:   50% chance
    Hard:     100% chance
  Zombie villager can be cured: weak potion + golden apple → 2-5 min conversion time
```

### 9H.43 Loot Table JSON Format (Complete)
```
Loot tables stored at: data/<namespace>/loot_tables/<path>.json
Loaded at world start, used by chests, mobs, blocks, fishing, gifts, etc.

Top-level structure:
  {
    "type": "minecraft:chest",         // or: entity, fishing, gift, advancement_reward, command
    "pools": [                         // array of pool objects
      {
        "rolls": {"min": 2, "max": 8}, // number of rolls (int or {min,max})
        "bonus_rolls": 0,              // extra rolls per luck level
        "entries": [
          {
            "type": "minecraft:item",  // item, loot_table, empty, dynamic, tag
            "name": "minecraft:diamond",
            "weight": 1,               // relative probability (default 1)
            "quality": 0,              // weight bonus per luck level
            "functions": [             // item modifier functions (optional)
              {
                "function": "minecraft:set_count",
                "count": {"min": 1, "max": 3}
              },
              {
                "function": "minecraft:enchant_randomly",
                "enchantments": ["minecraft:sharpness"]
              },
              {
                "function": "minecraft:set_damage",
                "damage": {"min": 0.15, "max": 0.8}  // durability %
              },
              {
                "function": "minecraft:set_nbt",
                "tag": "{display:{Name:'{\"text\":\"Special Sword\"}'}}"
              },
              {
                "function": "minecraft:furnace_smelt"  // smelt if applicable
              },
              {
                "function": "minecraft:looting_enchant",
                "count": {"min": 0, "max": 1}
              }
            ]
          }
        ],
        "conditions": [                // conditions for this pool
          {
            "condition": "minecraft:random_chance",
            "chance": 0.75             // 75% chance this pool activates
          },
          {
            "condition": "minecraft:killed_by_player"  // must be killed by player
          },
          {
            "condition": "minecraft:random_chance_with_looting",
            "chance": 0.025,
            "looting_multiplier": 0.01
          },
          {
            "condition": "minecraft:entity_properties",
            "entity": "this",          // "this", "killer", "player", "killer_player"
            "properties": {
              "on_fire": true
            }
          },
          {
            "condition": "minecraft:match_tool",
            "predicate": {
              "item": "minecraft:shears",
              "enchantments": [{"enchantment": "minecraft:silk_touch", "levels": {"min": 1}}]
            }
          },
          {
            "condition": "minecraft:table_bonus",
            "enchantment": "minecraft:fortune",
            "chances": [0.33, 0.50, 0.66, 0.75]  // per fortune level (0,1,2,3)
          },
          {
            "condition": "minecraft:inverted",
            "term": { "condition": "minecraft:random_chance", "chance": 0.2 }
          }
        ]
      }
    ]
  }

Entry types:
  "minecraft:item"        — regular item
  "minecraft:loot_table"  — reference another loot table (sub-table)
    {"type":"loot_table","name":"minecraft:chests/simple_dungeon","weight":1}
  "minecraft:empty"       — nothing (used to pad weight)
  "minecraft:dynamic"     — dynamic entry (drops based on block state, e.g. colored bed)
  "minecraft:tag"         — drop from item tag (1.13+, not in 1.12.2)

Available functions:
  set_count, set_damage, set_nbt, furnace_smelt, looting_enchant,
  enchant_with_levels (enchant at specific level range),
  enchant_randomly (random enchant on item),
  set_attributes, set_contents (for containers),
  exploration_map, fill_player_head

Available conditions:
  random_chance, random_chance_with_looting, kourced_by_player,
  entity_properties, match_tool, table_bonus, inverted,
  alternative (OR), location_check, weather_check,
  entity_scores, light_level, survives_explosion
```

### 9H.44 Dispenser Behavior Table
```
When a dispenser is activated (redstone pulse), it fires one item from its
inventory (random slot order, starting from slot 0) using the following rules:

  Item Type              | Behavior
  Arrow                  | Fired as arrow entity (speed ~3.0, gravity 0.05)
  Spectral Arrow         | Fired as spectral arrow (glows target)
  Tipped Arrow           | Fired as tipped arrow (applies potion effect)
  Egg                    | Thrown as egg entity (chance to spawn chicken)
  Snowball               | Thrown as snowball entity
  Splash Potion          | Thrown as splash potion
  Lingering Potion       | Thrown as lingering potion
  Water Bottle           | Thrown, breaks on hit (no water placement)
  Fire Charge            | Shoots fireball in facing direction (ignites)
  Flint & Steel          | Places fire on block in front (costs 1 durability)
  Bucket (empty)         | Picks up water/lava source in front → water/lava bucket
  Water Bucket           | Places water source in front (if air block)
  Lava Bucket            | Places lava source in front (if air block)
  Powder Snow Bucket     | Not in 1.12.2
  Bone Meal              | Applies bonemeal effect to block in front (if growable)
  Shears                | Shears sheep in front (costs 1 durability)
  Saddle                | Equips saddle on pig/horse/donkey/mule in front
  Spawn Egg             | Spawns mob in front
  Armor                 | Equips armor onto player/mob/armor stand in front
  Minecart              | Places minecart on rails in front
  Chest Minecart        | Places chest minecart on rails
  Furnace Minecart      | Places furnace minecart on rails
  TNT Minecart          | Places TNT minecart on rails
  Hopper Minecart       | Places hopper minecart on rails
  Boat                  | Places boat on water in front
  TNT                   | Primes TNT (sets Fuse:80) in front
  Flint                 | No effect (not implemented as a dispenser action)
  Glass Bottle          | Collects water from cauldron in front → water bottle
  Glass Bottle (empty)  | Same as above — collects water from cauldron
  Dragon's Breath       | Collects dragon breath from end gateway? No — from lingering potion cloud
  Firework Rocket       | Fires rocket in direction (flight + explosion)
  Glass                 | Places glass block? No — placed as block only if item is a block
  All other blocks      | Places block in front (if air/replaceable)
  All other items       | Ejects item as entity (dropped item)

Dispenser vs Dropper:
  Dispenser: actively uses items (shoots, places, equips)
  Dropper: always ejects item as dropped item entity (no special behavior)
  Both have 9 slots (3×3), same GUI

Dispenser face detection:
  Faces the direction it's placed (use blockstate facing property)
  "Front" = the block directly in front of the dispenser face
  If front is not air (or replaceable), item is dropped as entity
```

### 9H.45 Block Entity NBT Structures
```
Block entities (tile entities) persist NBT data per block. Key types:

Chest / Trapped Chest / Ender Chest:
  {id:"minecraft:chest", Items:[{Slot:0, id:"minecraft:diamond", Count:1, Damage:0}, ...],
   LootTable:"minecraft:chests/simple_dungeon", LootTableSeed:12345}
  Items: list of slot→item mappings, max 27 slots (single) or 54 (double)
  Double chest: second chest has paired tag, both share inventory
  LootTable: only present if unopened; on first open, generates loot and clears this tag

Furnace:
  {id:"minecraft:furnace", Items:[{Slot:0, ...}, {Slot:1, ...}, {Slot:2, ...}],
   BurnTime:200, CookTime:150, CookTimeTotal:200}
  Slot 0 = input (smelting), Slot 1 = fuel, Slot 2 = output
  BurnTime: ticks remaining for current fuel (0 = not lit)
  CookTime: ticks of current smelting progress (0 = idle)
  CookTimeTotal: ticks needed for complete smelt (always 200)
  Recipes: {id:"minecraft:iron_ingot"} — only for furnace minecart? No, furnace uses recipe ID in 1.12.2

Sign:
  {id:"minecraft:sign", Text1:'{"text":"Line 1"}', Text2:'{"text":"Line 2"}',
   Text3:'{"text":"Line 3"}', Text4:'{"text":"Line 4"}'}
  All 4 lines as JSON text components (max 15 chars per line in 1.12.2)
  Sign color: base color is black text on brown background
  Dye-able text: uses dye color as text color (right-click with dye)
  Glowing text: use glow ink sac (1.13+), not in 1.12.2

Enchanting Table:
  {id:"minecraft:enchanting_table"}
  No persistent NBT data (random seed generated per interaction)

Brewing Stand:
  {id:"minecraft:brewing_stand", Items:[{Slot:0, ...}, {Slot:1, ...}, {Slot:2, ...}, {Slot:3, ...}, {Slot:4, ...}],
   BrewTime:400, Fuel:20}
  Slot 0-2 = bottle slots (top 3), Slot 3 = ingredient (top center), Slot 4 = fuel (blaze powder, left)
  BrewTime: ticks remaining (400 = 20s full brew)
  Fuel: remaining operations (1 blaze powder = 20)

Beacon:
  {id:"minecraft:beacon", Levels:4, Primary:1, Secondary:3, PaymentItem:{id:"minecraft:diamond", Count:1, Damage:0}}
  Levels: pyramid tier (1-4)
  Primary/Secondary: effect IDs (see P11)
  PaymentItem: last payment item (for GUI display)

Command Block:
  {id:"minecraft:command_block", Command:"say Hello", SuccessCount:0, TrackOutput:1b,
   LastOutput:{text:"[15:30:00] Hello"}, LastExecution:0L, UpdateLastExecution:0b,
   powered:0b, auto:0b, conditionMet:0b}
  Command: the command string
  SuccessCount: number of successes from last execution
  TrackOutput: whether to store output
  LastOutput: JSON text of last output
  LastExecution/UpdateLastExecution: 1.12 NEW tags
  Types: impulse (normal), repeating, chain (via block state / block ID)

Hopper:
  {id:"minecraft:hopper", Items:[{Slot:0, ...}, {Slot:1, ...}, {Slot:2, ...}, {Slot:3, ...}, {Slot:4, ...}],
   TransferCooldown:0, Enabled:1b, LootTable:""}
  5 slots (0-4)
  TransferCooldown: ticks until next transfer (0 = ready, set to 8 on transfer)
  Enabled: 1b = active, 0b = locked by redstone signal
  LootTable: if set, generates loot on first transfer

Dispenser / Dropper:
  {id:"minecraft:dispenser", Items:[{Slot:0, ...}, ...{Slot:8, ...}], LootTable:""}
  9 slots (0-8, 3×3 grid)

Mob Spawner:
  {id:"minecraft:mob_spawner", Delay:200, MinSpawnDelay:200, MaxSpawnDelay:800,
   SpawnCount:4, MaxNearbyEntities:6, RequiredPlayerRange:16, SpawnRange:4,
   EntityId:"minecraft:zombie", SpawnData:{},
   SpawnPotentials:[{Weight:1, Entity:{id:"minecraft:zombie"}},
                    {Weight:1, Entity:{id:"minecraft:zombie_villager"}}]}
  Delay: ticks until next spawn (countdown)
  MinSpawnDelay/MaxSpawnDelay: random interval between spawns
  SpawnCount: number of mobs per spawn
  MaxNearbyEntities: cap on nearby similar entities
  RequiredPlayerRange: player distance to activate (16 blocks)
  SpawnRange: radius around spawner for mob placement
  EntityId: current mob type being spawned
  SpawnData: additional NBT to apply to spawned mob
  SpawnPotentials: weighted list of possible mobs (for randomized spawners)

Jukebox:
  {id:"minecraft:jukebox", RecordItem:{id:"minecraft:record_13", Count:1, Damage:0}}
  RecordItem: the music disc currently inserted (empty if no disc)
  Plays music while RecordItem is present
  Comparator output: 0=empty, 1=13, 2=cat, 3=blocks, 4=chirp, 5=far, 6=mall,
    7=mellohi, 8=stal, 9=strad, 10=ward, 11=11, 12=wait
  Ejects disc when broken or powered (redstone pulse to jukebox)

Note Block:
  {id:"minecraft:noteblock", note:12, powered:0b}
  note: 0-24 (F#3=0, G3=1, ... F#5=24, two octaves)
  powered: whether currently receiving redstone signal

Flower Pot:
  {id:"minecraft:flower_pot", Item:{id:"minecraft:red_tulip", Count:1, Damage:0}}
  Item: the plant inside the pot (empty if no plant, i.e. id is air)
  Valid plants: all small flowers, saplings, mushrooms, cactus, bamboo (1.13+), dead bush, fern, etc.

Skull:
  {id:"minecraft:skull", SkullType:3, Owner:{Id:"uuid", Name:"PlayerName", Properties:{textures:[{Value:"base64"}]}},
   Rot:0, NoteBlockSound:""}
  SkullType: 0=skeleton, 1=wither skeleton, 2=zombie, 3=player, 4=creeper, 5=dragon
  Owner: player skull data (with textures for skin)
  Rot: rotation (0-15)
  NoteBlockSound: custom sound when placed on note block

Piston (moving):
  {id:"minecraft:piston_extension", blockId:"minecraft:stone", blockData:0,
   facing:4, extending:0b, progress:0.0f}
  Created when piston is extending/retracting (the moving block entity)
  blockId: the block being pushed
  facing: direction of movement (0=down, 1=up, 2=north, 3=south, 4=west, 5=east)
  extending: 1b = extending, 0b = retracting
  progress: 0.0→1.0 animation progress

End Gateway:
  {id:"minecraft:end_gateway", Age:0, ExactTeleport:0b, ExitPortal:{X:0, Y:64, Z:0}}
  Age: age since creation (used for beam rendering)
  ExactTeleport: if 1b, teleport to exact ExitPortal position instead of searching
  ExitPortal: target portal position (default: ~1000 blocks out in End)

Structure Block:
  {id:"minecraft:structure_block", mode:"SAVE", name:"", posX:0, posY:0, posZ:0,
   sizeX:0, sizeY:0, sizeZ:0, mirror:"NONE", rotation:"NONE",
   ignoreEntities:0b, showboundingbox:1b, powered:0b, integrity:1.0f, seed:0L}
  Modes: "SAVE", "LOAD", "CORNER", "DATA"
  name: structure identifier (namespace:path)
  posX/Y/Z: offset from structure block corner
  sizeX/Y/Z: dimensions of area to save/load
  mirror: "NONE", "LEFT_RIGHT", "FRONT_BACK"
  rotation: "NONE", "CLOCKWISE_90", "CLOCKWISE_180", "COUNTERCLOCKWISE_90"
  integrity: 0.0-1.0 (block integrity for load)
  seed: random seed for integrity decay
```

### 9H.46 Farmland & Crop Mechanics
```
Farmland creation:
  Till grass/dirt with hoe (right-click) → farmland
  Farmland block: {moisture:0-7}
  Moisture level increases from water within 4 blocks (Manhattan distance, same Y level)
  Water source: water blocks or waterlogged blocks within range
  Hydration range: 4 blocks horizontally, at same Y level
  Farmland decay: without water, moisture decreases by 1 per random tick (3/4096 per tick)
  Untilled: if no water and moisture = 0, farmland turns to dirt when random ticked (2/4096 chance)
  Jumping/trampling: entity jumping on farmland → turns to dirt (if jump height > 0.5 blocks)
    - Player falling onto farmland → turns to dirt
    - Player walking on farmland → NO trampling
    - Other entities: same rules (trampling is jump/fall based)

Crop growth algorithm (per random tick):
  Growth chance depends on:
    - Light level ≥ 8 in block above crop (9+ for fully lit)
    - If light < 8, crop does NOT grow
    - If light ≥ 8: base chance = 1.0
    - Hydration bonus: farmland moisture > 0 → 4× growth speed
    - Growth stage increment chance per tick:
      - Dry farmland: growth chance = 1/3 (33.3% per growth attempt)
      - Hydrated farmland: growth chance = ~3/4 (75% per growth attempt)
  In 1.12.2, crops have specific growth stages:

  Wheat (7 stages, 0-6):
    Stage 0: seeds planted
    Stage 1-2: sprouting (small green shoots)
    Stage 3-4: mid-growth (taller stems)
    Stage 5-6: nearly mature (yellowish)
    Stage 7: fully grown (harvestable)
    Light requirement: 9+ for growth (or 8 for survival)
    Bone meal: advances by 2-4 stages

  Carrots (7 stages, 0-7):
    Same stage progression as wheat
    Stage 7 = fully grown (multiple carrots visible)
    Bone meal: advances by 2-4 stages

  Potatoes (7 stages, 0-7):
    Same as carrots
    Stage 7 = fully grown
    Bone meal: advances by 2-4 stages
    Drop chance: 1-4 potatoes (poisonous potato ~2% drop chance)

  Beetroot (4 stages, 0-3):
    Stage 0-2 = growing
    Stage 3 = fully grown
    Bone meal: advances by 1-2 stages
    Drops: 1 beetroot + 0-3 seeds

  Melon/Pumpkin Stem (7 stages, 0-7):
    Stage 0-6 = growing stem (bends as it grows)
    Stage 7 = mature stem (produces fruit on adjacent dirt/grass)
    Stem needs adjacent air block to place fruit
    Melon stem: fruit grows adjacent, up to 3 melons on same stem
    Pumpkin stem: same but only 1 pumpkin at a time
    Growth direction: random adjacent block
    Bone meal: advances by 2-4 stages

  Nether Wart (3 stages, 0-3):
    Only grows on soul sand
    Stage 0 = planted, 1-2 = growing, 3 = harvestable
    Bone meal: does NOT work on nether wart
    Growth rate: light independent (no light required)
    Drop: 2-4 nether wart at stage 3

  Cocoa (3 stages, 0-2):
    Only placed on jungle log side
    Stage 0 = small, 1 = medium, 2 = large (harvestable)
    Bone meal: advances by 1 stage
    Drop: 3 cocoa beans at stage 2, 1 at stage 0-1

  Sugar Cane (max 3-4 blocks tall):
    Grows 1 block every 16 random ticks (average)
    Requires water adjacent to the block below
    Grows to max height of 3 (bedrock) but can grow to 4 with bone meal? No, bone meal doesn't work on sugar cane pre-1.13... Actually bone meal works on sugar cane in 1.12.2? Let me check: bone meal does NOT work on sugar cane in 1.12.2 (added in 1.13). Sugar cane grows naturally only.

  Cactus (max 3 blocks tall):
    Grows 1 block every 16 random ticks
    Requires air above, no adjacent solid blocks (except cardinal directions)
    Damages entities touching it (1 HP/0.5s)

Bone meal behavior per crop:
  Wheat/Beetroot/Carrots/Potatoes: advances 2-4 stages
  Melon/Pumpkin stem: advances 2-4 stages
  Cocoa: advances 1 stage
  Nether wart: NO effect
  Sugar cane: NO effect in 1.12.2
  Grass block: generates tall grass/flowers above
  Sapling: advances growth (random chance to grow into tree)
  Mushroom: generates giant mushroom in same dimension
  Kelp/Sea pickle: not in 1.12.2
```

### 9H.47 Piston Block Push/Break List
```
Piston can push up to 12 blocks in a line (including the block directly in front).
Sticky piston pulls back 1 block on retract.

Blocks that are PUSHED by pistons:
  All full solid blocks: stone, dirt, planks, cobble, bricks, etc.
  Slabs (half), stairs (in some orientations), glowstone, glass, sea lantern
  Note blocks, jukeboxes, chests, furnaces, dispensers, droppers
  Redstone blocks, obsidian (yes, pistons can push obsidian — but not pull it)
  Enchanting tables, anvils, ender chests (yes, pushable)
  Beacons, spawners (pushable but NOT pullable by sticky pistons)
  Ice, packed ice, blue ice (1.13+)
  Sponge, clay, wool, carpets (yes, carpets are pushable)
  Beds (both halves move as one unit)
  Shulker boxes (move with contents, can be pushed/pulled)

Blocks that BREAK when pushed (drop as item):
  Torches, redstone torches       — break
  Redstone dust, repeaters, comparators — break
  Levers, buttons, pressure plates — break
  Trapdoors, doors, fence gates    — break (only if attached to moved block)
  Ladders, vines, signs            — break
  Flower pots, plants, crops       — break
  Rails (all types)                — break
  Tripwire hooks, tripwire         — break
  End rods, chorus flower/plant    — break
  Snow layers (≤ 1)                — break (higher layers pushed)
  Heads/skulls                     — break
  Item frames, paintings           — break (drop as entity)
  Banners                          — break

Blocks that CANNOT be pushed (immovable):
  Bedrock, obsidian, end portal frame, end portal, end gateway — immovable
  Command blocks, structure blocks, barrier, structure void — immovable
  Chests? No — chests ARE pushable. But double chests: both halves must move together
    Actually double chests cannot be pushed by pistons (only single chests)
  Furnace: pushable
  Spawner: pushable but not pullable
  Beacon: pushable
  Enchanting table: pushable
  Anvil: pushable
  Piston head / moving piston block: immovable (piston cannot push another piston?
    Actually extended pistons cannot be pushed. Retracted pistons can be pushed.)

Blocks with special sticky piston behavior:
  Slime block: sticks to adjacent blocks, pulls all stuck blocks on retract
    - If slime block is pulled, any block stuck to it also moves (chain reaction up to 12 total)
  Honey block (1.13+): not in 1.12.2
  Glue blocks? No — only slime block is sticky

Piston extension time: 2 ticks (0.1s) for extension, 2 ticks for retraction
Animation: block moves linearly over 2 ticks (smooth transition)
During movement: block entity (piston_extension) holds the moving block state
```

### 9H.48 Comparator Signal Strengths
```
Redstone comparator outputs signal strength proportional to container fullness.
Output: 0-15, where 15 = full stack × slot count

Formula:
  signal = floor(1 + (sum_of_item_counts × 14) / (max_single_stack_size × number_of_slots))
  Or simplified: signal = floor(1 + fullness_ratio × 14)
  Empty container = 0 signal

Container → max signal (when full):
  Slot count (for reference):
  Container              | Slots | Max/Stack | Signal 15 at
  Chest (single)         | 27    | 64        | 27 stacks (1728 items)
  Chest (double)         | 54    | 64        | 54 stacks (3456 items)
  Trapped Chest          | 27    | 64        | 27 stacks
  Ender Chest            | 27    | 64        | 27 stacks
  Shulker Box            | 27    | 64        | 27 stacks
  Dispenser/Dropper      | 9     | 64        | 9 stacks (576 items)
  Hopper                 | 5     | 64        | 5 stacks (320 items)
  Furnace                | 3     | 64        | 3 stacks (192 items)
  Brewing Stand          | 5     | 64        | 5 stacks (bottles + ingredient + fuel)
                         |       |           | Note: each bottle counts as 1 item
  Beacon                 | 1     | 64        | 1 stack (payment item, 64 items)
  Minecart Chest         | 27    | 64        | same as single chest
  Minecart Hopper        | 5     | 64        | same as hopper
  Jukebox                | 1     | 1         | 1 = has disc, 0 = empty
  Command Block          | —     | —         | Always output 0? No — SuccessCount output
  Flower Pot             | —     | —         | 0 = empty, 1 = any plant (treated as 1 slot, 1 item)
  Cauldron               | 3     | 1/level   | Output = water level (0, 1, 2, 3)
  End Portal Frame       | —     | —         | 15 if eye inserted, 0 if not
  Daylight Detector      | —     | —         | 0-15 based on sky light (inverted mode)

  Actual signal formula:
    fullness = sum(item_count / max_stack_size) over all slots
    signal = min(15, floor(1 + fullness * 14 / slot_count))
    Example: Double chest with 10 stacks of 64 cobble:
      fullness = 640/64 = 10 slot-equivalents
      signal = floor(1 + 10 * 14 / 54) = floor(1 + 2.59) = 3

Comparator modes:
  Front torch off = Compare mode: output = input_signal (signal A ≥ signal B)
    - Outputs power from back if side signal ≤ back signal
    - If side signal > back signal: output = 0
  Front torch on = Subtract mode: output = back_signal - side_signal
    - Subtracts side power from back power
    - Output = max(0, back - side)
```

### 9H.49 Hopper Transfer Mechanics (Detailed)
```
Transfer rules:
  Pull: hopper pulls items from container/block above it
  Push: hopper pushes items to container/block below or in front of it
  Speed: 1 item per 4 game ticks (0.2 seconds), 5 items per second max

Pull behavior (from above):
  Every 4 ticks (when TransferCooldown = 0):
    - Check block directly above hopper
    - If it's a container (chest, furnace, etc.): try to take 1 item from any slot
    - If it's a full block (stone, dirt, etc.): no pull
    - If it's an item entity above the hopper (within 1 block height): suck 1 item
    - Item entity suction range: 1.5 blocks above hopper center, 1.5 block radius cylinder
    - Suction also applies to items on top of hopper (same range)
  Pull priority: first container slot with items (slot 0 upward)
  Pull single item: takes 1 item from a stack (reduces stack count by 1)

Push behavior (to below/side):
  Every 4 ticks (when TransferCooldown = 0):
    - Check container below hopper (or in direction of pointing — hopper faces down by default)
    - If container has matching item with room: add 1 item to that stack
    - If container has empty slot: place 1 item in empty slot
    - If no valid destination: do nothing (item stays in hopper)
  Hopper pointing direction: use blockstate "facing" property
    - Down = push to container below
    - North/South/East/West = push to container on that side

Locking:
  Hopper locked when given redstone signal (powered = true, Enabled = 0b)
  Locked hopper: no pull, no push, no item suction
  Unlocked: normal operation

Cooldown:
  TransferCooldown set to 8 on each successful transfer
  Decrements by 1 each game tick
  When 0: transfer can occur again
  Total: 1 transfer per 8 ticks = 2.5 transfers/second per hopper

Item suction specifics:
  Items on ground within 1.5 blocks above hopper
  Items on top of hopper (same block position)
  Items in water flow: pushed into hopper if flow direction matches
  Items on half-slabs: if slab is above hopper level, suction height varies
  Items in minecart with hopper: suction range is 1 block below cart

Pipes/high-throughput:
  Multiple hoppers can chain (hopper → hopper → chest)
  Hoppers can face sideways into each other (loop prevention: items prefer shortest path)
  Stack of 64 items moves through 5 hoppers in ~4 seconds

Redstone interaction:
  Comparator on hopper: signal = fullness (same formula, 5 slots)
  Hopper above chest: pulls items from chest to hopper (then pushes further down)
  Hopper below furnace: pulls fuel/input/output? Actually hopper below furnace pulls output only
  Hoppers are used for auto-smelters: input → furnace top, fuel → furnace side, output → furnace bottom
```

### 9H.50 Water Item Flow
```
Items dropped in water are affected by water currents:
  Items float upward (buoyancy) in water at ~0.1-0.3 m/s:
    - Items rise until they reach the surface (top water block)
    - Items bob at surface: vertical oscillation (sine wave, ~0.5 amplitude, 1s period)
    - Items can be pushed downward by water flowing down

Water current speed:
  Still water: no horizontal movement, items float in place
  Flowing water: items move at ~1.39 m/s per water level difference
    - Level 7-8 (source): speed ~0 m/s (no flow)
    - Level 1-6: speed increases as level decreases
    - Items accelerate with water flow, stop when hitting solid block

Horizontal push:
  Items are pushed in the direction of water flow (based on block state)
  If item hits a wall, item stops + collects against wall
  Items can be pushed up 1 block if water flows into a block with a step (bob up)

Vertical water flow:
  Water falls downward (flowing down column)
  Items fall with water at ~0.5 m/s downward
  Items accumulate at bottom of waterfall (pool)

Item pickup via hopper:
  Items in water can be sucked into hoppers within range
  Water pushes item → item enters hopper range → hopper pulls item

Item despawn in water:
  Same 6000 tick (5 min) despawn timer as on land
  Timer pauses while item is in loaded chunk? No, timer always ticks
  Timer: Age NBT tag increments each tick; at 6000 → despawn

Item damage:
  Items cannot be destroyed by water (only by fire/lava/explosion/cactus)
  Items in lava: destroyed instantly (unless netherite, not in 1.12.2)
  Items in fire: destroyed after 1 tick
  Items on cactus: destroyed after 1 tick

Item merging:
  Items of same type and damage value within 1 block distance → merge
  Max stack size: 64 (or lower for tools, potions, etc.)
  Merging priority: equalize stacks (prefer balanced sizes)

Lava item destruction:
  Items touching lava source or flowing lava → destroyed (no drop)
  Exception: netherite items (not in 1.12.2)
  Items on ground near lava: destroyed if lava flows over them

Soul sand bubble column (not in 1.12.2 — 1.13+)
Magma block downward bubble (not in 1.12.2)
```

### 9H.51 Potion Color Mixing
```
Potion color is determined by the active effects on the potion.
Each effect type has an associated color (RGB) and a weight.

Color formula:
  total_red = 0, total_green = 0, total_blue = 0, total_weight = 0
  For each active effect on the potion:
    weight = effect's amplifier + 1 (minimum 1)
    total_red   += effect_color.red   × weight
    total_green += effect_color.green × weight
    total_blue  += effect_color.blue  × weight
    total_weight += weight
  final_color = (total_red / total_weight, total_green / total_weight, total_blue / total_weight)

Effect colors (RGB):
  Speed:         (124, 175, 198)  — light blue
  Slowness:      (90, 108, 134)   — gray-blue
  Haste:         (217, 192, 67)   — yellow
  Mining Fatigue:(74, 66, 23)     — dark yellow
  Strength:      (147, 36, 35)    — red
  Instant Health:(248, 36, 35)    — bright red
  Instant Damage:(67, 10, 9)      — dark red
  Jump Boost:    (34, 255, 76)    — green
  Nausea:        (85, 29, 74)     — purple
  Regeneration:  (205, 92, 171)   — pink
  Resistance:    (153, 69, 58)    — brown-red
  Fire Res:      (228, 154, 58)   — orange
  Water Breath:  (46, 112, 215)   — blue
  Invisibility:  (127, 131, 141)  — gray
  Blindness:     (31, 31, 35)     — dark gray
  Night Vision:  (31, 106, 153)   — dark blue
  Hunger:        (88, 118, 43)    — olive
  Weakness:      (72, 77, 72)     — dark gray-green
  Poison:        (78, 147, 49)    — green
  Wither:        (53, 42, 39)     — dark brown
  Health Boost:  (248, 125, 35)   — orange
  Absorption:    (37, 82, 164)    — blue
  Saturation:    (248, 36, 35)    — red
  Glowing:       (148, 160, 97)   — yellow-green
  Levitation:    (206, 255, 255)  — very light cyan
  Luck:          (51, 153, 0)     — green
  Bad Luck:      (192, 164, 0)    — golden

Default water bottle: no effects → transparent (no tint)
Awkward/Mundane/Thick potions: no effects → transparent
Splash/Lingering potion: same color as drinkable variant + slight transparency

Potion item NBT:
  {id:"minecraft:potion", Count:1, Damage:0,
   tag:{Potion:"minecraft:swiftness", CustomPotionEffects:[{Id:1, Amplifier:0, Duration:3600}]}}
  Potion: recipe type ID
  CustomPotionEffects: list of effects (Id, Amplifier, Duration)
  Color can be overridden with {display:{color:0xRRGGBB}}
```

### 9H.52 Experience Orb Mechanics
```
XP orbs are dropped by mobs, ores, and player actions.
They merge together when close and can be picked up by the player.

Orb values:
  Mob drops:        zombie/skeleton 5, creeper 5, spider 5, enderman 5,
                    slime/magma 1-4, ghast 5, blaze 10, guardian 10,
                    elder guardian 10, wither 50, ender dragon 500
  Ore drops:        coal 0-2, diamond 3-7, emerald 3-7, lapis 2-5,
                    redstone 1-5, nether quartz 2-5
  Furnace smelting: iron 0.7, gold 1.0, other 0.1-0.3
  Breeding:         1-7 per successful breeding
  Fishing:          1-6 per catch
  Trading:          varies (0-20 per trade)
  Enchanting:       consumed (not dropped)
  Adv. completion:  varies (30-100 per advancement)

Orb merging behavior:
  Orbs within 1 block of each other → merge into a single orb
  Merged orb value = sum of values, capped at 247
  Merged orb has single sprite, sizes up with value:
    Value 1-3:      tiny (6×6 pixels, green/yellow)
    Value 4-11:     small (9×9 pixels, yellow)
    Value 12-35:    medium (12×12 pixels, yellow)
    Value 36-79:    large (15×15 pixels, green/yellow)
    Value 80-247:   huge (18×18 pixels, green)
  Merging reduces the number of individual entities → performance optimization

Orb motion:
  Orbs float in the air (no gravity? Actually orbs do have gravity but also float slightly)
  Orbs drift toward player within 6 blocks:
    - Within 3 blocks: increases speed toward player
    - Within 1 block: instant pickup
  Orb speed toward player: ~0.3-0.5 m/s (accelerates as distance decreases)
  Orbs in water: float up, still attracted to player
  Orbs on ground: slight bounce (0.1 amplitude)

Orb pickup:
  Player within 1 block → orb is absorbed
  Pickup sound: entity.experience_orb.pickup
  Visual: orb shrinks and disappears on contact
  XP added: orb.value XP points added to player total
  Player XP is tracked in two values:
    - XpTotal: lifetime XP earned (never decreases)
    - XpLevel / XpP: current level + progress toward next level

Orb despawn:
  Orbs despawn after 5 minutes (6000 ticks, same as items)
  Orbs DO despawn in unloaded chunks (same as item entities)

Orb invulnerability:
  Orbs cannot be destroyed by damage (fire, lava, explosion) — they survive
  Orbs in lava: float on top (not destroyed, but can't be picked up)
  Orbs can be moved by water currents (same as items)
  Orbs in creative mode: not picked up (creative players don't get XP from orbs)
```

### 9H.53 Fire Spread Mechanics
```
Fire is spread by fire blocks (ID 51), ignited by flint & steel, fire charges, lightning, or lava.

Fire block properties:
  Age: 0-15 (increments randomly, higher age = more likely to decay)
    - Newly placed fire: age=0
    - Fire decays: when age reaches 15 → block becomes air
    - Age increases by 1 per random tick (3/4096 chance per tick)
    - Age decreases by 1 per random tick when raining on the block above
  Light level: fire provides light 15 when age=0, decreasing as age increases
    Light = 15 - age (but minimum 0)
  Collision: no collision (entities can walk through)
  Damage: 1 HP per 0.5s (10 ticks) standing in fire

Ignition sources:
  Flint & Steel: places fire block on the face clicked (costs 1 durability)
  Fire Charge: places fire block at target location (consumes item)
  Lava: blocks adjacent to lava have 2/3 chance per tick to ignite if flammable
  Lightning: creates fire at strike point (and nearby blocks)
  Explosion: TNT/creeper explosions have 1/3 chance to create fire per destroyed block
  Ghast Fireball: creates fire on impact
  Fire spread: existing fire can spread to adjacent flammable blocks

Flammable blocks list (with spread chance and burn time):
  Block                     | Flammability | Spread Chance | Burn Time (ticks)
  Oak/Spruce/Birch/Jungle   |              |               |
    Planks                  | 5            | 20            | 5
    Logs (oak/spruce/etc)   | 5            | 5             | 5 (damage variant)
    Stairs                  | 5            | 20            | 5
    Slabs                   | 5            | 20            | 5
    Fence                   | 5            | 20            | 5
    Fence Gate              | 5            | 20            | 5
    Door                    | 5            | 20            | 5
    Trapdoor                | 5            | 20            | 5
    Pressure Plate          | 5            | 20            | 5
    Button                  | 5            | 20            | 5
    Sign                    | 5            | 20            | 5
  Oak/Spruce/etc Leaves     | 30           | 60            | 30
  Wool (all 16 colors)      | 30           | 60            | 30
  Carpet (all 16)           | 30           | 60            | 30
  Bookshelf                 | 30           | 20            | 20
  Hay Bale                  | 60           | 20            | 20
  TNT                       | 15           | 100           | 15 (ignites TNT)
  Coal Block                | 5            | 5             | 100
  Dead Bush                 | 60           | 100           | 30
  Fern/Tall Grass           | 60           | 100           | 20
  Vine                      | 15           | 100           | 30
  Wooden tool item          | —            | —             | 200 (as fuel)
  Stick item                | —            | —             | 100 (as fuel)

  Flammability: chance that fire near this block will not be blocked (0=non-flammable)
  Spread Chance: 1/spread_chance per tick to ignite adjacent flammable block
  Burn Time: ticks until fire on this block decays (age increases per tick)

Fire spread algorithm (per random tick, 3/4096 per block):
  1. Check 4 adjacent blocks + 1 above for flammable blocks
  2. For each adjacent flammable block:
     a. Roll spread chance: if random(0, spread_chance) == 0 → ignite
     b. New fire block has age = floor(current_age + random(0,4)) / 2 (rounded up)
     c. Check rain: if block above is exposed to sky and raining → 50% chance to prevent spread
  3. Check below: fire below a flammable block spreads up faster
  4. Fire on netherrack: never decays (age stays 0), burns forever
  5. Fire on magma block: never decays

Fire extinguishing:
  Rain: fire on exposed blocks has 20% chance per random tick to be extinguished
    - Rain extinguishes any fire with age > 0 immediately
    - Rain never extinguishes fire on netherrack
  Water: water flowing into fire block → fire is replaced by air
  No air: fire without adjacent air block suffocates → 50% chance per tick to extinguish
  gamerule doFireTick: if false → fire never spreads, never decays, never extinguishes

Fire damage to entities:
  Entity standing in fire: 1 HP damage per tick (20 damage/sec)
  Entity on fire (after leaving): 1 HP per 0.5s, lasts 3-15 seconds based on source
  Fire Resistance potion: entity takes no fire damage while fire res is active
  Fire Protection enchantment: reduces fire damage by 8% per level
  Burning animation: entity model has flame overlay texture
  Sound: entity on fire emits crackling sound
  Visual: fire particles emit from burning entities
```

### 9H.54 Grass & Mycelium Spread Algorithm
```
Grass Block (ID 2) and Mycelium (ID 110) spread to adjacent Dirt blocks (ID 3).

Spread conditions (checked per random tick, 3/4096 per block):
  Grass spread:
    - Source block must be grass block with light level ≥ 9 above it
    - Target block must be dirt (or coarse dirt/podzol? coarce dirt and podzol do NOT convert)
    - Target must have light level ≥ 9 above it (or light level ≥ 4 with sky exposure ≥ 9)
    - Target must be within 1 block Manhattan distance (4 cardinal directions + below)
    - Target cannot be covered by any opaque block reducing light below threshold
    - Target cannot be under water or lava (waterlogged or source above)
    - Spread range: 1 block horizontal, up to 3 blocks downward (grass can spread to dirt below)
  
  Mycelium spread:
    - Same conditions as grass but: source must be mycelium
    - Target must be dirt
    - Light ≥ 9 above source (not target — mycelium cares about itself)
    - Mycelium only spreads in mushroom island biomes naturally
    - Can spread to adjacent dirt in other biomes if manually placed
  
  Coarse Dirt: does NOT convert to grass or mycelium
    - Must be tilled with hoe → dirt → then grass can spread
    - Or tilled with hoe → grass path? No, hoe on coarse dirt → dirt
  
  Podzol: does NOT convert to grass or mycelium
    - Only found in mega taiga biomes, does not spread
    - Can be obtained with silk touch

Grass to dirt (de-spread):
  Grass block turns to dirt when:
    - Any opaque block is placed above it (light level drops below 4)
    - Light level above grass block < 4 (e.g. covered by blocks)
    - Grass is under water (block above is water source/flowing)
    - Sheeps eating grass: grass block → dirt (temporary, regrows)
    - Player digging: grass block already drops dirt (silk touch needed for grass block item)

Mycelium to dirt:
  Same conditions as grass de-spread (light < 4, opaque block above, water above)
  Mycelium cannot coexist with light-blocking blocks above

Visual:
  Grass block side texture: green overlay depending on biome color
    - Transitions smoothly between colors (vertex blending)
  Mycelium: purple/black particles (townaura) when breaking
    - Particles: mycelium dust (tiny purple specks)

Grass Path block (ID 208):
  Created by right-clicking grass with shovel
  Does NOT revert to dirt unless broken
  Does NOT spread grass (path block is inert)
  Lower collision height: 0.9375 blocks (15/16)
  Player walking on grass path: no speed change
```

### 9H.55 Ice Formation & Melting
```
Ice (ID 79) forms from water in cold biomes or at high altitudes.
Frosted Ice (ID 212) is created by Frost Walker enchantment.

Ice formation conditions:
  Natural ice formation (non-frosted):
    - Water source block (still water, level 0)
    - Temperature at the block's Y level < 0.15
    - Block above must be air (exposed to sky or not)
    - Light level above does NOT matter (ice forms even in bright light)
    - Only forms during nighttime or in permafrost biomes
    - Water block must have been still for at least 1 tick
    - Ice forms instantly: water → ice when conditions met (random tick, 1/4096 per tick)
  
  Temperature at Y level:
    effective_temp = biome_temp - (Y - 64) × 0.0016667
    - At y=64: same as biome temperature
    - At y=128: biome_temp - 0.1067 (colder at high altitude)
    - At y=0: biome_temp + 0.1067 (warmer underground)
    - If effective_temp < 0.15 → water can freeze
    - Rivers, oceans in cold biomes: temperature < 0.15 → surface freezes
    - Swampland: temperature 0.8 → never freezes naturally (even in mountains above snow line)

Frosted Ice (Frost Walker enchantment):
  - Created when player with Frost Walker boots walks on water (source or flowing)
  - Frost Walker I: radius 2 blocks (5×5 area)
  - Frost Walker II: radius 3 blocks (7×7 area)
  - Frosted ice replaces water surface blocks in a circle centered on player
  - Frosted ice age: starts at 0, increments over time
  - Frosted ice melting: age increases, at age=3 → becomes water source
  - Frosted ice lasts ~10-30 seconds depending on light/temperature
  - Frosted ice: light can pass through (translucent)

Ice melting conditions:
  - Light level ≥ 12 (from any source: sun, blocks) → ice melts
    - Most light sources: torch (14), glowstone (15), etc.
    - Sunlight: level 15 during day, level 4 at night
    - Ice near a torch: melts (if light ≥ 12 at ice block)
    - Ice in direct sunlight: melts during day, re-freezes at night
  - Melting tick: ice checks per random tick (3/4096 per tick)
  - Melting: ice block → water source block (spreads)
  - Ice does NOT melt if block above is opaque (light blocked)
  - Ice in inventory: item form does not melt; only placed ice melts
  - Packed Ice (ID 174): never melts (always solid, light passing through)
  - Frosted Ice: automatically melts after short duration (age-based)

Piston pushing ice:
  - Ice can be pushed by pistons
  - When pushed: ice block moves to new position
  - If piston pushes ice into water → ice displaces water? No, water doesn't affect ice
  - Sticky piston can pull ice

Ice properties:
  - Friction: 0.98 (very slippery — entities slide on ice)
  - Transparency: translucent (light passes through, reduced by ~1 level? Actually ice is transparent)
  - Blast resistance: 0.5
  - Hardness: 0.5
  - Tool: none (breaks instantly with any tool, drops nothing unless silk touch)
  - Silk touch: drops ice block itself
  - Water flow: ice does NOT block water flow (water flows under ice?)
    Actually ice is a full solid block: water cannot flow through it
  - Boat on ice: increased speed (ice reduces friction → boats go faster)
```

### 9H.56 Mushroom Spread (Light Restrictions)
```
Brown Mushroom (ID 39) and Red Mushroom (ID 40) spread to adjacent blocks.

Placement requirements:
  - Can be placed on any block with a solid top surface (full block or top slab)
  - Light level < 13 (mushrooms cannot be placed in bright light)
    - Exception: can be placed at any light level in mushroom island biomes
    - Exception: can be placed in Nether at any light level
  - Can be placed on mycelium, podzol, or nylium (1.13+) regardless of light level
  - Can be placed on dirt/grass at low light levels

Spread mechanics:
  - Mushroom attempts to spread to adjacent blocks (4 cardinal directions + same block if broken?)
  - Spread range: up to 5 blocks away horizontally, and 3 vertically (random)
  - Spread check: per random tick (3/4096 per block)
  - Target block must be:
    - Same mushroom type (brown spreads to brown, red to red)
    - Target area has light level < 13 (or in mushroom island biome)
    - Target is a valid placement surface (solid top face)
    - No mushroom of same type already at target
  - Spread chance: 1/25 per spread attempt (4% chance per tick when valid)
  - In mushroom island biome: spread chance increased to 1/15 (6.7%)

Giant mushrooms:
  - Bone meal on a mushroom: if space permits, grows into giant mushroom
    - Red mushroom → giant red mushroom (mushroom block + stem)
    - Brown mushroom → giant brown mushroom
  - Giant mushroom size: 7 blocks tall (cap top at y+6)
  - Requirements:
    - Must be on dirt/grass/mycelium/podzol
    - Space: 7×7×6 blocks free above mushroom (cap requires 3×3 area)
    - Light: no restriction for giant mushroom growth
    - Mushroom island: giant mushrooms grow naturally (trees with mushroom caps)
  - Giant mushroom blocks drop mushrooms when broken (unless silk touch)

Mushroom block entities:
  - Huge mushroom blocks (ID 99 for brown, ID 100 for red)
  - Block states: stem (all sides), cap (top/side), pores (all sides, interior)
  - Stem: has bark texture on all sides
  - Cap: top texture is cap (brown or red), sides are pores
  - Stem side texture: hard brown/white striped
  - When broken: drops 0-2 mushrooms of corresponding type
  - Silk touch: drops block itself

Mushroom farming:
  - Can grow in dark caves (light level 0-12)
  - Can spread across mycelium farms in mushroom island biomes
  - Most efficient: dark room with mycelium floor, spaced ~3 blocks apart
  - Bone meal: causes rapid spread within the 5-block radius
  - Harvest: break mushrooms with any tool (instant, no tool required)
```

### 9H.57 Vine Growth
```
Vines (ID 106) grow downward from the bottom of a supported block.
Can be placed by right-clicking a block face with vines item.

Placement mechanics:
  - Vines can be placed on any solid block face (facing direction)
  - Can be placed on any of 4 horizontal faces (north, south, east, west)
  - Also can be placed on top of a block (grows downward from there)
  - Initial placement: one vine block at the clicked location

Growth algorithm (per random tick, 3/4096 per block):
  1. Check below current vine block:
     - If block below is air → can grow downward
     - Growth chance: 50% per random tick (1/2)
     - Create new vine block below with same facing direction(s)
     - New vine inherits all facing properties from parent
  2. Growth limit:
     - Vines grow to max ~7 blocks long from the source
     - Beyond 7 blocks: no further growth (hard limit in 1.12.2)
     - Can exceed 7 if placed manually (player can stack longer)
  3. Side growth:
     - Vines can spread laterally if adjacent to a solid block
     - If vine block has no adjacent solid block on its facing side → breaks
     - Spread range: 1 block horizontally from existing vine
     - New side vine: attaches to new solid block face

Vine breaking conditions:
  - No adjacent solid block on the facing direction → vine breaks (drops item)
  - Water flowing into vine block → vine breaks (drops item) 
  - Piston pushing vine → vine breaks
  - Block above vine is broken → vine above ground level falls? No, only topmost vine breaks
  - Vine facing support block is removed → vine breaks

Climbing:
  - Player can climb vines by pressing jump + move into vine
  - Climbing speed: same as ladder (0.12 blocks/tick upward? Actually same as ladder speed)
  - Vine does NOT slow horizontal movement (ladder does? Actually both slow slightly)
  - Player can hold space to climb up vines
  - Player can sneak to descend vines (press shift)
  - Vines provide collision box only in the direction faced (can walk through from other sides)

Vine facing states:
  - Vines can face up to 4 directions simultaneously (corner wrap) via blockstate
  - Blockstate properties: north=true/false, south=true/false, east=true/false, west=true/false
  - If only one face: vine appears as flat sheet on that face
  - If two adjacent faces: vine appears as corner (wraps around)
  - If all four faces: vine appears as full column (overgrown look)
  - Vines on a single post: wrap around all 4 sides = full block coverage

Shears: vine blocks can be collected with shears (drop vine item)
  - Without shears: vine → nothing (breaks, no drop)
  - With shears: vine block drops as item

Vine color:
  - Uses biome grass color (temperature/humidity map)
  - Color is biome-dependent (darker green in jungle, lighter in plains)
```

### 9H.58 Leaves Decay Algorithm
```
Leaves decay when they are no longer connected to a log block within range.
Applies to all 6 leaf types: oak, spruce, birch, jungle, acacia, dark oak.

Decay distance algorithm (checked per random tick, 1/20 chance per leaf block per tick):
  1. Determine "tree" connection:
     - Leaf block scans for nearest log block within radius
     - Radius = 4 blocks horizontally, 4 blocks vertically (Manhattan distance = 6?)
     - Actually: leaf checks for log within a 4-block Manhattan distance
     - If a log block is found within "4" blocks → leaf does NOT decay
     - If no log within range → leaf is scheduled for decay

  2. The distance check:
     - Uses blockstate properties: "distance" and "check_decay"
     - "distance": 1-7 (distance to nearest log)
     - "check_decay": true/false (whether to check for log connectivity)
     - When a log is broken or leaves are placed:
       - Nearby leaves get check_decay=true
       - Leaf distance is recalculated

  3. Exact algorithm from 1.12.2:
     a. When leaf block is placed or neighbor changes:
        - distance = 1 + min(distance of adjacent leaves with same log connection)
        - If distance > 4 (or > 6 in some implementations): set check_decay=true
        - If adjacent log: distance = 1
     b. Per random tick (1/20 chance per tick):
        - If check_decay = true:
          - Calculate minimum distance to a log block
          - Range: scan 4 blocks in all directions (Manhattan distance)
          - If distance ≤ 4: update leaf distance, set check_decay=false
          - If distance > 4: schedule decay (will break after brief delay)
          - Actually: leaf with distance > 4 and check_decay=true → breaks
     c. Decay timing:
        - Leaves decay instantly when check_decay becomes true and distance > 4
        - In practice: decay happens on the next random tick check
        - No delay like in older versions (pre-1.8 had up to 1 minute delay)
     d. Decay causes:
        - Log that was supporting leaves is removed
        - Leaves are placed too far from any log (e.g. player builds leaves in air)
        - Leaves placed by player: NO — player-placed leaves never decay
          - How to distinguish: check_decay blockstate property
          - Player-placed leaves: check_decay = false (from /setblock or /fill or creative)
          - Naturally generated leaves: check_decay = true

  4. Leaves with check_decay = false:
     - These leaves NEVER decay, even if no log is nearby
     - Player-placed leaves in survival: placed via /setblock or silk touch collection
     - Leaves obtained with shears and placed: also check_decay = false
     - These behave like permanent decoration blocks

  5. Leaf block drops (when they decay or are broken):
     - Without shears/silk touch: 
       - Oak leaves: 5% chance to drop sapling (0.5% for apple)
       - Spruce leaves: 5% chance to drop spruce sapling
       - Birch leaves: 5% chance to drop birch sapling
       - Jungle leaves: 2.5% chance to drop jungle sapling, 0.5% for cocoa? No, jungle leaves drop saplings
       - Acacia leaves: 5% chance to drop acacia sapling
       - Dark oak leaves: 5% chance to drop dark oak sapling
       - Sticks: 2% chance per leaf block
     - With shears: drops leaf block itself
     - With silk touch: drops leaf block itself
     - Fortune enchantment: increases sapling/apple drop chance
       - Fortune I: ~6.25%, II: ~8.33%, III: ~10%

  6. Persistence:
     - Leaf blocks from world gen have distance 1-4 (connected to tree logs)
     - When a tree is chopped (log removed): nearby leaves' check_decay becomes true
     - Leaves then decay if no longer connected to any log
     - Leaf blocks placed as part of world decoration (not trees) also have proper connectivity

Implementation notes:
  - Leaves use blockstate distance/debug properties in 1.12.2 blockstate JSON
  - Each leaf variant has its own blockstate file with distance and check_decay properties
  - Decay animation: leaves visually shake/disappear (no particle effect on decay)
```

### 9H.59 Off-Hand Item Restrictions
```
The off-hand slot (inventory slot, opposite main hand) was added in 1.9.
Default key to swap to off-hand: F (Swap Items).
Not all items can be placed in the off-hand. Rules:

Items that CAN go in off-hand:
  Any item (in creative mode — unrestricted)
  In survival mode, the following restrictions apply:

  Allowed in off-hand (survival):
    - Blocks (can be placed with right-click)
    - Tools (pickaxe, axe, shovel, hoe)
    - Weapons: sword, bow, arrow, shield, trident (1.13+), axe (as weapon)
    - Food (can be eaten with right-click)
    - Potions (can be drunk with right-click)
    - Buckets (empty, water, lava, milk)
    - Boat, minecart
    - Saddle, horse armor
    - Leads, name tags
    - Flint & steel, fire charge, shears
    - Fishing rod, carrot on a stick
    - Enchanted books, books
    - Maps (empty, filled)
    - Compass, clock
    - All miscellaneous items (string, paper, leather, etc.)
    - Music discs (for jukebox insertion)
    - Signs
    - Armor (can be equipped via right-click on armor stand)
    - Spawn eggs (creative only — spawn mob)
    - Any non-weapon item (generally allowed)

  NOT allowed in off-hand (survival):
    - Shield: CAN be placed in off-hand (this is the primary use — blocking)
    - Food: CAN be placed in off-hand
    - Potions: CAN be placed in off-hand
    - Actually there is NO restriction in survival for off-hand in 1.12.2
      Wait: there ARE restrictions in 1.12.2:
      
  Actual 1.12.2 off-hand restrictions:
    - Shields can ONLY be placed in off-hand? No, shields can be in either hand
    - Two-handed items (cannot coexist with off-hand item):
      - Bow: if a bow is in the main hand, the off-hand must be empty (or shield)
        Actually this is wrong — bow can be used with any off-hand item
      - Eating/drinking: when eating/drinking in main hand, off-hand item is still usable
      
    Actually, the REAL off-hand restrictions in 1.12.2 are minimal:
    - Totem of Undying: activates when held in either hand (main or off)
    - Shield: blocks when held in off-hand (right-click to block)
    - Food: eat from either hand
    - Potion: drink from either hand  
    - Blocks: place from either hand (if main hand is empty or holding non-block)
    - Tools: use from either hand
    - Weapons: attack from main hand only (off-hand items don't attack)
    - Map: shows in off-hand while main tool is used
    - Arrow: can be held in off-hand for crossbow? Not in 1.12.2 (no crossbow)
    
  The key "restriction" is functional rather than slot-based:
    - Main hand attack: only main hand weapon deals damage
    - Off-hand blocks: off-hand can place blocks but only if main hand is empty or not a block
    - Off-hand potions/food: can eat/drink from off-hand while main hand holds a weapon
    - Off-hand shield: blocking reduces damage by 100% in 180° forward arc
    - Off-hand map: shows location while mining with main hand

  Inverted controls: "Swap main hand with off-hand" key (F by default)
    - Swaps the items between main and off-hand slots
    - Useful for: editing, shield blocking, torch+sword combos
```

### 9H.60 Elytra Physics (Complete)
```
Elytra (ID: minecraft:elytra) are wings worn in the chestplate armor slot.
Enable flight by pressing jump while falling from a height ≥ 3 blocks.

Elytra activation:
  - Must be equipped in chestplate slot
  - Player must be falling (vertical velocity < -0.1 m/s, i.e. descending faster than walking)
  - Press jump (Space) while falling → elytra deploys
  - Visual: wings spread open on player model (animated)
  - Sound: rapid wind rushing sound (item.elytra.flying)
  - Deactivation: press jump again (stop gliding) OR touch ground

Physics model:
  Lift coefficient: 0.02 (converts forward velocity into upward force)
  Drag coefficient: 0.01 (air resistance, slows horizontal speed)
  Gravity: standard 0.08 m/s² downward (same as normal falling)
  
  Vertical motion:
    - With elytra deployed: vertical acceleration = gravity - lift
    - lift = horizontal_speed × lift_coefficient
    - At horizontal speed 20 m/s: lift = 20 × 0.02 = 0.4 m/s² upward
    - Net acceleration = 0.08 - 0.4 = -0.32 m/s² (ascending)
    - At horizontal speed 4 m/s: lift = 4 × 0.02 = 0.08 m/s²
    - Net acceleration = 0.08 - 0.08 = 0 (steady glide)
    - At horizontal speed < 4 m/s: net downward acceleration (descending)
  
  Horizontal motion:
    - Drag slows horizontal speed by 0.01 × current_speed per tick
    - drag = current_horizontal_speed × drag_coefficient × deltaTime
    - Horizontal speed slowly decays over time without boosting
    - Looking up/down affects horizontal speed:
      - Looking down (pitch > 0): increases horizontal speed (dive)
      - Looking up (pitch < 0): decreases horizontal speed (stall)
  
  Pitch control:
    - Look up (mouse up): nose up → slow down, gain altitude
    - Look down (mouse down): nose down → speed up, lose altitude
    - Pitch angle clamped to ±45° from horizontal while gliding
    - Actual pitch is interpolated: target_pitch = current_pitch + input * 0.3

  Speed ranges:
    - Minimum glide speed: ~4 m/s (below this, player falls instead of gliding)
    - Efficient glide speed: 7-12 m/s (optimal glide ratio)
    - Maximum sustainable speed: ~30 m/s (from firework boosting)
    - Terminal velocity (no elytra): ~36 m/s (but elytra prevents terminal fall)
    - Dive speed: up to ~40 m/s at 45° downward pitch
    - Stall speed: < 4 m/s → player starts falling normally

Firework boosting:
  - Right-click with firework rocket while gliding → boost
  - Boost: instant acceleration of ~30 m/s in the direction the player is facing
  - Damage: 4 HP per rocket (half hearts, 2 hearts damage)
  - Can chain multiple rockets in quick succession
  - Firework flight time: 1-3 gunpowder = 1-3 seconds flight → boost duration = flight_time
  - Flight duration: boost lasts ~1 second per gunpowder (horizontal speed boost)
  - Damage applies once per rocket (not per explosion for multi-star rockets)

Damage & durability:
  - Elytra takes durability damage while flying: 1 durability per second (20 ticks)
  - Full durability: 432
  - Total flight time per elytra: 432 seconds = 7.2 minutes (without Unbreaking)
  - Unbreaking enchantment: extends effective durability by 100/(level+1)% per use
  - Mending: repair elytra with XP orbs while gliding
  - Elytra breaks at 0 durability: becomes "broken elytra" (no flight ability)
    - Can be repaired in anvil with phantom membranes? No — Phantom membranes are 1.13+
    - In 1.12.2: elytra can only be repaired by combining two elytra in anvil (repair 5% + 12%)
    - Or use Mending enchantment to repair while gaining XP

Landing:
  - Touch ground → elytra auto-deploys? No, elytra stays deployed until press jump
  - Landing on ground at speed > 8 m/s → fall damage calculated:
    - fall_damage = ceil(horizontal_speed * 0.6) (spill damage)
    - Also: vertical fall distance since last ground contact
  - Water landing: cancels all momentum (safe landing)
  - Landing on slime block: bounce (50% velocity retained, no damage)
  - Landing on hay bale: 80% fall damage reduction

Elytra + Firework boosting tips:
  - To sustain flight: boost every 7-10 seconds (or as speed decays)
  - Optimal glide angle: ~15° downward (keeps speed above stall while minimizing descent)
  - To gain altitude: boost while looking ~15-30° upward
  - Longest flight: boost at regular intervals, maintain ~15-20 m/s speed

Disable elytra movement check gamerule:
  - /gamerule disableElytraMovementCheck true
  - Disables server-side speed validation for elytra flight
  - Allows faster flight without being "rubber-banded" back
  - Relevant in multiplayer; in single-player, doesn't apply
```

### 9H.61 Target Selectors (Complete @p @r @a @e @s Syntax)
```
Target selectors allow commands to target entities without specifying a name/UUID.
5 selectors available in 1.12.2:

Selector | Description
@p       | Nearest player (to command execution position)
@r       | Random player (or entity with type= parameter)
@a       | All players
@e       | All entities (players, mobs, items, etc.)
@s       | The executing entity (1.12 NEW)
         | In command blocks: the command block itself
         | In functions/advancements: the entity that triggered it
         | In /execute: the entity being executed as

Syntax:
  @<selector>[<parameter>=<value>,<parameter>=<value>,...]
  Parameters can be combined with AND logic (all must match for entity to be selected)

Parameters available for all selectors (@p, @r, @a, @e):
  x, y, z          — Coordinate origin for distance/volume checks
                    Default: command execution position (command block / executor)
  r, rm            — Maximum (r) and minimum (rm) radius from origin
                    r=10 → entities within 10 blocks, rm=5 → entities at least 5 blocks away
  dx, dy, dz       — Volume filter: entities within a box from (x,y,z) to (x+dx, y+dy, z+dz)
                    Used for rectangular selection instead of spherical
  c                — Count: maximum number of entities to return
                    @p[c=1] → nearest 1 player, @p[c=3] → nearest 3 players
                    @r[c=1] → random 1 player, @r[c=3] → random 3 players
                    @a[c=5] → first 5 players in selection order
                    @e[c=10] → first 10 entities
                    Negative: @e[c=-5] → farthest 5 entities (inverted order)
  m                — Game mode filter: 0=survival, 1=creative, 2=adventure, 3=spectator
                    @a[m=0] → all survival-mode players
                    m=-1 → all game modes (any)
  l, lm            — Experience level: maximum (l) and minimum (lm)
                    @p[l=5] → nearest player with ≤5 levels
  name             — Entity name filter (exact match)
                    @e[name=Zombie] → entities named "Zombie"
                    name=!Zombie → NOT named Zombie
  type             — Entity type filter (@e and @r only)
                    @e[type=zombie] → all zombies
                    type=!zombie → all entities except zombies
                    type= minecraft:zombie (fully qualified works)
                    Available values: all entity type IDs (65+ mob types, item, xp_orb, etc.)
  team             — Team filter
                    @p[team=red] → nearest player on team "red"
                    team=!red → NOT on team "red"
                    team= → no team (un-tagged players)
  scores           — Scoreboard objective filter
                    @p[scores={deaths=1..5}] → players with deaths score between 1 and 5
                    @a[scores={xp=10..}] → players with XP ≥ 10
                    @a[scores={xp=..10}] → players with XP ≤ 10
                    Supports multiple scores: scores={deaths=1..5, kills=10..}
  tag              — Scoreboard tag filter
                    @p[tag=alive] → players with tag "alive"
                    tag=!alive → NOT with tag "alive"
                    tag= → no tags
  ry, rym          — Yaw rotation: maximum (ry) and minimum (rym) yaw angle
                    @p[ry=90,rym=-90] → players looking within 90° of north
  rx, rxm          — Pitch rotation: maximum (rx) and minimum (rxm) pitch angle
                    @p[rx=45,rxm=-45] → players looking within 45° of horizontal

@r exclusive parameters:
  type             — @r[type=zombie] → random zombie entity (not just player)
  Cannot use type=! with @r (must specify positive type)

Examples of advanced selectors:
  @e[type=zombie,c=3]                        → nearest 3 zombies
  @e[type=!player,type=!item]                 → all entities except players and items
  @p[x=100,y=64,z=100,r=20]                  → nearest player within 20 blocks of (100,64,100)
  @a[m=1]                                     → all creative-mode players
  @e[type=sheep,team=animals,ry=45,rym=-45]   → sheep on team "animals" looking north-ish
  @r[type=zombie,rm=10,r=30]                  → random zombie 10-30 blocks away
  @p[scores={kills=5..ten}]                    → kill count between 5 and 10 (ten invalid — use 10)
  @p[scores={kills=10}]                        → kill count exactly 10
  @e[type=item,c=5]                            → nearest 5 item entities
  @e[tag=important,c=1]                        → nearest entity with "important" tag
  @a[lm=10,l=20]                               → players with levels 10-20

Implementation notes:
  - Selectors return array of entity objects (or single entity for @p/@r with c=1)
  - Sorting: @p/@a sorted by distance (nearest first), @r random, @e by UUID
  - Volume box (dx/dy/dz): uses inclusive coordinates
  - Distance (r/rm): Euclidean distance from (x,y,z) or executor position
  - If no entities match: returns empty array (no error for most commands)
```

### 9H.62 World Border Mechanics
```
The world border is a square boundary centered on (0,0) by default.
Controlled via /worldborder command.

World border properties:
  Center: (x, z) — default (0, 0), changeable via /worldborder center
  Size: side length in blocks — default 60,000,000 (600 million blocks, max ±30M)
  Starting size: initial size before animating
  Target size: final size after animation
  Time: milliseconds over which to animate from start to target size
  Damage: amount of damage per block outside the border
  Damage buffer: blocks the player can be outside before taking damage (default 5 blocks)

Commands:
  /worldborder set <sizeInBlocks> [timeInSeconds]
    - Sets border to sizeInBlocks (diameter = side length)
    - If timeInSeconds: animates from current size to target over that duration
    - Example: /worldborder set 100 10 → border shrinks to 100×100 over 10 seconds

  /worldborder center <x> <z>
    - Sets border center point
    - Default: 0 0

  /worldborder damage <damagePerBlock>
    - Damage per second per block outside buffer zone
    - Default: 0.2 damage/sec (0.2 HP/sec = 0.1 hearts/sec)

  /worldborder damage buffer <bufferBlocks>
    - Distance outside border before damage starts
    - Default: 5 blocks

  /worldborder warning distance <distance>
    - Distance from border at which warning is shown (red overlay on screen)
    - Default: 5 blocks

  /worldborder warning time <seconds>
    - Time before border animation completes to show warning
    - Default: 15 seconds

  /worldborder get
    - Returns current border size (in blocks)

Behavior:
  Player inside border: no effect
  Player within buffer zone (outside border but ≤ buffer blocks): no damage, visual warning
  Player beyond buffer zone: takes damage per second proportional to distance beyond buffer
  Border is a wall: cannot pass through (survival/creative)
  Border pushes entities back (mobs, items, projectiles)
  Border visualization: vertical red wall (semi-transparent) at the border edge
  Warning: screen edges turn red when player is within warning distance of border
  Warning during animation: if border is shrinking and will reach player within warning time

Damage formula:
  damage_per_tick = (distance_outside_border - buffer) * damage_amount / 20
  Where damage_amount default = 0.2 HP/s and buffer default = 5 blocks
  Example: player 10 blocks outside border, buffer=5, damage=0.2:
    damage = (10 - 5) * 0.2 / 20 = 0.05 HP per tick = 1 HP per second

Border rendering:
  Red glowing wall at border edge (vertical plane)
  Visible from both sides, extends from y=0 to y=256
  Wall has animated particles (red dust/sparkles)
  Underwater: border wall still visible (slightly muted)
  Distance-based: border wall becomes fully opaque at 2 chunks distance, fades beyond

Border interaction with entities:
  Mobs: pushed back inside border (cannot exit)
  Items: pushed back inside border
  Projectiles: destroyed when crossing border (or pushed back)
  Players: pushed back if not in creative/spectator mode
  Creative/spectator: can pass through border freely (but still takes damage if outside buffer)

World border in single-player:
  Default: effectively infinite (±30,000,000 blocks)
  Can be used for survival challenges (shrinking border)
  World border does NOT affect terrain generation (only restricts movement)
```

### 9H.63 Smooth Lighting Algorithm (Ambient Occlusion)
```
Smooth lighting (also called Ambient Occlusion or AO) softens lighting transitions
on block edges. Available as "Smooth Lighting" option in Video Settings:
  OFF: flat lighting per face (basic light value)
  MIN: minimal smoothing (some edge blending)
  MAX: full smooth lighting (ambient occlusion on all edges)

Algorithm overview:
  For each vertex on a block face, compute a light value that's the average of
  the surrounding blocks' light, weighted by how much those blocks block the sky.

Vertex light calculation:
  For each of the 4 corners of a block face:
    1. Sample 3 blocks adjacent to the corner in a 2×2 pattern:
       - The corner block (the block the vertex sits on)
       - The two adjacent blocks along the face edges
       - The diagonal block between them
    
    2. For each of these 3 samples:
       - Get sky light value and block light value
       - If the block is opaque: multiplier = 0.2 (ambient occlusion factor)
       - If the block is transparent (glass, leaves, etc.): multiplier = 0.8
       - If the block is air: multiplier = 1.0 (full light)
       - If the block allows light through but diffuses: multiplier varies
    
    3. Average the 3 samples (or weight them by occlusion):
       - vertex_sky_light = (sample[0] * occ + sample[1] * occ + sample[2] * occ) / 3
       - Same for block light
    
    4. Apply to vertex as vertex color:
       - Combine sky + block light into single brightness value
       - brightness = max(sky_light, block_light) / 15
       - Color = vec4(brightness, brightness, brightness, 1.0)

Corner occlusion values for different configurations:
  Configuration (corner, edge1, edge2, diagonal):
    All air:     1.0, 1.0, 1.0, 1.0 = full light (100%)
    One opaque:  0.8, 0.8, 0.8, 0.8 = slight shadow (80%)
    Two opaque:  0.6, 0.6, 0.6, 0.6 = medium shadow (60%)
    Three opaque: 0.2, 0.2, 0.2, 0.2 = deep shadow (20%)
    Corner+diag: 0.4, 0.4, 0.4, 0.4 = corner shadow (40%)

  Essentially: each opaque block at a corner reduces that vertex's light
  by a factor determined by how many of those blocks are opaque.

Full algorithm (for each quad):

  Given face corners A, B, C, D (4 vertices of a rectangle):

  For corner A:
    Get blocks: B1 = block at corner position (adjacent to face)
                B2 = block along edge
                B3 = block along other edge
                B4 = diagonal block
    occlusion = 0
    for each of (B1, B2, B3, B4):
      if block.opaque:
        occlusion += 0.25  (each opaque block adds 0.25 to occlusion)
    light_multiplier = 1.0 - occlusion * 0.4
    vertex_brightness = base_face_brightness * light_multiplier

  Then interpolate between:
    vertex_brightness = (vertex_brightness + neighbor_brightness) / 2

Result: edges where blocks meet get darker corners, creating the soft shadow effect.
Blocks like grass, stone, etc. have smooth transitions at their edges.

Performance:
  - Performed during chunk mesh building (not at render time)
  - Computed once per chunk, stored in vertex colors
  - Increases vertex count: each face corner needs separate vertex (no sharing)
  - Cost: ~20-30% more vertices + computation time during mesh build
  - Optimization: flat lighting for distant chunks (≥ render distance/2)

Smooth lighting levels:
  Minimum: only smooth light transitions on block edges (less computation)
  Maximum: full ambient occlusion with all 4 corner samples
  Visual difference: minimum has harder edges, maximum is softer
```

---

## P9I — Structure Loot Tables (Exact Chest Compositions)

### 9I.1 Dungeon Chest
```
Rolls: 1-3
Item                     | Weight | Count        | Notes
Saddle                   | 20     | 1            |
Iron Ingot               | 15     | 1-4          |
Bread                    | 15     | 1            |
Wheat                    | 15     | 1-4          |
Gunpowder                | 15     | 1-4          |
String                   | 15     | 1-4          |
Bucket                   | 10     | 1            |
Golden Apple             | 5      | 1            | Not enchanted
Redstone                 | 5      | 1-4          |
Coal                     | 5      | 1-4          |
Iron Bucket? No — bucket: 10
Actually: Bucket weight is 10 count 1, Golden Apple weight 5, Redstone 5 (1-4),
Coal 5 (1-4), Iron pick 5 (1), also: Name Tag 10, Music Disc 5, Enchanted Book 3
```

### 9I.2 Mineshaft Chest
```
Rolls: 2-4
Item                     | Weight | Count        | Notes
Iron Ingot               | 10     | 1-5          |
Gold Ingot               | 5      | 1-3          |
Redstone                 | 5      | 4-9          |
Diamond                  | 3      | 1-2          |
Coal                     | 10     | 3-8          |
Bread                    | 15     | 1-3          |
Melon Seeds              | 10     | 2-4          |
Pumpkin Seeds            | 10     | 2-4          |
Beetroot Seeds           | 10     | 2-4          |
Rail                     | 5      | 4-8          |
Powered Rail             | 2      | 1-4          |
Detector Rail            | 2      | 1-4          |
Activator Rail           | 2      | 1-4          |
Name Tag                 | 20     | 1            |
Enchanted Book           | 10     | 1            | Random enchant
Iron Pickaxe             | 5      | 1            |
Enchanted Golden Apple   | 1      | 1            | Rare!
Golden Apple             | 20     | 1            |
Torch                    | 15     | 10-20        |
Bucket                   | 10     | 1            |
```

### 9I.3 Stronghold Corridor Chest
```
Rolls: 2-3
Item                     | Weight | Count
Ender Pearl              | 10     | 1
Diamond                  | 3      | 1-2
Iron Ingot               | 10     | 1-5
Gold Ingot               | 5      | 1-3
Redstone                 | 5      | 4-9
Bread                    | 15     | 1-3
Apple                    | 15     | 1-3
Coal                     | 10     | 3-8
Iron Pickaxe             | 5      | 1
Iron Sword               | 5      | 1
Iron Chestplate          | 5      | 1
Iron Helmet              | 5      | 1
Iron Leggings            | 5      | 1
Iron Boots               | 5      | 1
Golden Apple             | 20     | 1
Enchanted Golden Apple   | 1      | 1
Saddle                   | 10     | 1
Iron Block               | 5      | 1-5 (actually it's iron ingot)
Gold Ingot               | 5      | 1-5
```

### 9I.4 Stronghold Library Chest
```
Rolls: 1-4
Item                     | Weight | Count | Notes
Book                     | 20     | 1-3    |
Paper                    | 20     | 2-7    |
Compass                  | 1      | 1      |
Enchanted Book           | 10     | 1      | Random enchant
```

### 9I.5 Stronghold Altar Chest (Portal Room)
```
Rolls: 2-4
Item                     | Weight | Count
Diamond                  | 5      | 1-3
Iron Ingot               | 15     | 4-9
Gold Ingot               | 15     | 4-9
Redstone                 | 15     | 4-9
Bread                    | 15     | 1-3
Apple                    | 15     | 1-3
Coal                     | 15     | 3-8
Iron Pickaxe             | 5      | 1
Iron Sword               | 5      | 1
Iron Chestplate          | 5      | 1
Iron Helmet              | 5      | 1
Iron Leggings            | 5      | 1
Iron Boots               | 5      | 1
Golden Apple             | 5      | 1
Emerald                  | 5      | 1-3
Ender Pearl              | 10     | 1-2
Eye of Ender             | 3      | 2-3
Saddle                   | 5      | 1
Iron Block               | 5      | 1-5
Gold Block               | 3      | 1-3
```

### 9I.6 Desert Temple Chest (4 chests)
```
Each chest: rolls 2-4
Item                     | Weight | Count
Diamond                  | 5      | 1-3
Iron Ingot               | 15     | 1-5
Gold Ingot               | 15     | 2-7
Emerald                  | 15     | 1-3
Bone                     | 15     | 4-6
Spider Eye               | 15     | 1-3
Rotten Flesh             | 15     | 3-7
Saddle                   | 10     | 1
Iron Pickaxe             | 5      | 1
Iron Sword               | 5      | 1
Iron Chestplate          | 5      | 1
Iron Helmet              | 5      | 1
Iron Leggings            | 5      | 1
Iron Boots               | 5      | 1
Golden Apple             | 20     | 1
Enchanted Golden Apple   | 2      | 1
Book (Enchanted)         | 10     | 1      | Random enchant
Enchanted Book           | 10     | 1
Gold Block               | 5      | 1
Bedrock? No — trap: TNT 9 blocks under pressure plate in center
```

### 9I.7 Jungle Temple Chest (2 chests)
```
Staircase chest (rolls 2-6):
Item                     | Weight | Count
Diamond                  | 3      | 1-3
Iron Ingot               | 10     | 1-5
Gold Ingot               | 5      | 2-7
Emerald                  | 3      | 1-3
Bone                     | 10     | 4-6
Saddle                   | 10     | 1
Iron Pickaxe             | 5      | 1
Iron Sword               | 5      | 1
Iron Chestplate          | 5      | 1
Iron Helmet              | 5      | 1
Iron Leggings            | 5      | 1
Iron Boots               | 5      | 1
Golden Apple             | 20     | 1
Book (Enchanted)         | 10     | 1      | Random enchant
Bamboo? No — 1.14+ item, skip

Puzzle chest (rolls 2-6): same items plus:
Item                     | Weight | Count
Bone                     | 15     | 4-6
Rotten Flesh             | 15     | 3-7
String                   | 15     | 4-6
Arrow                    | 15     | 5-15
```

### 9I.8 Nether Fortress Chest
```
Rolls: 2-4
Item                     | Weight | Count
Diamond                  | 3      | 1-3
Iron Ingot               | 10     | 1-5
Gold Ingot               | 5      | 1-3
Nether Wart              | 20     | 3-7
Blaze Rod                | 5      | 1-2
Ghast Tear               | 5      | 1-3
Glowstone Dust           | 10     | 1-5
Nether Brick             | 10     | 2-7
Saddle                   | 5      | 1
Iron Pickaxe             | 5      | 1
Golden Apple             | 20     | 1
Golden Chestplate        | 5      | 1
Golden Sword             | 5      | 1
Golden Helmet            | 5      | 1
Golden Leggings          | 5      | 1
Golden Boots             | 5      | 1
Iron Horse Armor         | 3      | 1
Gold Horse Armor         | 3      | 1
Diamond Horse Armor      | 3      | 1
Obsidian                 | 5      | 2-4
Iron Block               | 3      | 1-3
Gold Block               | 3      | 1-3
```

### 9I.9 End City Chest
```
Rolls: 2-6
Item                     | Weight | Count
Diamond                  | 5      | 2-7
Iron Ingot               | 10     | 4-8
Gold Ingot               | 10     | 2-7
Emerald                  | 5      | 2-6
Beetroot Seeds           | 10     | 1-10
Saddle                   | 5      | 1
Iron Pickaxe             | 5      | 1
Iron Sword               | 5      | 1
Iron Chestplate          | 5      | 1
Iron Helmet              | 5      | 1
Iron Leggings            | 5      | 1
Iron Boots               | 5      | 1
Diamond Pickaxe          | 3      | 1
Diamond Sword            | 3      | 1
Diamond Chestplate       | 3      | 1
Diamond Helmet           | 3      | 1
Diamond Leggings         | 3      | 1
Diamond Boots            | 3      | 1
Golden Apple             | 10     | 1
Enchanted Golden Apple   | 2      | 1
Enchanted Book           | 5      | 1      | Random enchant
Experience Bottle        | 10     | 1-3    | (bottle o' enchanting)
Iron Block               | 5      | 1-4
Gold Block               | 5      | 1-4
Diamond Block            | 2      | 1
Sponge                   | 5      | 1-2
Music Disc "far"         | 2      | 1
Music Disc "chirp"       | 2      | 1
Music Disc "strad"       | 2      | 1
Music Disc "mall"        | 2      | 1
Music Disc "mellohi"     | 2      | 1
Music Disc "13"          | 2      | 1
Music Disc "blocks"      | 2      | 1
Music Disc "cat"         | 2      | 1
Music Disc "wait"        | 2      | 1
Music Disc "stal"        | 2      | 1
Music Disc "ward"        | 2      | 1
Music Disc "11"          | 2      | 1

End Ship loot: Elytra in item frame, 1 Diamond Chestplate on armor stand,
  Rolls: 2-6, same as End City chest loot
```

### 9I.10 Igloo Chest
```
Basement chest (rolls 2-5):
Item                     | Weight | Count | Notes
Bread                    | 15     | 1-3    |
Apple                    | 15     | 1-3    |
Golden Apple             | 20     | 1      |
Enchanted Golden Apple   | 1      | 1      |
Coal                     | 10     | 1-4    |
Iron Ingot               | 10     | 1-5    |
Redstone                 | 5      | 1-4    |
Gold Ingot               | 5      | 1-3    |
Diamond                  | 3      | 1-2    |
Emerald                  | 3      | 1-2    |
Rotten Flesh             | 5      | 1-3    |
Bone                     | 5      | 1-3    |
Saddle                   | 5      | 1      |
Iron Sword               | 3      | 1      |
Iron Pickaxe             | 3      | 1      |
Golden Apple             | 5      | 1      |
Water Bottle             | 10     | 1-3    | For zombie villager cure
Brewing Stand            | 1      | 1      | Always present in igloo basement
Cauldron                 | 1      | 1      | 1/3 full, always present
```

### 9I.11 Woodland Mansion Chests
```
Various rooms: 1-3 chests per mansion

Small room chest (rolls 2-5):
Item                     | Weight | Count
Bread                    | 15     | 1-3
Wheat                    | 15     | 1-4
Coal                     | 10     | 1-4
Redstone                 | 10     | 1-4
String                   | 10     | 1-4
Gunpowder                | 10     | 1-4
Iron Ingot               | 10     | 1-4
Bone                     | 10     | 1-4
Gold Ingot               | 5      | 1-3
Diamond                  | 3      | 1-2
Emerald                  | 3      | 1-3
Saddle                   | 5      | 1
Iron Pickaxe             | 5      | 1
Iron Sword               | 5      | 1
Iron Chestplate          | 5      | 1
Iron Helmet              | 5      | 1
Iron Leggings            | 5      | 1
Iron Boots               | 5      | 1
Golden Apple             | 10     | 1
Enchanted Golden Apple   | 1      | 1
Enchanted Book           | 5      | 1
Name Tag                 | 5      | 1

Large room chest (rolls 3-8): same as small + more gold/diamond
```

### 9I.12 Bonus Chest
```
Rolls: 5-21 (if bonus chest enabled on world creation)
Item                     | Weight | Count | Notes
Wooden Axe               | 10     | 1      |
Wooden Pickaxe           | 10     | 1      |
Apple                    | 10     | 1-2    |
Bread                    | 10     | 1-2    |
Oak Log                  | 10     | 1-4    |
Oak Planks               | 10     | 1-4    |
Stick                    | 10     | 1-4    |
Torch                    | 10     | 1-4    |
Oak Sapling              | 5      | 1-2    |

Also from the "bonus chest" type:
Cobblestone              | 3      | 1-3
Stone Pickaxe            | 3      | 1
Stone Axe                | 3      | 1
Iron Ingot               | 3      | 1
Iron Pickaxe             | 1      | 1
Golden Apple             | 1      | 1
```

### 9I.13 Fishing Loot Table
```
Rolls: 1 (per cast)

Fishing "fish" pool (85%):
Item                     | Weight | Count
Raw Cod                  | 60     | 1
Raw Salmon               | 25     | 1
Tropical Fish            | 2      | 1
Pufferfish               | 13     | 1

Fishing "junk" pool (10%):
Item                     | Weight | Count
Lily Pad                 | 17     | 1     (actually: 2% of total)
Bowl                     | 10     | 1
Fishing Rod              | 2      | 1     (damaged 10-90%)
Leather                  | 10     | 1
Leather Boots            | 10     | 1     (damaged 10-90%)
Rotten Flesh             | 10     | 1
Stick                    | 5      | 1
String                   | 5      | 1
Water Bottle             | 10     | 1
Bone                     | 10     | 1
Ink Sac                  | 1      | 1
Tripwire Hook            | 10     | 1

Fishing "treasure" pool (5%):
Item                     | Weight | Count | Notes
Bow                      | 3      | 1      | Enchanted (level 1-30)
Enchanted Book           | 3      | 1      | Random enchant
Fishing Rod              | 3      | 1      | Enchanted (level 1-30)
Name Tag                 | 3      | 1
Saddle                   | 3      | 1
Nautilus Shell           | —      | —      | Not in 1.12.2 (1.13+)
```

---

## P10 — Sounds & Music

### 10.1 Sound Event Registry
```
All sounds referenced via resource location: minecraft:<category>.<name>
Sound files stored as .ogg (Vorbis) at:
  @minecraft/sounds/<category>/<name>.ogg

Sound categories (used in options slider):
  master    — Overall volume
  music     — Background music (menu, creative, nether, end, discs)
  record    — Jukebox music discs
  weather   — Rain, thunder
  hostile   — Hostile mob sounds
  friendly  — Passive mob sounds  
  players   — Player sounds (hurt, step, etc.)
  blocks    — Block step/break/place sounds
  ambient   — Cave ambience, nether ambience
  voice     — Narrator (1.12 NEW)

Sound JSON definitions stored at:
  @minecraft/sounds.json (maps sound events to .ogg files)

Event naming convention:
  block.<block_type>.<action>   — step, break, place, hit, fall
  entity.<entity>.<action>       — ambient, hurt, death, step, attack, swim, splash
  item.<item>.<action>           — equip, use, break
  music.<menu|game>.<variant>    — background music
  record.<disc_name>             — music discs
  ambient.<environment>.<type>   — cave, weather, nether
  ui.<action>                    — button click, toast, etc.
```

### 10.2 Music Disc Track Listing (12 Discs)
```
All discs play exclusively on jukebox. Obtained from dungeon/chest loot.
C418 is the composer for all 1.12.2 discs.

  Disc    | Track Title              | Length  | Description
  minecraft:record_13    | "13"    | 2:58    | Ambient, creepy — dripping, wind, echoing sounds
  minecraft:record_cat   | "cat"   | 3:04    | Upbeat synth melody, gentle rhythm
  minecraft:record_blocks| "blocks"| 5:43    | Calm, melodic, piano-driven
  minecraft:record_chirp | "chirp" | 3:00    | Cheerful, retro video game feel
  minecraft:record_far   | "far"   | 2:54    | Melancholic, slow, spacious
  minecraft:record_mall  | "mall"  | 3:17    | Calm, jazzy, shopping-mall feel
  minecraft:record_mellohi| "mellohi"| 1:36   | Somber, slow piano, glitchy
  minecraft:record_stal  | "stal"  | 2:30    | Laid-back jazz, walking bass
  minecraft:record_strad | "strad" | 3:08    | Upbeat, electronic, rhythmic
  minecraft:record_ward | "ward"  | 4:10    | Orchestral, dramatic, marching
  minecraft:record_11   | "11"    | 1:11    | Creepy ambient — footsteps, breathing, static (scratched/fragmented)
  minecraft:record_wait | "wait"  | 3:51    | Peaceful, melodic, slower tempo

Disc textures stored at:
  @minecraft/textures/items/record_13.png through record_wait.png (existing assets)
Disc item models stored at:
  @minecraft/models/item/record_13.json through record_wait.json
```

### 10.3 Background Music
```
Menu music: "Minecraft" theme (C418)
  File: @minecraft/sounds/music/menu/minecraft.ogg
  Plays on title screen, loops

Creative mode music: 4 tracks
  "Creative 1" — calm, piano
  "Creative 2" — more upbeat piano
  "Creative 3" — melodic
  "Creative 4" — contemplative
  Files: @minecraft/sounds/music/game/creative/

Nether music: 1 track
  "Nether" — ominous, ambient droning
  File: @minecraft/sounds/music/game/nether/nether.ogg

End music: 1 track
  "The End" — ethereal, choral
  File: @minecraft/sounds/music/game/end/end.ogg
  Also plays boss music: @minecraft/sounds/music/game/end/boss.ogg (dragon fight)

Music play rules:
  Menu music: plays on title screen, stops when entering world
  Game music: random interval 600-18000 ticks (30s-15min) between tracks
  Disc music: overrides all background music when playing on jukebox
  Jukebox range: ~65 blocks audible range
```

### 10.4 Cave Ambient Sounds
```
13 cave ambient sound files, played randomly when player is underground (sky light = 0):
  Files stored at: @minecraft/sounds/ambient/cave/

  Sound file           | Description
  cave1.ogg through cave19.ogg  | 13 total files (1-19 but only 13 exist)
  
  These are eerie, echoing sounds — dripping water, distant roars, wind, creaks
  Play interval: 600-6000 ticks (30s-5min) random delay between plays
  Volume: 0.7 (relative to ambient slider)
  Only plays when: player has no block above head with sky access (solid ceiling)
```

### 10.5 Weather Sounds
```
Rain:
  @minecraft/sounds/weather/rain/       — rain.ogg (looping background)
  @minecraft/sounds/ambient/weather/    — rain sounds (various intensities)
  Volume: proportional to rain intensity (0-1)

Thunder:
  @minecraft/sounds/ambient/weather/thunder1.ogg through thunder3.ogg
  Plays 1-3 seconds after lightning strike (based on distance)
  Volume proportional to distance (closer = louder)

Weather music: No dedicated music (only ambient sounds)
```

### 10.6 Block Sounds
```
All block sounds referenced as:
  block.<material>.step     — walking on block
  block.<material>.break    — breaking block
  block.<material>.place    — placing block
  block.<material>.hit      — mining block (repetitive)
  block.<material>.fall     — falling onto block (distance-based)

Material categories (with sound types):
  stone:        stone, cobblestone, ore, obsidian, bedrock, etc.
  wood:         planks, logs, crafting table, bookshelf, chest, door, fence, etc.
  gravel:       gravel, sand, concrete powder, soul sand
  grass:        grass, dirt, podzol, mycelium, etc.
  sand:         sand, red sand, sandstone
  snow:         snow, snow block, ice, packed ice
  ladder:       ladder, vines
  anvil:        anvil (all 3 damage stages)
  glass:        glass, glass pane, glowstone, beacon, sea lantern, redstone lamp
  cloth:        wool, carpet (all 16 colors)
  slime:        slime block
  metal:        iron block, gold block, diamond block, emerald block, etc.
  plant:        leaves, grass, flowers, crops, vines, etc.
  liquid:       water, lava
  chest:        chest, ender chest, trapped chest
  stone_button: stone button (click sound)
  wood_button:  wood button (click sound)

Files stored at: @minecraft/sounds/block/<material>/<action>1.ogg through <action>4.ogg
  (Each action has 2-5 randomized variants)
```

### 10.7 Entity Sounds
```
All entity sounds referenced as:
  entity.<entity_type>.<action>   — ambient, hurt, death, step, swim, splash, etc.

Per-mob sound mappings:
  Zombie:        entity.zombie.ambient/hurt/death/step/infect/unfect
  Skeleton:      entity.skeleton.ambient/hurt/death/step
  Creeper:       entity.creeper.ambient/hurt/death/primed/explode
  Spider:        entity.spider.ambient/hurt/death/step
  Enderman:      entity.enderman.ambient/hurt/death/stare/scream/teleport
  Slime:         entity.slime.squish/attack/big/medium/small
  Ghast:         entity.ghast.ambient/hurt/death/shoot/warn
  Blaze:         entity.blaze.ambient/hurt/death/shoot
  Pig:           entity.pig.ambient/hurt/death/step
  Cow:           entity.cow.ambient/hurt/death/step/milk
  Chicken:       entity.chicken.ambient/hurt/death/step/egg
  Sheep:         entity.sheep.ambient/hurt/death/step/shear
  Wolf:          entity.wolf.ambient/hurt/death/step/bark/growl/howl/pant/whine
  Villager:      entity.villager.ambient/hurt/death/trading/yes/no
  Horse:         entity.horse.ambient/hurt/death/gallop/step/jump/land/breath/armor
  Bat:           entity.bat.ambient/hurt/death/loops/takeoff
  Squid:         entity.squid.ambient/hurt/death
  Ocelot/Cat:    entity.cat.ambient/hurt/death/purr/purreow/meow
  Iron Golem:    entity.irongolem.ambient/hurt/death/step/throw/repair
  Snow Golem:    entity.snowman.ambient/hurt/death/shoot
  Guardian:      entity.guardian.ambient/hurt/death/flop/curse/attack/land_hit
  Witch:         entity.witch.ambient/hurt/death/drink/throw
  Rabbit:        entity.rabbit.ambient/hurt/death/jump/attack
  Llama:         entity.llama.ambient/hurt/death/spit/step/angry/chest
  Parrot:        entity.parrot.ambient/hurt/death/fly/eat/step (1.12 NEW)
  Vex:           entity.vex.ambient/hurt/death/charge
  Vindicator:    entity.vindication_illager.ambient/hurt/death/idle/celebrate
  Evoker:        entity.evocation_illager.ambient/hurt/death/prepare_attack/prepare_summon
                 entity.evocation_fangs.attack
  Illusioner:    entity.illusion_illager.ambient/hurt/death/mirror_move/prepare_mirror
                 entity.illusion_illager.prepare_blindness
  Shulker:       entity.shulker.ambient/hurt/death/shoot/close/open/bullet_hurt
  Endermite:     entity.endermite.ambient/hurt/death/step
  Wither:        entity.wither.ambient/hurt/death/spawn/shoot/break_block
  Ender Dragon:  entity.enderdragon.ambient/hurt/death/flap/growl/fireball

Player sounds:
  entity.player.hurt/drown/burn/death/fall(small/big)/drink/burp/eat/levelup
  entity.player.swim/splash/breathe
  entity.player.attack/crit/strong (sweep attack)
  entity.player.breath (underwater, gasping)

Files at: @minecraft/sounds/entity/<entity>/<action>1.ogg through <action>4.ogg
```

### 10.8 UI & Other Sounds
```
UI sounds:
  ui.button.click         — Button press (all menus)
  ui.toast.in             — Toast notification slide in
  ui.toast.out            — Toast notification slide out
  ui.toast.challenge_complete — Challenge advancement toast
  ui.crafting_take_result — Pick up crafted item
  ui.loom.take_result     — (ignore, 1.13+ loom)

Item sounds:
  item.<item_type>.equip_generic — Equip armor/tool (pop sound)
  item.armor.equip_<material>    — Leather/chain/iron/gold/diamond equip
  item.bucket.fill_water/lava    — Fill bucket
  item.bucket.empty_water/lava   — Empty bucket
  item.flintandsteel.use         — Flint & steel click
  item.hoe.till                  — Hoe tilling soil
  item.shovel.flatten            — Shovel making grass path
  item.axe.strip                 — Axe stripping wood (not in 1.12.2, 1.13+)
  item.elytra.flying             — Elytra wind sound
  item.firecharge.use            — Fire charge use
  item.firework.blast            — Firework explosion
  item.firework.twinkle          — Firework twinkle effect
  item.firework.large_blast      — Large firework explosion
  item.firework.launch           — Firework launch
  item.trident.*                 — Not in 1.12.2 (1.13+)

Weather/ambient:
  weather.rain/rainabove        — Rain
  weather.thunder               — Thunder
  ambient.cave.cave             — Cave ambience (see 10.4)
  ambient.nether.*              — Nether ambient sounds (if present in resource pack)

Note block: 25 distinct sounds based on instrument
  block.note_block.harp/basedrum/snare/hat/bass/pluck/bell/flute/chime/xylophone
  Each instrument maps to a different block type below the note block
  Two octaves (F#3-F#5 = semitones 0-24)

Portal: entity.minecart.riding (nether portal sound loop? Actually portal ambient uses ambient sound)
  block.portal.ambient/trigger/travel

Fire: block.fire.ambient, entity.firework.launch, etc.
```

### 10.9 Audio Implementation Notes
```
Audio engine:
  - Web Audio API with AudioContext
  - Sound pool: 32-64 active channels
  - 3D spatial audio: HRTF/panner node for positional sounds
  - Distance model: inverse (rolloff 1.0)
  - Sound attenuation: per-category volume + distance falloff
  - Per-category sliders: master, music, records, weather, hostile, friendly,
    players, blocks, ambient, voice
  - Music crossfade: 1-2 second crossfade between tracks
  - Sound file format: .ogg Vorbis (playable via decodeAudioData)
  - Disc must be fully decoded before playback (preload on world load)

Asset paths for sound files:
  @minecraft/sounds/<namespace>/<category>/<name>.ogg
  Where namespace is typically "minecraft" for vanilla sounds
  Example: @minecraft/sounds/minecraft/music/menu/minecraft.ogg
  Example: @minecraft/sounds/minecraft/entity/zombie/ambient1.ogg
  Example: @minecraft/sounds/minecraft/records/13.ogg

Subtitle system:
  - Displayed at bottom of screen when "Subtitles" enabled in options
  - Shows sound event name in localized form when sound plays
  - Example: subtitles.block.door.open → "*Door opens*"
  - Localized via lang file keys: subtitles.<category>.<name>
```

---

## P10A — Texture & Asset Pipeline

### 10A.1 Atlas Generation Strategy
```
All block/item textures combined into texture atlases for WebGL rendering.
Five atlases needed:
  - blocks.png:   All block face textures (16×16 tiles, up to 2048+ tiles)
  - items.png:    All item sprites (16×16 tiles)
  - entity.png:   Entity skins (various sizes, packed)
  - gui.png:      UI widgets, icons, container textures
  - environment.png: Sky, sun, moon, clouds

Block atlas:
  Each block face is 16×16 pixels
  Multi-face blocks: top/side/bottom may differ (grass, logs, dispensers, etc.)
  Animated textures: stored as separate frames within the atlas
    - water_still: 32 frames (×16 = 512px height in atlas)
    - lava_still:  32 frames
    - fire_layer_0/1: may have fewer frames (1-4)
    - Nether portal: 32 frames
    - prismarine: 4 frames (rough glow animation)
  UV coordinates: stored per-face as (u,v) range within atlas
  Mipmapping: auto-generated from atlas (requires atlas to be power-of-2)
  Padding: 1px border around each tile to prevent mipmap bleeding (clamp-to-edge also helps)

Atlas generation: build at runtime from individual PNG tiles
  Assets loaded from: @minecraft/textures/blocks/<texture_name>.png
  Metadata from: @minecraft/textures/blocks/<texture_name>.png.mcmeta (animation data)
  Tiles packed into atlas with sorted-by-size algorithm (greedy rect packing)
```

### 10A.2 Texture Directory Reference
```
Textures are pre-provided at @minecraft/textures/ with the following structure:

  blocks/           — Block face textures (16×16 PNG)
    Includes: all terrain textures (~400 files covering all block types)
    Animation metadata: matching .png.mcmeta files for animated textures
    Glazed terracotta: 16 directional pattern textures
    Concrete/concrete powder: 16 colors each
    Wool: 16 colors
    Glass/stained glass: 16 colors + plain
    Glass pane: 16 colored top textures + plain
    Logs: each wood type has side + top
    Planks: 6 wood types
    Leaves: 6 leaf types (with transparency)
    Custom block models reference these textures in blockstate JSON

  items/            — Item sprites (16×16 PNG)
    Includes: tools, weapons, armor (separate pieces), food, potions,
              spawn eggs, music discs, maps, etc. (~340 files)
    Tools have unique texture per material (wood/stone/iron/gold/diamond)
    Armor uses separate texture per piece (helmet/chestplate/leggings/boots)
    Leather armor has overlay layer
    Potions have bottle + overlay + contents tint
    Spawn eggs: base + overlay (dye color per mob type)

  entity/           — Entity skins (various sizes, typically symmetric)
    Subdirectories: armorstand/, banner/, bat, bed/, bear/, blaze, boat/,
      cat/, chest/, cow/, creeper/, endercrystal/, enderdragon/, enderman/,
      ghast/, horse/, illager/, llama/, parrot/, pig/, rabbit/, sheep/,
      shulker/, skeleton/, slime/, spider/, villager/, wither/, wolf/,
      zombie/, zombie_villager/, zombie_piglin? No, zombie_pigman.png
    Key files:
      steve.png / alex.png     — Default player skins (64×64)
      arrow.png               — Arrow entity
      beacon_beam.png         — Beacon beam
      shadow.png              — Entity shadow
      experience_orb.png      — XP orb
      chorus_fruit.png? No — end_portal.png
      end_portal.png          — End portal star field
      end_gateway_beam.png    — End gateway beam
      lead_knot.png           — Lead knot
      minecart.png            — Minecart
      snowman.png             — Snow golem
      squid.png               — Squid
      witch.png               — Witch

  gui/              — UI textures
    widgets.png           — Button, slot, tab, scrollbar, checkbox elements
    icons.png             — Health, hunger, armor, air, XP, hotbar, crosshair
    bars.png              — Boss bar textures
    book.png              — Book GUI (written book)
    recipe_book.png       — Recipe book overlay
    toasts.png            — Toast notification background
    spectator_widgets.png — Spectator menu
    stream_indicator.png  — Recording indicator
    container/            — Inventory container backgrounds
      crafting_table.png, furnace.png, enchanting_table.png, beacon.png,
      anvil.png, brewing_stand.png, dispenser.png, hopper.png,
      chest.png (double_chest.png), villager.png, horse.png,
      shulker_box.png, creative_inventory/tabs/
    advancements/         — Advancement tree background + icons
    title/                — Title screen textures (logo, panorama)
    presets/              — Superflat presets icons
    resource_packs.png, server_selection.png, world_selection.png,
    demo_background.png, options_background.png

  colormap/         — Color lookup tables (256×256 PNG)
    grass.png         — Grass color per temperature/humidity
    foliage.png       — Foliage color per temperature/humidity

  environment/      — Sky & environment textures
    clouds.png        — Cloud layer (fast/fancy graphics)
    moon_phases.png   — 8 moon phases in a row
    rain.png          — Rain streak texture
    snow.png          — Snow particle texture  
    sun.png           — Sun disc texture
    end_sky.png       — End dimension sky texture

  map/              — Map icons
    map_icons.png     — Map marker icons (player, item frame, etc.)

  misc/             — Miscellaneous textures
    shadow.png        — Entity shadow
    underbarrier.png  — Barrier block overlay? No — underbarrier.png is a subtle vignette
    village_chessboard.png — Village screen? No — pumpkineyes? forcefield? No
    Actually: forcefield.png, pumpkineyes.png, underbarrier.png, village_chessboard.png, village_dots.png, village_sky.png

  models/           — 3D model textures (for entities)
    Subdirectories for various entity models

  painting/         — Painting art
    paintings_kristoffer_zetterstrand.png  — All 26 paintings in single sheet

  particle/         — Particle textures
    particle.png      — Single particle texture atlas (all particle types in one sheet)
    footptint.png, angry.png, glow.png, huge.png, swoosh.png, bubble.png, etc.

  font/             — Font textures (ASCII glyphs)
    Default font: ASCII characters in a grid (page 0 = default.png)
    Alternative: Unicode page extras (if present)
```

### 10A.3 Blockstate JSON Format
```
Each block has a blockstate file at: @minecraft/blockstates/<block_name>.json
Format:
  {
    "variants": {
      "facing=north,powered=false": { "model": "minecraft:block/furnace" },
      "facing=south,powered=false": { "model": "minecraft:block/furnace", "y": 180 },
      "facing=east,powered=false":  { "model": "minecraft:block/furnace", "y": 90 },
      "facing=west,powered=false":  { "model": "minecraft:block/furnace", "y": 270 },
      "facing=north,powered=true":  { "model": "minecraft:block/furnace_on" },
      ...
    }
  }
  Or with "multipart" for connections (fences, glass panes, redstone):
  {
    "multipart": [
      { "apply": { "model": "minecraft:block/fence_post" } },
      { "when": { "north": "true" }, "apply": { "model": "minecraft:block/fence_side", "uvlock": true } },
      ...
    ]
  }

Block model format at: @minecraft/models/block/<model_name>.json
  {
    "parent": "block/cube_all",
    "textures": {
      "all": "minecraft:blocks/stone"
    }
  }
  Built-in parent models: cube_all, cube_top, cube_bottom_top, cube_column,
    orientable, cross, crop, fence_post, fence_side, wall, stairs, slab,
    door_bottom, door_top, trapdoor, torch, lever, button, pressure_plate,
    carpet, snow_height, flower_pot, chest, etc.

Item model format at: @minecraft/models/item/<item_name>.json
  {
    "parent": "minecraft:item/generated",
    "textures": {
      "layer0": "minecraft:items/diamond"
    }
  }
  Built-in parent models: generated, handheld (tools), builtin_entity (spawn eggs, banners)
  Layer tint: "layer0" can have "tintindex": 0 for biome coloring (grass, leaves, etc.)
```

### 10A.4 Animated Texture Format (.mcmeta)
```
Animation metadata stored alongside PNG:
  my_texture.png           — The texture sheet
  my_texture.png.mcmeta    — JSON animation metadata

Format example:
  {
    "animation": {
      "interpolate": false,      // Whether to interpolate between frames
      "frametime": 1,            // Game ticks per frame (default 1 = 0.05s)
      "frames": [0, 1, 2, 3, 4, 5]  // Frame order (optional; default = sequential all rows)
    }
  }

  The texture image is divided into horizontal strips, each strip is one frame.
  Height of each frame = image_height / number_of_frames.
  Example: water_still.png (16×512) = 32 frames × 16px high each

1.12.2 animated textures:
  water_still.png          — 32 frames (interpolate=false, frametime=1)
  water_flow.png           — 32 frames (interpolate=false, frametime=1)
  lava_still.png           — 32 frames (interpolate=false, frametime=1)
  lava_flow.png            — 32 frames (interpolate=false, frametime=1)
  fire_layer_0.png         — 1 frame? Actually fire has 2 frames (frametime=2)
  fire_layer_1.png         — 2 frames
  portal.png               — 32 frames (interpolate=false, frametime=1)
  prismarine_rough.png     — 4 frames (frametime=2)
  magma.png                — 2 frames (frametime=2)
  sea_lantern.png          — 4 frames? No, sea_lantern has no animation — but glow animation via .mcmeta
  command_block_*.png      — 4 frames (frametime=1) — pulse animation
  structure_block*.png     — 4 frames (frametime=1) — save mode pulse
  torch_on.png             — No animation (static flame effect via particle)
  all glazed terracotta:   — Have animation? Only rotating pattern, not via mcmeta
```

### 10A.5 Asset Loading Pipeline
```
1. On game start: scan @minecraft/ directory structure
2. Build asset registry:
   - blocks: iterate @minecraft/blockstates/*.json
   - items: @minecraft/models/item/*.json
   - textures: @minecraft/textures/**/*.png
   - sounds: @minecraft/sounds/**/*.ogg (loaded on demand)
3. Load blockstates JSON → build variant → model mapping
4. Load block models JSON → resolve parent inheritance → extract texture references
5. Load item models JSON → extract texture references
6. Load item texture → generate item atlas
7. Load block textures → generate block atlas
8. Load environment textures (sky, sun, moon, clouds) individually
9. Load GUI textures → build UI atlas or individual sprites
10. Load language file → @minecraft/lang/en_us.lang → key→value map
11. Load advancements → @minecraft/advancements/*.json
12. Load recipes → @minecraft/recipes/*.json
13. Load loot tables → @minecraft/loot_tables/**/*.json
14. Other: splashes.txt, credits.txt, end.txt, icons, sounds.json
```

---

## P11 — Status Effects (Complete 1.12.2)

### 11.1 Effect Table
```
ID  Effect           Type    Particles  Color     Notes
1   Speed            Benefit  Yes       #7CAFC6  +20% move speed per level
2   Slowness         Harm     Yes       #5A6C81  -15% move speed per level
3   Haste            Benefit  Yes       #D9C043  +20% mining speed per level
4   Mining Fatigue   Harm     Yes       #4A4217  -20% mining speed per level
5   Strength         Benefit  Yes       #932423  +3 melee damage per level
6   Instant Health   Benefit  No        #F82423  4 HP healed per level, damages undead
7   Instant Damage   Harm     No        #430A09  6 HP damage per level, heals undead
8   Jump Boost       Benefit  Yes       #22FF4C  +0.1 jump velocity per level
9   Nausea           Harm     Yes       #551D4A  warps screen (portal distortion effect)
10  Regeneration     Benefit  Yes       #CD5CAB  1 HP per 2.5s at I, 1 HP per 1.2s at II
11  Resistance       Benefit  Yes       #99453A  -20% damage per level
12  Fire Resistance  Benefit  No        #E49A3A  immune to fire, lava, and fire damage
13  Water Breathing  Benefit  No        #2E5299  no drowning, no breath bar depletion
14  Invisibility     Benefit  No        #7F8392  entity invisible (items/armor still visible)
15  Blindness        Harm     No        #1F1F23  black fog, no sprint/critical hits
16  Night Vision     Benefit  No        #1F1FA1  full brightness in darkness
17  Hunger           Harm     Yes       #587653  adds exhaustion, drains food faster
18  Weakness         Harm     Yes       #484D48  -4 melee damage per level (min 0)
19  Poison           Harm     Yes       #4E9331  1 HP per 2.5s at I, 1 HP per 1.2s at II, stops at 0.5 HP left
20  Wither           Harm     Yes       #352A27  1 HP per 2s at I, 1 HP per 1s at II, lethal (can kill)
21  Health Boost     Benefit  Yes       #F87D23  +4 max HP per level (temp health)
22  Absorption       Benefit  No        #2552A5  +4 golden hearts per level (absorb damage first)
23  Saturation       Benefit  No        #F82423  instantly restores food/saturation
24  Glowing          Benefit  No        #94A061  entity outline visible through blocks
25  Levitation       Harm     Yes       #CEFFFF  upward velocity (0.05 per tick, ~4 m/s at I)
26  Luck             Benefit  No        #339900  +1 luck attribute per level (affects loot quality)
27  Bad Luck         Harm     No        #990033  -1 luck attribute per level
```
Note: "Color" is the potion color for the status effect (used for splash potions, etc.)

### 11.2 Duration Sources
```
Offensive potions (hostile):
  Potion             Base duration  Splash (×0.75)  Lingering (×0.25)
  ────────────────────────────────────────────────────────────────────
  Poison             0:45 (I)       0:33.75         0:11.25
  Poison (extended)  1:30           1:07.5           0:22.5
  Poison II          0:21.75        0:16.31          0:5.44
  Wither             0:45           —                —
  Weakness           1:30           1:07.5           0:22.5
  Weakness (ext)     4:00           3:00             1:00
  Slowness           1:30           1:07.5           0:22.5
  Slowness (ext)     4:00           3:00             1:00
  Slowness IV        0:20 (from stray arrow)        (0:15 splash)
  Harming I          0              0 (instant)      0 (instant)
  Harming II         0              0 (instant)      0 (instant)

Beneficial potions:
  Potion             Base duration  Splash (×0.75)  Lingering (×0.25)
  ────────────────────────────────────────────────────────────────────
  Swiftness          3:00           2:15             0:45
  Swiftness (ext)    8:00           6:00             2:00
  Swiftness II       1:30           1:07.5           0:22.5
  Strength           3:00           2:15             0:45
  Strength (ext)     8:00           6:00             2:00
  Strength II        1:30           1:07.5           0:22.5
  Healing I          0              0 (instant)      0 (instant)
  Healing II         0              0                0
  Fire Resistance    3:00           2:15             0:45
  Fire Resist (ext)  8:00           6:00             2:00
  Water Breathing    3:00           2:15             0:45
  Water Breath (ext) 8:00           6:00             2:00
  Regeneration       0:45           0:33.75          0:11.25
  Regen (ext)        1:30           1:07.5           0:22.5
  Regen II           0:22.5         0:16.88          0:5.63
  Night Vision       3:00           2:15             0:45
  Night Vis (ext)    8:00           6:00             2:00
  Invisibility       3:00           2:15             0:45
  Invis (ext)        8:00           6:00             2:00
  Jump Boost         3:00           2:15             0:45
  Jump Boost (ext)   8:00           6:00             2:00
  Jump Boost II      1:30           1:07.5           0:22.5

Non-potion sources:
  Beacon:          Speed, Haste, Resistance, Jump Boost, Strength —
                   Primary effect: infinite in range (while in beam)
                   Secondary effect: extended range, Regen I
                   Duration: while player is within beam range (no timeout)
                   Range: 20-50 blocks (depends on pyramid size)
  Golden Apple:    Regeneration II 0:05, Absorption I 2:00
  Enchanted GA:    Regeneration III 0:30, Absorption IV 4:00, Resistance I 0:05, Fire Resistance I 5:00
  Totem of Undying: Absorption II 0:05, Regeneration II 0:05 (activates on death prevention)
  Elder Guardian:  Mining Fatigue III 5:00 (applied within 50 blocks)
  Shulker bullet:  Levitation I 0:10
  Wither skull:    Wither I 0:10
  Cave spider:     Poison II 0:07
  Husk:            Hunger I 0:10
  Spectral arrow:  Glowing I 0:10 (per arrow hit)
  Arrows (tipped): potion effect at 1/8 base duration
  Lingering potion: 1/4 base duration (1.13+, not in 1.12.2)
```

### 11.3 Effect Stacking & Mechanics
```
Amplifier: effects stack by increasing the amplifier value
  - Level I = amplifier 0, Level II = amplifier 1, etc.
  - Higher amplifier = stronger effect
  - Multiple sources of same effect: only the HIGHEST amplifier applies
  - EXCEPTION: when two effects have same amplifier, the LONGER duration persists

Duration extension:
  - Applying the same effect while already active refreshes duration
  - If new amplifier is higher: replaces entirely
  - If new amplifier is lower: no change (lower ignored)
  - If same amplifier: duration extends (overwrites with new duration, additive? 
    actually it resets to the new potion's duration)

Interaction rules:
  - Strength + Weakness: canceled (net: min(0, strength_bonus - weakness_penalty))
    Actually they don't cancel entirely, both apply simultaneously
    Total melee modifier = 3 per strength level - 4 per weakness level
  - Speed + Slowness: both apply, net = (1+0.2×speed) × (1-0.15×slowness) of base
  - Haste + Mining Fatigue: cancel similarly
  - Invisibility + any armor: armor items are visible even with invisibility
  - Invisibility + glowing: glowing outline still visible
  - Night Vision + Blindness: can see in dark but still has black fog
  - Absorption + Health Boost: golden hearts appear as extra layer above health boost hearts
    Health Boost adds MAX HP (shown as light yellow at end of health bar)
    Absorption adds golden hearts that deplete before real health

Removal:
  - Milk bucket: removes ALL status effects (both positive and negative)
  - Dying: removes all effects
  - Cure: zombie villager curing removes weakness during conversion
  - Beacon: effects fade when leaving range (fade over 5 seconds)
  - Death: all effects removed on respawn

Modifier attributes per effect:
  Effects can modify entity attributes (not just hardcoded behavior):
    Speed:         generic.movementSpeed [+0.2 per level, multiply]
    Slowness:      generic.movementSpeed [-0.15 per level, multiply]
    Haste:         generic.attackSpeed [+0.1 per level, multiply]
    Mining Fatigue: generic.attackSpeed [-0.1 per level, multiply]
    Strength:      generic.attackDamage [+3 per level, add] (but swords use different)
    Jump Boost:    generic.jumpStrength [+0.1 per level, add]  (internal attribute)
    Weakness:      generic.attackDamage [-1 per level... actually -4, add]
    Luck:          generic.luck [+1 per level, add]
    Bad Luck:      generic.luck [-1 per level, add]
    Health Boost:  generic.maxHealth [+4 per level, add]
    Absorption:    AbsorptionAmount [+4 per level, add]

Effect particles:
  - Ambient: effect particles from beacon or command with ambient flag
    Ambient particles are HALF as many and more transparent
  - Show particles: display particle effects around entity
  - Show icon: display status effect icon on HUD (always shown for drinkable potions)

Implementation:
  - Each entity has a Map<EffectType, EffectInstance>
  - EffectInstance: { type, amplifier, duration, ambient, showParticles, showIcon }
  - On tick: decrement duration, apply effect if duration > 0
  - On duration <= 0: remove effect
  - Particles: spawn at random offset from entity center every 0.5-1 second
  - Icon: rendered in inventory screen in top-right list
```

### 11.4 Beacon Effects
```
Beacon activation:
  - Place beacon block on top of a pyramid (iron/gold/diamond/emerald blocks)
  - Pyramid sizes:
    Tier 1: 3×3 base (9 blocks) → range 20 blocks
    Tier 2: 5×5 + 3×3 (34 blocks total) → range 30 blocks
    Tier 3: 7×7 + 5×5 + 3×3 (83 blocks) → range 40 blocks
    Tier 4: 9×9 + 7×7 + 5×5 + 3×3 (164 blocks) → range 50 blocks
  - Fuel: iron ingot, gold ingot, emerald, diamond, netherite ingot (1.16+)
    Fuel must be placed in beacon GUI

Primary powers (choose 1):
  Speed I, Haste I, Resistance I, Jump Boost I, Strength I
  - Applied to players within range
  - Pulse interval: every 4 seconds (80 ticks)

Secondary powers (tier 4 only):
  Regeneration I: applied in addition to primary power
  - OR - boosted primary: Range × 2 (not duration, the effect itself is level II)
    Actually: secondary power option 1 = Regeneration I, option 2 = primary power level II
    e.g. Haste I → Haste II (with secondary boost)

Visual:
  - Beam of light from beacon to sky
  - Beam color: tinted by glass panes above beacon
  - Beam particle effects: sparkling effect at beacon
  - Effect UI: icon appears in player's HUD effect list while in range
  - Range indicator: player can see if in range (effect icon appears/disappears)
```

---

## P12 — Storage & World Saving

### 12.1 World Format Overview
```
Single world per IndexedDB database. Worlds are isolated by unique name/seed.
Format is custom binary (not Anvil/MCRegion) for browser compatibility.

Database name: "mclient_world_{worldId}"
  worldId is a UUID generated on world creation

Object stores:
  meta          → { key, value } pairs for world metadata
  player        → single entry: player data blob
  chunks        → { cx, cz, dimension, data } per chunk  
  region        → { rx, rz, dimension, data } for region files (optional)

Versioning:
  Database version: integer, incremented on schema change
  Format version: stored in meta, checked on load
  Migration: on version mismatch, attempt upgrade or reject world
```

### 12.2 IndexedDB Schema (Detailed)
```
Database: "mclient_world_{worldId}" (opened with version=N)

Store: meta
  KeyPath: "key"
  Records:
    key: "version"          value: format version integer (currently 1)
    key: "seed"             value: world seed (number or string)
    key: "gameMode"         value: 0=survival, 1=creative, 2=adventure, 3=spectator
    key: "difficulty"       value: 0=peaceful, 1=easy, 2=normal, 3=hard
    key: "spawnX"           value: number
    key: "spawnY"           value: number
    key: "spawnZ"           value: number
    key: "time"             value: world time (long, ticks)
    key: "dayTime"          value: day time (0-24000)
    key: "raining"          value: boolean
    key: "rainTime"         value: ticks until rain stops
    key: "thundering"       value: boolean
    key: "thunderTime"      value: ticks until thunder stops
    key: "lastPlayed"       value: timestamp (Date.now())
    key: "gameRules"        value: JSON object of gamerule overrides
    key: "generatorName"    value: "default" | "flat" | "largeBiomes" | "amplified" | "customized" | "debug_all_block_states"
    key: "generatorOptions" value: JSON string (for superflat/customized)
    key: "allowCommands"    value: boolean

Store: player
  KeyPath: "id" (always "player")
  Value (binary blob):
    - Player position: x, y, z (float64 × 3 = 24 bytes)
    - Player rotation: yaw, pitch (float32 × 2 = 8 bytes)
    - Player velocity: vx, vy, vz (float64 × 3 = 24 bytes)
    - OnGround: boolean (1 byte)
    - Sleeping: boolean (1 byte)
    - Health: float32 (4 bytes)
    - Food: int32 (4 bytes)
    - FoodSaturation: float32 (4 bytes)
    - FoodExhaustion: float32 (4 bytes)
    - Score: int32 (4 bytes)
    - XpLevel: int32 (4 bytes)
    - XpProgress: float32 (4 bytes)
    - XpTotal: int32 (4 bytes)
    - Inventory: 41 slots × (itemId, damage, count, tag) serialized
    - Armor: 4 slots (head, chest, legs, feet)
    - Offhand: 1 slot
    - EnderChest: 27 slots
    - ActiveEffects: list of effect instances (type, amplifier, duration, ambient, showParticles, showIcon)
    - SelectedSlot: int32 (0-8)
    - Dimension: int32 (0=overworld, -1=nether, 1=end)
    - Fire: int16 (ticks remaining on fire)
    - Air: int16 (ticks of air left for drowning)
    - FallDistance: float32 (4 bytes)
    - HurtTime: int16
    - DeathTime: int16
    - AttackTime: int16
    - SleepingTimer: int16 (0 if not sleeping)
    - **Total**: ~400-800 bytes per save (variable due to inventory)

Store: chunks
  KeyPath: ["cx", "cz", "dimension"]
  Index: ["dimension"] (to iterate all chunks in a dimension)
  Value (compressed binary blob):

Chunk binary format (after decompression):
  Header (8 bytes):
    [0-1]   format_version: uint16 (currently 1)
    [2-3]   data_length: uint16 (length of following data)
    [4-5]   block_count: uint16 (for tracking, not critical)
    [6-7]   flags: uint16 (bitfield: dirty=1, lit=2, generated=4, populated=8)
  
  Section data (16 sections, y=0-15, each section = 16×16×16 blocks):
    For each section:
      [0]     non_air_block_count: uint8 (0-255, 0 = all air, skip section)
      [1-2]   bits_per_block: uint8 (4, 5, 6, 7, 8, 12, 16)
              - 4 bits: 16-color palette (compact, 0-15)
              - 5-8 bits: up to 256 unique block states
              - 12 bits: up to 4096 block states (no palette)
              - 16 bits: direct block IDs
      [3-4]   palette_length: varint (or uint16)
      [5+]    palette: palette_length × block_state_id (uint16 or uint32)
      Next:    block_storage: 4096 entries × bits_per_block bits (padded to 64-bit)
      Next:    block_light: 2048 bytes (half-byte per block, 4096 blocks)
      Next:    sky_light: 2048 bytes (half-byte per block)
    
    If non_air_block_count == 0: skip entire section (assume all air)
  
  Biome data (256 entries):
    256 × biome_id (uint8) = 256 bytes each for biomes in 4×4 columns
    or int32 biome IDs (4 bytes each = 1024 bytes for 1.13+)
    In 1.12: 256 bytes (biome ID 0-255)
  
  Heightmap:
    256 × short (2 bytes) = 512 bytes
    Height value: 0-255 (highest non-air block in column)
  
  Block entities (variable):
    count: varint
    For each:
      x, y, z: int32 (relative to chunk origin)
      type: string (e.g. "Chest", "Furnace", "Sign")
      data: JSON/NBT blob (custom items, text, etc.)
  
  Entities (variable):
    count: varint
    For each:
      uuid: string (UUID v4)
      type: string (e.g. "Zombie", "Sheep", "Item")
      position: x, y, z (float32 × 3)
      rotation: yaw, pitch (float32 × 2)
      velocity: vx, vy, vz (float32 × 3)
      data: JSON/NBT blob (health, equipment, AI state, etc.)

Compression:
  - LZ4 (preferred): fast compression/decompression, good for browser
  - Deflate (fallback): smaller but slower, zlib wrapper
  - Compression level: LZ4 default (fast), Deflate level 6
  - Compression applied to ENTIRE chunk blob (not per section)
```

### 12.3 World List Management
```
World list stored in a SEPARATE database: "mclient_worlds"
  Store: worlds
  KeyPath: "id"
  Records:
    id:           UUID (string)
    name:         string (world display name)
    seed:         number or string
    gameMode:     string
    difficulty:   string
    lastPlayed:   timestamp
    icon:         base64 PNG thumbnail (64×64) of player's position
    fileSize:     estimated storage size in bytes
    version:      format version (for migration checks)

World creation:
  1. Generate UUID for world
  2. Create IndexedDB database: "mclient_world_{uuid}"
  3. Initialize meta store with seed, defaults
  4. Generate spawn chunks (radius=2)
  5. Create player data at spawn position
  6. Add to world list database
  7. Open world (load player, spawn chunks)

World deletion:
  1. Remove from world list database
  2. Delete IndexedDB database: "mclient_world_{uuid}"
  Note: IndexedDB.deleteDatabase is async, may need time to complete

World renaming:
  Update name in world list database only (internal name inside world stays same)

World copying:
  - Copy entire IndexedDB to new database (read all stores, write to new DB)
  - Generate new UUID
  - Update player position (optional: keep same or reset to spawn)
  - Add to world list with new name
```

### 12.4 Save/Load Cycle
```
World open (load sequence):
  1. Open IndexedDB database
  2. Read meta store → seed, time, dimension, game mode, difficulty
  3. Read player store → position, inventory, health, XP
  4. Initialize world generator with seed
  5. Load spawn chunks (radius 2: 5×5 = 25 chunks)
  6. Set player at loaded position
  7. Set world time, weather, game rules
  8. World is ready (loaded in ~200-500ms depending on chunk count)

Auto-save (every 30 seconds):
  1. Mark ALL loaded chunks as dirty (flag)
  2. For each dirty chunk in loaded area:
     a. Serialize chunk to binary format
     b. Compress with LZ4
     c. Store in IndexedDB
     d. Clear dirty flag
  3. Save player data (always, not just when dirty)
  4. Save meta store (time, weather, etc.)
  5. Commit transaction

Save throttling:
  - Batch chunk writes: 50 chunks per transaction
  - Max 3 concurrent write transactions
  - Queue remaining dirty chunks for next save cycle
  - Priority: chunks nearest to player save first

On exit (save & quit):
  1. Force-save ALL loaded chunks (ignore dirty flag, save everything)
  2. Save player data (exact position, inventory)
  3. Save meta store
  4. Close database connection
  5. Return to world list / main menu

Critical save triggers:
  - Player death: save immediately (so "recent death point" is persisted)
  - Portal travel: save before dimension switch
  - Major inventory change: save after closing inventory GUI
  - Achievement unlock: save immediately

Safety mechanisms:
  - Save in a single IndexedDB transaction per batch
  - If a write fails: queue for retry, don't crash
  - Corrupted chunk: skip loading, regenerate on next world open
  - Backup: keep last successful save metadata (autosave timestamp)
  - Recovery: on load, check player position validity (not inside solid block)
    If player is inside a block: move to nearest air block
    If player is below y=0: move to world spawn
```

### 12.5 Compression Strategy
```
LZ4:
  - Fast compression: ~200 MB/s decompress (browser native via wasm)
  - Compression ratio: ~2-3× on chunk data (typical)
  - Implement via: lz4-wasm or lz4js library
  - Block size: 64KB chunks

Deflate:
  - Slower compression: ~50 MB/s decompress
  - Better compression: ~3-5× on chunk data
  - Standard zlib wrapper
  - Fallback if LZ4 unavailable

Storage estimation:
  - Each chunk: ~1-5 KB compressed (all sections, entities)
  - Region (32×32 chunks): ~1-5 MB compressed
  - Active loaded area (25 chunks): ~25-125 KB in memory per tick
  - Full world (all explored chunks): up to hundreds of MB
  - IndexedDB limit: depends on browser (typically >50% of disk space)

Cache strategy:
  - Loaded chunks: keep in memory (uncompressed)
  - Unloaded chunks (out of range): save to IndexedDB, remove from memory
  - Recent chunks (unloaded within 60s): keep compressed in memory cache (LRU, ~1000 chunks max)
  - World icon: cache in memory, regenerate every 30s autosave
```

### 12.6 NBT-like Binary Format (for inventory/entity data)
```
Custom binary format inspired by NBT, flattened for performance:

Tag types:
  0x00  TAG_End
  0x01  TAG_Byte      (int8)
  0x02  TAG_Short     (int16, LE)
  0x03  TAG_Int       (int32, LE)
  0x04  TAG_Long      (int64, LE)
  0x05  TAG_Float     (float32, LE)
  0x06  TAG_Double    (float64, LE)
  0x07  TAG_Byte_Array (uint32 length + byte[length])
  0x08  TAG_String    (uint16 length + UTF-8 string)
  0x09  TAG_List      (tag_type uint8 + uint32 length + data)
  0x0A  TAG_Compound  (sequence of TAG_named fields, terminated by TAG_End)
  0x0B  TAG_Int_Array (uint32 length + int32[length])
  0x0C  TAG_Long_Array (uint32 length + int64[length])

Named field in compound:
  tag_type (uint8) + name_length (uint16) + name (UTF-8 string) + data

Item stack format:
  id:      Tag_Short (item ID / block ID)
  count:   Tag_Byte (1-64)
  damage:  Tag_Short (durability or metadata)
  tag:     Tag_Compound (optional, for enchanted items, potions, written books, etc.)
    Example compound fields:
      ench: List of { id: short, lvl: short }
      display: { Name: string, Lore: list(string) }
      RepairCost: int
      AttributeModifiers: list
      Unbreakable: byte
      HideFlags: int
      Potion: string (potion effect)
      StoredEnchantments: list (for enchanted books)

This format is used for:
  - Inventory slot serialization
  - Entity equipment
  - Block entity contents (chest items, furnace smelting progress)
  - Villager trades (not persisted directly)
  - Player ender chest

Key differences from Java Edition NBT (for performance):
  - All integers are little-endian (not big-endian like Java NBT)
  - String length prefix is uint16 (not uint16 like Java)
  - No root compound name (unlike Java NBT which requires root tag + name)
  - Floats and doubles use LE IEEE 754 (same as Java)
```

---

## P13 — Technical Architecture

### 13.1 Engine Loop
```
requestAnimationFrame:
  1. Compute deltaTime
  2. Accumulate → while ≥ 50ms:
     a. Process inputs
     b. Update player physics (movement, collision, fall dmg)
     c. Update block ticks (random + scheduled)
     d. Update liquid flow
     e. Update entity AI & movement
     f. Update redstone
     g. Update chunk loading/unloading
     h. Subtract 50ms
  3. Interpolate entity positions
  4. Update camera (view bobbing, FOV effects)
  5. Render:
     a. Skybox
     b. Frustum culling
     c. Sort transparent blocks
     d. Draw opaque geometry (front→back)
     e. Draw transparent geometry (back→front)
     f. Draw entities
     g. Draw particles
     h. Draw held item (first-person)
     i. Draw block highlight
     j. Draw HUD
     k. Draw overlays (fire, water, pumpkin, nausea, etc.)
  6. Update audio listener
```

### 13.2 Chunk Mesh Pipeline
```
1. Dirty flag → mark self + 3 neighbors
2. For each block:
   a. Check exposed faces (air/transparent neighbor)
   b. Add quad: vertices, UV (from atlas), normal, light (interpolated), color tint
   c. Separate transparent blocks for alpha sort
3. Buffer → upload to GPU
4. Animated textures: uniform time offset for water/lava/fire

Geometry types: full cube, slab, stair, fence, pane, wall, cross, JSON model
```

### 13.3 Lighting
```
Sky light: top=15, propagate down through air, reduced by transparency
Block light: flood fill from emitters, decrease 1/block
Smooth lighting: vertex light = average of 3 adjacent blocks
```

### 13.4 Collision Detection
```
Player AABB: 0.6w × 1.8h (sneak 1.65), eye 1.62
Swept AABB: X then Z then Y separation
Step-up: 0.5 blocks auto
Entity hitboxes: per type (width × height)
Block picking: DDA ray march, max 4.5m (survival) / 5.0m (creative)
```

### 13.5 Sky Rendering Details
```
Sky dome: hemisphere rendered behind all world geometry
  Day sky: gradient from blue (#78A7FF at zenith) to lighter blue near horizon
  Night sky: gradient from dark blue (#0A0A1E at zenith) to black at horizon
  Sunset/sunrise: orange/red/pink gradient near horizon, transitioning to day/night above

Sun:
  Texture: @minecraft/textures/environment/sun.png (16×16, circular yellow-white)
  Position: rotates along a circular path — rises in east (+X), sets in west (-X)
  Angle: 0° at dawn, 90° at noon, 180° at sunset, 270° at midnight
  Size: visible disc in sky, ~8° apparent diameter
  Brightness: full during day, invisible at night (below horizon)
  Glow: lens flare effect (subtle radial glow)

Moon:
  Texture: @minecraft/textures/environment/moon_phases.png (128×16, 8 phases in a row)
  Phase: 8 phases cycling every 8 in-game days (0=full moon, 4=new moon, etc.)
  Position: 180° offset from sun (rises when sun sets)
  Size: same as sun disc (~8°)
  Light: provides sky light 4 at night (moonlight)
  Phase affects: slime spawn rate in swampland (more slimes during full moon)

Stars:
  Number: ~1500 white dots distributed on inner sky dome
  Brightness: varies randomly (each star has slightly different alpha)
  Twinkle: subtle pulsing of alpha over time (sine wave per star)
  Visible: when sky light ≤ 4 (night, twilight, underground looking up)
  Rotation: slowly rotates with sky dome (same axis as sun/moon)
  Positions: fixed pattern based on world seed (same stars every night)
  Size: 1-2 pixels each
  Color: pure white with slight blue tint (varies by star)

Clouds:
  Texture: @minecraft/textures/environment/clouds.png (tileable, 256×256)
  Layer: flat plane at y=128 (overworld only)
  Coverage: varies by biome and weather
  Shape: noise-generated cloud pattern from texture
  Movement: drifts from +Z to -Z (north→south) at ~1 block/sec
  Rendering: Fast graphics = flat transparent plane; Fancy = volumetric alpha
  Weather: rain = 100% coverage (no blue sky cracks); clear = partial

Fog:
  Distance: varies by biome and weather
  Default: render distance (set in options, 2-32 chunks)
  Rain: fog distance reduced by ~30%
  Water fog: ~8 blocks in clear water, ~2 blocks in swamp/colored water
  Lava fog: ~3 blocks (dense orange fog)
  Blindness: fog distance = 2-3 blocks (thick black fog)
  Night: slight fog increase (darkness effect)
  Fog color: matches sky color, affected by biome (swamp brown, mesa warm)

End sky:
  Texture: @minecraft/textures/environment/end_sky.png (tileable starfield texture)
  Rendering: rotating starfield sphere, centered on player
  Color: dark purple-black background with white dots
  No sun, no moon, no clouds (void-like atmosphere)
  Movement: slow rotation (1 revolution per ~30 minutes)

Nether sky:
  No sky rendering (bedrock ceiling at y=127)
  Fog: dense dark red/orange fog at ~3-6 block distance
  Color: #100810 (very dark red-black)
  No sun, no moon, no stars, no clouds
  Ash particles floating upward (visual only, decorative)
```

### 13.6 Enchantment Glint & Entity Rendering Details
```
Enchantment glint rendering:
  Enchanted items (tools, armor, weapons, books) render with a rainbow shimmer overlay.
  Applied in two places: item sprite (inventory/HUD) and 3D item model (in world/hand).

  Shader technique:
    1. Render item normally (base texture)
    2. Render overlay pass: use enchantment glint texture (white noise / gradient pattern)
       - The glint texture is a 128×128 rainbow gradient (sine wave along both axes)
       - Texture coordinates are offset by time: offset = sin(time * 0.003 + 0.1 * incident)
       - Applied using a second UV set that scrolls over time
       - Color = glint_texture_sample × enchant_color (typically white→purple)
    3. Alpha blend overlay with base: glint is semi-transparent (α ~0.3-0.5)
    4. Only render for items with enchantment NBT or enchanted book

  GLSL fragment shader pseudocode:
    vec2 glintUV = v_uv + vec2(u_time * 0.003, u_time * 0.003);
    vec4 glintColor = texture2D(u_glintTexture, glintUV);
    glintColor.a *= 0.4;  // transparency
    gl_FragColor = mix(baseColor, glintColor, glintColor.a);

  Floating item animation:
    Items on ground bob up and down with sine wave:
      y_offset = sin(time * 0.05 + item_seed) * 0.05
    Items rotate slowly (yaw rotation):
      rotation = time * 0.05 + item_seed
    Items in item frames: flat against wall, no bobbing
    Items in hand (first person): slight idle sway (breathing animation):
      drift_x = sin(time * 0.02) * 0.001
      drift_y = sin(time * 0.02 + 1) * 0.001

  Damage animation rendering:
    Player hurt overlay: red vignette texture (from gui/icons.png) rendered at screen edges
      - Alpha = max(0, HurtTime / 10)  (fades over 10 ticks)
    Entity hurt flash: entity model turns red for 0.5s
      - Implemented as vertex color modulation: vec4(1, 0.2, 0.2, 1) override
      - Only applies when hurtTime > 0
      - Fades: factor = hurtTime / maxHurtTime

  Held item first-person rendering:
    Arm swing animation: item rotates from rest → forward → back over 0.4s (8 ticks)
      - Phase 1 (0-4 ticks): item moves forward and slightly up
      - Phase 2 (4-8 ticks): item moves back to rest (recovery)
      - Rotation: pitch 0→-90°→0, yaw 0→-45°→0
    Mining animation: same as attack but item shakes (rapid small oscillations)
    Eating animation: item moves toward mouth (position shift over 32 ticks)
    Block breaking animation: crack overlay (destroy_stage_0 through 9) on block face
      - 10 stages of cracking texture from @minecraft/textures/blocks/destroy_stage_*.png
      - Applied as decal on block faces being mined
      - Blends with block texture: crack is dark line overlay

  First-person hand rendering:
    Hand model: simple 2D sprite or low-poly 3D of arm + held item
    Position: bottom-right of view, slightly rotated
    Bobbing: moves with view bobbing (walking/running oscillation)
    Sneaking: hand lowers by ~0.1 units

  View bobbing:
    Walking: sinusoidal up/down + left/right oscillation
      - Frequency: tied to step cycle (1 cycle per 2 steps ≈ 1.5m traveled)
      - Amplitude: y_bob = 0.1 × sin(step_cycle × π), x_bob = 0.1 × sin(step_cycle × π / 2 + π/2)
    Sprinting: increased amplitude (1.5× walking), faster frequency
    Sneaking: minimal bobbing (0.2× walking amplitude)
    Swimming: no bobbing (smooth water movement)
    Disabled: options toggle "View Bobbing" → off = no bobbing
```

### 13.7 Chest Opening/Closing Animation
```
Chests (single, double, trapped, ender) have a lid opening/closing animation.
The lid is a separate model component that rotates on its hinge.

Model structure:
  - Chest has 2 parts: lid + base
  - Lid: rotated around a hinge edge (the back edge), opening angle 0°-90°
  - Base: static (lower half of chest)
  - Lock/ latch: small protrusion on the front of the lid
  - Double chest: two halves side by side (left half + right half), each with own lid

Animation states:
  - Closed: lid angle = 0° (resting on base)
  - Open: lid angle = 90° (fully open, standing vertical)
  - Opening: transitioning from 0° to 90° over ~10 ticks (0.5s)
  - Closing: transitioning from 90° to 0° over ~10 ticks (0.5s)
  - Lid angle is interpolated smoothly using easing:
    - Opening: ease-out (starts fast, slows near 90°)
    - Closing: ease-in (starts slow, speeds up near 0°)
    - Formula: angle = target × (1 - pow(1 - progress, 3))  (cubic ease)

Animation triggers:
  - Open: player right-clicks chest → lid opens immediately
  - Close: player closes chest GUI → lid closes after ~0.2s delay
  - Latch snap: when lid reaches fully open, there's a subtle "snap" effect
  - If chest is broken while open: lid stays at current angle (block replaced)

Double chest animation:
  - Both chests open simultaneously
  - Chests connected side-by-side (north-south or east-west orientation)
  - Left chest lid and right chest lid rotate independently but with same timing
  - Middle latch line is shared (gap between the two chests)

Implementation (block entity renderer):
  - Lid transformation: rotate around Z axis at hinge point
  - Render lid + base as separate meshes
  - Animation driven by lid_angle (0.0 to 1.0) stored in block entity data
  - lid_angle interpolated per render frame (not per tick) for smooth 60fps
  - Block entity stores:
    - lidAngle (float, current angle 0.0-1.0)
    - prevLidAngle (float, previous tick's angle for interpolation)
    - numPlayersUsing (int, count of players with GUI open)

Chest block entity rendering in world:
  - Uses custom model (not JSON model) — hardcoded box mesh
  - Texture: @minecraft/textures/entity/chest/normal.png (single)
    - double chest: @minecraft/textures/entity/chest/normal_left.png + normal_right.png
    - trapped chest: trapped_single.png, trapped_left.png, trapped_right.png
    - ender chest: ender.png (single only, no double variant)
  - UVs for lid: top face + front face + sides
  - UVs for base: bottom face + front/back face + sides
  - Lock: small square on front of lid
  - Chest shadow: rendered as part of the model (darkening on base/lid)
```

### 13.8 Mob Spawner Rendering (Rotating Entity + Flame)
```
Mob Spawner (ID 52) is rendered as a cage with a rotating miniature entity inside
and flame particles around it.

Cage rendering:
  - Model: wireframe cage (iron bars style)
  - 3D model with 4 pillars + horizontal rings (top and bottom)
  - Texture: @minecraft/textures/entity/spawner.png (semi-transparent cage texture)
  - Blending: alpha transparency on cage (shows blocks behind)
  - Cage is static (does not move or rotate)

Rotating entity:
  - Inside the cage: a miniature version of the spawner's current entity spins
  - Rotation: continuously rotates around Y axis at ~1 revolution per 4 seconds
  - Scale: ~0.5× normal entity size (inside cage)
  - Hardcoded models for common entities (zombie, skeleton, spider, etc.)
  - If entity model is not available: render a placeholder (pig or question mark block?)
  - Entity does NOT animate (no walking/swimming animation — just hovers and rotates)
  - Entity is centered in the cage (vertical center of cage interior)
  - Entity hovers slightly: gentle up-down float (~0.05 block amplitude, 1s period)
  - No shadow cast by the rotating entity

Flame animation:
  - Fire particles rotate around the spawner cage (4 flames? 8 flames?)
    - Actually: 4 flame sprites positioned at the 4 corners of the cage
    - Flames rotate in a circle around the spawner (constant speed ~2s per revolution)
  - Each flame is a billboard sprite (always facing camera)
  - Texture: fire_layer_0.png (animated flame sprite)
  - Flame size: ~0.3 blocks
  - Flame flicker: slight scale oscillation (sin(time + offset) * 0.05)
  - Flame color: orange/yellow (tinted fire texture)
  - Flames rotate in pairs: opposite corners move together
  - No flame: spawner without entity set (spawner with no mob type → invisible entity, no flames)

Performance notes:
  - Rotating entity and flames are rendered every frame (not batched with chunk mesh)
  - Entity inside: render as a separate draw call (small entity model)
  - Flames: 4 billboard quads, alpha-tested
  - Spawner cage: single mesh (could be batched with static world geometry if needed)
  - For distant spawners (> 16 blocks): entity can be hidden, only cage remains visible
```

### 13.9 Ender Chest Starfield Animation
```
Ender Chest (ID 130) has a unique animated starfield texture on its front and top faces.
The starfield shifts slowly, creating a space/void-like appearance.

Visual:
  - Base/body: obsidian-like dark block with purple/turquoise highlights
  - Front face (lock side): animated starfield texture
  - Top face: also has animated starfield
  - Lid: same obsidian-like texture as base, with purple accents
  - Frame: obsidian corner trim (dark purple-teal)
  - Inner surface: starfield animation (continuously moving points of light)
  - The lock/latch on the front has a purple glow

Starfield texture animation:
  - Texture: derived from end_portal.png or a dedicated ender_chest.png with starfield
  - Actually uses ender_chest.png texture with:
    - The starfield portion of the texture scrolls over time
  - Scroll direction: UV offset is shifted each frame:
    - u_offset = time * 0.002 (slow horizontal drift)
    - v_offset = time * 0.001 (slower vertical drift)
  - Stars are white/dots on dark purple background
  - Stars twinkle: individual star brightness oscillates (pre-baked into texture animation)
  - Speed: very slow (~1 cycle per 10-20 seconds)
  - The starfield covers the front face and the interior (visible when lid is open)

Lid animation:
  - Ender chest lid opens similarly to normal chest (see 13.7)
  - Same hinge rotation (0° closed → 90° open)
  - Opening speed: same as normal chest (~10 ticks)
  - When lid opens: the starfield interior is revealed
  - When lid closes: starfield is hidden

Implementation notes:
  - Ender chest uses the same block entity renderer as normal chest
  - Unique texture: @minecraft/textures/entity/chest/ender.png
  - Starfield animation: achieved by offsetting UV coordinates on the front face
  - Separate mesh for the obsidian frame (static) vs starfield face (animated)
  - Ender chest does NOT have a double chest variant (always single)
  - Purple portal-like particles emit from ender chest when opened (ambient particle effect)
```

### 13.10 Shulker Box Lid Animation
```
Shulker Box (IDs 219-234, 16 colors) is a portable storage block that opens like a chest
but with a unique "pop-up" lid animation. All 16 color variants share the same animation.

Model structure:
  - Shulker box model has 2 parts: base (box body) + lid
  - Lid rotates UPWARD from one edge (like a hinged box opening upward)
  - Color variant: tinted based on color (white, orange, magenta, etc.)
  - Texture: shulker_top_<color>.png for lid, shulker_<color>.png for sides/bottom
  - Default (undyed): purple shulker box
  - All 16 dyed variants: same model, different textures

Animation states:
  - Closed: lid is flush with the top of the box (angle 0°)
  - Fully open: lid is vertical (90° angle from closed position)
  - Opening: lid hinges upward over ~10 ticks (0.5s)
  - Closing: lid hinges downward over ~10 ticks (0.5s)
  - Lid pivot: the back edge of the top face (opposite the front latch)

Animation curve:
  - Opening: ease-out cubic (fast start, slow finish)
  - Closing: ease-in cubic (slow start, fast finish)
  - lid_progress = 0.0 (closed) to 1.0 (open)
  - Lid angle = lid_progress * 90°
  - Visual: the lid rotates around its hinge axis (Z axis when facing the front)

Behavior:
  - Opens when player right-clicks the shulker box
  - Closes when player closes the GUI
  - Shulker box can be pushed by pistons (even with items inside)
    - Piston push: if shulker box is open and piston pushes it → lid snaps closed instantly
  - Shulker box items: lid shows raised/open when in item frame? No, item form is always closed
  - Break: if shulker box is broken while open → box drops with items inside, no animation
  - Sticky piston: can pull shulker box (contents preserved)

Implementation:
  - Same block entity renderer pattern as chest (lid angle interpolation)
  - Block entity stores: lidAngle, prevLidAngle, color
  - Render:
    1. Base: static cube minus the top face (replace top with lid mount area)
    2. Lid: rotated around hinge edge
    3. Color applied: multiply texture with color tint (or use separate texture per color)
  - The hinge is at the top-back of the box (so lid opens UP, not sideways like chest)
  - Lid rotation axis: along the X axis (local) for north-south orientation
  - On top of the lid: the shulker's "head" (small bump/decorative texture)

Shulker entity connection (visual only):
  - Shulker box block → NOT related to Shulker mob
  - The block and mob share visual design (purple shell-like texture)
  - Block does NOT move or attack — purely decorative storage block
```

---

## P14 — Implementation Phases

### Phase A: Skeleton (Week 1-2)
```
- Vite + TS project setup
- Engine loop (rAF, delta)
- Camera + pointer lock
- WebGL: clear, basic shader, single chunk flat world
- WASD + gravity
- Block raycast + break/place
```

### Phase B: World Foundation (Week 3-6)
```
- Noise terrain gen (heightmap + caves)
- Chunk system (mesh, load, unload, render distance)
- Block face culling
- Texture atlas
- 3 blocks: dirt, grass, stone
```

### Phase C: Core Gameplay (Week 7-12)
```
- All basic blocks (natural, wood, stone variants)
- Tools + tier system + break times
- Creative inventory (12 tabs)
- Survival inventory + hotbar
- HUD (health, hunger, armor, XP, hotbar)
- Crafting table (3×3) + recipe book
- Furnace + smelting
- Chest (single + double)
```

### Phase D: Player Mechanics (Week 13-16)
```
- Sprint/sneak/jump (exact values)
- Fall damage + reduction blocks
- Food/saturation/exhaustion
- Natural regen + starvation
- Status effects (all 27)
- XP orbs + enchanting table
- Anvil + brewing stand
```

### Phase E: Mobs (Week 17-24)
```
- Entity base: pos, vel, hitbox, AI, pathfinding
- Item entities (drop, pickup, merge, despawn)
- XP orbs
- Passive: sheep, cow, chicken, pig, rabbit
- Hostile: zombie, skeleton, creeper, spider
- Enderman, slime, cave spider
- Horse/donkey/mule (stats, taming, breeding)
- Parrots + ocelots/cats (taming)
- Mob spawning (natural + spawner)
- Mob AI: wander, chase, flee, melee, ranged
- Animal breeding + growth mechanics
```

### Phase F: Advanced Systems (Week 25-32)
```
- Redstone: dust, torch, repeater, comparator, piston
- Redstone: lever, button, plates, doors, trapdoors
- Water + lava flow
- Farming (till, plant, grow, harvest)
- Fishing (fish, junk, treasure loot tables)
- Villager trading (all 7 professions)
- Enchanting table (bookshelf formula, weights)
- Anvil (repair, rename, prior work penalty, combining)
- Brewing stand (blaze powder fuel)
- Beacon pyramid (tiers, effects, payment)
```

### Phase G: UI Polish (Week 33-36)
```
- Main menu with panorama
- Title screen + world list
- Create new world screen
- Options (all tabs)
- Pause menu
- F3 debug screen
- Advancements tree + toasts
- Statistics
- Death screen + respawn
- Loading screen
```

### Phase H: Dimensions (Week 37-40)
```
- Nether terrain + biome + mobs
- Nether Fortress
- The End terrain + dragon fight
- End City + End Ship (elytra)
- Portals (obsidian frame + lighting)
- Bed explosion in wrong dimension
```

### Phase I: Redstone & Mechanics (Week 41-44)
```
- Full redstone: all components tested
- Note block instrument mapping
- Dispenser/dropper item interactions
- Rail + minecart system (5 types)
- Daylight detector
- Observer BUD behavior
- TNT chain explosions
- Explosion mechanics (TNT, creeper, bed, end crystal)
- Boat mechanics + minecart speeds
- Fireworks crafting + flight
- Armor stand + item frame + painting + lead mechanics
```

### Phase J: Content Complete (Week 45-48)
```
- All 400+ blocks with correct properties (no 1.13+ content)
- All 650+ items
- All mobs with drops/AI (incl. illusioner, killer bunny)
- All crafting/smelting/brewing recipes
- All enchantments + effects
- World gen all biomes + structures
- Scoreboard + teams system
- Full command set (53 commands)
- All 22 gamerules
- Map system (zoom, cloning, colors)
- Save/load with IndexedDB (full player NBT structure)
- Tutorial hints + splash text + Minceraft egg
```

### Phase K: Polish & Performance (Week 49-52)
```
- Sound system complete (block, entity, music, ambient, UI)
- Particle system (break, fire, smoke, rain, enchant, portal)
- Chunk loading prioritization (near→far)
- Dirty re-mesh optimization
- LOD / render distance 2-32
- Web Workers for chunk gen
- Memory leak fixes
- FPS optimization (draw calls, buffer uploads)
- Cross-browser testing (Chrome, Firefox, Edge)
```

---

## P15 — Performance Targets & Optimization

### 15.1 Frame Rate Targets
```
Target: Solid 60 FPS on mid-range hardware (2024 laptop/desktop)
  - Minimum: 30 FPS at render distance 8
  - Optimal: 60 FPS at render distance 16
  - Target: 60 FPS at render distance 32 (with all optimizations)

Frame budget (60 FPS = 16.67ms per frame):
  Game logic (tick):  3-5ms  (in Web Worker if possible, main thread fallback)
  Chunk mesh:         2-4ms  (deferred to worker, async upload)
  Render setup:       1-2ms  (frustum cull, sorting by distance, material batching)
  Opaque geometry:    3-5ms  (front→back ordering, instanced draws)
  Transparent geom:   1-2ms  (back→front, alpha blend, separate draw pass)
  Entities:           1-2ms  (batch + instanced rendering, culling)
  Particles:          0.5ms  (GPU particles, point sprites)
  HUD/UI:             0.5ms  (2D canvas overlay, composite)
  Sky + fog:          0.5ms  (fullscreen quad, single draw call)
  Post-processing:    0.5ms  (bloom for beacon/glow? Optional, skip if slow)
  Total budget:       12-22ms (some frames may exceed, but average must hold 16.67ms)

60 FPS breakdown graph:
  [game tick][chunk mesh][setup][opaque][transparent][entities][particles][UI][sky]
  |--3-5ms--|---2-4ms---|---1ms----|--4ms--|--1ms--|---1ms---|---0.5ms-|-0.5ms-|-0.5ms-|
  ======16.67ms total=============

Fallback modes:
  - 30 FPS (33.33ms budget): allow 2× the time for each category
    Render distance reduced to 8, particles halved, entities culled more aggressively
  - Unlimited FPS: no frame cap, render as fast as possible (use for benchmarks)
  - VSync: match display refresh rate (60Hz default, 120/144Hz supported)

Draw call budget:
  Opaque chunks:   ≤ 200 draw calls (one per chunk material, merged where possible)
  Transparent:     ≤ 50 draw calls (water, glass, ice — sorted manually)
  Entities:        ≤ 100 draw calls (instanced batches, max 16 per entity type)
  Particles:       1-2 draw calls (point sprites, instanced)
  Sky/UI:          2-4 draw calls
  Total:           ≤ 350 draw calls per frame

Vertex/triangle budget:
  Per chunk:       ~200-5000 triangles (varies by terrain complexity)
  Total:           ≤ 2M triangles per frame at render distance 16
                   ≤ 5M triangles per frame at render distance 32
  Entity:          ≤ 100K triangles (all entities combined)
```

### 15.2 Memory Targets
```
Total memory: ≤ 500 MB peak usage
  - JavaScript heap:        ≤ 150 MB (includes block data, entities, game state)
  - GPU texture memory:     ≤ 100 MB (atlassed + mipmaps + entity textures)
  - GPU geometry buffers:   ≤ 100 MB (chunk meshes, ~50KB per chunk average)
  - Audio buffers:          ≤ 50 MB  (loaded on demand, decoded PCM)
  - IndexedDB:              ≤ 100 MB (world saves, progressive on-disk)

Per-chunk memory budget (CPU side):
  Block data (palette):    2-4 KB (palette-based, min 2 bits per block for simple chunks)
    Air-only chunks:       24 bytes (full air palette, 1 entry)
    Single biome chunks:   2 KB (single palette entry + block IDs)
    Complex chunks:        4 KB (full palette with 20+ block types)
  Light data:              4 KB (2 bytes per block: sky + block light nibbles)
  Biomes:                  256 bytes (1 byte per column, 16×16)
  Heightmap:               512 bytes (2 bytes per column, 16×16, top block Y)
  Block entities:          0-8 KB (chest, furnace, spawner NBT data)
  Entities:                0-16 KB (mobs in chunk, NBT-like data)
  Mesh data (pending):     16-64 KB (vertex + index buffers before GPU upload)
  Total per loaded chunk:  ~25-90 KB (average ~50 KB CPU-side)

Per-chunk memory budget (GPU side):
  Vertex buffer:           8-32 KB (16 bytes/vertex × 500-2000 vertices)
  Index buffer:            2-8 KB  (4 bytes/index × 500-2000 indices)
  Total per chunk (GPU):   10-40 KB (average ~25 KB)

Render distance memory scaling:
  8 chunks  (17×17=289):   ~7 MB  GPU mesh + ~14 MB CPU data
  16 chunks (33×33=1089):  ~27 MB GPU mesh + ~54 MB CPU data
  24 chunks (49×49=2401):  ~60 MB GPU mesh + ~120 MB CPU data
  32 chunks (65×65=4225):  ~106 MB GPU mesh + ~211 MB CPU data

Entity memory budget:
  Per entity:              200-2000 bytes (varies by complexity)
    Item entity:           ~200 bytes (position, velocity, item stack)
    Passive mob:           ~500 bytes (AI state, health, NBT)
    Hostile mob:           ~800 bytes (AI goals, equipment, path)
    Player:                ~2000 bytes (inventory, stats, effects, Ender Chest)
  Entity cap per dimension: 200 (soft cap, performance degrades beyond)
    Item entities:         max 100 per loaded area (beyond = merge stacks)
    XP orbs:               max 50 per loaded area (beyond = merge into larger orbs)
    Mobs:                  max 50 hostile + 50 passive per loaded area
    Projectiles:           max 20 arrows + 10 other projectiles per loaded area
  Entity memory total:     ≤ 400 KB (all entities loaded)

Texture memory:
  Block atlas:            ~16 MB (4096×4096 RGBA, 8 bytes per px with mipmaps → ~21 MB)
  Entity atlas:           ~8 MB (2048×2048 RGBA, with mipmaps → ~11 MB)
  Item atlas:             ~4 MB (1024×1024 RGBA)
  GUI atlas:              ~2 MB (512×512 RGBA)
  Sky textures:           ~1 MB (skybox faces, sun, moon)
  Environment:            ~2 MB (clouds, particles, misc)
  Total textures:         ~40 MB GPU (well under 100 MB cap)

Audio memory:
  Background music:       ~20 MB (10 tracks × 2 MB average MP3→PCM)
  Sound effects:          ~15 MB (200+ sounds, short, compressed Ogg → PCM on load)
  Ambient:                ~5 MB (cave, nether, end ambience loops)
  Total audio:            ~40 MB peak (loaded lazily, unloaded on dimension change)
  Memory optimization:    keep only current-dimension sounds loaded
                          pre-load: dimension change triggers sound cache swap
```

### 15.3 Optimization Techniques
```
Chunk meshing:
  Greedy meshing: combine adjacent coplanar faces with same texture
    Algorithm: for each 16×16 slice of chunk (3 axes: Y, X, Z directions)
      1. Build 2D grid of face visibility (is face exposed to air?)
      2. Scan grid left→right, top→bottom, merge contiguous quads
      3. Output: one quad per merged region (4 vertices, 6 indices)
    Reduction: greedy meshing reduces vertex count by 60-80% vs naive meshing
      Naive: 6 faces per visible block = up to 24K vertices per chunk
      Greedy: typically 500-3000 vertices per chunk
  Face culling: skip faces adjacent to opaque blocks (interior faces)
    Check: if neighbor block is opaque (full cube, not transparent), skip face
    Also skip: faces at chunk boundary if adjacent chunk not loaded
      (adjacent chunks will mesh their own boundary faces)
  Dirty flag: track chunk modification state
    CHUNK_CLEAN: no changes since last mesh
    CHUNK_DIRTY: block changed within chunk → remesh this chunk
    CHUNK_NEIGHBOR_DIRTY: neighbor changed → remesh this chunk (boundary faces only)
    CHUNK_INVALID: chunk unloaded or regenerated → full remesh
    Propagation: block change at (x,y,z) → mark chunk(x,z) dirty + neighbors
  Mesh pooling: reuse typed arrays for vertex/index data
    Pool size: pre-allocate pool of 64 ArrayBuffers (64KB each)
    On mesh generation: acquire buffer from pool → fill → transfer → return to pool
    Avoids GC pressure from frequent allocations during chunk loading
  Priority queue: nearest chunks first for initial load
    Ordered by: distance² from camera center to chunk center
    Processing: sort pending chunks by priority, process top N per frame
    Movement: re-prioritize when player moves (chunks behind get lower priority)
  Partial remesh (boundary only):
    When neighbor chunk changes: only remesh the 2 face layers adjacent to boundary
    Saves ~80% of meshing work for boundary updates

LOD (Level of Detail) system:
  LOD levels (optional, for high render distance):
    LOD 0 (0-16 chunks): full detail mesh (16×16×256 block faces)
    LOD 1 (16-24 chunks): merged mesh (8×8×128 block quads, 2×2 block merging)
      Merging: treat 2×2×2 block groups as single block, use dominant material
      Vertex count: ~25% of LOD 0
    LOD 2 (24-32 chunks): block-colored quads (16×16×64, averaged colors per cell)
      No texture: just face color based on average block color in cell
      Vertex count: ~6% of LOD 0
    LOD 3 (32+ chunks): heightmap-only mesh (1 quad per column, colored by biome)
      Minimal geometry: 256 quads per chunk
  Transition: crossfade between LOD levels (alpha blend over 0.5s)
  Storage: LOD meshes stored separately from full mesh, updated less frequently
    LOD 1: update every 5 seconds (not per block change)
    LOD 2+: update every 30 seconds (mostly static)
  When to use: only activate LOD if render distance > 16
    If render distance ≤ 16: only LOD 0 (no LOD overhead)
  Implementation: separate mesh per LOD level, rendered with adjusted draw distance

Texture atlas:
  Layout: single atlas image for all block textures
    Dimensions: 4096×4096 pixels (supports up to 256×256 individual 16×16 textures)
    Grid: 256 columns × 256 rows of 16×16 textures? Actually 4096/16 = 256 per side
      Total capacity: 65536 textures (more than enough for ~1000 block textures)
    Actual used: ~2000 individual textures (blocks, items: 16×16 each)
    Padding: 1 pixel border around each texture (prevents bleeding at mip levels)
    Border color: average edge color of the texture (replicate edge pixels)
  Atlas generation:
    Load all block textures (PNG from assets)
    Sort by usage frequency (most-used → center of atlas for better mip behavior? Actually
      position doesn't matter for mip quality, just pack efficiently)
    Pack into atlas: row-by-row, left to right, top to bottom
    Generate mipmaps: GPU-generated (gl.generateMipmap) or pre-baked
    Upload: single gl.texImage2D call with full atlas
  UV mapping:
    Each block face maps to a sub-rectangle in the atlas
    UV coordinates stored per vertex (2 shorts, precision: 1/65536 of atlas)
    UV offset per block face: calculated from atlas position
  Mipmap filtering:
    Min filter: LINEAR_MIPMAP_LINEAR (trilinear, best quality at distance)
    Mag filter: NEAREST (pixel-art look for Minecraft blocks at close range)
    Anisotropic filtering: max 4× (balance quality/performance)
  Texture variations: for blocks with multiple textures (dirt, grass, stone):
    Each variant gets its own slot in the atlas
    Face assignment: randomly chosen per block face during chunk meshing
    Uses world-seeded RNG based on block position for consistent appearance

WebGL renderer:
  Renderer choice: Three.js (higher-level abstractions, easier development)
    Fallback: pure WebGL2 if Three.js overhead is too high (benchmark both)
    Three.js benefits: scene graph, camera, frustum culling, material system, loader support
    Three.js costs: ~500KB bundle, abstraction overhead (~5-10% CPU time)
    Decision: use Three.js for initial build, profile, port hot paths to raw WebGL if needed
    Raw WebGL2: only for custom shaders (sky, water, chunk mesh)
  Indexed geometry: ELEMENT_ARRAY_BUFFER for all meshes
    Vertex format: tightly packed 16 bytes per vertex
      position: 3 × int16 (signed, 1-unit precision, range [-32768, 32767])
      uv: 2 × uint16 (atlas UV, 1/65536 precision)
      light: 2 × uint8 (sky light + block light, 0-15 encoded as 0-255)
      color: 4 × uint8 (RGBA tint, for biome coloring and AO)
    Index format: uint32 (allow >65535 vertices per chunk mesh)
  Instanced rendering: for repeated geometry (tall grass, fences, torches, pressure plates)
    Use THREE.InstancedMesh or custom instanced draw calls
    Max instances per draw: 256 (keep within uniform array limits)
    Per-instance data: model matrix (4×4 float) + light data (4 bytes)
  Frustum culling:
    Per-chunk AABB test against view frustum (6 planes)
    Test: check if any corner of chunk AABB is inside frustum
    Optimization: skip culling test for chunks fully within near distance (< 32 blocks)
    Update: recalculate frustum planes on camera rotation/movement
    Frequency: every frame (fast, 6 plane-point tests per chunk)
  Occlusion culling:
    Software occlusion: ray-based from camera to chunk center
    Check: if ray passes through solid terrain (opaque blocks), chunk is occluded
    Implementation:
      1. Project chunk onto screen (screen-space bounding box)
      2. Read depth from previous frame's depth buffer (or maintain software depth map)
      3. If chunk's minimum depth > depth buffer value → occluded
    Simpler alternative: skip chunks behind nearest terrain wall (directional check)
      Cast ray from camera through chunk center
      If ray hits opaque blocks at close range (< chunk distance), cull chunk
    Conservative: only cull if chunk is fully behind solid geometry
    Update: every 5 frames (not per frame, expensive)
  Shader optimization:
    Pre-calculate lighting into vertex colors (no per-pixel diffuse/specular)
      Light values: sky + block light combined with smooth lighting AO
      Lighting is interpolated across faces via vertex colors
    Alpha test: discard fragments with alpha < 0.5 (leaves, glass, water)
      Avoids alpha blend overhead for semi-transparent blocks
      Water: use alpha blend (requires separate draw pass, sorted back-to-front)
    Use mediump precision in fragment shader (faster on mobile-class GPUs)
    Minimize texture lookups: single atlas lookup per fragment (no multi-texturing)
    Conditional discard: early exit for fully transparent fragments
  Sorting:
    Opaque geometry: front-to-back (minimize overdraw)
      Sort chunks by distance from camera, nearest first
      Enable depth testing + depth write
    Transparent geometry: back-to-front (correct alpha blending)
      Sort individual transparent quads by distance, farthest first
      Disable depth write, enable depth testing + alpha blend
      Special case: water is sorted per-chunk (not per-quad, for performance)
  Vertex format: tight packing (16 bytes per vertex)
    Each vertex: 16 bytes total (was 32 bytes with floats before packing)
    Position: 3 × int16 = 6 bytes (range ±32768 blocks, 1-block precision is sufficient)
    UV: 2 × uint16 = 4 bytes (atlas UV, 1/65536 of atlas — sub-pixel precision)
    Light: 2 × uint8 = 2 bytes (sky light nibble + block light nibble, stored as byte each)
    Color: 4 × uint8 = 4 bytes (RGBA tint for biome grass/water coloring)
    Note: positions need offset for chunk origin within the world (subtract chunk base coords,
    store local position as int16 relative to chunk base)

Entity rendering:
  Instance all similar entities (batch sheep, cows, etc.)
    Each entity type: 1 instanced mesh draw call
    Per-instance: position, rotation, scale, texture index (uniform or attribute)
  LOD: distant entities rendered as billboard sprites
    Transition distance: 64 blocks (full 3D) → 128 blocks (billboard)
    Billboard: single quad facing camera, entity texture applied
    Texture: rendered entity frame (captured from 3D model at 8 angles)
    Billboard size: scales with distance (fixed screen-space size)
  Cull entities outside render distance
    Also cull entities behind solid geometry (player cannot see)
    Check: frustum culling per entity AABB
  Animation: skeletal animation via uniform blend weights (pre-computed per frame)
    Max bones: 24 per entity (skinned mesh)
    Animation frames: stored as array of bone transforms (quaternion + position)
    Interpolation: linear between keyframes (no blending between animations for perf)
  Entity texture atlas: all entity textures in single 2048×2048 atlas
    Similar to block atlas, but for mob/entity skins
    UV mapping per entity from atlas coordinates

Particles:
  GPU-based particle system using point sprites (gl.POINTS)
    Max particles: 5000 (pre-allocated buffer)
    Particle vertex: position (3×float16), velocity (3×float16), life (float16), size (float16),
                    color (4×uint8) = 20 bytes per particle
    Update: compute shader or transform feedback (update on GPU each frame)
    Fallback: CPU update if GPU transform feedback not available
  Emitter types:
    Block break: 8 particles per broken block, 0.5s lifetime
    Block dig: 4 particles per hit, 0.2s lifetime
    Crit: 3 particles, 0.5s lifetime
    Enchant: 5 particles, 1s lifetime
    Portal: 2 particles, 2s lifetime (float upward)
    Rain: 100-500 particles, 2s lifetime (see P9D)
    Snow: 50-200 particles, 5s lifetime (see P9D)
    Lava drip: 1 particle per block, 0.5s lifetime
    Water drip: 1 particle per block, 0.5s lifetime
    Smoke: 1 particle, 2s lifetime
    Explosion: 50 particles, 1s lifetime
    Firework: 100 particles, 2s lifetime
  Particle caps:
    Total: 5000 max
    Per-emitter: based on importance (explosions get priority)
    Distance culling: particles > 64 blocks from camera are not spawned
    Culling: particles behind camera clipped early
  GPU update: use transform feedback (WebGL2) or custom ping-pong buffers
    Each frame: update positions based on velocity + gravity
    Kill particles with life <= 0 (recycle to new emissions)

Asset loading:
  Lazy loading: only load textures/sounds when needed
    Block textures: all loaded at startup (small, baked into atlas)
    Entity textures: loaded when entity type first appears
    Sound effects: loaded on first play (cached for reuse)
    Music: loaded when dimension changes or music track changes
  Preload: critical textures on world join
    Critical: sky textures, GUI textures, player model, held item models
    Load these synchronously during world load screen
  Async decode: audio via AudioContext.decodeAudioData
    Loading: fetch ArrayBuffer → decodeAudioData → store as AudioBuffer
    Caching: Map<string, AudioBuffer> (unused sounds evicted on memory pressure)
  Texture compression: use S3TC/DXT if available (EXT_texture_compression_s3tc)
    Fallback: uncompressed RGBA (higher memory but always works)
    Compression: pre-compress textures offline? Not needed for 16×16 textures
    Rationale: 16×16 textures are so small that compression isn't needed
  Preload completion: world load screen shows progress bar (asset loading %)
    Steps: load block textures → atlas generation → GUI textures → entity textures → sounds
    Progress: 0-100%, updated each asset category completion

Workers:
  Chunk generation in separate Web Worker
    Worker types:
      - TerrainWorker: terrain noise, biome generation, feature placement
      - MeshWorker: greedy meshing, vertex buffer generation
      - LightWorker: light propagation (sky light + block light)
    Or combine all into one worker per chunk:
      ChunkWorker: generates terrain + mesh + light for one chunk
        Simpler to manage, but each chunk takes longer
  Worker pool: 2-4 workers for high-churn situations (moving fast, flying)
    Pool management:
      - Idle workers pick from pending chunk queue (FIFO with priority)
      - Busy workers: currently processing a chunk
      - Max active: 4 (limited by CPU cores)
  Communication:
    Main → Worker: chunk coordinates + world seed (+ needed neighbors data)
    Worker → Main: mesh buffer (ArrayBuffer via Transferrable) + metadata
    Transfer: use postMessage with Transferrable objects (zero-copy)
    SharedArrayBuffer: optional, for shared world data (if enabled in browser)
      Allows all workers to read block palette without copying
  Worker responsibilities:
    TerrainWorker:
      - Generate heightmap from noise (3-5 octaves, 2D)
      - Apply biome modifiers (temperature, humidity → biome map)
      - Carve caves (3D noise, 3 octaves)
      - Carve ravines (line-based, depth function)
      - Place ore veins (per-biome distribution)
      - Place surface features (grass, flowers, trees, etc.)
      - Generate block palette from terrain data
      Output: block palette (Uint8Array or Uint16Array × 65536 entries)
    LightWorker:
      - Input: block palette (from TerrainWorker)
      - Initialize sky light: 15 for air/exposed blocks, 0 for opaque
      - Propagate sky light: flood-fill from top, decreasing by 1 per block
      - Initialize block light: emit from light sources (torches, glowstone, etc.)
      - Propagate block light: flood-fill from sources, decreasing by 1 per block
      - Calculate smooth lighting + ambient occlusion
      Output: light data (Uint8Array × 65536 entries, each: sky light nibble + block light nibble)
    MeshWorker:
      - Input: block palette + light data + texture atlas UV map
      - Run greedy meshing (3 axes)
      - Generate vertex buffer: position, UV, light, color
      - Generate index buffer
      Output: vertex ArrayBuffer + index ArrayBuffer (Transferrable)
  Schedule:
    Player moves → calculate new chunks needed → queue for TerrainWorker
    TerrainWorker done → queue for LightWorker
    LightWorker done → queue for MeshWorker
    MeshWorker done → transfer to main → upload to GPU → render
    Pipelining: all 3 workers can run simultaneously for different chunks
    Throughput: target 2-4 chunks per second fully processed
  Fallback: if Web Workers not available (old browser or restricted)
    Run chunk gen on main thread (uses requestIdleCallback)
    Split work: 5ms chunks of work per frame (avoid frame drops)
    Worse performance but functional

Memory optimization:
  Chunk unloading: chunks > render distance + 1 → free GPU memory immediately
    Unload distance: render distance + 2 (keeps a 2-chunk buffer for smooth scrolling)
    On unload: remove from GPU → remove from worker queue → free CPU data
    Cache: recently unloaded chunks kept in CPU-only cache (max 32 chunks)
      Cache hit: reload from cache (no regeneration, just remesh)
      Cache eviction: LRU, oldest unloaded first
    Cache size cap: 32 chunks (≈ 1.6 MB CPU data, worth it)
  Texture unload: only if not used by any loaded chunk
    Since block textures are atlassed: always loaded
    Entity/item textures: unload if no entity of that type loaded
    Audio unload: unload music tracks when switching dimensions
  Garbage collection: minimize allocations per frame
    Object pooling:
      - Chunk mesh data: pool of 64 ArrayBuffers
      - Entity objects: pool of 100 entity instances (reuse on despawn/spawn)
      - Particle objects: pre-allocated 5000 particle slots
      - Block position structs: reusable temp array
    Allocation budget:
      - < 10 small objects per frame (mostly for events/triggers)
      - Zero large allocations per frame (chunk buffers use Transferrable = no extra copy)
    Frame timing: monitor GC pauses. If > 5ms pause detected, reduce allocation rate
  IndexedDB: throttle writes to 1 per 5 seconds (debounce)
    Batch changes: accumulate block changes, flush every 5 seconds
    Also write on: dimension change, player quit, critical events (death, advancement)
    Write strategy: incremental (only modified chunks, not full world)
    Compression: LZ4 (fast) or Deflate (compact) before write
```

### 15.4 Chunk Loading Performance
```
Target chunk load time: < 50ms per chunk on worker thread
  - Noise generation:    10-20ms (Simplex noise, 3-5 octaves for heightmap)
  - Biome generation:    2-5ms (temperature/humidity noise, biome lookup)
  - Cave/feature carve:  5-10ms (3D noise cave system, ore veins)
  - Block population:    5-10ms (ore gen, cave carve, feature placement)
  - Light propagation:   5-15ms (sky light + block light flooding)
  - Mesh building:       10-20ms (greedy meshing + face culling)
  - Buffer upload:       < 1ms  (transfer to GPU via Transferrable postMessage)
  Total worker time:     ~40-80ms (parallel: terrain + light can overlap with mesh)

Chunk load pipeline (async, parallelizable):
  1. Request: main thread determines chunks needed
     Check: is chunk in cache? → skip to step 4
     Check: is chunk already queued? → skip (deduplicate)
     Queue: add to priority queue with distance-based priority
  2. Terrain gen: worker generates block data (terrain + features)
  3. Light calc: worker runs light propagation
  4. Mesh build: worker generates vertex/index buffers
  5. Transfer: postMessage with Transferrable ArrayBuffers
  6. GPU upload: main thread creates VBO/IBO, adds chunk to scene
  7. Render: chunk appears in next frame

Concurrent chunk loading: 2-4 chunks generated in parallel (worker pool)
  Pipeline depth: each worker can handle 1 chunk at a time
  With 3 workers and 3 stages per chunk: up to 9 chunks in flight simultaneously
  Actual throughput: 2-4 chunks per second (limited by light propagation, which is sequential-heavy)

Chunk loading priority:
  Radial: nearest to camera first (chunks within 2 chunks of player get highest priority)
  Directional: chunks in movement direction have higher priority
    Compute: dot(cameraDirection, chunkDirection). Positive = toward = higher priority
    Weight: 1.5× multiplier for chunks in movement direction
  Visual importance: chunks above horizon get slightly higher priority (player looks forward)
  Biome-based: no priority difference between biomes
  Queue management:
    Max queue size: 256 pending chunks (beyond that, drop lowest priority)
    Re-prioritize: when player moves, recalculate priorities (every 10 ticks)
    Deduplication: same chunk cannot be queued twice (check before enqueue)
  Max load: 8-16 chunks per second sustained (worker + transfer throughput)

Chunk update budget (per frame):
  Generation budget: ≤ 4ms on main thread (just queue management + GPU upload)
  Mesh upload: ≤ 1ms per chunk (bufferData is fast, just copy to GPU)
  Queue management: ≤ 0.5ms (priority sort, dedup checks)
  Total main thread chunk work: ≤ 5ms per frame

Initial world load:
  Load spawn chunks: immediate area (5×5 = 25 chunks) first
  Priority: center ring (3×3 = 9 chunks) → outer ring → remaining
  Target: playable within 3 seconds (first 25 chunks loaded and meshed)
  Full render distance loaded within 10 seconds (1089 chunks at rd=16)

Chunk loading visual feedback:
  Unloaded chunks: rendered as fog (solid grey or sky-colored)
  Loading chunks: fade in over 200ms once mesh is uploaded
  Failed chunks: stay as fog (retry once, then skip)
```

### 15.5 Three.js vs Raw WebGL2 Tradeoffs
```
Three.js approach:
  Benefits:
    - Scene graph + camera management (out of the box)
    - Material system (ShaderMaterial for custom, MeshBasicMaterial for simple)
    - Frustum culling (built into Object3D hierarchy)
    - Loaders (TextureLoader, FileLoader, etc.)
    - Render loop (requestAnimationFrame + clock)
    - Pointer/raycaster for block interaction
    - Performance: decent for 60fps Minecraft-like scenes
    - Faster development time (weeks vs months for custom renderer)
  Costs:
    - ~500KB bundle size (gzipped: ~150KB)
    - ~5-10% CPU overhead per frame (scene graph traversal, matrix updates)
    - Less control over draw call ordering (manual override needed for transparency)
    - Limited instancing support (InstancedMesh works but needs custom per-instance data)
  Decision: USE Three.js for initial build. Profile. Port hot paths if needed.

Raw WebGL2 approach (future optimization if Three.js is bottleneck):
  Benefits:
    - Zero overhead (no scene graph, direct draw calls)
    - Full control over buffer management, sorting, instancing
    - Smaller bundle (no library code)
    - Can implement custom occlusion culling more easily
  Costs:
    - Much longer development time
    - Manual camera math (projection, view matrix)
    - Manual frustum culling
    - Manual shader management (compile, link, uniform setup)
    - Manual texture management
  Migration path: keep Three.js scene graph for entities + UI, raw WebGL for chunk meshes
    Hybrid: Three.js handles camera, entities, sky, UI
    Raw: custom mesh rendering for chunk geometry (direct gl.drawElements)
    This gives 80% of performance benefit with 20% of the work

ShaderMaterial custom shaders (Three.js approach):
  Chunk shader: ShaderMaterial with custom vertex + fragment
    vertex: unpack position, apply UV offset, pass light + color to fragment
    fragment: sample texture atlas, apply light, apply fog
  Water shader: ShaderMaterial with animated UV (scrolling water texture)
    vertex: slight vertex displacement for wave effect
    fragment: alpha blend, light tint, slight transparency
  Sky shader: custom fullscreen quad
    vertex: simple position pass-through
    fragment: gradient from zenith to horizon, sun/moon rendering, stars
  Entity shader: MeshBasicMaterial or custom for mob rendering
    vertex: apply bone animation transforms
    fragment: sample entity atlas, apply lighting (ambient only)
```

### 15.6 Rendering Pipeline (Detailed)
```
Each frame rendering pipeline:
  1. Camera update: compute view matrix from player position/rotation
     Apply smooth interpolation (interpolationFactor for between-tick rendering)
     Update frustum planes from projection × view matrix
  2. Server (game state): run game tick (if accumulator ≥ 50ms)
     All game logic (see 0.5 Tick System)
  3. Chunk management:
     Check for newly loaded chunks (worker results)
     Upload any pending mesh buffers to GPU
     Process chunk unload queue (remove from scene, free memory)
  4. Visibility determination:
     For each loaded chunk: frustum test (6 planes)
     For visible chunks: occlusion test (if enabled)
     For each entity: frustum test + distance check
     Sort opaque chunks: front to back (by distance from camera)
     Sort transparent geometry: back to front
     Sort entities: by distance (front to back for opaque, back to front for blended)
  5. Depth pre-pass (optional):
     Render all opaque geometry with color write disabled
     Write-only depth buffer (early-z optimization)
     Skip if GPU does not benefit from early-z (test on target hardware)
  6. Opaque pass:
     Set depth write ON, depth test LEQUAL
     Bind block texture atlas
     For each opaque chunk (front to back):
       Bind VAO for chunk → draw elements
     Water pass: separate after opaque
       Set alpha blend ON, depth write OFF
       Render water faces (transparent, sorted back to front)
  7. Entity pass:
     For each entity type:
       Bind entity texture atlas
       Bind instanced mesh VAO
       Update instance data (positions, rotations)
       Draw elements instanced
  8. Particle pass:
     Bind particle point sprite texture
     Bind particle VAO
     Draw point sprites (count = active particles)
  9. Sky + fog pass:
     Render sky dome (fullscreen quad or sphere)
     Apply fog overlay (fullscreen quad, color based on weather/time)
  10. UI pass:
      Clear depth buffer (or disable depth test)
      Render HUD elements (crosshair, hotbar, health, hunger, XP, air)
      Render chat (if visible)
      Render debug overlay (F3 menu)
      Render tooltips, toast notifications
  11. Post-processing (optional):
      Vignette (at low health)
      Portal tint (when in nether portal)
      Night vision effect (green tint overlay)
```

### 15.7 Rendering Statistics (for F3 debug)
```
Displayed in F3 menu:
  Graphics info:
    - Renderer: Three.js / WebGL2 (version string)
    - GPU: renderer.info.renderer (name, vendor)
    - Screen: resolution + scale factor
    - VSync: enabled/disabled
  Performance:
    - FPS: current instantaneous FPS + 1s rolling average
    - Frame time: ms per frame (last frame + average)
    - Render distance: current setting
    - Loaded chunks: [loaded count] / [rendered count] / [total in range]
    - Mesh vertex count: total vertices across all chunk meshes
    - Draw calls: total draw calls per frame (opaque + transparent + entities + particles + UI)
    - Triangles: total triangles per frame
  Memory:
    - GPU memory: estimated texture + buffer memory (MB)
    - JS heap: current heap size (MB) + heap limit
    - Entity count: [total entities] / [max]
    - Particle count: [active particles] / [max 5000]
    - Chunk mesh memory: estimated GPU mesh memory (MB)
  Timing:
    - Tick time: game tick processing time (ms)
    - Render time: frame rendering time (ms) — total
    - Chunk gen time: time spent on chunk generation this frame (ms)
    - GPU time: estimated GPU time via EXT_disjoint_timer_query (if available)
  World info:
    - Chunk gen queue: pending / active generations
    - Chunk upload queue: pending GPU uploads
    - Workers: active / idle workers
  Misc:
    - Biome at player position
    - Light level at player position (sky + block)
    - Local difficulty: clamped_difficulty value
    - Moon phase: current phase number

F3 layout:
  Left side: text block with all statistics
  Toggle: F3 shows/hides overlay
  Sub-toggles:
    F3+B: show hitboxes (entity AABB + look vector)
    F3+D: clear chat (not debug related)
    F3+G: show chunk boundaries (wireframe overlay)
    F3+N: cycle creative ↔ spectator
    F3+Q: show help (list F3 commands)
```

---
