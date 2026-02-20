# R-STRIKER RPG Dialogue & Pilot HUD System

**Integration Document** — Toriyama Character Enhancement via RPGDialogue.jsx  
**Version:** 1.0.0  
**Status:** Implementation Ready  
**Dependencies:** RPGDialogue.jsx (included in codebase)

---

## Description

This document specifies the integration of the RPGDialogue.jsx system into R-STRIKER to address the character dimension weaknesses identified in the Toriyama Alignment Assessment. Rather than embedding faces directly into the wireframe game entities (which would conflict with the TRON aesthetic), we leverage three distinct dialogue/portrait systems to inject personality and expression without compromising visual consistency.

The system provides three modes of character expression: (1) full-screen narrative dialogue for story beats and mission briefings, (2) in-game toast-style communications for dynamic NPC interactions, and (3) a persistent Doom-style pilot HUD portrait that reacts to gameplay state. Together, these address Face-First Creation (score 2→4), Hero Approachability (score 2→4), and Tonal Balance (score 3→4) from the original assessment.

---

## Functionality

### Mode 1: Fullscreen Narrative Dialogue (JRPG Style)

The fullscreen mode activates during pre-mission briefings, post-mission debriefs, story cutscenes, and critical narrative moments. Characters appear as high-resolution portraits (512×512 minimum) with multiple expression frames, positioned left-side with dialogue box on right.

**When to Trigger:**
- Mission start briefings (Commander introduces objectives)
- Mission completion screens (pilot debrief, rescued personnel dialogue)
- Story beats between campaign stages
- First encounter with new enemy types (Intel Officer explains threat)
- Game over / continue screens (pilot's last transmission)

**Character Roster for Fullscreen Mode:**
- **"Viper" (Player Pilot):** 6 expressions — neutral, determined, shocked, injured, victorious, desperate
- **"Command" (Mission Controller):** 4 expressions — neutral, concerned, urgent, relieved
- **"Intel" (Intelligence Officer):** 4 expressions — neutral, analytical, warning, excited
- **"Rescued" (Generic Rescued Personnel):** 3 expressions — grateful, panicked, relieved
- **"Enemy Commander" (Campaign Antagonist):** 4 expressions — menacing, amused, enraged, defeated

**Visual Style Requirements:**
- Characters drawn in clean anime-adjacent style (not wireframe) to contrast with gameplay
- Color palette maintains TRON influence (cyan highlights, dark backgrounds, neon accents)
- Portraits should have transparent backgrounds for gradient overlay composition
- Expression changes can be instant (cut) or use 2-frame crossfade (150ms)

**Dialogue Box Specifications:**
- Uses RPGDialogue.jsx `FullscreenDialogue` component
- Gold border (#ffd700) maintained for visual consistency with existing component
- Typewriter speed: 25ms per character for dramatic pacing
- Character name tag positioned top-left of dialogue box
- Continue indicator (▼) appears when text complete
- Space/Enter/Click advances dialogue

**Data Structure for Fullscreen Dialogues:**
```javascript
const missionBriefing = {
  mode: 'fullscreen',
  sequence: [
    {
      character: 'Command',
      characterImage: '/portraits/command_neutral.png',
      pose: 'neutral',
      text: "Viper, we've got a situation in Sector 7. SAM installations have locked down the canyon approach.",
    },
    {
      character: 'Command',
      characterImage: '/portraits/command_concerned.png',
      pose: 'concerned',
      text: "Intel shows mobile units moving to reinforce. You'll need to punch through before they arrive.",
    },
    {
      character: 'Intel',
      characterImage: '/portraits/intel_warning.png',
      pose: 'warning',
      text: "Watch for the new Viper-class mobile SAMs. Smaller profile, faster lock-on. Don't let them track you.",
    },
  ]
};
```

---

### Mode 2: In-Game Toast Communications (Video Call Style)

The toast mode appears during active gameplay as a pop-out "video call" style overlay. Small pixel art portraits (48×48 or 64×64) appear with animated talking frames while text displays in an adjacent speech bubble. This allows speaking NPCs to have expression without interrupting gameplay flow.

**When to Trigger:**
- Objective updates mid-mission ("Viper, new hostiles incoming!")
- Rescued personnel calling for extraction ("Help! We're pinned down!")
- Enemy taunts when player takes damage ("Is that the best you've got?")
- Pilot acknowledgments ("Copy that, moving to waypoint.")
- Environmental warnings ("Fuel critical! RTB immediately!")
- Intel updates on enemy movements ("SAM site coming online at grid 4-7!")

**Character Roster for Toast Mode (Pixel Art):**
- **"Viper" (Player Pilot):** 4 animation states — idle (2 frames), talking (4 frames), alarmed (2 frames), damaged (2 frames)
- **"Command":** 3 animation states — idle (2 frames), talking (4 frames), urgent (2 frames)
- **"Ground Team":** Generic rescued personnel — 2 animation states — talking (4 frames), panicked (4 frames)
- **"Enemy":** Intercepted transmissions — 2 animation states — talking (4 frames), laughing (3 frames)

**Visual Style Requirements:**
- 48×48 pixel art portraits, scaled 2× for display (96×96 rendered)
- Image rendering set to `pixelated` to preserve sharp pixels
- Talking animation cycles through frames at 150ms per frame during text display
- Returns to idle frame when text complete
- Frame border color indicates transmission type:
  - Cyan (#4a9eff) — Friendly/Player
  - Orange (#ff6b35) — Enemy intercept
  - Gold (#ffd700) — Command priority
  - Red (#ff4444) — Critical/Emergency

**Position and Timing:**
- Default position: bottom-left (matches current RPGDialogue default)
- Can shift to bottom-right when action is on left side of screen
- Auto-advance after 3 seconds of completed text display
- Player can click to dismiss immediately
- Multiple toasts queue; maximum 1 visible at a time

**Integration with Game Events:**
```javascript
// Example: Hooking toast dialogue to game events
gameEvents.on('objectiveUpdate', (data) => {
  showToastDialogue({
    character: 'Command',
    portraitFrames: [
      '/sprites/command_talk_1.png',
      '/sprites/command_talk_2.png',
      '/sprites/command_talk_3.png',
      '/sprites/command_talk_4.png',
    ],
    text: data.message,
    position: 'bottom-left',
    autoAdvance: true,
    autoAdvanceDelay: 3000,
    borderColor: '#ffd700', // Command priority
  });
});

gameEvents.on('playerDamaged', (data) => {
  if (data.severity === 'critical') {
    showToastDialogue({
      character: 'Viper',
      portraitFrames: [
        '/sprites/viper_damaged_1.png',
        '/sprites/viper_damaged_2.png',
      ],
      text: "Taking heavy fire! Armor critical!",
      position: 'bottom-left',
      autoAdvance: true,
      autoAdvanceDelay: 2500,
      borderColor: '#ff4444', // Emergency
    });
  }
});
```

**Data Structure for Toast Dialogues:**
```javascript
const toastMessage = {
  mode: 'toast',
  character: 'Ground Team',
  portraitFrames: [
    '/sprites/ground_panic_1.png',
    '/sprites/ground_panic_2.png',
    '/sprites/ground_panic_3.png',
    '/sprites/ground_panic_4.png',
  ],
  text: "We're taking fire! Get us out of here!",
  position: 'bottom-left',
  autoAdvance: true,
  autoAdvanceDelay: 3500,
  typewriterSpeed: 20,
};
```

---

### Mode 3: Pilot HUD Portrait (Doom Face Style)

A persistent pilot portrait sits in the game HUD, displaying a 4-frame animation system that reacts to player state. This addresses the Toriyama recommendation to "imply a pilot" and transforms the helicopter from equipment into inhabited character.

**HUD Position:**
- Centered at bottom of screen, within or adjacent to existing resource bars
- Portrait size: 64×64 pixels, scaled to 128×128 rendered
- Framed with cyan wireframe border to match TRON aesthetic
- Subtle scanline overlay effect (optional) for CRT/retro feel

**Animation State Machine:**

| State | Condition | Frames | Animation |
|-------|-----------|--------|-----------|
| `idle_healthy` | HP ≥ 75%, not in combat | 2 | Slow blink cycle (3s period) |
| `idle_wounded` | HP 50-74%, not in combat | 2 | Occasional wince, heavier breathing |
| `idle_critical` | HP 25-49%, not in combat | 2 | Pained expression, sweat drops |
| `idle_danger` | HP < 25%, not in combat | 2 | Desperate/panicked, rapid breathing |
| `look_left` | Player turning left | 1 | Eyes/head shift left |
| `look_right` | Player turning right | 1 | Eyes/head shift right |
| `firing_rockets` | Firing rockets | 2 | Determined/aggressive, recoil reaction |
| `firing_gun` | Firing minigun | 2 | Focused, slight vibration |
| `taking_damage` | On hit event | 3 | Pain flash, returns to appropriate idle |
| `victory` | Mission complete | 4 | Smile, thumbs up (implied), relief |
| `defeat` | Game over | 3 | Despair, helmet cracks (final frame) |

**State Transition Logic:**
```javascript
const pilotPortraitState = {
  // Base state determined by HP
  getBaseState(hp, maxHp) {
    const percent = (hp / maxHp) * 100;
    if (percent >= 75) return 'idle_healthy';
    if (percent >= 50) return 'idle_wounded';
    if (percent >= 25) return 'idle_critical';
    return 'idle_danger';
  },
  
  // Override states (temporary, return to base after duration)
  overrides: {
    'look_left': { duration: 0, persistent: true }, // While turning
    'look_right': { duration: 0, persistent: true }, // While turning
    'firing_rockets': { duration: 500 },
    'firing_gun': { duration: 200 },
    'taking_damage': { duration: 400 },
    'victory': { duration: 3000 },
    'defeat': { duration: null }, // Permanent until reset
  },
};
```

**Frame Timing:**
- Idle animations: 500ms per frame
- Action animations: 100-150ms per frame
- Damage flash: 80ms per frame
- Look left/right: Instant transition, holds while input active

**Sprite Sheet Organization:**
```
pilot_portrait_sheet.png (512×256)
├── Row 0: idle_healthy (2), idle_wounded (2), idle_critical (2), idle_danger (2)
├── Row 1: look_left (1), look_right (1), firing_rockets (2), firing_gun (2), padding (2)
├── Row 2: taking_damage (3), victory (4), padding (1)
└── Row 3: defeat (3), padding (5)
```

**React Component Structure:**
```javascript
const PilotHUD = ({ hp, maxHp, isFiring, weaponType, turnDirection, missionState }) => {
  const [currentFrame, setCurrentFrame] = useState(0);
  const [currentState, setCurrentState] = useState('idle_healthy');
  
  // State determination logic
  useEffect(() => {
    if (missionState === 'victory') {
      setCurrentState('victory');
    } else if (missionState === 'defeat') {
      setCurrentState('defeat');
    } else if (turnDirection === 'left') {
      setCurrentState('look_left');
    } else if (turnDirection === 'right') {
      setCurrentState('look_right');
    } else if (isFiring) {
      setCurrentState(weaponType === 'rockets' ? 'firing_rockets' : 'firing_gun');
    } else {
      setCurrentState(getBaseState(hp, maxHp));
    }
  }, [hp, maxHp, isFiring, weaponType, turnDirection, missionState]);
  
  // Frame animation loop
  useEffect(() => {
    const frameCount = PILOT_STATES[currentState].frames;
    const frameSpeed = PILOT_STATES[currentState].speed;
    
    const interval = setInterval(() => {
      setCurrentFrame(f => (f + 1) % frameCount);
    }, frameSpeed);
    
    return () => clearInterval(interval);
  }, [currentState]);
  
  return (
    <div className="pilot-hud-container">
      <div className="pilot-frame wireframe-border">
        <img 
          src={getPilotFrame(currentState, currentFrame)}
          alt="Pilot"
          style={{ imageRendering: 'pixelated' }}
        />
      </div>
    </div>
  );
};
```

---

## Technical Implementation

### File Structure

```
/src
├── /components
│   ├── RPGDialogue.jsx          # Existing component (imported)
│   ├── PilotHUD.jsx              # New: Doom-style pilot portrait
│   └── DialogueManager.jsx       # New: Manages dialogue queue/sequences
├── /assets
│   ├── /portraits                # Fullscreen character art (512×512)
│   │   ├── viper_neutral.png
│   │   ├── viper_determined.png
│   │   ├── command_neutral.png
│   │   └── ...
│   ├── /sprites                  # Pixel art portraits (48×48)
│   │   ├── viper_talk_sheet.png
│   │   ├── command_talk_sheet.png
│   │   └── ...
│   └── /hud
│       └── pilot_portrait_sheet.png
├── /data
│   ├── dialogues_briefings.json  # Mission briefing scripts
│   ├── dialogues_toasts.json     # In-game toast messages
│   └── pilot_states.json         # Animation state definitions
└── /hooks
    └── useGameDialogue.js        # Hook for triggering dialogues from game
```

### DialogueManager Component

The DialogueManager wraps the game and provides context for triggering dialogues from anywhere in the component tree.

```javascript
import React, { createContext, useContext, useState, useCallback } from 'react';
import RPGDialogue, { useDialogueSequence } from './RPGDialogue';

const DialogueContext = createContext(null);

export const useDialogue = () => useContext(DialogueContext);

export const DialogueManager = ({ children }) => {
  const [toastQueue, setToastQueue] = useState([]);
  const [currentToast, setCurrentToast] = useState(null);
  const [fullscreenDialogue, setFullscreenDialogue] = useState(null);
  
  const showToast = useCallback((dialogue) => {
    setToastQueue(q => [...q, dialogue]);
  }, []);
  
  const showFullscreen = useCallback((sequence) => {
    setFullscreenDialogue(sequence);
  }, []);
  
  const dismissFullscreen = useCallback(() => {
    setFullscreenDialogue(null);
  }, []);
  
  // Process toast queue
  useEffect(() => {
    if (!currentToast && toastQueue.length > 0) {
      setCurrentToast(toastQueue[0]);
      setToastQueue(q => q.slice(1));
    }
  }, [currentToast, toastQueue]);
  
  const handleToastComplete = useCallback(() => {
    setCurrentToast(null);
  }, []);
  
  return (
    <DialogueContext.Provider value={{ showToast, showFullscreen, dismissFullscreen }}>
      {children}
      
      {/* Toast Overlay */}
      {currentToast && (
        <RPGDialogue
          mode="toast"
          {...currentToast}
          visible={true}
          onComplete={handleToastComplete}
        />
      )}
      
      {/* Fullscreen Overlay */}
      {fullscreenDialogue && (
        <FullscreenSequence
          dialogues={fullscreenDialogue}
          onComplete={dismissFullscreen}
        />
      )}
    </DialogueContext.Provider>
  );
};
```

### Integration Points with R-STRIKER Game Loop

```javascript
// In main game component
const RStrikerGame = () => {
  const { showToast, showFullscreen } = useDialogue();
  
  // Mission start
  useEffect(() => {
    if (missionState === 'starting') {
      showFullscreen(getMissionBriefing(currentMission));
    }
  }, [missionState]);
  
  // Dynamic events
  const handleEnemyDestroyed = (enemy) => {
    if (enemy.type === 'sam_mobile' && !firstMobileSamKilled) {
      showToast({
        character: 'Intel',
        portraitFrames: INTEL_TALK_FRAMES,
        text: "Mobile SAM neutralized! Watch for more in the area.",
        autoAdvance: true,
      });
      setFirstMobileSamKilled(true);
    }
  };
  
  const handlePlayerDamage = (amount, source) => {
    if (player.hp < player.maxHp * 0.25) {
      showToast({
        character: 'Command',
        portraitFrames: COMMAND_URGENT_FRAMES,
        text: "Viper, your armor is critical! RTB for repairs!",
        borderColor: '#ff4444',
        autoAdvance: true,
      });
    }
  };
  
  const handleRescue = (personnel) => {
    showToast({
      character: 'Rescued',
      portraitFrames: RESCUED_GRATEFUL_FRAMES,
      text: personnel.dialogue || "Thank you! Get us out of here!",
      autoAdvance: true,
    });
  };
  
  return (
    <div className="game-container">
      <GameCanvas 
        onEnemyDestroyed={handleEnemyDestroyed}
        onPlayerDamage={handlePlayerDamage}
        onRescue={handleRescue}
      />
      <PilotHUD 
        hp={player.hp}
        maxHp={player.maxHp}
        isFiring={player.isFiring}
        weaponType={player.currentWeapon}
        turnDirection={player.turnDirection}
        missionState={missionState}
      />
    </div>
  );
};
```

---

## Style Guide

### Color Palette

| Element | Color | Hex | Usage |
|---------|-------|-----|-------|
| Friendly Border | Cyan | #4a9eff | Player/allied communications |
| Command Border | Gold | #ffd700 | Priority transmissions, fullscreen |
| Enemy Border | Orange | #ff6b35 | Intercepted enemy comms |
| Emergency Border | Red | #ff4444 | Critical warnings |
| Background Dark | Navy | #0f0f1a | Dialogue box background |
| Background Mid | Slate | #1a1a2e | Gradient layer |
| Text Primary | White | #ffffff | Dialogue text |
| Text Secondary | Gray | #a0a0a0 | Hints, subtitles |

### Typography

- **Dialogue Text:** "Press Start 2P" for retro feel, fallback to "Courier New" monospace
- **Character Names:** Same font, uppercase, 10-12px
- **Hints:** Same font, 8px, reduced opacity

### Portrait Art Guidelines

**Fullscreen Portraits (512×512):**
- Clean line art with cel-shading style
- Limited color palette (max 8 colors per character + gradients)
- Strong silhouettes that read at thumbnail size
- Expressions exaggerated for readability
- Transparent background with subtle edge glow

**Pixel Art Portraits (48×48):**
- Strict 48×48 canvas, no anti-aliasing
- Maximum 16 colors per character
- Clear face direction and expression at small scale
- Animation frames on single horizontal strip
- 2px outline in darkest character color

**Pilot HUD Sprites (64×64):**
- Same rules as pixel portraits
- Must include flight helmet/visor design
- Expressions visible through visor (eyes, mouth)
- Consistent lighting direction (top-left) across all frames

---

## Testing Scenarios

### Scenario 1: Mission Briefing Flow
1. Player selects mission from menu
2. Fullscreen dialogue initiates with Command character
3. Player advances through 3-5 dialogue beats
4. Final dialogue from Intel with mission-specific warnings
5. Dialogue dismisses, game loads mission

**Expected:** Smooth transitions, no input lag, text completes before auto-advance

### Scenario 2: In-Game Toast Cascade
1. Player destroys first SAM site
2. Toast appears: Command congratulates
3. Player immediately takes damage
4. Current toast completes naturally
5. Damage warning toast appears next
6. Player rescues personnel
7. Rescue toast queues after warning

**Expected:** Toasts queue properly, never overlap, audio cues (if any) don't stack

### Scenario 3: Pilot Portrait State Transitions
1. Player starts mission at full HP → idle_healthy
2. Player turns left → look_left (immediate)
3. Player releases turn → returns to idle_healthy
4. Player fires rockets → firing_rockets (500ms)
5. Player takes 30% damage → idle_wounded
6. Player takes 60% damage → idle_critical
7. Player completes mission → victory (3s celebration)

**Expected:** State transitions feel responsive, no stuck states, damage flash interrupts any state

### Scenario 4: Toast During Fullscreen
1. Fullscreen dialogue active
2. Game event triggers toast (shouldn't happen, but edge case)
3. Toast should queue and appear after fullscreen dismisses

**Expected:** No visual overlap, queue preserved

---

## Accessibility Requirements

- All dialogue text must be readable at default game resolution
- Typewriter effect can be disabled in options (instant text display)
- Portrait animations can be reduced/disabled for motion sensitivity
- High contrast mode: Borders thicken to 6px, background opacity increases to 100%
- Screen reader mode: Dialogue text pushed to aria-live region

---

## Performance Goals

- Fullscreen dialogue overlay: < 16ms to render (60fps)
- Toast appearance: < 100ms from trigger to visible
- Portrait frame swap: < 16ms
- Memory footprint for all portraits: < 5MB total
- Sprite sheets should be power-of-two dimensions for GPU optimization

---

## Extended Features

### Voice Acting Support (Future)
- Each dialogue entry can include optional `audioUrl` property
- Audio plays during typewriter effect
- Typewriter speed syncs to audio duration if provided

### Dialogue Branching (Future)
- Fullscreen mode can present 2-3 response options
- Player selection affects subsequent dialogue and potentially gameplay

### Enemy Personality Transmissions (Future)
- Named enemy aces have unique portraits and taunts
- Defeating an ace triggers their defeat dialogue
- Builds toward Toriyama's "memorable encounters" goal

---

## Toriyama Alignment Impact

| Dimension | Before | After | Change |
|-----------|--------|-------|--------|
| Face-First Creation | 2 | 4 | +2 (All major characters have expressive faces) |
| Hero Approachability | 2 | 4 | +2 (Pilot HUD creates persistent character presence) |
| Tonal Balance | 3 | 4 | +1 (Dialogue enables humor/drama interplay) |
| Mundane Transformation | 2 | 3 | +1 (Named characters elevate generic archetypes) |
| **Overall Score** | 3.2 | 4.0 | +0.8 |

This integration transforms R-STRIKER from "wireframe Desert Strike clone" toward "TRON meets Dragon Quest"—the ideal end state identified in the Toriyama assessment—by giving every interaction a face and every face a personality.

---

*Document prepared for TINS implementation. All specifications are deterministic and sufficient for code generation.*
