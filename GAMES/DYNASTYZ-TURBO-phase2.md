# DYNASTYZ-TURBO Phase 2: Military Formation Spawning System

## Overview

Replace the current circular spawning pattern with a military formation system inspired by Dynasty Warriors. Enemies spawn in organized unit formations (squares, lines, echelons) grouped by type, creating immersive battlefield aesthetics with balanced threat distribution.

---

## Design Principles

1. **Type Cohesion**: Each formation contains only one enemy type
2. **Threat Balancing**: Weaker enemies spawn in larger formations, stronger enemies in smaller elite squads
3. **Spatial Separation**: Formations spawn with gaps between them, creating distinct "army units"
4. **Wave Scaling**: Later waves feature more formations and tighter spacing
5. **Formation Variety**: Mix of squares, lines, and echelons for visual diversity

---

## Formation Types

### 1. Square Formation (Phalanx)
```
■ ■ ■ ■ ■
■ ■ ■ ■ ■
■ ■ ■ ■ ■
■ ■ ■ ■ ■
```
- **Use case**: Standard infantry (grunt, drone, basic)
- **Sizes**: 4x4 (16), 5x5 (25), 3x3 (9)
- **Spacing**: 2 units between enemies
- **Behavior**: Advances as a block toward player

### 2. Line Formation (Skirmish Line)
```
■ ■ ■ ■ ■ ■ ■ ■ ■ ■
```
- **Use case**: Ranged units (shooter, sniper, elite)
- **Sizes**: 6-12 enemies wide, 1-2 rows deep
- **Spacing**: 3 units between enemies (wider for firing lanes)
- **Behavior**: Maintains distance, fires volleys

### 3. Echelon Formation (Wedge/Arrow)
```
        ■
      ■ ■ ■
    ■ ■ ■ ■ ■
  ■ ■ ■ ■ ■ ■ ■
```
- **Use case**: Assault units (hunter, sentinel, heavy)
- **Sizes**: 3-5 rows, expanding width
- **Spacing**: 2.5 units
- **Behavior**: Aggressive push toward player

### 4. Diamond Formation (Elite Guard)
```
    ■
  ■ ■ ■
■ ■ ■ ■ ■
  ■ ■ ■
    ■
```
- **Use case**: Boss escorts (swarm_lord, nemesis guard)
- **Sizes**: 5-13 enemies
- **Spacing**: 2 units
- **Behavior**: Protects center unit or boss

### 5. Scattered Formation (Swarm)
```
  ■   ■     ■
■     ■   ■
  ■       ■   ■
    ■   ■     ■
```
- **Use case**: Fast units (drone, basic flankers)
- **Sizes**: 8-20 enemies
- **Spacing**: Random 1-4 units
- **Behavior**: Chaotic approach from multiple angles

---

## Enemy Type to Formation Mapping

| Enemy Type | Primary Formation | Secondary Formation | Max Per Formation |
|------------|------------------|---------------------|-------------------|
| basic | Square | Scattered | 25 |
| shooter | Line | Square | 12 |
| tank | Square | Echelon | 9 |
| sniper | Line | Diamond | 6 |
| drone | Scattered | Square | 20 |
| elite | Line | Diamond | 8 |
| sentinel | Echelon | Line | 6 |
| hunter | Echelon | Scattered | 8 |
| swarm_lord | Diamond | Echelon | 4 |
| nemesis | Diamond (solo center) | - | 1 |

---

## Wave Composition System

### Wave Structure
Each wave consists of multiple "Army Groups", each containing 1-3 formations of the same type.

```typescript
interface ArmyGroup {
  enemyType: EnemyType;
  formation: FormationType;
  count: number;
  spawnDirection: number; // Angle from center (0-360)
  spawnDistance: number;  // Distance from arena center
}

interface WaveComposition {
  waveNumber: number;
  groups: ArmyGroup[];
  spawnDelay: number; // Stagger between groups
}
```

### Wave Scaling Formula

| Wave | Total Enemies | Formations | Formation Types Available |
|------|---------------|------------|---------------------------|
| 1-2 | 15-25 | 1-2 | Square only |
| 3-4 | 30-45 | 2-3 | Square, Line |
| 5-6 | 50-70 | 3-4 | Square, Line, Scattered |
| 7-8 | 75-100 | 4-5 | All except Diamond |
| 9+ | 100-150 | 5-7 | All formations |
| Boss (10, 15, 20) | +50% | +1 Diamond | Nemesis in Diamond center |

---

## Spawn Position Calculation

### Formation Origin Points
Formations spawn at cardinal and intercardinal directions around the arena edge:
- **Primary spawns**: N, E, S, W (0°, 90°, 180°, 270°)
- **Secondary spawns**: NE, SE, SW, NW (45°, 135°, 225°, 315°)
- **Tertiary spawns**: Between primaries (22.5°, 67.5°, etc.)

### Formation Offset from Origin
```typescript
function calculateFormationPositions(
  formation: FormationType,
  origin: Vector3,
  facing: number, // Direction toward arena center
  enemyCount: number,
  spacing: number
): Vector3[] {
  // Returns array of spawn positions for each enemy in formation
}
```

### Minimum Distances
- **Between formations**: 15 units minimum
- **From player start**: 25 units minimum
- **From arena edge**: 5 units buffer

---

## Task Checklist

### Phase 2.1: Type System & Configuration
- [ ] **2.1.1** Create `src/config/formations.config.ts`
  - Define `FormationType` enum: 'square' | 'line' | 'echelon' | 'diamond' | 'scattered'
  - Define `FormationConfig` interface with size, spacing, shape parameters
  - Define `FORMATION_TYPES` constant with all formation configurations
  - Define `ENEMY_FORMATION_MAP` linking enemy types to preferred formations

- [ ] **2.1.2** Create `src/config/waves.config.ts`
  - Define `ArmyGroup` interface
  - Define `WaveComposition` interface
  - Create `generateWaveComposition(waveNumber: number)` function
  - Define base compositions for waves 1-10, with scaling formula for 10+

- [ ] **2.1.3** Update `src/config/index.ts`
  - Export new formation and wave configs

### Phase 2.2: Formation Position Calculator
- [ ] **2.2.1** Create `src/utils/formations.ts`
  - Implement `calculateSquarePositions(count, spacing, origin, facing)`
  - Implement `calculateLinePositions(count, spacing, origin, facing)`
  - Implement `calculateEchelonPositions(count, spacing, origin, facing)`
  - Implement `calculateDiamondPositions(count, spacing, origin, facing)`
  - Implement `calculateScatteredPositions(count, spacing, origin, facing)`
  - Implement `getFormationPositions(type, count, spacing, origin, facing)` dispatcher

- [ ] **2.2.2** Create `src/utils/spawnDirector.ts`
  - Implement `selectSpawnDirections(groupCount)` - picks non-overlapping angles
  - Implement `calculateSpawnOrigin(angle, distance)` - converts to world position
  - Implement `validateSpawnArea(positions, existingEnemies)` - collision check

- [ ] **2.2.3** Update `src/utils/index.ts`
  - Export new formation utilities

### Phase 2.3: Wave Spawner Refactor
- [ ] **2.3.1** Update `src/systems/WaveSpawner.tsx`
  - Replace `getWaveEnemies()` with `generateWaveComposition()` integration
  - Implement formation-based spawning in `spawnWave()`
  - Add staggered spawn timing for dramatic effect (0.5s between formations)
  - Track active formations for potential group behavior

- [ ] **2.3.2** Add spawn visualization (optional debug)
  - Show formation outlines before spawn
  - Display spawn direction indicators

### Phase 2.4: Formation Behavior (Optional Enhancement)
- [ ] **2.4.1** Update `src/components/entities/Enemy.tsx`
  - Add `formationId` to enemy state
  - Add `formationRole` ('leader' | 'member')
  - Implement loose formation maintenance (enemies try to stay near formation mates)

- [ ] **2.4.2** Update `src/stores/useEntitiesStore.ts`
  - Add formation tracking to enemy spawning
  - Add `getFormationMembers(formationId)` selector

### Phase 2.5: Testing & Balancing
- [ ] **2.5.1** Verify wave 1-5 spawning with new formations
- [ ] **2.5.2** Verify wave 6-10 with mixed formations
- [ ] **2.5.3** Verify boss waves (10, 15, 20) with Diamond formations
- [ ] **2.5.4** Performance test with 100+ enemies in formations
- [ ] **2.5.5** Balance pass - adjust formation sizes and spawn rates

---

## Implementation Details

### 2.1.1 Formation Config Structure

```typescript
// src/config/formations.config.ts

export type FormationType = 'square' | 'line' | 'echelon' | 'diamond' | 'scattered';

export interface FormationConfig {
  type: FormationType;
  baseSpacing: number;      // Units between enemies
  minCount: number;         // Minimum enemies for this formation
  maxCount: number;         // Maximum enemies for this formation
  rowGrowth: 'fixed' | 'expanding' | 'random';
}

export const FORMATION_TYPES: Record<FormationType, FormationConfig> = {
  square: {
    type: 'square',
    baseSpacing: 2,
    minCount: 4,
    maxCount: 25,
    rowGrowth: 'fixed',
  },
  line: {
    type: 'line',
    baseSpacing: 3,
    minCount: 4,
    maxCount: 12,
    rowGrowth: 'fixed',
  },
  echelon: {
    type: 'echelon',
    baseSpacing: 2.5,
    minCount: 3,
    maxCount: 15,
    rowGrowth: 'expanding',
  },
  diamond: {
    type: 'diamond',
    baseSpacing: 2,
    minCount: 5,
    maxCount: 13,
    rowGrowth: 'expanding',
  },
  scattered: {
    type: 'scattered',
    baseSpacing: 2,
    minCount: 6,
    maxCount: 20,
    rowGrowth: 'random',
  },
};

export const ENEMY_FORMATION_MAP: Record<EnemyType, { primary: FormationType; secondary: FormationType }> = {
  basic: { primary: 'square', secondary: 'scattered' },
  shooter: { primary: 'line', secondary: 'square' },
  tank: { primary: 'square', secondary: 'echelon' },
  sniper: { primary: 'line', secondary: 'diamond' },
  drone: { primary: 'scattered', secondary: 'square' },
  elite: { primary: 'line', secondary: 'diamond' },
  sentinel: { primary: 'echelon', secondary: 'line' },
  hunter: { primary: 'echelon', secondary: 'scattered' },
  swarm_lord: { primary: 'diamond', secondary: 'echelon' },
  nemesis: { primary: 'diamond', secondary: 'diamond' },
};
```

### 2.2.1 Square Formation Algorithm

```typescript
function calculateSquarePositions(
  count: number,
  spacing: number,
  origin: THREE.Vector3,
  facing: number
): THREE.Vector3[] {
  const positions: THREE.Vector3[] = [];
  const side = Math.ceil(Math.sqrt(count));
  const offset = (side - 1) * spacing / 2;

  for (let row = 0; row < side && positions.length < count; row++) {
    for (let col = 0; col < side && positions.length < count; col++) {
      const localX = col * spacing - offset;
      const localZ = row * spacing - offset;

      // Rotate to face toward arena center
      const rotatedX = localX * Math.cos(facing) - localZ * Math.sin(facing);
      const rotatedZ = localX * Math.sin(facing) + localZ * Math.cos(facing);

      positions.push(new THREE.Vector3(
        origin.x + rotatedX,
        origin.y,
        origin.z + rotatedZ
      ));
    }
  }

  return positions;
}
```

### 2.2.1 Echelon Formation Algorithm

```typescript
function calculateEchelonPositions(
  count: number,
  spacing: number,
  origin: THREE.Vector3,
  facing: number
): THREE.Vector3[] {
  const positions: THREE.Vector3[] = [];
  let row = 0;
  let placed = 0;

  while (placed < count) {
    const rowWidth = row * 2 + 1; // 1, 3, 5, 7...
    const enemiesInRow = Math.min(rowWidth, count - placed);
    const rowOffset = (enemiesInRow - 1) * spacing / 2;

    for (let i = 0; i < enemiesInRow; i++) {
      const localX = i * spacing - rowOffset;
      const localZ = row * spacing;

      // Rotate to face toward arena center
      const rotatedX = localX * Math.cos(facing) - localZ * Math.sin(facing);
      const rotatedZ = localX * Math.sin(facing) + localZ * Math.cos(facing);

      positions.push(new THREE.Vector3(
        origin.x + rotatedX,
        origin.y,
        origin.z + rotatedZ
      ));
      placed++;
    }
    row++;
  }

  return positions;
}
```

### 2.3.1 Wave Spawner Integration

```typescript
// In WaveSpawner.tsx

const spawnWave = async () => {
  const composition = generateWaveComposition(currentWave);
  const spawnDirections = selectSpawnDirections(composition.groups.length);

  for (let i = 0; i < composition.groups.length; i++) {
    const group = composition.groups[i];
    const direction = spawnDirections[i];
    const origin = calculateSpawnOrigin(direction, ARENA_CONFIG.size - 5);
    const facing = Math.atan2(-origin.x, -origin.z); // Face toward center

    const positions = getFormationPositions(
      group.formation,
      group.count,
      FORMATION_TYPES[group.formation].baseSpacing,
      origin,
      facing
    );

    // Spawn enemies at calculated positions
    positions.forEach(pos => {
      spawnEnemy(group.enemyType, pos);
    });

    // Stagger between formations for dramatic effect
    if (i < composition.groups.length - 1) {
      await delay(composition.spawnDelay);
    }
  }
};
```

---

## Visual Reference

### Wave 1 Example (Simple Square)
```
Player: P    Formation: ■

                    ■ ■ ■ ■
                    ■ ■ ■ ■
                    ■ ■ ■ ■
                    ■ ■ ■ ■

        P
```

### Wave 5 Example (Mixed Formations)
```
                Line (shooters)
            ■ ■ ■ ■ ■ ■ ■ ■

    Echelon                     Square
    (hunters)                   (basic)
        ■               P       ■ ■ ■ ■
      ■ ■ ■                     ■ ■ ■ ■
    ■ ■ ■ ■ ■                   ■ ■ ■ ■
                                ■ ■ ■ ■

            Scattered (drones)
          ■   ■     ■
        ■     ■   ■
          ■       ■   ■
```

### Wave 10 Boss Example (Diamond + Support)
```
            Line (elites)
        ■ ■ ■ ■ ■ ■ ■ ■

                            Diamond (swarm_lords)
                                  ■
Square                          ■ ■ ■
(basic)         P             ■ ■ N ■ ■  <- Nemesis center
■ ■ ■ ■ ■                       ■ ■ ■
■ ■ ■ ■ ■                         ■
■ ■ ■ ■ ■
■ ■ ■ ■ ■

        Echelon (sentinels)
              ■
            ■ ■ ■
          ■ ■ ■ ■ ■
```

---

## Success Criteria

1. **Visual Impact**: Formations are clearly visible and distinct on the battlefield
2. **Performance**: No frame drops with 100+ enemies in formations
3. **Balance**: Wave difficulty scales smoothly from 1 to 20
4. **Variety**: Each wave feels different due to formation mixing
5. **Cohesion**: Enemy types stay in their formation groups initially
6. **Dynasty Warriors Feel**: Large-scale battles with organized enemy armies

---

## Files to Create

| File | Purpose |
|------|---------|
| `src/config/formations.config.ts` | Formation type definitions and enemy mappings |
| `src/config/waves.config.ts` | Wave composition generation |
| `src/utils/formations.ts` | Position calculation algorithms |
| `src/utils/spawnDirector.ts` | Spawn direction and validation |

## Files to Modify

| File | Changes |
|------|---------|
| `src/config/index.ts` | Export new configs |
| `src/utils/index.ts` | Export new utilities |
| `src/systems/WaveSpawner.tsx` | Replace spawning logic with formation system |
| `src/stores/useEntitiesStore.ts` | Optional: Add formation tracking |
| `src/components/entities/Enemy.tsx` | Optional: Add formation behavior |

---

## Estimated Complexity

- **Phase 2.1 (Config)**: Low - Type definitions and constants
- **Phase 2.2 (Calculator)**: Medium - Math for position calculations
- **Phase 2.3 (Spawner)**: Medium - Integration and timing
- **Phase 2.4 (Behavior)**: Medium-High - Optional group AI
- **Phase 2.5 (Testing)**: Low - Verification and tuning

Total estimated tasks: 15 core + 2 optional
