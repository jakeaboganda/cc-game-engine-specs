# Data Flow Architecture

## Overview

This document describes how data flows through the CC Game Engine, from player input to screen rendering, and how different systems communicate.

## High-Level Data Flow

```
Player Input
    ↓
Input System (raw events → game actions)
    ↓
Game State Machine (action validation)
    ↓
Gameplay Systems (ECS systems process actions)
    ↓
Event Bus (broadcast state changes)
    ↓
Animation/Audio/UI Systems (react to events)
    ↓
Rendering System (visualize current state)
    ↓
Screen Output
```

## Detailed Flow Diagrams

### Turn-Based Combat Loop

```
┌─────────────────────────────────────────────────────────────┐
│ TURN START                                                  │
│ ┌─────────────────────────────────────────────────────┐    │
│ │ 1. State Machine: BeginTurn(entity_id)             │    │
│ │    ↓                                                │    │
│ │ 2. Combat System: Reset action points, apply DoTs   │    │
│ │    ↓                                                │    │
│ │ 3. Event Bus: TurnStarted { entity, turn_number }  │    │
│ │    ↓                                                │    │
│ │ 4. AI System (if AI turn): Evaluate → Select Action│    │
│ │    OR                                               │    │
│ │    Input System (if player turn): Wait for input   │    │
│ └─────────────────────────────────────────────────────┘    │
│                                                             │
│ PLAYER/AI ACTION                                            │
│ ┌─────────────────────────────────────────────────────┐    │
│ │ 5. Action Queue: Push(action)                       │    │
│ │    ↓                                                │    │
│ │ 6. Combat System: Validate action                   │    │
│ │    ↓                                                │    │
│ │ 7. Execute action:                                  │    │
│ │    - Movement: Update position, trigger animation   │    │
│ │    - Attack: Roll to-hit → Roll damage → Apply     │    │
│ │    - Ability: Check range/LOS → Apply effects      │    │
│ │    ↓                                                │    │
│ │ 8. Event Bus: Multiple events                       │    │
│ │    - UnitMoved / AttackHit / AbilityUsed           │    │
│ │    - DamageTaken / StatusApplied / etc.            │    │
│ │    ↓                                                │    │
│ │ 9. Subscribers react:                               │    │
│ │    - Animation System: Queue animations             │    │
│ │    - Audio System: Play SFX                         │    │
│ │    - UI System: Update health bars                  │    │
│ │    - Combat Log: Record event                       │    │
│ └─────────────────────────────────────────────────────┘    │
│                                                             │
│ TURN END                                                    │
│ ┌─────────────────────────────────────────────────────┐    │
│ │ 10. State Machine: EndTurn()                        │    │
│ │     ↓                                               │    │
│ │ 11. Combat System: Check victory conditions         │    │
│ │     ↓                                               │    │
│ │ 12. Turn Manager: Advance to next entity            │    │
│ │     ↓                                               │    │
│ │ 13. Loop back to TURN START                         │    │
│ └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### Input Processing Flow

```
Keyboard/Mouse/Gamepad
    ↓
OS Event (winit::Event)
    ↓
Input Manager: Capture raw input
    ↓
┌────────────────────────────────────┐
│ Per-Player Processing              │
│ ┌────────────────────────────────┐ │
│ │ 1. Device Mapping              │ │
│ │    Player 1 → Keyboard+Mouse   │ │
│ │    Player 2 → Gamepad 0        │ │
│ └────────────────────────────────┘ │
│ ┌────────────────────────────────┐ │
│ │ 2. Context Filtering           │ │
│ │    Current context: InCombat   │ │
│ │    Filter actions to combat    │ │
│ │    actions only                │ │
│ └────────────────────────────────┘ │
│ ┌────────────────────────────────┐ │
│ │ 3. Action Mapping              │ │
│ │    Mouse Click → SelectUnit    │ │
│ │    Space → ConfirmAction       │ │
│ │    Gamepad A → SelectUnit      │ │
│ └────────────────────────────────┘ │
└────────────────────────────────────┘
    ↓
Game Action Queue
    ↓
State Machine: Process actions
    ↓
Gameplay Systems: Execute
```

### Rendering Pipeline Data Flow

```
┌────────────────────────────────────────────────┐
│ Frame Start: Collect Visible Entities         │
│ ┌────────────────────────────────────────────┐ │
│ │ ECS Query: All entities with Position      │ │
│ │            and Sprite components           │ │
│ │     ↓                                      │ │
│ │ Camera System: Frustum culling             │ │
│ │     ↓                                      │ │
│ │ Visible Entities List                      │ │
│ └────────────────────────────────────────────┘ │
└────────────────────────────────────────────────┘
    ↓
┌────────────────────────────────────────────────┐
│ Depth Sorting                                  │
│ ┌────────────────────────────────────────────┐ │
│ │ For each visible entity:                   │ │
│ │   depth_key = y_position * 1000            │ │
│ │             - elevation * 10               │ │
│ │             + depth_bias                   │ │
│ │     ↓                                      │ │
│ │ Sort entities by depth_key                 │ │
│ └────────────────────────────────────────────┘ │
└────────────────────────────────────────────────┘
    ↓
┌────────────────────────────────────────────────┐
│ Batching                                       │
│ ┌────────────────────────────────────────────┐ │
│ │ Group entities by texture:                 │ │
│ │   grass_tileset → [ent1, ent5, ent9]      │ │
│ │   warrior_sprite → [ent2, ent3]           │ │
│ │   effects_atlas → [ent4, ent6, ent7]      │ │
│ │     ↓                                      │ │
│ │ Build instance buffers per batch           │ │
│ └────────────────────────────────────────────┘ │
└────────────────────────────────────────────────┘
    ↓
┌────────────────────────────────────────────────┐
│ GPU Submission                                 │
│ ┌────────────────────────────────────────────┐ │
│ │ For each batch (in depth order):           │ │
│ │   Bind texture                             │ │
│ │   Bind instance buffer                     │ │
│ │   Draw instanced                           │ │
│ │     ↓                                      │ │
│ │ Submit command buffer to GPU               │ │
│ └────────────────────────────────────────────┘ │
└────────────────────────────────────────────────┘
    ↓
Screen Output
```

### Asset Loading Flow

```
Asset Request (from code)
    ↓
Asset Manager: Check cache
    │
    ├─ Cache Hit → Return handle immediately
    │
    └─ Cache Miss
        ↓
    Load Queue: Add request
        ↓
    Background Thread: Load file from disk
        ↓
    Parse/Decode (PNG → Texture, OGG → Audio)
        ↓
    GPU Upload (for textures) or Audio Buffer (for sounds)
        ↓
    Cache: Store loaded asset
        ↓
    Notify: Asset ready
        ↓
    Waiting systems: Use asset
```

### Hot Reload Flow

```
File Change (user edits file)
    ↓
File Watcher: Detect change
    ↓
Asset Manager: Identify affected assets
    ↓
┌─────────────────────────────────────────┐
│ Reload Pipeline                         │
│ ┌─────────────────────────────────────┐ │
│ │ 1. Unload old asset version         │ │
│ │    ↓                                │ │
│ │ 2. Load new version from disk       │ │
│ │    ↓                                │ │
│ │ 3. Parse and validate               │ │
│ │    ↓                                │ │
│ │ 4. Update cache (swap old for new)  │ │
│ │    ↓                                │ │
│ │ 5. Fire AssetReloaded event         │ │
│ └─────────────────────────────────────┘ │
└─────────────────────────────────────────┘
    ↓
Event Subscribers:
    - Rendering: Re-upload texture to GPU
    - Content System: Re-parse unit definitions
    - ECS: Update existing entities with new data
    ↓
Visual update (no restart needed)
```

### Save/Load Flow

#### Save Flow

```
User clicks "Save"
    ↓
Save Manager: Collect game state
    ↓
┌──────────────────────────────────────────┐
│ Serialize Components                     │
│ ┌──────────────────────────────────────┐ │
│ │ ECS: Query all saveable entities     │ │
│ │   - Position, Stats, Inventory       │ │
│ │   - Status Effects, AI State         │ │
│ │     ↓                                │ │
│ │ Serialize to SaveFile struct         │ │
│ │   - Metadata (timestamp, version)    │ │
│ │   - Game state (entities, turn data) │ │
│ │   - Session state (campaign progress)│ │
│ └──────────────────────────────────────┘ │
└──────────────────────────────────────────┘
    ↓
Serialization (RON/TOML/Binary)
    ↓
Write to disk
    - Temp file first (.tmp)
    - Atomic rename to .save
    ↓
Backup previous save (.backup)
    ↓
Save complete
```

#### Load Flow

```
User clicks "Load" → selects save file
    ↓
Save Manager: Read file
    ↓
Deserialize SaveFile
    ↓
Version Check
    │
    ├─ Same version → proceed
    │
    └─ Old version → run migrations
        ↓
┌──────────────────────────────────────────┐
│ Restore Game State                       │
│ ┌──────────────────────────────────────┐ │
│ │ 1. Clear current ECS world           │ │
│ │    ↓                                 │ │
│ │ 2. Spawn entities from save data     │ │
│ │    ↓                                 │ │
│ │ 3. Restore components                │ │
│ │    ↓                                 │ │
│ │ 4. Restore turn manager state        │ │
│ │    ↓                                 │ │
│ │ 5. Restore action queue              │ │
│ │    ↓                                 │ │
│ │ 6. Transition to InGame state        │ │
│ └──────────────────────────────────────┘ │
└──────────────────────────────────────────┘
    ↓
Game resumes from saved point
```

## Data Ownership & Lifetimes

### Asset Handles

```rust
// Strong handle (keeps asset alive)
struct AssetHandle<T> {
    id: AssetId,
    inner: Arc<T>,  // Reference counted
}

// When last handle is dropped, asset can be unloaded
impl<T> Drop for AssetHandle<T> {
    fn drop(&mut self) {
        if Arc::strong_count(&self.inner) == 1 {
            asset_manager.mark_for_unload(&self.id);
        }
    }
}
```

**Lifetime rules**:
- Assets live as long as at least one handle exists
- Weak handles don't prevent unloading
- Critical assets (UI, fallback textures) held permanently

### Entity Ownership

```rust
// ECS owns all entities
// Components are stored in type-specific storages
// Queries borrow components immutably or mutably

// Example: Movement system
fn movement_system(
    mut positions: Query<&mut Position>,  // Mutable borrow
    targets: Query<&MovementTarget>,       // Immutable borrow
) {
    for (mut pos, target) in positions.iter_mut().zip(targets.iter()) {
        pos.0 = lerp(pos.0, target.0, 0.1);
    }
}
```

**Ownership rules**:
- ECS owns entities
- Systems borrow components (no ownership transfer)
- Resources owned by World, borrowed by systems

### Event Lifetime

```rust
// Events are cloned and distributed to subscribers
pub fn publish<E: Event + Clone>(&mut self, event: E) {
    for handler in self.get_handlers::<E>() {
        handler.handle(event.clone());  // Each handler gets a copy
    }
}

// Or move semantics for expensive events
pub fn publish_move<E: Event>(&mut self, event: E) {
    let handlers = self.get_handlers::<E>();
    if handlers.len() == 1 {
        handlers[0].handle(event);  // Move to single handler
    } else {
        // Clone for multiple handlers
        for handler in handlers {
            handler.handle(event.clone());
        }
    }
}
```

## Memory Layout

### Sprite Batching

```
Texture Atlas:
┌─────────────────────────────────────┐
│ [sprite1] [sprite2] [sprite3]       │
│ [sprite4] [sprite5] [sprite6]       │
│ [sprite7] [sprite8] ...             │
└─────────────────────────────────────┘

Instance Buffer (GPU):
┌──────────────────────────────────────┐
│ Instance 0: { pos, uv, color, ... }  │
│ Instance 1: { pos, uv, color, ... }  │
│ Instance 2: { pos, uv, color, ... }  │
│ ...                                  │
└──────────────────────────────────────┘

Single Draw Call:
  DrawInstanced(num_instances, texture_atlas)
```

### ECS Component Layout (Bevy-style)

```
Component Storage (Array of Structs):

Position Storage:
[HexPos(1,2,0), HexPos(3,4,1), HexPos(0,0,0), ...]
  Entity 0      Entity 1        Entity 2

UnitStats Storage:
[Stats{hp:30..}, Stats{hp:25..}, Stats{hp:40..}, ...]
  Entity 0       Entity 1         Entity 2

Benefits:
- Cache-friendly iteration
- Efficient querying
- Parallel processing
```

## Data Validation

### Input Validation

```rust
fn validate_action(action: &GameAction, state: &GameState) -> Result<()> {
    match action {
        GameAction::Move { unit, path } => {
            // 1. Check unit exists
            let entity = state.get_entity(*unit)
                .ok_or(GameError::InvalidUnit)?;
            
            // 2. Check it's this unit's turn
            if state.turn_manager.current_unit() != *unit {
                return Err(GameError::NotYourTurn);
            }
            
            // 3. Check path is valid
            for step in path {
                if !state.hex_grid.is_walkable(*step, entity.movement_type) {
                    return Err(GameError::InvalidPath);
                }
            }
            
            // 4. Check movement points
            let cost = calculate_path_cost(path);
            if entity.movement_points < cost {
                return Err(GameError::InsufficientMovement);
            }
            
            Ok(())
        },
        // ... other actions
    }
}
```

### Content Validation

```rust
fn validate_unit_definition(def: &UnitDefinition) -> Result<()> {
    // 1. Required fields
    if def.name.is_empty() {
        return Err(ContentError::MissingField("name"));
    }
    
    // 2. Numeric ranges
    if def.max_hp <= 0 {
        return Err(ContentError::InvalidValue("max_hp must be > 0"));
    }
    
    // 3. Asset references exist
    if !asset_manager.exists(&def.sprite) {
        return Err(ContentError::MissingAsset(def.sprite.clone()));
    }
    
    // 4. Ability references valid
    for ability_id in &def.abilities {
        if !content_registry.has_ability(ability_id) {
            return Err(ContentError::UnknownAbility(ability_id.clone()));
        }
    }
    
    Ok(())
}
```

## Communication Patterns

### Request-Response (Synchronous)

```rust
// Pathfinding: Request path, get immediate response
let path = hex_grid.find_path(start, goal, movement_type)?;
```

### Event-Driven (Asynchronous)

```rust
// Combat: Fire event, multiple systems react
event_bus.publish(DamageTaken {
    entity: target,
    amount: 15,
    source: attacker,
});

// Subscribers handle independently:
// - Animation system queues hurt animation
// - Audio system plays hurt sound
// - UI system updates health bar
// - Combat log adds entry
```

### Query (Read-Only)

```rust
// UI queries game state without modifying
fn render_ui(query: Query<(&Position, &UnitStats)>) {
    for (pos, stats) in query.iter() {
        draw_health_bar(pos, stats.hp, stats.max_hp);
    }
}
```

### Command (Deferred Execution)

```rust
// Action queue: Commands stored, executed later
action_queue.push(GameAction::Attack {
    attacker: player_unit,
    target: enemy_unit,
});

// Later, during update:
while let Some(action) = action_queue.pop() {
    execute_action(action);
}
```

## State Synchronization

### Deterministic Execution

For save/load and replay:

```rust
// Use fixed timestep
const FIXED_TIMESTEP: f32 = 1.0 / 60.0;

fn update_loop(dt: f32) {
    accumulator += dt;
    
    while accumulator >= FIXED_TIMESTEP {
        // Deterministic update
        update_systems(FIXED_TIMESTEP);
        accumulator -= FIXED_TIMESTEP;
    }
    
    // Rendering (variable timestep, doesn't affect game state)
    let alpha = accumulator / FIXED_TIMESTEP;
    render_interpolated(alpha);
}
```

**Determinism guarantees**:
- All random numbers use seeded RNG (stored in save file)
- Fixed timestep for physics/movement
- No reliance on real-time clocks
- No floating-point non-determinism (use integer math where possible)

### State Snapshots

```rust
struct StateSnapshot {
    turn_number: u32,
    entity_states: Vec<EntitySnapshot>,
    random_seed: u64,
}

// Take snapshot before each action
fn execute_action(action: GameAction) {
    let snapshot = create_snapshot();
    undo_stack.push(snapshot);
    
    perform_action(action);
}

// Undo: Restore from snapshot
fn undo() -> Result<()> {
    let snapshot = undo_stack.pop()
        .ok_or(GameError::NothingToUndo)?;
    
    restore_snapshot(snapshot);
    Ok(())
}
```

## Data Flow Performance

### Bottleneck Identification

**Hot paths** (executed every frame):
- Input processing
- Animation updates
- Rendering (sprite batching, depth sorting)
- Camera transform calculations

**Warm paths** (executed every turn):
- AI evaluation
- Pathfinding
- Combat resolution
- Event dispatching

**Cold paths** (infrequent):
- Asset loading
- Save/load
- Content parsing
- Map generation

### Optimization Strategies

**Cache locality**:
```rust
// Bad: Pointer chasing
for entity in entities {
    let pos = entity.position;  // Cache miss
    let sprite = entity.sprite;  // Cache miss
    render(pos, sprite);
}

// Good: Component iteration (SoA)
for (pos, sprite) in positions.iter().zip(sprites.iter()) {
    render(pos, sprite);  // Sequential access
}
```

**Batching**:
```rust
// Bad: Individual draw calls
for sprite in sprites {
    draw_sprite(sprite);  // 1000 draw calls
}

// Good: Batched
let batches = group_by_texture(sprites);
for (texture, batch) in batches {
    draw_instanced(texture, batch);  // 10 draw calls
}
```

**Lazy evaluation**:
```rust
// Don't compute until needed
struct LazyPath {
    start: HexPosition,
    goal: HexPosition,
    cached: Option<Vec<HexPosition>>,
}

impl LazyPath {
    fn get(&mut self, grid: &HexGrid) -> &[HexPosition] {
        if self.cached.is_none() {
            self.cached = Some(grid.find_path(self.start, self.goal));
        }
        self.cached.as_ref().unwrap()
    }
}
```

## Debugging Data Flow

### Tracing Events

```rust
use tracing::{info, debug, trace};

fn execute_attack(attacker: EntityId, target: EntityId) {
    info!(?attacker, ?target, "Executing attack");
    
    let to_hit_roll = roll_d20();
    debug!(?to_hit_roll, "To-hit roll");
    
    if to_hit_roll >= target_ac {
        let damage = roll_damage();
        info!(?damage, "Hit! Damage dealt");
        apply_damage(target, damage);
    } else {
        info!("Miss!");
    }
}

// Output:
// INFO execute_attack: Executing attack attacker=Entity(5) target=Entity(12)
// DEBUG execute_attack: To-hit roll to_hit_roll=17
// INFO execute_attack: Hit! Damage dealt damage=8
```

### State Inspection

```rust
// Debug UI showing current state
fn debug_ui(ui: &mut egui::Ui, state: &GameState) {
    ui.heading("Game State");
    ui.label(format!("Turn: {}", state.turn_manager.round_number));
    ui.label(format!("Active entity: {:?}", state.turn_manager.current_unit()));
    
    ui.separator();
    ui.heading("Event Queue");
    for event in &state.event_bus.queue {
        ui.label(format!("{:?}", event));
    }
}
```

### Flow Visualization

```rust
// Log data flow through systems
#[instrument]
fn input_system() { /* ... */ }

#[instrument]
fn ai_system() { /* ... */ }

#[instrument]
fn combat_system() { /* ... */ }

// Generates hierarchical trace:
// input_system [0.2ms]
// ai_system [1.5ms]
//   ├─ evaluate_behavior_tree [0.8ms]
//   └─ select_action [0.7ms]
// combat_system [0.5ms]
```
