# Input System

## Overview

Unified input abstraction supporting keyboard, mouse, and multiple game controllers for local co-op gameplay.

## Design Principles

1. **Input abstraction**: Map raw inputs to game actions
2. **Multi-player support**: 2-8 simultaneous players
3. **Rebindable controls**: Players configure their own mappings
4. **Turn-based friendly**: Input queueing and confirmation
5. **Platform agnostic**: Works on Windows, Linux, macOS

## Input Architecture

### Input Layers

```
Raw Input (OS events) 
    ↓
Input Manager (device detection, polling)
    ↓
Action Mapping (raw → game actions)
    ↓
Player Input (per-player action queue)
    ↓
Game Logic (consume actions)
```

### Input Manager

```rust
struct InputManager {
    keyboard: KeyboardState,
    mouse: MouseState,
    controllers: Vec<GamepadState>,
    action_maps: HashMap<PlayerId, ActionMap>,
}

struct KeyboardState {
    keys_down: HashSet<KeyCode>,
    keys_pressed: HashSet<KeyCode>,   // this frame only
    keys_released: HashSet<KeyCode>,  // this frame only
}

struct MouseState {
    position: Vec2,
    delta: Vec2,
    buttons_down: HashSet<MouseButton>,
    wheel_delta: f32,
}

struct GamepadState {
    device_id: u32,
    buttons_down: HashSet<GamepadButton>,
    axes: HashMap<GamepadAxis, f32>,
    rumble: f32,
}
```

## Action Mapping

### Game Actions

Define high-level actions instead of raw keys:

```rust
enum GameAction {
    // Camera
    CameraMoveUp,
    CameraMoveDown,
    CameraMoveLeft,
    CameraMoveRight,
    CameraZoomIn,
    CameraZoomOut,
    
    // Unit control
    SelectUnit,
    ConfirmSelection,
    CancelAction,
    MoveUnit,
    Attack,
    UseAbility(u8),  // 0-9 ability hotkeys
    
    // Turn management
    EndTurn,
    
    // UI
    OpenMenu,
    ToggleInventory,
    NextUnit,
    PreviousUnit,
}
```

### Action Map

```rust
struct ActionMap {
    bindings: HashMap<InputSource, GameAction>,
}

enum InputSource {
    Key(KeyCode),
    MouseButton(MouseButton),
    GamepadButton(GamepadButton),
    GamepadAxis(GamepadAxis, AxisDirection),
}
```

**Example bindings**:
```toml
# config/input/player1_keyboard.toml
[actions]
camera_move_up = "W"
camera_move_down = "S"
camera_move_left = "A"
camera_move_right = "D"
select_unit = "LeftClick"
confirm_selection = "Space"
cancel_action = "Escape"
attack = "Q"
end_turn = "Enter"

[abilities]
ability_0 = "1"
ability_1 = "2"
ability_2 = "3"
```

```toml
# config/input/player1_gamepad.toml
[actions]
camera_move = "LeftStick"
select_unit = "A"
confirm_selection = "A"
cancel_action = "B"
attack = "X"
use_ability = "Y"
end_turn = "Start"

[ui]
next_unit = "RightBumper"
previous_unit = "LeftBumper"
```

## Multi-Player Input

### Player Assignment

```rust
struct PlayerInput {
    player_id: PlayerId,
    device: InputDevice,
    action_map: ActionMap,
    cursor_position: Vec2,  // each player has own cursor
}

enum InputDevice {
    KeyboardMouse,
    Gamepad(u32),
}
```

**Device assignment flow**:
1. At game start, show "Press any button to join"
2. Detect input from new device → assign to next player slot
3. Each player gets unique cursor color/icon
4. If keyboard+mouse, only one player can use it

### Cursor Management

For simultaneous input (non-turn-based exploration):
- Each gamepad controls its own cursor
- Keyboard+mouse shares one cursor
- Cursors snap to hex centers
- Visual indicator per player (colored circle with player number)

### Input Queueing

Turn-based games don't need instant response:

```rust
struct InputQueue {
    pending_actions: VecDeque<(PlayerId, GameAction)>,
}
```

**Benefits**:
- Handle inputs arriving while animating
- Allow "undo" by clearing queue
- Confirmation prompts without blocking
- AI can insert actions into same queue

## Input Contexts

Different game states need different input handling:

```rust
enum InputContext {
    MainMenu,
    InCombat,
    UnitMovement,
    AbilityTargeting,
    Inventory,
    Dialogue,
}

struct ContextualInput {
    active_context: InputContext,
    context_stack: Vec<InputContext>,  // for nested contexts
}
```

**Example**: When targeting an ability:
- Push `AbilityTargeting` context
- Only movement and confirm/cancel are valid
- ESC pops context back to `InCombat`
- Prevents accidental menu opens

## Hex Selection (Isometric 3D)

Mouse/cursor to hex conversion with elevation handling:

```rust
struct IsometricHexSelector {
    current_position: Option<HexPosition>,  // 3D position (hex + elevation)
    hover_position: Option<HexPosition>,
    selected_unit: Option<EntityId>,
    elevation_mode: ElevationSelectionMode,
}

enum ElevationSelectionMode {
    Auto,        // Automatically pick most likely elevation
    Fixed(i32),  // Lock to specific elevation level
    Cycle,       // Cycle through elevations with mouse wheel
}

fn update_hex_selection(
    input: &InputManager,
    camera: &IsometricCamera,
    map: &HexMap,
    mode: &ElevationSelectionMode,
) -> Option<HexPosition> {
    let screen_pos = input.mouse.position;
    let world_pos = camera.screen_to_world(screen_pos);
    
    // Convert screen position to hex (2D)
    let hex = iso_pixel_to_hex(world_pos, &camera.projection);
    
    // Determine elevation
    let elevation = match mode {
        ElevationSelectionMode::Auto => {
            // Pick elevation based on context
            select_best_elevation(hex, map, input)
        },
        ElevationSelectionMode::Fixed(elev) => *elev,
        ElevationSelectionMode::Cycle => {
            // User scrolls mouse wheel to change elevation
            handle_elevation_cycling(hex, input)
        },
    };
    
    Some(HexPosition { hex, elevation })
}

fn select_best_elevation(hex: HexCoord, map: &HexMap, input: &InputManager) -> i32 {
    // Heuristics for picking elevation automatically:
    
    // 1. If a unit is at this hex, use unit's elevation
    if let Some(unit) = map.get_unit_at_hex(hex) {
        return unit.position.elevation;
    }
    
    // 2. If tile exists at this hex, use tile elevation
    if let Some(tile) = map.get_tile(hex) {
        return tile.base_elevation;
    }
    
    // 3. If selecting for movement, use current unit's elevation
    if let Some(selected_unit) = get_selected_unit() {
        return selected_unit.position.elevation;
    }
    
    // 4. Default to ground level
    0
}

fn handle_elevation_cycling(hex: HexCoord, input: &InputManager) -> i32 {
    static mut CURRENT_ELEVATION: i32 = 0;
    
    unsafe {
        // Mouse wheel up = increase elevation
        if input.mouse.wheel_delta > 0.0 {
            CURRENT_ELEVATION = (CURRENT_ELEVATION + 1).min(map.max_elevation);
        }
        // Mouse wheel down = decrease elevation
        else if input.mouse.wheel_delta < 0.0 {
            CURRENT_ELEVATION = (CURRENT_ELEVATION - 1).max(map.min_elevation);
        }
        
        CURRENT_ELEVATION
    }
}
```

**Visual feedback** (isometric):
- **Hover highlight**: Subtle glow on hex at correct elevation
- **Selected highlight**: Bright outline with elevation indicator
- **Invalid target**: Red X or dimmed (e.g., can't reach that elevation)
- **Elevation layers**: Semi-transparent overlay showing different elevation levels
- **Height markers**: Visual pillar/number showing selected elevation

**Mouse picking challenges in isometric**:
- Overlapping hexes (higher elevation hexes visually overlap lower ones)
- Depth ambiguity (which hex did user click?)

**Solutions**:
1. **Raycasting**: Cast ray from mouse through all hex layers, pick closest to camera
2. **Depth buffer**: Use GPU depth buffer to determine which sprite was clicked
3. **Priority system**: Prioritize units > terrain features > ground tiles
4. **Hover feedback**: Show clearly which hex is under cursor before clicking

**Gamepad hex selection**:
```rust
struct GamepadHexCursor {
    position: HexPosition,
    move_speed: f32,
    snap_to_units: bool,  // Auto-snap to nearby units
}

fn update_gamepad_cursor(cursor: &mut GamepadHexCursor, input: &GamepadState) {
    // Left stick moves cursor in isometric space
    let stick_x = input.axes.get(&GamepadAxis::LeftX).unwrap_or(&0.0);
    let stick_y = input.axes.get(&GamepadAxis::LeftY).unwrap_or(&0.0);
    
    // Convert stick input to hex direction
    if stick_x.abs() > DEADZONE || stick_y.abs() > DEADZONE {
        let direction = Direction::from_angle(stick_y.atan2(*stick_x));
        let neighbor = cursor.position.hex.neighbor(direction);
        
        cursor.position.hex = neighbor;
        
        // Auto-adjust elevation to terrain
        if let Some(tile) = map.get_tile(neighbor) {
            cursor.position.elevation = tile.base_elevation;
        }
    }
    
    // Shoulder buttons change elevation
    if input.buttons_down.contains(&GamepadButton::LeftBumper) {
        cursor.position.elevation -= 1;
    }
    if input.buttons_down.contains(&GamepadButton::RightBumper) {
        cursor.position.elevation += 1;
    }
}
```

## Controller Rumble

Haptic feedback for game events:

```rust
fn trigger_rumble(
    gamepad: &mut GamepadState,
    intensity: f32,
    duration_ms: u32,
) {
    gamepad.rumble = intensity;
    // Stop after duration
}
```

**Use cases**:
- Light rumble on hit
- Heavy rumble on critical hit or damage taken
- Pulsing rumble while low health
- Unique patterns for different abilities

## Input Buffering

For fighting-game-style combo inputs (optional):

```rust
struct InputBuffer {
    buffer: VecDeque<(GameAction, Instant)>,
    buffer_window_ms: u64,  // how long to keep inputs
}
```

**Example**: "Quick ability" by pressing ability key twice:
- Buffer stores last 500ms of inputs
- Check for patterns: [UseAbility(1), UseAbility(1)]
- Trigger enhanced version if pattern matches

## Accessibility Features

- **Button hold vs press**: Options for both
- **Deadzone adjustment**: For worn-out controllers
- **Auto-fire**: Hold button = repeated presses
- **One-handed mode**: All functions on one side of keyboard
- **Sticky keys**: Modifiers don't need to be held
- **Input assist**: Snap cursor to valid targets

## Platform-Specific

### Keyboard

- **Text input**: For naming characters, chat
- **Modifier keys**: Shift/Ctrl/Alt for alternate actions
- **Numpad**: Optional alternative controls

### Mouse

- **Drag selection**: Box-select multiple units (RTS-style)
- **Edge scrolling**: Move camera when cursor near edge
- **Context menus**: Right-click for actions
- **Scroll wheel**: Zoom or cycle units

### Gamepad

- **Stick sensitivity curves**: Linear, quadratic, cubic
- **Trigger thresholds**: When does a trigger count as "pressed"
- **Vibration intensity**: Global rumble strength setting

## Configuration

Persistent input settings:

```toml
# ~/.cc-game-engine/input_config.toml
[player1]
device = "keyboard_mouse"
sensitivity = 1.0
invert_y = false
action_map = "default_keyboard"

[player2]
device = "gamepad_0"
deadzone = 0.15
vibration = 0.8
action_map = "default_gamepad"
```

**In-game rebinding**:
- Show current binding
- "Press new key/button"
- Detect conflicts (same key for two actions)
- Reset to defaults option

## Debug & Testing

- **Input display**: Show all active inputs on-screen
- **Virtual controller**: Simulate gamepad with keyboard
- **Input recording**: Record and replay input sequences
- **Latency testing**: Measure input-to-screen time

## Performance

- **Polling rate**: 60Hz minimum, 120Hz+ ideal
- **Latency budget**: <16ms from input to action start
- **Prediction**: Start move animation before server confirms (for networked mode)
