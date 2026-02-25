# CC Game Engine Specifications

A Rust-based game engine designed for couch co-op tactical strategy games with **2.5D isometric rendering** and hex-based combat similar to tactical RPGs (XCOM, Final Fantasy Tactics, Tactics Ogre).

## Project Goals

- **2.5D Isometric**: Classic tactical RPG perspective with elevation and depth
- **Ease of Use**: Quick prototyping with data-driven content and integrated level editor
- **Couch Co-op First**: Local multiplayer (2-8 players) as the primary use case
- **Tactical Depth**: Hex-based combat with elevation advantage, line of sight, and positioning
- **Hot Reloading**: Rapid iteration without recompilation

## Core Features

### Rendering & Graphics
- **2.5D isometric projection** with depth sorting
- Multi-level terrain with elevation and cliffs
- 8-directional sprite animations
- Shadows, particles, and visual effects
- Camera controls (pan, zoom, optional rotation)
- Split-screen support for local multiplayer

### Tactical Combat
- **Hex-based positioning** with cube coordinates
- **Elevation mechanics**: High ground advantage (+to-hit, +damage, +crit)
- Turn-based combat with initiative system
- Line of sight and cover (elevation-aware)
- Abilities with 3D area effects (spheres, cylinders, cones)
- Status effects and damage-over-time
- Falling damage from cliffs and knockback

### Gameplay Systems
- **3D pathfinding** accounting for elevation and movement costs
- AI with elevation-aware tactics (seek high ground, avoid cliffs)
- Data-driven content (units, abilities, maps in TOML/RON)
- Event system for combat logs, achievements, and modding
- Save/load with autosave and cloud sync support
- Hot-reloadable assets and content

### Development Tools
- **Integrated level editor** with WYSIWYG editing
- Terrain painting and elevation sculpting
- Unit placement and scenario configuration
- Real-time preview and playtesting
- LOS and pathfinding visualization
- Map validation and error checking

## Documentation Structure

```
specs/
├── architecture/
│   └── overview.md              - System architecture and tech stack
├── features/
│   ├── hex-grid.md              - 3D hex coordinates, pathfinding, LOS
│   ├── combat-system.md         - Initiative, attacks, elevation bonuses
│   ├── rendering.md             - Isometric 2.5D rendering pipeline
│   ├── animation.md             - Directional sprites, movement, projectiles
│   ├── input.md                 - Multi-player input, hex selection
│   ├── ai.md                    - Behavior trees, elevation tactics
│   ├── audio.md                 - Music, SFX, spatial audio
│   ├── events.md                - Event bus and pub/sub system
│   ├── state-management.md      - Game flow and turn management
│   ├── save-load.md             - Serialization and persistence
│   ├── asset-pipeline.md        - Loading, hot-reload, packing
│   ├── data-driven-content.md   - TOML-based content system
│   └── level-editor.md          - Integrated map editor
└── api/                         - API designs (planned)
```

## Key Specifications

### Hex Grid System (3D)
- Cube coordinate system with elevation
- 3D pathfinding (A* with elevation costs)
- Elevation-aware line of sight
- Cliffs, slopes, and multi-level terrain
- High ground combat bonuses
- Fall damage and knockback mechanics

### 2.5D Isometric Rendering
- Isometric projection (30° angle, 2:1 ratio)
- Depth-sorted sprite layering
- Elevation and shadow rendering
- Support for 3D models or pre-rendered sprites
- Cliff face rendering between levels
- Camera with pan, zoom, and rotation

### Combat & Tactics
- Turn-based with initiative rolls
- Elevation modifiers (+3 to-hit, +2 damage from major high ground)
- 3D range calculations for abilities
- Area effects with vertical spread (spheres, cylinders)
- Cover and line of sight from elevation
- Environmental hazards and knockback

### Content Pipeline
- TOML/RON data files for units, abilities, maps
- Hot-reload for instant iteration
- Modding support via Lua/Rhai scripting
- Asset streaming and compression
- Version-controlled content (Git-friendly)

### Level Editor
- Terrain painting and elevation sculpting
- Automatic ramp and cliff generation
- Object placement with prefab system
- Unit spawn zones and scenario configuration
- Real-time preview and playtesting
- Validation and error checking
- Export to TOML for version control

## Technology Stack (Proposed)

- **Language**: Rust (performance + safety)
- **ECS**: Bevy ECS or custom implementation
- **Rendering**: wgpu (modern, cross-platform)
- **Serialization**: serde with RON/TOML/JSON
- **Audio**: kira or rodio
- **Pathfinding**: Custom A* with elevation
- **Scripting**: Lua or Rhai for modding

## Performance Targets

- **60 FPS** on mid-range hardware (5-year-old laptop)
- **50x50 hex maps** with multi-level terrain
- **100+ units** on screen with elevation
- **2-8 simultaneous players** (local co-op)
- **Sub-second** load times for scenarios

## Development Status

**Phase**: Specification Complete ✅

All core systems have been specified:
- ✅ Architecture and design principles
- ✅ Hex grid with elevation (3D positioning)
- ✅ Combat system with high ground mechanics
- ✅ Isometric 2.5D rendering pipeline
- ✅ Directional animation system
- ✅ Multi-player input handling
- ✅ Elevation-aware AI tactics
- ✅ Audio and event systems
- ✅ Save/load and asset pipeline
- ✅ Data-driven content framework
- ✅ Integrated level editor

**Next Steps**:
- API design and interface specifications
- Example game scenarios
- Prototype implementation
- Asset creation pipeline
- Modding documentation

## Contributing

This is a specification repository for design and planning. Implementation will follow in a separate repository.

For questions or suggestions, open an issue or discussion.

## License

TBD
