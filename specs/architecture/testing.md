# Testing Strategy

## Testing Philosophy

**Goals**:
1. Catch bugs early (before they reach players)
2. Enable confident refactoring
3. Document expected behavior
4. Facilitate rapid iteration

**Principles**:
- Test behavior, not implementation
- Keep tests fast and deterministic
- Test at the appropriate level (unit vs integration)
- Automate everything possible

## Test Pyramid

```
        ╱╲
       ╱  ╲
      ╱ E2E╲         Few, slow, expensive
     ╱──────╲
    ╱        ╲
   ╱ Integration╲   Medium number, medium speed
  ╱──────────────╲
 ╱                ╲
╱   Unit Tests     ╲  Many, fast, cheap
╱────────────────────╲
```

**Distribution**:
- 70% Unit tests (individual functions/modules)
- 25% Integration tests (systems working together)
- 5% End-to-end tests (full game scenarios)

## Unit Testing

### What to Unit Test

✅ **Good candidates**:
- Pure functions (deterministic, no side effects)
- Math/calculation logic
- Hex coordinate conversion
- Damage calculation
- Pathfinding algorithms
- Action validation
- Data serialization/deserialization

❌ **Poor candidates**:
- Rendering (visual testing better suited)
- Audio playback
- File I/O (integration tests better)
- Timing-dependent code

### Example: Hex Coordinate Math

```rust
// cc-tactics/src/hex/coord.rs

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct HexCoord {
    pub q: i32,
    pub r: i32,
    pub s: i32,
}

impl HexCoord {
    pub fn new(q: i32, r: i32) -> Self {
        Self { q, r, s: -q - r }
    }
    
    pub fn distance(&self, other: &HexCoord) -> i32 {
        ((self.q - other.q).abs() + (self.r - other.r).abs() + (self.s - other.s).abs()) / 2
    }
    
    pub fn neighbor(&self, direction: usize) -> HexCoord {
        const DIRECTIONS: [(i32, i32, i32); 6] = [
            (1, 0, -1), (1, -1, 0), (0, -1, 1),
            (-1, 0, 1), (-1, 1, 0), (0, 1, -1),
        ];
        
        let (dq, dr, ds) = DIRECTIONS[direction];
        HexCoord {
            q: self.q + dq,
            r: self.r + dr,
            s: self.s + ds,
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_hex_creation() {
        let hex = HexCoord::new(1, 2);
        assert_eq!(hex.q, 1);
        assert_eq!(hex.r, 2);
        assert_eq!(hex.s, -3);
    }
    
    #[test]
    fn test_hex_distance() {
        let a = HexCoord::new(0, 0);
        let b = HexCoord::new(3, 0);
        assert_eq!(a.distance(&b), 3);
        
        let c = HexCoord::new(2, 2);
        assert_eq!(a.distance(&c), 4);
    }
    
    #[test]
    fn test_hex_neighbors() {
        let center = HexCoord::new(0, 0);
        
        // Test all 6 neighbors
        let neighbors = (0..6).map(|dir| center.neighbor(dir)).collect::<Vec<_>>();
        
        assert_eq!(neighbors[0], HexCoord::new(1, 0));   // E
        assert_eq!(neighbors[1], HexCoord::new(1, -1));  // NE
        assert_eq!(neighbors[2], HexCoord::new(0, -1));  // NW
        assert_eq!(neighbors[3], HexCoord::new(-1, 0));  // W
        assert_eq!(neighbors[4], HexCoord::new(-1, 1));  // SW
        assert_eq!(neighbors[5], HexCoord::new(0, 1));   // SE
    }
    
    #[test]
    fn test_neighbor_distance_is_one() {
        let center = HexCoord::new(5, 5);
        
        for dir in 0..6 {
            let neighbor = center.neighbor(dir);
            assert_eq!(center.distance(&neighbor), 1);
        }
    }
}
```

### Example: Combat Damage Calculation

```rust
// cc-tactics/src/combat/damage.rs

pub fn calculate_damage(
    base_damage: DiceRoll,
    attacker_stats: &UnitStats,
    target_stats: &UnitStats,
    elevation_diff: i32,
) -> i32 {
    let base = base_damage.roll();
    let damage_mod = attacker_stats.damage_bonus;
    
    // Elevation bonus
    let elevation_bonus = match elevation_diff {
        d if d >= 2 => 2,
        1 => 1,
        _ => 0,
    };
    
    let total_damage = base + damage_mod + elevation_bonus;
    
    // Apply armor reduction
    let after_armor = (total_damage - target_stats.armor).max(1);
    
    // Apply resistances
    apply_resistances(after_armor, target_stats.resistances)
}

#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_basic_damage_calculation() {
        let damage_roll = DiceRoll::Fixed(10);  // No randomness for testing
        
        let attacker = UnitStats {
            damage_bonus: 3,
            ..Default::default()
        };
        
        let target = UnitStats {
            armor: 5,
            resistances: HashMap::new(),
            ..Default::default()
        };
        
        let damage = calculate_damage(damage_roll, &attacker, &target, 0);
        
        // 10 (base) + 3 (bonus) - 5 (armor) = 8
        assert_eq!(damage, 8);
    }
    
    #[test]
    fn test_elevation_damage_bonus() {
        let damage_roll = DiceRoll::Fixed(10);
        let attacker = UnitStats::default();
        let target = UnitStats::default();
        
        // No elevation advantage
        let damage_even = calculate_damage(damage_roll, &attacker, &target, 0);
        
        // 1 level advantage
        let damage_high = calculate_damage(damage_roll, &attacker, &target, 1);
        assert_eq!(damage_high, damage_even + 1);
        
        // 2+ levels advantage
        let damage_very_high = calculate_damage(damage_roll, &attacker, &target, 2);
        assert_eq!(damage_very_high, damage_even + 2);
    }
    
    #[test]
    fn test_minimum_damage_is_one() {
        let damage_roll = DiceRoll::Fixed(1);
        
        let attacker = UnitStats::default();
        
        let heavily_armored = UnitStats {
            armor: 100,
            ..Default::default()
        };
        
        let damage = calculate_damage(damage_roll, &attacker, &heavily_armored, 0);
        
        // Should never deal 0 damage
        assert_eq!(damage, 1);
    }
}
```

### Parameterized Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use rstest::rstest;
    
    #[rstest]
    #[case(0, 0, 0, 0)]   // Same position
    #[case(1, 0, -1, 1)]  // 1 hex away
    #[case(3, 0, -3, 3)]  // 3 hexes away
    #[case(2, 2, -4, 4)]  // 4 hexes away
    fn test_hex_distance_cases(
        #[case] q1: i32,
        #[case] r1: i32,
        #[case] q2: i32,
        #[case] expected: i32,
    ) {
        let a = HexCoord::new(0, 0);
        let b = HexCoord::new(q1, r1);
        assert_eq!(a.distance(&b), expected);
    }
}
```

---

## Integration Testing

### What to Integration Test

✅ **Good candidates**:
- System interactions (e.g., combat + animation)
- Event flow (action → multiple systems react)
- Content loading pipeline
- Save/load round-trip
- Pathfinding on complex maps

### Example: Turn-Based Combat Flow

```rust
// tests/combat_integration.rs
use cc_tactics::*;
use cc_content::*;

fn setup_test_scenario() -> GameState {
    let mut state = GameState::new();
    
    // Create test map
    let mut grid = HexGrid::new(10, 10);
    for q in 0..10 {
        for r in 0..10 {
            grid.set_tile(HexCoord::new(q, r), HexTile {
                terrain: TerrainType::Grass,
                elevation: 0,
                ..Default::default()
            });
        }
    }
    state.hex_grid = grid;
    
    // Spawn test units
    let player = state.spawn_unit("warrior", HexPosition::new(2, 2, 0), Team::Player);
    let enemy = state.spawn_unit("goblin", HexPosition::new(7, 7, 0), Team::Enemy);
    
    state.turn_manager.initialize(vec![player, enemy]);
    
    state
}

#[test]
fn test_full_turn_execution() {
    let mut state = setup_test_scenario();
    
    // Player's turn
    assert_eq!(state.turn_manager.current_team(), Team::Player);
    
    let player_id = state.turn_manager.current_unit();
    let player_pos = state.get_entity(player_id).unwrap().position;
    
    // Move player
    let move_target = player_pos.hex.neighbor(0); // Move east
    let move_action = GameAction::Move {
        unit: player_id,
        path: vec![player_pos, HexPosition::new(move_target.q, move_target.r, 0)],
    };
    
    state.execute_action(move_action).expect("Move failed");
    
    // Verify movement
    let new_pos = state.get_entity(player_id).unwrap().position;
    assert_eq!(new_pos.hex, move_target);
    
    // End turn
    state.end_turn();
    
    // Enemy's turn now
    assert_eq!(state.turn_manager.current_team(), Team::Enemy);
}

#[test]
fn test_attack_and_damage() {
    let mut state = setup_test_scenario();
    
    let attacker = state.turn_manager.current_unit();
    let target = state.entities.iter()
        .find(|(_, e)| e.team == Team::Enemy)
        .map(|(id, _)| *id)
        .unwrap();
    
    // Move attacker adjacent to target
    let target_pos = state.get_entity(target).unwrap().position;
    let adjacent = target_pos.hex.neighbor(0);
    state.get_entity_mut(attacker).unwrap().position = HexPosition::new(adjacent.q, adjacent.r, 0);
    
    let target_hp_before = state.get_entity(target).unwrap().stats.hp;
    
    // Execute attack
    let attack_action = GameAction::Attack { attacker, target };
    let result = state.execute_action(attack_action).expect("Attack failed");
    
    // Verify damage dealt
    let target_hp_after = state.get_entity(target).unwrap().stats.hp;
    assert!(target_hp_after < target_hp_before, "No damage dealt");
}

#[test]
fn test_event_propagation() {
    let mut state = setup_test_scenario();
    
    let mut event_log = Vec::new();
    
    // Subscribe to events
    state.event_bus.subscribe(|event: &GameEvent| {
        event_log.push(event.clone());
    });
    
    let attacker = state.turn_manager.current_unit();
    let target = state.entities.iter()
        .find(|(_, e)| e.team == Team::Enemy)
        .map(|(id, _)| *id)
        .unwrap();
    
    // Position for attack
    let target_pos = state.get_entity(target).unwrap().position;
    let adjacent = target_pos.hex.neighbor(0);
    state.get_entity_mut(attacker).unwrap().position = HexPosition::new(adjacent.q, adjacent.r, 0);
    
    // Attack
    state.execute_action(GameAction::Attack { attacker, target }).unwrap();
    
    // Process events
    state.event_bus.process_events();
    
    // Verify events fired
    assert!(event_log.iter().any(|e| matches!(e, GameEvent::AttackInitiated { .. })));
    assert!(event_log.iter().any(|e| matches!(e, GameEvent::DamageTaken { .. })));
}
```

### Example: Content Loading

```rust
// tests/content_loading.rs
use cc_content::*;
use std::path::PathBuf;

#[test]
fn test_load_unit_definition() {
    let content_dir = PathBuf::from("test_data/content");
    
    let mut registry = ContentRegistry::new();
    registry.load_directory(&content_dir).expect("Failed to load content");
    
    // Verify unit loaded
    let warrior = registry.get_unit("warrior").expect("Warrior not found");
    
    assert_eq!(warrior.name, "Warrior");
    assert_eq!(warrior.max_hp, 30);
    assert!(warrior.abilities.contains(&"power_attack".to_string()));
}

#[test]
fn test_content_validation() {
    let invalid_toml = r#"
        [unit]
        name = "Invalid Unit"
        # Missing required fields: max_hp, sprite, etc.
    "#;
    
    let result: Result<UnitDefinition, _> = toml::from_str(invalid_toml);
    
    assert!(result.is_err(), "Should reject invalid content");
}

#[test]
fn test_hot_reload() {
    use std::fs;
    use std::thread;
    use std::time::Duration;
    
    let temp_dir = tempfile::tempdir().unwrap();
    let unit_file = temp_dir.path().join("warrior.toml");
    
    // Initial content
    fs::write(&unit_file, r#"
        [unit]
        id = "warrior"
        name = "Warrior"
        max_hp = 30
    "#).unwrap();
    
    let mut registry = ContentRegistry::new();
    registry.enable_hot_reload();
    registry.load_file(&unit_file).unwrap();
    
    let warrior = registry.get_unit("warrior").unwrap();
    assert_eq!(warrior.max_hp, 30);
    
    // Modify content
    fs::write(&unit_file, r#"
        [unit]
        id = "warrior"
        name = "Warrior"
        max_hp = 35
    "#).unwrap();
    
    // Wait for file watcher
    thread::sleep(Duration::from_millis(100));
    registry.process_file_changes();
    
    // Verify reload
    let warrior = registry.get_unit("warrior").unwrap();
    assert_eq!(warrior.max_hp, 35);
}
```

---

## Property-Based Testing

Use `proptest` for testing properties that should hold for any input:

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn test_hex_distance_symmetric(q1 in -100i32..100, r1 in -100i32..100, q2 in -100i32..100, r2 in -100i32..100) {
        let a = HexCoord::new(q1, r1);
        let b = HexCoord::new(q2, r2);
        
        // Distance should be symmetric
        prop_assert_eq!(a.distance(&b), b.distance(&a));
    }
    
    #[test]
    fn test_hex_distance_triangle_inequality(
        q1 in -100i32..100, r1 in -100i32..100,
        q2 in -100i32..100, r2 in -100i32..100,
        q3 in -100i32..100, r3 in -100i32..100,
    ) {
        let a = HexCoord::new(q1, r1);
        let b = HexCoord::new(q2, r2);
        let c = HexCoord::new(q3, r3);
        
        // Triangle inequality: dist(a, c) <= dist(a, b) + dist(b, c)
        prop_assert!(a.distance(&c) <= a.distance(&b) + b.distance(&c));
    }
    
    #[test]
    fn test_damage_never_negative(
        base in 1i32..100,
        bonus in -10i32..10,
        armor in 0i32..50,
    ) {
        let damage_roll = DiceRoll::Fixed(base);
        let attacker = UnitStats { damage_bonus: bonus, ..Default::default() };
        let target = UnitStats { armor, ..Default::default() };
        
        let damage = calculate_damage(damage_roll, &attacker, &target, 0);
        
        // Damage should always be at least 1
        prop_assert!(damage >= 1);
    }
}
```

---

## Performance Testing

### Benchmarking

Use `criterion` for performance regression testing:

```rust
// benches/pathfinding_bench.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};
use cc_tactics::hex::{HexGrid, HexPosition, MovementType};

fn pathfinding_benchmark(c: &mut Criterion) {
    let grid = create_test_grid(50, 50);
    
    c.bench_function("pathfinding_short_distance", |b| {
        b.iter(|| {
            grid.find_path(
                black_box(HexPosition::new(0, 0, 0)),
                black_box(HexPosition::new(5, 5, 0)),
                MovementType::Walk,
            )
        });
    });
    
    c.bench_function("pathfinding_long_distance", |b| {
        b.iter(|| {
            grid.find_path(
                black_box(HexPosition::new(0, 0, 0)),
                black_box(HexPosition::new(45, 45, 0)),
                MovementType::Walk,
            )
        });
    });
    
    c.bench_function("pathfinding_with_obstacles", |b| {
        b.iter(|| {
            grid.find_path(
                black_box(HexPosition::new(0, 0, 0)),
                black_box(HexPosition::new(25, 25, 0)),
                MovementType::Walk,
            )
        });
    });
}

criterion_group!(benches, pathfinding_benchmark);
criterion_main!(benches);
```

**Cargo.toml**:
```toml
[[bench]]
name = "pathfinding_bench"
harness = false
```

**Run benchmarks**:
```bash
cargo bench
```

---

## Fuzzing

Use `cargo-fuzz` for finding edge cases:

```rust
// fuzz/fuzz_targets/pathfinding.rs
#![no_main]
use libfuzzer_sys::fuzz_target;
use cc_tactics::hex::{HexGrid, HexPosition, MovementType};

fuzz_target!(|data: &[u8]| {
    if data.len() < 12 {
        return;
    }
    
    let start = HexPosition::new(
        data[0] as i32 % 50,
        data[1] as i32 % 50,
        data[2] as i32 % 5,
    );
    
    let goal = HexPosition::new(
        data[3] as i32 % 50,
        data[4] as i32 % 50,
        data[5] as i32 % 5,
    );
    
    let grid = HexGrid::new(50, 50);
    
    // Should not panic for any input
    let _ = grid.find_path(start, goal, MovementType::Walk);
});
```

**Run fuzzing**:
```bash
cargo fuzz run pathfinding
```

---

## Visual Testing

For rendering and UI, use snapshot testing:

```rust
// tests/visual_tests.rs
#[cfg(feature = "visual-testing")]
mod tests {
    use cc_engine::render::*;
    
    #[test]
    fn test_render_hex_grid() {
        let renderer = TestRenderer::new(800, 600);
        let grid = create_test_grid();
        
        renderer.render_grid(&grid);
        
        let screenshot = renderer.capture_screenshot();
        
        // Compare against golden image
        insta::assert_snapshot!("hex_grid", screenshot);
    }
}
```

Use `insta` crate for snapshot testing.

---

## Test Utilities

### Test Data Builders

```rust
// cc-tactics/src/test_utils.rs (only compiled for tests)

#[cfg(test)]
pub mod builders {
    use super::*;
    
    pub struct UnitBuilder {
        unit: Unit,
    }
    
    impl UnitBuilder {
        pub fn new() -> Self {
            Self {
                unit: Unit::default(),
            }
        }
        
        pub fn with_hp(mut self, hp: i32) -> Self {
            self.unit.stats.hp = hp;
            self.unit.stats.max_hp = hp;
            self
        }
        
        pub fn with_position(mut self, pos: HexPosition) -> Self {
            self.unit.position = pos;
            self
        }
        
        pub fn with_team(mut self, team: Team) -> Self {
            self.unit.team = team;
            self
        }
        
        pub fn build(self) -> Unit {
            self.unit
        }
    }
    
    pub struct GameStateBuilder {
        state: GameState,
    }
    
    impl GameStateBuilder {
        pub fn new() -> Self {
            Self {
                state: GameState::new(),
            }
        }
        
        pub fn with_map_size(mut self, width: i32, height: i32) -> Self {
            self.state.hex_grid = HexGrid::new(width, height);
            self
        }
        
        pub fn add_unit(mut self, unit: Unit) -> Self {
            self.state.add_entity(unit);
            self
        }
        
        pub fn build(self) -> GameState {
            self.state
        }
    }
}

// Usage in tests:
#[test]
fn test_with_builder() {
    use test_utils::builders::*;
    
    let player = UnitBuilder::new()
        .with_hp(30)
        .with_position(HexPosition::new(0, 0, 0))
        .with_team(Team::Player)
        .build();
    
    let state = GameStateBuilder::new()
        .with_map_size(20, 20)
        .add_unit(player)
        .build();
    
    // Test using state...
}
```

---

## Continuous Integration

### GitHub Actions Example

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:
  CARGO_TERM_COLOR: always

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Install Rust
      uses: actions-rs/toolchain@v1
      with:
        toolchain: stable
        profile: minimal
        components: rustfmt, clippy
        override: true
    
    - name: Cache cargo
      uses: actions/cache@v3
      with:
        path: |
          ~/.cargo/bin/
          ~/.cargo/registry/index/
          ~/.cargo/registry/cache/
          ~/.cargo/git/db/
          target/
        key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}
    
    - name: Check formatting
      run: cargo fmt --all -- --check
    
    - name: Clippy
      run: cargo clippy --all-targets --all-features -- -D warnings
    
    - name: Run tests
      run: cargo test --all-features --workspace
    
    - name: Run benchmarks (check only)
      run: cargo bench --no-run --all-features --workspace
    
    - name: Build release
      run: cargo build --release --all-features --workspace
```

---

## Test Coverage

Use `tarpaulin` for code coverage:

```bash
cargo install cargo-tarpaulin

cargo tarpaulin --out Html --output-dir coverage/
```

**Goal**: Aim for 70%+ coverage on critical paths.

---

## Testing Checklist

Before releasing:

- [ ] All unit tests pass
- [ ] All integration tests pass
- [ ] No clippy warnings
- [ ] Code formatted (`cargo fmt`)
- [ ] Documentation builds (`cargo doc`)
- [ ] Benchmarks run without regression
- [ ] Manual playtesting of new features
- [ ] Performance profiling on representative scenarios
- [ ] Save/load round-trip tested
- [ ] Hot-reload tested
- [ ] Content validation catches errors

---

## Test-Driven Development (TDD)

Recommended workflow:

1. **Write failing test** - Define expected behavior
2. **Make it pass** - Implement minimal code
3. **Refactor** - Clean up while keeping tests green
4. **Repeat**

Example:
```rust
// 1. Write test first
#[test]
fn test_pathfinding_avoids_obstacles() {
    let mut grid = HexGrid::new(10, 10);
    
    // Place obstacle
    grid.set_tile(HexCoord::new(5, 5), HexTile {
        blocking: true,
        ..Default::default()
    });
    
    let path = grid.find_path(
        HexPosition::new(0, 0, 0),
        HexPosition::new(10, 10, 0),
        MovementType::Walk,
    );
    
    assert!(path.is_some());
    
    // Path should not go through (5, 5)
    assert!(!path.unwrap().iter().any(|p| p.hex == HexCoord::new(5, 5)));
}

// 2. Implement to make test pass

// 3. Refactor and optimize

// 4. Move to next feature
```
