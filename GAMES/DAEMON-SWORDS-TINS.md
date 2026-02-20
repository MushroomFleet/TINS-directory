# DAEMON-SWORDS

A top-down 2D action roguelike where you ARE the cursed demonic blade. Possess enemies, master weapon transformations, and carve through procedurally generated dungeons using a deep combat system built on spline-based attack patterns and elemental affinities.

---

## Description

DAEMON-SWORDS evolves the "playing as an object" concept into a full combat experience. The player is a sentient demonic sword—a cursed blade with a single eye embedded in its hilt. When dropped, the blade attracts nearby enemies with its hypnotic glow. Once picked up, the player gains full control of the host, wielding themselves as the weapon.

The blade can be **reforged** at stations throughout the dungeon, spending accumulated XP to transform into different weapon types—each with unique attack patterns, range profiles, and special abilities. Combat uses a **spline-based attack system** where damage zones are defined by curves emanating from the wielder, creating diverse slash patterns from focused stabs to wide sweeping arcs.

Killing enemies imbues the blade with **elemental affinity**, visually represented by colored glows. Affinities create rock-paper-scissors dynamics against bosses and unlock affinity-gated areas, adding puzzle and strategic layers to the possession loop.

**Core Loop:**
1. Blade lies dormant → Eye opens → Attracts enemies
2. Enemy picks up blade → Player possesses host
3. Fight using weapon's attack patterns and special abilities
4. Host dies → Absorb XP → Drop to ground → Repeat
5. Find forge → Spend XP → Transform weapon type
6. Gain affinity from kills → Use strategically against bosses and puzzles

---

## Functionality

### 1. The Daemon Blade (Player Entity)

The blade is the player's persistent identity. It cannot move independently but persists through all host deaths, accumulating power.

**Properties:**
```javascript
DaemonBlade = {
  id: "daemon-blade-player",
  position: { x: number, y: number },
  
  // Progression
  totalXP: number,
  bladeLevel: number,                    // 1-3, affects available attacks
  
  // Weapon Form
  weaponType: "SWORD" | "DAGGERS" | "TRIDENT" | "HALBERD" | "CHAIN" | "LIGHTNING_ROD",
  
  // Affinity System
  currentAffinity: string | null,        // e.g., "GOBLIN", "SKELETON", "DEMON"
  affinityStrength: number,              // 0-100, decays over time or resets on forge
  affinityColor: { r, g, b },            // Visual glow color
  
  // Light & Attraction
  eyeState: "OPEN" | "CLOSED" | "GLINTING",
  baseLightRadius: 100,                  // px, when eye closed
  eyeLightBonus: 60,                     // Additional light when eye open
  attractionRadius: 200,                 // Scales with blade level
  
  // State
  state: "GROUNDED" | "HELD" | "FALLING",
  currentHost: string | null,
  isRejecting: boolean                   // True when spacebar held to reject pickup
}
```

**Eye Mechanics:**
- `OPEN`: Default grounded state. Full light radius (`baseLightRadius + eyeLightBonus`). Actively attracts nearest enemy.
- `GLINTING`: Pulsing attraction state. Eye visually pulses, attraction strength increased.
- `CLOSED`: Player holds spacebar. Light reduced to `baseLightRadius` only. Rejects the nearest enemy attempting pickup—they are pushed back and the next-nearest enemy becomes the attraction target.

**Visual Design:**
- Blade sprite changes based on `weaponType`
- Single eye embedded in hilt/handle area
- Eye animates: blink idle, wide-open attract, squint when closed
- Colored glow aura based on `affinityColor` intensity scaled by `affinityStrength`

### 2. Weapon Types & Reforging

Each weapon type has distinct characteristics affecting attack patterns, damage distribution, and playstyle.

**Weapon Type Definitions:**
```javascript
WeaponTypes = {
  SWORD: {
    name: "Daemon Sword",
    baseRadius: 80,           // Attack reach
    attackSpeed: 1.0,         // Multiplier
    damagePerHit: 25,
    damageOverTime: false,
    pattern: "ARC",           // Single sweeping arc
    description: "Balanced. Focused damage in medium arc."
  },
  
  DAGGERS: {
    name: "Daemon Daggers",
    baseRadius: 40,           // Short range
    attackSpeed: 2.5,         // Very fast
    damagePerHit: 8,
    damageOverTime: true,     // Rapid hits accumulate
    pattern: "FLURRY",        // Rapid small arcs
    description: "Blazing fast, short range. Death by a thousand cuts."
  },
  
  TRIDENT: {
    name: "Daemon Trident",
    baseRadius: 100,
    attackSpeed: 0.8,
    damagePerHit: 20,
    damageOverTime: false,
    pattern: "TRIPLE_LINE",   // Three parallel stabs forward
    description: "Wide parallel thrusts. Covers horizontal space."
  },
  
  HALBERD: {
    name: "Daemon Halberd",
    baseRadius: 150,          // Longest reach
    attackSpeed: 0.4,         // Very slow
    damagePerHit: 60,         // Highest alpha damage
    damageOverTime: false,
    pattern: "FULL_SWEEP",    // 270-degree sweep
    description: "Massive reach, devastating damage, glacial speed."
  },
  
  CHAIN: {
    name: "Daemon Chain",
    baseRadius: 120,
    attackSpeed: 0.7,
    damagePerHit: 15,
    damageOverTime: false,
    pattern: "CHAIN_LINK",    // Hits chain through nearby enemies
    chainLimit: 4,            // Max enemies chained
    chainRadius: 60,          // Proximity for chain to jump
    description: "Blade on chain. Damage chains through clustered enemies."
  },
  
  LIGHTNING_ROD: {
    name: "Daemon Lightning Rod",
    baseRadius: 90,
    attackSpeed: 0.6,
    damagePerHit: 12,
    damageOverTime: false,
    pattern: "FORK_LIGHTNING", // Forking chains to multiple targets
    forkCount: 3,             // How many forks per chain
    maxTargets: 8,            // Total enemies hit
    description: "Forking lightning. Devastates groups."
  }
}
```

**Reforging System:**
- **Forge Stations** appear in each dungeon floor (minimum 1, placed in safe rooms)
- Interact with forge to open reforge menu
- Spend XP to transform weapon type
- Reforging resets `bladeLevel` to 1 but preserves `totalXP` spent
- Reforging clears current `affinity` (fresh blade)

**Reforge Costs:**
```javascript
ReforgeCosts = {
  SWORD: 0,          // Default form, always free to return
  DAGGERS: 100,
  TRIDENT: 150,
  HALBERD: 200,
  CHAIN: 250,
  LIGHTNING_ROD: 300
}
```

### 3. Blade Leveling & Attack Unlocks

Each weapon form has 3 levels, unlocking progressively more powerful attacks.

**Leveling Formula:**
```javascript
xpToNextBladeLevel = Math.floor(50 * Math.pow(2, currentBladeLevel))
// Level 1→2: 100 XP
// Level 2→3: 200 XP
```

**Attack Unlocks by Level:**

| Level | Attack Type | Description |
|-------|-------------|-------------|
| 1 | Basic Attack | Standard attack pattern for weapon type |
| 2 | Charged Attack | Hold attack button for empowered strike |
| 3 | Special Attack | Unique devastating ability per weapon |

**Special Attacks (Level 3):**
```javascript
SpecialAttacks = {
  SWORD: {
    name: "Judgement Slash",
    description: "Massive frontal arc with extended range, brief invulnerability",
    cooldown: 8000  // ms
  },
  DAGGERS: {
    name: "Phantom Flurry",
    description: "Teleport between nearby enemies, striking each",
    cooldown: 6000
  },
  TRIDENT: {
    name: "Impale",
    description: "Long-range triple piercing thrust, pins first enemy hit",
    cooldown: 10000
  },
  HALBERD: {
    name: "Cataclysm",
    description: "360-degree devastating spin, knockback on all hit",
    cooldown: 12000
  },
  CHAIN: {
    name: "Grapple Storm",
    description: "Pull all chained enemies to center, stun on collision",
    cooldown: 9000
  },
  LIGHTNING_ROD: {
    name: "Storm Call",
    description: "Massive AOE lightning strike at cursor position",
    cooldown: 10000
  }
}
```

### 4. Spline-Based Attack System

Attacks are defined by **spline paths** that emanate from the wielder. Damage is calculated by checking enemy intersection with the spline's **thickness zones**.

**Attack Definition Schema:**
```javascript
AttackPattern = {
  id: string,
  weaponType: string,
  level: number,                    // Minimum blade level to use
  
  // Spline Definition
  spline: {
    type: "BEZIER" | "LINEAR" | "ARC",
    controlPoints: [
      { x: number, y: number, time: number }  // Relative to wielder, time is animation progress 0-1
    ],
    thickness: number,              // Width of damage zone along spline
    segments: number                // Resolution for collision checks
  },
  
  // Timing
  windupMs: number,                 // Delay before damage active
  activeMs: number,                 // Duration damage zone exists
  recoveryMs: number,               // Cooldown before next attack
  
  // Damage
  baseDamage: number,
  damageType: "INSTANT" | "TICK",   // TICK = damage over time while in zone
  tickRateMs: number,               // For TICK type, how often damage applies
  
  // Modifiers
  knockback: number,                // 0 = none, 1 = strong
  affectedByAffinity: boolean       // Whether elemental bonus applies
}
```

**Example Attack Patterns:**

**Sword Basic Slash:**
```javascript
{
  id: "sword_basic",
  weaponType: "SWORD",
  level: 1,
  spline: {
    type: "ARC",
    controlPoints: [
      { x: 30, y: -40, time: 0 },    // Start right-back
      { x: 60, y: 0, time: 0.5 },    // Swing through front
      { x: 30, y: 40, time: 1 }      // End right-forward
    ],
    thickness: 30,
    segments: 12
  },
  windupMs: 100,
  activeMs: 200,
  recoveryMs: 150,
  baseDamage: 25,
  damageType: "INSTANT",
  knockback: 0.3,
  affectedByAffinity: true
}
```

**Trident Triple Thrust:**
```javascript
{
  id: "trident_basic",
  weaponType: "TRIDENT",
  level: 1,
  spline: {
    type: "LINEAR",
    // Three parallel lines
    paths: [
      { startX: 0, startY: -25, endX: 100, endY: -25 },
      { startX: 0, startY: 0, endX: 100, endY: 0 },
      { startX: 0, startY: 25, endX: 100, endY: 25 }
    ],
    thickness: 15,
    segments: 8
  },
  windupMs: 150,
  activeMs: 180,
  recoveryMs: 250,
  baseDamage: 20,
  damageType: "INSTANT",
  knockback: 0.5,
  affectedByAffinity: true
}
```

**Halberd Full Sweep:**
```javascript
{
  id: "halberd_basic",
  weaponType: "HALBERD",
  level: 1,
  spline: {
    type: "ARC",
    controlPoints: [
      { x: -100, y: -100, time: 0 },
      { x: 150, y: 0, time: 0.5 },
      { x: -100, y: 100, time: 1 }
    ],
    thickness: 40,
    segments: 20
  },
  windupMs: 400,
  activeMs: 500,
  recoveryMs: 600,
  baseDamage: 60,
  damageType: "INSTANT",
  knockback: 0.8,
  affectedByAffinity: true
}
```

**Chain Link Attack:**
```javascript
{
  id: "chain_basic",
  weaponType: "CHAIN",
  level: 1,
  spline: {
    type: "BEZIER",
    controlPoints: [
      { x: 0, y: 0, time: 0 },
      { x: 60, y: -20, time: 0.3 },
      { x: 120, y: 0, time: 1 }
    ],
    thickness: 20,
    segments: 10
  },
  windupMs: 200,
  activeMs: 300,
  recoveryMs: 350,
  baseDamage: 15,
  damageType: "INSTANT",
  chainEffect: {
    enabled: true,
    maxChains: 4,
    chainRadius: 60,
    damageDecay: 0.8     // Each chain does 80% of previous
  },
  knockback: 0.2,
  affectedByAffinity: true
}
```

### 5. Defensive Mechanics

**Auto-Block:**
While holding the blade, incoming projectiles and melee attacks within a frontal cone are automatically deflected.

```javascript
AutoBlock = {
  coneAngle: 90,            // Degrees, centered on facing direction
  blockRadius: 50,          // Distance from wielder
  blockDamageReduction: 0.8,// 80% damage blocked
  staminaCost: 5,           // Per block
  cooldownMs: 100           // Minimum time between blocks
}
```

**Perfect Parry (Timed Block):**
Pressing the block button (spacebar while held) at the exact moment of impact triggers a parry.

```javascript
PerfectParry = {
  windowMs: 150,            // Timing window for perfect parry
  reflectProjectiles: true, // Projectiles return to sender
  riposteWindow: 500,       // Ms after parry where next attack does 2x damage
  staminaRefund: true       // No stamina cost on perfect parry
}
```

**Dash:**
Quick directional dash with brief invulnerability frames.

```javascript
Dash = {
  distance: 100,            // Pixels traveled
  durationMs: 150,          // Animation time
  iFramesMs: 100,           // Invulnerability duration
  cooldownMs: 800,          // Time between dashes
  staminaCost: 15
}
```

### 6. Affinity System

Killing enemies imbues the blade with their essence, creating elemental advantages/disadvantages.

**Affinity Acquisition:**
```javascript
AffinityGain = {
  onKill: 25,               // Affinity strength gained per kill
  maxStrength: 100,
  decayRatePerSecond: 0.5,  // Passive decay when not killing
  onReforge: "RESET"        // Affinity clears when reforging
}
```

**Enemy Affinity Types:**
```javascript
AffinityTypes = {
  GOBLIN: {
    color: { r: 100, g: 200, b: 80 },   // Sickly green
    strongAgainst: ["SKELETON", "GHOST"],
    weakAgainst: ["DEMON", "BEAST"]
  },
  SKELETON: {
    color: { r: 200, g: 180, b: 220 },  // Pale purple
    strongAgainst: ["DEMON", "BEAST"],
    weakAgainst: ["GOBLIN", "ELEMENTAL"]
  },
  DEMON: {
    color: { r: 220, g: 50, b: 30 },    // Crimson red
    strongAgainst: ["GOBLIN", "ELEMENTAL"],
    weakAgainst: ["SKELETON", "GHOST"]
  },
  BEAST: {
    color: { r: 180, g: 130, b: 80 },   // Earthy brown
    strongAgainst: ["GHOST", "ELEMENTAL"],
    weakAgainst: ["SKELETON", "DEMON"]
  },
  GHOST: {
    color: { r: 150, g: 220, b: 255 },  // Spectral blue
    strongAgainst: ["DEMON", "BEAST"],
    weakAgainst: ["GOBLIN", "SKELETON"]
  },
  ELEMENTAL: {
    color: { r: 255, g: 200, b: 50 },   // Bright gold
    strongAgainst: ["GOBLIN", "GHOST"],
    weakAgainst: ["BEAST", "DEMON"]
  }
}
```

**Affinity Combat Modifiers:**
```javascript
AffinityDamageModifiers = {
  strongAgainst: 1.5,       // 50% bonus damage
  neutral: 1.0,
  weakAgainst: 0.7          // 30% damage reduction
}
```

**Visual Representation:**
- Blade emits colored particle aura matching `affinityColor`
- Aura intensity scales with `affinityStrength` (0-100)
- Eye iris color tints toward affinity color
- Attack splines render with affinity-colored trails

### 7. Affinity-Gated Puzzles

Dungeons contain doors and mechanisms requiring specific affinities.

**Affinity Door:**
```javascript
AffinityDoor = {
  requiredAffinity: string,       // e.g., "SKELETON"
  minimumStrength: 50,            // Minimum affinity strength to open
  visualIndicator: true,          // Door glows required color
  onInsufficientAffinity: "DISPLAY_HINT"  // Show which affinity needed
}
```

**Affinity Inverter (Puzzle Element):**
```javascript
AffinityInverter = {
  type: "PEDESTAL",
  targetAffinity: string,         // What affinity this inverter grants
  duration: 30000,                // Ms the temporary affinity lasts
  overridesExisting: true,        // Temporarily replaces current affinity
  visualEffect: "AURA_SWAP"       // Player sees color change
}
```

**Puzzle Example Flow:**
1. Player has GOBLIN affinity from recent kills
2. Encounters SKELETON-gated door
3. Finds Affinity Inverter pedestal nearby
4. Activating pedestal temporarily grants SKELETON affinity for 30 seconds
5. Player rushes to door, opens it, proceeds
6. After 30s, original GOBLIN affinity returns

### 8. Host Possession System

Adapted from R-Torch framework with blade-specific modifications.

**Host Entity Schema:**
```javascript
Host = {
  id: string,
  position: { x: number, y: number },
  velocity: { x: number, y: number },
  
  // Stats
  health: number,
  maxHealth: number,
  stamina: number,
  maxStamina: number,
  moveSpeed: number,
  
  // Affinity
  affinityType: string,           // This host's affinity type
  
  // AI State (when not possessed)
  aiState: "IDLE" | "PATROL" | "ATTRACTED" | "HOSTILE",
  targetPosition: { x, y } | null,
  
  // Possession
  isPossessed: boolean,
  holdingBlade: boolean,
  
  // Visual
  parts: PartAssembly,            // Modular character parts
  animations: AnimationSet
}
```

**Attraction Mechanics:**
- When blade is GROUNDED with eye OPEN, emits attraction pulse every 500ms
- Nearest enemy within `attractionRadius` enters ATTRACTED state
- ATTRACTED enemies path toward blade, ignoring other hostiles
- When enemy reaches blade, possession triggers automatically
- If player holds spacebar (eye CLOSED), nearest ATTRACTED enemy is rejected and pushed back 50px

**Possession Transfer:**
```javascript
PossessionFlow = {
  onHostDeath: [
    "Calculate XP from host kills",
    "Add XP to blade totalXP",
    "Check blade level up",
    "Inherit affinity from host (if different, blend or override based on strength)",
    "Set blade state to FALLING",
    "Play fall animation (300ms)",
    "Set blade state to GROUNDED",
    "Open eye, begin attraction"
  ],
  
  onPickup: [
    "Set blade state to HELD",
    "Set host isPossessed = true",
    "Attach blade to host R_HAND bone",
    "Transfer player control to host",
    "Close eye (no attraction while held)"
  ]
}
```

### 9. Modular Character System

All humanoid entities use unified skeletal/parts assembly for visual variety.

**Bone Structure:**
```
                    [HEAD]
                      │
            ┌────────[TORSO]────────┐
            │         │             │
        [L_ARM]    [SPINE]      [R_ARM]
            │         │             │
        [L_HAND]   [HIPS]       [R_HAND] ← Blade attaches here
                      │          
                ┌─────┴─────┐
            [L_LEG]     [R_LEG]
                │           │
            [L_FOOT]    [R_FOOT]
```

**Part Definition:**
```javascript
Part = {
  id: string,                     // e.g., "head_skeleton_01"
  type: "HEAD" | "TORSO" | "L_ARM" | "R_ARM" | "L_LEG" | "R_LEG",
  affinityType: string,           // Which affinity pool this part belongs to
  spriteSheet: string,
  animations: {
    idle: { frames: [number], fps: number },
    walk: { frames: [number], fps: number },
    attack: { frames: [number], fps: number },
    death: { frames: [number], fps: number },
    hit: { frames: [number], fps: number }
  },
  attachOffset: { x: number, y: number },
  zIndex: number,
  stats: {
    healthMod?: number,
    speedMod?: number,
    staminaMod?: number
  }
}
```

**NPC Generation:**
```javascript
generateNPC = (affinityType, difficulty) => {
  const pool = PartPools[affinityType];
  return {
    head: randomFrom(pool.heads),
    torso: randomFrom(pool.torsos),
    leftArm: randomFrom(pool.arms),
    rightArm: randomFrom(pool.arms),
    leftLeg: randomFrom(pool.legs),
    rightLeg: randomFrom(pool.legs),
    affinityType: affinityType,
    // Stats calculated from part modifiers + difficulty scaling
  };
}
```

### 10. Procedural Dungeon Generation

Dungeons are generated using seed-based room placement with guaranteed forge room and progression path.

**Dungeon Schema:**
```javascript
Dungeon = {
  seed: number,
  width: number,                  // In tiles
  height: number,
  tileSize: 32,                   // Pixels
  
  rooms: [Room],
  corridors: [Corridor],
  
  spawnPoints: [SpawnPoint],
  forgeLocations: [{ x, y }],     // Minimum 1 per floor
  affinityDoors: [AffinityDoor],
  inverters: [AffinityInverter],
  
  startRoom: string,              // Room ID where player blade spawns
  exitRoom: string                // Room ID with stairs to next floor
}
```

**Room Types:**
```javascript
RoomTypes = {
  SPAWN: { minSize: 8, maxSize: 12, features: ["blade_pedestal"] },
  COMBAT: { minSize: 10, maxSize: 20, features: ["enemy_spawns"] },
  FORGE: { minSize: 6, maxSize: 8, features: ["forge_station"], isSafeRoom: true },
  PUZZLE: { minSize: 8, maxSize: 15, features: ["affinity_door", "inverter"] },
  BOSS: { minSize: 20, maxSize: 25, features: ["boss_spawn", "arena_hazards"] },
  TREASURE: { minSize: 5, maxSize: 7, features: ["xp_cache"] },
  EXIT: { minSize: 8, maxSize: 10, features: ["stairs_down"] }
}
```

**Generation Algorithm:**
1. Place SPAWN room at random position
2. Generate room graph using BSP (Binary Space Partitioning)
3. Ensure one FORGE room exists
4. Place EXIT room furthest from SPAWN
5. Scatter COMBAT, PUZZLE, TREASURE rooms
6. Connect rooms with corridors
7. Place enemy spawn points in COMBAT rooms (mixed affinity types)
8. Place affinity doors requiring affinities available earlier in dungeon
9. Validate path exists from SPAWN to EXIT

### 11. Fog of War & Lighting

The dungeon exists in darkness. Only the blade's light reveals the world.

**Lighting Model:**
```javascript
Lighting = {
  ambientLight: 0.05,             // 5% base visibility (outlines only)
  
  bladeLightSource: {
    baseRadius: 100,              // When eye closed
    eyeOpenBonus: 60,             // Additional when eye open
    affinityGlowBonus: 20,        // At max affinity strength
    color: "dynamic",             // Tinted by affinity
    falloff: "QUADRATIC"          // Realistic light falloff
  },
  
  fogOfWar: {
    exploredAlpha: 0.3,           // Seen areas remain slightly visible
    unexploredAlpha: 0,           // Pure black
    updateOnMove: true
  }
}
```

**Visual Rendering:**
- Full darkness shader over dungeon
- Blade light creates circular reveal with soft edges
- Affinity color tints the light subtly
- Enemy eyes/features glow faintly at edge of light (danger indication)
- Forge stations emit faint welcoming glow (wayfinding)

### 12. XP & Progression

XP is the universal currency for leveling and reforging.

**XP Sources:**
```javascript
XPRewards = {
  enemyKill: {
    base: 10,
    perDifficulty: 5,             // +5 per difficulty tier
    affinityBonus: 0.2            // +20% if strong affinity
  },
  bossKill: {
    base: 200,
    perFloor: 50                  // Scales with dungeon depth
  },
  exploration: {
    newRoom: 5,
    treasureCache: 25
  }
}
```

**Spending XP:**
- Blade level-up: Automatic when threshold reached
- Reforging: Manual spend at forge station
- Note: Reforging does not reduce total XP (tracked separately), only spends "available XP pool"

```javascript
Progression = {
  availableXP: number,            // Spendable pool
  totalXPEarned: number,          // Lifetime total (for achievements/unlocks)
  bladeLevel: number,             // Current level 1-3
  xpToNextLevel: function         // 50 * 2^level formula
}
```

### 13. Boss Encounters

Each dungeon floor culminates in a boss fight. Bosses have specific affinity weaknesses.

**Boss Schema:**
```javascript
Boss = {
  id: string,
  name: string,
  affinityType: string,           // Boss's own affinity
  weakAgainst: [string],          // Affinities that deal bonus damage
  
  health: number,
  phases: [BossPhase],            // Multi-phase fights
  
  attacks: [BossAttack],          // Unique attack patterns
  arena: {                        // Arena modifications
    hazards: [Hazard],
    spawnAdds: boolean,
    addAffinityTypes: [string]    // What affinity adds spawn (for player to harvest)
  }
}
```

**Boss Affinity Strategy:**
- Boss arena spawns adds of specific affinity types
- Player can strategically let host die, get picked up by specific add
- Kill adds of the type strong against boss
- Build affinity, then engage boss with damage bonus

**Example Boss:**
```javascript
{
  id: "boss_lich_king",
  name: "The Lich King",
  affinityType: "SKELETON",
  weakAgainst: ["GOBLIN", "ELEMENTAL"],  // Takes extra damage from these
  health: 1000,
  phases: [
    { healthThreshold: 1.0, attacks: ["bone_wave", "summon_skeletons"] },
    { healthThreshold: 0.5, attacks: ["bone_wave", "death_beam", "mass_summon"] }
  ],
  arena: {
    spawnAdds: true,
    addAffinityTypes: ["SKELETON", "GOBLIN"]  // Goblins spawn so player can get correct affinity
  }
}
```

---

## Technical Implementation

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                        Game Loop                            │
│  ┌─────────┐  ┌──────────┐  ┌────────┐  ┌───────────────┐  │
│  │  Input  │→ │  Update  │→ │ Render │→ │ Present Frame │  │
│  └─────────┘  └──────────┘  └────────┘  └───────────────┘  │
└─────────────────────────────────────────────────────────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
┌──────────────┐ ┌─────────────┐ ┌─────────────────┐
│ BladeSystem  │ │ CombatSystem│ │ DungeonManager  │
│ - Possession │ │ - Attacks   │ │ - Generation    │
│ - Eye State  │ │ - Splines   │ │ - Fog of War    │
│ - Affinity   │ │ - Blocking  │ │ - Room State    │
│ - Attraction │ │ - Damage    │ │ - Spawn Control │
└──────────────┘ └─────────────┘ └─────────────────┘
        │               │               │
        └───────────────┼───────────────┘
                        ▼
              ┌──────────────────┐
              │   EntityManager  │
              │   - NPCs         │
              │   - Projectiles  │
              │   - Pickups      │
              └──────────────────┘
```

### Component Structure (React)

```
<GameContainer>
  <DungeonRenderer>
    <TileMap />
    <FogOfWarOverlay />
    <EntityLayer>
      <NPCRenderer />
      <BladeRenderer />
      <ProjectileRenderer />
    </EntityLayer>
    <LightingLayer />
    <AttackSplineRenderer />
  </DungeonRenderer>
  <UIOverlay>
    <HealthStaminaBar />
    <AffinityIndicator />
    <WeaponDisplay />
    <XPCounter />
  </UIOverlay>
  <ForgeModal />
  <PauseMenu />
</GameContainer>
```

### Key Algorithms

**Spline Damage Calculation:**
```javascript
function calculateSplineDamage(spline, enemies, attackDef) {
  const hitEnemies = [];
  
  // Generate points along spline
  const splinePoints = generateSplinePoints(spline, attackDef.segments);
  
  for (const enemy of enemies) {
    // Check distance to any spline segment
    for (let i = 0; i < splinePoints.length - 1; i++) {
      const dist = pointToSegmentDistance(
        enemy.position,
        splinePoints[i],
        splinePoints[i + 1]
      );
      
      if (dist <= spline.thickness / 2) {
        hitEnemies.push({
          enemy,
          damage: calculateDamage(attackDef, enemy),
          knockbackDir: calculateKnockback(splinePoints[i], enemy.position)
        });
        break; // Only hit once per attack
      }
    }
  }
  
  // Handle chain effects
  if (attackDef.chainEffect?.enabled) {
    hitEnemies.push(...propagateChain(hitEnemies, enemies, attackDef.chainEffect));
  }
  
  return hitEnemies;
}
```

**Affinity Damage Modifier:**
```javascript
function calculateDamage(attackDef, enemy) {
  let damage = attackDef.baseDamage;
  
  if (attackDef.affectedByAffinity && blade.currentAffinity) {
    const affinityData = AffinityTypes[blade.currentAffinity];
    
    if (affinityData.strongAgainst.includes(enemy.affinityType)) {
      damage *= AffinityDamageModifiers.strongAgainst;
    } else if (affinityData.weakAgainst.includes(enemy.affinityType)) {
      damage *= AffinityDamageModifiers.weakAgainst;
    }
    
    // Scale by affinity strength (50-100% of bonus)
    const strengthMod = 0.5 + (blade.affinityStrength / 200);
    damage *= strengthMod;
  }
  
  return Math.floor(damage);
}
```

**Eye Rejection Mechanic:**
```javascript
function updateAttraction() {
  if (blade.state !== "GROUNDED") return;
  
  const nearbyEnemies = getEnemiesInRadius(blade.position, blade.attractionRadius);
  
  if (nearbyEnemies.length === 0) return;
  
  // Sort by distance
  nearbyEnemies.sort((a, b) => 
    distance(a.position, blade.position) - distance(b.position, blade.position)
  );
  
  if (blade.isRejecting) {
    // Push away nearest enemy
    const rejected = nearbyEnemies[0];
    const pushDir = normalize(subtract(rejected.position, blade.position));
    rejected.position = add(rejected.position, scale(pushDir, 50));
    rejected.aiState = "HOSTILE"; // They're now angry
    
    // Attract second nearest instead
    if (nearbyEnemies.length > 1) {
      nearbyEnemies[1].aiState = "ATTRACTED";
    }
  } else {
    // Normal attraction
    nearbyEnemies[0].aiState = "ATTRACTED";
  }
}
```

### Data Persistence

```javascript
SaveData = {
  // Run state (lost on death/quit)
  currentRun: {
    seed: number,
    floor: number,
    bladeState: DaemonBlade,
    dungeonState: Dungeon,
    exploredRooms: [string]
  },
  
  // Persistent unlocks
  meta: {
    totalXPEarned: number,
    highestFloor: number,
    bossesDefeated: [string],
    weaponsUnlocked: [string],     // Which reforge options available
    achievementsUnlocked: [string]
  }
}
```

### Controls

**Keyboard:**
| Key | Action |
|-----|--------|
| WASD | Move host |
| Mouse | Aim direction |
| Left Click | Attack |
| Hold Left Click | Charged attack (level 2+) |
| Right Click | Special attack (level 3) |
| Space (while grounded) | Hold to close eye / reject pickup |
| Space (while held) | Dash in movement direction |
| E | Interact (forge, doors) |
| Tab | Parry / Timed block |
| Esc | Pause menu |

**Gamepad:**
| Button | Action |
|--------|--------|
| Left Stick | Move |
| Right Stick | Aim |
| RT | Attack |
| LT | Parry |
| A | Dash |
| X | Interact |
| Y | Special Attack |
| LB (hold) | Reject pickup (when grounded) |

---

## Style Guide

### Visual Aesthetic
- **Dark fantasy** with demonic undertones
- High contrast between darkness and blade light
- Enemies have glowing eyes/features visible at light edge
- Affinity colors are saturated and distinct
- Gore is stylized (particle effects, not graphic)

### Color Palette
```
Background/Darkness: #0a0a0f
Dungeon Stone: #2a2a35, #1f1f28
Blade Metal: #8090a0 (neutral), tinted by affinity
Eye (open): #ffdd44 with #ff6644 pupil
Eye (closed): #443322

Affinity Colors:
- Goblin: #64c850
- Skeleton: #c8b4dc
- Demon: #dc3220
- Beast: #b48250
- Ghost: #96dcff
- Elemental: #ffc832
```

### Animation Timing
- Eye blink: 150ms close, 100ms open
- Glint pulse: 800ms cycle
- Attack windups: Weapon-specific (see attack definitions)
- Dash: 150ms travel
- Host death: 400ms collapse, 300ms blade fall
- Pickup: 200ms reach, 100ms grasp

### Audio Cues (Placeholder Descriptions)
- Eye open: Low ethereal hum
- Attraction pulse: Subtle heartbeat
- Pickup: Metallic ring + possession whoosh
- Attack by weapon type: Distinct slash/thrust/chain sounds
- Affinity gain: Crystalline absorption tone
- Forge: Anvil strikes + flame roar
- Perfect parry: Sharp metallic clash + time-slow effect

---

## Testing Scenarios

### Core Mechanics
1. **Eye Rejection:** Drop blade, let enemies approach, hold space to reject first, verify second enemy picks up instead
2. **Affinity Acquisition:** Kill 4 goblins, verify green glow appears at ~100 strength
3. **Affinity Combat:** With goblin affinity, fight skeleton (should deal 1.5x), fight demon (should deal 0.7x)
4. **Reforge:** Accumulate 150 XP, use forge, transform to trident, verify attack pattern changes

### Combat System
5. **Spline Damage:** Use sword, verify enemies hit by arc take damage, enemies outside arc don't
6. **Chain Attack:** Use chain weapon against cluster of 5 enemies, verify damage chains to 4 max
7. **Lightning Fork:** Use lightning rod against group of 10, verify forking behavior caps at 8 hits
8. **Perfect Parry:** Time block at projectile impact, verify reflection
9. **Charged Attack:** Hold attack for 500ms, verify charged version triggers

### Progression
10. **Level Up:** Earn 100 XP, verify blade reaches level 2, verify charged attack unlocks
11. **Reforge Reset:** Reforge from level 3 sword to daggers, verify level resets to 1
12. **Affinity Door:** With goblin affinity at 60 strength, approach skeleton door, verify blocked
13. **Inverter:** Use skeleton inverter, verify temporary affinity swap, verify reversion after timeout

### Edge Cases
14. **Host Dies During Rejection:** While holding space to reject, current host dies—verify blade drops correctly, rejection state clears
15. **Zero Enemies:** Blade grounded with no enemies in attraction range—verify stable state, no errors
16. **Chain on Single Enemy:** Use chain attack on lone enemy—verify no chain, just direct hit
17. **Forge With Insufficient XP:** Try to reforge without enough XP—verify blocked with clear feedback

---

## Accessibility

### Visual
- High contrast mode: Increases blade light radius by 50%, adds enemy outlines
- Colorblind modes: Adjusts affinity colors with pattern overlays (stripes, dots, etc.)
- Screen reader: Announces room type, nearby enemies, affinity status
- Reduced motion: Disables screen shake, simplifies particle effects

### Controls
- Full key rebinding
- Mouse-only mode: Click-to-move, click-to-attack
- Gamepad support with customizable layout
- One-handed mode: Condenses controls to single side of keyboard

### Difficulty Modifiers
- Enemy damage scaling (50%-150%)
- Auto-parry assist mode
- Longer parry windows
- Slower enemy attack windups

---

## Performance Goals

| Metric | Target |
|--------|--------|
| Frame Rate | 60 FPS stable |
| Initial Load | < 4 seconds |
| Dungeon Generation | < 800ms for 150x150 |
| Max Active Entities | 80 without frame drops |
| Spline Calculations | < 2ms per attack |
| Memory Usage | < 250MB |
| Bundle Size | < 8MB (excluding audio assets) |

---

## Extended Features (Post-MVP)

1. **Blade Memories:** Hosts you possess retain fragments of past hosts' kills/actions, affecting their dialogue or behavior if possessed again

2. **Dual Wielding Bosses:** Late-game bosses wield two daemon blades—defeat them to unlock dual-form reforges

3. **Affinity Fusion:** At max strength, killing an enemy of different affinity creates hybrid affinity with combined strengths/weaknesses

4. **Weapon Skill Trees:** Each weapon type has branching upgrade paths beyond level 3

5. **Endless Abyss Mode:** Infinite descending dungeon with leaderboards

6. **Blade Corruption:** Optional mechanic where heavy kill counts unlock darker attacks but reduce light radius

7. **Multiplayer - Blade Rivalry:** Two players as competing daemon blades in shared dungeon

8. **Sound Design:** Positional audio, dynamic music reacting to combat intensity and affinity

9. **Achievement System:** Track total possessions, affinity mastery, weapon proficiency, speedrun times

---

*This document serves as the complete specification for DAEMON-SWORDS. An implementation generated from this TINS README should produce a fully playable action roguelike demonstrating the daemon blade possession mechanic, spline-based combat system, weapon reforging, elemental affinities, and procedural dungeon generation.*
