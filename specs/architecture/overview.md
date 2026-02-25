# Architecture Overview

## Layered Design

The engine follows a three-layer architecture to maximize flexibility and prototyping speed:

### 1. Engine Layer (Low-level)
Core systems that are game-agnostic:
- Rendering pipeline (2D focus)
- Input handling & controller abstraction
- Asset loading & management
- Audio system
- Event bus
- Hot-reloading infrastructure

### 2. Tactics Layer (Mid-level)
Tactical game abstractions:
- Hex grid mathematics & coordinate systems
- Pathfinding (A* with movement costs)
- Line of sight calculations
- Turn-based state machine
- Combat resolution system
- AI behavior framework

### 3. Content Layer (High-level)
Game-specific implementations:
- Unit definitions (stats, abilities, classes)
- Ability/skill templates
- Map/scenario data
- Game rules & victory conditions
- UI layouts

## Core Principles

**Separation of Concerns**: Each layer should be usable independently. The engine layer could power other game types; the tactics layer could use a different renderer.

**Data-Driven**: Game content defined in JSON/TOML/RON files, not hardcoded in Rust.

**Hot-Reload Everything**: Changes to assets, stats, maps, or abilities should reflect immediately without restart.

**ECS-Based**: Entity-Component-System architecture (consider Bevy ECS or custom implementation) for flexibility.

## Key Systems

### State Management
- Game state machine (menu → setup → combat → results)
- Turn tracking & initiative system
- Save/load system for mid-game persistence

### Rules Engine
- Declarative rule definitions
- Action validation
- Event-triggered effects
- Status effect management

### Rendering
- Hex grid rendering with coordinate-to-pixel conversion
- Sprite layering (terrain → objects → units → effects → UI)
- Camera system (pan, zoom, maybe per-player viewports)
- Animation system (tweening for movement, attacks, effects)

### Input
- Multi-controller support (2-8 players)
- Turn-based input queueing
- Per-player cursor/selection system
- Unified input abstraction (keyboard, gamepad, mouse)

## Technology Stack (Proposed)

- **Language**: Rust
- **ECS**: Bevy ECS or custom
- **Rendering**: wgpu or Bevy's renderer
- **Serialization**: serde with RON/TOML/JSON
- **Pathfinding**: Custom A* implementation
- **Scripting** (optional): Lua or Rhai for modding

## Performance Targets

For couch co-op tactical games:
- 60 FPS on modest hardware
- Support 4-8 simultaneous players
- Maps up to 50x50 hexes
- 20-30 active units per encounter
