# DYNASTY-Z

## Description

DYNASTY-Z is a third-person action game built with React, Three.js (@react-three/fiber and @react-three/drei), featuring "inverted bullet-hell" gameplay where the player deflects enemy projectiles back at them. The core fantasy is being an unstoppable melee warrior who can dispatch massive armies through skillful blocking and ricochet mechanics, inspired by Metal Gear Rising: Revengeance's parry system, Dynasty Warriors' crowd combat, and Vampire Survivors' progression mechanics.

The game uses placeholder geometry (cubes for player, spheres for enemies) designed for future replacement with animated GLB/GLTF models. A Fabric.js 2D overlay provides UI elements including health bars, XP indicators, and ability cooldowns.

**Core Gameplay Loop:** Enter arena → Enemies spawn in waves → Deflect projectiles back at enemies → Gain XP → Level up abilities → Face harder waves → Repeat.

---

## Functionality

### 1. Player System

#### 1.1 Player Entity
- **Visual:** Cyan-colored `<Box>` geometry (1×2×1 units) representing a humanoid placeholder
- **Position:** Starts at world origin `(0, 1, 0)` (y=1 keeps player grounded on floor plane)
- **Collision:** Bounding box hitbox matching mesh dimensions
- **Health:** Starts at 100 HP, displayed on HUD
- **Invulnerability:** Brief flash effect (opacity toggle) when hit

#### 1.2 Movement Controls (Twin-Stick 3D)
```
WASD / Left Stick: Movement (relative to camera facing)
  W = Forward, S = Backward, A = Strafe Left, D = Strafe Right
Mouse / Right Stick: Camera orbit (look direction)
Movement Speed: 8 units/second (base)
Sprint: Hold Shift = 14 units/second
```

#### 1.3 Camera System
- **Type:** Third-person orbit camera locked behind player
- **Distance:** 12 units behind, 6 units above player
- **Controls:** Mouse X rotates camera horizontally around player
- **Vertical Look:** Mouse Y tilts camera 10° to 60° (clamped)
- **Smoothing:** Lerp factor 0.1 for smooth follow
- **Implementation:** Custom camera rig, not OrbitControls (player-centric)

### 2. Deflection System (Core Mechanic)

#### 2.1 Automatic Deflection (Passive)
- **Trigger:** Any projectile entering player's deflection radius (2.5 units)
- **Effect:** Projectile bounces in random direction (random unit vector)
- **Visual:** Blue spark particle burst at deflection point
- **Audio Cue:** "ting" sound effect placeholder
- **No Input Required:** Always active

#### 2.2 Timed Deflection (Active - Parry)
- **Input:** Left Mouse Button / Gamepad X pressed
- **Window:** 200ms parry window
- **Trigger:** Projectile enters deflection radius DURING parry window
- **Effect:** Projectile reflects toward nearest enemy (auto-lock)
- **Visual:** Golden flash + larger particle burst
- **Damage Bonus:** Reflected projectile deals 2× damage
- **Audio Cue:** Satisfying "clang" sound

#### 2.3 Held Deflection (Auto-Lock Mode)
- **Input:** Left Mouse Button / Gamepad X held down
- **Effect:** ALL projectiles in deflection radius auto-target nearest enemy
- **Targeting Priority:** 
  1. Closest enemy in front (±45° cone)
  2. Closest enemy overall
  3. Random direction if no enemies
- **Trade-off:** Cannot perform melee attack while holding
- **Visual:** Faint targeting reticle appears on locked enemy

#### 2.4 Deflection Physics
```javascript
// Projectile reflection calculation
const deflectDirection = (projectile, player, targetEnemy, deflectType) => {
  switch(deflectType) {
    case 'passive': 
      return randomUnitVector();
    case 'parry':
    case 'held':
      const toEnemy = targetEnemy.position.clone().sub(player.position).normalize();
      return toEnemy;
  }
}
```

### 3. Combat System

#### 3.1 Melee Attack (Slash)
- **Input:** Right Mouse Button / Gamepad Y
- **Animation:** Quick forward thrust (placeholder: box scales forward 3× width)
- **Hitbox:** Box (3×2×4 units) extending in player's facing direction
- **Damage:** 25 base damage
- **Range:** 4 units forward from player center
- **Cooldown:** 400ms between attacks
- **Effect:** Destroys all enemies within hitbox
- **Visual:** Red slash trail effect (simple plane with gradient texture)

#### 3.2 Dodge/Dash
- **Input:** Space + Direction / Gamepad B + Left Stick
- **Distance:** 6 units in input direction
- **Duration:** 150ms total movement time
- **I-Frames:** Full invulnerability during dash (150ms)
- **Cooldown:** 500ms between dodges
- **Visual:** Afterimage effect (3 fading ghost copies of player mesh)
- **No Direction Input:** Dash backward relative to camera

#### 3.3 Damage Calculation
```javascript
// Player takes damage
const playerTakeDamage = (amount) => {
  if (isInvulnerable || isDashing) return; // I-frames check
  health -= amount;
  isInvulnerable = true;
  setTimeout(() => isInvulnerable = false, 500); // Brief mercy invuln
  if (health <= 0) triggerGameOver();
}

// Enemy takes damage  
const enemyTakeDamage = (enemy, amount, source) => {
  enemy.health -= amount;
  if (enemy.health <= 0) {
    destroyEnemy(enemy);
    grantXP(enemy.xpValue);
  }
}
```

### 4. Enemy System

#### 4.1 Base Enemy Entity
- **Visual:** Red `<Sphere>` geometry (radius 0.8 units)
- **Health:** 30 HP (base)
- **XP Value:** 10 XP (base)
- **Collision:** Sphere collider matching mesh
- **AI State Machine:** Idle → Aware → Attacking

#### 4.2 Enemy Types

| Type | Visual | Health | Projectile | Fire Rate | XP | Behavior |
|------|--------|--------|------------|-----------|----|----|
| Grunt | Red sphere | 30 | Single shot | 2s | 10 | Direct aim at player |
| Rapid | Orange sphere | 20 | Burst (3) | 1.5s | 15 | Strafe while shooting |
| Heavy | Dark red sphere (1.2× size) | 80 | Slow large shot | 3s | 25 | Stationary, tracks player |
| Sniper | Purple sphere | 15 | Fast precise shot | 4s | 20 | Maintains distance |

#### 4.3 Enemy AI Behavior
```javascript
const enemyBehavior = {
  detectionRadius: 30, // units - becomes "aware" of player
  attackRadius: 25,    // units - begins shooting
  
  states: {
    idle: {
      behavior: 'Wander randomly within spawn area',
      transition: 'Player enters detectionRadius → aware'
    },
    aware: {
      behavior: 'Turn to face player, begin approach',
      transition: 'Player in attackRadius → attacking'
    },
    attacking: {
      behavior: 'Fire projectiles at player position, slight leading',
      transition: 'Player exits detectionRadius → idle'
    }
  }
}
```

#### 4.4 Enemy Spawning System
- **Wave-Based:** Enemies spawn in waves with increasing difficulty
- **Spawn Points:** 8 spawn locations in circle around arena (radius 40 units)
- **Wave Composition:**
```javascript
const waveConfig = {
  wave1: { grunt: 5 },
  wave2: { grunt: 8, rapid: 2 },
  wave3: { grunt: 10, rapid: 4, heavy: 1 },
  wave4: { grunt: 12, rapid: 6, heavy: 2, sniper: 2 },
  // Pattern: +3 grunt, +2 rapid, +1 heavy, +1 sniper per wave
  waveN: (n) => ({
    grunt: 5 + (n-1) * 3,
    rapid: Math.max(0, (n-2) * 2),
    heavy: Math.max(0, n - 2),
    sniper: Math.max(0, n - 3)
  })
}
```
- **Spawn Rate:** Staggered, 0.3s between each enemy spawn
- **Next Wave:** Triggers when all enemies destroyed + 3s delay

### 5. Projectile System

#### 5.1 Projectile Properties
```javascript
const projectileTypes = {
  standard: {
    speed: 15,        // units/second
    damage: 10,
    size: 0.3,        // radius
    color: 'yellow',
    lifetime: 5       // seconds before auto-destroy
  },
  burst: {
    speed: 18,
    damage: 7,
    size: 0.2,
    color: 'orange',
    lifetime: 4
  },
  heavy: {
    speed: 8,
    damage: 25,
    size: 0.6,
    color: 'darkred',
    lifetime: 6
  },
  sniper: {
    speed: 35,
    damage: 30,
    size: 0.15,
    color: 'purple',
    lifetime: 3
  }
}
```

#### 5.2 Projectile Collision
- **Vs Player:** Trigger damage if not deflected, destroy projectile
- **Vs Enemy:** Deal projectile damage to enemy, destroy projectile
- **Vs World:** Destroy on hitting arena boundaries
- **Vs Projectile:** Pass through (no projectile-projectile collision)

#### 5.3 Projectile Pooling
- **Pool Size:** 200 projectiles (reuse destroyed projectiles)
- **Implementation:** Array of inactive projectiles, activate on fire, deactivate on collision

### 6. Progression System

#### 6.1 XP and Leveling
```javascript
const levelingSystem = {
  xpToLevel: (level) => 100 * level, // Level 2 = 200 XP, Level 3 = 300 XP, etc.
  maxLevel: 20,
  
  onLevelUp: (newLevel) => {
    pauseGame();
    showUpgradeSelection(3); // Offer 3 random upgrades
    // Game resumes after selection
  }
}
```

#### 6.2 Upgrade Types
```javascript
const upgrades = {
  // Stat Upgrades (stackable)
  health_boost: { effect: '+20 Max HP', maxStacks: 5 },
  speed_boost: { effect: '+10% Movement Speed', maxStacks: 5 },
  deflect_radius: { effect: '+0.5 Deflection Radius', maxStacks: 3 },
  parry_window: { effect: '+50ms Parry Window', maxStacks: 3 },
  
  // Attack Upgrades (stackable)
  slash_damage: { effect: '+10 Slash Damage', maxStacks: 5 },
  slash_width: { effect: '+1 Slash Width', maxStacks: 3 },
  slash_range: { effect: '+1 Slash Range', maxStacks: 3 },
  
  // Special Upgrades (unique - only one allowed)
  ricochet: { effect: 'Deflected projectiles bounce to 2nd enemy', unique: true },
  explosive_parry: { effect: 'Perfect parries create AOE explosion', unique: true },
  dash_damage: { effect: 'Dash damages enemies in path', unique: true },
  vampiric: { effect: 'Heal 5 HP per enemy killed', unique: true },
  bullet_time: { effect: 'Parry window slows time 50% for 0.5s', unique: true }
}
```

#### 6.3 Upgrade Selection UI
- **Display:** 3 cards shown on level up
- **Interaction:** Click/press 1-2-3 to select
- **Contents:** Upgrade name, description, current stack count if applicable
- **Reroll:** Press R to reroll options (1 free reroll per level)

### 7. Game Arena

#### 7.1 Environment
- **Floor:** Large plane (100×100 units) with grid texture
- **Boundaries:** Invisible walls at edges (or visual low walls)
- **Lighting:** 
  - Ambient light (intensity 0.4)
  - Directional light from above-front (intensity 0.8)
- **Skybox:** Simple gradient (dark blue to black) or solid color

#### 7.2 Arena Hazards (Future Enhancement)
- Marked as extension point for obstacles, cover, elevation changes

### 8. User Interface (Fabric.js 2D Overlay)

#### 8.1 HUD Layout
```
┌─────────────────────────────────────────────────────────┐
│ [HP BAR████████░░]  100/100    Wave: 3    Enemies: 12   │
│ [XP BAR██████░░░░░░]  Level 5                           │
│                                                          │
│                                                          │
│                      [CROSSHAIR]                         │
│                                                          │
│                                                          │
│                                                          │
│                                     [DASH] ●●●          │
│ [SLASH COOLDOWN]                    [PARRY INDICATOR]   │
└─────────────────────────────────────────────────────────┘
```

#### 8.2 HUD Elements
- **Health Bar:** Top-left, red fill, shows current/max
- **XP Bar:** Below health, blue fill, shows progress to next level
- **Level Display:** Number next to XP bar
- **Wave Counter:** Top-center
- **Enemy Counter:** Shows remaining enemies in wave
- **Cooldown Indicators:** Bottom for slash and dash
- **Parry Indicator:** Flashes when in parry window
- **Crosshair:** Center screen, simple dot or small cross

#### 8.3 Game State Screens

**Pause Menu:**
```
PAUSED
[Resume]
[Restart]
[Quit to Menu]
```

**Game Over Screen:**
```
GAME OVER
Wave Reached: X
Enemies Defeated: Y
Final Level: Z
[Restart] [Main Menu]
```

**Level Up Screen:**
```
LEVEL UP!
Choose an Upgrade:
[Card 1] [Card 2] [Card 3]
Press R to Reroll (1 remaining)
```

---

## Technical Implementation

### 1. Project Structure
```
dynasty-z/
├── src/
│   ├── components/
│   │   ├── Game.jsx           # Main game component
│   │   ├── Player.jsx         # Player entity + controls
│   │   ├── Enemy.jsx          # Enemy entity component
│   │   ├── Projectile.jsx     # Projectile component
│   │   ├── Arena.jsx          # Floor, boundaries, lighting
│   │   ├── Camera.jsx         # Third-person camera rig
│   │   ├── Effects.jsx        # Particles, trails, flashes
│   │   └── UI/
│   │       ├── HUD.jsx        # Fabric.js canvas overlay
│   │       ├── HealthBar.jsx
│   │       ├── XPBar.jsx
│   │       ├── UpgradeScreen.jsx
│   │       └── GameOverScreen.jsx
│   ├── systems/
│   │   ├── useGameState.js    # Zustand store
│   │   ├── useInput.js        # Input handling hook
│   │   ├── useCollision.js    # Collision detection
│   │   ├── useSpawner.js      # Enemy wave spawning
│   │   └── useProgression.js  # XP and upgrades
│   ├── utils/
│   │   ├── math.js            # Vector helpers
│   │   ├── constants.js       # Game balance values
│   │   └── pooling.js         # Object pooling utilities
│   ├── App.jsx
│   └── index.jsx
├── public/
│   └── index.html
└── package.json
```

### 2. Dependencies
```json
{
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "@react-three/fiber": "^8.15.0",
    "@react-three/drei": "^9.88.0",
    "three": "^0.158.0",
    "zustand": "^4.4.0",
    "fabric": "^5.3.0"
  }
}
```

### 3. State Management (Zustand)
```javascript
const useGameStore = create((set, get) => ({
  // Game State
  gameState: 'menu', // 'menu' | 'playing' | 'paused' | 'levelup' | 'gameover'
  wave: 1,
  
  // Player State
  playerHealth: 100,
  playerMaxHealth: 100,
  playerPosition: [0, 1, 0],
  playerRotation: 0,
  isInvulnerable: false,
  isDashing: false,
  
  // Progression
  xp: 0,
  level: 1,
  upgrades: {},
  
  // Combat Stats (modified by upgrades)
  stats: {
    moveSpeed: 8,
    sprintSpeed: 14,
    deflectRadius: 2.5,
    parryWindow: 200,
    slashDamage: 25,
    slashWidth: 3,
    slashRange: 4,
    dashDistance: 6,
    dashCooldown: 500,
    slashCooldown: 400,
  },
  
  // Entities
  enemies: [],
  projectiles: [],
  
  // Actions
  damagePlayer: (amount) => {...},
  damageEnemy: (id, amount) => {...},
  spawnEnemy: (type, position) => {...},
  fireProjectile: (origin, direction, type) => {...},
  deflectProjectile: (id, newDirection, isParry) => {...},
  addXP: (amount) => {...},
  applyUpgrade: (upgradeId) => {...},
  // ... etc
}))
```

### 4. Core Game Loop
```javascript
// In Game.jsx using useFrame
useFrame((state, delta) => {
  if (gameState !== 'playing') return;
  
  // 1. Process Input
  updatePlayerMovement(delta);
  updateCameraPosition(delta);
  
  // 2. Update Entities
  updateEnemyAI(delta);
  updateProjectiles(delta);
  
  // 3. Check Collisions
  checkProjectileDeflections();
  checkProjectileHits();
  checkMeleeHits();
  
  // 4. Clean Up
  removeDeadEnemies();
  removeExpiredProjectiles();
  
  // 5. Check Wave Completion
  checkWaveStatus();
});
```

### 5. Collision Detection
```javascript
// Sphere-Sphere (projectile-player/enemy)
const sphereCollision = (a, b) => {
  const distance = a.position.distanceTo(b.position);
  return distance < (a.radius + b.radius);
};

// Box-Sphere (melee hitbox-enemy)
const boxSphereCollision = (box, sphere) => {
  // Find closest point on box to sphere center
  const closest = new THREE.Vector3(
    Math.max(box.min.x, Math.min(sphere.position.x, box.max.x)),
    Math.max(box.min.y, Math.min(sphere.position.y, box.max.y)),
    Math.max(box.min.z, Math.min(sphere.position.z, box.max.z))
  );
  const distance = closest.distanceTo(sphere.position);
  return distance < sphere.radius;
};

// Deflection zone check
const inDeflectionZone = (projectile, player, deflectRadius) => {
  return projectile.position.distanceTo(player.position) < deflectRadius;
};
```

### 6. Input System
```javascript
const useInput = () => {
  const keys = useRef({});
  const mouse = useRef({ x: 0, y: 0, left: false, right: false });
  
  useEffect(() => {
    const onKeyDown = (e) => keys.current[e.code] = true;
    const onKeyUp = (e) => keys.current[e.code] = false;
    const onMouseMove = (e) => {
      mouse.current.x = e.movementX;
      mouse.current.y = e.movementY;
    };
    const onMouseDown = (e) => {
      if (e.button === 0) mouse.current.left = true;
      if (e.button === 2) mouse.current.right = true;
    };
    const onMouseUp = (e) => {
      if (e.button === 0) mouse.current.left = false;
      if (e.button === 2) mouse.current.right = false;
    };
    
    // Pointer lock for FPS-style mouse control
    document.addEventListener('click', () => {
      document.body.requestPointerLock();
    });
    
    window.addEventListener('keydown', onKeyDown);
    window.addEventListener('keyup', onKeyUp);
    document.addEventListener('mousemove', onMouseMove);
    document.addEventListener('mousedown', onMouseDown);
    document.addEventListener('mouseup', onMouseUp);
    
    return () => { /* cleanup */ };
  }, []);
  
  return { keys, mouse };
};
```

### 7. Fabric.js HUD Integration
```javascript
// HUD.jsx
const HUD = () => {
  const canvasRef = useRef(null);
  const fabricRef = useRef(null);
  const { playerHealth, playerMaxHealth, xp, level, wave } = useGameStore();
  
  useEffect(() => {
    // Initialize Fabric canvas
    fabricRef.current = new fabric.Canvas(canvasRef.current, {
      selection: false,
      renderOnAddRemove: false,
    });
    
    // Create HUD elements
    createHealthBar(fabricRef.current);
    createXPBar(fabricRef.current);
    createWaveCounter(fabricRef.current);
    
    return () => fabricRef.current.dispose();
  }, []);
  
  // Update HUD on state changes
  useEffect(() => {
    updateHealthBar(fabricRef.current, playerHealth, playerMaxHealth);
  }, [playerHealth, playerMaxHealth]);
  
  return (
    <canvas
      ref={canvasRef}
      style={{
        position: 'absolute',
        top: 0,
        left: 0,
        pointerEvents: 'none',
        zIndex: 100,
      }}
      width={window.innerWidth}
      height={window.innerHeight}
    />
  );
};
```

---

## Data Models

### Player
```typescript
interface Player {
  position: Vector3;
  rotation: number;        // Y-axis rotation (facing direction)
  velocity: Vector3;
  health: number;
  maxHealth: number;
  isInvulnerable: boolean;
  isDashing: boolean;
  isParrying: boolean;
  parryStartTime: number;
  lastSlashTime: number;
  lastDashTime: number;
}
```

### Enemy
```typescript
interface Enemy {
  id: string;
  type: 'grunt' | 'rapid' | 'heavy' | 'sniper';
  position: Vector3;
  rotation: number;
  health: number;
  maxHealth: number;
  state: 'idle' | 'aware' | 'attacking';
  lastFireTime: number;
  xpValue: number;
}
```

### Projectile
```typescript
interface Projectile {
  id: string;
  type: 'standard' | 'burst' | 'heavy' | 'sniper';
  position: Vector3;
  velocity: Vector3;
  damage: number;
  ownerId: string;         // Enemy who fired it (null if deflected)
  isDeflected: boolean;
  isParryDeflected: boolean;
  spawnTime: number;
  lifetime: number;
}
```

### Upgrade
```typescript
interface Upgrade {
  id: string;
  name: string;
  description: string;
  effect: (stats: Stats) => Stats;
  isUnique: boolean;
  maxStacks: number;
  currentStacks: number;
}
```

### GameState
```typescript
interface GameState {
  status: 'menu' | 'playing' | 'paused' | 'levelup' | 'gameover';
  wave: number;
  enemiesRemaining: number;
  totalEnemiesDefeated: number;
  currentXP: number;
  currentLevel: number;
  activeUpgrades: Map<string, number>;  // upgradeId -> stack count
}
```

---

## Edge Cases and Error Handling

### Collision Edge Cases
- **Multiple projectiles hitting simultaneously:** Process all, apply damage once with mercy invulnerability
- **Dash through projectile:** I-frames prevent damage, projectile continues (not deflected)
- **Enemy at exact deflection radius:** Include (use `<=` comparison)
- **No enemies for auto-target:** Deflect in random direction
- **Player pushed outside arena:** Clamp position to boundaries

### State Edge Cases
- **Level up during dash:** Queue level up screen, show after dash completes
- **Wave complete during pause:** Wait for unpause to show wave transition
- **Zero health during level up:** Game over takes priority, cancel level up
- **Rapid level ups:** Queue upgrades, process one at a time

### Performance Safeguards
- **Max enemies:** Cap at 100 simultaneous enemies
- **Max projectiles:** Cap at 200 active projectiles
- **Entity culling:** Don't render enemies beyond 60 units from player
- **Object pooling:** Reuse enemy/projectile objects instead of creating new ones

### Input Edge Cases
- **Pointer lock lost:** Pause game, show "click to resume"
- **Tab away from game:** Pause automatically
- **Held button on game over:** Clear input state on state transitions

---

## Accessibility

### Visual
- **High contrast colors:** Player (cyan), Enemies (red), Projectiles (yellow)
- **Colorblind mode (future):** Pattern-based differentiation option
- **Screen shake:** Toggle option (default on, can disable)
- **Flash effects:** Reduced intensity option

### Audio (Placeholder Implementation)
- **Sound cues:** Distinct sounds for deflect vs parry vs hit
- **Spatial audio:** 3D positioned enemy firing sounds
- **Volume controls:** Master, SFX, Music sliders

### Controls
- **Rebindable keys (future):** All actions remappable
- **Gamepad support (future):** Full controller support
- **Mouse sensitivity:** Adjustable in options

---

## Performance Goals

### Target Metrics
- **Frame Rate:** Stable 60 FPS with 50+ enemies on mid-range hardware
- **Input Latency:** <16ms input-to-visual response
- **Load Time:** <3 seconds initial load

### Optimization Strategies
- **Instanced Meshes:** Use InstancedMesh for enemies and projectiles
- **Object Pooling:** Pre-allocate and reuse all game entities
- **Frustum Culling:** drei's default, enhanced with distance culling
- **State Batching:** Batch Zustand updates to prevent excessive re-renders
- **RAF Sync:** Ensure game loop is tied to requestAnimationFrame

---

## Testing Scenarios

### Core Mechanics
1. **Passive Deflection:** Stand still, let projectile approach, verify random deflection
2. **Parry Deflection:** Click during projectile approach, verify enemy targeting
3. **Held Deflection:** Hold button, multiple projectiles redirect to enemies
4. **Melee Attack:** Slash through group of enemies, verify all in hitbox destroyed
5. **Dodge:** Dash through projectile stream, verify no damage taken

### Progression
6. **XP Gain:** Kill enemies, verify XP bar fills correctly
7. **Level Up:** Reach XP threshold, verify upgrade screen appears
8. **Upgrade Application:** Select upgrade, verify stat change
9. **Stacking:** Take same upgrade multiple times, verify cumulative effect

### Wave System
10. **Wave Start:** Begin game, verify correct enemy count spawns
11. **Wave Progression:** Clear wave, verify next wave spawns after delay
12. **Wave Scaling:** Reach wave 5+, verify increased difficulty

### Edge Cases
13. **Simultaneous Hits:** Ensure only one damage instance with mercy invuln
14. **Death Condition:** Health reaches 0, verify game over screen
15. **Boundary Test:** Walk to arena edge, verify cannot exit

---

## Extended Features (Future Enhancements)

### Phase 2: Model Integration
- Replace Box with animated player GLTF
- Replace Spheres with enemy GLTF models
- Add weapon mesh to player
- Animation system integration (idle, walk, slash, dash, deflect)

### Phase 3: Advanced Gameplay
- Additional enemy types (melee, boss)
- Combo system (chain parries for multiplier)
- Ultimate ability (screen-clear attack)
- Multiple arenas with different layouts

### Phase 4: Meta Progression
- Persistent unlocks between runs
- Starting weapon selection
- Achievement system
- Leaderboards

---

## Style Guide

### Color Palette
```
Player:     #00FFFF (Cyan)
Enemies:    #FF4444 (Red) with type variations
Projectiles:#FFFF00 (Yellow) / type-specific
Floor:      #1a1a2e (Dark blue-gray)
Grid Lines: #4a4a5e (Lighter gray)
UI Health:  #FF0000 (Red)
UI XP:      #4488FF (Blue)
UI Text:    #FFFFFF (White)
Background: #0a0a0f (Near black)
```

### Typography (HUD)
- Font: System sans-serif (Arial/Helvetica fallback)
- Health/XP numbers: 18px bold
- Wave counter: 24px bold
- Level indicator: 20px bold
- Upgrade cards: 16px regular, titles 18px bold

### Visual Effects
- Deflection spark: 8 particles, spread 45°, fade over 0.2s
- Parry flash: Screen edge golden pulse, 0.1s
- Dash afterimage: 3 copies, 30% opacity each, staggered fade
- Slash trail: Red gradient plane, follows swing arc, 0.3s duration

---

## Implementation Order

### Phase 1: Core Foundation
1. Project setup with Vite + React + R3F
2. Basic arena (floor, lighting, boundaries)
3. Player cube with WASD movement
4. Third-person camera following player

### Phase 2: Combat Basics
5. Enemy spheres with basic spawning
6. Projectile system (enemies fire at player)
7. Passive deflection mechanic
8. Player health and damage

### Phase 3: Advanced Combat
9. Parry (timed) deflection with targeting
10. Held deflection with auto-lock
11. Melee slash attack
12. Dodge with i-frames

### Phase 4: Systems
13. Wave spawning system
14. XP and leveling
15. Upgrade system with UI
16. Enemy AI behaviors

### Phase 5: Polish
17. Fabric.js HUD integration
18. Particle effects
19. Game state management (menu, pause, game over)
20. Performance optimization (pooling, instancing)

---

*This TINS README is designed for AI code generation. All specifications are explicit and complete enough to produce a functional implementation.*
