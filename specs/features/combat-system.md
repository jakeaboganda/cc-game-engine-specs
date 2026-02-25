# Tactical Combat System

## Initiative & Turn Order

### Initiative System
```rust
struct Initiative {
    base_speed: i32,
    roll_modifier: i32,  // random component
    delays: i32,         // actions that postpone turn
}
```

**Turn calculation**:
1. At combat start, roll initiative: `base_speed + 1d20 + modifier`
2. Sort units by initiative (highest first)
3. Track current turn index

### Turn Structure
Each turn consists of:
- **Move action**: Movement up to speed in hexes
- **Standard action**: Attack, cast spell, use item
- **Bonus action** (optional): Quick action, reaction
- **Free actions**: Speak, drop item, etc.

**Alternative**: Action point system where move/attack/ability each cost points.

## Action Economy

### Movement
```rust
struct Movement {
    speed: i32,              // hexes per turn
    current_spent: i32,      // hexes moved this turn
    terrain_cost: HashMap<TerrainType, i32>,
}
```

**Rules**:
- Difficult terrain costs 2 movement per hex
- Moving through allied units: costs normal movement
- Moving through enemy units: requires special ability or impossible
- Opportunity attacks: enemies may attack when you leave threatened hexes

### Attacks

```rust
struct Attack {
    range: Range,           // Melee(1) or Ranged(min, max)
    hit_bonus: i32,
    damage: DiceRoll,       // e.g., 2d6 + 3
    damage_type: DamageType,
}

enum Range {
    Melee(i32),            // adjacent hexes
    Ranged(i32, i32),      // min and max range
    Touch,                  // same hex only
}
```

**Attack resolution**:
1. Check range & line of sight
2. Roll to hit: `1d20 + hit_bonus >= target_armor`
3. On success, roll damage
4. Apply resistances/vulnerabilities
5. Subtract from target HP

### Abilities

```rust
struct Ability {
    name: String,
    action_cost: ActionType,
    cooldown: i32,          // turns until reusable
    targeting: TargetRule,
    effects: Vec<Effect>,
}

enum TargetRule {
    SingleEnemy(i32),       // range
    SingleAlly(i32),
    AreaOfEffect(AoeShape),
    Self,
}

enum AoeShape {
    Circle { center: HexCoord, radius: i32 },
    Cone { origin: HexCoord, direction: Direction, length: i32 },
    Line { start: HexCoord, end: HexCoord, width: i32 },
}
```

## Status Effects

```rust
struct StatusEffect {
    id: String,
    duration: Duration,
    effects: Vec<StatModifier>,
    on_tick: Option<Effect>,  // triggers each turn
}

enum Duration {
    Turns(i32),
    UntilRemoved,
    EndOfEncounter,
}

enum StatModifier {
    AddToStat { stat: Stat, amount: i32 },
    MultiplyState { stat: Stat, factor: f32 },
    SetStat { stat: Stat, value: i32 },
}
```

**Common effects**:
- Poison: damage over time
- Stunned: skip turn
- Haste: extra movement/action
- Blind: attack penalty, reduced LOS
- Invisible: enemies can't target directly

## Damage & Defense

### Damage Types
- Physical: slashing, piercing, bludgeoning
- Elemental: fire, cold, lightning, acid
- Special: necrotic, radiant, psychic

### Defense
```rust
struct Defense {
    armor_class: i32,       // target number to hit
    resistances: HashMap<DamageType, ResistType>,
    immunities: Vec<DamageType>,
}

enum ResistType {
    Half,      // take 50% damage
    Quarter,   // take 25% damage
}
```

## Combat Events

Event-driven system for:
- `OnAttack` - before attack roll
- `OnHit` - successful hit
- `OnDamage` - before damage applied
- `OnDamageTaken` - after damage applied
- `OnKill` - unit reduced to 0 HP
- `OnTurnStart` / `OnTurnEnd`
- `OnMove` - unit changes position

Abilities and equipment can hook into these events.

## Victory Conditions

Flexible condition system:
```toml
[victory]
type = "eliminate_all"
target_team = "enemies"

# OR
[victory]
type = "reach_location"
location = { q = 10, r = 10 }
required_units = ["hero"]

# OR
[victory]
type = "survive"
duration_turns = 10
```

## AI Behavior (Simplified)

Behavior tree structure:
```
Root: Selector
  - Sequence: Attack
    - Condition: EnemyInRange?
    - Action: MoveToAttackRange
    - Action: ExecuteAttack
  - Sequence: Support Ally
    - Condition: AllyHurt?
    - Action: MoveToAlly
    - Action: Heal
  - Action: MoveTowardsNearestEnemy (fallback)
```

Each unit can have different behavior profiles (aggressive, defensive, support).

## Example Combat Flow

1. **Initiative Phase**: Roll initiative, determine turn order
2. **Turn Loop**:
   - Activate next unit
   - Apply start-of-turn effects (poison damage, etc.)
   - Player/AI chooses actions (move → attack → end turn)
   - Resolve actions (pathfinding, attack rolls, damage)
   - Apply end-of-turn effects
   - Check victory conditions
3. **End Combat**: Award XP, loot, transition to next state
