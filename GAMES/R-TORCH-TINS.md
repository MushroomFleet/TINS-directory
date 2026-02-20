# R-Torch

## Description

R-Torch is a top-down 2D sprite-based dungeon crawler where the player IS a torch—not a character holding a torch, but the torch itself. The torch is the sole light source in a world of absolute darkness, creating an intimate fog-of-war that reveals only the immediate surroundings.

The torch cannot move on its own. Instead, it exerts an irresistible attraction on nearby NPCs. When an NPC picks up the torch, the player assumes control of that creature, navigating the procedurally-generated dungeon, engaging in combat, and accumulating experience. When the possessed NPC dies, the XP transfers to the torch, which drops to the ground and waits for the next host.

This is a proof-of-concept showcasing gameplay without a player body—using possession as the core navigation and progression mechanic.

## Core Concept Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                        GAME LOOP                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   [TORCH ON GROUND] ──attracts──> [NEAREST NPC]                │
│          ▲                              │                       │
│          │                              ▼                       │
│          │                     [NPC PICKS UP TORCH]            │
│          │                              │                       │
│          │                              ▼                       │
│     [XP ABSORBED]              [PLAYER CONTROLS NPC]           │
│     [NPC DIES]                          │                       │
│          ▲                              ▼                       │
│          │                     [COMBAT / EXPLORE]              │
│          │                              │                       │
│          └──────────────────────────────┘                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Functionality

### 1. The Torch (Player Entity)

The torch is the player's persistent identity across all host deaths.

**Properties:**
```javascript
Torch = {
  id: "torch-player",
  position: { x: number, y: number },      // World position
  totalXP: number,                          // Accumulated across all hosts
  level: number,                            // Torch level (affects attraction radius)
  lightRadius: number,                      // Base: 120px, scales with level
  attractionRadius: number,                 // Base: 200px, scales with level
  isHeld: boolean,                          // True when possessed by NPC
  currentHost: string | null,               // ID of possessing NPC
  state: "GROUNDED" | "HELD" | "FALLING"
}
```

**Behaviors:**
- When `state === "GROUNDED"`: Emits attraction pulse every 500ms affecting NPCs within `attractionRadius`
- When `state === "HELD"`: Position locked to host's hand bone, player controls host
- When `state === "FALLING"`: Brief animation (300ms) transitioning from host death to grounded
- Light radius increases by 10px per torch level (max 300px)
- Attraction radius increases by 25px per torch level (max 500px)

**Leveling Formula:**
```javascript
xpToNextLevel = Math.floor(100 * Math.pow(1.5, currentLevel))
```

### 2. Modular Character System (PuppetJSX Integration)

All humanoid entities use **PuppetJSX**, a React-based skeletal animation system. This provides bone hierarchy, part assignment, animation interpolation, and runtime rendering out of the box.

**PuppetJSX Skeleton Structure:**
```
root (origin point)
└── torso
    ├── head
    ├── arm_left
    │   └── hand_left      ← shield/torch attaches here
    ├── arm_right
    │   └── hand_right     ← weapon attaches here
    ├── leg_left
    │   └── foot_left
    └── leg_right
        └── foot_right
```

**Bone Definition (from PuppetJSX):**
```javascript
// PuppetJSX skeleton bones have this structure:
Bone = {
  id: string,                    // "torso", "arm_left", etc.
  parent: string | null,         // Parent bone ID
  children: string[],            // Child bone IDs
  localTransform: {
    position: { x: number, y: number },
    rotation: number,            // Degrees
    scale: { x: number, y: number }
  },
  zIndex: number                 // Render order
}
```

**Part Definition Schema (Extended for R-Torch):**
```javascript
// Extends PuppetJSX part format with game stats
Part = {
  // PuppetJSX standard fields
  id: string,                    // e.g., "head_goblin_01"
  name: string,                  // "Goblin Head"
  category: "head" | "torso" | "arm" | "hand" | "leg" | "foot" | "weapon" | "accessory",
  width: number,                 // Sprite width in pixels
  height: number,                // Sprite height in pixels
  color: string,                 // Fallback color for placeholder rendering
  offset: { x: number, y: number },
  zIndexModifier: number,
  
  // R-Torch extensions
  imageUrl?: string,             // Path to actual sprite (optional, falls back to color)
  stats: {                       // Stat modifiers applied when part is equipped
    health?: number,             // Added to entity maxHealth
    damage?: number,             // Added to entity damage
    speed?: number,              // Multiplier (1.0 = normal)
    armor?: number               // Damage reduction
  },
  faction?: "NPC" | "MONSTER" | "ANY",  // Which pool this part belongs to
  rarity?: number                // 0.0-1.0, affects spawn weighting
}
```

**Part Pools (Organized by PuppetJSX Categories):**
```javascript
PartPools = {
  head: {
    npc: ["head_human_01", "head_villager_01", "head_merchant_01"],
    monster: ["head_goblin_01", "head_skeleton_01", "head_orc_01", "head_demon_01"],
    any: ["head_basic"]
  },
  torso: {
    npc: ["torso_tunic_01", "torso_robe_01", "torso_basic"],
    monster: ["torso_goblin_01", "torso_skeleton_01", "torso_armor"],
    any: ["torso_basic"]
  },
  arm: {
    npc: ["arm_basic", "arm_sleeve_01"],
    monster: ["arm_goblin_01", "arm_skeleton_01", "arm_armored"],
    any: ["arm_basic"]
  },
  hand: {
    npc: ["hand_basic", "hand_glove"],
    monster: ["hand_claw_01", "hand_bone_01"],
    any: ["hand_basic"]
  },
  leg: {
    npc: ["leg_basic", "leg_pants_01"],
    monster: ["leg_goblin_01", "leg_skeleton_01", "leg_armored"],
    any: ["leg_basic"]
  },
  foot: {
    npc: ["foot_basic", "foot_boot"],
    monster: ["foot_claw_01", "foot_bone_01"],
    any: ["foot_basic"]
  },
  weapon: {
    all: ["weapon_sword", "weapon_staff", "club_wooden", "dagger_bone", "axe_iron"]
  },
  accessory: {
    all: ["shield", "torch_held"]  // torch_held is special - indicates entity is carrying player
  }
}
```

**PuppetJSX Character Format (Used for Entity Assembly):**
```javascript
// This is the native PuppetJSX character format we'll use
PuppetCharacterData = {
  id: string,                    // "char_goblin_01"
  name: string,                  // "Goblin Warrior"
  skeletonId: "humanoid_skeleton",
  parts: {
    head: string,                // Part ID -> "head_goblin_01"
    torso: string,
    arm_left: string,
    arm_right: string,
    hand_left: string,           // Can be shield/accessory
    hand_right: string,          // Can be weapon
    leg_left: string,
    leg_right: string,
    foot_left: string,
    foot_right: string
  }
}
```

### 3. NPC / Entity System (PuppetJSX Runtime)

Entities are game objects that wrap PuppetJSX characters with game logic.

**Entity Schema:**
```javascript
Entity = {
  id: string,                              // Unique instance ID
  isNPC: boolean,                          // True = can be possessed, False = monster
  position: { x: number, y: number },
  velocity: { x: number, y: number },
  facing: "LEFT" | "RIGHT",                // PuppetJSX uses scale.x flip for facing
  state: "IDLE" | "WANDERING" | "ATTRACTED" | "POSSESSED" | "COMBAT" | "DEAD",
  
  // PuppetJSX Character Data (the visual representation)
  puppetData: PuppetCharacterData,         // Native PuppetJSX character format
  puppetRuntime: PuppetCharacter | null,   // Runtime instance for animation
  
  // Current animation state
  currentAnimation: string,                // "idle", "walk", "attack", "death"
  animationTime: number,                   // Current playback time
  
  // Computed stats (aggregated from all equipped parts)
  stats: {
    maxHealth: number,
    currentHealth: number,
    damage: number,
    speed: number,          // Pixels per second
    armor: number,
    attackSpeed: number     // Attacks per second
  },
  
  // AI state (when not possessed)
  ai: {
    targetPosition: { x: number, y: number } | null,
    aggroTarget: string | null,
    lastStateChange: number,
    behaviorType: "PASSIVE" | "AGGRESSIVE" | "TERRITORIAL"
  },
  
  // XP accumulated while possessed
  sessionXP: number,
  
  // Torch attachment (only when carrying torch)
  hasTorch: boolean
}
```

**Assembly ID System (Serialized PuppetJSX Character):**

For spawning and network transmission, we serialize the PuppetJSX character to a compact string:

```javascript
// Format mirrors PuppetJSX parts object, pipe-delimited
// "head|torso|arm_left|arm_right|hand_left|hand_right|leg_left|leg_right|foot_left|foot_right"
// Example: "head_goblin_01|torso_armor|arm_basic|arm_basic|hand_glove|weapon_sword|leg_armored|leg_armored|foot_boot|foot_boot"

function assemblyIdToCharacter(assemblyId, entityId) {
  const partIds = assemblyId.split("|");
  return {
    id: entityId,
    name: `Entity_${entityId}`,
    skeletonId: "humanoid_skeleton",
    parts: {
      head: partIds[0] || null,
      torso: partIds[1] || null,
      arm_left: partIds[2] || null,
      arm_right: partIds[3] || null,
      hand_left: partIds[4] || null,
      hand_right: partIds[5] || null,
      leg_left: partIds[6] || null,
      leg_right: partIds[7] || null,
      foot_left: partIds[8] || null,
      foot_right: partIds[9] || null
    }
  };
}

function characterToAssemblyId(character) {
  const p = character.parts;
  return [
    p.head, p.torso, p.arm_left, p.arm_right, p.hand_left,
    p.hand_right, p.leg_left, p.leg_right, p.foot_left, p.foot_right
  ].map(id => id || "null").join("|");
}

function generateRandomAssemblyId(seed, isNPC) {
  const rng = seededRandom(seed);
  const pool = isNPC ? "npc" : "monster";
  
  const pickPart = (category, allowNull = false) => {
    const parts = PartPools[category]?.[pool] || PartPools[category]?.any || [];
    if (parts.length === 0 || (allowNull && rng() > 0.8)) return "null";
    return parts[Math.floor(rng() * parts.length)];
  };
  
  return [
    pickPart("head"),
    pickPart("torso"),
    pickPart("arm"),        // arm_left
    pickPart("arm"),        // arm_right
    pickPart("hand"),       // hand_left (could be shield)
    rng() > 0.3 ? pickPart("weapon", true) : pickPart("hand"),  // hand_right
    pickPart("leg"),        // leg_left
    pickPart("leg"),        // leg_right
    pickPart("foot"),       // foot_left
    pickPart("foot")        // foot_right
  ].join("|");
}
```

**Entity Factory with PuppetJSX Integration:**
```javascript
function createEntity(assemblyId, position, isNPC, animations) {
  const id = `entity_${Date.now()}_${Math.random().toString(36).slice(2, 6)}`;
  
  // Convert assembly ID to PuppetJSX character format
  const puppetData = assemblyIdToCharacter(assemblyId, id);
  
  // Calculate combined stats from all parts
  const stats = calculateStatsFromParts(puppetData.parts);
  
  // Create PuppetJSX runtime instance for animation playback
  const puppetRuntime = new PuppetCharacter(
    { version: "1.0", type: "character", data: puppetData },
    defaultSkeleton,
    animations.idle,
    loadedPartImages
  );
  
  return {
    id,
    isNPC,
    position: { ...position },
    velocity: { x: 0, y: 0 },
    facing: "RIGHT",
    state: "IDLE",
    puppetData,
    puppetRuntime,
    currentAnimation: "idle",
    animationTime: 0,
    stats: {
      ...stats,
      currentHealth: stats.maxHealth
    },
    ai: {
      targetPosition: null,
      aggroTarget: null,
      lastStateChange: Date.now(),
      behaviorType: isNPC ? "PASSIVE" : "AGGRESSIVE"
    },
    sessionXP: 0,
    hasTorch: false
  };
}

function calculateStatsFromParts(parts) {
  let health = 50, damage = 5, speed = 100, armor = 0;
  
  for (const partId of Object.values(parts)) {
    if (!partId || partId === "null") continue;
    const part = getPartById(partId);
    if (part?.stats) {
      health += part.stats.health || 0;
      damage += part.stats.damage || 0;
      speed *= part.stats.speed || 1;
      armor += part.stats.armor || 0;
    }
  }
  
  return { maxHealth: health, damage, speed, armor, attackSpeed: 1 };
}
```

### 4. Possession Mechanic

**Attraction Phase:**
1. Torch on ground emits attraction pulse every 500ms
2. All NPCs (`isNPC === true`) within `attractionRadius` receive attraction event
3. Closest NPC changes state to `ATTRACTED`
4. Attracted NPC pathfinds directly to torch position
5. Other attracted NPCs revert to previous state if closer NPC found

**Possession Trigger:**
```javascript
function checkPossession(torch, entities) {
  if (torch.state !== "GROUNDED") return;
  
  const nearbyNPCs = entities.filter(e => 
    e.isNPC && 
    e.state !== "DEAD" &&
    distance(torch.position, e.position) < PICKUP_RADIUS  // 24px
  );
  
  if (nearbyNPCs.length > 0) {
    const host = nearbyNPCs[0];  // Closest
    initiatePossession(torch, host);
  }
}

function initiatePossession(torch, host) {
  torch.state = "HELD";
  torch.currentHost = host.id;
  torch.isHeld = true;
  
  host.state = "POSSESSED";
  host.ai.targetPosition = null;
  host.ai.aggroTarget = null;
  
  // Emit possession event for UI/audio
  emit("POSSESSION_START", { torch, host });
}
```

**Possession End (Host Death):**
```javascript
function onHostDeath(torch, host) {
  // Transfer XP
  torch.totalXP += host.sessionXP;
  checkTorchLevelUp(torch);
  
  // Drop torch at host position
  torch.position = { ...host.position };
  torch.state = "FALLING";
  torch.currentHost = null;
  torch.isHeld = false;
  
  // After fall animation
  setTimeout(() => {
    torch.state = "GROUNDED";
    emit("TORCH_DROPPED", { torch, position: torch.position });
  }, 300);
}
```

### 5. Player Controls

Controls apply to the currently possessed NPC, not the torch directly.

**Input Mapping:**
```javascript
Controls = {
  movement: {
    up: ["W", "ArrowUp"],
    down: ["S", "ArrowDown"],
    left: ["A", "ArrowLeft"],
    right: ["D", "ArrowRight"]
  },
  actions: {
    attack: ["Space", "MouseLeft"],
    interact: ["E"],
    drop: ["Q"]  // Voluntarily release torch (debug/advanced)
  }
}
```

**Movement Processing:**
```javascript
function processInput(inputState, host, deltaTime) {
  if (!host || host.state !== "POSSESSED") return;
  
  const moveVector = { x: 0, y: 0 };
  
  if (inputState.up) moveVector.y -= 1;
  if (inputState.down) moveVector.y += 1;
  if (inputState.left) moveVector.x -= 1;
  if (inputState.right) moveVector.x += 1;
  
  // Normalize diagonal movement
  const magnitude = Math.sqrt(moveVector.x ** 2 + moveVector.y ** 2);
  if (magnitude > 0) {
    moveVector.x /= magnitude;
    moveVector.y /= magnitude;
  }
  
  // Apply speed
  host.velocity.x = moveVector.x * host.stats.speed;
  host.velocity.y = moveVector.y * host.stats.speed;
  
  // Update facing direction
  if (moveVector.x < 0) host.facing = "LEFT";
  else if (moveVector.x > 0) host.facing = "RIGHT";
  else if (moveVector.y < 0) host.facing = "UP";
  else if (moveVector.y > 0) host.facing = "DOWN";
}
```

### 6. Fog of War / Darkness System

The entire game world exists in darkness. Only the torch illuminates.

**Rendering Approach:**
```javascript
// 1. Render all visible game elements to offscreen canvas
// 2. Create darkness overlay (full black)
// 3. Cut out light circle using composite operation
// 4. Apply soft edge gradient to light boundary
// 5. Composite darkness over game layer

DarknessSystem = {
  ambientLight: 0,                    // 0 = pitch black, 1 = full bright
  lightSources: [                      // Only torch by default
    { 
      id: "torch", 
      position: dynamicFromTorch,
      radius: dynamicFromTorchLevel,
      intensity: 1.0,
      color: "#FFA500",               // Orange torch glow
      flicker: true,
      flickerIntensity: 0.1,
      flickerSpeed: 8                 // Hz
    }
  ]
}
```

**Light Rendering (Canvas 2D):**
```javascript
function renderDarkness(ctx, width, height, lightSources) {
  // Create darkness layer
  ctx.save();
  ctx.globalCompositeOperation = "source-over";
  ctx.fillStyle = "rgba(0, 0, 0, 1)";
  ctx.fillRect(0, 0, width, height);
  
  // Cut out light areas
  ctx.globalCompositeOperation = "destination-out";
  
  for (const light of lightSources) {
    const gradient = ctx.createRadialGradient(
      light.position.x, light.position.y, 0,
      light.position.x, light.position.y, light.radius
    );
    
    const flicker = light.flicker 
      ? 1 - (Math.sin(Date.now() * light.flickerSpeed * 0.001) * light.flickerIntensity)
      : 1;
    
    gradient.addColorStop(0, `rgba(255, 255, 255, ${light.intensity * flicker})`);
    gradient.addColorStop(0.7, `rgba(255, 255, 255, ${light.intensity * flicker * 0.5})`);
    gradient.addColorStop(1, "rgba(255, 255, 255, 0)");
    
    ctx.fillStyle = gradient;
    ctx.beginPath();
    ctx.arc(light.position.x, light.position.y, light.radius, 0, Math.PI * 2);
    ctx.fill();
  }
  
  ctx.restore();
}
```

**Visibility Culling:**
```javascript
function isVisible(entityPosition, lightSources) {
  for (const light of lightSources) {
    if (distance(entityPosition, light.position) < light.radius * 1.2) {
      return true;
    }
  }
  return false;
}

// Entities outside light radius:
// - Do not render sprites
// - AI still processes (monsters move in darkness)
// - Sounds still emit (audio cue system)
```

### 7. Combat System

**Attack Mechanics:**
```javascript
function performAttack(attacker) {
  if (attacker.state === "DEAD") return;
  
  const attackRange = attacker.parts.weapon?.stats.range || 32;  // pixels
  const attackArc = 90;  // degrees
  const attackDamage = attacker.stats.damage;
  
  // Find targets in attack cone
  const facingAngle = FACING_ANGLES[attacker.facing];
  const targets = entities.filter(e => {
    if (e.id === attacker.id || e.state === "DEAD") return false;
    
    const dist = distance(attacker.position, e.position);
    if (dist > attackRange) return false;
    
    const angle = Math.atan2(
      e.position.y - attacker.position.y,
      e.position.x - attacker.position.x
    ) * (180 / Math.PI);
    
    const angleDiff = Math.abs(normalizeAngle(angle - facingAngle));
    return angleDiff < attackArc / 2;
  });
  
  // Apply damage
  for (const target of targets) {
    const finalDamage = Math.max(1, attackDamage - target.stats.armor);
    target.stats.currentHealth -= finalDamage;
    
    emit("DAMAGE_DEALT", { attacker, target, damage: finalDamage });
    
    if (target.stats.currentHealth <= 0) {
      killEntity(target, attacker);
    }
  }
}

function killEntity(entity, killer) {
  entity.state = "DEAD";
  entity.velocity = { x: 0, y: 0 };
  
  // XP reward
  const xpReward = calculateXPReward(entity);
  if (killer.state === "POSSESSED") {
    killer.sessionXP += xpReward;
    emit("XP_GAINED", { amount: xpReward, total: killer.sessionXP });
  }
  
  emit("ENTITY_DEATH", { entity, killer });
  
  // If this was the possessed host, trigger torch drop
  if (entity.state === "POSSESSED") {
    onHostDeath(torch, entity);
  }
}
```

### 8. Procedural Dungeon Generation

**Dungeon Schema:**
```javascript
Dungeon = {
  seed: number,
  width: number,                         // Grid width (tiles)
  height: number,                        // Grid height (tiles)
  tileSize: 32,                          // Pixels per tile
  
  tiles: Tile[][],                       // 2D grid
  rooms: Room[],
  corridors: Corridor[],
  
  spawnPoints: SpawnPoint[],
  propPlacements: PropPlacement[],       // From level editor overlays
  
  // Computed navigation
  navMesh: NavNode[],
  collisionMap: boolean[][]              // true = blocked
}

Tile = {
  type: "FLOOR" | "WALL" | "PIT" | "DOOR" | "STAIRS",
  variant: number,                       // Visual variant index
  revealed: boolean,                     // Has player seen this tile?
  passable: boolean
}

Room = {
  id: string,
  bounds: { x: number, y: number, width: number, height: number },
  type: "SPAWN" | "COMBAT" | "TREASURE" | "BOSS" | "HUB",
  connectedTo: string[],                 // Room IDs
  cleared: boolean
}
```

**Generation Algorithm (BSP + Cellular Automata hybrid):**
```javascript
function generateDungeon(seed, width, height) {
  const rng = seededRandom(seed);
  
  // 1. BSP partitioning for room placement
  const partitions = bspPartition(width, height, 8, 15, rng);  // min 8, max 15 room size
  
  // 2. Create rooms within partitions (with padding)
  const rooms = partitions.map((p, i) => createRoom(p, i, rng));
  
  // 3. Connect rooms via corridors (minimum spanning tree + some extra)
  const corridors = connectRooms(rooms, rng);
  
  // 4. Convert to tile grid
  const tiles = createTileGrid(width, height, rooms, corridors);
  
  // 5. Cellular automata pass for organic feel (optional caves)
  // applyCAPass(tiles, rng);
  
  // 6. Place doors at corridor/room boundaries
  placeDoors(tiles, rooms, corridors);
  
  // 7. Designate spawn room (furthest from center)
  const spawnRoom = rooms.reduce((best, r) => 
    distance(roomCenter(r), { x: width/2, y: height/2 }) >
    distance(roomCenter(best), { x: width/2, y: height/2 }) ? r : best
  );
  spawnRoom.type = "SPAWN";
  
  // 8. Generate navigation mesh
  const navMesh = generateNavMesh(tiles);
  
  // 9. Create collision map
  const collisionMap = tiles.map(row => row.map(t => !t.passable));
  
  return { seed, width, height, tileSize: 32, tiles, rooms, corridors, navMesh, collisionMap, spawnPoints: [], propPlacements: [] };
}
```

**Seeded Random Implementation:**
```javascript
function seededRandom(seed) {
  let state = seed;
  return function() {
    state = (state * 1103515245 + 12345) & 0x7fffffff;
    return state / 0x7fffffff;
  };
}
```

### 9. Proximity-Based Spawning System

Entities spawn dynamically based on player (torch) position, not preloaded.

**SpawnPoint Schema:**
```javascript
SpawnPoint = {
  id: string,
  position: { x: number, y: number },
  spawnRadius: number,                   // Distance from torch to trigger
  despawnRadius: number,                 // Distance to despawn (larger)
  maxEntities: number,                   // Max concurrent from this point
  spawnInterval: number,                 // ms between spawns
  lastSpawn: number,                     // timestamp
  
  entityPool: {
    assemblyIds: string[],               // Possible assemblies to spawn
    weights: number[],                   // Probability weights
    isNPC: boolean                       // NPCs vs monsters
  },
  
  activeEntities: string[],              // IDs of spawned entities
  totalSpawned: number,
  budget: number | null                  // null = infinite
}
```

**Spawn Logic:**
```javascript
function updateSpawning(torch, spawnPoints, entities, deltaTime) {
  const torchPos = torch.isHeld 
    ? entities.find(e => e.id === torch.currentHost)?.position 
    : torch.position;
  
  for (const sp of spawnPoints) {
    const dist = distance(torchPos, sp.position);
    
    // Despawn check
    sp.activeEntities = sp.activeEntities.filter(id => {
      const entity = entities.find(e => e.id === id);
      if (!entity) return false;
      
      const entityDist = distance(torchPos, entity.position);
      if (entityDist > sp.despawnRadius && entity.state !== "POSSESSED") {
        despawnEntity(entity);
        return false;
      }
      return true;
    });
    
    // Spawn check
    if (dist < sp.spawnRadius && 
        sp.activeEntities.length < sp.maxEntities &&
        (sp.budget === null || sp.totalSpawned < sp.budget) &&
        Date.now() - sp.lastSpawn > sp.spawnInterval) {
      
      const assemblyId = weightedRandom(sp.entityPool.assemblyIds, sp.entityPool.weights);
      const spawnPos = findValidSpawnPosition(sp.position, 64, entities);
      
      if (spawnPos) {
        const entity = spawnEntity(assemblyId, spawnPos, sp.entityPool.isNPC);
        sp.activeEntities.push(entity.id);
        sp.totalSpawned++;
        sp.lastSpawn = Date.now();
      }
    }
  }
}
```

### 10. Level Editor Overlay System

The level editor creates "customization layers" that apply on top of procedurally generated dungeons.

**Overlay Schema:**
```javascript
LevelOverlay = {
  id: string,
  name: string,
  targetSeed: number | null,             // null = apply to any dungeon
  
  propOverrides: PropPlacement[],
  spawnOverrides: SpawnPoint[],
  tileOverrides: TileOverride[],
  triggerZones: TriggerZone[]
}

PropPlacement = {
  id: string,
  propType: string,                      // "barrel", "chest", "torch_wall", etc.
  position: { x: number, y: number },    // World position
  rotation: number,
  scale: number,
  interactive: boolean,
  data: object                           // Prop-specific data (chest contents, etc.)
}

TileOverride = {
  position: { x: number, y: number },    // Grid position
  newType: Tile["type"],
  newVariant: number
}

TriggerZone = {
  id: string,
  bounds: { x: number, y: number, width: number, height: number },
  triggerType: "ENTER" | "EXIT" | "STAY",
  action: string,                        // Action ID to execute
  data: object,
  oneShot: boolean,
  triggered: boolean
}
```

**Editor Component Interface:**
```javascript
// Level Editor provides:
// 1. Seed input to generate base dungeon
// 2. Visual overlay of dungeon with grid
// 3. Drag-drop prop palette
// 4. Spawn point placement tool
// 5. Tile painting brush
// 6. Trigger zone drawing tool
// 7. Export/Import overlay JSON
// 8. Test play button (loads dungeon + overlay)
```

---

## Technical Implementation

### Component Architecture

```
src/
├── components/
│   ├── game/
│   │   ├── GameCanvas.jsx              # Main render surface
│   │   ├── GameLoop.jsx                # RAF loop, deltaTime
│   │   └── Camera.jsx                  # Viewport following torch/host
│   │
│   ├── puppet/                         # PuppetJSX Integration
│   │   ├── PuppetJSX.jsx               # Full editor (dev mode only)
│   │   ├── PuppetRuntime.js            # Runtime class extracted from PuppetJSX
│   │   ├── PuppetRenderer.jsx          # Renders a PuppetCharacter to canvas
│   │   ├── SkeletonEngine.js           # Transform calculations
│   │   └── AnimationEngine.js          # Frame interpolation & playback
│   │
│   ├── entities/
│   │   ├── Torch.jsx                   # Torch rendering & logic
│   │   ├── Entity.jsx                  # Generic entity wrapper
│   │   ├── EntityPuppet.jsx            # Entity + PuppetJSX integration
│   │   └── EntityAnimator.jsx          # Animation state machine
│   │
│   ├── world/
│   │   ├── Dungeon.jsx                 # Dungeon container
│   │   ├── TileRenderer.jsx            # Efficient tile batching
│   │   ├── DarknessOverlay.jsx         # Fog of war
│   │   └── Props.jsx                   # Interactive props
│   │
│   ├── systems/
│   │   ├── InputSystem.jsx             # Keyboard/mouse handling
│   │   ├── PhysicsSystem.jsx           # Movement, collision
│   │   ├── CombatSystem.jsx            # Attack resolution
│   │   ├── AISystem.jsx                # NPC/monster behaviors
│   │   ├── SpawnSystem.jsx             # Proximity spawning
│   │   └── PossessionSystem.jsx        # Torch <-> host binding
│   │
│   ├── ui/
│   │   ├── HUD.jsx                     # Health, XP, host info
│   │   ├── TorchStatusDisplay.jsx      # Torch level, total XP
│   │   ├── Minimap.jsx                 # Revealed areas (optional)
│   │   └── DeathScreen.jsx             # Host death transition
│   │
│   └── editor/
│       ├── LevelEditor.jsx             # Main editor container
│       ├── EditorToolbar.jsx           # Tool selection
│       ├── PropPalette.jsx             # Draggable props
│       ├── SpawnPointTool.jsx          # Place/configure spawns
│       ├── TileBrush.jsx               # Paint tiles
│       ├── TriggerZoneTool.jsx         # Define triggers
│       └── CharacterEditorLauncher.jsx # Opens PuppetJSX for part/anim editing
│
├── hooks/
│   ├── useGameLoop.js                  # Custom RAF hook
│   ├── useInput.js                     # Input state hook
│   ├── useEntities.js                  # Entity management
│   ├── useDungeon.js                   # Dungeon generation
│   └── usePuppet.js                    # PuppetJSX runtime hook
│
├── lib/
│   ├── puppetjsx/                      # Extracted PuppetJSX core modules
│   │   ├── calculateWorldTransform.js  # Bone transform math
│   │   ├── interpolateFrames.js        # Animation interpolation
│   │   └── PuppetCharacter.js          # Runtime class from integration doc
│   │
│   ├── dungeonGenerator.js             # BSP + tile generation
│   ├── pathfinding.js                  # A* implementation
│   └── seededRandom.js                 # Deterministic RNG
│
├── data/
│   ├── skeleton.json                   # PuppetJSX humanoid skeleton definition
│   ├── parts/                          # Part definitions (extended PuppetJSX format)
│   │   ├── heads.json
│   │   ├── torsos.json
│   │   ├── arms.json
│   │   ├── hands.json
│   │   ├── legs.json
│   │   ├── feet.json
│   │   ├── weapons.json
│   │   └── accessories.json
│   │
│   ├── animations/                     # PuppetJSX animation JSONs
│   │   ├── idle.json
│   │   ├── walk.json
│   │   ├── attack.json
│   │   ├── hit.json
│   │   └── death.json
│   │
│   ├── characters/                     # Pre-built character templates
│   │   ├── npc_villager.json
│   │   ├── npc_merchant.json
│   │   ├── monster_goblin.json
│   │   └── monster_skeleton.json
│   │
│   ├── props.json                      # Environmental prop definitions
│   ├── tiles.json                      # Tile type definitions
│   └── spawns.json                     # Default spawn configs
│
└── assets/
    ├── sprites/
    │   ├── parts/                      # Character part sprites (match PuppetJSX IDs)
    │   │   ├── head_basic.png
    │   │   ├── head_goblin_01.png
    │   │   ├── torso_basic.png
    │   │   └── ...
    │   ├── props/                      # Environmental props
    │   ├── tiles/                      # Floor/wall tiles
    │   └── fx/                         # Effects (torch glow, hit, etc.)
    │
    └── audio/
        ├── sfx/                        # Sound effects
        └── ambient/                    # Background loops
```

### PuppetJSX Runtime Integration

The key integration point is the `PuppetCharacter` class from the PuppetJSX integration guide:

```javascript
// lib/puppetjsx/PuppetCharacter.js
// This is the runtime class for playing animations outside the editor

class PuppetCharacter {
  constructor(characterJSON, skeletonJSON, animationJSON, partsImages) {
    this.character = characterJSON.data;
    this.skeleton = skeletonJSON;
    this.animation = animationJSON.data;
    this.parts = partsImages;  // { partId: { image: HTMLImageElement, ...partData } }
    this.currentTime = 0;
    this.isPlaying = true;
    this.playbackSpeed = 1.0;
  }
  
  update(deltaTime) {
    if (!this.isPlaying) return;
    
    this.currentTime += deltaTime * this.playbackSpeed;
    
    const totalDuration = this.animation.frames.reduce((sum, f) => sum + f.duration, 0);
    
    if (this.currentTime >= totalDuration) {
      if (this.animation.loop) {
        this.currentTime = this.currentTime % totalDuration;
      } else {
        this.currentTime = totalDuration;
        this.isPlaying = false;
      }
    }
  }
  
  getCurrentFrameData() {
    let accumulatedTime = 0;
    
    for (let i = 0; i < this.animation.frames.length; i++) {
      const frame = this.animation.frames[i];
      const frameEndTime = accumulatedTime + frame.duration;
      
      if (this.currentTime < frameEndTime) {
        const nextIndex = (i + 1) % this.animation.frames.length;
        const progress = (this.currentTime - accumulatedTime) / frame.duration;
        
        return {
          currentFrame: frame,
          nextFrame: this.animation.frames[nextIndex],
          progress
        };
      }
      
      accumulatedTime = frameEndTime;
    }
    
    return {
      currentFrame: this.animation.frames[0],
      nextFrame: this.animation.frames[1] || this.animation.frames[0],
      progress: 0
    };
  }
  
  interpolateAndCalculateTransforms(frameData) {
    const { currentFrame, nextFrame, progress } = frameData;
    const interpolated = {};
    
    // Interpolate bone transforms between frames
    for (const boneId in this.skeleton.bones) {
      const bone = this.skeleton.bones[boneId];
      const t1 = currentFrame?.bones?.[boneId] || bone.localTransform;
      const t2 = nextFrame?.bones?.[boneId] || bone.localTransform;
      
      interpolated[boneId] = {
        position: {
          x: t1.position.x + (t2.position.x - t1.position.x) * progress,
          y: t1.position.y + (t2.position.y - t1.position.y) * progress
        },
        rotation: lerpAngle(t1.rotation, t2.rotation, progress),
        scale: {
          x: t1.scale.x + (t2.scale.x - t1.scale.x) * progress,
          y: t1.scale.y + (t2.scale.y - t1.scale.y) * progress
        }
      };
    }
    
    // Calculate world transforms from local
    return calculateAllWorldTransforms(this.skeleton, interpolated);
  }
  
  render(ctx, x, y, scale = 1, flipX = false) {
    const frameData = this.getCurrentFrameData();
    const transforms = this.interpolateAndCalculateTransforms(frameData);
    
    // Build render list sorted by z-index
    const renderList = [];
    
    for (const [boneId, partId] of Object.entries(this.character.parts)) {
      if (!partId || partId === "null") continue;
      
      const partData = this.parts[partId];
      if (!partData) continue;
      
      const bone = this.skeleton.bones[boneId];
      const worldTransform = transforms[boneId];
      if (!worldTransform) continue;
      
      renderList.push({
        partData,
        x: worldTransform.position.x + (partData.offset?.x || 0),
        y: worldTransform.position.y + (partData.offset?.y || 0),
        rotation: worldTransform.rotation,
        scale: worldTransform.scale,
        zIndex: (bone.zIndex || 0) + (partData.zIndexModifier || 0)
      });
    }
    
    renderList.sort((a, b) => a.zIndex - b.zIndex);
    
    // Render all parts
    ctx.save();
    ctx.translate(x, y);
    if (flipX) ctx.scale(-1, 1);
    ctx.scale(scale, scale);
    
    for (const item of renderList) {
      ctx.save();
      ctx.translate(item.x, item.y);
      ctx.rotate(item.rotation * Math.PI / 180);
      ctx.scale(item.scale.x, item.scale.y);
      
      if (item.partData.image) {
        // Draw sprite
        ctx.drawImage(
          item.partData.image, 
          -item.partData.width / 2, 
          -item.partData.height / 2
        );
      } else {
        // Fallback: draw colored rectangle
        ctx.fillStyle = item.partData.color || "#888";
        ctx.fillRect(
          -item.partData.width / 2,
          -item.partData.height / 2,
          item.partData.width,
          item.partData.height
        );
      }
      
      ctx.restore();
    }
    
    ctx.restore();
  }
  
  playAnimation(animationJSON) {
    this.animation = animationJSON.data || animationJSON;
    this.currentTime = 0;
    this.isPlaying = true;
  }
  
  stop() {
    this.isPlaying = false;
  }
  
  reset() {
    this.currentTime = 0;
  }
}

// Helper: angle interpolation
function lerpAngle(a, b, t) {
  let delta = b - a;
  while (delta > 180) delta -= 360;
  while (delta < -180) delta += 360;
  return a + delta * t;
}

export default PuppetCharacter;
```

### State Management

```javascript
// Central game state (React Context + useReducer)
GameState = {
  // Core state
  torch: Torch,
  entities: Map<string, Entity>,
  dungeon: Dungeon,
  
  // Runtime state
  gameTime: number,
  isPaused: boolean,
  gamePhase: "LOADING" | "PLAYING" | "PAUSED" | "HOST_DEATH" | "GAME_OVER",
  
  // Camera
  camera: {
    position: { x: number, y: number },
    zoom: number,
    shake: { x: number, y: number, duration: number }
  },
  
  // Input (updated each frame)
  input: {
    movement: { x: number, y: number },
    attack: boolean,
    interact: boolean
  },
  
  // Level editor state (separate context)
  editor: {
    isActive: boolean,
    selectedTool: string,
    overlay: LevelOverlay,
    history: LevelOverlay[]  // Undo stack
  }
}
```

### Main Game Loop

```jsx
// GameLoop.jsx
function GameLoop({ children }) {
  const { state, dispatch } = useGameState();
  const inputRef = useRef(useInput());
  const lastTimeRef = useRef(performance.now());
  
  useEffect(() => {
    let animationId;
    
    function tick(currentTime) {
      const deltaTime = (currentTime - lastTimeRef.current) / 1000;
      lastTimeRef.current = currentTime;
      
      if (!state.isPaused) {
        // 1. Process input
        const input = inputRef.current.getState();
        
        // 2. Update possession system
        updatePossession(state.torch, state.entities);
        
        // 3. Process player input (if host exists)
        if (state.torch.currentHost) {
          const host = state.entities.get(state.torch.currentHost);
          processPlayerInput(input, host, deltaTime);
        }
        
        // 4. Update AI for non-possessed entities
        updateAI(state.entities, state.torch, deltaTime);
        
        // 5. Update physics/movement
        updatePhysics(state.entities, state.dungeon.collisionMap, deltaTime);
        
        // 6. Process combat
        updateCombat(state.entities, deltaTime);
        
        // 7. Update spawning
        updateSpawning(state.torch, state.dungeon.spawnPoints, state.entities, deltaTime);
        
        // 8. Update camera
        updateCamera(state.camera, state.torch, state.entities);
        
        // 9. Update animations
        updateAnimations(state.entities, deltaTime);
        
        dispatch({ type: "TICK", deltaTime });
      }
      
      animationId = requestAnimationFrame(tick);
    }
    
    animationId = requestAnimationFrame(tick);
    return () => cancelAnimationFrame(animationId);
  }, [state.isPaused]);
  
  return children;
}
```

### Entity Rendering with PuppetJSX

```jsx
// EntityPuppet.jsx - Renders an entity using PuppetJSX runtime
function EntityPuppet({ entity, canvasRef, camera, isLit }) {
  const puppetRef = useRef(null);
  
  // Initialize or update PuppetCharacter runtime
  useEffect(() => {
    if (!entity.puppetRuntime) return;
    puppetRef.current = entity.puppetRuntime;
  }, [entity.puppetRuntime]);
  
  // Animation state machine
  useEffect(() => {
    if (!puppetRef.current) return;
    
    const animationMap = {
      "IDLE": animations.idle,
      "WANDERING": animations.walk,
      "ATTRACTED": animations.walk,
      "POSSESSED": entity.velocity.x !== 0 || entity.velocity.y !== 0 
        ? animations.walk 
        : animations.idle,
      "COMBAT": animations.attack,
      "DEAD": animations.death
    };
    
    const targetAnim = animationMap[entity.state] || animations.idle;
    if (puppetRef.current.animation.id !== targetAnim.data.id) {
      puppetRef.current.playAnimation(targetAnim);
    }
  }, [entity.state, entity.velocity]);
  
  // Don't render if not lit
  if (!isLit || entity.state === "DEAD") return null;
  
  // Render to canvas
  const ctx = canvasRef.current?.getContext("2d");
  if (!ctx || !puppetRef.current) return null;
  
  // Convert world position to screen position
  const screenPos = worldToScreen(entity.position, camera);
  
  // Update animation
  puppetRef.current.update(16.67); // Assume 60fps for now
  
  // Render with facing direction (flipX for LEFT)
  puppetRef.current.render(
    ctx, 
    screenPos.x, 
    screenPos.y, 
    camera.zoom,
    entity.facing === "LEFT"
  );
  
  // Render torch attachment if entity has torch
  if (entity.hasTorch) {
    renderTorchOnHand(ctx, puppetRef.current, screenPos, camera);
  }
  
  return null; // Rendering is imperative to canvas
}

function renderTorchOnHand(ctx, puppet, screenPos, camera) {
  // Get world transform of hand_left bone (where torch attaches)
  const frameData = puppet.getCurrentFrameData();
  const transforms = puppet.interpolateAndCalculateTransforms(frameData);
  const handTransform = transforms.hand_left;
  
  if (!handTransform) return;
  
  // Draw torch glow at hand position
  ctx.save();
  ctx.translate(
    screenPos.x + handTransform.position.x * camera.zoom,
    screenPos.y + handTransform.position.y * camera.zoom
  );
  
  // Torch flame sprite or simple glow
  const gradient = ctx.createRadialGradient(0, -8, 2, 0, -8, 16);
  gradient.addColorStop(0, "rgba(255, 200, 50, 1)");
  gradient.addColorStop(0.5, "rgba(255, 100, 0, 0.6)");
  gradient.addColorStop(1, "rgba(255, 50, 0, 0)");
  
  ctx.fillStyle = gradient;
  ctx.beginPath();
  ctx.arc(0, -8, 16, 0, Math.PI * 2);
  ctx.fill();
  
  // Torch handle
  ctx.fillStyle = "#4A3728";
  ctx.fillRect(-2, -4, 4, 20);
  
  ctx.restore();
}

// Animation data registry
const animations = {
  idle: null,   // Loaded from data/animations/idle.json
  walk: null,   // Loaded from data/animations/walk.json
  attack: null, // Loaded from data/animations/attack.json
  hit: null,    // Loaded from data/animations/hit.json
  death: null   // Loaded from data/animations/death.json
};

async function loadAnimations() {
  animations.idle = await fetch('/data/animations/idle.json').then(r => r.json());
  animations.walk = await fetch('/data/animations/walk.json').then(r => r.json());
  animations.attack = await fetch('/data/animations/attack.json').then(r => r.json());
  animations.hit = await fetch('/data/animations/hit.json').then(r => r.json());
  animations.death = await fetch('/data/animations/death.json').then(r => r.json());
}
```

**PuppetJSX Animation Format (for reference):**
```json
{
  "version": "1.0",
  "type": "animation",
  "data": {
    "id": "anim_walk_001",
    "name": "Walk Cycle",
    "skeletonId": "humanoid_skeleton",
    "loop": true,
    "frames": [
      {
        "index": 0,
        "duration": 100,
        "bones": {
          "arm_left": { "position": { "x": -15, "y": -10 }, "rotation": -20, "scale": { "x": 1, "y": 1 } },
          "arm_right": { "position": { "x": 15, "y": -10 }, "rotation": 20, "scale": { "x": 1, "y": 1 } },
          "leg_left": { "position": { "x": -8, "y": 15 }, "rotation": -30, "scale": { "x": 1, "y": 1 } },
          "leg_right": { "position": { "x": 8, "y": 15 }, "rotation": 30, "scale": { "x": 1, "y": 1 } }
        }
      },
      {
        "index": 1,
        "duration": 100,
        "bones": {
          "torso": { "position": { "x": 0, "y": -22 }, "rotation": 0, "scale": { "x": 1, "y": 1 } },
          "arm_left": { "position": { "x": -15, "y": -10 }, "rotation": 0, "scale": { "x": 1, "y": 1 } },
          "arm_right": { "position": { "x": 15, "y": -10 }, "rotation": 0, "scale": { "x": 1, "y": 1 } },
          "leg_left": { "position": { "x": -8, "y": 15 }, "rotation": 0, "scale": { "x": 1, "y": 1 } },
          "leg_right": { "position": { "x": 8, "y": 15 }, "rotation": 0, "scale": { "x": 1, "y": 1 } }
        }
      },
      {
        "index": 2,
        "duration": 100,
        "bones": {
          "arm_left": { "position": { "x": -15, "y": -10 }, "rotation": 20, "scale": { "x": 1, "y": 1 } },
          "arm_right": { "position": { "x": 15, "y": -10 }, "rotation": -20, "scale": { "x": 1, "y": 1 } },
          "leg_left": { "position": { "x": -8, "y": 15 }, "rotation": 30, "scale": { "x": 1, "y": 1 } },
          "leg_right": { "position": { "x": 8, "y": 15 }, "rotation": -30, "scale": { "x": 1, "y": 1 } }
        }
      },
      {
        "index": 3,
        "duration": 100,
        "bones": {
          "torso": { "position": { "x": 0, "y": -22 }, "rotation": 0, "scale": { "x": 1, "y": 1 } },
          "arm_left": { "position": { "x": -15, "y": -10 }, "rotation": 0, "scale": { "x": 1, "y": 1 } },
          "arm_right": { "position": { "x": 15, "y": -10 }, "rotation": 0, "scale": { "x": 1, "y": 1 } },
          "leg_left": { "position": { "x": -8, "y": 15 }, "rotation": 0, "scale": { "x": 1, "y": 1 } },
          "leg_right": { "position": { "x": 8, "y": 15 }, "rotation": 0, "scale": { "x": 1, "y": 1 } }
        }
      }
    ]
  }
}
```

### Darkness Overlay Component

```jsx
// DarknessOverlay.jsx
function DarknessOverlay({ torch, entities, camera, canvasSize }) {
  const canvasRef = useRef(null);
  
  useEffect(() => {
    const ctx = canvasRef.current?.getContext("2d");
    if (!ctx) return;
    
    // Get torch position in screen space
    const torchWorldPos = torch.isHeld
      ? entities.get(torch.currentHost)?.position
      : torch.position;
    
    if (!torchWorldPos) return;
    
    const screenPos = worldToScreen(torchWorldPos, camera);
    
    // Render darkness with light cutout
    ctx.clearRect(0, 0, canvasSize.width, canvasSize.height);
    
    // Full darkness
    ctx.fillStyle = "black";
    ctx.fillRect(0, 0, canvasSize.width, canvasSize.height);
    
    // Cut out torch light
    ctx.globalCompositeOperation = "destination-out";
    
    const flickerOffset = Math.sin(Date.now() * 0.008) * torch.lightRadius * 0.05;
    const radius = torch.lightRadius + flickerOffset;
    
    const gradient = ctx.createRadialGradient(
      screenPos.x, screenPos.y, 0,
      screenPos.x, screenPos.y, radius
    );
    gradient.addColorStop(0, "rgba(255, 255, 255, 1)");
    gradient.addColorStop(0.6, "rgba(255, 255, 255, 0.8)");
    gradient.addColorStop(0.85, "rgba(255, 255, 255, 0.3)");
    gradient.addColorStop(1, "rgba(255, 255, 255, 0)");
    
    ctx.fillStyle = gradient;
    ctx.beginPath();
    ctx.arc(screenPos.x, screenPos.y, radius, 0, Math.PI * 2);
    ctx.fill();
    
    ctx.globalCompositeOperation = "source-over";
    
    // Add warm glow overlay
    ctx.globalCompositeOperation = "source-atop";
    const glowGradient = ctx.createRadialGradient(
      screenPos.x, screenPos.y, 0,
      screenPos.x, screenPos.y, radius * 0.7
    );
    glowGradient.addColorStop(0, "rgba(255, 150, 50, 0.15)");
    glowGradient.addColorStop(1, "rgba(255, 100, 0, 0)");
    ctx.fillStyle = glowGradient;
    ctx.fillRect(0, 0, canvasSize.width, canvasSize.height);
    
  }, [torch, entities, camera, canvasSize]);
  
  return (
    <canvas
      ref={canvasRef}
      width={canvasSize.width}
      height={canvasSize.height}
      style={{
        position: "absolute",
        top: 0,
        left: 0,
        pointerEvents: "none",
        zIndex: 100
      }}
    />
  );
}
```

---

## Style Guide

### Visual Aesthetic

**Overall:** Dark fantasy pixel art, 16x16 or 32x32 base sprites, limited palette per entity type.

**Color Palette:**
```
Background/Darkness: #000000, #0a0a0a
Stone Walls: #2a2a3a, #3a3a4a, #4a4a5a
Floor Tiles: #1a1a2a, #252535, #303045
Torch Light: #FFA500 → #FF6600 → #FF3300 (gradient)
Blood/Damage: #8B0000, #FF0000
UI Text: #FFFFFF, #CCCCCC
XP/Gold: #FFD700
Health: #00FF00 → #FF0000
```

**Sprite Guidelines:**
- All parts must align to the bone attachment system
- 4-frame walk cycles minimum
- 3-frame attack animations minimum
- 1-frame idle, 1-frame hit reaction
- Death animations: 4 frames collapsing
- Consistent lighting direction (top-left source for sprites)

### UI Style

```css
/* HUD styling */
.hud-container {
  font-family: "Press Start 2P", monospace;
  font-size: 8px;
  color: #ffffff;
  text-shadow: 2px 2px 0 #000000;
}

.health-bar {
  background: #1a1a1a;
  border: 2px solid #4a4a4a;
  height: 12px;
}

.health-bar-fill {
  background: linear-gradient(to right, #00ff00, #ffff00, #ff0000);
  height: 100%;
  transition: width 0.2s ease-out;
}

.xp-display {
  color: #ffd700;
  font-size: 10px;
}

.torch-level {
  color: #ffa500;
  animation: flicker 0.5s infinite alternate;
}

@keyframes flicker {
  from { opacity: 0.9; }
  to { opacity: 1.0; }
}
```

---

## Data Models

### PuppetJSX Skeleton Definition

```json
{
  "id": "humanoid_skeleton",
  "name": "Humanoid",
  "bones": {
    "root": {
      "id": "root",
      "parent": null,
      "children": ["torso"],
      "localTransform": { "position": { "x": 0, "y": 0 }, "rotation": 0, "scale": { "x": 1, "y": 1 } },
      "zIndex": 0
    },
    "torso": {
      "id": "torso",
      "parent": "root",
      "children": ["head", "arm_left", "arm_right", "leg_left", "leg_right"],
      "localTransform": { "position": { "x": 0, "y": -20 }, "rotation": 0, "scale": { "x": 1, "y": 1 } },
      "zIndex": 5
    },
    "head": {
      "id": "head",
      "parent": "torso",
      "children": [],
      "localTransform": { "position": { "x": 0, "y": -30 }, "rotation": 0, "scale": { "x": 1, "y": 1 } },
      "zIndex": 10
    },
    "arm_left": {
      "id": "arm_left",
      "parent": "torso",
      "children": ["hand_left"],
      "localTransform": { "position": { "x": -15, "y": -10 }, "rotation": 0, "scale": { "x": 1, "y": 1 } },
      "zIndex": 3
    },
    "hand_left": {
      "id": "hand_left",
      "parent": "arm_left",
      "children": [],
      "localTransform": { "position": { "x": 0, "y": 20 }, "rotation": 0, "scale": { "x": 1, "y": 1 } },
      "zIndex": 3
    },
    "arm_right": {
      "id": "arm_right",
      "parent": "torso",
      "children": ["hand_right"],
      "localTransform": { "position": { "x": 15, "y": -10 }, "rotation": 0, "scale": { "x": 1, "y": 1 } },
      "zIndex": 7
    },
    "hand_right": {
      "id": "hand_right",
      "parent": "arm_right",
      "children": [],
      "localTransform": { "position": { "x": 0, "y": 20 }, "rotation": 0, "scale": { "x": 1, "y": 1 } },
      "zIndex": 7
    },
    "leg_left": {
      "id": "leg_left",
      "parent": "torso",
      "children": ["foot_left"],
      "localTransform": { "position": { "x": -8, "y": 15 }, "rotation": 0, "scale": { "x": 1, "y": 1 } },
      "zIndex": 4
    },
    "foot_left": {
      "id": "foot_left",
      "parent": "leg_left",
      "children": [],
      "localTransform": { "position": { "x": 0, "y": 20 }, "rotation": 0, "scale": { "x": 1, "y": 1 } },
      "zIndex": 4
    },
    "leg_right": {
      "id": "leg_right",
      "parent": "torso",
      "children": ["foot_right"],
      "localTransform": { "position": { "x": 8, "y": 15 }, "rotation": 0, "scale": { "x": 1, "y": 1 } },
      "zIndex": 6
    },
    "foot_right": {
      "id": "foot_right",
      "parent": "leg_right",
      "children": [],
      "localTransform": { "position": { "x": 0, "y": 20 }, "rotation": 0, "scale": { "x": 1, "y": 1 } },
      "zIndex": 6
    }
  }
}
```

### Part Definition Example (Extended PuppetJSX Format)

```json
{
  "id": "head_goblin_01",
  "name": "Goblin Head",
  "category": "head",
  "width": 32,
  "height": 32,
  "color": "#7CB342",
  "imageUrl": "/assets/sprites/parts/head_goblin_01.png",
  "offset": { "x": 0, "y": 0 },
  "zIndexModifier": 0,
  "stats": {
    "health": 5,
    "damage": 0,
    "speed": 1.0,
    "armor": 0
  },
  "faction": "MONSTER",
  "rarity": 0.6
}
```

### Character Template Example (PuppetJSX Format)

```json
{
  "version": "1.0",
  "type": "character",
  "data": {
    "id": "char_goblin_warrior",
    "name": "Goblin Warrior",
    "skeletonId": "humanoid_skeleton",
    "parts": {
      "head": "head_goblin_01",
      "torso": "torso_goblin_01",
      "arm_left": "arm_goblin_01",
      "arm_right": "arm_goblin_01",
      "hand_left": "hand_claw_01",
      "hand_right": "weapon_club",
      "leg_left": "leg_goblin_01",
      "leg_right": "leg_goblin_01",
      "foot_left": "foot_claw_01",
      "foot_right": "foot_claw_01"
    }
  },
  "rtorch_meta": {
    "isNPC": false,
    "behaviorType": "AGGRESSIVE",
    "spawnWeight": 0.4,
    "xpValue": 15
  }
}
```

### Spawn Point Example

```json
{
  "id": "spawn_goblin_cave_01",
  "position": { "x": 512, "y": 384 },
  "spawnRadius": 300,
  "despawnRadius": 600,
  "maxEntities": 4,
  "spawnInterval": 5000,
  "entityPool": {
    "assemblyIds": [
      "head_goblin_01|torso_goblin_01|arm_goblin_01|arm_goblin_01|hand_claw_01|weapon_club|leg_goblin_01|leg_goblin_01|foot_claw_01|foot_claw_01",
      "head_goblin_02|torso_goblin_01|arm_goblin_01|arm_goblin_02|hand_basic|weapon_dagger|leg_goblin_01|leg_goblin_02|foot_basic|foot_basic"
    ],
    "characterTemplates": [
      "char_goblin_warrior",
      "char_goblin_scout"
    ],
    "weights": [0.7, 0.3],
    "isNPC": false
  },
  "budget": 20
}
```

### Level Overlay Example

```json
{
  "id": "overlay_tutorial_01",
  "name": "Tutorial Dungeon Overlay",
  "targetSeed": 12345,
  "propOverrides": [
    {
      "id": "prop_chest_01",
      "propType": "chest",
      "position": { "x": 256, "y": 192 },
      "rotation": 0,
      "scale": 1,
      "interactive": true,
      "data": {
        "contents": ["weapon_sword", "potion_health"],
        "locked": false
      }
    }
  ],
  "spawnOverrides": [
    {
      "id": "spawn_friendly_01",
      "position": { "x": 128, "y": 128 },
      "spawnRadius": 200,
      "despawnRadius": 400,
      "maxEntities": 1,
      "spawnInterval": 0,
      "entityPool": {
        "characterTemplates": ["char_villager_basic"],
        "weights": [1],
        "isNPC": true
      },
      "budget": 1
    }
  ],
  "tileOverrides": [],
  "triggerZones": [
    {
      "id": "trigger_tutorial_start",
      "bounds": { "x": 96, "y": 96, "width": 64, "height": 64 },
      "triggerType": "ENTER",
      "action": "SHOW_TUTORIAL_MESSAGE",
      "data": { "message": "WASD to move. Space to attack." },
      "oneShot": true,
      "triggered": false
    }
  ]
}
```

---

## PuppetJSX Editor Integration

The full PuppetJSX editor can be accessed from the Level Editor for creating custom characters and animations.

### Workflow: Creating Custom Entities

1. **Open PuppetJSX Editor** from Level Editor toolbar
2. **Design Character:**
   - Select parts from library for each bone
   - Use "Generate Random" for procedural combinations
   - Preview assembled character in real-time
3. **Create/Edit Animations:**
   - Switch to Animation Editor mode
   - Pose bones for each keyframe
   - Set frame durations
   - Preview with playback controls
4. **Export:**
   - Export character JSON → Save to `data/characters/`
   - Export animation JSON → Save to `data/animations/`
5. **Register in Game:**
   - Add character template ID to spawn pools
   - Reference animation IDs in EntityAnimator

### Using Exported Assets in R-Torch

```javascript
// Loading a character template created in PuppetJSX editor
async function loadCharacterTemplate(templateId) {
  const response = await fetch(`/data/characters/${templateId}.json`);
  const characterJSON = await response.json();
  
  // Convert to entity
  const assemblyId = characterToAssemblyId(characterJSON.data);
  return assemblyId;
}

// Spawning an entity from a template
function spawnFromTemplate(templateId, position, isNPC) {
  const template = loadedTemplates[templateId];
  const entity = createEntity(
    characterToAssemblyId(template.data),
    position,
    isNPC,
    animations
  );
  
  // Apply template metadata
  if (template.rtorch_meta) {
    entity.ai.behaviorType = template.rtorch_meta.behaviorType;
    entity.xpValue = template.rtorch_meta.xpValue;
  }
  
  return entity;
}
```

### Hot Reloading in Development

```javascript
// Dev mode: watch for changes to character/animation files
if (process.env.NODE_ENV === 'development') {
  const ws = new WebSocket('ws://localhost:3001/hotreload');
  
  ws.onmessage = (event) => {
    const { type, path, data } = JSON.parse(event.data);
    
    if (type === 'character') {
      loadedTemplates[data.data.id] = data;
      console.log(`Hot reloaded character: ${data.data.name}`);
    }
    
    if (type === 'animation') {
      animations[data.data.name.toLowerCase()] = data;
      console.log(`Hot reloaded animation: ${data.data.name}`);
    }
  };
}
```

---

## Testing Scenarios

### Core Mechanics

1. **Torch Attraction Test:**
   - Place torch on ground
   - Spawn NPC at varying distances
   - Verify NPC only responds within attractionRadius
   - Verify closest NPC is selected when multiple in range

2. **Possession Transfer Test:**
   - Initiate possession
   - Verify player input controls host
   - Kill host via combat
   - Verify XP transfers to torch
   - Verify torch drops at death location
   - Verify new NPC can be attracted

3. **Combat Resolution Test:**
   - Attack entity with possessed host
   - Verify damage calculation (damage - armor)
   - Verify XP awarded on kill
   - Test attack arc detection accuracy

4. **Spawning System Test:**
   - Move torch toward spawn point
   - Verify entities spawn within radius
   - Move torch away beyond despawnRadius
   - Verify entities despawn (unless possessed)
   - Verify budget limits respected

### Edge Cases

1. **No NPCs Available:**
   - All NPCs killed, torch on ground
   - Game should wait indefinitely (or spawn emergency NPC)
   
2. **Multiple Simultaneous Attractions:**
   - Multiple NPCs equidistant from torch
   - Deterministic selection (e.g., lowest ID)

3. **Host Dies While Moving:**
   - Ensure torch position correctly captures death location
   - No velocity inheritance to torch

4. **Spawn Point Overlaps:**
   - Two spawn points with overlapping radii
   - Both should function independently

### Performance

1. **Entity Count Stress Test:**
   - 50+ entities active
   - Maintain 60fps
   - Culling for off-screen entities

2. **Dungeon Size Test:**
   - Generate 200x200 tile dungeon
   - Verify generation completes < 1 second
   - Verify pathfinding remains performant

---

## Accessibility

1. **Visual:**
   - High contrast mode option (brighter torch light)
   - Colorblind-friendly damage/health indicators
   - Adjustable UI scale

2. **Controls:**
   - Rebindable keys
   - Mouse-only mode (click to move)
   - Gamepad support

3. **Audio:**
   - Visual indicators for all sound cues
   - Subtitles for any voiced content

---

## Performance Goals

| Metric | Target |
|--------|--------|
| Frame Rate | 60 FPS stable |
| Initial Load | < 3 seconds |
| Dungeon Generation | < 500ms for 100x100 |
| Max Active Entities | 100 without frame drops |
| Memory Usage | < 200MB |
| Bundle Size | < 5MB (excluding assets) |

---

## Extended Features (Post-MVP)

1. **Torch Upgrades:** Spend XP on torch abilities (larger light, faster attraction, host stat bonuses)

2. **Host Memories:** Accumulated kills/actions affect host behavior when repossessed

3. **Boss Encounters:** Unique boss rooms with custom mechanics and special PuppetJSX characters

4. **Permadeath Mode:** Torch XP resets on certain conditions

5. **Multiplayer:** Multiple torches, cooperation or competition

6. **Sound Design:** Positional audio cues for off-screen threats

7. **Achievement System:** Track possession count, total kills, furthest explored

8. **PuppetJSX Extensions:**
   - Custom skeleton types (quadrupeds, flying creatures, slimes)
   - Extended animation frame count (8, 12, 16 frames)
   - IK (Inverse Kinematics) for procedural arm/weapon positioning
   - Particle attachments for magical effects on bones
   - Sprite atlas export for optimized loading

9. **Part Designer:** In-game tool to create new parts with:
   - Color picker for placeholder parts
   - Stat assignment sliders
   - Immediate preview on test entity

10. **Animation Blending:** Smooth transitions between animations using PuppetJSX interpolation

---

## Part Assets & Image Loading

### Sprite Requirements

All part sprites should follow these guidelines for PuppetJSX compatibility:

```
Part Sprite Specifications:
├── Format: PNG with transparency
├── Size: Match width/height in part definition (typically 16x16, 24x24, or 32x32)
├── Origin: Center of sprite (PuppetJSX draws from center)
├── Naming: Must match part ID (e.g., "head_goblin_01.png" for id "head_goblin_01")
└── Organization: /assets/sprites/parts/{category}/{part_id}.png
```

### Preloading Part Images

```javascript
// lib/assetLoader.js
async function preloadPartImages(partsLibrary) {
  const loadPromises = partsLibrary.map(async (part) => {
    if (!part.imageUrl) {
      // No image, will use color fallback
      return { id: part.id, image: null, ...part };
    }
    
    return new Promise((resolve, reject) => {
      const img = new Image();
      img.onload = () => resolve({ 
        id: part.id, 
        image: img,
        width: part.width,
        height: part.height,
        offset: part.offset,
        color: part.color,
        zIndexModifier: part.zIndexModifier
      });
      img.onerror = () => {
        console.warn(`Failed to load image for part: ${part.id}`);
        resolve({ id: part.id, image: null, ...part }); // Fallback to color
      };
      img.src = part.imageUrl;
    });
  });
  
  const loaded = await Promise.all(loadPromises);
  return Object.fromEntries(loaded.map(p => [p.id, p]));
}

// Usage
const loadedPartImages = await preloadPartImages(partsLibrary);

// Pass to PuppetCharacter instances
const puppet = new PuppetCharacter(
  characterJSON,
  skeleton,
  animationJSON,
  loadedPartImages  // { partId: { image, width, height, offset, color, zIndexModifier } }
);
```

### Default Parts Library (Minimal Set)

```javascript
// data/parts/index.js - Aggregates all part definitions
const defaultParts = [
  // HEADS
  { id: 'head_basic', name: 'Basic Head', category: 'head', width: 24, height: 24, color: '#FFD4B8', offset: { x: 0, y: 0 }, zIndexModifier: 0, stats: { health: 0 }, faction: 'ANY' },
  { id: 'head_goblin_01', name: 'Goblin Head', category: 'head', width: 28, height: 28, color: '#7CB342', offset: { x: 0, y: 0 }, zIndexModifier: 0, stats: { health: 5 }, faction: 'MONSTER' },
  { id: 'head_skeleton_01', name: 'Skull', category: 'head', width: 24, height: 26, color: '#E0E0E0', offset: { x: 0, y: 0 }, zIndexModifier: 0, stats: { health: 0, armor: 2 }, faction: 'MONSTER' },
  { id: 'head_human_01', name: 'Human Head', category: 'head', width: 24, height: 24, color: '#FFD4B8', offset: { x: 0, y: 0 }, zIndexModifier: 0, stats: { health: 0 }, faction: 'NPC' },
  
  // TORSOS
  { id: 'torso_basic', name: 'Basic Torso', category: 'torso', width: 32, height: 40, color: '#5C6BC0', offset: { x: 0, y: 0 }, zIndexModifier: 0, stats: { health: 10 }, faction: 'ANY' },
  { id: 'torso_goblin_01', name: 'Goblin Torso', category: 'torso', width: 30, height: 36, color: '#558B2F', offset: { x: 0, y: 0 }, zIndexModifier: 0, stats: { health: 8, speed: 1.1 }, faction: 'MONSTER' },
  { id: 'torso_armor', name: 'Plate Armor', category: 'torso', width: 36, height: 44, color: '#78909C', offset: { x: 0, y: 0 }, zIndexModifier: 0, stats: { health: 20, armor: 5, speed: 0.85 }, faction: 'ANY' },
  
  // ARMS
  { id: 'arm_basic', name: 'Basic Arm', category: 'arm', width: 10, height: 20, color: '#FFD4B8', offset: { x: 0, y: 0 }, zIndexModifier: 0, stats: {}, faction: 'ANY' },
  { id: 'arm_goblin_01', name: 'Goblin Arm', category: 'arm', width: 10, height: 18, color: '#7CB342', offset: { x: 0, y: 0 }, zIndexModifier: 0, stats: { damage: 1 }, faction: 'MONSTER' },
  
  // HANDS
  { id: 'hand_basic', name: 'Basic Hand', category: 'hand', width: 10, height: 10, color: '#FFD4B8', offset: { x: 0, y: 0 }, zIndexModifier: 0, stats: {}, faction: 'ANY' },
  { id: 'hand_claw_01', name: 'Claw', category: 'hand', width: 12, height: 14, color: '#5D4037', offset: { x: 0, y: 0 }, zIndexModifier: 0, stats: { damage: 3 }, faction: 'MONSTER' },
  { id: 'hand_glove', name: 'Gauntlet', category: 'hand', width: 12, height: 12, color: '#78909C', offset: { x: 0, y: 0 }, zIndexModifier: 0, stats: { armor: 1 }, faction: 'ANY' },
  
  // LEGS
  { id: 'leg_basic', name: 'Basic Leg', category: 'leg', width: 12, height: 24, color: '#5C6BC0', offset: { x: 0, y: 0 }, zIndexModifier: 0, stats: {}, faction: 'ANY' },
  { id: 'leg_goblin_01', name: 'Goblin Leg', category: 'leg', width: 10, height: 22, color: '#558B2F', offset: { x: 0, y: 0 }, zIndexModifier: 0, stats: { speed: 1.05 }, faction: 'MONSTER' },
  { id: 'leg_armored', name: 'Armored Leg', category: 'leg', width: 14, height: 26, color: '#78909C', offset: { x: 0, y: 0 }, zIndexModifier: 0, stats: { armor: 1, speed: 0.95 }, faction: 'ANY' },
  
  // FEET
  { id: 'foot_basic', name: 'Basic Foot', category: 'foot', width: 14, height: 8, color: '#5D4037', offset: { x: 0, y: 0 }, zIndexModifier: 0, stats: {}, faction: 'ANY' },
  { id: 'foot_boot', name: 'Steel Boot', category: 'foot', width: 16, height: 10, color: '#78909C', offset: { x: 0, y: 0 }, zIndexModifier: 0, stats: { armor: 1 }, faction: 'ANY' },
  { id: 'foot_claw_01', name: 'Clawed Foot', category: 'foot', width: 14, height: 10, color: '#5D4037', offset: { x: 0, y: 0 }, zIndexModifier: 0, stats: { speed: 1.02 }, faction: 'MONSTER' },
  
  // WEAPONS (attach to hand_right)
  { id: 'weapon_sword', name: 'Iron Sword', category: 'weapon', width: 10, height: 40, color: '#B0BEC5', offset: { x: 6, y: -12 }, zIndexModifier: 2, stats: { damage: 8 }, faction: 'ANY' },
  { id: 'weapon_club', name: 'Wooden Club', category: 'weapon', width: 14, height: 36, color: '#6D4C41', offset: { x: 4, y: -10 }, zIndexModifier: 2, stats: { damage: 5 }, faction: 'ANY' },
  { id: 'weapon_staff', name: 'Magic Staff', category: 'weapon', width: 8, height: 52, color: '#7B1FA2', offset: { x: 4, y: -18 }, zIndexModifier: 2, stats: { damage: 3 }, faction: 'ANY' },
  { id: 'weapon_dagger', name: 'Bone Dagger', category: 'weapon', width: 6, height: 24, color: '#ECEFF1', offset: { x: 4, y: -6 }, zIndexModifier: 2, stats: { damage: 4, speed: 1.1 }, faction: 'ANY' },
  
  // ACCESSORIES (attach to hand_left)
  { id: 'shield', name: 'Round Shield', category: 'accessory', width: 24, height: 28, color: '#795548', offset: { x: -8, y: -4 }, zIndexModifier: 3, stats: { armor: 3 }, faction: 'ANY' },
  { id: 'torch_held', name: 'Torch (Held)', category: 'accessory', width: 8, height: 24, color: '#FF6F00', offset: { x: 0, y: -8 }, zIndexModifier: 4, stats: {}, faction: 'ANY' }
];

export default defaultParts;
```

---

*This document serves as the complete specification for R-Torch. The implementation integrates **PuppetJSX** for all character rendering and animation, providing a robust skeletal system with bone hierarchy, part assignment, keyframe animation, and interpolation. An implementation generated from this TINS README should produce a fully playable proof-of-concept demonstrating the torch possession mechanic, procedural dungeons, modular character system (via PuppetJSX), and level editor overlay functionality.*

**Key Integration Points:**
- PuppetJSX `defaultSkeleton` used for all humanoid entities
- PuppetJSX `defaultParts` extended with R-Torch stat system
- PuppetJSX `PuppetCharacter` runtime class for animation playback
- PuppetJSX Editor accessible for custom character/animation creation
- Assembly ID system serializes PuppetJSX character data for spawning
