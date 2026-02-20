# Open World Driving Combat — Framework Implementation Plan

> **Target**: Claude Code autonomous implementation
> **Stack**: Three.js 0.122 (CDN) + Cannon-ES 0.20 (CDN) + Vite 6 + TypeScript 5.8
> **Zero external assets** — all geometry, audio, and terrain are procedural
> **Reference codebase**: `raycastvehicle-demo` (Moon Defenders)

---

## Table of Contents

1. [Project Scaffold](#1-project-scaffold)
2. [Core Framework — Demo.ts + utils.ts](#2-core-framework)
3. [Raycast Vehicle](#3-raycast-vehicle)
4. [Infinite Terrain System](#4-infinite-terrain-system)
5. [Procedural Noise & Hashing](#5-procedural-noise--hashing)
6. [Biome / Planet System](#6-biome--planet-system)
7. [Skybox Manager](#7-skybox-manager)
8. [Entity Manager — AI & Spawning](#8-entity-manager)
9. [Procedural Robot Generator](#9-procedural-robot-generator)
10. [Sector Manager — Room Generation](#10-sector-manager)
11. [POI / Collectible System](#11-poi--collectible-system)
12. [Weapons — VLMS Missiles](#12-weapons--vlms-missiles)
13. [Weapons — Point Defense Cannons](#13-weapons--point-defense-cannons)
14. [Weapons — Orbital Strike](#14-weapons--orbital-strike)
15. [Explosion System](#15-explosion-system)
16. [Buff / Progression System](#16-buff--progression-system)
17. [Base Building System](#17-base-building-system)
18. [Turret & Drone Defense](#18-turret--drone-defense)
19. [Spaceship & Planet Travel](#19-spaceship--planet-travel)
20. [Save System](#20-save-system)
21. [Procedural Audio](#21-procedural-audio)
22. [HUD & UI](#22-hud--ui)
23. [WebGPU Acceleration (Optional)](#23-webgpu-acceleration)
24. [Game Loop — Wiring Everything Together](#24-game-loop)
25. [Cheat / Debug System](#25-cheat--debug-system)

---

## 1. Project Scaffold

### Files to Create

```
project-root/
├── index.html
├── index.tsx            ← Main entry, game loop, all wiring
├── Demo.ts              ← Three.js + Cannon.js framework wrapper
├── utils.ts             ← Shape/body ↔ mesh conversion
├── style.css            ← Minimal UI styles
├── package.json
├── vite.config.ts
├── tsconfig.json
├── src/
│   ├── chunk-manager.ts
│   ├── entity-manager.ts
│   ├── sector-manager.ts
│   ├── poi-manager.ts
│   ├── moon-terrain.ts
│   ├── moon-zb.ts
│   ├── robot-generator.ts
│   ├── skybox-manager.ts
│   ├── biomes/
│   │   └── BiomeConfig.ts
│   ├── systems/
│   │   ├── BuffSystem.ts
│   │   ├── BuffUI.ts
│   │   ├── BuffPool.ts
│   │   ├── BaseBuilding.ts
│   │   ├── BaseManager.ts
│   │   ├── TurretSystem.ts
│   │   ├── DroneSystem.ts
│   │   ├── ExplosionSystem.ts
│   │   ├── Spaceship.ts
│   │   ├── PlanetTravelUI.ts
│   │   ├── CheatManager.ts
│   │   └── CheatMenuUI.ts
│   ├── config/
│   │   └── explosion-config.ts
│   ├── save-system/
│   │   ├── index.ts
│   │   ├── SaveManager.ts
│   │   ├── SaveSerializer.ts
│   │   ├── SaveStorage.ts
│   │   ├── SaveValidator.ts
│   │   ├── SaveUI.ts
│   │   └── SaveTypes.ts
│   ├── gpu/                    ← Optional WebGPU
│   │   ├── index.ts
│   │   ├── WebGPUContext.ts
│   │   ├── HeightfieldCompute.ts
│   │   ├── CraterCompute.ts
│   │   ├── GPUTelemetry.ts
│   │   └── shaders/
│   │       ├── heightfield.wgsl
│   │       └── craters.wgsl
│   └── types/
│       └── webgpu.d.ts
```

### package.json

```json
{
  "name": "open-world-driving-combat",
  "private": true,
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "@rollup/rollup-win32-x64-msvc": "^4.56.0"
  },
  "devDependencies": {
    "@types/node": "^22.14.0",
    "typescript": "~5.8.2",
    "vite": "^6.2.0"
  }
}
```

### vite.config.ts

```typescript
import path from 'path';
import { defineConfig } from 'vite';

export default defineConfig({
  base: './',
  server: { port: 3000, host: '0.0.0.0' },
  resolve: {
    alias: { '@': path.resolve(__dirname, '.') }
  },
  assetsInclude: ['**/*.wgsl'],
});
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "typeRoots": ["./node_modules/@types", "./src/types"],
    "skipLibCheck": true,
    "types": ["node"],
    "moduleResolution": "bundler",
    "isolatedModules": true,
    "moduleDetection": "force",
    "allowJs": true,
    "jsx": "react-jsx",
    "paths": { "@/*": ["./*"] },
    "allowImportingTsExtensions": true,
    "noEmit": true
  }
}
```

### style.css

```css
* { margin: 0; padding: 0; box-sizing: border-box; }
body { overflow: hidden; background: #000; font-family: 'Courier New', monospace; color: #0f0; }
canvas { display: block; }
```

---

## 2. Core Framework

### Demo.ts — Three.js + Cannon.js Wrapper

This is the engine shell. It manages the physics world, renderer, scene graph, and body↔visual synchronization.

**Key architecture:**

```typescript
// CDN imports (no npm packages for Three/Cannon)
// @ts-ignore
import * as CANNON from 'https://unpkg.com/cannon-es@0.20.0/dist/cannon-es.js';
// @ts-ignore
import * as THREE from 'https://unpkg.com/three@0.122.0/build/three.module.js';
import { bodyToMesh } from './utils.js';

export class Demo extends CANNON.EventTarget {
  bodies: any[] = [];    // Cannon.js bodies
  visuals: any[] = [];   // Three.js meshes — always same length as bodies[]
  world: any;
  scene: any;
  camera: any;
  renderer: any;
  settings: any;
  listeners: Record<string, Function[]> = {};

  constructor(options?: any) {
    super(options);
    this.settings = {
      stepFrequency: 60,
      quatNormalizeSkip: 2,
      quatNormalizeFast: true,
      gx: 0, gy: 0, gz: 0,
      iterations: 3,
      tolerance: 0.0001,
      maxSubSteps: 20,
      paused: false,
      ...options,
    };
    this.world = new CANNON.World();
    this.initThree();
    this.initGeometryCaches();
    this.animate();
    window.addEventListener('resize', this.resize);
  }
```

**Critical methods:**
- `addVisual(body)` — creates Three.js mesh from Cannon body, pushes to both arrays
- `removeVisual(body)` — removes from both arrays and scene
- `updateVisuals()` — syncs `bodies[i].interpolatedPosition` → `visuals[i].position` each frame
- `updatePhysics()` — calls `world.step(1/60, delta, maxSubSteps)`
- `animate()` — `requestAnimationFrame` loop: physics → visuals → render

**Renderer setup:**
```typescript
initThree() {
  this.camera = new THREE.PerspectiveCamera(24, window.innerWidth / window.innerHeight, 5, 2000);
  this.scene = new THREE.Scene();
  this.scene.fog = new THREE.Fog(0x222222, 1000, 2000);
  this.renderer = new THREE.WebGLRenderer({ antialias: true });
  this.renderer.setSize(window.innerWidth, window.innerHeight);
  this.renderer.shadowMap.enabled = true;
  this.renderer.shadowMap.type = THREE.PCFSoftShadowMap;
  document.body.appendChild(this.renderer.domElement);
  // Ambient + SpotLight + DirectionalLight
}
```

### utils.ts — Shape↔Geometry Conversion

Converts every Cannon.js shape type to Three.js geometry:

```typescript
export function shapeToGeometry(shape: any): any {
  switch (shape.type) {
    case CANNON.Shape.types.SPHERE:
      return new THREE.SphereGeometry(shape.radius, 8, 8);
    case CANNON.Shape.types.BOX:
      return new THREE.BoxGeometry(
        shape.halfExtents.x * 2,
        shape.halfExtents.y * 2,
        shape.halfExtents.z * 2
      );
    case CANNON.Shape.types.CYLINDER:
      return new THREE.CylinderGeometry(shape.radiusTop, shape.radiusBottom, shape.height, shape.numSegments);
    case CANNON.Shape.types.HEIGHTFIELD:
      // Build BufferGeometry from heightfield triangle pillars
      // Iterate xi, yi, getConvexTrianglePillar, push vertices
    case CANNON.Shape.types.CONVEXPOLYHEDRON:
      // Fan-triangulate faces, BufferAttribute position + indices
    case CANNON.Shape.types.TRIMESH:
      // getTriangleVertices loop
  }
}

export function bodyToMesh(body: any, material: any): any {
  const group = new THREE.Group();
  group.position.copy(body.position);
  group.quaternion.copy(body.quaternion);
  body.shapes.forEach((shape, i) => {
    const mesh = new THREE.Mesh(shapeToGeometry(shape), material);
    mesh.position.copy(body.shapeOffsets[i]);
    mesh.quaternion.copy(body.shapeOrientations[i]);
    group.add(mesh);
  });
  return group;
}
```

---

## 3. Raycast Vehicle

The vehicle is the central gameplay object. Built on `CANNON.RaycastVehicle`.

### Physics Configuration

```typescript
// Chassis
const chassisShape = new CANNON.Box(new CANNON.Vec3(2, 0.5, 1));  // 4m x 1m x 2m
const chassisBody = new CANNON.Body({ mass: 150 });
chassisBody.addShape(chassisShape);

// Vehicle
const vehicle = new CANNON.RaycastVehicle({
  chassisBody,
  indexRightAxis: 0,    // X
  indexUpAxis: 1,       // Y
  indexForwardAxis: 2,  // Z
});

// Suspension per wheel
const wheelOptions = {
  radius: 0.5,
  directionLocal: new CANNON.Vec3(0, -1, 0),
  suspensionStiffness: 30,
  suspensionRestLength: 0.3,
  frictionSlip: 1.4,
  dampingRelaxation: 2.3,
  dampingCompression: 4.4,
  maxSuspensionForce: 100000,
  rollInfluence: 0.01,
  axleLocal: new CANNON.Vec3(-1, 0, 0),
  chassisConnectionPointLocal: new CANNON.Vec3(/* per-corner */),
  customSlidingRotationalSpeed: -30,
  useCustomSlidingRotationalSpeed: true,
};

// Add 4 wheels at corners: FL(-1,0,1), FR(1,0,1), RL(-1,0,-1), RR(1,0,-1)
// Scale by half-extents
vehicle.addToWorld(world);
```

### Controls

```
W/↑         — applyEngineForce(-500, wheels 2,3)  // rear-drive
S/↓         — applyEngineForce(+500, wheels 2,3)  // reverse
A/← D/→     — setSteeringValue(±0.5, wheels 0,1)  // front steer
Space        — E-brake: rear wheels brake 1000000N, frictionSlip → 0.7
B            — Handbrake toggle
R            — Recovery flip (upright quaternion, +5m Y)
Shift+R      — Teleport to base beacon
```

### Visual Representation

```typescript
// Chassis mesh: dark metallic BoxGeometry
const chassisMesh = new THREE.Mesh(
  new THREE.BoxGeometry(4, 1, 2),
  new THREE.MeshStandardMaterial({ color: 0x222222, metalness: 0.8, roughness: 0.3 })
);

// 4 wheel meshes: CylinderGeometry (radius 0.5, width 0.4)
// 4 spotlight meshes: SpotLight per corner (warm white, intensity 5.0)
// 4 turret meshes: small box on each corner for point defense
```

### Camera — Third Person

```typescript
// In the physics postStep callback:
const offset = new THREE.Vector3(0, 8, -16);  // behind and above
offset.applyQuaternion(chassisMesh.quaternion);
const targetPos = chassisMesh.position.clone().add(offset);
camera.position.lerp(targetPos, 0.05);
camera.lookAt(chassisMesh.position);
```

---

## 4. Infinite Terrain System

### chunk-manager.ts

Manages a grid of terrain chunks around the player. Each chunk is a Cannon.js `Heightfield` + Three.js mesh.

**Constants:**
```typescript
const CHUNK_SIZE = 64;       // meters per chunk edge
const LOAD_RADIUS = 8;      // chunks to keep loaded
const UNLOAD_RADIUS = 12;   // chunks beyond this get disposed
```

**LOD System — 4 tiers:**

| Tier | Distance       | Resolution | Vertices |
|------|---------------|-----------|----------|
| 1    | 0–64m         | 33×33     | 1,089    |
| 2    | 64–128m       | 17×17     | 289      |
| 3    | 128–256m      | 9×9       | 81       |
| 4    | 256m+         | 5×5       | 25       |

**Chunk lifecycle:**
1. Each frame, compute player's chunk coordinates `(floor(px/64), floor(pz/64))`
2. For each chunk in load radius: if not loaded, queue for generation
3. Process generation queue (1–2 chunks per frame to avoid hitches)
4. For each loaded chunk beyond unload radius: dispose physics body + mesh
5. For existing chunks: check if LOD tier changed, regenerate if so

**Generating a chunk:**
```typescript
function generateChunk(cx: number, cz: number, resolution: number) {
  const data: number[][] = [];
  for (let i = 0; i < resolution; i++) {
    data[i] = [];
    for (let j = 0; j < resolution; j++) {
      const wx = cx * CHUNK_SIZE + (i / (resolution - 1)) * CHUNK_SIZE;
      const wz = cz * CHUNK_SIZE + (j / (resolution - 1)) * CHUNK_SIZE;
      data[i][j] = moonHeight(wx, wz);  // from moon-terrain.ts
    }
  }

  // Cannon.js heightfield
  const shape = new CANNON.Heightfield(data, { elementSize: CHUNK_SIZE / (resolution - 1) });
  const body = new CANNON.Body({ mass: 0 });
  body.addShape(shape);
  body.position.set(cx * CHUNK_SIZE, 0, cz * CHUNK_SIZE);
  // Rotate -90° around X (Cannon heightfields face +Y by default)
  body.quaternion.setFromEuler(-Math.PI / 2, 0, 0);
  world.addBody(body);

  // Three.js mesh from heightfield triangles
  const geometry = shapeToGeometry(shape);
  // Apply biome-appropriate vertex colors based on height
  const mesh = new THREE.Mesh(geometry, terrainMaterial);
  mesh.position.copy(body.position);
  mesh.quaternion.copy(body.quaternion);
  scene.add(mesh);
}
```

**Height query function** (used for spawning, placement):
```typescript
getHeightAt(worldX: number, worldZ: number): number {
  // Find which chunk contains this point
  // Bilinear interpolation between 4 nearest heightfield samples
  return moonHeight(worldX, worldZ);
}
```

---

## 5. Procedural Noise & Hashing

### moon-zb.ts — xxHash32 + Layer Salts

Deterministic, fast hashing for all procedural generation:

```typescript
// xxHash32 implementation — ~2-4ns per call
export function xxHash32(seed: number, input: number, salt: number): number {
  // Standard xxHash32 with 4 rounds
  // Returns uint32
}

// Derive multiple values from a single hash
export function deriveValues(hash: number, count: number): number[] {
  return Array.from({ length: count }, (_, i) =>
    xxHash32(hash, i, 0x9E3779B9) / 0xFFFFFFFF
  );
}

// Layer salts — ensure different noise per generation layer
export const LAYER_SALTS = {
  TERRAIN_BASE: 0x12345678,
  CRATER: 0x9ABCDEF0,
  REGOLITH: 0xDEADBEEF,
  SECTOR: 0xCAFEBABE,
  ENTITY: 0xF00DFACE,
  POI: 0xBAADF00D,
};
```

### moon-terrain.ts — Height Function

Multi-layer terrain generation:

```typescript
export function moonHeight(x: number, z: number): number {
  let h = 0;

  // Layer 1: Foundation (multi-octave simplex)
  h += simplex2D(x * 0.005, z * 0.005) * 20;    // Large features
  h += simplex2D(x * 0.02, z * 0.02) * 5;        // Medium detail
  h += simplex2D(x * 0.08, z * 0.08) * 1;        // Fine detail

  // Layer 2: Craters — radial falloff from procedural crater positions
  h += craterDisplacement(x, z);

  // Layer 3: Regolith — very high frequency, low amplitude
  h += simplex2D(x * 0.5, z * 0.5) * 0.2;

  return h;
}

function craterDisplacement(x: number, z: number): number {
  // Hash position to determine nearby crater centers
  // For each crater: compute distance, apply rim/bowl profile
  // Raised rim at edge, depression in center
  // Exponential falloff based on distance/radius ratio
}
```

---

## 6. Biome / Planet System

### BiomeConfig.ts

Each planet defines: terrain colors, sky, lighting, gameplay modifiers.

```typescript
export interface BiomeConfig {
  id: string;
  name: string;
  description: string;
  floorRange: { start: number; end: number };
  bossesToUnlockNext: number;
  terrain: {
    baseColor: number;
    heightScale: number;
    noiseFrequency: number;
    craterDensity: number;
    specialFeatures: string[];
  };
  palette: { low: number; mid: number; high: number; accent: number };
  sky: {
    type: 'starfield' | 'atmosphere' | 'haze';
    primaryColor: number;
    secondaryColor: number;
    starDensity: number;
    celestialBodies: CelestialBody[];
  };
  lighting: {
    ambientColor: number; ambientIntensity: number;
    sunColor: number; sunIntensity: number;
    sunPosition: { x: number; y: number; z: number };
    fogColor: number; fogDensity: number;
  };
  gameplay: {
    gravityMultiplier: number;
    visibilityRange: number;
    enemyDensityMod: number;
    difficulty: 'easy' | 'medium' | 'hard' | 'extreme' | 'nightmare';
  };
}
```

**6 planets in progression order:**

| Planet | Floors | Gravity | Difficulty | Visual Theme |
|--------|--------|---------|------------|-------------|
| Luna   | 0–4    | 0.166g  | Easy       | Gray craters, starfield, Earth visible |
| Mars   | 5–9    | 0.38g   | Medium     | Rust dunes, butterscotch sky |
| Europa | 10–14  | 0.134g  | Hard       | Blue ice, Jupiter looming |
| Titan  | 15–19  | 0.14g   | Hard       | Orange haze, methane lakes |
| Venus  | 20–24  | 0.904g  | Extreme    | Volcanic, crushing fog |
| Io     | 25–30  | 0.183g  | Nightmare  | Sulfur yellow, Jupiter massive |

**Helper functions:**
- `getPlanetForFloor(floor)` — returns BiomeConfig for current floor
- `canUnlockPlanet(id, bossDefeats)` — checks previous planet's boss defeat count
- `getNextPlanet(currentId)` — returns next in progression

---

## 7. Skybox Manager

### skybox-manager.ts

Procedural skybox using a large sphere with shader-like vertex colors.

```typescript
export class SkyboxManager {
  private skyMesh: any;    // THREE.Mesh (inverted sphere, radius 900)
  private stars: any;      // THREE.Points (BufferGeometry)
  private celestials: any[]; // Sphere meshes for planets/suns

  constructor(scene: any) {
    // Create inverted sphere (faces inward)
    const geo = new THREE.SphereGeometry(900, 32, 32);
    // Flip normals for inside rendering
  }

  updateForBiome(biome: BiomeConfig) {
    // Set sky gradient colors from biome.sky.primaryColor/secondaryColor
    // Adjust star density (Points geometry with random positions on sphere)
    // Place celestial bodies at specified azimuth/elevation
    // Apply glow (additive sprite) to bodies that have it
  }
}
```

---

## 8. Entity Manager

### entity-manager.ts — Robot AI & Combat

Manages all enemy robots: spawning, AI behavior, combat, death.

**Constants:**
```typescript
const SECTOR_SIZE = 256;
const SPAWN_CHECK_INTERVAL = 1000;  // ms
const AGGRO_RANGE = 200;
const MELEE_RANGE = 5;
const MIN_ROBOT_SPACING = 8;
const MAX_ROBOTS_PER_SECTOR = 24;
const FLOOR_Y_SEPARATION = 15;
```

**Spatial hash** for efficient queries:
```typescript
class SpatialHash {
  private cells: Map<string, Set<string>> = new Map();
  private cellSize: number;

  insert(id: string, x: number, z: number): void;
  remove(id: string, x: number, z: number): void;
  query(x: number, z: number, radius: number): string[];
}
```

**Robot mesh pool** — prevents GC pressure:
```typescript
class RobotMeshPool {
  // Key: `${classId}_${variantId % 50}`
  // Each variant has up to 4 pooled copies
  acquire(stats: RobotStats): THREE.Group;
  release(mesh: THREE.Group): void;
}
```

**AI behavior per frame:**
1. Compute distance to player
2. If distance < AGGRO_RANGE and Y-separation < 15: activate
3. Move toward player (class-specific speed)
4. If distance < MELEE_RANGE: deal damage via callback
5. Ranged classes fire projectiles at intervals
6. Division mechanic: every 4s, robot can split (max 3 times)

**Boss encounters:**
```typescript
const BOSS_CONFIG = {
  phaseThresholds: [0.75, 0.5, 0.25],
  minionSpawnInterval: 8000,
  minionsPerWave: 3,
  arenaRadius: 80,
  enrageThreshold: 0.25,
  enrageSpeedMult: 1.5,
  enrageDamageMult: 1.5,
};
```
- Boss spawns in `boss_lair` sector type
- Phases trigger at 75%, 50%, 25% HP
- Below 25%: enrage (1.5x speed and damage)
- Spawns 3 minions every 8 seconds

**Damage callback pattern:**
```typescript
entityManager.onPlayerDamaged = (rawDamage: number) => {
  let damage = rawDamage * (1 - armor / 100);  // armor reduction
  if (shieldHP > 0) {
    const shieldAbsorb = damage * 0.5;
    shieldHP -= shieldAbsorb;
    damage -= shieldAbsorb;
  }
  playerHealth -= damage;
  // Phoenix Protocol check (auto-revive buff)
};

entityManager.onRobotKilled = (robotId, position, stats) => {
  score += stats.scoreValue;
  totalKills++;
  explosionSystem.spawn(position, stats.class.id);
  playSound('robotDeath');
};
```

---

## 9. Procedural Robot Generator

### robot-generator.ts

Generates robot meshes and stats from hash seeds. No external models.

**8 robot classes:**

| Class        | HP   | Speed | Damage | Behavior |
|-------------|------|-------|--------|----------|
| Scout       | 30   | 4.0   | 5      | Fast approach, weak |
| Tank        | 200  | 1.5   | 15     | Slow, heavy armor |
| Sniper      | 50   | 2.0   | 40     | Ranged, stays back |
| Bruiser     | 120  | 2.5   | 25     | Melee charge |
| Speeder     | 40   | 6.0   | 10     | Hit-and-run |
| Artillery   | 80   | 1.0   | 50     | Long range AOE |
| Infiltrator | 60   | 3.5   | 20     | Flanking AI |
| Boss        | 1000+| 2.0   | 30     | Multi-phase |

**Mesh generation:**
```typescript
export function createRobotMesh(stats: RobotStats): THREE.Group {
  const group = new THREE.Group();
  const seed = stats.meshSeed;
  const [bodyH, bodyW, legH, armL, headS] = deriveValues(seed, 5);

  // Body: Box with class-specific proportions
  // Legs: Cylinders (2-6 based on class)
  // Arms: Optional boxes (tank/bruiser get them)
  // Head: Sphere or box
  // Eyes: Small emissive spheres (red glow)
  // Color from class palette + seed variation

  return group;
}

export function generateRobotStats(
  classId: string, floor: number, sectorSeed: number
): RobotStats {
  // Scale HP/damage/speed by floor (1 + floor * 0.1)
  // Apply biome difficulty multiplier
  // Hash-based variation (±15%)
}
```

---

## 10. Sector Manager

### sector-manager.ts

Divides the world into 256m sectors, each with a deterministic type and difficulty.

**Sector types:**
- `empty` — sparse enemies
- `outpost` — moderate enemies + POIs
- `fortress` — dense enemies, structures
- `boss_lair` — guaranteed boss encounter
- `wasteland` — environmental hazards
- `cache` — rich in collectibles

```typescript
export class SectorManager {
  getSector(worldX: number, worldZ: number): Sector {
    const sx = Math.floor(worldX / SECTOR_SIZE);
    const sz = Math.floor(worldZ / SECTOR_SIZE);
    const hash = xxHash32(worldSeed, sx * 10000 + sz, LAYER_SALTS.SECTOR);
    return {
      key: `${sx},${sz}`,
      type: determineSectorType(hash),
      difficulty: determineDifficulty(hash, currentFloor),
      enemyCount: deriveEnemyCount(hash, type),
      seed: hash,
    };
  }
}
```

---

## 11. POI / Collectible System

### poi-manager.ts

Procedurally placed collectibles that reward health, ammo, score, or shields.

```typescript
export class POIManager {
  // Spawn POIs based on sector seed
  // Types: health_pack, ammo_crate, score_crystal, shield_cell
  // Visual: floating, rotating mesh with glow
  // Collection: distance check (< 3m), auto-collect
  // Callback: onPOICollected(type, value)
  // Respawn: never (tracked in save state per sector)
}
```

---

## 12. Weapons — VLMS Missiles

### Implemented in index.tsx — Vertical Launch Missile System

**Configuration:**
```typescript
const MISSILE_CONFIG = {
  maxPoolSize: 64,
  salvoSize: 8,
  reloadTime: 3000,
  replenishTime: 6000,
  overheatCooldown: 10000,
  flightTime: 2000,
  explosionRadius: 12,
  targetRange: 800,
  launchHeight: 15,
};
```

**Targeting system:**
1. Use spatial hash to find enemies within `targetRange`
2. Filter by Y-separation (same floor only)
3. Lock up to `salvoSize` targets (scalable via buffs)
4. **Spread fire**: 2+ missiles per target, distributed evenly
5. **Focus fire**: all missiles on single target

**Missile flight — parabolic arc:**
```typescript
function updateMissile(missile, t) {
  // t = normalized time (0→1)
  const h = MISSILE_CONFIG.launchHeight;
  missile.position.y = startY + 4 * h * t * (1 - t);  // Parabola
  missile.position.lerpVectors(start, target, t);       // Horizontal
  missile.lookAt(target);  // Orient toward target
}
```

**On impact:**
- Query spatial hash for enemies in `explosionRadius`
- Damage with linear falloff: `damage = min + (max-min) * (1 - dist/radius)`
- Apply buff multipliers: `baseDamage * (stats.missileDamage / 10)`
- Overcharge buff: every 5th shot → 3× damage

**Mesh pooling**: Pre-allocate 64 cylinder meshes, show/hide as needed.

---

## 13. Weapons — Point Defense Cannons

### 4 auto-targeting cannons on vehicle corners

**Configuration:**
```typescript
const PD_CONFIG = {
  range: 20,
  fireRate: 150,    // ms per cannon
  damage: 15,
  muzzleFlashDuration: 80,
};
```

**Optimization techniques:**
- **Cached vectors** (16 pre-allocated `THREE.Vector3`): zero allocations per frame
- **Target cache**: updates every 50ms (~3 frames), not every frame
- **Muzzle flash pool**: 16 pre-allocated additive-blending meshes

**Targeting logic:**
```typescript
// Per cannon (4 total, each covers a quadrant):
function findTarget(cannonWorldPos, cannonForward) {
  let bestScore = -Infinity;
  for (const enemy of targetCache) {
    const dir = enemy.position.clone().sub(cannonWorldPos);
    const dist = dir.length();
    if (dist > PD_CONFIG.range) continue;
    dir.normalize();
    const dot = dir.dot(cannonForward);
    if (dot < -0.3) continue;  // Behind cannon
    const score = dot * 2 - dist / PD_CONFIG.range;
    if (score > bestScore) { bestScore = score; bestTarget = enemy; }
  }
}
```

**Visual**: turret mesh rotates toward target, raycasted line to target, muzzle flash.

---

## 14. Weapons — Orbital Strike

### Helldivers-style stratagem system

**Stratagem code entry:**
- Ctrl key toggles stratagem input mode
- Arrow sequence: ↑←↓↓→ (WASD: w-a-s-s-d)
- UI shows arrow icons with progress highlight
- Cooldown: 60 seconds after use

**Requires**: Orbital Tactical Uplink building (or cheat)

**Sequence:**
1. Code entered → grenade throw (parabolic, 45°, 25 m/s)
2. Grenade impacts → target marker placed
3. 5-second countdown with escalating audio chimes
4. Visual beam from sky (200m tall glowing cylinder)
5. Damage: 9999 in 15m radius (ignores shields, instant-kill everything)

---

## 15. Explosion System

### ExplosionSystem.ts — InstancedMesh Particle Pooling

Optimized for 100+ simultaneous explosions.

**4 particle types per explosion:**
- `core` — bright white sphere, expands then fades
- `fire` — orange-yellow spheres, radial velocity
- `spark` — tiny fast particles, high initial speed
- `smoke` — dark gray, slow rise, long lifetime

**Architecture:**
```typescript
export class ExplosionSystem {
  // One InstancedMesh per particle type (single draw call each)
  private coreMesh: THREE.InstancedMesh;    // max 200
  private fireMesh: THREE.InstancedMesh;    // max 800
  private sparkMesh: THREE.InstancedMesh;   // max 400
  private smokeMesh: THREE.InstancedMesh;   // max 400

  // Flat arrays for particle data (SOA layout for cache efficiency)
  private particles: ParticleData[] = [];
  private activeIndices: Set<number> = new Set();  // skip inactive

  spawn(position: THREE.Vector3, className: string) {
    // className maps to explosion size/color via ROBOT_TO_EXPLOSION config
    // Allocate N particles from pool, set velocities/lifetimes
  }

  update(dt: number) {
    // Only iterate activeIndices (not all 1800 particles)
    // Update positions: pos += vel * dt, vel.y -= gravity * dt (smoke rises)
    // Update scale: ease functions per type
    // Update matrix: dummy.position/scale → setMatrixAt
    // Mark expired particles inactive
  }
}
```

**Boss explosions** also spawn a point light (only boss-tier, too expensive for all).

---

## 16. Buff / Progression System

### BuffSystem.ts + BuffPool.ts + BuffUI.ts

Vampire Survivors-style progression with 50+ stackable buffs.

**Trigger events:**
```typescript
const TRIGGERS = {
  HEIGHT_MILESTONES: [50, 100, 200, 350, 500, 700, 1000, 1500, 2000, 2500, 3000],
  TIME_TRIGGER_INTERVAL: 45000,  // ms
  KILL_MILESTONES: [10, 25, 50, 100, 150, 200, 300, 400, 500],
};
```

When triggered: present 3 random buffs (rarity-weighted), player picks 1.

**Buff stats object:**
```typescript
export interface BuffStats {
  lockOnRange: number;      // multiplier
  lockOnTime: number;       // multiplier
  missileDamage: number;    // base 10
  salvoSize: number;        // base 8
  reloadSpeed: number;      // multiplier
  pdDamage: number;         // multiplier
  pdRange: number;          // multiplier
  pdFireRate: number;       // multiplier
  maxHealth: number;        // additive
  armor: number;            // percentage (0-100)
  shieldMax: number;        // additive
  moveSpeed: number;        // multiplier
  xpMultiplier: number;     // multiplier
  // ... 20+ more fields
}
```

**Rarity distribution:**
- Common: 60%
- Uncommon: 25%
- Rare: 12%
- Legendary: 3%

**UI**: Left-edge panel, non-intrusive. Keys 1/2/3 to select. Shows inventory of active buffs with stack counts.

---

## 17. Base Building System

### BaseBuilding.ts + BaseManager.ts

Linear unlock chain of buildings, placed at a fixed base location.

**Building definitions (13 types):**

| Building | Cost | Build Time | Effect |
|----------|------|-----------|--------|
| Home Beacon | 1,000 | 5s | Respawn point, heals within 50m |
| Long Range Radar | 2,500 | 10s | 10× radar range at base |
| Repulsor Bubble | 5,000 | 15s | Shields base from enemies |
| Extraction Complex | 10,000 | 20s | Passive XP generation |
| Drone Factory | 15,000 | 25s | Spawns combat drones |
| Landing Pad | 20,000 | 30s | Enables interplanetary travel |
| Orbital Tactical | 25,000 | 35s | Unlocks orbital strike |
| Turret ×3 | 5K–15K | 10–20s | Auto-defense turrets |
| Extractor ×3 | 8K–20K | 15–25s | Additional XP extractors |

**Placement**: All buildings placed at base beacon position, offset in a grid pattern.
**Persistence**: Building state saved. On floor change, visual meshes rebuild at new terrain heights.

---

## 18. Turret & Drone Defense

### TurretSystem.ts

```typescript
export class TurretSystem {
  private turrets: Turret[] = [];
  // Each turret: position, range, damage, fireRate, lastFired
  // update(): query EntityManager for enemies in range, fire at closest
  // Damage callback → main score system
}
```

### DroneSystem.ts

```typescript
export class DroneSystem {
  private drones: Drone[] = [];  // max 8
  // Orbit player position at fixed radius
  // Auto-engage enemies within drone range
  // Requires Drone Factory building
}
```

---

## 19. Spaceship & Planet Travel

### Spaceship.ts + PlanetTravelUI.ts

- Requires Landing Pad building
- Interact at landing pad → opens planet selection overlay
- Shows unlocked planets with boss defeat progress
- Travel triggers: fade out → reset terrain/enemies → apply new biome → fade in

---

## 20. Save System

### save-system/ directory

**5-slot system:**
- 3 manual save slots
- 1 quick save (F5)
- 1 auto save (periodic)

**SaveTypes.ts — State structure:**
```typescript
export interface GameSaveData {
  version: number;
  timestamp: number;
  checksum: string;
  game: {
    health: number; maxHealth: number;
    score: number; floor: number; planet: string;
    playtime: number;
  };
  world: {
    vehiclePos: [number, number, number];
    vehicleVel: [number, number, number];
    vehicleQuat: [number, number, number, number];
  };
  buffs: {
    active: Array<{ id: string; stacks: number }>;
    triggers: { kills: number; height: number; time: number };
  };
  progression: {
    exploredSectors: string[];
    defeatedBosses: string[];
    collectedPOIs: string[];
    bossDefeatsPerPlanet: Record<string, number>;
    unlockedPlanets: string[];
    buildings: Array<{ type: string; built: boolean }>;
  };
}
```

**SaveValidator.ts**: xxHash32 checksum of serialized JSON, validated on load.
**SaveStorage.ts**: `localStorage.setItem('save_slot_N', json)`.
**SaveUI.ts**: ESC key opens overlay with slot management + graphics settings.

---

## 21. Procedural Audio

### All sounds synthesized via Web Audio API — no external files

```typescript
const audioCtx = new AudioContext();
const masterGain = audioCtx.createGain();
masterGain.gain.value = 0.3;
masterGain.connect(audioCtx.destination);
```

**Sound catalog (16 types):**

| Sound | Technique | Frequency | Duration |
|-------|-----------|-----------|----------|
| Lock Ping | Sine sweep | 1200→800 Hz | 100ms |
| Missile Launch | Sawtooth + noise | 100→400 Hz | 200ms |
| PD Fire | Square wave | 2000→800 Hz | 50ms |
| Explosion | Low sine + noise burst | 60–120 Hz | 500ms |
| Damage Taken | Harsh sawtooth + square | 200–400 Hz | 300ms |
| Robot Death | Filtered noise crackle | Broadband | 200ms |
| POI Pickup | Ascending chime (C5-E5-G5) | 523–784 Hz | 300ms |
| Boss Alert | Descending ominous | 200→80 Hz | 1000ms |
| Buff Available | Pleasant ascending | 400→800 Hz | 200ms |
| Buff Select | Confirmation arpeggio | C-E-G-C | 400ms |
| Construction | Mechanical clunk | 100–300 Hz | 150ms |
| Build Complete | Rising 3-note chime | 400-500-600 Hz | 400ms |
| Error | Low buzz | 80 Hz | 200ms |
| Turret Fire | Sharp pop | 1000 Hz | 30ms |
| Drone Fire | High zap | 3000→1500 Hz | 40ms |
| Orbital Strike | Multi-phase sequence | Various | 6000ms |

**Pattern for each sound:**
```typescript
function playExplosion() {
  if (cooldowns.explosion > Date.now()) return;  // prevent spam
  cooldowns.explosion = Date.now() + 100;

  // Low boom
  const osc = audioCtx.createOscillator();
  osc.type = 'sine';
  osc.frequency.setValueAtTime(80, audioCtx.currentTime);
  osc.frequency.exponentialRampToValueAtTime(30, audioCtx.currentTime + 0.5);

  const gain = audioCtx.createGain();
  gain.gain.setValueAtTime(0.4, audioCtx.currentTime);
  gain.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + 0.5);

  osc.connect(gain).connect(masterGain);
  osc.start();
  osc.stop(audioCtx.currentTime + 0.5);

  // Noise burst (create buffer of random samples)
  // ... similar pattern with noise source
}
```

---

## 22. HUD & UI

### All HTML overlay elements, no canvas UI (except radar)

**Top-Left HUD panel:**
```html
<div id="hud" style="position:fixed; top:10px; left:10px; font-family:monospace; color:#0f0; font-size:13px; text-shadow:0 0 4px #0f0;">
  <!-- Updated every frame via innerHTML -->
  HP: 100/100 (+25 shield)<br>
  Score: 123,456 (×1.5)<br>
  Luna - Floor 1<br>
  Outpost | Medium<br>
  Enemies: 12 | Kills: 456<br>
  Height: 45.2 (Max: 127.8)<br>
  Bosses: 3 | Planets: 2/6<br>
  Buffs: 8
</div>
```

**Top-Right Missile HUD:**
```
═ VLMS ═
Ammo: 64/64
Mode: SPREAD
[██████████]
READY
```

**Top-Center Radar** (Canvas 140×140):
- Green ring border
- Forward indicator (vehicle heading)
- Orange pulsing arrow → nearest enemy
- Green arrow → base beacon
- Red dots → nearby enemies (distance-scaled size)

**Boss HP Bar** (top-center during encounters):
- Phase-colored gradient
- Enrage animation (red pulse)
- Boss name + phase number

**Buff Selection Panel** (left edge, triggered by milestones):
- 3 buff cards with rarity borders
- Keys 1/2/3 to select
- Auto-dismiss after 15 seconds (picks first)

---

## 23. WebGPU Acceleration

### Optional — Falls back to CPU if WebGPU unavailable

**gpu/WebGPUContext.ts**: Initialize adapter + device, handle capability detection.

**gpu/HeightfieldCompute.ts**: WGSL compute shader for terrain height generation.
- Dispatches workgroup per heightfield sample
- 10–50× faster than CPU simplex noise
- Output: Float32Array of heights, consumed by chunk-manager

**gpu/CraterCompute.ts**: WGSL compute shader for crater deformation.

**gpu/shaders/heightfield.wgsl:**
```wgsl
@group(0) @binding(0) var<storage, read_write> heights: array<f32>;
@group(0) @binding(1) var<uniform> params: HeightParams;

@compute @workgroup_size(64)
fn main(@builtin(global_invocation_id) id: vec3<u32>) {
    let idx = id.x;
    let x = f32(idx % params.width) * params.elementSize + params.offsetX;
    let z = f32(idx / params.width) * params.elementSize + params.offsetZ;
    heights[idx] = moonHeight(x, z);  // Noise functions ported to WGSL
}
```

---

## 24. Game Loop

### index.tsx — Main Entry Point & Wiring

This is the largest file (~4700 lines in reference). It wires everything together.

**Initialization order:**
```typescript
// 1. Create Demo instance (Three.js + Cannon.js)
const app = new Demo({ shadows: true });

// 2. Configure physics world
const world = app.getWorld();
world.gravity.set(0, -10, 0);
world.broadphase = new CANNON.SAPBroadphase(world);
world.defaultContactMaterial.friction = 0.3;

// 3. Create vehicle (chassis + 4 wheels)
const vehicle = createVehicle(world);

// 4. Initialize subsystems
const chunkManager = new ChunkManager(world, app.scene);
const sectorManager = new SectorManager(worldSeed);
const entityManager = new EntityManager(app.scene, world, sectorManager);
const poiManager = new POIManager(app.scene, sectorManager);
const explosionSystem = new ExplosionSystem(app.scene);
const buffSystem = new BuffSystem();
const baseManager = new BaseManager(app.scene, world);
const turretSystem = new TurretSystem(app.scene, entityManager);
const droneSystem = new DroneSystem(app.scene, entityManager);
const skyboxManager = new SkyboxManager(app.scene);
const saveManager = new SaveManager();

// 5. Wire callbacks
entityManager.onPlayerDamaged = (dmg) => { /* apply damage */ };
entityManager.onRobotKilled = (id, pos, stats) => { /* score, explode */ };
poiManager.onPOICollected = (type, value) => { /* apply reward */ };
buffSystem.onBuffTriggered = () => { /* show buff selection UI */ };

// 6. Set up input handlers (keyboard state map)
// 7. Start game loop
```

**Physics postStep callback (60Hz):**
```typescript
world.addEventListener('postStep', () => {
  // 1. Update chassis/wheel visual transforms
  // 2. Apply steering/engine forces based on input
  // 3. Apply e-brake if active
  // 4. Update terrain chunks (chunkManager.update(vehiclePos))
  // 5. Update entities (entityManager.update(vehiclePos, dt))
  // 6. Update POIs (poiManager.update(vehiclePos))
  // 7. Update base systems (baseManager, turrets, drones)
  // 8. Base healing (if in range)
  // 9. Buff system triggers check
  // 10. Camera follow (lerp to offset behind vehicle)
  // 11. Camera shake (from nearby explosions)
  // 12. Update spotlights (follow vehicle orientation)
  // 13. Update HUD text
});
```

**Render loop (requestAnimationFrame):**
```typescript
function renderLoop() {
  requestAnimationFrame(renderLoop);
  // 1. Update missile flight paths
  // 2. Update point defense targeting + firing
  // 3. Update orbital strike sequence
  // 4. Update explosion particles
  // 5. Update radar canvas
  // 6. Update navigation indicator
  // 7. renderer.render(scene, camera)
}
```

---

## 25. Cheat / Debug System

### CheatManager.ts + CheatMenuUI.ts

**Activation**: Type "IDKFA" at any time → toggles cheat menu.

**F1**: Show/hide cheat panel.

**Cheats:**
- God Mode (invincibility)
- Infinite Ammo (no reload)
- All Weapons (orbital strike without building)
- Max Buffs (apply all legendary buffs)
- Teleport (click to teleport)
- Kill All (clear current sector)
- Unlock All Planets
- Speed ×2/×4

---

## Implementation Order (Recommended)

Execute these phases sequentially. Each phase produces a testable result.

### Phase 1 — Drivable Vehicle on Flat Ground
1. Project scaffold (package.json, vite.config.ts, tsconfig.json, style.css)
2. `Demo.ts` + `utils.ts` (framework)
3. Vehicle creation in `index.tsx` (chassis + wheels + controls)
4. Third-person camera
5. **Test**: WASD driving on a flat Cannon.js plane

### Phase 2 — Infinite Terrain
6. `moon-zb.ts` (hashing)
7. `moon-terrain.ts` (height function)
8. `chunk-manager.ts` (LOD terrain generation)
9. **Test**: Drive over procedurally generated terrain

### Phase 3 — Enemies & Combat
10. `robot-generator.ts` (procedural meshes + stats)
11. `sector-manager.ts` (sector types)
12. `entity-manager.ts` (spawning + AI + combat)
13. HUD (health, score, enemy count)
14. **Test**: Enemies spawn and attack, player takes damage

### Phase 4 — Weapons
15. VLMS missile system (targeting, flight, AOE)
16. Point defense cannons (auto-targeting)
17. Explosion system (InstancedMesh particles)
18. Missile HUD + target lock boxes
19. **Test**: Full combat loop — drive, lock, fire, destroy

### Phase 5 — Progression
20. `BuffPool.ts` + `BuffSystem.ts` + `BuffUI.ts`
21. POI system (collectibles)
22. Score system + kill tracking
23. **Test**: Buff selection triggers at milestones

### Phase 6 — Base Building
24. `BaseBuilding.ts` + `BaseManager.ts`
25. `TurretSystem.ts` + `DroneSystem.ts`
26. Orbital strike system
27. **Test**: Build base, turrets fire, drones orbit

### Phase 7 — Multi-Planet
28. `BiomeConfig.ts` (all 6 planets)
29. `skybox-manager.ts`
30. `Spaceship.ts` + `PlanetTravelUI.ts`
31. Floor progression (N key to descend)
32. **Test**: Travel between planets, visual changes

### Phase 8 — Polish
33. Procedural audio (all 16 sounds)
34. Radar canvas
35. Save system (full save/load)
36. Cheat system
37. WebGPU acceleration (optional)
38. **Test**: Full game loop, save/load, all planets

---

## Critical Patterns to Preserve

### CDN Imports (No npm for Three/Cannon)
```typescript
// @ts-ignore
import * as CANNON from 'https://unpkg.com/cannon-es@0.20.0/dist/cannon-es.js';
// @ts-ignore
import * as THREE from 'https://unpkg.com/three@0.122.0/build/three.module.js';
```

### Mesh Pooling (Prevent GC)
Always pool meshes that are created/destroyed frequently: missiles, particles, enemy meshes. Never `new THREE.Mesh()` in hot loops.

### Spatial Hashing (O(1) Queries)
Use grid-based spatial hash for all range queries: missile targeting, point defense, entity proximity. Never iterate all entities linearly.

### Cached Vectors (Zero Allocation)
Pre-allocate `THREE.Vector3` / `CANNON.Vec3` for per-frame calculations. Never create vectors in update loops.

### Separation of Physics and Render Loops
Physics runs at fixed 60Hz via `world.step()`. Rendering runs at display refresh rate via `requestAnimationFrame`. The `postStep` callback bridges them.
