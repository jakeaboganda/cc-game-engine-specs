# Level Editor Specification

## Overview

Integrated map editor for creating tactical scenarios in the CC Game Engine. Supports hex-based terrain creation, elevation sculpting, unit placement, and scenario configuration. Built directly into the engine for WYSIWYG editing with live preview.

## Design Philosophy

**Ease of Use**: Non-programmers should be able to create compelling maps
**Visual Feedback**: Instant preview of changes in-game rendering
**Iteration Speed**: Hot-reload, undo/redo, quick iteration
**Power User Tools**: Advanced features for experienced designers
**Export Flexibility**: Multiple formats (TOML, JSON, binary)

## Editor Architecture

```rust
struct LevelEditor {
    mode: EditorMode,
    active_map: EditableMap,
    tool: EditorTool,
    selection: Selection,
    undo_stack: UndoStack,
    camera: IsometricCamera,
    ui_state: EditorUIState,
    preview_mode: PreviewMode,
}

enum EditorMode {
    Terrain,      // Paint terrain, adjust elevation
    Objects,      // Place props, cover, terrain features
    Units,        // Spawn points, unit configuration
    Scenario,     // Victory conditions, objectives
    Testing,      // Playtest the map
}

enum EditorTool {
    // Terrain tools
    Paint { brush: Brush, terrain_type: TerrainType },
    Elevate { brush: Brush, amount: i32 },
    Smooth { brush: Brush, strength: f32 },
    Ramp { start: HexCoord, end: HexCoord },
    
    // Object tools
    Place { object_type: ObjectType },
    Move,
    Rotate,
    Delete,
    
    // Unit tools
    PlaceUnit { unit_type: UnitTypeId },
    PlaceSpawnZone { team: Team, radius: i32 },
    
    // Selection tools
    Select,
    BoxSelect,
    
    // Navigation
    Pan,
    Zoom,
    Rotate,
}
```

## Terrain Editing

### Terrain Painting

```rust
struct TerrainPainter {
    brush: Brush,
    terrain_type: TerrainType,
    blend_mode: BlendMode,
}

struct Brush {
    shape: BrushShape,
    size: i32,          // Radius in hexes
    falloff: Falloff,
    strength: f32,      // 0.0 to 1.0
}

enum BrushShape {
    Circle,
    Square,
    Hex,
    Custom(Vec<HexCoord>),  // User-defined pattern
}

enum Falloff {
    Linear,      // Edge fades linearly
    Smooth,      // Ease-in/out
    Hard,        // Sharp edge
}

enum BlendMode {
    Replace,     // Overwrite existing terrain
    Blend,       // Mix with existing (for transitions)
    Overlay,     // Add on top (for decals)
}
```

**Painting workflow**:
1. Select terrain type from palette (grass, stone, water, etc.)
2. Choose brush size/shape
3. Click and drag to paint
4. Automatic texture blending at borders

**Visual feedback**:
- Brush preview (translucent circle showing affected area)
- Real-time texture updates
- Minimap reflection

### Elevation Sculpting

```rust
struct ElevationTool {
    brush: Brush,
    mode: ElevationMode,
}

enum ElevationMode {
    Raise { amount: i32 },      // Raise by N levels
    Lower { amount: i32 },      // Lower by N levels
    Flatten { target_level: i32 }, // Set to specific elevation
    Smooth { strength: f32 },   // Average with neighbors
    Terrace { levels: Vec<i32> }, // Create stepped levels
}

impl ElevationTool {
    fn apply(&self, map: &mut EditableMap, center: HexCoord) {
        let affected_hexes = self.brush.get_affected_hexes(center);
        
        for hex in affected_hexes {
            match self.mode {
                ElevationMode::Raise { amount } => {
                    map.elevate_hex(hex, amount);
                },
                ElevationMode::Flatten { target_level } => {
                    map.set_elevation(hex, target_level);
                },
                ElevationMode::Smooth { strength } => {
                    let avg = map.average_neighbor_elevation(hex);
                    let current = map.get_elevation(hex);
                    let new_elev = lerp(current, avg, strength);
                    map.set_elevation(hex, new_elev);
                },
                // ...
            }
        }
        
        // Auto-generate cliff faces where needed
        map.update_cliff_geometry();
    }
}
```

**Elevation editing features**:
- **Raise/Lower**: Click and drag to sculpt terrain
- **Flatten**: Make area perfectly flat at target elevation
- **Smooth**: Blend elevation changes for gentle slopes
- **Terrace**: Create stepped plateaus automatically
- **Noise**: Add procedural variation
- **Copy elevation**: Sample elevation from one area, paste to another

**Visual indicators**:
- Color-coded elevation overlay (heatmap)
- Elevation numbers on hexes
- Grid showing cliff faces
- 3D wireframe preview

### Ramp Creation

```rust
struct RampTool {
    start: HexCoord,
    end: HexCoord,
    width: i32,
    style: RampStyle,
}

enum RampStyle {
    Straight,     // Direct line from A to B
    Curved,       // Smooth arc
    Switchback,   // Zigzag path for large elevation changes
}

impl RampTool {
    fn create_ramp(&self, map: &mut EditableMap) {
        let start_elev = map.get_elevation(self.start);
        let end_elev = map.get_elevation(self.end);
        let elevation_diff = end_elev - start_elev;
        
        let path = self.calculate_path();
        let step = elevation_diff as f32 / path.len() as f32;
        
        for (i, hex) in path.iter().enumerate() {
            let elevation = start_elev + (step * i as f32) as i32;
            map.set_elevation(*hex, elevation);
            
            // Create slope corners
            map.set_corner_elevations(*hex, self.calculate_slope_corners(i, &path));
        }
        
        map.update_cliff_geometry();
    }
}
```

**Usage**:
1. Click start position
2. Click end position (different elevation)
3. Tool automatically creates smooth ramp
4. Adjust width, curvature with controls

### Cliff Editing

```rust
struct CliffTool {
    mode: CliffMode,
}

enum CliffMode {
    Auto,       // Auto-generate cliffs where elevation jumps 2+ levels
    Manual,     // Manually place cliff faces
    Remove,     // Remove cliff, replace with slope
    Style { cliff_type: CliffType },
}

enum CliffType {
    Rock,
    Ice,
    Wood,       // Wooden palisade
    Metal,      // Metal wall
}
```

**Auto-cliff generation**:
- Detects elevation differences > 1 level
- Automatically renders appropriate cliff face
- Chooses corner pieces correctly
- Matches terrain type (grass cliff vs stone cliff)

**Manual mode**:
- Fine control over cliff placement
- Override auto-generated cliffs
- Custom cliff styles (walls, fortifications)

## Object Placement

### Terrain Features

```rust
struct ObjectPlacer {
    object_library: HashMap<String, ObjectTemplate>,
    placement_mode: PlacementMode,
}

struct ObjectTemplate {
    id: String,
    name: String,
    sprite: String,
    footprint: Vec<HexCoord>,  // Hexes occupied
    blocks_movement: bool,
    blocks_los: bool,
    cover_level: CoverLevel,
    height: i32,               // Visual height (for rendering)
}

enum PlacementMode {
    Single,         // Place one object
    Scatter,        // Random scatter within brush
    Line,           // Place along a line
    Grid,           // Evenly spaced grid
}
```

**Object categories**:
- **Trees**: Forests, single trees, dead trees
- **Rocks**: Boulders, rock formations, cliffs
- **Structures**: Houses, walls, towers, ruins
- **Cover**: Crates, barrels, sandbags, low walls
- **Water features**: Lakes, rivers, waterfalls
- **Props**: Campfires, tents, treasure chests, markers

**Placement workflow**:
1. Select object from library (categorized browser)
2. Click to place or drag to scatter
3. Object snaps to hex grid
4. Visual preview before placement
5. Properties panel for adjustments

**Object properties**:
- Position (hex + elevation)
- Rotation (8 directions)
- Scale (for decoration only, not gameplay)
- Tint/color variation
- Block movement/LOS flags
- Cover level
- Custom metadata (for scenario scripting)

### Prefabs

```rust
struct Prefab {
    name: String,
    objects: Vec<PlacedObject>,
    bounding_box: Rect,
}

struct PlacedObject {
    template_id: String,
    offset: HexCoord,      // Relative to prefab origin
    elevation_offset: i32,
    rotation: Direction,
    properties: HashMap<String, String>,
}
```

**Prefab examples**:
- **Watchtower**: Tower + walls + ladder
- **Forest clearing**: Circle of trees with open center
- **Camp**: Tents + campfire + crates
- **Bridge**: Multiple sections spanning gap
- **Ruins**: Partial walls + rubble + props

**Usage**:
1. Create complex structure once
2. Save as prefab
3. Reuse across multiple maps
4. Update prefab → updates all instances

## Unit Placement

### Spawn Zones

```rust
struct SpawnZone {
    team: Team,
    center: HexPosition,
    shape: SpawnShape,
    units: Vec<UnitSpawn>,
}

enum SpawnShape {
    Point,                    // Single hex
    Circle { radius: i32 },
    Rectangle { width: i32, height: i32 },
    Custom(Vec<HexCoord>),
}

struct UnitSpawn {
    unit_type: UnitTypeId,
    position: Option<HexPosition>,  // None = random in zone
    facing: Option<Direction>,
    level: u32,
    equipment: Vec<ItemId>,
    ai_behavior: String,
}

enum Team {
    Player,
    Enemy,
    Neutral,
    Ally,
}
```

**Spawn zone editor**:
1. Select team (player, enemy, neutral)
2. Draw spawn area on map
3. Add unit types to zone
4. Configure individual units
   - Level
   - Equipment
   - AI behavior
   - Starting position (fixed or random)
   - Facing direction

**Visual representation**:
- Color-coded overlays (blue = player, red = enemy, yellow = neutral)
- Semi-transparent zone boundaries
- Unit icons showing spawns
- Preview mode: Show units at spawn locations

### Unit Configuration

```rust
struct UnitConfigurator {
    unit_type: UnitTypeId,
    overrides: UnitOverrides,
}

struct UnitOverrides {
    hp: Option<i32>,
    stats: HashMap<Stat, i32>,
    abilities: Vec<AbilityId>,
    equipment: Vec<ItemId>,
    ai_personality: Option<String>,
    behavior_overrides: HashMap<String, String>,
}
```

**Per-unit customization**:
- Override stats (HP, damage, speed, etc.)
- Add/remove abilities
- Equip custom gear
- Set AI personality (aggressive, defensive, etc.)
- Custom behavior flags

**Templates**:
- Save unit configurations as templates
- "Elite Warrior", "Weak Scout", "Boss Mage", etc.
- Reuse across scenarios

## Scenario Configuration

```rust
struct ScenarioEditor {
    metadata: ScenarioMetadata,
    objectives: Vec<Objective>,
    events: Vec<ScenarioEvent>,
    variables: HashMap<String, ScenarioValue>,
}

struct ScenarioMetadata {
    name: String,
    description: String,
    difficulty: Difficulty,
    max_turns: Option<u32>,
    recommended_level: u32,
    author: String,
    tags: Vec<String>,
}

struct Objective {
    id: String,
    description: String,
    condition: ObjectiveCondition,
    required: bool,  // vs optional
}

enum ObjectiveCondition {
    EliminateAll { team: Team },
    EliminateTarget { unit_id: String },
    ReachLocation { positions: Vec<HexPosition>, team: Team },
    Survive { turns: u32 },
    Protect { unit_id: String, until_turn: u32 },
    Collect { item_id: String, count: u32 },
    Custom { script: String },
}

struct ScenarioEvent {
    trigger: EventTrigger,
    actions: Vec<EventAction>,
}

enum EventTrigger {
    TurnStart(u32),
    TurnEnd(u32),
    UnitDeath(String),
    VariableEquals(String, ScenarioValue),
    PlayerEntersZone(Vec<HexCoord>),
}

enum EventAction {
    SpawnUnits { zone_id: String },
    ShowMessage { text: String },
    SetVariable { name: String, value: ScenarioValue },
    ChangeObjective { objective_id: String, new_state: ObjectiveState },
    EndScenario { result: ScenarioResult },
}
```

**Scenario editor UI**:

### Objectives Panel
- Add/remove objectives
- Set victory/defeat conditions
- Mark required vs optional
- Preview objective descriptions

### Events Timeline
- Visual timeline showing triggers
- Drag-and-drop event creation
- Test event chains
- Debug event execution

### Variables
- Define scenario state variables
- Track progress (enemies_killed, items_collected, etc.)
- Conditional logic based on variables

**Example scenario setup**:
```toml
[scenario]
name = "Defend the Village"
description = "Hold off waves of enemies until reinforcements arrive"
max_turns = 15

[[objectives]]
id = "survive"
description = "Survive for 15 turns"
condition = { type = "survive", turns = 15 }
required = true

[[objectives]]
id = "protect_elder"
description = "Keep the village elder alive"
condition = { type = "protect", unit_id = "elder", until_turn = 15 }
required = true

[[events]]
trigger = { type = "turn_start", turn = 5 }
actions = [
    { type = "spawn_units", zone_id = "reinforcement_wave_1" },
    { type = "show_message", text = "More enemies approaching from the north!" },
]

[[events]]
trigger = { type = "turn_start", turn = 10 }
actions = [
    { type = "spawn_units", zone_id = "reinforcement_wave_2" },
]

[[events]]
trigger = { type = "variable_equals", name = "enemies_killed", value = 10 }
actions = [
    { type = "show_message", text = "Their numbers are thinning!" },
]
```

## Testing & Preview

### Preview Modes

```rust
enum PreviewMode {
    Edit,           // Normal edit mode
    Play,           // Fully playable test
    LineOfSight,    // Visualize LOS from selected positions
    Pathfinding,    // Show movement ranges and paths
    Elevation,      // Heatmap of elevations
    Lighting,       // Preview lighting (if implemented)
}
```

**Edit mode**:
- See terrain, objects, spawn zones
- Edit tools active
- No game logic running

**Play mode** (in-editor testing):
- Click "Test" button
- Spawns units at designated zones
- Fully playable combat
- Can end test and return to edit mode
- Maintains edit state (can test, adjust, test again)

**LOS preview**:
- Click hex → shows LOS from that position
- Color-coded visible/hidden areas
- Accounts for elevation and cover
- Test multiple positions to verify balance

**Pathfinding preview**:
- Select unit type → shows movement range
- Click destination → shows path
- Highlights impassable terrain
- Verifies map connectivity

**Elevation preview**:
- Heatmap visualization (red = high, blue = low)
- Toggle between rendering modes
- Helps verify elevation makes sense

### Quick Test

```rust
struct QuickTestConfig {
    spawn_player_at: Option<HexPosition>,
    spawn_enemies: bool,
    use_scenario_rules: bool,
    ai_enabled: bool,
    invincible_mode: bool,  // For testing without dying
}
```

**One-click testing**:
- "Quick Test" button spawns player at cursor position
- Optional: Spawn enemies from nearest spawn zone
- Test specific areas of map quickly
- Exit back to editing instantly

## Camera & Navigation

### Camera Controls

```rust
struct EditorCamera {
    camera: IsometricCamera,
    mode: CameraMode,
    bookmarks: Vec<CameraBookmark>,
}

enum CameraMode {
    Free,       // Standard WASD + mouse
    Orbit,      // Rotate around selected object/hex
    Track,      // Follow selected unit in preview
}

struct CameraBookmark {
    name: String,
    position: Vec3,
    zoom: f32,
    rotation: f32,
}
```

**Navigation**:
- **WASD**: Pan camera
- **Mouse wheel**: Zoom in/out
- **Middle mouse drag**: Pan
- **Q/E**: Rotate view (90° increments)
- **F**: Frame selection (zoom to selected objects)
- **Home**: Reset camera to default view

**Bookmarks**:
- Save camera positions for quick navigation
- "Spawn area 1", "Boss arena", "Bridge choke point", etc.
- Hotkeys (Ctrl+1-9) to jump to bookmarks

### Minimap

```rust
struct EditorMinimap {
    zoom: f32,
    show_elevation: bool,
    show_objects: bool,
    show_spawns: bool,
    show_objectives: bool,
}
```

**Features**:
- Bird's-eye view of entire map
- Click to jump camera
- Toggle layers (terrain, objects, units, objectives)
- Export as image for reference

## Grid & Snapping

```rust
struct GridSettings {
    show_grid: bool,
    show_coordinates: bool,
    snap_to_grid: bool,
    highlight_walkable: bool,
    highlight_los_blockers: bool,
}
```

**Grid overlays**:
- Hex outlines
- Coordinate labels (q, r)
- Elevation numbers
- Walkable/impassable coloring
- LOS blocker highlighting

**Snapping**:
- Objects snap to hex centers by default
- Hold Shift to disable snapping
- Fine-tune with arrow keys

## Layers & Visibility

```rust
struct LayerManager {
    layers: Vec<EditorLayer>,
    active_layer: usize,
}

struct EditorLayer {
    name: String,
    visible: bool,
    locked: bool,
    contents: LayerContents,
}

enum LayerContents {
    Terrain,
    TerrainFeatures,
    Objects,
    Units,
    Spawns,
    Objectives,
    Effects,
    Annotations,  // Notes, markers, measurement tools
}
```

**Layer system**:
- Toggle visibility per layer
- Lock layers to prevent accidental edits
- Isolate layer (hide all others)
- Rename layers
- Reorder layers (for complex scenes)

## Selection & Manipulation

### Selection Tools

```rust
struct SelectionManager {
    selection: Vec<SelectableEntity>,
    mode: SelectionMode,
}

enum SelectableEntity {
    Hex(HexCoord),
    Object(ObjectId),
    Unit(UnitId),
    SpawnZone(ZoneId),
}

enum SelectionMode {
    Single,      // Click to select one
    Add,         // Shift+click to add to selection
    Subtract,    // Ctrl+click to remove from selection
    Box,         // Drag box to select multiple
}
```

**Multi-selection**:
- Select multiple objects/hexes
- Apply changes to all selected
- Group operations (move, delete, duplicate)

### Transform Tools

```rust
struct TransformTool {
    mode: TransformMode,
    pivot: TransformPivot,
    snap_increment: f32,
}

enum TransformMode {
    Move,
    Rotate,
    Scale,     // For props only, not gameplay-affecting
    Elevate,   // Move vertically (change elevation)
}

enum TransformPivot {
    Individual,   // Each object around its center
    Center,       // All around shared center
    Selection,    // Around selection bounding box center
}
```

**Usage**:
- **G**: Grab/move selected objects
- **R**: Rotate selected objects
- **E**: Elevate (change elevation level)
- **S**: Scale (decorative only)
- **Arrow keys**: Nudge selection

## Properties Panel

Context-sensitive panel showing properties of selection:

### Hex Properties
- Coordinates (q, r, s)
- Elevation
- Terrain type
- Corner elevations (for slopes)
- Movement cost
- Special flags (hazard, objective marker, etc.)

### Object Properties
- Name / ID
- Position
- Rotation
- Blocks movement / LOS
- Cover level
- Height (for rendering)
- Custom data (key-value pairs)

### Unit Properties
- Unit type
- Team
- Level
- Stats overrides
- Equipment
- AI behavior
- Spawn conditions

### Zone Properties
- Team
- Shape (point, circle, rectangle)
- Unit list
- Spawn timing (turn 0, triggered, etc.)

## Validation & Errors

```rust
struct MapValidator {
    errors: Vec<ValidationError>,
    warnings: Vec<ValidationWarning>,
}

enum ValidationError {
    NoPlayerSpawn,
    NoObjectives,
    UnreachableArea { hexes: Vec<HexCoord> },
    InvalidElevation { hex: HexCoord, reason: String },
    MissingAsset { path: String },
}

enum ValidationWarning {
    LargeElevationJump { hex: HexCoord },
    UnbalancedSpawns { player_count: usize, enemy_count: usize },
    LongPathfindingDistance,
    NoEnemies,
    TooManyUnits { count: usize, recommended: usize },
}
```

**Auto-validation**:
- Runs on save
- Checks for common issues
- Highlights problems on map
- Suggests fixes

**Validation checks**:
- At least one player spawn exists
- At least one objective defined
- All areas reachable via pathfinding
- No isolated hexes (surrounded by impassable terrain)
- No impossible elevation jumps (2+ levels without ramp)
- All referenced assets exist
- Unit counts reasonable for map size
- Balance suggestions (player vs enemy forces)

## File Operations

### Save / Load

```rust
struct MapFile {
    version: String,
    metadata: ScenarioMetadata,
    map: MapData,
    objects: Vec<PlacedObject>,
    spawns: Vec<SpawnZone>,
    scenario: ScenarioConfig,
}
```

**File formats**:
- **.toml**: Human-readable, VCS-friendly
- **.json**: For tool interop
- **.pak**: Binary, compressed (for distribution)

**Autosave**:
- Save every N minutes (configurable)
- Keep last 5 autosaves
- Recover from autosave on crash

**Version control**:
- TOML format works with Git
- Track changes line-by-line
- Easy collaboration

### Export Options

```rust
enum ExportFormat {
    GameReady,      // Optimized for game (binary, compressed)
    Editable,       // Human-readable (TOML)
    Image,          // Render map to PNG (for documentation)
    Heightmap,      // Export elevation as grayscale image
}
```

**Export targets**:
- **Game**: Fully playable scenario
- **Share**: Standalone file for community
- **Screenshot**: High-res render for showcase
- **Heightmap**: For importing to other tools

### Import

```rust
enum ImportSource {
    FromFile(PathBuf),
    FromHeightmap { image: PathBuf, scale: f32 },
    FromTemplate(TemplateId),
}
```

**Import heightmap**:
- Import elevation data from grayscale image
- Auto-generate terrain based on height
- Useful for natural-looking terrain

**Templates**:
- "Open Field", "Forest", "Castle Siege", etc.
- Pre-configured starting points
- Customizable after import

## UI/UX Design

### Layout

```
┌──────────────────────────────────────────────────┐
│  Menu Bar: File | Edit | View | Tools | Scenario│
├──────┬───────────────────────────────────┬───────┤
│      │                                   │       │
│ Tool │                                   │ Prop- │
│ Bar  │         Main Viewport             │ erties│
│      │      (Isometric 3D View)          │ Panel │
│ [🖌] │                                   │       │
│ [⬆] │                                   │  ...  │
│ [🌲] │                                   │       │
│ [👤] │                                   │       │
│ [⚙] │                                   │       │
│      ├───────────────────────────────────┤       │
│      │      Timeline / Events            │       │
└──────┴───────────────────────────────────┴───────┘
```

### Tool Palette

**Terrain Tab**:
- Paint terrain
- Elevate/lower
- Smooth
- Create ramp
- Create cliff

**Objects Tab**:
- Object browser (categorized)
- Place object
- Move/rotate/delete
- Prefab library

**Units Tab**:
- Unit browser
- Place spawn zone
- Configure units
- AI behavior

**Scenario Tab**:
- Objectives
- Events
- Variables
- Metadata

### Keyboard Shortcuts

**General**:
- `Ctrl+N`: New map
- `Ctrl+O`: Open map
- `Ctrl+S`: Save
- `Ctrl+Z`: Undo
- `Ctrl+Y`: Redo
- `Ctrl+D`: Duplicate selection
- `Delete`: Delete selection
- `F`: Frame selection
- `Space`: Quick test at cursor

**Tools**:
- `B`: Terrain paint brush
- `E`: Elevation tool
- `T`: Ramp tool
- `O`: Place object
- `U`: Place unit
- `G`: Grab/move
- `R`: Rotate
- `S`: Scale

**View**:
- `Tab`: Toggle UI visibility (full-screen viewport)
- `H`: Toggle grid
- `M`: Toggle minimap
- `L`: Toggle layer visibility
- `1-9`: Jump to camera bookmarks

## Performance Considerations

**Large maps**:
- Chunk-based rendering (only render visible area)
- LOD for distant objects
- Lazy loading of assets
- Background autosave (don't freeze editor)

**Responsiveness**:
- Undo/redo operations < 50ms
- Brush strokes render in real-time (60 FPS)
- Preview mode transition < 200ms

**Memory management**:
- Unload unused textures
- Compress autosaves
- Limit undo stack size (last 50 actions)

## Future Extensions

- **Scripting**: Lua/Rhai for custom behaviors
- **Terrain procedural generation**: Noise-based natural terrain
- **AI navmesh visualization**: Show where AI can pathfind
- **Multiplayer preview**: Test with multiple players
- **Asset creation tools**: Sprite/tileset editor integration
- **Localization**: Multi-language scenario text
- **Workshop integration**: Steam Workshop or similar for sharing maps
