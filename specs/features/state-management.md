# State Management & Game Flow

## Overview

State machine controlling game flow from menus through combat to results. Clear separation between game states for maintainability.

## Game State Architecture

```rust
enum GameState {
    Boot,
    MainMenu,
    CharacterSelect,
    MapSelect,
    Loading,
    InGame(GameplayState),
    Paused,
    Results,
    Credits,
}

enum GameplayState {
    Setup,           // Placing units before combat
    PlayerTurn(PlayerId),
    EnemyTurn,
    Animation,       // Waiting for animations to complete
    Victory,
    Defeat,
}
```

### State Machine

```rust
struct StateMachine {
    current: GameState,
    previous: Option<GameState>,
    pending_transition: Option<GameState>,
}

impl StateMachine {
    fn transition(&mut self, new_state: GameState) {
        self.on_exit(self.current);
        self.previous = Some(self.current);
        self.current = new_state;
        self.on_enter(self.current);
    }
    
    fn on_enter(&mut self, state: GameState) {
        match state {
            GameState::InGame(_) => self.load_game_data(),
            GameState::MainMenu => self.load_menu_ui(),
            // ...
        }
    }
    
    fn on_exit(&mut self, state: GameState) {
        match state {
            GameState::InGame(_) => self.cleanup_game(),
            // ...
        }
    }
}
```

## State Lifecycle

### Boot → MainMenu

```
Boot
  ├─ Load engine config
  ├─ Initialize systems (renderer, audio, input)
  ├─ Load core assets (fonts, UI sprites)
  └─ Transition → MainMenu
```

### MainMenu → InGame

```
MainMenu
  ├─ Player selects "New Game"
  └─ Transition → CharacterSelect

CharacterSelect
  ├─ Players choose units
  └─ Transition → MapSelect

MapSelect
  ├─ Players choose scenario
  └─ Transition → Loading

Loading
  ├─ Load map data
  ├─ Load unit assets
  ├─ Initialize combat state
  └─ Transition → InGame(Setup)

InGame(Setup)
  ├─ Players place units
  ├─ All players ready
  └─ Transition → InGame(PlayerTurn(0))
```

### Combat Turn Loop

```
InGame(PlayerTurn(0))
  ├─ Player 0 acts (move, attack, etc.)
  ├─ Confirms end turn
  └─ Transition → InGame(Animation)

InGame(Animation)
  ├─ Play all queued animations
  ├─ Animations complete
  └─ Check victory conditions
     ├─ If won → InGame(Victory)
     ├─ If lost → InGame(Defeat)
     └─ Else → InGame(PlayerTurn(next_player))

InGame(Victory) or InGame(Defeat)
  ├─ Show results
  ├─ Player confirms
  └─ Transition → Results

Results
  ├─ Display stats, rewards
  ├─ Player continues
  └─ Transition → MainMenu or next scenario
```

## Turn Management

```rust
struct TurnManager {
    turn_order: Vec<EntityId>,
    current_turn_index: usize,
    round_number: u32,
    turn_time_limit: Option<f32>,  // Optional timer
}

impl TurnManager {
    fn next_turn(&mut self) -> EntityId {
        self.current_turn_index += 1;
        
        if self.current_turn_index >= self.turn_order.len() {
            self.current_turn_index = 0;
            self.round_number += 1;
        }
        
        self.turn_order[self.current_turn_index]
    }
    
    fn current_unit(&self) -> EntityId {
        self.turn_order[self.current_turn_index]
    }
    
    fn is_player_turn(&self, player_id: PlayerId) -> bool {
        let unit = self.current_unit();
        self.get_owner(unit) == player_id
    }
}
```

### Initiative System

Determine turn order at combat start:

```rust
fn calculate_initiative(units: &[Entity]) -> Vec<EntityId> {
    let mut initiatives: Vec<_> = units.iter()
        .map(|unit| {
            let roll = rand::thread_rng().gen_range(1..=20);
            let total = unit.initiative_bonus + roll;
            (unit.id, total)
        })
        .collect();
    
    // Sort descending (highest initiative first)
    initiatives.sort_by(|a, b| b.1.cmp(&a.1));
    
    initiatives.into_iter().map(|(id, _)| id).collect()
}
```

## Action Queue

Store player actions before execution:

```rust
struct ActionQueue {
    pending: VecDeque<GameAction>,
    executing: Option<GameAction>,
}

enum GameAction {
    Move { unit: EntityId, path: Vec<HexCoord> },
    Attack { attacker: EntityId, target: EntityId },
    UseAbility { caster: EntityId, ability: AbilityId, targets: Vec<EntityId> },
    EndTurn,
    Undo,  // Revert last action
}

impl ActionQueue {
    fn push(&mut self, action: GameAction) {
        self.pending.push_back(action);
    }
    
    fn execute_next(&mut self) -> Option<GameAction> {
        if self.executing.is_some() {
            return None;  // Still executing previous action
        }
        
        self.executing = self.pending.pop_front();
        self.executing.clone()
    }
    
    fn on_action_complete(&mut self) {
        self.executing = None;
    }
}
```

### Undo System

Allow players to undo moves during their turn:

```rust
struct UndoStack {
    snapshots: Vec<GameSnapshot>,
    max_depth: usize,  // e.g., 5 undos
}

struct GameSnapshot {
    unit_positions: HashMap<EntityId, HexCoord>,
    unit_states: HashMap<EntityId, UnitState>,
    action_points_spent: i32,
}

impl UndoStack {
    fn save_snapshot(&mut self, state: &GameState) {
        let snapshot = GameSnapshot::from_state(state);
        self.snapshots.push(snapshot);
        
        if self.snapshots.len() > self.max_depth {
            self.snapshots.remove(0);
        }
    }
    
    fn undo(&mut self, state: &mut GameState) -> bool {
        if let Some(snapshot) = self.snapshots.pop() {
            snapshot.restore(state);
            true
        } else {
            false
        }
    }
    
    fn clear(&mut self) {
        self.snapshots.clear();
    }
}
```

**Rules**:
- Clear undo stack when turn ends
- Can't undo attacks (once damage is dealt)
- Can undo movement before confirming

## Pause System

```rust
struct PauseManager {
    is_paused: bool,
    pause_menu: Option<MenuHandle>,
}

impl PauseManager {
    fn toggle_pause(&mut self) {
        self.is_paused = !self.is_paused;
        
        if self.is_paused {
            self.pause_menu = Some(create_pause_menu());
            audio.pause_music();
        } else {
            self.pause_menu = None;
            audio.resume_music();
        }
    }
}
```

**Pause menu options**:
- Resume
- Settings
- Save game
- Load game
- Quit to menu
- Quit to desktop

## Victory/Defeat Conditions

Flexible condition system:

```rust
struct VictoryCondition {
    check: Box<dyn Fn(&GameState) -> bool>,
    description: String,
}

struct DefeatCondition {
    check: Box<dyn Fn(&GameState) -> bool>,
    description: String,
}

// Example conditions
fn all_enemies_defeated(state: &GameState) -> bool {
    state.entities.iter()
        .filter(|e| e.team == Team::Enemy)
        .all(|e| e.is_dead())
}

fn all_players_defeated(state: &GameState) -> bool {
    state.entities.iter()
        .filter(|e| e.team == Team::Player)
        .all(|e| e.is_dead())
}

fn reached_objective(state: &GameState) -> bool {
    let hero = state.get_entity_by_tag("hero");
    hero.position == state.objective_location
}
```

**Check every turn**:
```rust
fn check_end_conditions(&self) -> Option<EndState> {
    for condition in &self.victory_conditions {
        if (condition.check)(&self.game_state) {
            return Some(EndState::Victory);
        }
    }
    
    for condition in &self.defeat_conditions {
        if (condition.check)(&self.game_state) {
            return Some(EndState::Defeat);
        }
    }
    
    None
}
```

## State Persistence

Track session info:

```rust
struct SessionState {
    current_scenario: ScenarioId,
    completed_scenarios: Vec<ScenarioId>,
    player_roster: Vec<UnitDefinition>,  // Persistent units across scenarios
    campaign_progress: CampaignProgress,
}

struct CampaignProgress {
    current_chapter: u32,
    unlocked_scenarios: HashSet<ScenarioId>,
    story_flags: HashMap<String, bool>,
}
```

## Debug State Control

Dev tools for testing:

```rust
struct DebugStateController {
    force_victory: bool,
    force_defeat: bool,
    skip_animations: bool,
    instant_turns: bool,  // Skip AI thinking time
}

impl DebugStateController {
    fn force_transition(&mut self, state: GameState) {
        // Bypass normal state validation
        self.state_machine.current = state;
    }
    
    fn set_turn_index(&mut self, index: usize) {
        self.turn_manager.current_turn_index = index;
    }
}
```

## Performance

State updates are synchronous, so keep them fast:
- Condition checks: O(n) or better
- State transitions: <1ms
- Action validation: <5ms

**Optimization**:
- Cache frequently-checked values (unit count, team status)
- Update caches incrementally (when units die, not every frame)
- Lazy evaluation of expensive conditions

## Example State Flow Diagram

```
┌─────────┐
│  Boot   │
└────┬────┘
     │
     v
┌─────────┐     ┌──────────────┐
│MainMenu │────>│CharacterSelect│
└────┬────┘     └──────┬───────┘
     │                 │
     │<────────────────┘
     │
     v
┌─────────┐     ┌─────────┐
│MapSelect│────>│ Loading │
└─────────┘     └────┬────┘
                     │
                     v
              ┌──────────────┐
              │ InGame(Setup)│
              └──────┬───────┘
                     │
                     v
            ┌────────────────────┐
            │InGame(PlayerTurn)  │<───┐
            └──────┬─────────────┘    │
                   │                  │
                   v                  │
            ┌──────────────┐          │
            │InGame(Animation)│       │
            └──────┬─────────┘        │
                   │                  │
          ┌────────┴────────┐         │
          │                 │         │
          v                 v         │
    ┌─────────┐      ┌─────────┐     │
    │ Victory │      │ Defeat  │     │
    └────┬────┘      └────┬────┘     │
         │                │          │
         └────────┬───────┘          │
                  │                  │
                  v                  │
            ┌─────────┐              │
            │ Results │              │
            └────┬────┘              │
                 │                   │
                 v                   │
            ┌─────────┐              │
            │MainMenu │──────────────┘
            └─────────┘
```
