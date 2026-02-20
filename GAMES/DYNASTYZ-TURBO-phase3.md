# DYNASTYZ-TURBO Phase 3: Wave Progression Fix & Difficulty Scaling

## Overview

Fix the wave progression system that is currently stuck on Wave 1, and scale enemy counts to 3x to match the player's enhanced capabilities (hurricane slash, beam weapons, missiles). Implement kill-count based wave advancement and regular boss appearances in later waves.

---

## Problem Analysis

### Current Issues
1. **Wave stuck at 1**: The wave completion detection is not triggering properly
2. **`waveEnemiesRemaining` not decrementing**: Likely disconnected from kill tracking
3. **Enemy counts too low**: Player is now very powerful with 360° hurricane slash

### Root Cause Investigation
The `WaveSpawner` checks:
```typescript
if (
  waveEnemiesRemaining === 0 &&
  !isSpawning.current &&
  !waveComplete.current &&
  enemies.length === 0
)
```

Need to verify:
- Is `waveEnemiesRemaining` being decremented when enemies die?
- Is `enemies.length` reaching 0?
- Are there timing issues with the async spawn system?

---

## Solution Design

### Kill-Based Wave Advancement
Instead of relying on `waveEnemiesRemaining`, use total kills to trigger wave transitions:

```typescript
// Track kills needed per wave
const waveKillThreshold = getWaveTotalEnemies(currentWave);
const waveStartKills = totalKillsAtWaveStart;

// Check if wave is complete
if (totalKills - waveStartKills >= waveKillThreshold) {
  // Advance to next wave
}
```

### Difficulty Scaling (3x Enemies)
Apply a global multiplier to enemy counts:
```typescript
const DIFFICULTY_MULTIPLIER = 3;

// In wave composition
grunt: (5 + (waveNum - 1) * 2) * DIFFICULTY_MULTIPLIER,
```

### Boss Frequency Enhancement
- Nemesis appears every 5 waves starting at wave 5 (instead of 10)
- Additional mini-bosses (swarm_lord) appear more frequently
- Boss formations include elite escorts

---

## Task Checklist

### Phase 3.1: Diagnose Wave Progression Issue
- [ ] **3.1.1** Read `useGameStore.ts` to check `waveEnemiesRemaining` logic
- [ ] **3.1.2** Read `useEntitiesStore.ts` to check if kills decrement wave counter
- [ ] **3.1.3** Identify the disconnect between enemy death and wave tracking

### Phase 3.2: Implement Kill-Based Wave Tracking
- [ ] **3.2.1** Update `useGameStore.ts`
  - Add `waveStartKills: number` to track kills at wave start
  - Add `getWaveProgress()` selector
  - Update `setWave()` to record starting kill count

- [ ] **3.2.2** Update `WaveSpawner.tsx`
  - Replace `waveEnemiesRemaining === 0` check with kill-based check
  - Use `totalKills - waveStartKills >= expectedKills` for wave completion
  - Ensure spawning completes before checking kills

### Phase 3.3: Apply 3x Difficulty Multiplier
- [ ] **3.3.1** Update `src/config/waves.config.ts`
  - Add `DIFFICULTY_MULTIPLIER = 3` constant
  - Apply multiplier to all enemy counts in `calculateBaseEnemyCounts()`
  - Export multiplier for UI display if needed

- [ ] **3.3.2** Verify formation splitting handles larger counts
  - Ensure formations split properly with 3x enemies
  - May need to increase `maxPerFormation` limits

### Phase 3.4: Enhanced Boss Frequency
- [ ] **3.4.1** Update boss spawn logic in `waves.config.ts`
  - Nemesis: Every 5 waves starting at wave 5 (waves 5, 10, 15, 20...)
  - Swarm_lord: Appears from wave 5+, scaling count
  - Add elite escorts to boss formations

- [ ] **3.4.2** Create boss wave compositions
  - Boss waves get Diamond formation with Nemesis center
  - Surrounding elite/sentinel formations as guards
  - Dramatic spawn timing for boss entrance

### Phase 3.5: Wave UI Feedback
- [ ] **3.5.1** Update `WaveInfo.tsx` (if exists) or HUD
  - Show wave progress: "Wave X - 45/120 enemies"
  - Add visual indicator when wave is complete
  - Show "WAVE COMPLETE" message briefly

### Phase 3.6: Testing & Verification
- [ ] **3.6.1** Verify wave 1 → 2 transition works
- [ ] **3.6.2** Verify wave 5 spawns first Nemesis
- [ ] **3.6.3** Verify 3x enemy counts spawn correctly
- [ ] **3.6.4** Performance test with 100+ enemies
- [ ] **3.6.5** Build verification

---

## Implementation Details

### 3.2.1 useGameStore Updates

```typescript
interface GameStoreState {
  // ... existing
  waveStartKills: number;  // NEW: kills when wave started
}

interface GameStoreActions {
  // ... existing
  setWave: (wave: number, enemyCount: number) => void;  // Update signature
}

// In setWave action:
setWave: (wave, enemyCount) => set({
  wave,
  waveEnemiesRemaining: enemyCount,
  waveStartKills: get().totalKills,  // Record current kills
}),
```

### 3.2.2 WaveSpawner Kill-Based Check

```typescript
// In WaveSpawner.tsx
const totalKills = useGameStore(s => s.totalKills);
const waveStartKills = useGameStore(s => s.waveStartKills);

// Calculate expected kills for current wave
const expectedKills = getWaveTotalEnemies(wave);
const waveKills = totalKills - waveStartKills;

// Check wave completion
useEffect(() => {
  if (gameState !== 'playing') return;
  if (isSpawning.current || waveComplete.current) return;

  // Wave complete when all expected enemies killed
  if (waveKills >= expectedKills && enemies.length === 0) {
    waveComplete.current = true;
    setTimeout(() => {
      const nextWave = wave + 1;
      useGameStore.getState().incrementWave();
      waveComplete.current = false;
      spawnWave(nextWave);
    }, 2000);
  }
}, [totalKills, waveStartKills, enemies.length, gameState, wave]);
```

### 3.3.1 Difficulty Multiplier

```typescript
// src/config/waves.config.ts

export const DIFFICULTY_MULTIPLIER = 3;

const calculateBaseEnemyCounts = (waveNum: number): WaveEnemyCounts => {
  const mult = DIFFICULTY_MULTIPLIER;

  return {
    grunt: (5 + (waveNum - 1) * 2) * mult,
    rapid: Math.max(0, (waveNum - 2) * 2) * mult,
    heavy: Math.max(0, waveNum - 2) * mult,
    sniper: Math.max(0, waveNum - 3) * mult,
    drone: Math.max(0, (waveNum - 2) * 2) * mult,
    elite: Math.max(0, Math.floor(waveNum - 4)) * mult,
    sentinel: Math.max(0, Math.floor((waveNum - 5) / 2)) * mult,
    hunter: Math.max(0, Math.floor(waveNum - 6)) * mult,
    swarm_lord: Math.max(0, Math.floor((waveNum - 4) / 2)) * mult,  // Earlier appearance
    nemesis: waveNum >= 5 && waveNum % 5 === 0 ? 1 : 0,  // Every 5 waves from wave 5
  };
};
```

### 3.4.1 Boss Wave Compositions

```typescript
// Enhanced boss logic
const getBossCount = (waveNum: number): number => {
  if (waveNum < 5) return 0;
  if (waveNum % 5 !== 0) return 0;

  // Scale bosses: 1 at wave 5, 2 at wave 15, 3 at wave 25, etc.
  return Math.floor(waveNum / 15) + 1;
};

const getEliteEscortCount = (waveNum: number): number => {
  if (waveNum < 5) return 0;
  if (waveNum % 5 !== 0) return 0;

  // 4 elite escorts per boss
  return getBossCount(waveNum) * 4;
};
```

---

## Expected Wave Compositions (with 3x multiplier)

| Wave | Grunts | Rapid | Heavy | Sniper | Drone | Elite | Sentinel | Hunter | Swarm Lord | Nemesis | Total |
|------|--------|-------|-------|--------|-------|-------|----------|--------|------------|---------|-------|
| 1 | 15 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 15 |
| 2 | 21 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 21 |
| 3 | 27 | 6 | 3 | 0 | 6 | 0 | 0 | 0 | 0 | 0 | 42 |
| 4 | 33 | 12 | 6 | 3 | 12 | 0 | 0 | 0 | 0 | 0 | 66 |
| 5 | 39 | 18 | 9 | 6 | 18 | 3 | 0 | 0 | 3 | **1** | **97** |
| 6 | 45 | 24 | 12 | 9 | 24 | 6 | 3 | 0 | 6 | 0 | 129 |
| 7 | 51 | 30 | 15 | 12 | 30 | 9 | 3 | 3 | 9 | 0 | 162 |
| 8 | 57 | 36 | 18 | 15 | 36 | 12 | 6 | 6 | 12 | 0 | 198 |
| 9 | 63 | 42 | 21 | 18 | 42 | 15 | 6 | 9 | 15 | 0 | 231 |
| 10 | 69 | 48 | 24 | 21 | 48 | 18 | 9 | 12 | 18 | **1** | **268** |

---

## Files to Modify

| File | Changes |
|------|---------|
| `src/stores/useGameStore.ts` | Add `waveStartKills`, update `setWave()` |
| `src/config/waves.config.ts` | Add `DIFFICULTY_MULTIPLIER`, adjust boss timing |
| `src/systems/WaveSpawner.tsx` | Kill-based wave completion check |
| `src/components/ui/WaveInfo.tsx` | Show wave progress (optional) |

---

## Success Criteria

1. **Wave Progression**: Game advances from wave 1 → 2 → 3... correctly
2. **Kill Tracking**: Waves complete based on actual kills, not counter
3. **3x Difficulty**: Enemy counts tripled across all waves
4. **Boss Frequency**: Nemesis appears at waves 5, 10, 15, 20...
5. **Performance**: No lag with 100+ enemies on screen
6. **Formations**: Larger enemy counts still spawn in proper formations

---

## Rollback Plan

If issues arise:
1. Revert `DIFFICULTY_MULTIPLIER` to 1
2. Restore original `waveEnemiesRemaining` check as fallback
3. Keep formation system intact (it works independently)
