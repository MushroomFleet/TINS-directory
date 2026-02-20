# Dynasty-Z-Turbo-2: Full Implementation Plan

## Description

Dynasty-Z-Turbo-2 is a third-person action game combining inverted bullet-hell deflection mechanics with massive-scale open-world warfare. The game merges three proven Zerobytes position-as-seed systems into a unified experience where players fight Dynasty Warriors-scale armies across procedurally generated infinite terrain, using the core deflection loop from Phase 1 enhanced with Toriyama character design and Kojima narrative architecture from Phase 2.

**Core Fantasy:** An unstoppable guardian who returns disorder to order through skillful deflection, fighting across an endless world where every location, enemy, and encounter is deterministically generated from coordinates alone.

**Technical Foundation:** All systems use position-as-seed (Zerobytes) methodology ensuring O(1) access, deterministic reproducibility, and zero storage overhead for infinite content.

---

## Reference Sources

The following source files provide working implementations to be integrated. Reference these by name during implementation:

| File | Purpose | Key Exports |
|------|---------|-------------|
| `3DLayeredTerrainV3.jsx` | Endless terrain generation | `TerrainDatabase`, `MainThreadGenerator`, `LOD_LEVELS`, noise functions |
| `PuppetJSX_Zerobytes.jsx` | NPC/RPG systems, dungeon layouts | `generateCharacterStats`, `generateRoom`, `coordsToSeed`, `coherentValue` |
| `ZB-3DCombatLayerV3.jsx` | Massive-scale combat | `SpatialGrid`, `InstancedArmyRenderer`, `generateArmy`, `resolveBattleRound` |

---

## Functionality

### 1. Unified World System

#### 1.1 World Coordinate Architecture

```javascript
const WORLD_CONFIG = {
  CHUNK_SIZE: 64,           // Terrain chunk size in units
  REGION_SIZE: 8,           // Chunks per region (8x8 = 64 chunks)
  SECTOR_SIZE: 512,         // World units per combat sector
  
  // Seed hierarchy for determinism
  SEED_LAYERS: {
    WORLD: 0x57524C44,      // Base world seed
    TERRAIN: 0x5452524E,    // Terrain generation
    SPAWNS: 0x5350574E,     // Enemy spawn locations
    NPCS: 0x4E504353,       // NPC/character generation
    LOOT: 0x4C4F4F54,       // Loot placement
    EVENTS: 0x45564E54      // World events/encounters
  }
};

// Unified coordinate-to-seed function
const worldSeed = (x, z, layer, baseSeed) => {
  return positionHash(x, z, layer, baseSeed ^ WORLD_CONFIG.SEED_LAYERS[layer]);
};
```

#### 1.2 Terrain Integration

Integrate `3DLayeredTerrainV3.jsx` terrain system:

```javascript
// Terrain chunk request flow
const getTerrainChunk = async (chunkX, chunkZ, cameraPos, worldSeed) => {
  const lod = getLODLevel(chunkX, chunkZ, 
    Math.floor(cameraPos.x / CHUNK_SIZE),
    Math.floor(cameraPos.z / CHUNK_SIZE)
  );
  
  // Check IndexedDB cache first
  const cached = await terrainDB.getChunk(worldSeed, chunkX, chunkZ, lod.level);
  if (cached) return deserializeChunk(cached);
  
  // Generate using MainThreadGenerator
  const chunk = await generator.generateChunk(chunkX, chunkZ, {
    resolution: lod.resolution,
    worldSeed,
    useBiomes: true
  });
  
  // Cache for future visits
  await terrainDB.saveChunk(worldSeed, chunkX, chunkZ, lod.level, chunk);
  return chunk;
};
```

#### 1.3 Region-Based Content Seeding

```javascript
// Region determines difficulty, biome blend, and encounter density
const getRegionProperties = (worldX, worldZ, worldSeed) => {
  const regionX = Math.floor(worldX / (CHUNK_SIZE * REGION_SIZE));
  const regionZ = Math.floor(worldZ / (CHUNK_SIZE * REGION_SIZE));
  
  // Coherent noise for smooth regional variation
  const difficulty = coherentValue(regionX * 0.1, regionZ * 0.1, worldSeed + 1000);
  const hostility = coherentValue(regionX * 0.15, regionZ * 0.15, worldSeed + 2000);
  
  return {
    baseDifficulty: 1.0 + (difficulty + 1) * 2.5,  // 1.0 to 6.0
    enemyDensity: 0.3 + (hostility + 1) * 0.35,    // 0.3 to 1.0
    eliteChance: Math.max(0, difficulty * 0.15),
    bossThreshold: difficulty > 0.7
  };
};
```

### 2. Enemy Generation System

#### 2.1 Spawn Point Generation

Spawn points are deterministic based on world coordinates:

```javascript
// From PuppetJSX: Room-based spawn locations adapted for open world
const generateSpawnPoint = (worldX, worldZ, worldSeed) => {
  const spawnSeed = coordsToSeed(
    Math.floor(worldX / 10), 
    Math.floor(worldZ / 10), 
    0, 
    worldSeed + SEED_LAYERS.SPAWNS
  );
  
  const rng = deriveValuesFromSeed(spawnSeed, 10);
  const region = getRegionProperties(worldX, worldZ, worldSeed);
  
  // Determine if spawn exists at this location
  if (rng[0] > region.enemyDensity) return null;
  
  // Enemy type weighted by regional difficulty
  const typeWeights = getEnemyTypeWeights(region.baseDifficulty);
  const enemyType = weightedSelect(ENEMY_TYPES, typeWeights, rng[1]);
  
  return {
    position: { x: worldX, z: worldZ },
    enemyType,
    count: 1 + Math.floor(rng[2] * (3 + region.baseDifficulty)),
    elite: rng[3] < region.eliteChance,
    seed: spawnSeed
  };
};
```

#### 2.2 Enemy Types (Phase 2 Enhanced)

```javascript
const ENEMY_TYPES = {
  PLINKER: {
    id: 'plinker',
    stats: { health: 30, xpValue: 10, fireRate: 2000 },
    projectileType: 'standard',
    visual: {
      shape: 'bouncy-sphere',
      size: 0.8,
      color: '#FF7744',
      faceExpression: 'nervous-enthusiasm'
    },
    behavior: {
      movement: 'shuffle-approach',
      attackStyle: 'direct-aim',
      groupBehavior: 'glance-at-each-other'
    }
  },
  
  SCATTERLING: {
    id: 'scatterling',
    stats: { health: 20, xpValue: 15, fireRate: 1500 },
    projectileType: 'burst',
    visual: {
      shape: 'tri-body-splits',
      size: 0.5,
      color: '#FFAA44',
      faceExpression: 'panicked-scanning'
    },
    behavior: {
      movement: 'strafe-split',
      attackStyle: 'burst-fire',
      splitBehavior: 'separates-while-moving'
    }
  },
  
  NEEDLER: {
    id: 'needler',
    stats: { health: 15, xpValue: 20, fireRate: 4000 },
    projectileType: 'sniper',
    visual: {
      shape: 'thin-vertical',
      size: { height: 1.5, width: 0.15 },
      color: '#8844AA',
      faceExpression: 'unblinking-track'
    },
    behavior: {
      movement: 'maintain-distance',
      attackStyle: 'precision-charged',
      tracking: 'smooth-rotation'
    }
  },
  
  KERNEL: {
    id: 'kernel',
    stats: { health: 80, xpValue: 25, fireRate: 3000 },
    projectileType: 'heavy',
    visual: {
      shape: 'dense-sphere',
      size: 0.5,  // SMALLER than Plinker (villain inversion)
      color: '#220808',
      faceExpression: 'unblinking-void'
    },
    behavior: {
      movement: 'stationary',
      attackStyle: 'slow-massive',
      presence: 'perfect-stillness'
    }
  }
};
```

#### 2.3 Army Generation for Large Battles

Integrate `ZB-3DCombatLayerV3.jsx` army generation:

```javascript
// Generate enemy army at world position
const generateEnemyArmy = (centerX, centerZ, worldSeed) => {
  const region = getRegionProperties(centerX, centerZ, worldSeed);
  const armySeed = worldSeed(centerX, centerZ, 'SPAWNS', worldSeed);
  
  // Scale army size by region difficulty
  const baseCount = 20 + Math.floor(region.baseDifficulty * 15);
  const spacing = 2.0 - region.enemyDensity * 0.5;
  
  // Use combat layer's generateArmy with terrain integration
  const army = generateArmy(centerX, centerZ, baseCount, spacing, armySeed, 1, 0);
  
  // Assign enemy types based on position
  army.forEach((unit, idx) => {
    const typeRng = hashToFloat(subHash(armySeed, idx * 100));
    const typeWeights = getEnemyTypeWeights(region.baseDifficulty);
    unit.enemyType = weightedSelect(Object.values(ENEMY_TYPES), typeWeights, typeRng);
    
    // Override base stats with enemy type stats
    Object.assign(unit.stats, unit.enemyType.stats);
    
    // Set Y position from terrain
    unit.simPos.y = getTerrainHeight(unit.simPos.x, unit.simPos.z, worldSeed);
    unit.renderPos.y = unit.simPos.y;
  });
  
  return army;
};
```

### 3. Player System

#### 3.1 Player State

```javascript
const createPlayer = (startX, startZ, worldSeed) => ({
  position: new THREE.Vector3(startX, getTerrainHeight(startX, startZ, worldSeed), startZ),
  velocity: new THREE.Vector3(),
  rotation: 0,
  
  // Combat state
  health: { current: 100, max: 100 },
  isInvulnerable: false,
  isDashing: false,
  isParrying: false,
  parryStartTime: 0,
  lastSlashTime: 0,
  lastDashTime: 0,
  
  // Progression
  level: 1,
  currentXP: 0,
  xpToNextLevel: 100,
  upgrades: new Map(),
  
  // Stats (modified by upgrades)
  stats: {
    moveSpeed: 8,
    sprintSpeed: 14,
    deflectRadius: 2.5,
    parryWindow: 200,
    slashDamage: 25,
    slashCooldown: 400,
    dashDistance: 6,
    dashDuration: 150,
    dashCooldown: 500
  },
  
  // Consequence tracking (hidden)
  tracking: {
    totalDeflections: 0,
    parryDeflections: 0,
    passiveDeflections: 0,
    meleeKills: 0,
    deflectionKills: 0,
    positionSamples: [],
    nearDeathRecoveries: 0
  }
});
```

#### 3.2 Movement System

```javascript
const updatePlayerMovement = (player, input, delta, worldSeed) => {
  // Camera-relative movement direction
  const moveDir = new THREE.Vector3();
  if (input.forward) moveDir.z -= 1;
  if (input.backward) moveDir.z += 1;
  if (input.left) moveDir.x -= 1;
  if (input.right) moveDir.x += 1;
  
  if (moveDir.length() > 0) {
    moveDir.normalize();
    moveDir.applyAxisAngle(new THREE.Vector3(0, 1, 0), player.cameraRotation);
    
    const speed = input.sprint ? player.stats.sprintSpeed : player.stats.moveSpeed;
    player.velocity.x = moveDir.x * speed;
    player.velocity.z = moveDir.z * speed;
    player.rotation = Math.atan2(moveDir.x, moveDir.z);
  } else {
    player.velocity.x *= 0.8;
    player.velocity.z *= 0.8;
  }
  
  // Apply movement
  player.position.x += player.velocity.x * delta;
  player.position.z += player.velocity.z * delta;
  
  // Ground follow
  const targetY = getTerrainHeight(player.position.x, player.position.z, worldSeed);
  player.position.y = THREE.MathUtils.lerp(player.position.y, targetY + 1, 0.2);
};
```

#### 3.3 Dash System

```javascript
const performDash = (player, input, currentTime) => {
  if (currentTime - player.lastDashTime < player.stats.dashCooldown) return false;
  if (player.isDashing) return false;
  
  player.isDashing = true;
  player.isInvulnerable = true;
  player.lastDashTime = currentTime;
  
  // Dash direction from input or backward from camera
  let dashDir = new THREE.Vector3();
  if (input.moveDir.length() > 0) {
    dashDir.copy(input.moveDir).normalize();
  } else {
    dashDir.set(0, 0, 1).applyAxisAngle(new THREE.Vector3(0, 1, 0), player.cameraRotation);
  }
  
  player.dashVelocity = dashDir.multiplyScalar(
    player.stats.dashDistance / (player.stats.dashDuration / 1000)
  );
  
  // End dash after duration
  setTimeout(() => {
    player.isDashing = false;
    player.isInvulnerable = false;
  }, player.stats.dashDuration);
  
  return true;
};
```

### 4. Deflection System

#### 4.1 Core Deflection Logic

```javascript
const DEFLECT_TYPES = {
  PASSIVE: 'passive',   // Auto-deflect, random direction
  PARRY: 'parry',       // Timed, targeted, 2x damage
  HELD: 'held'          // Continuous, all targeted
};

const checkDeflection = (projectile, player, currentTime) => {
  const dist = projectile.position.distanceTo(player.position);
  if (dist > player.stats.deflectRadius) return null;
  
  // Determine deflection type
  let deflectType = DEFLECT_TYPES.PASSIVE;
  
  if (player.isHoldingDeflect) {
    deflectType = DEFLECT_TYPES.HELD;
  } else if (player.isParrying) {
    const parryElapsed = currentTime - player.parryStartTime;
    if (parryElapsed < player.stats.parryWindow) {
      deflectType = DEFLECT_TYPES.PARRY;
    }
  }
  
  return deflectType;
};

const deflectProjectile = (projectile, player, deflectType, enemies, worldSeed) => {
  // Calculate deflection direction
  let newDirection;
  
  switch (deflectType) {
    case DEFLECT_TYPES.PASSIVE:
      // Random direction using projectile position as seed
      const seed = positionHash(
        projectile.position.x,
        projectile.position.y,
        projectile.position.z,
        worldSeed
      );
      const angle = hashToFloat(seed) * Math.PI * 2;
      const elevation = hashToFloat(subHash(seed, 1)) * 0.5 - 0.25;
      newDirection = new THREE.Vector3(
        Math.cos(angle) * Math.cos(elevation),
        Math.sin(elevation),
        Math.sin(angle) * Math.cos(elevation)
      );
      break;
      
    case DEFLECT_TYPES.PARRY:
    case DEFLECT_TYPES.HELD:
      // Target nearest enemy
      const target = findNearestEnemy(player.position, enemies, deflectType === DEFLECT_TYPES.PARRY);
      if (target) {
        newDirection = target.position.clone().sub(player.position).normalize();
      } else {
        // Fallback to forward direction
        newDirection = new THREE.Vector3(0, 0, -1)
          .applyAxisAngle(new THREE.Vector3(0, 1, 0), player.rotation);
      }
      break;
  }
  
  // Update projectile
  projectile.velocity = newDirection.multiplyScalar(projectile.speed * 1.2);
  projectile.isDeflected = true;
  projectile.isParryDeflected = deflectType === DEFLECT_TYPES.PARRY;
  projectile.ownerId = null;  // Now belongs to player
  
  // Track for consequences
  player.tracking.totalDeflections++;
  if (deflectType === DEFLECT_TYPES.PARRY) {
    player.tracking.parryDeflections++;
  } else {
    player.tracking.passiveDeflections++;
  }
  
  return { type: deflectType, direction: newDirection };
};
```

#### 4.2 Deflection VFX

```javascript
const createDeflectionVFX = (position, deflectType, direction) => {
  const vfx = {
    position: position.clone(),
    type: deflectType,
    startTime: performance.now(),
    particles: []
  };
  
  switch (deflectType) {
    case DEFLECT_TYPES.PASSIVE:
      // Water ripple effect (wu wei)
      for (let i = 0; i < 12; i++) {
        const angle = (i / 12) * Math.PI * 2;
        vfx.particles.push({
          offset: new THREE.Vector3(Math.cos(angle), 0, Math.sin(angle)),
          color: new THREE.Color('#88CCFF'),
          opacity: 0.6,
          scale: 0.2,
          velocity: 3
        });
      }
      vfx.duration = 300;
      vfx.flash = { color: '#AADDFF', intensity: 0.3 };
      break;
      
    case DEFLECT_TYPES.PARRY:
      // Snake strike trail (cultivated de)
      for (let i = 0; i < 24; i++) {
        const t = i / 24;
        vfx.particles.push({
          offset: direction.clone().multiplyScalar(t * 2),
          color: new THREE.Color('#FFD700'),
          opacity: 1.0 - t * 0.5,
          scale: 0.15 * (1 - t * 0.5),
          velocity: 0
        });
      }
      vfx.duration = 400;
      vfx.flash = { color: '#FFD700', intensity: 0.6, screenEdge: true };
      vfx.trail = { color: '#FFAA00', width: 2 };
      break;
      
    case DEFLECT_TYPES.HELD:
      // Calm focus aura
      vfx.aura = {
        color: new THREE.Color('#44AAFF'),
        radius: 2.5,
        opacity: 0.4,
        pulseRate: 2000
      };
      vfx.duration = 100;  // Continuous while held
      break;
  }
  
  return vfx;
};
```

### 5. Combat Integration

#### 5.1 Battle Sector Management

```javascript
class BattleSectorManager {
  constructor(worldSeed) {
    this.worldSeed = worldSeed;
    this.activeSectors = new Map();
    this.blueGrid = new SpatialGrid(5);
    this.redGrid = new SpatialGrid(5);
    this.renderer = null;
  }
  
  initRenderer(scene, maxUnits = 2000) {
    this.renderer = new InstancedArmyRenderer(scene, maxUnits);
  }
  
  getSector(worldX, worldZ) {
    const sectorX = Math.floor(worldX / WORLD_CONFIG.SECTOR_SIZE);
    const sectorZ = Math.floor(worldZ / WORLD_CONFIG.SECTOR_SIZE);
    const key = `${sectorX},${sectorZ}`;
    
    if (!this.activeSectors.has(key)) {
      this.activeSectors.set(key, this.generateSector(sectorX, sectorZ));
    }
    return this.activeSectors.get(key);
  }
  
  generateSector(sectorX, sectorZ) {
    const centerX = (sectorX + 0.5) * WORLD_CONFIG.SECTOR_SIZE;
    const centerZ = (sectorZ + 0.5) * WORLD_CONFIG.SECTOR_SIZE;
    const region = getRegionProperties(centerX, centerZ, this.worldSeed);
    
    // Generate enemy army for this sector
    const enemies = generateEnemyArmy(centerX, centerZ, this.worldSeed);
    
    return {
      id: `sector_${sectorX}_${sectorZ}`,
      bounds: {
        minX: sectorX * WORLD_CONFIG.SECTOR_SIZE,
        maxX: (sectorX + 1) * WORLD_CONFIG.SECTOR_SIZE,
        minZ: sectorZ * WORLD_CONFIG.SECTOR_SIZE,
        maxZ: (sectorZ + 1) * WORLD_CONFIG.SECTOR_SIZE
      },
      enemies,
      region,
      projectiles: [],
      round: 0
    };
  }
  
  update(playerPos, delta, worldSeed) {
    const currentSector = this.getSector(playerPos.x, playerPos.z);
    
    // Update enemy AI and combat
    this.updateEnemyAI(currentSector, playerPos, delta);
    this.updateProjectiles(currentSector, delta);
    
    // Update spatial grids
    this.redGrid.insertAll(currentSector.enemies.filter(e => e.alive));
    
    // Update instanced rendering
    if (this.renderer) {
      this.renderer.updateAll([], currentSector.enemies);
    }
  }
  
  updateEnemyAI(sector, playerPos, delta) {
    sector.enemies.forEach(enemy => {
      if (!enemy.alive) return;
      
      const toPlayer = new THREE.Vector3()
        .subVectors(playerPos, enemy.simPos);
      const dist = toPlayer.length();
      
      // State transitions
      if (dist < 25) {
        enemy.state = 'attacking';
      } else if (dist < 30) {
        enemy.state = 'aware';
      } else {
        enemy.state = 'idle';
      }
      
      // Behavior based on enemy type and state
      this.executeEnemyBehavior(enemy, playerPos, dist, delta, sector);
    });
  }
  
  executeEnemyBehavior(enemy, playerPos, dist, delta, sector) {
    const type = enemy.enemyType;
    
    switch (enemy.state) {
      case 'attacking':
        // Check fire cooldown
        const now = performance.now();
        if (now - enemy.lastFireTime > type.stats.fireRate) {
          this.fireProjectile(enemy, playerPos, sector);
          enemy.lastFireTime = now;
        }
        
        // Movement based on type
        if (type.behavior.movement !== 'stationary') {
          this.moveEnemy(enemy, playerPos, type.behavior.movement, delta);
        }
        break;
        
      case 'aware':
        // Approach player
        if (type.behavior.movement !== 'stationary') {
          this.moveEnemy(enemy, playerPos, 'approach', delta);
        }
        break;
    }
  }
  
  fireProjectile(enemy, playerPos, sector) {
    const type = enemy.enemyType;
    const projectileConfig = PROJECTILE_TYPES[type.projectileType];
    
    // Direction with slight leading
    const toPlayer = new THREE.Vector3().subVectors(playerPos, enemy.simPos).normalize();
    
    // Burst fire for scatterlings
    const burstCount = type.attackStyle === 'burst-fire' ? 3 : 1;
    
    for (let i = 0; i < burstCount; i++) {
      const spread = type.attackStyle === 'burst-fire' ? (i - 1) * 0.1 : 0;
      const dir = toPlayer.clone().applyAxisAngle(
        new THREE.Vector3(0, 1, 0), spread
      );
      
      sector.projectiles.push({
        id: `proj_${enemy.id}_${sector.round}_${i}`,
        position: enemy.simPos.clone(),
        velocity: dir.multiplyScalar(projectileConfig.speed),
        damage: projectileConfig.damage,
        size: projectileConfig.size,
        color: projectileConfig.color,
        ownerId: enemy.id,
        isDeflected: false,
        spawnTime: performance.now(),
        lifetime: projectileConfig.lifetime * 1000
      });
    }
  }
}
```

### 6. Progression System

#### 6.1 XP and Leveling

```javascript
const LEVELING = {
  xpToLevel: (level) => 100 * level,
  maxLevel: 20
};

const grantXP = (player, amount) => {
  player.currentXP += amount;
  
  while (player.currentXP >= player.xpToNextLevel && player.level < LEVELING.maxLevel) {
    player.currentXP -= player.xpToNextLevel;
    player.level++;
    player.xpToNextLevel = LEVELING.xpToLevel(player.level + 1);
    return { leveledUp: true, newLevel: player.level };
  }
  
  return { leveledUp: false };
};
```

#### 6.2 Upgrade System (Phase 2 Naming)

```javascript
const UPGRADES = [
  {
    id: 'serpents_precision',
    name: "Serpent's Precision",
    description: 'Expands the window for perfect deflection',
    effect: (stats) => ({ ...stats, parryWindow: stats.parryWindow * 1.25 }),
    maxStacks: 5
  },
  {
    id: 'turtles_endurance',
    name: "Turtle's Endurance",
    description: 'Fortifies your life force',
    effect: (stats, player) => {
      player.health.max += 20;
      player.health.current += 20;
      return stats;
    },
    maxStacks: 10
  },
  {
    id: 'mandates_clarity',
    name: "Mandate's Clarity",
    description: 'Extends the zone of deflection',
    effect: (stats) => ({ ...stats, deflectRadius: stats.deflectRadius + 0.3 }),
    maxStacks: 5
  },
  {
    id: 'heavens_swiftness',
    name: "Heaven's Swiftness",
    description: 'Quickens your movement',
    effect: (stats) => ({
      ...stats,
      moveSpeed: stats.moveSpeed * 1.1,
      sprintSpeed: stats.sprintSpeed * 1.1
    }),
    maxStacks: 5
  },
  {
    id: 'returned_force',
    name: 'Returned Force',
    description: 'Deflected chaos seeks additional targets',
    effect: (stats) => ({ ...stats, ricochetCount: (stats.ricochetCount || 0) + 1 }),
    maxStacks: 3
  },
  {
    id: 'voids_echo',
    name: "Void's Echo",
    description: 'Perfect deflection releases burst of energy',
    effect: (stats) => ({ ...stats, parryExplosionRadius: (stats.parryExplosionRadius || 0) + 2 }),
    maxStacks: 3
  }
];

const selectRandomUpgrades = (player, count = 3) => {
  // Filter available upgrades
  const available = UPGRADES.filter(u => {
    const stacks = player.upgrades.get(u.id) || 0;
    return stacks < u.maxStacks;
  });
  
  // Shuffle and take
  const shuffled = [...available].sort(() => Math.random() - 0.5);
  return shuffled.slice(0, count);
};
```

### 7. Consequence System (Hidden)

#### 7.1 Hidden Tracking

```javascript
const updateConsequenceTracking = (player, event) => {
  switch (event.type) {
    case 'deflection':
      // Already tracked in deflectProjectile
      break;
      
    case 'kill':
      if (event.source === 'melee') {
        player.tracking.meleeKills++;
      } else {
        player.tracking.deflectionKills++;
      }
      break;
      
    case 'position_sample':
      player.tracking.positionSamples.push({
        position: player.position.clone(),
        time: performance.now()
      });
      // Keep last 100 samples
      if (player.tracking.positionSamples.length > 100) {
        player.tracking.positionSamples.shift();
      }
      break;
      
    case 'near_death_recovery':
      if (player.health.current < player.health.max * 0.2) {
        player.tracking.nearDeathRecoveries++;
      }
      break;
  }
};
```

#### 7.2 Consequence Evaluation

```javascript
const evaluateConsequences = (player) => {
  const t = player.tracking;
  const consequences = [];
  
  // Parry ratio
  const parryRatio = t.totalDeflections > 0 
    ? t.parryDeflections / t.totalDeflections 
    : 0;
    
  if (parryRatio >= 0.9) {
    consequences.push({
      type: 'mirror_spawn',
      message: null  // Never reveal
    });
  }
  
  // Pure deflection
  if (t.meleeKills === 0 && t.deflectionKills > 50) {
    consequences.push({
      type: 'pure_deflection_ending',
      message: null
    });
  }
  
  // Heavy melee
  if (t.meleeKills > t.deflectionKills) {
    consequences.push({
      type: 'aggressive_spawns',
      effect: { spawnCloser: true, moreAggressive: true }
    });
  }
  
  // Ghost signature
  const ghostSignature = {
    parryStyle: parryRatio > 0.7 ? 'precision' : 'flow',
    combatStyle: t.meleeKills > t.deflectionKills * 0.5 ? 'aggressive' : 'reflective',
    endurance: t.nearDeathRecoveries
  };
  
  return { consequences, ghostSignature };
};
```

### 8. Pacing System

#### 8.1 Wave Management for Open World

```javascript
const createWaveManager = (worldSeed) => ({
  currentWave: 0,
  waveState: 'idle',  // idle, spawning, active, clearing, breathing
  breathStartTime: 0,
  
  BREATH_PHASES: {
    chaosClearing: { duration: 1000 },
    stillness: { duration: 3000 },
    centering: { optional: true },
    anticipation: { duration: 1500 }
  },
  
  startBreath() {
    this.waveState = 'clearing';
    this.breathStartTime = performance.now();
  },
  
  updateBreath(player, currentTime) {
    const elapsed = currentTime - this.breathStartTime;
    
    if (this.waveState === 'clearing' && elapsed > 1000) {
      this.waveState = 'stillness';
    } else if (this.waveState === 'stillness' && elapsed > 4000) {
      // Check for centering bonus
      const inCenter = player.position.length() < 5;
      if (inCenter && player.velocity.length() < 0.1) {
        grantXP(player, 10);  // Small de bonus
      }
      this.waveState = 'anticipation';
    } else if (this.waveState === 'anticipation' && elapsed > 5500) {
      this.waveState = 'spawning';
      this.currentWave++;
      return true;  // Ready for next wave
    }
    
    return false;
  }
});
```

---

## Technical Implementation

### React Component Structure

```javascript
// Main game component
const DynastyZTurbo = () => {
  const [gameState, setGameState] = useState('menu');
  const [player, setPlayer] = useState(null);
  const [worldSeed, setWorldSeed] = useState(42);
  
  // Refs for Three.js objects
  const sceneRef = useRef(null);
  const cameraRef = useRef(null);
  const rendererRef = useRef(null);
  const battleManagerRef = useRef(null);
  const terrainManagerRef = useRef(null);
  
  useEffect(() => {
    if (gameState === 'playing') {
      initGame();
      return () => cleanupGame();
    }
  }, [gameState]);
  
  const initGame = async () => {
    // Initialize Three.js scene
    sceneRef.current = new THREE.Scene();
    // ... camera, renderer setup
    
    // Initialize terrain system
    terrainManagerRef.current = new MainThreadGenerator(2);
    await terrainDB.initialize();
    
    // Initialize battle system
    battleManagerRef.current = new BattleSectorManager(worldSeed);
    battleManagerRef.current.initRenderer(sceneRef.current);
    
    // Create player
    setPlayer(createPlayer(0, 0, worldSeed));
    
    // Start game loop
    requestAnimationFrame(gameLoop);
  };
  
  const gameLoop = (timestamp) => {
    if (gameState !== 'playing') return;
    
    const delta = (timestamp - lastTimestamp) / 1000;
    lastTimestamp = timestamp;
    
    // Update systems
    updateInput();
    updatePlayerMovement(player, input, delta, worldSeed);
    updateDeflection(player, battleManagerRef.current);
    battleManagerRef.current.update(player.position, delta, worldSeed);
    updateTerrain(player.position);
    
    // Render
    rendererRef.current.render(sceneRef.current, cameraRef.current);
    
    requestAnimationFrame(gameLoop);
  };
  
  return (
    <div className="game-container">
      <canvas ref={canvasRef} />
      <HUD player={player} />
      {gameState === 'levelup' && (
        <LevelUpScreen 
          upgrades={selectRandomUpgrades(player)} 
          onSelect={handleUpgradeSelect}
        />
      )}
    </div>
  );
};
```

---

## Implementation Order

### Phase 1: Core Integration (Week 1-2)

1. **Project Setup**
   - Vite + React + R3F project structure
   - Import and integrate `3DLayeredTerrainV3.jsx`
   - Import and integrate `ZB-3DCombatLayerV3.jsx`
   - Import utility functions from `PuppetJSX_Zerobytes.jsx`

2. **Unified Seed System**
   - Implement `WORLD_CONFIG` with seed hierarchy
   - Create `worldSeed()` function for consistent hashing
   - Test determinism across terrain + combat + spawns

3. **Basic Player**
   - Movement with terrain following
   - Third-person camera
   - Basic collision bounds

### Phase 2: Combat Foundation (Week 3-4)

4. **Enemy Spawning**
   - Position-based spawn point generation
   - Region difficulty scaling
   - Enemy type selection

5. **Projectile System**
   - Enemy firing behavior
   - Projectile pooling
   - Basic collision detection

6. **Deflection Mechanics**
   - Passive deflection (auto)
   - Parry deflection (timed)
   - Held deflection (continuous)

### Phase 3: Systems Integration (Week 5-6)

7. **Army Rendering**
   - Integrate `InstancedArmyRenderer`
   - Per-enemy-type coloring
   - Health-based visual feedback

8. **Spatial Optimization**
   - Integrate `SpatialGrid` for combat
   - Sector-based enemy management
   - Culling for distant sectors

9. **Progression System**
   - XP from deflection kills
   - Level up system
   - Upgrade application

### Phase 4: Polish & Enhancement (Week 7-8)

10. **VFX System**
    - Deflection particles
    - Enemy death animations
    - Environmental effects

11. **Audio Integration**
    - Deflection sounds (passive vs parry)
    - Enemy type audio
    - Environmental ambiance

12. **Consequence Tracking**
    - Hidden stat tracking
    - Ghost signature generation
    - Consequence application

### Phase 5: Final Integration (Week 9-10)

13. **Pacing System**
    - Inter-wave breath
    - Level-up sanctuary
    - Post-death contemplation

14. **UI/HUD**
    - Health/XP display
    - Upgrade selection screen
    - End-run statistics

15. **Performance Optimization**
    - Profile and optimize
    - LOD tuning
    - Memory management

---

## Testing Scenarios

### Determinism Tests
1. Generate world at seed 12345, record terrain heights at 100 points
2. Restart, regenerate—heights must match exactly
3. Generate enemy army at (500, 500)—composition must match
4. Run 10 combat rounds without player input—damage must match

### Deflection Tests
5. Passive deflection produces random directions
6. Parry within window targets nearest enemy
7. Held deflection targets all projectiles
8. Deflected projectile damages enemies

### Performance Tests
9. 200+ enemies render at 60 FPS
10. Terrain LOD reduces triangles by 75% at distance
11. No frame drops during wave transitions

### Integration Tests
12. Terrain height matches at spawn points
13. Enemies path correctly on terrain
14. Player deflection radius scales with upgrades
15. XP grants correctly from enemy types

---

## Style Guide

### Color Palette
```
Player:          #00CCCC (Cyan)
Plinker:         #FF7744 (Warm Orange)
Scatterling:     #FFAA44 (Light Orange)
Needler:         #8844AA (Purple)
Kernel:          #220808 (Dark Red-Black)
Passive VFX:     #88CCFF (Soft Blue)
Parry VFX:       #FFD700 (Gold)
Terrain Water:   #1E3A5F
Terrain Grass:   #2D5A2E
```

### Typography
- HUD: System sans-serif, 16-18px
- Wave Counter: 24px bold
- Upgrade Names: 18px bold (celestial naming)
- Statistics: Monospace for numbers

---

*This TINS README provides complete specifications for implementing Dynasty-Z-Turbo-2, integrating three proven Zerobytes systems into a unified massive-scale action experience.*
