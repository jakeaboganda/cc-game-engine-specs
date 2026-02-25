# Component Architecture

## System Components

Detailed breakdown of major engine components and their responsibilities.

## Layer 1: Engine Core

### Rendering System

**Responsibilities**:
- Isometric projection and camera transforms
- Sprite rendering with depth sorting
- Shader management and GPU communication
- Frame timing and vsync
- Multi-viewport support (split-screen)

**Key Types**:
```rust
struct RenderEngine {
    device: wgpu::Device,
    queue: wgpu::Queue,
    surface: wgpu::Surface,
    pipeline: RenderPipeline,
    camera_manager: CameraManager,
    sprite_batcher: SpriteBatcher,
}

struct RenderPipeline {
    terrain_pass: TerrainRenderPass,
    sprite_pass: SpriteRenderPass,
    particle_pass: ParticleRenderPass,
    ui_pass: UIRenderPass,
    post_process: PostProcessPipeline,
}

struct SpriteBatcher {
    batches: HashMap<TextureHandle, Vec<SpriteInstance>>,
    depth_sorted_queue: Vec<DepthSortedSprite>,
    instance_buffer: wgpu::Buffer,
}
```

**Interfaces**:
```rust
trait Renderer {
    fn render_frame(&mut self, scene: &Scene) -> Result<()>;
    fn resize(&mut self, width: u32, height: u32);
    fn set_camera(&mut self, camera: Camera);
    fn submit_sprite(&mut self, sprite: Sprite, position: Vec3, depth: f32);
}
```

**Dependencies**:
- ↓ Asset Manager (textures, shaders)
- ↑ Game State (what to render)
- ↔ Camera System

---

### Input System

**Responsibilities**:
- Raw input polling (keyboard, mouse, gamepad)
- Action mapping (raw input → game actions)
- Multi-player input routing
- Input buffering and replay

**Key Types**:
```rust
struct InputManager {
    devices: Vec<InputDevice>,
    action_maps: HashMap<PlayerId, ActionMap>,
    event_queue: VecDeque<InputEvent>,
    replay_recorder: Option<ReplayRecorder>,
}

enum InputDevice {
    KeyboardMouse(KeyboardMouseState),
    Gamepad(GamepadState),
}

struct ActionMap {
    bindings: HashMap<RawInput, GameAction>,
    contexts: Vec<InputContext>,
}
```

**Interfaces**:
```rust
trait InputProvider {
    fn poll(&mut self) -> Vec<RawInputEvent>;
    fn set_action_map(&mut self, player: PlayerId, map: ActionMap);
    fn get_action(&self, player: PlayerId, action: GameAction) -> ActionState;
}
```

**Dependencies**:
- ↓ OS/Platform layer (gilrs, winit)
- ↑ Game State (consumes actions)
- ↔ UI System (input context switching)

---

### Audio Engine

**Responsibilities**:
- Audio playback (music, SFX, voice)
- Spatial audio positioning
- Audio mixing and ducking
- Streaming and buffering

**Key Types**:
```rust
struct AudioEngine {
    output_stream: cpal::Stream,
    music_player: MusicPlayer,
    sfx_player: SfxPlayer,
    mixer: AudioMixer,
    spatial_audio: SpatialAudioProcessor,
}

struct AudioMixer {
    buses: HashMap<BusId, AudioBus>,
    master_volume: f32,
    active_sounds: Vec<ActiveSound>,
}
```

**Interfaces**:
```rust
trait AudioSystem {
    fn play_sound(&mut self, sound: SoundId, params: PlayParams) -> SoundHandle;
    fn play_music(&mut self, track: MusicId, fade_in: f32);
    fn set_listener_position(&mut self, position: Vec3);
    fn update(&mut self, dt: f32);
}
```

**Dependencies**:
- ↓ Asset Manager (audio files)
- ↑ Game State (trigger sounds)
- ↔ Event System (audio event triggers)

---

### Asset Manager

**Responsibilities**:
- Asset loading and caching
- Hot-reload file watching
- Reference counting and lifetime management
- Asset packing and compression

**Key Types**:
```rust
struct AssetManager {
    textures: AssetCache<TextureAsset>,
    audio: AssetCache<AudioAsset>,
    data: AssetCache<DataAsset>,
    file_watcher: Option<FileWatcher>,
    load_queue: VecDeque<AssetLoadRequest>,
}

struct AssetCache<T> {
    assets: HashMap<AssetId, Arc<T>>,
    metadata: HashMap<AssetId, AssetMetadata>,
    lru_tracker: LruCache,
}
```

**Interfaces**:
```rust
trait AssetLoader {
    fn load<T: Asset>(&mut self, id: &str) -> AssetHandle<T>;
    fn load_async<T: Asset>(&mut self, id: &str) -> AssetFuture<T>;
    fn reload(&mut self, id: &str);
    fn unload(&mut self, id: &str);
}
```

**Dependencies**:
- ↓ File System / Archive
- ↑ All systems (consumers)
- ↔ Hot Reload System

---

### Event Bus

**Responsibilities**:
- Event publication and subscription
- Event queueing and dispatching
- Event filtering and prioritization
- Event recording (for replay/debug)

**Key Types**:
```rust
struct EventBus {
    subscribers: HashMap<TypeId, Vec<SubscriberHandle>>,
    event_queue: VecDeque<Box<dyn Event>>,
    immediate_mode: bool,
    recorder: Option<EventRecorder>,
}

trait Event: Any + Send + Sync {
    fn type_id(&self) -> TypeId;
}

trait EventHandler: Send + Sync {
    fn handle(&mut self, event: &dyn Event);
    fn priority(&self) -> i32 { 0 }
}
```

**Interfaces**:
```rust
trait EventSystem {
    fn subscribe<E: Event>(&mut self, handler: Box<dyn EventHandler>);
    fn publish<E: Event>(&mut self, event: E);
    fn process_events(&mut self);
}
```

**Dependencies**:
- ↔ All systems (publish/subscribe)

---

## Layer 2: Tactics Framework

### Hex Grid System

**Responsibilities**:
- Hex coordinate math (cube, axial, offset)
- Pathfinding (A* with elevation)
- Line of sight calculations
- Range queries and neighbor lookups
- Spatial hashing for performance

**Key Types**:
```rust
struct HexGrid {
    tiles: HashMap<HexCoord, HexTile>,
    spatial_hash: SpatialHashMap3D,
    bounds: GridBounds,
    pathfinding_cache: PathfindingCache,
}

struct PathfindingCache {
    recent_paths: LruCache<(HexPosition, HexPosition), Vec<HexPosition>>,
    flow_fields: HashMap<HexPosition, FlowField>,
}

struct SpatialHashMap3D {
    cells: HashMap<(i32, i32, i32), Vec<EntityId>>,
    cell_size: i32,
}
```

**Interfaces**:
```rust
trait HexGridSystem {
    fn find_path(&self, from: HexPosition, to: HexPosition, movement_type: MovementType) -> Option<Vec<HexPosition>>;
    fn has_line_of_sight(&self, from: HexPosition, to: HexPosition) -> bool;
    fn get_hexes_in_range(&self, center: HexPosition, range: i32) -> Vec<HexPosition>;
    fn get_tile(&self, coord: HexCoord) -> Option<&HexTile>;
}
```

**Dependencies**:
- ↑ Combat System (movement, LOS)
- ↑ AI System (pathfinding)
- ↔ Map Data (tile storage)

---

### Combat System

**Responsibilities**:
- Turn order and initiative
- Attack resolution (to-hit, damage)
- Ability execution and targeting
- Status effect management
- Combat event generation

**Key Types**:
```rust
struct CombatSystem {
    turn_manager: TurnManager,
    action_queue: ActionQueue,
    status_effects: StatusEffectManager,
    damage_calculator: DamageCalculator,
}

struct TurnManager {
    turn_order: Vec<EntityId>,
    current_turn_index: usize,
    round_number: u32,
    initiative_rolls: HashMap<EntityId, i32>,
}

struct ActionQueue {
    pending: VecDeque<GameAction>,
    executing: Option<ExecutingAction>,
    undo_stack: UndoStack,
}
```

**Interfaces**:
```rust
trait CombatResolver {
    fn resolve_attack(&mut self, attacker: EntityId, target: EntityId) -> AttackResult;
    fn apply_ability(&mut self, caster: EntityId, ability: AbilityId, targets: Vec<EntityId>) -> AbilityResult;
    fn apply_damage(&mut self, entity: EntityId, damage: i32, damage_type: DamageType);
    fn end_turn(&mut self) -> EntityId; // Returns next entity
}
```

**Dependencies**:
- ↓ Hex Grid (range, LOS checks)
- ↓ ECS (entity stats, components)
- ↑ Event System (combat events)
- ↑ Animation System (visual feedback)

---

### AI System

**Responsibilities**:
- Behavior tree evaluation
- Utility-based decision making
- Tactical position scoring
- Action selection and execution
- AI personality management

**Key Types**:
```rust
struct AISystem {
    controllers: HashMap<EntityId, AIController>,
    behavior_trees: HashMap<String, BehaviorTree>,
    personalities: HashMap<String, AIPersonality>,
}

struct AIController {
    entity: EntityId,
    behavior_tree: String,
    personality: String,
    blackboard: AIBlackboard,
    current_plan: Option<AIPlan>,
}

struct AIBlackboard {
    variables: HashMap<String, BlackboardValue>,
    perceived_threats: Vec<EntityId>,
    target: Option<EntityId>,
    last_known_positions: HashMap<EntityId, HexPosition>,
}
```

**Interfaces**:
```rust
trait AIExecutor {
    fn evaluate(&mut self, entity: EntityId, game_state: &GameState) -> Option<GameAction>;
    fn update_perception(&mut self, entity: EntityId, game_state: &GameState);
    fn assign_behavior(&mut self, entity: EntityId, behavior: String);
}
```

**Dependencies**:
- ↓ Hex Grid (pathfinding, tactical evaluation)
- ↓ Combat System (action execution)
- ↓ ECS (entity queries)
- ↑ Game State (decision making)

---

### State Machine

**Responsibilities**:
- Game state transitions
- Turn management
- Victory/defeat detection
- Pause/resume handling

**Key Types**:
```rust
struct GameStateMachine {
    current: GameState,
    previous: Option<GameState>,
    pending_transition: Option<GameState>,
    substates: HashMap<GameState, Box<dyn StateHandler>>,
}

trait StateHandler {
    fn on_enter(&mut self, context: &mut GameContext);
    fn on_exit(&mut self, context: &mut GameContext);
    fn update(&mut self, context: &mut GameContext, dt: f32) -> Option<GameState>;
}
```

**Interfaces**:
```rust
trait StateMachine {
    fn transition(&mut self, new_state: GameState);
    fn update(&mut self, dt: f32);
    fn current_state(&self) -> GameState;
}
```

**Dependencies**:
- ↑ All gameplay systems
- ↔ Event System (state change events)
- ↔ Save/Load (state persistence)

---

## Layer 3: Content Layer

### Data Loader

**Responsibilities**:
- TOML/RON/JSON parsing
- Data validation and error reporting
- Content hot-reloading
- Mod loading and priority

**Key Types**:
```rust
struct ContentLoader {
    content_registry: ContentRegistry,
    validators: Vec<Box<dyn ContentValidator>>,
    mod_loader: ModLoader,
}

struct ContentRegistry {
    units: HashMap<UnitTypeId, UnitDefinition>,
    abilities: HashMap<AbilityId, AbilityDefinition>,
    maps: HashMap<MapId, MapDefinition>,
    scenarios: HashMap<ScenarioId, ScenarioDefinition>,
}
```

**Interfaces**:
```rust
trait ContentProvider {
    fn load_content(&mut self, path: &Path) -> Result<()>;
    fn get_unit(&self, id: &str) -> Option<&UnitDefinition>;
    fn get_ability(&self, id: &str) -> Option<&AbilityDefinition>;
    fn reload_content(&mut self);
}
```

**Dependencies**:
- ↓ Asset Manager (file loading)
- ↑ Gameplay Systems (content consumers)
- ↔ Mod System

---

### Entity Factory

**Responsibilities**:
- Entity instantiation from templates
- Component setup and configuration
- Pooling and recycling
- Spawn effects and animations

**Key Types**:
```rust
struct EntityFactory {
    templates: HashMap<String, EntityTemplate>,
    entity_pool: EntityPool,
    spawn_effects: HashMap<String, SpawnEffect>,
}

struct EntityTemplate {
    components: Vec<ComponentTemplate>,
    initial_state: EntityState,
}
```

**Interfaces**:
```rust
trait EntitySpawner {
    fn spawn_unit(&mut self, template: &str, position: HexPosition) -> EntityId;
    fn spawn_from_definition(&mut self, def: &UnitDefinition, position: HexPosition) -> EntityId;
    fn despawn(&mut self, entity: EntityId);
}
```

**Dependencies**:
- ↓ Content Loader (templates)
- ↓ ECS (entity creation)
- ↑ Animation System (spawn effects)

---

## Cross-Cutting Concerns

### Entity-Component-System (ECS)

**Responsibilities**:
- Entity lifetime management
- Component storage and queries
- System scheduling and execution
- Parallel query execution

**Key Types**:
```rust
// If using Bevy ECS:
use bevy_ecs::prelude::*;

// Custom components:
#[derive(Component)]
struct Position(HexPosition);

#[derive(Component)]
struct UnitStats {
    hp: i32,
    max_hp: i32,
    armor: i32,
    // ...
}

#[derive(Component)]
struct AnimationState {
    current: AnimationId,
    frame: usize,
    elapsed: f32,
}

// Systems:
fn movement_system(
    mut query: Query<(&mut Position, &MovementTarget)>,
    time: Res<Time>,
) {
    // Update positions
}
```

**System Execution Order**:
```rust
fn build_schedule() -> Schedule {
    Schedule::default()
        // Input processing
        .add_system(input_system)
        
        // Gameplay logic
        .add_system(ai_system.after(input_system))
        .add_system(combat_system.after(ai_system))
        .add_system(movement_system.after(combat_system))
        .add_system(animation_system.after(movement_system))
        
        // Rendering (read-only)
        .add_system(render_system.after(animation_system))
        .add_system(ui_system.after(render_system))
}
```

---

### Resource Management

**Responsibilities**:
- Memory budgets and limits
- Asset streaming and unloading
- Garbage collection (for pooled objects)
- Performance monitoring

**Key Types**:
```rust
struct ResourceManager {
    memory_tracker: MemoryTracker,
    asset_manager: AssetManager,
    entity_pool: EntityPool,
    performance_stats: PerformanceStats,
}

struct MemoryTracker {
    texture_memory: usize,
    audio_memory: usize,
    entity_memory: usize,
    total_limit: usize,
}
```

**Policies**:
- Textures: Max 512MB, unload LRU when over limit
- Audio: Streaming for music, preload for SFX
- Entities: Pool up to 1000 entities
- Content: Keep all referenced content loaded

---

### Error Handling

**Strategy**:
- Use `Result<T, E>` for recoverable errors
- Use `panic!` only for programmer errors (invariants violated)
- Log errors with context (tracing crate)
- Graceful degradation (fallback assets, skip broken content)

**Error Types**:
```rust
#[derive(Debug, thiserror::Error)]
enum GameError {
    #[error("Asset not found: {0}")]
    AssetNotFound(String),
    
    #[error("Invalid content: {path}: {reason}")]
    InvalidContent { path: PathBuf, reason: String },
    
    #[error("Combat error: {0}")]
    CombatError(String),
    
    #[error("Render error: {0}")]
    RenderError(#[from] wgpu::Error),
}

type Result<T> = std::result::Result<T, GameError>;
```

**Error Handling Patterns**:
```rust
// Load with fallback
let texture = asset_manager.load("sprite.png")
    .unwrap_or_else(|e| {
        error!("Failed to load sprite: {}", e);
        asset_manager.fallback_texture()
    });

// Propagate errors up
fn load_scenario(path: &Path) -> Result<Scenario> {
    let data = fs::read_to_string(path)
        .map_err(|e| GameError::InvalidContent {
            path: path.to_path_buf(),
            reason: e.to_string(),
        })?;
    
    toml::from_str(&data)
        .map_err(|e| GameError::InvalidContent {
            path: path.to_path_buf(),
            reason: e.to_string(),
        })
}
```

---

## Component Interaction Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    Game Loop                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐             │
│  │  Input   │→ │Gameplay  │→ │ Render   │             │
│  │ Systems  │  │ Systems  │  │ Systems  │             │
│  └──────────┘  └──────────┘  └──────────┘             │
└─────────────────────────────────────────────────────────┘
         ↓              ↓              ↓
┌─────────────────────────────────────────────────────────┐
│                    ECS Core                             │
│  ┌──────────────────────────────────────────────┐      │
│  │  Entities │ Components │ Resources            │      │
│  └──────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────┘
         ↓              ↓              ↓
┌─────────────────────────────────────────────────────────┐
│              Shared Infrastructure                      │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐               │
│  │Event │  │Asset │  │Audio │  │Save/ │               │
│  │ Bus  │  │ Mgr  │  │Engine│  │Load  │               │
│  └──────┘  └──────┘  └──────┘  └──────┘               │
└─────────────────────────────────────────────────────────┘
```

## Threading Model

**Main Thread**:
- Input processing
- Gameplay logic (turn-based, no tight timing)
- ECS system execution
- Rendering command submission

**Background Threads**:
- Asset loading (async)
- Audio mixing (separate thread)
- File watching (hot-reload)
- Pathfinding (expensive, can be async)

**Synchronization**:
```rust
// Asset loading
let texture_handle = asset_manager.load_async("sprite.png");
// Returns immediately, texture loads in background

// Later, check if ready:
if texture_handle.is_ready() {
    let texture = texture_handle.get();
}

// Or block until ready:
let texture = texture_handle.wait();
```

**Concurrency Tools**:
- `Arc` for shared ownership across threads
- `Mutex` / `RwLock` for interior mutability
- Channels (`mpsc`) for thread communication
- `async`/`await` for async operations
- Bevy ECS parallel query system

---

## Performance Characteristics

### Expected Bottlenecks

1. **Rendering**: Sprite batching and depth sorting
   - Solution: Spatial hashing, instancing, early culling

2. **Pathfinding**: Large maps or many AI units
   - Solution: Caching, flow fields, hierarchical pathfinding

3. **Asset Loading**: Large textures or audio files
   - Solution: Streaming, compression, async loading

4. **Event Processing**: Many events per frame
   - Solution: Event batching, filtering, prioritization

### Performance Budgets (per frame at 60 FPS = 16.67ms)

- Input: 0.5ms
- Gameplay logic: 4ms
- AI: 2ms
- Pathfinding: 1ms
- Animation: 1ms
- Physics/collision: 1ms
- Rendering (CPU): 3ms
- Rendering (GPU): 4ms
- Audio: 0.5ms (separate thread)
- Margin: 2ms

### Profiling Strategy

```rust
// Use tracing for hierarchical profiling
use tracing::instrument;

#[instrument]
fn update_ai_systems(world: &mut World) {
    // Automatically tracked by tracing
    for controller in &mut ai_controllers {
        controller.evaluate(world);
    }
}

// Or manual spans for fine-grained control
fn render_frame() {
    let _span = tracing::span!(tracing::Level::INFO, "render_frame").entered();
    
    {
        let _span = tracing::span!(tracing::Level::DEBUG, "depth_sort").entered();
        depth_sort_sprites();
    }
    
    {
        let _span = tracing::span!(tracing::Level::DEBUG, "draw_calls").entered();
        submit_draw_calls();
    }
}
```

Use `tracy` or `puffin` for real-time profiling GUI.
