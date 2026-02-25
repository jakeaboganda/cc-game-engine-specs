# Event System

## Overview

Decoupled event bus for game events, enabling loose coupling between systems and supporting features like combat logs, achievements, and mod hooks.

## Architecture

```rust
struct EventBus {
    subscribers: HashMap<TypeId, Vec<Box<dyn EventHandler>>>,
    event_queue: VecDeque<Box<dyn Event>>,
    immediate_mode: bool,  // Process events immediately vs end-of-frame
}

trait Event: Any + Send + Sync {
    fn type_id(&self) -> TypeId;
}

trait EventHandler: Send + Sync {
    fn handle(&mut self, event: &dyn Event);
}
```

## Event Types

### Game Events

```rust
enum GameEvent {
    // State changes
    GameStateChanged { from: GameState, to: GameState },
    TurnStarted { entity: EntityId, turn_number: u32 },
    TurnEnded { entity: EntityId },
    RoundStarted { round_number: u32 },
    
    // Combat
    UnitSpawned { entity: EntityId, position: HexCoord },
    UnitMoved { entity: EntityId, from: HexCoord, to: HexCoord },
    AttackInitiated { attacker: EntityId, target: EntityId },
    AttackHit { attacker: EntityId, target: EntityId, damage: i32 },
    AttackMissed { attacker: EntityId, target: EntityId },
    DamageTaken { entity: EntityId, amount: i32, source: DamageSource },
    HealingReceived { entity: EntityId, amount: i32, source: EntityId },
    UnitDied { entity: EntityId, killer: Option<EntityId> },
    
    // Abilities
    AbilityUsed { caster: EntityId, ability: AbilityId, targets: Vec<EntityId> },
    AbilityCooldownReady { entity: EntityId, ability: AbilityId },
    
    // Status effects
    StatusApplied { entity: EntityId, status: StatusId, duration: i32 },
    StatusRemoved { entity: EntityId, status: StatusId },
    StatusTicked { entity: EntityId, status: StatusId },
    
    // UI
    UnitSelected { entity: EntityId },
    UnitDeselected { entity: EntityId },
    MenuOpened { menu: MenuId },
    MenuClosed { menu: MenuId },
    
    // Victory/Defeat
    VictoryConditionMet { condition: String },
    DefeatConditionMet { condition: String },
}
```

### Example Event Flow

**Player attacks enemy**:
```
1. Player clicks attack button
   → Event: AbilityUsed { caster: player, ability: "basic_attack", targets: [enemy] }

2. Combat system processes attack
   → Event: AttackInitiated { attacker: player, target: enemy }

3. Roll to hit succeeds
   → Event: AttackHit { attacker: player, target: enemy, damage: 15 }

4. Damage applied to enemy
   → Event: DamageTaken { entity: enemy, amount: 15, source: player }

5. Enemy HP reaches 0
   → Event: UnitDied { entity: enemy, killer: Some(player) }

6. Check victory conditions
   → Event: VictoryConditionMet { condition: "all_enemies_defeated" }
```

## Event Subscription

```rust
impl EventBus {
    fn subscribe<E: Event>(&mut self, handler: impl EventHandler + 'static) {
        let type_id = TypeId::of::<E>();
        self.subscribers
            .entry(type_id)
            .or_insert_with(Vec::new)
            .push(Box::new(handler));
    }
    
    fn publish<E: Event>(&mut self, event: E) {
        if self.immediate_mode {
            self.dispatch_event(&event);
        } else {
            self.event_queue.push_back(Box::new(event));
        }
    }
    
    fn process_events(&mut self) {
        while let Some(event) = self.event_queue.pop_front() {
            self.dispatch_event(event.as_ref());
        }
    }
    
    fn dispatch_event(&mut self, event: &dyn Event) {
        let type_id = event.type_id();
        if let Some(handlers) = self.subscribers.get_mut(&type_id) {
            for handler in handlers {
                handler.handle(event);
            }
        }
    }
}
```

## Event Handlers

### Combat Log

```rust
struct CombatLogHandler {
    log: Vec<String>,
}

impl EventHandler for CombatLogHandler {
    fn handle(&mut self, event: &dyn Event) {
        match event.downcast_ref::<GameEvent>() {
            Some(GameEvent::AttackHit { attacker, target, damage }) => {
                let msg = format!(
                    "{} attacks {} for {} damage!",
                    get_name(attacker),
                    get_name(target),
                    damage
                );
                self.log.push(msg);
            },
            Some(GameEvent::UnitDied { entity, killer }) => {
                let msg = if let Some(killer) = killer {
                    format!("{} was slain by {}!", get_name(entity), get_name(killer))
                } else {
                    format!("{} has died.", get_name(entity))
                };
                self.log.push(msg);
            },
            _ => {}
        }
    }
}
```

### Achievement System

```rust
struct AchievementHandler {
    achievements: HashMap<String, Achievement>,
    unlocked: HashSet<String>,
}

struct Achievement {
    id: String,
    name: String,
    description: String,
    condition: Box<dyn Fn(&GameEvent) -> bool>,
}

impl EventHandler for AchievementHandler {
    fn handle(&mut self, event: &dyn Event) {
        for (id, achievement) in &self.achievements {
            if !self.unlocked.contains(id) {
                if let Some(game_event) = event.downcast_ref::<GameEvent>() {
                    if (achievement.condition)(game_event) {
                        self.unlock(id.clone());
                    }
                }
            }
        }
    }
}

// Example achievement
Achievement {
    id: "first_blood",
    name: "First Blood",
    description: "Defeat your first enemy",
    condition: Box::new(|event| {
        matches!(event, GameEvent::UnitDied { killer: Some(_), .. })
    }),
}
```

### Statistics Tracker

```rust
struct StatisticsHandler {
    total_damage_dealt: i32,
    total_healing_done: i32,
    units_killed: i32,
    abilities_used: i32,
    turns_taken: i32,
}

impl EventHandler for StatisticsHandler {
    fn handle(&mut self, event: &dyn Event) {
        match event.downcast_ref::<GameEvent>() {
            Some(GameEvent::DamageTaken { amount, .. }) => {
                self.total_damage_dealt += amount;
            },
            Some(GameEvent::HealingReceived { amount, .. }) => {
                self.total_healing_done += amount;
            },
            Some(GameEvent::UnitDied { .. }) => {
                self.units_killed += 1;
            },
            Some(GameEvent::AbilityUsed { .. }) => {
                self.abilities_used += 1;
            },
            Some(GameEvent::TurnEnded { .. }) => {
                self.turns_taken += 1;
            },
            _ => {}
        }
    }
}
```

### Audio Triggers

```rust
struct AudioEventHandler {
    audio_engine: AudioEngine,
    sound_mappings: HashMap<GameEvent, SoundEffect>,
}

impl EventHandler for AudioEventHandler {
    fn handle(&mut self, event: &dyn Event) {
        if let Some(game_event) = event.downcast_ref::<GameEvent>() {
            if let Some(sound) = self.sound_mappings.get(game_event) {
                self.audio_engine.play_sfx(sound.clone());
            }
        }
    }
}
```

## Event Priority

Control handler execution order:

```rust
struct PrioritizedHandler {
    handler: Box<dyn EventHandler>,
    priority: i32,  // Higher = earlier execution
}

impl EventBus {
    fn subscribe_with_priority<E: Event>(
        &mut self,
        handler: impl EventHandler + 'static,
        priority: i32,
    ) {
        // Insert sorted by priority
    }
}
```

**Use cases**:
- Pre-damage modifiers (armor, resistances) run before damage log
- Animation triggers run after state changes
- UI updates run last

## Event Filtering

Subscribe to filtered events:

```rust
struct FilteredHandler<F>
where
    F: Fn(&dyn Event) -> bool,
{
    handler: Box<dyn EventHandler>,
    filter: F,
}

// Example: Only handle events for a specific entity
let handler = FilteredHandler {
    handler: Box::new(my_handler),
    filter: |event| {
        if let Some(e) = event.downcast_ref::<GameEvent::UnitMoved>() {
            e.entity == player_entity
        } else {
            false
        }
    },
};
```

## Event Replay

Record and replay events for debugging/replays:

```rust
struct EventRecorder {
    recorded_events: Vec<(f32, Box<dyn Event>)>,  // (timestamp, event)
    is_recording: bool,
}

impl EventRecorder {
    fn start_recording(&mut self) {
        self.recorded_events.clear();
        self.is_recording = true;
    }
    
    fn stop_recording(&mut self) {
        self.is_recording = false;
    }
    
    fn replay(&self, event_bus: &mut EventBus, speed: f32) {
        // Re-publish events at recorded timestamps
        for (timestamp, event) in &self.recorded_events {
            // Wait until timestamp * speed
            event_bus.publish(event.clone());
        }
    }
}
```

**Use cases**:
- Replay combat for analysis
- Bug reproduction (save event log with bug reports)
- Sharing cool moments
- AI training data

## Performance

### Batching

Group related events:

```rust
enum BatchedEvent {
    Single(GameEvent),
    Batch(Vec<GameEvent>),
}

// Example: Apply damage to multiple units at once
event_bus.publish(BatchedEvent::Batch(vec![
    GameEvent::DamageTaken { entity: enemy1, amount: 20, source: fireball },
    GameEvent::DamageTaken { entity: enemy2, amount: 20, source: fireball },
    GameEvent::DamageTaken { entity: enemy3, amount: 20, source: fireball },
]));
```

Handlers can process batch more efficiently than individual events.

### Event Pooling

Reuse event objects:

```rust
struct EventPool<E> {
    pool: Vec<E>,
}

impl<E: Event + Default> EventPool<E> {
    fn acquire(&mut self) -> E {
        self.pool.pop().unwrap_or_default()
    }
    
    fn release(&mut self, event: E) {
        self.pool.push(event);
    }
}
```

Reduces allocations for frequently-fired events.

## Debug Tools

```rust
struct EventDebugger {
    log_all_events: bool,
    log_filter: Option<TypeId>,
    event_count: HashMap<TypeId, usize>,
    last_events: VecDeque<(Instant, String)>,  // Recent event log
}

impl EventHandler for EventDebugger {
    fn handle(&mut self, event: &dyn Event) {
        let type_id = event.type_id();
        *self.event_count.entry(type_id).or_insert(0) += 1;
        
        if self.log_all_events || self.log_filter == Some(type_id) {
            println!("[Event] {:?}", event);
            self.last_events.push_back((Instant::now(), format!("{:?}", event)));
        }
    }
}
```

**Debug UI**:
- Event count table (which events fire most)
- Real-time event log
- Filter by event type
- Pause event processing

## Modding Support

Allow mods to hook into events:

```rust
struct ModEventHandler {
    script: LuaScript,
}

impl EventHandler for ModEventHandler {
    fn handle(&mut self, event: &dyn Event) {
        // Call Lua function with event data
        self.script.call_function("on_event", event);
    }
}
```

**Example mod**:
```lua
function on_event(event)
    if event.type == "UnitDied" then
        print("A unit has fallen!")
        
        -- Custom logic: Spawn loot
        if math.random() > 0.5 then
            spawn_item(event.entity.position, "health_potion")
        end
    end
end
```

## Best Practices

1. **Immutable events**: Events should be read-only data
2. **No side effects in constructors**: Events describe what happened, not what to do
3. **Use specific events**: `UnitMoved` better than generic `EntityChanged`
4. **Batch where possible**: Reduce event spam for mass operations
5. **Subscribe early, unsubscribe late**: Avoid missing events or memory leaks
6. **Keep handlers fast**: Slow handlers block all events
7. **Avoid event loops**: Handler emitting events that trigger itself

## Example Integration

```rust
struct Game {
    event_bus: EventBus,
    combat_log: CombatLogHandler,
    achievements: AchievementHandler,
    audio: AudioEventHandler,
}

impl Game {
    fn new() -> Self {
        let mut event_bus = EventBus::new();
        
        // Register handlers
        event_bus.subscribe(CombatLogHandler::new());
        event_bus.subscribe(AchievementHandler::new());
        event_bus.subscribe(AudioEventHandler::new());
        
        Self {
            event_bus,
            // ...
        }
    }
    
    fn update(&mut self, dt: f32) {
        // Process queued events
        self.event_bus.process_events();
        
        // Game logic may publish new events
        // ...
    }
}
```
