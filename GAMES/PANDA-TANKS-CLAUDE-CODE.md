# PANDA-TANKS — Claude Code Development Plan

A TRON-inspired tank combat game built with Panda3D, structured for incremental agentic development with Claude Code and Opus 4.5.

## Development Philosophy

This plan follows **consequence-aware agentic development**:
- Each phase is independently testable
- Verification steps confirm success before proceeding
- Reversible checkpoints enable safe iteration
- Clear "done" criteria prevent scope creep

## Quick Start for Claude Code

```bash
# Initialize project
mkdir -p panda_tanks && cd panda_tanks
python -m venv venv && source venv/bin/activate
pip install panda3d numpy

# Verify installation
python -c "from direct.showbase.ShowBase import ShowBase; print('Panda3D ready')"
```

---

# Phase 0: Project Overview

## Project Summary

PANDA-TANKS is a desktop 3D tank combat game with TRON wireframe aesthetics. Players battle AI enemies in geometric arenas using energy weapons and abilities.

## Architecture

```
panda_tanks/
├── main.py                 # Entry point, ShowBase subclass
├── config.py               # Constants, colors, settings
├── screens/                # Game screens (menu, battle, etc.)
├── entities/               # 3D objects (tank, arena, projectiles)
├── systems/                # Game logic (physics, AI, combat)
├── ui/                     # DirectGUI components
├── data/                   # Levels, fonts
└── shaders/                # GLSL wireframe effects
```

## Phase Breakdown

| Phase | Name | Goal | Dependencies |
|-------|------|------|--------------|
| 1 | Foundation | ShowBase app, screen system, input | None |
| 2 | Entities | Tank, arena, projectiles rendering | Phase 1 |
| 3 | Systems | Physics, AI, combat mechanics | Phase 2 |
| 4 | Gameplay | Battle screen, HUD, game loop | Phase 3 |
| 5 | Menus | Main menu, bunker, settings, results | Phase 4 |
| 6 | Editor | Level editor with save/load | Phase 5 |
| 7 | Polish | Audio, effects, optimization | Phase 6 |

## Technology Stack

- **Engine:** Panda3D 1.10.13+
- **Language:** Python 3.10+
- **Physics:** Bullet (built-in)
- **UI:** DirectGUI
- **Rendering:** Custom GLSL shaders + LineSegs

## Success Criteria

- [ ] Game launches and displays main menu
- [ ] Player can customize loadout and start battle
- [ ] Combat works with multiple AI enemies
- [ ] Levels can be created, saved, and loaded
- [ ] 60 FPS on mid-range hardware

---

# Phase 1: Foundation

**Goal:** Create runnable application with screen management and input handling.

**Duration:** ~2 hours

**Deliverables:**
- `main.py` — ShowBase application
- `config.py` — Constants and colors
- `screens/base_screen.py` — Abstract screen class
- `screens/test_screen.py` — Placeholder test screen
- `systems/input_manager.py` — Keyboard/mouse input

## Step 1.1: Project Structure

### Commands
```bash
mkdir -p panda_tanks/{screens,entities,systems,ui,data/levels,shaders}
touch panda_tanks/__init__.py
touch panda_tanks/screens/__init__.py
touch panda_tanks/entities/__init__.py
touch panda_tanks/systems/__init__.py
touch panda_tanks/ui/__init__.py
```

### Verification
```bash
ls -la panda_tanks/
# Should show: screens/ entities/ systems/ ui/ data/ shaders/ __init__.py
```

## Step 1.2: Configuration Module

### File: `panda_tanks/config.py`
```python
"""Global configuration and constants."""

from panda3d.core import Vec4

# Window settings
WINDOW_TITLE = "PANDA-TANKS"
WINDOW_SIZE = (1280, 720)
TARGET_FPS = 60

# TRON color palette (as Vec4 for Panda3D)
COLORS = {
    'cyan': Vec4(0, 1, 1, 1),
    'cyan_dark': Vec4(0, 0.67, 0.67, 1),
    'cyan_glow': Vec4(0, 1, 1, 0.5),
    'orange': Vec4(1, 0.4, 0, 1),
    'orange_dark': Vec4(0.67, 0.27, 0, 1),
    'magenta': Vec4(1, 0, 1, 1),
    'white': Vec4(1, 1, 1, 1),
    'black': Vec4(0, 0, 0, 1),
    'dark_blue': Vec4(0, 0.067, 0.133, 1),
    'grid_blue': Vec4(0, 0.067, 0.27, 1),
    'health_green': Vec4(0, 1, 0, 1),
    'damage_red': Vec4(1, 0, 0, 1),
}

# Gameplay constants
TANK_SPEED = 10.0
TANK_ROTATION_SPEED = 2.0
PROJECTILE_SPEED = 50.0
MAX_ENEMIES = 20
MAX_PROJECTILES = 100

# Screen names
SCREEN_MAIN_MENU = 'main_menu'
SCREEN_BUNKER = 'bunker'
SCREEN_BATTLE = 'battle'
SCREEN_RESULTS = 'results'
SCREEN_EDITOR = 'editor'
SCREEN_SETTINGS = 'settings'
```

### Verification
```bash
cd panda_tanks
python -c "from config import COLORS, WINDOW_TITLE; print(WINDOW_TITLE, COLORS['cyan'])"
# Should print: PANDA-TANKS Vec4(0, 1, 1, 1)
```

## Step 1.3: Base Screen Class

### File: `panda_tanks/screens/base_screen.py`
```python
"""Abstract base class for game screens."""

from abc import ABC, abstractmethod
from direct.showbase.DirectObject import DirectObject
from panda3d.core import NodePath


class BaseScreen(DirectObject, ABC):
    """
    Base class for all game screens.
    
    Each screen manages its own:
    - 3D scene content (via self.root)
    - 2D UI elements (via self.ui_root)
    - Input event handling
    - Update logic
    """
    
    def __init__(self, game):
        DirectObject.__init__(self)
        self.game = game
        self.is_active = False
        
        # Root nodes for organization
        self.root = NodePath(f'{self.__class__.__name__}_root')
        self.ui_root = NodePath(f'{self.__class__.__name__}_ui')
        
    @abstractmethod
    def setup(self):
        """Initialize screen content. Called once."""
        pass
    
    @abstractmethod
    def update(self, dt: float):
        """Update screen logic. Called every frame when active."""
        pass
    
    def show(self, **kwargs):
        """Show the screen and activate input handling."""
        self.root.reparentTo(self.game.render)
        self.ui_root.reparentTo(self.game.aspect2d)
        self.is_active = True
        self.on_show(**kwargs)
    
    def hide(self):
        """Hide the screen and deactivate input handling."""
        self.root.detachNode()
        self.ui_root.detachNode()
        self.is_active = False
        self.on_hide()
    
    def on_show(self, **kwargs):
        """Called when screen becomes visible. Override for custom behavior."""
        pass
    
    def on_hide(self):
        """Called when screen is hidden. Override for cleanup."""
        pass
    
    def destroy(self):
        """Clean up all resources."""
        self.ignoreAll()
        self.root.removeNode()
        self.ui_root.removeNode()
```

## Step 1.4: Test Screen (Placeholder)

### File: `panda_tanks/screens/test_screen.py`
```python
"""Test screen for verifying the screen system works."""

from direct.gui.DirectGui import DirectLabel
from panda3d.core import LineSegs, Vec4

from .base_screen import BaseScreen
from ..config import COLORS


class TestScreen(BaseScreen):
    """Simple test screen with wireframe box and label."""
    
    def setup(self):
        # Create a wireframe box
        self._create_wireframe_box()
        
        # Create a label
        self.label = DirectLabel(
            text="PANDA-TANKS\nPress ESC to quit",
            text_fg=COLORS['cyan'],
            text_scale=0.1,
            text_align=0,  # Center
            pos=(0, 0, 0.5),
            parent=self.ui_root
        )
        
        # Accept input
        self.accept('escape', self.game.userExit)
        self.accept('arrow_up', self._rotate_box, [1])
        self.accept('arrow_down', self._rotate_box, [-1])
    
    def _create_wireframe_box(self):
        """Create a simple wireframe cube."""
        lines = LineSegs('test_box')
        lines.setColor(COLORS['cyan'])
        lines.setThickness(2)
        
        # Box corners
        s = 2  # size
        corners = [
            (-s, -s, -s), (s, -s, -s), (s, s, -s), (-s, s, -s),
            (-s, -s, s), (s, -s, s), (s, s, s), (-s, s, s)
        ]
        
        # Bottom face
        for i in range(4):
            lines.moveTo(*corners[i])
            lines.drawTo(*corners[(i + 1) % 4])
        
        # Top face
        for i in range(4, 8):
            lines.moveTo(*corners[i])
            lines.drawTo(*corners[4 + (i - 3) % 4])
        
        # Vertical edges
        for i in range(4):
            lines.moveTo(*corners[i])
            lines.drawTo(*corners[i + 4])
        
        self.box = self.root.attachNewNode(lines.create())
        self.box.setPos(0, 20, 0)
        self.rotation_speed = 0
    
    def _rotate_box(self, direction):
        self.rotation_speed = direction * 50
    
    def update(self, dt: float):
        if hasattr(self, 'box'):
            self.box.setH(self.box.getH() + self.rotation_speed * dt)
            # Dampen rotation
            self.rotation_speed *= 0.98
```

## Step 1.5: Input Manager

### File: `panda_tanks/systems/input_manager.py`
```python
"""Centralized input handling."""

from dataclasses import dataclass, field
from direct.showbase.DirectObject import DirectObject
from panda3d.core import Vec2


@dataclass
class InputState:
    """Current state of all inputs."""
    # Movement
    forward: bool = False
    backward: bool = False
    left: bool = False
    right: bool = False
    
    # Combat
    fire_primary: bool = False
    fire_secondary: bool = False
    use_ability: bool = False
    
    # Mouse
    mouse_pos: Vec2 = field(default_factory=lambda: Vec2(0, 0))
    mouse_delta: Vec2 = field(default_factory=lambda: Vec2(0, 0))
    
    # UI
    pause: bool = False
    toggle_view: bool = False


class InputManager(DirectObject):
    """
    Manages keyboard and mouse input.
    
    Usage:
        input_mgr = InputManager(game)
        # In update loop:
        input_mgr.update()
        if input_mgr.state.forward:
            move_forward()
    """
    
    def __init__(self, game):
        DirectObject.__init__(self)
        self.game = game
        self.state = InputState()
        self.enabled = True
        
        self._pressed = set()
        self._last_mouse = Vec2(0, 0)
        self._single_press = {}
        
        self._setup_bindings()
    
    def _setup_bindings(self):
        """Register all key bindings."""
        bindings = {
            'forward': ['w', 'arrow_up'],
            'backward': ['s', 'arrow_down'],
            'left': ['a', 'arrow_left'],
            'right': ['d', 'arrow_right'],
            'use_ability': ['space'],
            'pause': ['escape'],
            'toggle_view': ['v'],
        }
        
        for action, keys in bindings.items():
            for key in keys:
                self.accept(key, self._on_press, [action])
                self.accept(f'{key}-up', self._on_release, [action])
        
        # Mouse buttons
        self.accept('mouse1', self._on_mouse, ['primary', True])
        self.accept('mouse1-up', self._on_mouse, ['primary', False])
        self.accept('mouse3', self._on_mouse, ['secondary', True])
        self.accept('mouse3-up', self._on_mouse, ['secondary', False])
    
    def _on_press(self, action: str):
        if self.enabled:
            self._pressed.add(action)
    
    def _on_release(self, action: str):
        self._pressed.discard(action)
    
    def _on_mouse(self, button: str, pressed: bool):
        if not self.enabled:
            return
        if button == 'primary':
            self.state.fire_primary = pressed
        elif button == 'secondary':
            self.state.fire_secondary = pressed
    
    def update(self):
        """Update input state. Call once per frame."""
        if not self.enabled:
            return
        
        # Update movement
        self.state.forward = 'forward' in self._pressed
        self.state.backward = 'backward' in self._pressed
        self.state.left = 'left' in self._pressed
        self.state.right = 'right' in self._pressed
        self.state.use_ability = 'use_ability' in self._pressed
        
        # Single-press actions (only true for one frame)
        for action in ['pause', 'toggle_view']:
            was_pressed = self._single_press.get(action, False)
            is_pressed = action in self._pressed
            setattr(self.state, action, is_pressed and not was_pressed)
            self._single_press[action] = is_pressed
        
        # Mouse position
        if self.game.mouseWatcherNode.hasMouse():
            m = self.game.mouseWatcherNode.getMouse()
            current = Vec2(m.x, m.y)
            self.state.mouse_delta = current - self._last_mouse
            self.state.mouse_pos = current
            self._last_mouse = current
        else:
            self.state.mouse_delta = Vec2(0, 0)
    
    def set_enabled(self, enabled: bool):
        """Enable or disable input processing."""
        self.enabled = enabled
        if not enabled:
            self._pressed.clear()
```

## Step 1.6: Main Application

### File: `panda_tanks/main.py`
```python
"""PANDA-TANKS main application entry point."""

from direct.showbase.ShowBase import ShowBase
from direct.task import Task
from panda3d.core import WindowProperties, loadPrcFileData

from .config import WINDOW_TITLE, WINDOW_SIZE, COLORS
from .systems.input_manager import InputManager
from .screens.test_screen import TestScreen


# Configure Panda3D before ShowBase init
loadPrcFileData('', f'''
    window-title {WINDOW_TITLE}
    win-size {WINDOW_SIZE[0]} {WINDOW_SIZE[1]}
    show-frame-rate-meter #t
    sync-video #t
''')


class PandaTanks(ShowBase):
    """Main game application."""
    
    def __init__(self):
        ShowBase.__init__(self)
        
        # Set background color (TRON dark blue)
        self.setBackgroundColor(COLORS['dark_blue'])
        
        # Disable default mouse camera control
        self.disableMouse()
        
        # Initialize input manager
        self.input_manager = InputManager(self)
        
        # Screen management
        self.screens = {}
        self.current_screen = None
        
        # Register test screen
        self._register_screen('test', TestScreen(self))
        
        # Start with test screen
        self.switch_screen('test')
        
        # Main update task
        self.taskMgr.add(self._update, 'main_update')
        
        print("PANDA-TANKS initialized. Press ESC to quit.")
    
    def _register_screen(self, name: str, screen):
        """Register a screen for later use."""
        screen.setup()
        self.screens[name] = screen
    
    def switch_screen(self, name: str, **kwargs):
        """Switch to a different screen."""
        if self.current_screen:
            self.current_screen.hide()
        
        if name in self.screens:
            self.current_screen = self.screens[name]
            self.current_screen.show(**kwargs)
        else:
            print(f"Warning: Screen '{name}' not found")
    
    def _update(self, task):
        """Main game loop."""
        dt = globalClock.getDt()
        
        # Update input
        self.input_manager.update()
        
        # Update current screen
        if self.current_screen and self.current_screen.is_active:
            self.current_screen.update(dt)
        
        return Task.cont


def main():
    """Entry point."""
    game = PandaTanks()
    game.run()


if __name__ == '__main__':
    main()
```

## Step 1.7: Package Entry Point

### File: `panda_tanks/__init__.py`
```python
"""PANDA-TANKS game package."""

from .main import PandaTanks, main

__version__ = '0.1.0'
__all__ = ['PandaTanks', 'main']
```

### File: `run.py` (in project root)
```python
#!/usr/bin/env python
"""Run PANDA-TANKS."""

from panda_tanks import main

if __name__ == '__main__':
    main()
```

## Phase 1 Verification

```bash
# From project root (parent of panda_tanks/)
python run.py
```

### Expected Behavior
- Window opens with title "PANDA-TANKS"
- Dark blue background
- Cyan wireframe cube visible
- "PANDA-TANKS" text displayed
- Arrow keys rotate the cube
- ESC quits the application
- FPS counter shown in corner

### Verification Checklist
- [ ] Window opens at 1280x720
- [ ] Background is dark blue
- [ ] Wireframe cube renders in cyan
- [ ] Text renders correctly
- [ ] Input system responds to keys
- [ ] ESC closes application
- [ ] No errors in console

---

# Phase 2: Entities

**Goal:** Create renderable game objects (tank, arena, projectiles).

**Duration:** ~3 hours

**Prerequisites:** Phase 1 complete

**Deliverables:**
- `entities/tank.py` — Wireframe tank with turret
- `entities/arena.py` — Level geometry renderer
- `entities/projectile.py` — Energy projectile
- `entities/grid.py` — Ground grid

## Step 2.1: Wireframe Utilities

### File: `panda_tanks/entities/wireframe.py`
```python
"""Utilities for creating wireframe geometry."""

from panda3d.core import NodePath, Vec3, Vec4, LineSegs
import math


def create_wireframe_box(size: Vec3, color: Vec4, thickness: float = 2) -> NodePath:
    """Create a wireframe box centered at origin."""
    lines = LineSegs('wireframe_box')
    lines.setColor(color)
    lines.setThickness(thickness)
    
    hx, hy, hz = size.x / 2, size.y / 2, size.z / 2
    
    # 8 corners
    corners = [
        Vec3(-hx, -hy, -hz), Vec3(hx, -hy, -hz),
        Vec3(hx, hy, -hz), Vec3(-hx, hy, -hz),
        Vec3(-hx, -hy, hz), Vec3(hx, -hy, hz),
        Vec3(hx, hy, hz), Vec3(-hx, hy, hz),
    ]
    
    # 12 edges
    edges = [
        (0,1), (1,2), (2,3), (3,0),  # bottom
        (4,5), (5,6), (6,7), (7,4),  # top
        (0,4), (1,5), (2,6), (3,7),  # sides
    ]
    
    for a, b in edges:
        lines.moveTo(corners[a])
        lines.drawTo(corners[b])
    
    return NodePath(lines.create())


def create_wireframe_cylinder(radius: float, height: float, 
                               segments: int, color: Vec4,
                               thickness: float = 2) -> NodePath:
    """Create a wireframe cylinder (as n-sided prism)."""
    lines = LineSegs('wireframe_cylinder')
    lines.setColor(color)
    lines.setThickness(thickness)
    
    hz = height / 2
    
    # Generate circle points
    def circle_point(i, z):
        angle = 2 * math.pi * i / segments
        return Vec3(radius * math.cos(angle), radius * math.sin(angle), z)
    
    # Bottom circle
    for i in range(segments):
        lines.moveTo(circle_point(i, -hz))
        lines.drawTo(circle_point((i + 1) % segments, -hz))
    
    # Top circle
    for i in range(segments):
        lines.moveTo(circle_point(i, hz))
        lines.drawTo(circle_point((i + 1) % segments, hz))
    
    # Vertical edges
    for i in range(segments):
        lines.moveTo(circle_point(i, -hz))
        lines.drawTo(circle_point(i, hz))
    
    return NodePath(lines.create())


def create_grid(width: int, height: int, cell_size: float,
                color: Vec4, thickness: float = 1) -> NodePath:
    """Create a flat grid on the XY plane."""
    lines = LineSegs('grid')
    lines.setColor(color)
    lines.setThickness(thickness)
    
    w = width * cell_size
    h = height * cell_size
    
    # Horizontal lines
    for y in range(height + 1):
        lines.moveTo(0, y * cell_size, 0)
        lines.drawTo(w, y * cell_size, 0)
    
    # Vertical lines
    for x in range(width + 1):
        lines.moveTo(x * cell_size, 0, 0)
        lines.drawTo(x * cell_size, h, 0)
    
    return NodePath(lines.create())
```

## Step 2.2: Tank Entity

### File: `panda_tanks/entities/tank.py`
```python
"""Tank entity with body and rotating turret."""

from panda3d.core import NodePath, Vec3, Vec4
import math

from .wireframe import create_wireframe_box, create_wireframe_cylinder
from ..config import COLORS


class Tank:
    """
    Wireframe tank with independent turret rotation.
    
    Structure:
    - Body: 2x3x0.5 box
    - Treads: Two thin boxes on sides
    - Turret: Cylinder + cannon box
    """
    
    def __init__(self, parent: NodePath, color: Vec4 = None, is_player: bool = False):
        self.color = color or COLORS['cyan']
        self.is_player = is_player
        
        # Root node
        self.root = NodePath('tank')
        self.root.reparentTo(parent)
        
        # Build geometry
        self._build_body()
        self._build_turret()
        
        # State
        self._rotation = 0.0
        self._turret_rotation = 0.0
    
    def _build_body(self):
        """Construct tank body and treads."""
        # Main body
        self.body = create_wireframe_box(Vec3(2, 3, 0.5), self.color)
        self.body.reparentTo(self.root)
        self.body.setPos(0, 0, 0.25)
        
        # Left tread
        left_tread = create_wireframe_box(Vec3(0.3, 3.2, 0.6), self.color)
        left_tread.reparentTo(self.root)
        left_tread.setPos(-1.15, 0, 0.3)
        
        # Right tread
        right_tread = create_wireframe_box(Vec3(0.3, 3.2, 0.6), self.color)
        right_tread.reparentTo(self.root)
        right_tread.setPos(1.15, 0, 0.3)
    
    def _build_turret(self):
        """Construct turret and cannon."""
        # Turret pivot point
        self.turret_pivot = NodePath('turret_pivot')
        self.turret_pivot.reparentTo(self.root)
        self.turret_pivot.setPos(0, 0, 0.5)
        
        # Turret base (octagonal cylinder)
        turret_base = create_wireframe_cylinder(0.6, 0.3, 8, self.color)
        turret_base.reparentTo(self.turret_pivot)
        turret_base.setPos(0, 0, 0.15)
        
        # Cannon
        self.cannon = create_wireframe_box(Vec3(0.15, 2.0, 0.15), self.color)
        self.cannon.reparentTo(self.turret_pivot)
        self.cannon.setPos(0, 1.2, 0.3)
    
    @property
    def position(self) -> Vec3:
        return self.root.getPos()
    
    @position.setter
    def position(self, pos: Vec3):
        self.root.setPos(pos)
    
    @property
    def rotation(self) -> float:
        """Body rotation in radians."""
        return self._rotation
    
    @rotation.setter
    def rotation(self, angle: float):
        self._rotation = angle
        self.root.setH(math.degrees(angle))
    
    @property
    def turret_rotation(self) -> float:
        """Turret rotation in radians (relative to body)."""
        return self._turret_rotation
    
    @turret_rotation.setter
    def turret_rotation(self, angle: float):
        self._turret_rotation = angle
        self.turret_pivot.setH(math.degrees(angle))
    
    def get_cannon_tip(self) -> Vec3:
        """Get world position of cannon end (for spawning projectiles)."""
        return self.cannon.getPos(self.root.getParent()) + Vec3(0, 2.5, 0.3)
    
    def get_forward(self) -> Vec3:
        """Get forward direction vector."""
        return Vec3(math.sin(self._rotation), math.cos(self._rotation), 0)
    
    def get_turret_forward(self) -> Vec3:
        """Get turret forward direction vector."""
        total_angle = self._rotation + self._turret_rotation
        return Vec3(math.sin(total_angle), math.cos(total_angle), 0)
    
    def destroy(self):
        """Remove tank from scene."""
        self.root.removeNode()
```

## Step 2.3: Projectile Entity

### File: `panda_tanks/entities/projectile.py`
```python
"""Energy projectile entity."""

from panda3d.core import NodePath, Vec3, Vec4, LineSegs
import uuid

from ..config import COLORS


class Projectile:
    """
    Fast-moving energy projectile.
    
    Visual: Small glowing sphere made of crossing circles.
    """
    
    def __init__(self, parent: NodePath, position: Vec3, velocity: Vec3,
                 damage: float = 15, owner_id: str = '', 
                 color: Vec4 = None, lifetime: float = 3.0):
        self.id = str(uuid.uuid4())[:8]
        self.owner_id = owner_id
        self.damage = damage
        self.lifetime = lifetime
        self.max_lifetime = lifetime
        self.velocity = Vec3(velocity)
        self.color = color or COLORS['cyan']
        self.alive = True
        
        # Create visual
        self.root = NodePath(f'projectile_{self.id}')
        self.root.reparentTo(parent)
        self.root.setPos(position)
        
        self._build_visual()
    
    def _build_visual(self):
        """Create projectile visual (small sphere of lines)."""
        lines = LineSegs('projectile_visual')
        lines.setColor(self.color)
        lines.setThickness(2)
        
        radius = 0.2
        segments = 8
        
        # Three crossing circles (XY, XZ, YZ planes)
        import math
        for plane in [(0, 1, 2), (0, 2, 1), (1, 2, 0)]:
            for i in range(segments):
                a1 = 2 * math.pi * i / segments
                a2 = 2 * math.pi * (i + 1) / segments
                
                p1 = [0, 0, 0]
                p2 = [0, 0, 0]
                
                p1[plane[0]] = radius * math.cos(a1)
                p1[plane[1]] = radius * math.sin(a1)
                p2[plane[0]] = radius * math.cos(a2)
                p2[plane[1]] = radius * math.sin(a2)
                
                lines.moveTo(*p1)
                lines.drawTo(*p2)
        
        self.visual = self.root.attachNewNode(lines.create())
    
    @property
    def position(self) -> Vec3:
        return self.root.getPos()
    
    @position.setter
    def position(self, pos: Vec3):
        self.root.setPos(pos)
    
    def update(self, dt: float) -> bool:
        """
        Update projectile position.
        Returns False if projectile should be destroyed.
        """
        if not self.alive:
            return False
        
        # Move
        self.root.setPos(self.root.getPos() + self.velocity * dt)
        
        # Spin for visual effect
        self.visual.setH(self.visual.getH() + 360 * dt)
        
        # Lifetime
        self.lifetime -= dt
        if self.lifetime <= 0:
            self.alive = False
            return False
        
        return True
    
    def destroy(self):
        """Remove projectile from scene."""
        self.alive = False
        self.root.removeNode()
```

## Step 2.4: Arena Entity

### File: `panda_tanks/entities/arena.py`
```python
"""Arena/level geometry renderer."""

from panda3d.core import NodePath, Vec3, Vec4, LineSegs
from dataclasses import dataclass
from typing import List

from .wireframe import create_wireframe_box, create_grid
from ..config import COLORS


@dataclass
class Wall:
    """Wall definition."""
    position: Vec3
    size: Vec3
    is_barrier: bool = False  # Barrier = cover (doesn't block movement)


class Arena:
    """
    Renders level geometry.
    
    - Ground grid
    - Walls (solid obstacles)
    - Barriers (cover, blocks projectiles)
    - Boundary glow
    """
    
    WALL_HEIGHT = 4.0
    BARRIER_HEIGHT = 2.0
    
    def __init__(self, parent: NodePath, width: int = 50, height: int = 50):
        self.width = width
        self.height = height
        
        self.root = NodePath('arena')
        self.root.reparentTo(parent)
        
        self.walls: List[Wall] = []
        self._wall_nodes: List[NodePath] = []
        
        self._build_ground()
        self._build_boundary()
    
    def _build_ground(self):
        """Create ground grid."""
        self.ground = create_grid(
            self.width, self.height, 1.0,
            COLORS['grid_blue'], thickness=1
        )
        self.ground.reparentTo(self.root)
        # Center the grid
        self.ground.setPos(-self.width / 2, -self.height / 2, 0)
    
    def _build_boundary(self):
        """Create glowing boundary lines."""
        lines = LineSegs('boundary')
        lines.setColor(COLORS['cyan_glow'])
        lines.setThickness(4)
        
        hw, hh = self.width / 2, self.height / 2
        
        lines.moveTo(-hw, -hh, 0)
        lines.drawTo(hw, -hh, 0)
        lines.drawTo(hw, hh, 0)
        lines.drawTo(-hw, hh, 0)
        lines.drawTo(-hw, -hh, 0)
        
        # Add vertical lines at corners
        for x, y in [(-hw, -hh), (hw, -hh), (hw, hh), (-hw, hh)]:
            lines.moveTo(x, y, 0)
            lines.drawTo(x, y, 3)
        
        self.boundary = self.root.attachNewNode(lines.create())
    
    def add_wall(self, position: Vec3, size: Vec3, is_barrier: bool = False):
        """Add a wall or barrier to the arena."""
        wall = Wall(position, size, is_barrier)
        self.walls.append(wall)
        
        # Create visual
        height = self.BARRIER_HEIGHT if is_barrier else self.WALL_HEIGHT
        color = COLORS['cyan_glow'] if is_barrier else COLORS['cyan']
        
        wall_node = create_wireframe_box(
            Vec3(size.x, size.y, height),
            color
        )
        wall_node.reparentTo(self.root)
        wall_node.setPos(position.x, position.y, height / 2)
        
        self._wall_nodes.append(wall_node)
    
    def clear_walls(self):
        """Remove all walls."""
        for node in self._wall_nodes:
            node.removeNode()
        self._wall_nodes.clear()
        self.walls.clear()
    
    def destroy(self):
        """Clean up arena."""
        self.root.removeNode()
```

## Step 2.5: Entity Test Screen

### File: `panda_tanks/screens/entity_test_screen.py`
```python
"""Test screen for entity rendering."""

from panda3d.core import Vec3

from .base_screen import BaseScreen
from ..entities.tank import Tank
from ..entities.arena import Arena
from ..entities.projectile import Projectile
from ..config import COLORS


class EntityTestScreen(BaseScreen):
    """Test all entities together."""
    
    def setup(self):
        # Create arena
        self.arena = Arena(self.root, width=30, height=30)
        
        # Add some walls
        self.arena.add_wall(Vec3(5, 5, 0), Vec3(4, 1, 0))
        self.arena.add_wall(Vec3(-5, -5, 0), Vec3(1, 6, 0))
        self.arena.add_wall(Vec3(0, 10, 0), Vec3(8, 1, 0), is_barrier=True)
        
        # Create player tank
        self.player_tank = Tank(self.root, COLORS['cyan'], is_player=True)
        self.player_tank.position = Vec3(0, -5, 0)
        
        # Create enemy tank
        self.enemy_tank = Tank(self.root, COLORS['orange'])
        self.enemy_tank.position = Vec3(5, 8, 0)
        self.enemy_tank.rotation = 3.14  # Face opposite direction
        
        # Projectiles list
        self.projectiles = []
        
        # Camera setup
        self.game.camera.setPos(0, -25, 20)
        self.game.camera.lookAt(0, 0, 0)
        
        # Input
        self.accept('space', self._fire_projectile)
        self.accept('escape', self._back_to_menu)
    
    def _fire_projectile(self):
        """Fire a projectile from player tank."""
        pos = self.player_tank.position + Vec3(0, 0, 0.5)
        vel = self.player_tank.get_turret_forward() * 30
        
        proj = Projectile(self.root, pos, vel, owner_id='player')
        self.projectiles.append(proj)
    
    def _back_to_menu(self):
        self.game.switch_screen('test')
    
    def update(self, dt: float):
        # Rotate player turret with mouse
        mouse = self.game.input_manager.state.mouse_pos
        self.player_tank.turret_rotation += mouse.x * 2 * dt
        
        # Move player with keys
        inp = self.game.input_manager.state
        move = Vec3(0, 0, 0)
        if inp.forward:
            move += self.player_tank.get_forward() * 5 * dt
        if inp.backward:
            move -= self.player_tank.get_forward() * 5 * dt
        if inp.left:
            self.player_tank.rotation += 2 * dt
        if inp.right:
            self.player_tank.rotation -= 2 * dt
        
        self.player_tank.position = self.player_tank.position + move
        
        # Update projectiles
        for proj in self.projectiles[:]:
            if not proj.update(dt):
                proj.destroy()
                self.projectiles.remove(proj)
    
    def on_hide(self):
        # Clean up projectiles
        for proj in self.projectiles:
            proj.destroy()
        self.projectiles.clear()
```

## Phase 2 Verification

Update `main.py` to include the entity test screen:

```python
# In main.py, add import and registration:
from .screens.entity_test_screen import EntityTestScreen

# In __init__, add:
self._register_screen('entity_test', EntityTestScreen(self))
```

Update test_screen.py to allow switching:
```python
# Add to TestScreen.setup():
self.accept('e', lambda: self.game.switch_screen('entity_test'))
```

### Test Commands
```bash
python run.py
# Press 'e' to switch to entity test
```

### Expected Behavior
- Arena with grid and boundary visible
- Player tank (cyan) controllable with WASD
- Enemy tank (orange) visible
- Walls and barriers rendered
- Space fires projectiles
- Projectiles move and expire

### Verification Checklist
- [ ] Arena grid renders correctly
- [ ] Boundary glow visible
- [ ] Walls render at correct height
- [ ] Barriers render shorter and semi-transparent
- [ ] Player tank moves with WASD
- [ ] Player tank rotates with A/D
- [ ] Turret follows mouse (somewhat)
- [ ] Projectiles fire with space
- [ ] Projectiles despawn after 3 seconds
- [ ] No errors in console

---

# Phase 3: Systems

**Goal:** Implement physics, AI, and combat mechanics.

**Duration:** ~4 hours

**Prerequisites:** Phase 2 complete

**Deliverables:**
- `systems/physics.py` — Bullet physics integration
- `systems/ai.py` — Enemy AI state machine
- `systems/combat.py` — Damage and weapon systems
- `types/game_types.py` — Game state dataclasses

## Step 3.1: Game Types

### File: `panda_tanks/types/__init__.py`
```python
"""Type definitions for game state."""

from .game_types import *
```

### File: `panda_tanks/types/game_types.py`
```python
"""Core game type definitions."""

from dataclasses import dataclass, field
from typing import List, Optional, Dict
from panda3d.core import Vec3
from enum import Enum
import uuid


class WeaponType(Enum):
    STANDARD = 'standard'
    RAPID = 'rapid'
    HEAVY = 'heavy'
    SPREAD = 'spread'


class AbilityType(Enum):
    BOOST = 'boost'
    CLOAK = 'cloak'
    SCAN = 'scan'
    REPAIR = 'repair'


class AIState(Enum):
    IDLE = 'idle'
    PATROL = 'patrol'
    CHASE = 'chase'
    ATTACK = 'attack'
    FLEE = 'flee'


@dataclass
class WeaponStats:
    """Weapon configuration."""
    damage: float
    fire_rate: float  # shots per second
    projectile_speed: float
    spread_count: int = 1
    spread_angle: float = 0.0
    splash_radius: float = 0.0


WEAPON_CONFIGS: Dict[WeaponType, WeaponStats] = {
    WeaponType.STANDARD: WeaponStats(15, 2.0, 50),
    WeaponType.RAPID: WeaponStats(8, 5.0, 60),
    WeaponType.HEAVY: WeaponStats(40, 0.5, 30, splash_radius=5),
    WeaponType.SPREAD: WeaponStats(10, 1.0, 45, spread_count=3, spread_angle=0.26),
}


@dataclass
class TankState:
    """State for any tank (player or enemy)."""
    id: str = field(default_factory=lambda: str(uuid.uuid4())[:8])
    position: Vec3 = field(default_factory=lambda: Vec3(0, 0, 0))
    rotation: float = 0.0
    turret_rotation: float = 0.0
    velocity: Vec3 = field(default_factory=lambda: Vec3(0, 0, 0))
    health: float = 100.0
    max_health: float = 100.0
    weapon: WeaponType = WeaponType.STANDARD
    weapon_cooldown: float = 0.0
    is_alive: bool = True


@dataclass
class AIBehavior:
    """AI state and configuration."""
    state: AIState = AIState.IDLE
    target_position: Optional[Vec3] = None
    last_known_player_pos: Optional[Vec3] = None
    state_timer: float = 0.0
    detection_range: float = 30.0
    attack_range: float = 25.0
    accuracy: float = 0.7
    move_speed: float = 5.0


@dataclass
class GameStats:
    """Statistics for current game session."""
    enemies_destroyed: int = 0
    total_enemies: int = 0
    shots_fired: int = 0
    shots_hit: int = 0
    damage_taken: float = 0.0
    time_seconds: float = 0.0
```

## Step 3.2: Physics System

### File: `panda_tanks/systems/physics.py`
```python
"""Physics system using Panda3D's built-in collision."""

from panda3d.core import (
    CollisionTraverser, CollisionHandlerPusher, CollisionHandlerQueue,
    CollisionNode, CollisionSphere, CollisionBox, CollisionRay,
    BitMask32, Vec3, Point3, NodePath
)
from typing import Optional, Tuple, List
from dataclasses import dataclass


# Collision masks
MASK_TANK = BitMask32.bit(0)
MASK_WALL = BitMask32.bit(1)
MASK_PROJECTILE = BitMask32.bit(2)
MASK_PICKUP = BitMask32.bit(3)


@dataclass
class CollisionResult:
    """Result of a collision check."""
    hit: bool
    entity_id: str = ''
    point: Vec3 = None
    normal: Vec3 = None


class PhysicsSystem:
    """
    Handles collision detection and response.
    
    Uses Panda3D's collision system for:
    - Tank-wall collisions (pusher)
    - Projectile-tank/wall collisions (queue)
    - Line-of-sight raycasts
    """
    
    def __init__(self, game):
        self.game = game
        
        # Collision traverser
        self.traverser = CollisionTraverser('physics')
        
        # Pusher for tank movement
        self.pusher = CollisionHandlerPusher()
        
        # Queue for projectile hits
        self.queue = CollisionHandlerQueue()
        
        # Store collision nodes by entity id
        self.colliders = {}
    
    def create_tank_collider(self, entity_id: str, tank_node: NodePath) -> NodePath:
        """Create collision geometry for a tank."""
        cnode = CollisionNode(f'tank_{entity_id}')
        cnode.addSolid(CollisionSphere(0, 0, 0.5, 1.5))
        cnode.setFromCollideMask(MASK_WALL | MASK_TANK)
        cnode.setIntoCollideMask(MASK_PROJECTILE | MASK_TANK)
        
        cnp = tank_node.attachNewNode(cnode)
        cnp.setTag('entity_id', entity_id)
        cnp.setTag('type', 'tank')
        
        # Add to pusher for wall collision response
        self.traverser.addCollider(cnp, self.pusher)
        self.pusher.addCollider(cnp, tank_node)
        
        self.colliders[entity_id] = cnp
        return cnp
    
    def create_wall_collider(self, entity_id: str, parent: NodePath,
                             position: Vec3, size: Vec3) -> NodePath:
        """Create collision geometry for a wall."""
        cnode = CollisionNode(f'wall_{entity_id}')
        cnode.addSolid(CollisionBox(Point3(0, 0, 2), size.x/2, size.y/2, 2))
        cnode.setFromCollideMask(BitMask32.allOff())
        cnode.setIntoCollideMask(MASK_WALL | MASK_PROJECTILE)
        
        cnp = parent.attachNewNode(cnode)
        cnp.setPos(position)
        cnp.setTag('entity_id', entity_id)
        cnp.setTag('type', 'wall')
        
        # Add as pusher target
        self.pusher.addInPattern('%fn-into-%in')
        
        self.colliders[entity_id] = cnp
        return cnp
    
    def create_projectile_collider(self, entity_id: str, 
                                    projectile_node: NodePath) -> NodePath:
        """Create collision geometry for a projectile."""
        cnode = CollisionNode(f'projectile_{entity_id}')
        cnode.addSolid(CollisionSphere(0, 0, 0, 0.3))
        cnode.setFromCollideMask(MASK_WALL | MASK_TANK)
        cnode.setIntoCollideMask(BitMask32.allOff())
        
        cnp = projectile_node.attachNewNode(cnode)
        cnp.setTag('entity_id', entity_id)
        cnp.setTag('type', 'projectile')
        
        # Add to queue for hit detection
        self.traverser.addCollider(cnp, self.queue)
        
        self.colliders[entity_id] = cnp
        return cnp
    
    def remove_collider(self, entity_id: str):
        """Remove a collider."""
        if entity_id in self.colliders:
            cnp = self.colliders[entity_id]
            self.traverser.removeCollider(cnp)
            cnp.removeNode()
            del self.colliders[entity_id]
    
    def update(self):
        """Run collision detection. Call once per frame."""
        self.traverser.traverse(self.game.render)
    
    def get_projectile_hits(self) -> List[Tuple[str, str, Vec3]]:
        """
        Get all projectile collision events.
        Returns list of (projectile_id, hit_entity_id, hit_point).
        """
        hits = []
        
        for i in range(self.queue.getNumEntries()):
            entry = self.queue.getEntry(i)
            
            from_node = entry.getFromNodePath()
            into_node = entry.getIntoNodePath()
            
            proj_id = from_node.getTag('entity_id')
            hit_id = into_node.getTag('entity_id')
            hit_point = entry.getSurfacePoint(self.game.render)
            
            if proj_id and hit_id:
                hits.append((proj_id, hit_id, Vec3(hit_point)))
        
        self.queue.clearEntries()
        return hits
    
    def raycast(self, origin: Vec3, direction: Vec3, 
                distance: float, mask: BitMask32 = MASK_WALL) -> CollisionResult:
        """
        Cast a ray and return first hit.
        Useful for line-of-sight checks.
        """
        # Create temporary ray
        ray_node = CollisionNode('raycast')
        ray = CollisionRay(origin, direction)
        ray_node.addSolid(ray)
        ray_node.setFromCollideMask(mask)
        ray_node.setIntoCollideMask(BitMask32.allOff())
        
        ray_np = self.game.render.attachNewNode(ray_node)
        
        # Create temporary queue
        ray_queue = CollisionHandlerQueue()
        ray_traverser = CollisionTraverser('raycast')
        ray_traverser.addCollider(ray_np, ray_queue)
        
        # Traverse
        ray_traverser.traverse(self.game.render)
        
        # Find closest hit within distance
        result = CollisionResult(hit=False)
        
        if ray_queue.getNumEntries() > 0:
            ray_queue.sortEntries()
            entry = ray_queue.getEntry(0)
            
            hit_point = entry.getSurfacePoint(self.game.render)
            hit_dist = (Vec3(hit_point) - origin).length()
            
            if hit_dist <= distance:
                result = CollisionResult(
                    hit=True,
                    entity_id=entry.getIntoNodePath().getTag('entity_id'),
                    point=Vec3(hit_point),
                    normal=Vec3(entry.getSurfaceNormal(self.game.render))
                )
        
        # Cleanup
        ray_np.removeNode()
        
        return result
    
    def check_line_of_sight(self, from_pos: Vec3, to_pos: Vec3) -> bool:
        """Check if there's clear line of sight between two points."""
        direction = to_pos - from_pos
        distance = direction.length()
        if distance < 0.1:
            return True
        
        result = self.raycast(from_pos, direction.normalized(), distance, MASK_WALL)
        return not result.hit
```

## Step 3.3: AI System

### File: `panda_tanks/systems/ai.py`
```python
"""Enemy AI behavior system."""

from panda3d.core import Vec3
from typing import Optional, Callable
import math
import random

from ..types.game_types import AIState, AIBehavior, TankState


# AI presets
AI_PRESETS = {
    'patrol': AIBehavior(
        detection_range=30, attack_range=25,
        accuracy=0.6, move_speed=3.0
    ),
    'guard': AIBehavior(
        detection_range=25, attack_range=20,
        accuracy=0.7, move_speed=4.0
    ),
    'aggressive': AIBehavior(
        detection_range=40, attack_range=30,
        accuracy=0.8, move_speed=6.0
    ),
}


class AISystem:
    """
    Updates enemy AI behavior.
    
    State machine:
    IDLE -> PATROL (wandering)
    PATROL -> CHASE (player detected)
    CHASE -> ATTACK (in range)
    ATTACK -> CHASE (player moved away)
    ATTACK/CHASE -> FLEE (low health)
    """
    
    def __init__(self, check_los: Callable[[Vec3, Vec3], bool]):
        """
        Args:
            check_los: Function to check line of sight between two points.
        """
        self.check_los = check_los
    
    def update(self, enemy: TankState, enemy_ai: AIBehavior,
               player: TankState, dt: float) -> dict:
        """
        Update AI for one enemy.
        
        Returns dict of state changes to apply:
        - 'position': new position
        - 'rotation': new rotation
        - 'turret_rotation': new turret rotation
        - 'should_fire': True if should fire
        """
        changes = {
            'should_fire': False
        }
        
        # Calculate distance and visibility
        to_player = player.position - enemy.position
        distance = to_player.length()
        can_see = self.check_los(
            enemy.position + Vec3(0, 0, 1),
            player.position + Vec3(0, 0, 1)
        )
        
        # Update timer
        enemy_ai.state_timer -= dt
        
        # Remember player position if visible
        if can_see:
            enemy_ai.last_known_player_pos = Vec3(player.position)
        
        # State transitions
        new_state = self._determine_state(
            enemy, enemy_ai, player, distance, can_see
        )
        
        if new_state != enemy_ai.state:
            enemy_ai.state = new_state
            enemy_ai.state_timer = 2.0
        
        # Execute behavior
        if enemy_ai.state == AIState.IDLE:
            pass
        
        elif enemy_ai.state == AIState.PATROL:
            changes.update(self._do_patrol(enemy, enemy_ai, dt))
        
        elif enemy_ai.state == AIState.CHASE:
            target = enemy_ai.last_known_player_pos or player.position
            changes.update(self._do_chase(enemy, enemy_ai, target, dt))
        
        elif enemy_ai.state == AIState.ATTACK:
            changes.update(self._do_attack(enemy, enemy_ai, player))
            changes['should_fire'] = enemy_ai.state_timer <= 1.5  # Attack after brief delay
        
        elif enemy_ai.state == AIState.FLEE:
            changes.update(self._do_flee(enemy, enemy_ai, player.position, dt))
        
        return changes
    
    def _determine_state(self, enemy: TankState, ai: AIBehavior,
                         player: TankState, distance: float, 
                         can_see: bool) -> AIState:
        """Determine appropriate AI state."""
        # Low health -> flee
        if enemy.health < enemy.max_health * 0.25:
            return AIState.FLEE
        
        if can_see:
            if distance <= ai.attack_range:
                return AIState.ATTACK
            elif distance <= ai.detection_range:
                return AIState.CHASE
        elif ai.last_known_player_pos:
            return AIState.CHASE
        
        return AIState.PATROL
    
    def _do_patrol(self, enemy: TankState, ai: AIBehavior, dt: float) -> dict:
        """Wander between random points."""
        changes = {}
        
        # Pick new patrol point if needed
        if ai.target_position is None or ai.state_timer <= 0:
            ai.target_position = Vec3(
                enemy.position.x + random.uniform(-15, 15),
                enemy.position.y + random.uniform(-15, 15),
                0
            )
            ai.state_timer = 5.0
        
        # Move toward target
        new_pos, new_rot = self._move_toward(
            enemy.position, ai.target_position, 
            ai.move_speed * 0.5 * dt
        )
        
        changes['position'] = new_pos
        changes['rotation'] = new_rot
        
        return changes
    
    def _do_chase(self, enemy: TankState, ai: AIBehavior,
                  target: Vec3, dt: float) -> dict:
        """Move toward target."""
        new_pos, new_rot = self._move_toward(
            enemy.position, target, ai.move_speed * dt
        )
        
        return {'position': new_pos, 'rotation': new_rot}
    
    def _do_attack(self, enemy: TankState, ai: AIBehavior,
                   player: TankState) -> dict:
        """Aim at player."""
        turret_rot = self._aim_at(enemy.position, player.position, ai.accuracy)
        return {'turret_rotation': turret_rot - enemy.rotation}
    
    def _do_flee(self, enemy: TankState, ai: AIBehavior,
                 player_pos: Vec3, dt: float) -> dict:
        """Move away from player."""
        away = enemy.position - player_pos
        if away.length() < 0.1:
            away = Vec3(random.uniform(-1, 1), random.uniform(-1, 1), 0)
        
        flee_target = enemy.position + away.normalized() * 20
        new_pos, new_rot = self._move_toward(
            enemy.position, flee_target, ai.move_speed * 1.2 * dt
        )
        
        return {'position': new_pos, 'rotation': new_rot}
    
    def _move_toward(self, current: Vec3, target: Vec3, 
                     speed: float) -> tuple:
        """Move toward target. Returns (new_position, rotation)."""
        direction = target - current
        distance = direction.length()
        
        if distance < 0.1:
            return current, math.atan2(direction.x, direction.y)
        
        direction = direction.normalized()
        move_dist = min(distance, speed)
        
        new_pos = current + direction * move_dist
        rotation = math.atan2(direction.x, direction.y)
        
        return new_pos, rotation
    
    def _aim_at(self, from_pos: Vec3, to_pos: Vec3, accuracy: float) -> float:
        """Calculate angle to aim at target with some inaccuracy."""
        direction = to_pos - from_pos
        angle = math.atan2(direction.x, direction.y)
        
        # Add inaccuracy
        deviation = (1 - accuracy) * 0.5
        angle += random.uniform(-deviation, deviation)
        
        return angle
```

## Step 3.4: Combat System

### File: `panda_tanks/systems/combat.py`
```python
"""Combat and weapon systems."""

from panda3d.core import Vec3
from typing import List, Tuple, Optional
import math

from ..types.game_types import WeaponType, WeaponStats, WEAPON_CONFIGS, TankState


class CombatSystem:
    """
    Handles weapon firing and damage.
    """
    
    def __init__(self):
        self.pending_damage: List[Tuple[str, float, Vec3]] = []
    
    def can_fire(self, tank: TankState) -> bool:
        """Check if tank can fire."""
        return tank.weapon_cooldown <= 0 and tank.is_alive
    
    def fire_weapon(self, tank: TankState, 
                    turret_forward: Vec3) -> List[Tuple[Vec3, Vec3, float]]:
        """
        Fire tank's weapon.
        
        Returns list of (position, velocity, damage) for each projectile.
        """
        if not self.can_fire(tank):
            return []
        
        weapon = WEAPON_CONFIGS[tank.weapon]
        tank.weapon_cooldown = 1.0 / weapon.fire_rate
        
        # Spawn position (cannon tip)
        spawn_pos = tank.position + Vec3(0, 0, 0.8) + turret_forward * 2.5
        
        projectiles = []
        
        for i in range(weapon.spread_count):
            # Calculate spread angle
            if weapon.spread_count > 1:
                spread_offset = (i - (weapon.spread_count - 1) / 2) * weapon.spread_angle
            else:
                spread_offset = 0
            
            # Rotate velocity
            base_angle = math.atan2(turret_forward.x, turret_forward.y)
            angle = base_angle + spread_offset
            
            velocity = Vec3(
                math.sin(angle) * weapon.projectile_speed,
                math.cos(angle) * weapon.projectile_speed,
                0
            )
            
            projectiles.append((Vec3(spawn_pos), velocity, weapon.damage))
        
        return projectiles
    
    def apply_damage(self, tank: TankState, damage: float, 
                     hit_point: Optional[Vec3] = None) -> bool:
        """
        Apply damage to a tank.
        
        Returns True if tank was destroyed.
        """
        if not tank.is_alive:
            return False
        
        tank.health -= damage
        
        if tank.health <= 0:
            tank.health = 0
            tank.is_alive = False
            return True
        
        return False
    
    def update_cooldowns(self, tanks: List[TankState], dt: float):
        """Update weapon cooldowns for all tanks."""
        for tank in tanks:
            if tank.weapon_cooldown > 0:
                tank.weapon_cooldown -= dt
```

## Phase 3 Verification

Create a combat test that integrates all systems:

### File: `panda_tanks/screens/combat_test_screen.py`
```python
"""Combat test with physics and AI."""

from panda3d.core import Vec3

from .base_screen import BaseScreen
from ..entities.tank import Tank
from ..entities.arena import Arena
from ..entities.projectile import Projectile
from ..systems.physics import PhysicsSystem
from ..systems.ai import AISystem, AI_PRESETS
from ..systems.combat import CombatSystem
from ..types.game_types import TankState, AIBehavior
from ..config import COLORS


class CombatTestScreen(BaseScreen):
    """Test combat with physics and AI."""
    
    def setup(self):
        # Systems
        self.physics = PhysicsSystem(self.game)
        self.ai = AISystem(self.physics.check_line_of_sight)
        self.combat = CombatSystem()
        
        # Arena
        self.arena = Arena(self.root, 40, 40)
        self.arena.add_wall(Vec3(8, 0, 0), Vec3(2, 10, 0))
        self.arena.add_wall(Vec3(-8, 5, 0), Vec3(6, 2, 0))
        
        # Add wall colliders
        for i, wall in enumerate(self.arena.walls):
            self.physics.create_wall_collider(
                f'wall_{i}', self.root, wall.position, wall.size
            )
        
        # Player
        self.player_state = TankState(id='player')
        self.player_state.position = Vec3(0, -10, 0)
        self.player_tank = Tank(self.root, COLORS['cyan'], is_player=True)
        self.player_tank.position = self.player_state.position
        self.physics.create_tank_collider('player', self.player_tank.root)
        
        # Enemy
        self.enemy_state = TankState(id='enemy')
        self.enemy_state.position = Vec3(5, 10, 0)
        self.enemy_ai = AIBehavior(**AI_PRESETS['aggressive'].__dict__)
        self.enemy_tank = Tank(self.root, COLORS['orange'])
        self.enemy_tank.position = self.enemy_state.position
        self.physics.create_tank_collider('enemy', self.enemy_tank.root)
        
        # Projectiles
        self.projectiles = {}  # id -> (Projectile, owner_id)
        
        # Camera
        self.game.camera.setPos(0, -30, 25)
        self.game.camera.lookAt(0, 0, 0)
        
        # Input
        self.accept('escape', lambda: self.game.switch_screen('test'))
    
    def update(self, dt: float):
        inp = self.game.input_manager.state
        
        # Player movement
        if inp.forward:
            self.player_state.position += self.player_tank.get_forward() * 8 * dt
        if inp.backward:
            self.player_state.position -= self.player_tank.get_forward() * 8 * dt
        if inp.left:
            self.player_state.rotation += 2.5 * dt
        if inp.right:
            self.player_state.rotation -= 2.5 * dt
        
        # Player turret aim (simple mouse follow)
        self.player_state.turret_rotation += inp.mouse_delta.x * 2
        
        # Player firing
        if inp.fire_primary:
            self._fire(self.player_state, self.player_tank)
        
        # Update player tank visual
        self.player_tank.position = self.player_state.position
        self.player_tank.rotation = self.player_state.rotation
        self.player_tank.turret_rotation = self.player_state.turret_rotation
        
        # AI update
        if self.enemy_state.is_alive:
            ai_changes = self.ai.update(
                self.enemy_state, self.enemy_ai,
                self.player_state, dt
            )
            
            if 'position' in ai_changes:
                self.enemy_state.position = ai_changes['position']
            if 'rotation' in ai_changes:
                self.enemy_state.rotation = ai_changes['rotation']
            if 'turret_rotation' in ai_changes:
                self.enemy_state.turret_rotation = ai_changes['turret_rotation']
            
            if ai_changes.get('should_fire'):
                self._fire(self.enemy_state, self.enemy_tank)
            
            # Update enemy tank visual
            self.enemy_tank.position = self.enemy_state.position
            self.enemy_tank.rotation = self.enemy_state.rotation
            self.enemy_tank.turret_rotation = self.enemy_state.turret_rotation
        
        # Update cooldowns
        self.combat.update_cooldowns([self.player_state, self.enemy_state], dt)
        
        # Update projectiles
        for proj_id in list(self.projectiles.keys()):
            proj, owner = self.projectiles[proj_id]
            if not proj.update(dt):
                self.physics.remove_collider(proj_id)
                proj.destroy()
                del self.projectiles[proj_id]
        
        # Physics
        self.physics.update()
        
        # Check hits
        for proj_id, hit_id, hit_point in self.physics.get_projectile_hits():
            if proj_id in self.projectiles:
                proj, owner = self.projectiles[proj_id]
                
                # Don't hit owner
                if hit_id == owner:
                    continue
                
                # Apply damage
                if hit_id == 'player':
                    destroyed = self.combat.apply_damage(
                        self.player_state, proj.damage, hit_point
                    )
                    print(f"Player hit! Health: {self.player_state.health}")
                elif hit_id == 'enemy':
                    destroyed = self.combat.apply_damage(
                        self.enemy_state, proj.damage, hit_point
                    )
                    print(f"Enemy hit! Health: {self.enemy_state.health}")
                    if destroyed:
                        self.enemy_tank.root.hide()
                
                # Destroy projectile
                self.physics.remove_collider(proj_id)
                proj.destroy()
                del self.projectiles[proj_id]
    
    def _fire(self, tank_state: TankState, tank_visual: Tank):
        """Fire weapon for a tank."""
        turret_fwd = tank_visual.get_turret_forward()
        proj_data = self.combat.fire_weapon(tank_state, turret_fwd)
        
        for pos, vel, damage in proj_data:
            color = COLORS['cyan'] if tank_state.id == 'player' else COLORS['orange']
            proj = Projectile(self.root, pos, vel, damage, tank_state.id, color)
            self.physics.create_projectile_collider(proj.id, proj.root)
            self.projectiles[proj.id] = (proj, tank_state.id)
```

### Verification Commands
```bash
# Add to main.py:
# from .screens.combat_test_screen import CombatTestScreen
# self._register_screen('combat_test', CombatTestScreen(self))

# Add to test_screen.py:
# self.accept('c', lambda: self.game.switch_screen('combat_test'))

python run.py
# Press 'c' for combat test
```

### Expected Behavior
- Player tank controllable
- Enemy AI chases and attacks player
- Projectiles cause damage
- Health decreases on hits
- Enemy dies when health reaches 0

### Verification Checklist
- [ ] Physics collision prevents walking through walls
- [ ] AI enemy detects player
- [ ] AI enemy chases when in range
- [ ] AI enemy attacks when close
- [ ] Projectiles spawn and move correctly
- [ ] Projectile-tank collision detected
- [ ] Damage applied correctly
- [ ] Enemy destroyed at 0 health
- [ ] No console errors

---

# Phases 4-7: Summary

Due to length constraints, here's the overview for remaining phases:

## Phase 4: Gameplay (Battle Screen)
- Full battle screen with game loop
- HUD with health, energy, weapon status
- Minimap showing enemies
- Victory/defeat conditions
- Pause menu

## Phase 5: Menus
- Main menu with navigation
- Bunker screen (loadout selection)
- Settings screen (graphics, audio, controls)
- Results screen (stats, score)

## Phase 6: Level Editor
- 2D orthographic editor view
- Tool palette (wall, spawn, pickup)
- Save/load JSON levels
- Test play button

## Phase 7: Polish
- Audio system (SFX, music)
- Particle effects (explosions, trails)
- Screen transitions
- Performance optimization

---

# Development Commands Reference

```bash
# Setup
mkdir panda_tanks && cd panda_tanks
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install panda3d numpy

# Run game
python run.py

# Run specific test
python -c "from panda_tanks.screens.test_screen import TestScreen; print('OK')"

# Check imports
python -c "from panda_tanks import main; print('All imports OK')"
```

---

# Claude Code Instructions

When implementing this project:

1. **Work phase by phase** — Complete each phase fully before moving on
2. **Verify each step** — Run the verification commands after each step
3. **Test incrementally** — Use the test screens to validate functionality
4. **Keep files small** — Split files over 200 lines
5. **Handle errors gracefully** — Add try/except for file operations
6. **Document changes** — Update this README as you implement

## Agentic Workflow

```
1. Read current phase requirements
2. Check prerequisites are met
3. Implement step by step
4. Run verification after each step
5. Fix any errors before proceeding
6. Mark phase complete when all checks pass
7. Move to next phase
```

## Safety Considerations

- **Reversible:** All file operations can be undone with git
- **Incremental:** Each phase builds on working code
- **Testable:** Every feature has verification steps
- **Isolated:** Test screens prevent breaking main flow

---

*This development plan is optimized for Claude Code agentic implementation with Opus 4.5. Each phase is self-contained with explicit verification criteria.*
