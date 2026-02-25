# Module Organization

## Crate Structure

Recommended workspace organization for the CC Game Engine.

```
cc-game-engine/          (workspace root)
├── Cargo.toml           (workspace manifest)
├── crates/
│   ├── cc-engine/       (core engine)
│   ├── cc-tactics/      (tactics framework)
│   ├── cc-content/      (content loading)
│   ├── cc-editor/       (level editor)
│   └── cc-game/         (game executable)
├── content/             (game content - TOML/assets)
├── tools/               (build tools, scripts)
└── docs/                (additional documentation)
```

## Crate Dependency Graph

```
cc-game (exe)
  ↓
  ├─ cc-editor
  │    ↓
  │    └─ cc-tactics
  │         ↓
  │         └─ cc-engine
  │
  └─ cc-content
       ↓
       └─ cc-tactics
            ↓
            └─ cc-engine
```

## Core Engine Crate (`cc-engine`)

**Purpose**: Platform-agnostic game engine primitives

```
cc-engine/
├── Cargo.toml
└── src/
    ├── lib.rs              (public API)
    │
    ├── render/
    │   ├── mod.rs
    │   ├── camera.rs       (IsometricCamera, Camera2D)
    │   ├── sprite.rs       (Sprite, SpriteSheet, SpriteBatcher)
    │   ├── depth_sort.rs   (DepthSorter, render ordering)
    │   ├── pipeline.rs     (RenderPipeline, passes)
    │   ├── mesh.rs         (3D models, optional)
    │   └── shader.rs       (Shader management)
    │
    ├── input/
    │   ├── mod.rs
    │   ├── manager.rs      (InputManager, device polling)
    │   ├── action_map.rs   (ActionMap, bindings)
    │   ├── gamepad.rs      (Gamepad handling)
    │   └── keyboard.rs     (Keyboard + mouse)
    │
    ├── audio/
    │   ├── mod.rs
    │   ├── engine.rs       (AudioEngine, playback)
    │   ├── mixer.rs        (AudioMixer, buses)
    │   ├── spatial.rs      (Spatial audio)
    │   └── music.rs        (MusicPlayer, streaming)
    │
    ├── asset/
    │   ├── mod.rs
    │   ├── manager.rs      (AssetManager, caching)
    │   ├── handle.rs       (AssetHandle<T>)
    │   ├── loader.rs       (Async loading)
    │   ├── hot_reload.rs   (File watching)
    │   └── pack.rs         (Asset packing)
    │
    ├── event/
    │   ├── mod.rs
    │   ├── bus.rs          (EventBus, pub/sub)
    │   ├── handler.rs      (EventHandler trait)
    │   └── recorder.rs     (Event recording/replay)
    │
    ├── math/
    │   ├── mod.rs
    │   ├── vec.rs          (Vec2, Vec3, Vec4)
    │   ├── matrix.rs       (Mat4, transforms)
    │   ├── rect.rs         (Rect, bounds)
    │   └── color.rs        (Color, gradients)
    │
    └── util/
        ├── mod.rs
        ├── time.rs         (Time, Clock, Timer)
        ├── pool.rs         (Object pooling)
        ├── lru.rs          (LRU cache)
        └── error.rs        (EngineError types)
```

**Public API Example**:
```rust
// lib.rs
pub mod render;
pub mod input;
pub mod audio;
pub mod asset;
pub mod event;
pub mod math;
pub mod util;

// Re-exports for convenience
pub use render::{RenderEngine, Camera, Sprite};
pub use input::{InputManager, GameAction};
pub use audio::AudioEngine;
pub use asset::AssetManager;
pub use event::EventBus;
```

---

## Tactics Framework Crate (`cc-tactics`)

**Purpose**: Hex-grid tactical game systems

```
cc-tactics/
├── Cargo.toml
│   dependencies:
│     cc-engine = { path = "../cc-engine" }
│
└── src/
    ├── lib.rs
    │
    ├── hex/
    │   ├── mod.rs
    │   ├── coord.rs        (HexCoord, HexPosition)
    │   ├── grid.rs         (HexGrid, tile storage)
    │   ├── pathfinding.rs  (A* implementation)
    │   ├── line_of_sight.rs (LOS calculations)
    │   └── spatial.rs      (Spatial hashing 3D)
    │
    ├── combat/
    │   ├── mod.rs
    │   ├── turn_manager.rs (TurnManager, initiative)
    │   ├── action.rs       (GameAction enum)
    │   ├── resolver.rs     (Combat resolution)
    │   ├── damage.rs       (Damage calculation)
    │   └── status.rs       (Status effects)
    │
    ├── ai/
    │   ├── mod.rs
    │   ├── behavior_tree.rs (BehaviorTree, nodes)
    │   ├── utility.rs      (Utility-based AI)
    │   ├── controller.rs   (AIController)
    │   ├── personality.rs  (AIPersonality)
    │   └── blackboard.rs   (AIBlackboard)
    │
    ├── animation/
    │   ├── mod.rs
    │   ├── controller.rs   (AnimationController)
    │   ├── tween.rs        (Tweening, easing)
    │   ├── sprite_anim.rs  (Sprite animation)
    │   └── movement.rs     (Movement animation)
    │
    ├── state/
    │   ├── mod.rs
    │   ├── machine.rs      (GameStateMachine)
    │   ├── turn.rs         (Turn-based state)
    │   └── victory.rs      (Victory conditions)
    │
    └── ecs/
        ├── mod.rs
        ├── components.rs   (Position, Stats, etc.)
        ├── systems.rs      (System definitions)
        └── resources.rs    (Global resources)
```

**Public API Example**:
```rust
// lib.rs
pub mod hex;
pub mod combat;
pub mod ai;
pub mod animation;
pub mod state;
pub mod ecs;

pub use hex::{HexCoord, HexPosition, HexGrid};
pub use combat::{TurnManager, GameAction};
pub use ai::AIController;
pub use state::GameStateMachine;
```

---

## Content Crate (`cc-content`)

**Purpose**: Data loading, validation, and hot-reload

```
cc-content/
├── Cargo.toml
│   dependencies:
│     cc-engine = { path = "../cc-engine" }
│     cc-tactics = { path = "../cc-tactics" }
│     serde = { version = "1.0", features = ["derive"] }
│     toml = "0.8"
│
└── src/
    ├── lib.rs
    │
    ├── loader/
    │   ├── mod.rs
    │   ├── unit.rs         (UnitDefinition loader)
    │   ├── ability.rs      (AbilityDefinition loader)
    │   ├── map.rs          (MapDefinition loader)
    │   └── scenario.rs     (ScenarioDefinition loader)
    │
    ├── registry/
    │   ├── mod.rs
    │   ├── content.rs      (ContentRegistry)
    │   ├── cache.rs        (Content caching)
    │   └── validator.rs    (Content validation)
    │
    ├── definitions/
    │   ├── mod.rs
    │   ├── unit.rs         (UnitDefinition struct)
    │   ├── ability.rs      (AbilityDefinition struct)
    │   ├── map.rs          (MapDefinition struct)
    │   ├── scenario.rs     (ScenarioDefinition struct)
    │   └── item.rs         (ItemDefinition struct)
    │
    ├── factory/
    │   ├── mod.rs
    │   ├── entity.rs       (EntityFactory)
    │   └── spawn.rs        (Spawn helpers)
    │
    └── mod_support/
        ├── mod.rs
        ├── loader.rs       (ModLoader)
        ├── manifest.rs     (Mod manifest parsing)
        └── priority.rs     (Load order, overrides)
```

**Public API Example**:
```rust
// lib.rs
pub mod definitions;
pub mod loader;
pub mod registry;
pub mod factory;
pub mod mod_support;

pub use definitions::{UnitDefinition, AbilityDefinition, MapDefinition};
pub use registry::ContentRegistry;
pub use factory::EntityFactory;
```

---

## Editor Crate (`cc-editor`)

**Purpose**: Level editor and tooling

```
cc-editor/
├── Cargo.toml
│   dependencies:
│     cc-engine = { path = "../cc-engine" }
│     cc-tactics = { path = "../cc-tactics" }
│     cc-content = { path = "../cc-content" }
│     egui = "0.27"
│
└── src/
    ├── lib.rs              (Editor as library)
    │
    ├── ui/
    │   ├── mod.rs
    │   ├── main_window.rs  (Main editor UI)
    │   ├── toolbar.rs      (Tool palette)
    │   ├── properties.rs   (Properties panel)
    │   └── viewport.rs     (3D viewport)
    │
    ├── tools/
    │   ├── mod.rs
    │   ├── terrain.rs      (Terrain painter)
    │   ├── elevation.rs    (Elevation sculpting)
    │   ├── object.rs       (Object placement)
    │   ├── unit.rs         (Unit spawner)
    │   └── selection.rs    (Selection tools)
    │
    ├── map/
    │   ├── mod.rs
    │   ├── editable.rs     (EditableMap)
    │   ├── validator.rs    (Map validation)
    │   └── exporter.rs     (Export to game format)
    │
    ├── scenario/
    │   ├── mod.rs
    │   ├── objectives.rs   (Objective editor)
    │   ├── events.rs       (Event timeline)
    │   └── metadata.rs     (Scenario metadata)
    │
    └── preview/
        ├── mod.rs
        ├── playtest.rs     (In-editor playtesting)
        ├── los_preview.rs  (LOS visualization)
        └── pathfind_preview.rs (Pathfinding preview)
```

---

## Game Executable Crate (`cc-game`)

**Purpose**: Main game executable, ties everything together

```
cc-game/
├── Cargo.toml
│   dependencies:
│     cc-engine = { path = "../cc-engine" }
│     cc-tactics = { path = "../cc-tactics" }
│     cc-content = { path = "../cc-content" }
│     cc-editor = { path = "../cc-editor" }
│
└── src/
    ├── main.rs             (Entry point)
    │
    ├── game/
    │   ├── mod.rs
    │   ├── app.rs          (GameApp, main loop)
    │   ├── states.rs       (Game state handlers)
    │   └── config.rs       (Game configuration)
    │
    ├── ui/
    │   ├── mod.rs
    │   ├── main_menu.rs    (Main menu UI)
    │   ├── hud.rs          (In-game HUD)
    │   ├── pause_menu.rs   (Pause menu)
    │   └── results.rs      (Victory/defeat screen)
    │
    ├── systems/
    │   ├── mod.rs
    │   ├── setup.rs        (System registration)
    │   └── schedule.rs     (System scheduling)
    │
    └── save/
        ├── mod.rs
        ├── manager.rs      (SaveManager)
        └── migration.rs    (Save file migrations)
```

**Main.rs Example**:
```rust
use cc_engine::{RenderEngine, InputManager, AudioEngine, AssetManager, EventBus};
use cc_tactics::{HexGrid, TurnManager, GameStateMachine};
use cc_content::ContentRegistry;
use cc_editor::Editor;

fn main() -> anyhow::Result<()> {
    // Initialize logging
    tracing_subscriber::fmt::init();
    
    // Create engine systems
    let render_engine = RenderEngine::new()?;
    let input_manager = InputManager::new()?;
    let audio_engine = AudioEngine::new()?;
    let asset_manager = AssetManager::new("assets/")?;
    let event_bus = EventBus::new();
    
    // Create gameplay systems
    let hex_grid = HexGrid::new(50, 50);
    let turn_manager = TurnManager::new();
    let state_machine = GameStateMachine::new();
    
    // Load content
    let mut content_registry = ContentRegistry::new();
    content_registry.load_directory("content/")?;
    
    // Build game app
    let mut app = GameApp::new(
        render_engine,
        input_manager,
        audio_engine,
        asset_manager,
        event_bus,
        hex_grid,
        turn_manager,
        state_machine,
        content_registry,
    );
    
    // Run main loop
    app.run()
}
```

---

## Feature Flags

Use Cargo features for optional functionality:

```toml
# cc-engine/Cargo.toml
[features]
default = ["3d-models"]

# Optional features
3d-models = ["dep:obj", "dep:gltf"]
networking = ["dep:quinn", "dep:bincode"]
editor = ["dep:egui"]
profiling = ["dep:tracy-client"]
```

**Usage**:
```rust
#[cfg(feature = "3d-models")]
pub mod mesh;

#[cfg(feature = "networking")]
pub mod netcode;
```

---

## Internal Module Organization

### Visibility Rules

```rust
// Public API (exported from crate)
pub struct HexCoord { /* ... */ }

// Public within crate only
pub(crate) struct InternalHelper { /* ... */ }

// Public within module only
pub(super) fn helper_function() { /* ... */ }

// Private (default)
struct PrivateImplementation { /* ... */ }
```

### Module Pattern

```rust
// terrain/mod.rs (module root)
mod tile;         // terrain/tile.rs (private)
mod painter;      // terrain/painter.rs (private)

pub use tile::Tile;            // Re-export public types
pub use painter::TerrainPainter;

// Provide unified API
pub struct TerrainSystem {
    tiles: HashMap<HexCoord, Tile>,
    painter: TerrainPainter,
}

impl TerrainSystem {
    pub fn new() -> Self { /* ... */ }
    pub fn paint(&mut self, /* ... */) { /* ... */ }
}
```

---

## Testing Organization

```
cc-tactics/
├── src/
│   ├── hex/
│   │   ├── coord.rs
│   │   └── pathfinding.rs
│   └── ...
└── tests/              (Integration tests)
    ├── hex_grid_tests.rs
    ├── pathfinding_tests.rs
    └── combat_tests.rs
```

**Unit tests** (in same file as code):
```rust
// hex/pathfinding.rs
impl HexGrid {
    pub fn find_path(...) -> Option<Vec<HexPosition>> {
        // Implementation
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_straight_path() {
        let grid = HexGrid::new(10, 10);
        let path = grid.find_path(
            HexPosition::new(0, 0, 0),
            HexPosition::new(5, 0, 0),
            MovementType::Walk,
        );
        
        assert!(path.is_some());
        assert_eq!(path.unwrap().len(), 6); // Start + 5 steps
    }
}
```

**Integration tests** (separate files):
```rust
// tests/combat_tests.rs
use cc_tactics::*;

#[test]
fn test_full_combat_round() {
    // Setup game state
    let mut state = create_test_state();
    
    // Execute combat
    let result = state.execute_attack(player_unit, enemy_unit);
    
    // Verify outcome
    assert!(result.hit);
    assert_eq!(enemy_hp_after, enemy_hp_before - expected_damage);
}
```

---

## Documentation

### Doc Comments

```rust
/// Camera for isometric 2.5D rendering.
///
/// Supports pan, zoom, and optional rotation. Projects 3D world coordinates
/// (hex + elevation) to 2D screen coordinates using isometric projection.
///
/// # Example
///
/// ```
/// use cc_engine::render::IsometricCamera;
///
/// let mut camera = IsometricCamera::new(1920, 1080);
/// camera.set_position(Vec3::new(10.0, 10.0, 0.0));
/// camera.set_zoom(1.5);
///
/// let screen_pos = camera.world_to_screen(Vec3::new(5.0, 5.0, 1.0));
/// ```
pub struct IsometricCamera {
    /// World position (x, y, elevation)
    pub position: Vec3,
    
    /// Zoom level (1.0 = default, 2.0 = 2x zoomed)
    pub zoom: f32,
    
    // ... private fields
}
```

### Module Documentation

```rust
//! Hex grid system with 3D positioning.
//!
//! This module provides hex-based coordinate systems, pathfinding, and
//! line-of-sight calculations for tactical games with elevation.
//!
//! # Coordinate Systems
//!
//! - **Cube coordinates**: Primary system for calculations
//! - **Axial coordinates**: Compact storage format
//! - **Offset coordinates**: For displaying to users
//!
//! # Examples
//!
//! ```
//! use cc_tactics::hex::{HexCoord, HexPosition, HexGrid};
//!
//! let grid = HexGrid::new(20, 20);
//! let start = HexPosition::new(0, 0, 0);
//! let goal = HexPosition::new(10, 5, 2);
//!
//! if let Some(path) = grid.find_path(start, goal, MovementType::Walk) {
//!     println!("Path length: {}", path.len());
//! }
//! ```

pub mod coord;
pub mod grid;
pub mod pathfinding;
pub mod line_of_sight;
```

---

## Build Configuration

### Workspace Cargo.toml

```toml
[workspace]
members = [
    "crates/cc-engine",
    "crates/cc-tactics",
    "crates/cc-content",
    "crates/cc-editor",
    "crates/cc-game",
]

resolver = "2"

[workspace.package]
version = "0.1.0"
authors = ["Your Name <you@example.com>"]
edition = "2021"
license = "MIT OR Apache-2.0"

[workspace.dependencies]
# Shared dependencies with locked versions
serde = { version = "1.0", features = ["derive"] }
anyhow = "1.0"
thiserror = "1.0"
tracing = "0.1"

# Rendering
wgpu = "0.19"
winit = "0.29"

# Math
glam = "0.27"

# Async
tokio = { version = "1.0", features = ["full"] }

[profile.dev]
opt-level = 1  # Some optimizations for debug builds

[profile.release]
lto = true
codegen-units = 1
strip = true
```

### Individual Crate Cargo.toml

```toml
# crates/cc-tactics/Cargo.toml
[package]
name = "cc-tactics"
version.workspace = true
authors.workspace = true
edition.workspace = true
license.workspace = true

[dependencies]
cc-engine = { path = "../cc-engine" }

# Workspace dependencies
serde.workspace = true
anyhow.workspace = true
glam.workspace = true

# Crate-specific
bevy_ecs = "0.13"
```

---

## Import Conventions

```rust
// Standard library
use std::collections::HashMap;
use std::path::{Path, PathBuf};

// External crates (alphabetical)
use anyhow::Result;
use serde::{Deserialize, Serialize};
use tracing::info;

// Internal crates (workspace)
use cc_engine::{AssetManager, EventBus};
use cc_tactics::HexGrid;

// Current crate (use crate::)
use crate::combat::TurnManager;
use crate::hex::{HexCoord, HexPosition};

// Re-exports for convenience
pub use cc_engine::render::{Camera, Sprite};
```

---

## Code Organization Best Practices

1. **Keep modules focused**: Each module should have a single, clear purpose
2. **Limit public API surface**: Export only what's needed, keep internals private
3. **Use re-exports**: Create clean API at crate root
4. **Separate data from logic**: Definitions in one module, systems in another
5. **Group related functionality**: Combat-related code together, rendering together, etc.
6. **Avoid circular dependencies**: Ensure clean dependency tree
7. **Document public APIs**: All public items should have doc comments
8. **Write tests alongside code**: Unit tests in same file, integration tests separate
