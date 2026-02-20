# Desert Harvest — Framework Implementation Plan

> **Target**: Claude Code autonomous implementation  
> **Stack**: Three.js 0.122 (CDN) + Cannon-ES 0.20 (CDN) + Vite 6 + TypeScript 5.8  
> **Zero external assets** — all geometry, audio, and terrain are procedural  
> **Reference codebase**: `raycastvehicle-demo` (Moon Defenders) adapted for RTS-lite harvester gameplay

---

## Game Vision

**Desert Harvest** is a Command & Conquer-inspired experience where the player directly drives a heavily-armed harvester truck across an endless procedural desert. The core loop: collect ORE from resource zones, transport it to your Refinery, build your base through a linear tech chain, produce combat units, and command them via a tactical map overlay to destroy enemy command centers.

**Key Differentiators from Original Plan:**
- Player is the harvester (Supreme Commander ACU-style hero unit)
- Single persistent desert biome with weather/time visual variations
- RTS-lite unit production and command (tactical map with fog of war)
- Resource economy (ORE collection + transport → construction budget)
- Victory through objective completion, then seed reset for replayability
- Orbital strikes as endgame superweapons for destroying underground bunkers

---

## Table of Contents

1. [Project Scaffold](#1-project-scaffold)
2. [Core Framework — Demo.ts + utils.ts](#2-core-framework)
3. [Harvester Vehicle](#3-harvester-vehicle)
4. [Infinite Desert Terrain](#4-infinite-desert-terrain)
5. [Procedural Noise & Hashing](#5-procedural-noise--hashing)
6. [Weather & Time System](#6-weather--time-system)
7. [Skybox Manager](#7-skybox-manager)
8. [Resource System](#8-resource-system)
9. [Sector Manager — Zone Generation](#9-sector-manager--zone-generation)
10. [Entity Manager — AI & Spawning](#10-entity-manager--ai-spawning)
11. [Procedural Enemy Generator](#11-procedural-enemy-generator)
12. [Enemy Bases & Outposts](#12-enemy-bases--outposts)
13. [Weapons — VLMS Missiles](#13-weapons--vlms-missiles)
14. [Weapons — Orbital Strike](#14-weapons--orbital-strike)
15. [Explosion System](#15-explosion-system)
16. [Base Building System](#16-base-building-system)
17. [Unit Production & Command](#17-unit-production--command)
18. [Tactical Map Overlay](#18-tactical-map-overlay)
19. [Resource Beacons](#19-resource-beacons)
20. [Difficulty Scaling (Floors)](#20-difficulty-scaling-floors)
21. [Save System](#21-save-system)
22. [Procedural Audio](#22-procedural-audio)
23. [HUD & UI](#23-hud--ui)
24. [WebGPU Acceleration (Optional)](#24-webgpu-acceleration)
25. [Game Loop — Wiring Everything Together](#25-game-loop)
26. [Cheat / Debug System](#26-cheat--debug-system)

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
│   ├── resource-manager.ts      ← ORE/ENERGY economy
│   ├── desert-terrain.ts        ← Sand dune generation
│   ├── desert-zb.ts             ← ZeroByte hashing for desert
│   ├── enemy-generator.ts       ← Procedural enemy meshes/stats
│   ├── skybox-manager.ts
│   ├── weather/
│   │   └── WeatherConfig.ts     ← Time of day + weather variations
│   ├── systems/
│   │   ├── BaseBuilding.ts
│   │   ├── BaseManager.ts
│   │   ├── UnitFactory.ts       ← Drone/Heli/Tank production
│   │   ├── UnitManager.ts       ← Unit AI + command execution
│   │   ├── TacticalMap.ts       ← RTS overlay with fog of war
│   │   ├── CommandSystem.ts     ← Unit orders + groups
│   │   ├── TurretSystem.ts      ← Perimeter defense turrets
│   │   ├── ExplosionSystem.ts
│   │   ├── OrbitalStrike.ts     ← GPS beacon + strike system
│   │   ├── ResourceBeacon.ts    ← Navigation beacons (max 3)
│   │   ├── VLMSSystem.ts        ← Missile weapons
│   │   ├── CheatManager.ts
│   │   └── CheatMenuUI.ts
│   ├── config/
│   │   ├── explosion-config.ts
│   │   ├── building-config.ts   ← Tech chain definitions
│   │   └── unit-config.ts       ← Unit stats/costs
│   ├── save-system/
│   │   ├── index.ts
│   │   ├── SaveManager.ts
│   │   ├── SaveSerializer.ts
│   │   ├── SaveStorage.ts
│   │   ├── SaveValidator.ts
│   │   ├── SaveUI.ts
│   │   └── SaveTypes.ts
│   ├── gpu/                     ← Optional WebGPU
│   │   ├── index.ts
│   │   ├── WebGPUContext.ts
│   │   ├── HeightfieldCompute.ts
│   │   ├── GPUTelemetry.ts
│   │   └── shaders/
│   │       └── heightfield.wgsl
│   └── types/
│       └── webgpu.d.ts
```

### package.json

```json
{
  "name": "desert-harvest",
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

This is the engine shell. It manages the physics world, renderer, scene graph, and body↔visual synchronization. **Architecture preserved from original plan.**

```typescript
// CDN imports (no npm packages for Three/Cannon)
// @ts-ignore
import * as CANNON from 'https://unpkg.com/cannon-es@0.20.0/dist/cannon-es.js';
// @ts-ignore
import * as THREE from 'https://unpkg.com/three@0.122.0/build/three.module.js';
import { bodyToMesh } from './utils.js';

export class Demo extends CANNON.EventTarget {
  bodies: any[] = [];
  visuals: any[] = [];
  world: any;
  scene: any;
  camera: any;
  renderer: any;
  settings: any;

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

  initThree() {
    this.camera = new THREE.PerspectiveCamera(24, window.innerWidth / window.innerHeight, 5, 2000);
    this.scene = new THREE.Scene();
    // Desert fog - sand/dust colored
    this.scene.fog = new THREE.Fog(0xD4A574, 500, 1500);
    this.renderer = new THREE.WebGLRenderer({ antialias: true });
    this.renderer.setSize(window.innerWidth, window.innerHeight);
    this.renderer.shadowMap.enabled = true;
    this.renderer.shadowMap.type = THREE.PCFSoftShadowMap;
    document.body.appendChild(this.renderer.domElement);
  }
}
```

**Critical methods:**
- `addVisual(body)` — creates Three.js mesh from Cannon body
- `removeVisual(body)` — removes from both arrays and scene
- `updateVisuals()` — syncs physics to visuals each frame
- `updatePhysics()` — calls `world.step(1/60, delta, maxSubSteps)`

### utils.ts — Shape↔Geometry Conversion

Converts every Cannon.js shape type to Three.js geometry. **Unchanged from original plan.**

---

## 3. Harvester Vehicle

### The Hero Unit — Supreme Commander ACU-Style

The player's harvester truck is the central unit:
- **Directly controlled** in 3rd person (WASD + mouse)
- **Heavily armed** with VLMS missiles (auto-targeting nearest enemies)
- **Highest HP unit** in the game (5000 HP)
- **Death = Game Over** (defeat condition)
- **Resource collector** when parked in ORE zones
- **Resource transporter** to Refinery for conversion

### Vehicle Configuration

```typescript
interface HarvesterConfig {
  // Chassis
  chassisMass: 8000,
  chassisWidth: 4,
  chassisHeight: 3,
  chassisLength: 8,
  
  // Stats
  maxHP: 5000,
  armor: 50,              // Damage reduction %
  
  // Movement
  maxSpeed: 25,           // m/s - slow but steady
  acceleration: 8,
  turnRate: 0.8,
  
  // Cargo
  oreCapacity: 500,
  currentOre: 0,
  
  // Weapons
  vlmsAmmo: 64,
  vlmsMaxAmmo: 64,
  vlmsReloadTime: 0.5,
  
  // Collection
  oreCollectionRate: 0,   // Set by current zone
  isCollecting: false,
}
```

### Vehicle Physics Setup

```typescript
function createHarvester(world: any): HarvesterVehicle {
  const chassisShape = new CANNON.Box(new CANNON.Vec3(4, 1.5, 2));
  const chassisBody = new CANNON.Body({ mass: 8000 });
  chassisBody.addShape(chassisShape);
  chassisBody.position.set(0, 4, 0);
  
  const vehicle = new CANNON.RaycastVehicle({
    chassisBody,
    indexRightAxis: 0,
    indexUpAxis: 1,
    indexForwardAxis: 2,
  });
  
  // 6 wheels (3 axles) for heavy truck
  const wheelOptions = {
    radius: 1.2,
    directionLocal: new CANNON.Vec3(0, -1, 0),
    suspensionStiffness: 50,
    suspensionRestLength: 0.8,
    frictionSlip: 3,
    dampingRelaxation: 2.3,
    dampingCompression: 4.4,
    maxSuspensionForce: 200000,
    rollInfluence: 0.1,
    axleLocal: new CANNON.Vec3(-1, 0, 0),
    chassisConnectionPointLocal: new CANNON.Vec3(),
    maxSuspensionTravel: 0.5,
  };
  
  // Front axle (steering)
  wheelOptions.chassisConnectionPointLocal.set(1.5, 0, 3);
  vehicle.addWheel(wheelOptions);
  wheelOptions.chassisConnectionPointLocal.set(-1.5, 0, 3);
  vehicle.addWheel(wheelOptions);
  
  // Middle axle
  wheelOptions.chassisConnectionPointLocal.set(1.5, 0, 0);
  vehicle.addWheel(wheelOptions);
  wheelOptions.chassisConnectionPointLocal.set(-1.5, 0, 0);
  vehicle.addWheel(wheelOptions);
  
  // Rear axle
  wheelOptions.chassisConnectionPointLocal.set(1.5, 0, -3);
  vehicle.addWheel(wheelOptions);
  wheelOptions.chassisConnectionPointLocal.set(-1.5, 0, -3);
  vehicle.addWheel(wheelOptions);
  
  vehicle.addToWorld(world);
  return { vehicle, chassisBody, config: { ...defaultHarvesterConfig } };
}
```

### Harvester Visual Design

```typescript
function createHarvesterMesh(): THREE.Group {
  const group = new THREE.Group();
  
  // Main body - industrial yellow/orange
  const bodyMat = new THREE.MeshStandardMaterial({ 
    color: 0xE8A92A, metalness: 0.3, roughness: 0.7 
  });
  const body = new THREE.Mesh(new THREE.BoxGeometry(8, 3, 4), bodyMat);
  body.position.y = 1.5;
  group.add(body);
  
  // Cab (front)
  const cabMat = new THREE.MeshStandardMaterial({ color: 0x333333, metalness: 0.5 });
  const cab = new THREE.Mesh(new THREE.BoxGeometry(3, 2, 3.5), cabMat);
  cab.position.set(0, 3, 1);
  group.add(cab);
  
  // Ore hopper (back)
  const hopperMat = new THREE.MeshStandardMaterial({ color: 0x666666, metalness: 0.4 });
  const hopper = new THREE.Mesh(new THREE.BoxGeometry(7, 2.5, 3.5), hopperMat);
  hopper.position.set(0, 3, -1.5);
  group.add(hopper);
  
  // VLMS launcher pods (top rear)
  const launcherMat = new THREE.MeshStandardMaterial({ color: 0x1a1a1a });
  const leftLauncher = new THREE.Mesh(new THREE.BoxGeometry(1, 0.8, 2), launcherMat);
  leftLauncher.position.set(1.5, 4.5, -2);
  group.add(leftLauncher);
  const rightLauncher = leftLauncher.clone();
  rightLauncher.position.set(-1.5, 4.5, -2);
  group.add(rightLauncher);
  
  return group;
}
```

### Third-Person Camera

```typescript
class HarvesterCamera {
  offset = new THREE.Vector3(0, 15, -30);
  lookOffset = new THREE.Vector3(0, 2, 10);
  lerpFactor = 0.05;
  
  update(camera: THREE.PerspectiveCamera, harvester: HarvesterVehicle) {
    const chassisPos = harvester.chassisBody.position;
    const chassisQuat = harvester.chassisBody.quaternion;
    
    const rotatedOffset = this.offset.clone();
    rotatedOffset.applyQuaternion(new THREE.Quaternion(
      chassisQuat.x, chassisQuat.y, chassisQuat.z, chassisQuat.w
    ));
    
    const targetPos = new THREE.Vector3(
      chassisPos.x + rotatedOffset.x,
      chassisPos.y + rotatedOffset.y,
      chassisPos.z + rotatedOffset.z
    );
    
    camera.position.lerp(targetPos, this.lerpFactor);
    
    const rotatedLook = this.lookOffset.clone();
    rotatedLook.applyQuaternion(new THREE.Quaternion(
      chassisQuat.x, chassisQuat.y, chassisQuat.z, chassisQuat.w
    ));
    camera.lookAt(
      chassisPos.x + rotatedLook.x,
      chassisPos.y + rotatedLook.y,
      chassisPos.z + rotatedLook.z
    );
  }
}
```

---

## 4. Infinite Desert Terrain

### Sand Dune Generation (No Craters)

The terrain is an endless desert with rolling sand dunes via layered noise. No impact craters.

### desert-terrain.ts

```typescript
import { hashNoise2D, hashNoise2DOctaves } from './desert-zb.js';

const DUNE_CONFIG = {
  primaryScale: 0.002,
  primaryAmplitude: 40,
  secondaryScale: 0.01,
  secondaryAmplitude: 8,
  detailScale: 0.05,
  detailAmplitude: 1.5,
  windAngle: Math.PI * 0.25,
};

export function desertHeight(x: number, z: number, seed: number = 0): number {
  // Rotate by wind angle for directional dunes
  const cos = Math.cos(DUNE_CONFIG.windAngle);
  const sin = Math.sin(DUNE_CONFIG.windAngle);
  const rx = x * cos - z * sin;
  const rz = x * sin + z * cos;
  
  // Primary dunes - asymmetric for realistic shape
  const primaryNoise = hashNoise2DOctaves(
    rx * DUNE_CONFIG.primaryScale,
    rz * DUNE_CONFIG.primaryScale * 0.5,
    seed, 3
  );
  const primary = Math.pow(Math.abs(primaryNoise), 0.8) * 
    Math.sign(primaryNoise) * DUNE_CONFIG.primaryAmplitude;
  
  // Secondary ripples
  const secondary = hashNoise2DOctaves(
    x * DUNE_CONFIG.secondaryScale,
    z * DUNE_CONFIG.secondaryScale,
    seed + 1000, 2
  ) * DUNE_CONFIG.secondaryAmplitude;
  
  // Fine detail
  const detail = hashNoise2D(
    x * DUNE_CONFIG.detailScale,
    z * DUNE_CONFIG.detailScale,
    seed + 2000
  ) * DUNE_CONFIG.detailAmplitude;
  
  return 20 + primary + secondary + detail;
}

export function desertColor(height: number, slope: number): THREE.Color {
  const lightSand = new THREE.Color(0xE8D4A8);
  const darkSand = new THREE.Color(0xC4A66A);
  const redSand = new THREE.Color(0xD4A574);
  
  const heightFactor = Math.min(1, Math.max(0, (height - 20) / 50));
  const slopeFactor = Math.min(1, slope * 2);
  
  const color = lightSand.clone();
  color.lerp(darkSand, slopeFactor * 0.5);
  color.lerp(redSand, heightFactor * 0.3);
  return color;
}
```

### Chunk Manager

```typescript
const CHUNK_CONFIG = {
  chunkSize: 64,
  resolution: 32,
  lodLevels: 4,
  maxLoadedChunks: 100,
  loadRadius: 3,
  unloadRadius: 5,
};

class ChunkManager {
  private chunks: Map<string, DesertChunk> = new Map();
  private worldSeed: number;
  
  update(playerPos: THREE.Vector3) {
    const playerChunkX = Math.floor(playerPos.x / CHUNK_CONFIG.chunkSize);
    const playerChunkZ = Math.floor(playerPos.z / CHUNK_CONFIG.chunkSize);
    
    // Load nearby chunks
    for (let dx = -CHUNK_CONFIG.loadRadius; dx <= CHUNK_CONFIG.loadRadius; dx++) {
      for (let dz = -CHUNK_CONFIG.loadRadius; dz <= CHUNK_CONFIG.loadRadius; dz++) {
        const key = `${playerChunkX + dx},${playerChunkZ + dz}`;
        if (!this.chunks.has(key)) {
          this.loadChunk(playerChunkX + dx, playerChunkZ + dz);
        }
      }
    }
    
    // Unload distant chunks
    for (const [key, chunk] of this.chunks) {
      const [cx, cz] = key.split(',').map(Number);
      const dist = Math.max(Math.abs(cx - playerChunkX), Math.abs(cz - playerChunkZ));
      if (dist > CHUNK_CONFIG.unloadRadius) {
        this.unloadChunk(key);
      }
    }
  }
}
```

---

## 5. Procedural Noise & Hashing

### desert-zb.ts — ZeroByte Position-as-Seed System

Distance from origin determines difficulty floor.

```typescript
export function hash2D(x: number, y: number, seed: number = 0): number {
  const n = Math.sin(x * 12.9898 + y * 78.233 + seed) * 43758.5453;
  return n - Math.floor(n);
}

export function hashNoise2D(x: number, y: number, seed: number = 0): number {
  const ix = Math.floor(x);
  const iy = Math.floor(y);
  const fx = x - ix;
  const fy = y - iy;
  
  const ux = fx * fx * (3 - 2 * fx);
  const uy = fy * fy * (3 - 2 * fy);
  
  const a = hash2D(ix, iy, seed);
  const b = hash2D(ix + 1, iy, seed);
  const c = hash2D(ix, iy + 1, seed);
  const d = hash2D(ix + 1, iy + 1, seed);
  
  return a + (b - a) * ux + (c - a) * uy + (a - b - c + d) * ux * uy;
}

export function hashNoise2DOctaves(x: number, y: number, seed: number, octaves: number): number {
  let value = 0, amplitude = 1, frequency = 1, maxValue = 0;
  for (let i = 0; i < octaves; i++) {
    value += hashNoise2D(x * frequency, y * frequency, seed + i * 100) * amplitude;
    maxValue += amplitude;
    amplitude *= 0.5;
    frequency *= 2;
  }
  return (value / maxValue) * 2 - 1;
}

// Difficulty floor based on distance from origin
export function getFloorAtPosition(x: number, z: number): number {
  const distance = Math.sqrt(x * x + z * z);
  return Math.floor(distance / 1000); // Every 1000 units = +1 floor
}

export function getSectorSeed(sectorX: number, sectorZ: number, worldSeed: number): number {
  return Math.floor(hash2D(sectorX, sectorZ, worldSeed) * 0x7FFFFFFF);
}
```

---

## 6. Weather & Time System

### Visual-Only Variations (No Gameplay Impact)

```typescript
export enum TimeOfDay { Dawn = 'dawn', Day = 'day', Dusk = 'dusk', Night = 'night' }
export enum WeatherType { Clear = 'clear', Dusty = 'dusty', Sandstorm = 'sandstorm', Overcast = 'overcast' }

export const TIME_CONFIGS: Record<TimeOfDay, TimeConfig> = {
  dawn: {
    sunColor: 0xFFAA66, sunIntensity: 0.6,
    ambientColor: 0x666688, ambientIntensity: 0.3,
    fogColor: 0xFFCCAA, fogNear: 400, fogFar: 1200,
    skyGradient: ['#1a1a3a', '#FF6644', '#FFAA66'],
    sunAngle: 10,
  },
  day: {
    sunColor: 0xFFFFEE, sunIntensity: 1.2,
    ambientColor: 0x8888AA, ambientIntensity: 0.4,
    fogColor: 0xD4A574, fogNear: 500, fogFar: 1500,
    skyGradient: ['#4488CC', '#88BBEE', '#AACCFF'],
    sunAngle: 60,
  },
  dusk: {
    sunColor: 0xFF6644, sunIntensity: 0.5,
    ambientColor: 0x664444, ambientIntensity: 0.25,
    fogColor: 0xAA6644, fogNear: 350, fogFar: 1000,
    skyGradient: ['#1a1a3a', '#AA4422', '#FFAA44'],
    sunAngle: 5,
  },
  night: {
    sunColor: 0x4466AA, sunIntensity: 0.15,
    ambientColor: 0x222244, ambientIntensity: 0.1,
    fogColor: 0x111122, fogNear: 200, fogFar: 800,
    skyGradient: ['#000011', '#111133', '#222244'],
    sunAngle: -30,
  },
};

class WeatherManager {
  private currentState: WeatherState;
  private transitionSpeed = 0.001;
  
  update(dt: number) {
    // Smooth lerp between weather states
    // Update fog, sun, ambient lighting
  }
}
```

---

## 7. Skybox Manager

### Procedural Desert Sky

```typescript
class SkyboxManager {
  private material: THREE.ShaderMaterial;
  
  constructor(scene: THREE.Scene) {
    const geometry = new THREE.SphereGeometry(1000, 32, 32);
    geometry.scale(-1, 1, 1);
    
    this.material = new THREE.ShaderMaterial({
      vertexShader: `
        varying vec3 vWorldPosition;
        void main() {
          vec4 worldPosition = modelMatrix * vec4(position, 1.0);
          vWorldPosition = worldPosition.xyz;
          gl_Position = projectionMatrix * viewMatrix * worldPosition;
        }
      `,
      fragmentShader: `
        uniform vec3 topColor, middleColor, bottomColor;
        uniform float sunSize;
        uniform vec3 sunPosition, sunColor;
        varying vec3 vWorldPosition;
        
        void main() {
          vec3 dir = normalize(vWorldPosition);
          float y = dir.y * 0.5 + 0.5;
          vec3 sky = y > 0.5 
            ? mix(middleColor, topColor, (y - 0.5) * 2.0)
            : mix(bottomColor, middleColor, y * 2.0);
          
          vec3 sunDir = normalize(sunPosition);
          float sunDot = dot(dir, sunDir);
          sky = mix(sky, sunColor, smoothstep(1.0 - sunSize, 1.0, sunDot));
          sky += sunColor * pow(max(0.0, sunDot), 8.0) * 0.5;
          
          gl_FragColor = vec4(sky, 1.0);
        }
      `,
      uniforms: {
        topColor: { value: new THREE.Color(0x4488CC) },
        middleColor: { value: new THREE.Color(0x88BBEE) },
        bottomColor: { value: new THREE.Color(0xD4A574) },
        sunSize: { value: 0.02 },
        sunPosition: { value: new THREE.Vector3(100, 60, 50) },
        sunColor: { value: new THREE.Color(0xFFFFEE) },
      },
      side: THREE.BackSide,
    });
    
    scene.add(new THREE.Mesh(geometry, this.material));
  }
}
```

---

## 8. Resource System

### ORE + ENERGY Economy

```typescript
export interface ResourceState {
  ore: number;               // Carried by harvester
  oreCapacity: number;       // Harvester cargo limit (500)
  storedOre: number;         // At refinery (not yet converted)
  constructionBudget: number; // Converted from ore
  energy: number;            // From pylons
  energyCapacity: number;    // Based on pylon count
}

export interface OreZone {
  id: string;
  position: THREE.Vector3;
  radius: number;
  richness: number;          // 1-5 multiplier
  totalOre: number;          // Depletes over time
  collectionRate: number;    // Base ore/second
}

export class ResourceManager {
  private state: ResourceState;
  private activeZone: OreZone | null = null;
  
  // Called every frame while harvester is stationary in zone
  update(dt: number, harvesterVelocity: number) {
    if (this.activeZone && harvesterVelocity < 1) {
      const rate = this.activeZone.collectionRate * this.activeZone.richness;
      const collected = Math.min(
        rate * dt,
        this.state.oreCapacity - this.state.ore,
        this.activeZone.totalOre
      );
      this.state.ore += collected;
      this.activeZone.totalOre -= collected;
    }
    
    // Energy regen from pylons
    this.state.energy = Math.min(
      this.state.energyCapacity,
      this.state.energy + this.pylons.length * 2 * dt
    );
  }
  
  depositOre() {
    if (this.refinery && this.state.ore > 0) {
      this.state.storedOre += this.state.ore;
      this.state.ore = 0;
      this.processOre();
    }
  }
  
  canAfford(oreCost: number, energyCost: number): boolean {
    return this.state.constructionBudget >= oreCost && 
           this.state.energy >= energyCost;
  }
}
```

---

## 9. Sector Manager — Zone Generation

### Sector Types

```typescript
export enum SectorType {
  Empty = 'empty',
  OreField = 'ore_field',
  EnemyOutpost = 'enemy_outpost',
  EnemyBase = 'enemy_base',
  Ruins = 'ruins',
  Bunker = 'bunker',          // Requires orbital strike
}

export interface Sector {
  x: number; z: number;
  type: SectorType;
  seed: number;
  floor: number;
  discovered: boolean;
  cleared: boolean;
  claimed: boolean;
  oreZone?: OreZone;
  enemySpawns?: EnemySpawn[];
}

const SECTOR_SIZE = 256;

export class SectorManager {
  private sectors: Map<string, Sector> = new Map();
  
  private generateSector(sx: number, sz: number): Sector {
    const seed = getSectorSeed(sx, sz, this.worldSeed);
    const floor = getFloorAtPosition(sx * SECTOR_SIZE, sz * SECTOR_SIZE);
    const rand = () => hash2D(sx, sz, seed++);
    
    const distFromOrigin = Math.sqrt(sx * sx + sz * sz);
    let type: SectorType;
    
    if (distFromOrigin < 2) {
      type = rand() < 0.5 ? SectorType.Empty : SectorType.OreField;
    } else {
      const roll = rand();
      if (roll < 0.35) type = SectorType.Empty;
      else if (roll < 0.55) type = SectorType.OreField;
      else if (roll < 0.75) type = SectorType.EnemyOutpost;
      else if (roll < 0.88) type = SectorType.EnemyBase;
      else if (roll < 0.95) type = SectorType.Ruins;
      else type = SectorType.Bunker;
    }
    
    return { x: sx, z: sz, type, seed, floor, discovered: false, cleared: false, claimed: false };
  }
  
  discoverRadius(worldX: number, worldZ: number, radius: number) {
    // Reveal fog of war in radius
  }
}
```

---

## 10. Entity Manager — AI & Spawning

```typescript
export interface Enemy {
  id: string;
  body: CANNON.Body;
  mesh: THREE.Group;
  stats: EnemyStats;
  ai: EnemyAI;
  isAlive: boolean;
}

export class EntityManager {
  private enemies: Map<string, Enemy> = new Map();
  private spatialHash: SpatialHash;
  
  onEnemyKilled: (enemy: Enemy) => void;
  onPlayerDamaged: (damage: number) => void;
  
  update(playerPos: THREE.Vector3, dt: number) {
    const sector = this.sectorManager.getSector(playerPos.x, playerPos.z);
    
    // Spawn enemies if sector has spawns
    if (sector.enemySpawns && !sector.cleared) {
      for (const spawn of sector.enemySpawns) {
        this.updateSpawner(spawn, playerPos, dt);
      }
    }
    
    // Update all enemy AI
    for (const enemy of this.enemies.values()) {
      if (!enemy.isAlive) continue;
      enemy.ai.update(playerPos, dt);
      this.spatialHash.update(enemy.id, enemy.body.position.x, enemy.body.position.z);
    }
  }
  
  getEnemiesInRadius(x: number, z: number, radius: number): Enemy[] {
    return this.spatialHash.queryRadius(x, z, radius)
      .map(id => this.enemies.get(id))
      .filter(e => e?.isAlive) as Enemy[];
  }
}
```

---

## 11. Procedural Enemy Generator

```typescript
export function generateEnemyStats(tier: number, seed: number): EnemyStats {
  const rand = (offset: number) => hash2D(tier, seed, offset);
  return {
    tier,
    maxHP: Math.floor((50 + tier * 75) * (0.8 + rand(0) * 0.4)),
    damage: Math.floor((5 + tier * 8) * (0.8 + rand(1) * 0.4)),
    speed: (8 + tier * 1.5) * (0.9 + rand(2) * 0.2),
    attackRange: 3 + tier * 0.5 + rand(3) * 2,
    scoreValue: 100 * tier,
  };
}

export function generateEnemyMesh(tier: number, seed: number): THREE.Group {
  const group = new THREE.Group();
  const colors = [0x8B4513, 0x666666, 0x444444, 0x882222, 0x111111];
  const color = colors[Math.min(tier - 1, colors.length - 1)];
  const mat = new THREE.MeshStandardMaterial({ color, metalness: 0.3 + tier * 0.1 });
  const scale = 1 + tier * 0.3;
  
  if (tier <= 2) {
    // Humanoid raider
    group.add(new THREE.Mesh(new THREE.CylinderGeometry(0.5 * scale, 0.3 * scale, 2 * scale, 8), mat));
  } else {
    // Mechanical drone/tank
    group.add(new THREE.Mesh(new THREE.BoxGeometry(1.5 * scale, 1 * scale, 2 * scale), mat));
  }
  return group;
}
```

---

## 12. Enemy Bases & Outposts

```typescript
export enum EnemyStructureType {
  Barracks = 'barracks',
  Factory = 'factory',
  Turret = 'turret',
  CommandCenter = 'command',
  Wall = 'wall',
}

export function generateEnemyBase(sector: Sector): EnemyStructure[] {
  const structures: EnemyStructure[] = [];
  const centerX = sector.x * SECTOR_SIZE + SECTOR_SIZE / 2;
  const centerZ = sector.z * SECTOR_SIZE + SECTOR_SIZE / 2;
  
  // Command center at base center (must destroy to clear sector)
  structures.push(createEnemyStructure(
    EnemyStructureType.CommandCenter,
    new THREE.Vector3(centerX, 0, centerZ),
    sector.floor, true
  ));
  
  // Surrounding structures
  const count = 4 + sector.floor;
  for (let i = 0; i < count; i++) {
    const angle = (i / count) * Math.PI * 2;
    const dist = 40 + sector.floor * 5;
    structures.push(createEnemyStructure(
      randomStructureType(),
      new THREE.Vector3(centerX + Math.cos(angle) * dist, 0, centerZ + Math.sin(angle) * dist),
      sector.floor, false
    ));
  }
  
  return structures;
}
```

---

## 13. Weapons — VLMS Missiles

### Auto-Targeting Parabolic Arc Missiles

```typescript
export class VLMSSystem {
  private missiles: Map<string, Missile> = new Map();
  private salvoSize = 4;
  private missileSpeed = 80;
  private baseDamage = 50;
  private aoeRadius = 8;
  private maxRange = 150;
  
  fire(launchPos: THREE.Vector3, ammoCount: number): number {
    const targets = this.entityManager.getEnemiesInRadius(launchPos.x, launchPos.z, this.maxRange);
    if (targets.length === 0) return 0;
    
    targets.sort((a, b) => distTo(a, launchPos) - distTo(b, launchPos));
    const count = Math.min(this.salvoSize, ammoCount, targets.length);
    
    for (let i = 0; i < count; i++) {
      setTimeout(() => this.launchMissile(launchPos, targets[i % targets.length].position), i * 100);
    }
    return count;
  }
  
  update(dt: number) {
    for (const missile of this.missiles.values()) {
      missile.elapsed += dt;
      const t = missile.elapsed / missile.flightTime;
      
      if (t >= 1) {
        this.onImpact(missile.targetPos, missile.damage, missile.aoeRadius);
        this.missiles.delete(missile.id);
        continue;
      }
      
      // Parabolic arc
      const pos = new THREE.Vector3().lerpVectors(missile.startPos, missile.targetPos, t);
      pos.y += Math.sin(t * Math.PI) * missile.arcHeight;
      missile.mesh.position.copy(pos);
    }
  }
}
```

---

## 14. Weapons — Orbital Strike

### Helldivers-Style GPS Beacon System

Unlocked by Orbital Command. Required for destroying underground bunkers.

```typescript
export enum OrbitalStrikeType {
  Railgun = 'railgun',       // Single high-damage (5000 dmg)
  Barrage = 'barrage',       // 12 smaller impacts
  Napalm = 'napalm',         // Area denial
  EMP = 'emp',               // Disables vehicles
}

const STRIKE_CONFIGS = {
  railgun: { damage: 5000, radius: 5, impactCount: 1, delay: 3, cooldown: 60, keybind: '1' },
  barrage: { damage: 500, radius: 8, impactCount: 12, delay: 4, cooldown: 90, keybind: '2' },
  napalm: { damage: 200, radius: 25, impactCount: 1, delay: 5, cooldown: 120, keybind: '3' },
  emp: { damage: 100, radius: 40, impactCount: 1, delay: 2, cooldown: 45, keybind: '4' },
};

export class OrbitalStrikeSystem {
  private cooldowns: Map<OrbitalStrikeType, number> = new Map();
  private isUnlocked = false;
  
  throwBeacon(playerPos: THREE.Vector3, throwDirection: THREE.Vector3, throwForce: number) {
    if (!this.isReady(this.selectedStrike)) return;
    
    const landingPos = playerPos.clone().add(throwDirection.multiplyScalar(throwForce * 2));
    const beacon = this.createBeacon(landingPos, this.selectedStrike);
    
    this.cooldowns.set(this.selectedStrike, STRIKE_CONFIGS[this.selectedStrike].cooldown);
    setTimeout(() => this.executeStrike(beacon), STRIKE_CONFIGS[this.selectedStrike].delay * 1000);
  }
}
```

---

## 15. Explosion System

### InstancedMesh Particles

```typescript
export const EXPLOSION_CONFIGS = {
  missile: { particleCount: 30, lifetime: 0.8, startScale: 0.5, endScale: 3, startColor: 0xFFAA00, endColor: 0xFF2200 },
  orbital_railgun: { particleCount: 100, lifetime: 2, startScale: 2, endScale: 20, startColor: 0xFFFFFF, endColor: 0x4488FF },
  orbital_barrage: { particleCount: 50, lifetime: 1.2, startScale: 1, endScale: 8 },
  structure_destroy: { particleCount: 80, lifetime: 1.5, startScale: 1, endScale: 5, startColor: 0x888888, endColor: 0x222222 },
};

class ExplosionSystem {
  private particles: InstancedMesh;
  private activeExplosions: Explosion[] = [];
  // Pool and reuse particles for performance
}
```

---

## 16. Base Building System

### Linear Tech Chain

```
Beacon → EnergyPylon → Refinery → Radar → DroneFactory → Helipad → TankFactory → PerimeterDefense → OrbitalCommand
```

```typescript
export enum BuildingType {
  Beacon = 'beacon',
  EnergyPylon = 'energy_pylon',
  Refinery = 'refinery',
  Radar = 'radar',
  DroneFactory = 'drone_factory',
  Helipad = 'helipad',
  TankFactory = 'tank_factory',
  PerimeterDefense = 'perimeter_defense',
  OrbitalCommand = 'orbital_command',
}

export const BUILD_ORDER: BuildingType[] = [
  BuildingType.Beacon,
  BuildingType.EnergyPylon,
  BuildingType.Refinery,
  BuildingType.Radar,
  BuildingType.DroneFactory,
  BuildingType.Helipad,
  BuildingType.TankFactory,
  BuildingType.PerimeterDefense,
  BuildingType.OrbitalCommand,
];

export const BUILDING_CONFIGS: Record<BuildingType, BuildingConfig> = {
  beacon: { name: 'Command Beacon', oreCost: 100, energyCost: 0, buildTime: 5, hp: 1000, unlocks: ['Base placement', 'Harvester healing'] },
  energy_pylon: { name: 'Energy Pylon', oreCost: 150, energyCost: 0, buildTime: 8, hp: 500, unlocks: ['+100 energy capacity', '+2/sec regen'] },
  refinery: { name: 'Ore Refinery', oreCost: 300, energyCost: 50, buildTime: 15, hp: 1500, unlocks: ['Ore deposit', 'Budget conversion'] },
  radar: { name: 'Radar Array', oreCost: 400, energyCost: 100, buildTime: 20, hp: 800, unlocks: ['Tactical map', 'Fog of war reveal', 'Unit commands'] },
  drone_factory: { name: 'Drone Factory', oreCost: 500, energyCost: 150, buildTime: 25, hp: 1200, unlocks: ['Scout Drone', 'Attack Drone'] },
  helipad: { name: 'Helipad', oreCost: 700, energyCost: 200, buildTime: 30, hp: 1000, unlocks: ['Gunship', 'Transport Heli'] },
  tank_factory: { name: 'Tank Factory', oreCost: 900, energyCost: 300, buildTime: 40, hp: 2000, unlocks: ['Light Tank', 'Heavy Tank', 'Artillery'] },
  perimeter_defense: { name: 'Perimeter Defense', oreCost: 600, energyCost: 250, buildTime: 20, hp: 800, unlocks: ['Auto-turrets around base'] },
  orbital_command: { name: 'Orbital Command', oreCost: 2000, energyCost: 500, buildTime: 60, hp: 3000, unlocks: ['Railgun', 'Barrage', 'Napalm', 'EMP strikes'] },
};

class BaseManager {
  canBuild(type: BuildingType): { allowed: boolean; reason?: string } {
    const buildIndex = BUILD_ORDER.indexOf(type);
    if (buildIndex > 0) {
      const required = BUILD_ORDER[buildIndex - 1];
      if (!this.hasCompleted(required)) return { allowed: false, reason: `Requires ${BUILDING_CONFIGS[required].name}` };
    }
    if (!this.resourceManager.canAfford(config.oreCost, config.energyCost)) return { allowed: false, reason: 'Insufficient resources' };
    return { allowed: true };
  }
}
```

---

## 17. Unit Production & Command

### Unit Types

```typescript
export enum UnitType {
  ScoutDrone = 'scout_drone',
  AttackDrone = 'attack_drone',
  Gunship = 'gunship',
  TransportHeli = 'transport_heli',
  LightTank = 'light_tank',
  HeavyTank = 'heavy_tank',
  Artillery = 'artillery',
}

export const UNIT_CONFIGS: Record<UnitType, UnitConfig> = {
  scout_drone: { oreCost: 50, energyCost: 20, buildTime: 5, hp: 100, damage: 10, attackRange: 20, moveSpeed: 30, isFlying: true, factory: BuildingType.DroneFactory },
  attack_drone: { oreCost: 100, energyCost: 40, buildTime: 8, hp: 200, damage: 25, attackRange: 25, moveSpeed: 25, isFlying: true, factory: BuildingType.DroneFactory },
  gunship: { oreCost: 300, energyCost: 100, buildTime: 20, hp: 600, damage: 50, attackRange: 40, moveSpeed: 20, isFlying: true, factory: BuildingType.Helipad },
  transport_heli: { oreCost: 200, energyCost: 60, buildTime: 15, hp: 400, damage: 15, moveSpeed: 35, isFlying: true, factory: BuildingType.Helipad },
  light_tank: { oreCost: 200, energyCost: 50, buildTime: 15, hp: 500, damage: 40, attackRange: 30, moveSpeed: 15, isFlying: false, factory: BuildingType.TankFactory },
  heavy_tank: { oreCost: 500, energyCost: 150, buildTime: 30, hp: 1500, damage: 100, attackRange: 35, moveSpeed: 8, isFlying: false, factory: BuildingType.TankFactory },
  artillery: { oreCost: 400, energyCost: 120, buildTime: 25, hp: 300, damage: 200, attackRange: 80, moveSpeed: 6, isFlying: false, factory: BuildingType.TankFactory },
};
```

### Unit Manager

```typescript
export enum UnitBehavior { Guard = 'guard', Defensive = 'defensive', Aggressive = 'aggressive' }

export interface UnitOrder {
  type: 'move' | 'attack' | 'patrol' | 'guard';
  position?: THREE.Vector3;
  targetId?: string;
}

class UnitManager {
  private units: Map<string, PlayerUnit> = new Map();
  private groups: Map<number, Set<string>> = new Map();
  
  orderUnit(unitId: string, order: UnitOrder) { /* execute order */ }
  orderGroup(groupId: number, order: UnitOrder) { /* order all units in group */ }
  createGroup(unitIds: string[]): number { /* create numbered group */ }
  setBehavior(unitId: string, behavior: UnitBehavior) { /* change AI stance */ }
}
```

---

## 18. Tactical Map Overlay

### RTS Command Interface (Requires Radar)

```typescript
class TacticalMap {
  private canvas: HTMLCanvasElement;
  private isVisible = false;
  private mapScale = 2;
  private selectedUnits: Set<string> = new Set();
  
  toggle() {
    this.isVisible = !this.isVisible;
    this.canvas.style.display = this.isVisible ? 'block' : 'none';
  }
  
  render() {
    // Draw fog of war (undiscovered = black)
    // Draw terrain (discovered sectors)
    // Draw ore zones (green circles)
    // Draw enemy bases (red)
    // Draw player base (blue)
    // Draw units (green triangles for player, red for enemy)
    // Draw resource beacons (yellow markers)
    // Draw selection box if dragging
  }
  
  onMouseDown(e: MouseEvent) {
    if (e.button === 0) this.startSelection(e);
  }
  
  onRightClick(e: MouseEvent) {
    const worldPos = this.screenToWorld(e.offsetX, e.offsetY);
    const enemyAtPos = this.getEnemyAt(worldPos);
    
    if (enemyAtPos) {
      this.unitManager.orderGroup(this.selectedGroup, { type: 'attack', targetId: enemyAtPos.id });
    } else {
      this.unitManager.orderGroup(this.selectedGroup, { type: 'move', position: worldPos });
    }
  }
}
```

---

## 19. Resource Beacons

### Navigation Aids (Max 3, FIFO)

```typescript
class ResourceBeaconSystem {
  private beacons: ResourceBeacon[] = [];
  private maxBeacons = 3;
  
  placeBeacon(position: THREE.Vector3) {
    if (this.beacons.length >= this.maxBeacons) {
      const oldest = this.beacons.shift()!;
      this.removeBeaconVisual(oldest);
    }
    
    const beacon: ResourceBeacon = {
      id: `beacon_${Date.now()}`,
      position: position.clone(),
      mesh: this.createBeaconMesh(),
    };
    beacon.mesh.position.copy(position);
    this.scene.add(beacon.mesh);
    this.beacons.push(beacon);
  }
  
  getBeaconDirections(fromPos: THREE.Vector3): BeaconDirection[] {
    return this.beacons.map(b => ({
      id: b.id,
      direction: b.position.clone().sub(fromPos).normalize(),
      distance: b.position.distanceTo(fromPos),
    }));
  }
}
```

---

## 20. Difficulty Scaling (Floors)

### ZeroByte Position-Based Difficulty

Floor increases automatically with distance from origin. Higher floor = stronger enemies, richer ore, more enemy bases.

```typescript
// Already defined in desert-zb.ts
export function getFloorAtPosition(x: number, z: number): number {
  const distance = Math.sqrt(x * x + z * z);
  return Math.floor(distance / 1000);
}

// Applied in sector generation
const floor = getFloorAtPosition(sx * SECTOR_SIZE, sz * SECTOR_SIZE);
const enemyTier = Math.min(5, 1 + floor + Math.floor(rand() * 2));
const oreRichness = Math.min(5, 1 + Math.floor(rand() * 3) + Math.floor(floor * 0.5));
```

---

## 21. Save System

```typescript
interface SaveData {
  version: string;
  worldSeed: number;
  timestamp: number;
  harvester: { position: [number, number, number]; hp: number; ore: number; ammo: number };
  resources: ResourceState;
  buildings: { type: BuildingType; position: [number, number, number]; hp: number; isComplete: boolean }[];
  units: { type: UnitType; position: [number, number, number]; hp: number; groupId: number | null }[];
  discoveredSectors: string[];
  clearedSectors: string[];
  beacons: [number, number, number][];
  objectives: { id: string; completed: boolean }[];
}

class SaveManager {
  save(slot: number) {
    const data = this.serialize();
    localStorage.setItem(`desert_harvest_${slot}`, JSON.stringify(data));
  }
  
  load(slot: number): boolean {
    const json = localStorage.getItem(`desert_harvest_${slot}`);
    if (!json) return false;
    const data = JSON.parse(json) as SaveData;
    this.deserialize(data);
    return true;
  }
}
```

---

## 22. Procedural Audio

### All Sounds Generated at Runtime

```typescript
const AUDIO_CONFIG = {
  engine_idle: { type: 'noise', frequency: 60, gain: 0.1 },
  engine_drive: { type: 'noise', frequency: 120, gain: 0.2 },
  missile_launch: { type: 'sweep', startFreq: 800, endFreq: 200, duration: 0.3 },
  missile_impact: { type: 'explosion', duration: 0.5 },
  orbital_incoming: { type: 'sweep', startFreq: 2000, endFreq: 100, duration: 2 },
  orbital_impact: { type: 'explosion', duration: 1.5, gain: 0.8 },
  ore_collect: { type: 'tone', frequency: 440, duration: 0.1 },
  building_complete: { type: 'fanfare', notes: [523, 659, 784], duration: 0.5 },
  unit_ready: { type: 'tone', frequency: 880, duration: 0.2 },
  enemy_death: { type: 'noise', duration: 0.3 },
};
```

---

## 23. HUD & UI

### Top-Left Status Panel

```html
<div id="hud">
  HP: 4500/5000<br>
  ORE: 234/500 | Budget: 1,250<br>
  Energy: 85/100<br>
  Floor: 3 | Sector: Ore Field<br>
  Enemies: 8 | Kills: 127
</div>
```

### Top-Right Missile HUD

```
═ VLMS ═
Ammo: 48/64
[████████░░]
READY
```

### Top-Center Radar (140×140 Canvas)

- Green ring border
- Forward indicator (harvester heading)
- Orange arrow → nearest enemy
- Yellow markers → resource beacons (max 3)
- Blue arrow → base
- Red dots → enemies

### Orbital Strike Selection (Bottom)

```
[1] Railgun (READY)  [2] Barrage (45s)  [3] Napalm (READY)  [4] EMP (READY)
```

### Tactical Map (M key, requires Radar)

- Full-screen overlay
- Fog of war
- Click to select units
- Right-click to issue orders
- Unit behavior buttons (Guard/Defensive/Aggressive)

---

## 24. WebGPU Acceleration (Optional)

### GPU Heightfield Generation

```wgsl
@group(0) @binding(0) var<storage, read_write> heights: array<f32>;
@group(0) @binding(1) var<uniform> params: HeightParams;

@compute @workgroup_size(64)
fn main(@builtin(global_invocation_id) id: vec3<u32>) {
    let idx = id.x;
    let x = f32(idx % params.width) * params.elementSize + params.offsetX;
    let z = f32(idx / params.width) * params.elementSize + params.offsetZ;
    heights[idx] = desertHeight(x, z);
}
```

---

## 25. Game Loop

### index.tsx Main Entry

```typescript
// 1. Create Demo instance
const app = new Demo({ shadows: true });
const world = app.getWorld();
world.gravity.set(0, -10, 0);

// 2. Create harvester
const harvester = createHarvester(world);

// 3. Initialize subsystems
const chunkManager = new ChunkManager(world, app.scene, worldSeed);
const sectorManager = new SectorManager(worldSeed);
const resourceManager = new ResourceManager();
const entityManager = new EntityManager(app.scene, world, sectorManager);
const baseManager = new BaseManager(app.scene, world, resourceManager);
const unitManager = new UnitManager(app.scene, world, entityManager);
const vlmsSystem = new VLMSSystem(app.scene, entityManager, onExplosion);
const orbitalStrike = new OrbitalStrikeSystem(app.scene);
const tacticalMap = new TacticalMap(sectorManager, unitManager, entityManager, baseManager);
const beaconSystem = new ResourceBeaconSystem(app.scene);
const weatherManager = new WeatherManager(app.scene);
const saveManager = new SaveManager();

// 4. Physics postStep (60Hz)
world.addEventListener('postStep', () => {
  chunkManager.update(harvesterPos);
  entityManager.update(harvesterPos, dt);
  resourceManager.update(dt, harvesterVelocity);
  baseManager.update(dt);
  unitManager.update(dt);
  harvesterCamera.update(app.camera, harvester);
  updateHUD();
});

// 5. Render loop
function renderLoop() {
  requestAnimationFrame(renderLoop);
  vlmsSystem.update(dt);
  orbitalStrike.update(dt);
  explosionSystem.update(dt);
  weatherManager.update(dt);
  tacticalMap.render();
  app.renderer.render(app.scene, app.camera);
}
```

---

## 26. Cheat / Debug System

**Activation**: Type "IDKFA" → toggles cheat menu.

**Cheats:**
- God Mode (invincibility)
- Infinite Ammo
- Infinite Resources
- All Buildings Unlocked
- Instant Build
- Reveal Map
- Kill All Enemies (current sector)
- Teleport (click on tactical map)
- Spawn Units
- Force Weather/Time

---

## Implementation Order

### Phase 1 — Drivable Harvester
1. Project scaffold
2. Demo.ts + utils.ts
3. Harvester creation + controls
4. Third-person camera
5. **Test**: WASD driving on flat plane

### Phase 2 — Infinite Desert
6. desert-zb.ts (hashing)
7. desert-terrain.ts (dune generation)
8. chunk-manager.ts
9. **Test**: Drive over procedural desert

### Phase 3 — Resource Collection
10. resource-manager.ts
11. Ore zones in sector-manager
12. Collection when stationary in zone
13. **Test**: Collect ore, see cargo fill

### Phase 4 — Base Building
14. building-config.ts
15. BaseBuilding.ts + BaseManager.ts
16. Beacon → Pylon → Refinery chain
17. Ore deposit at refinery
18. **Test**: Build base, deposit ore, gain budget

### Phase 5 — Combat
19. enemy-generator.ts
20. entity-manager.ts (spawning + AI)
21. VLMSSystem.ts (missiles)
22. explosion-system.ts
23. **Test**: Enemies spawn, harvester fights back

### Phase 6 — Unit Production
24. unit-config.ts
25. UnitFactory.ts + UnitManager.ts
26. Drone/Helipad/Tank factories
27. **Test**: Build factory, spawn units

### Phase 7 — Tactical Map
28. Radar building enables map
29. TacticalMap.ts
30. Fog of war
31. Unit selection + commands
32. **Test**: Open map, select units, issue orders

### Phase 8 — Endgame
33. OrbitalCommand building
34. OrbitalStrike.ts
35. Bunker sector type
36. Victory conditions
37. **Test**: Unlock orbital, destroy bunker

### Phase 9 — Polish
38. Weather/time visuals
39. Procedural audio
40. Save system
41. Resource beacons
42. Cheat system
43. **Test**: Full game loop

---

## Critical Patterns to Preserve

### CDN Imports
```typescript
// @ts-ignore
import * as CANNON from 'https://unpkg.com/cannon-es@0.20.0/dist/cannon-es.js';
// @ts-ignore
import * as THREE from 'https://unpkg.com/three@0.122.0/build/three.module.js';
```

### Mesh Pooling
Always pool missiles, particles, enemy meshes. Never `new THREE.Mesh()` in hot loops.

### Spatial Hashing
O(1) range queries for missile targeting, enemy proximity. Never iterate all entities.

### Cached Vectors
Pre-allocate vectors for per-frame calculations. Zero allocation in update loops.

### Physics/Render Separation
Physics: fixed 60Hz via `world.step()`. Render: display refresh via `requestAnimationFrame`. `postStep` bridges them.
