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
    can_climb: bool,         // Can move up cliffs
    can_fly: bool,           // Ignores elevation costs
}
```

**Rules**:
- Difficult terrain costs 2 movement per hex
- Moving uphill costs extra (see hex-grid.md for elevation movement costs)
- Moving through allied units: costs normal movement
- Moving through enemy units: requires special ability or impossible
- Opportunity attacks: enemies may attack when you leave threatened hexes
- Jumping down: Free movement down 1 level, may take fall damage for 2+
- Flying units: Ignore elevation, can cross cliffs

### Attacks

```rust
struct Attack {
    range: Range,           // Melee(1) or Ranged(min, max)
    hit_bonus: i32,
    damage: DiceRoll,       // e.g., 2d6 + 3
    damage_type: DamageType,
    uses_3d_range: bool,    // If true, accounts for elevation in range calc
}

enum Range {
    Melee(i32),            // adjacent hexes
    Ranged(i32, i32),      // min and max range (horizontal distance)
    Ranged3D(i32, i32),    // 3D distance (accounts for elevation)
    Touch,                  // same hex only
}
```

**Attack resolution**:
1. Check range & line of sight (3D-aware, see hex-grid.md)
2. Calculate elevation bonus/penalty
3. Roll to hit: `1d20 + hit_bonus + elevation_modifier >= target_armor`
4. On success, roll damage (may get bonus damage from high ground)
5. Apply resistances/vulnerabilities
6. Subtract from target HP

### Elevation Combat Modifiers

```rust
fn calculate_elevation_modifier(attacker: HexPosition, target: HexPosition) -> CombatModifier {
    let elevation_diff = attacker.elevation - target.elevation;
    
    CombatModifier {
        to_hit: match elevation_diff {
            d if d >= 2 => 3,    // Major high ground: +3 to hit
            1 => 2,              // Minor high ground: +2 to hit
            0 => 0,              // Even ground: no modifier
            -1 => -1,            // Shooting uphill: -1 to hit
            _ => -2,             // Major low ground: -2 to hit
        },
        damage_bonus: match elevation_diff {
            d if d >= 2 => 2,    // +2 damage from major high ground
            1 => 1,              // +1 damage from minor high ground
            _ => 0,
        },
        crit_chance: if elevation_diff >= 1 { 1 } else { 0 },  // +5% crit from high ground
    }
}

struct CombatModifier {
    to_hit: i32,
    damage_bonus: i32,
    crit_chance: i32,
}
```

**High ground advantages**:
- Easier to hit targets below you (better angle of attack)
- Bonus damage (gravity assists melee, clear shot for ranged)
- Increased critical hit chance
- Better line of sight (see over obstacles)

**Low ground disadvantages**:
- Harder to hit targets above you
- Target may have cover from elevation
- Reduced visibility

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
    SingleEnemy(i32),       // range (horizontal)
    SingleEnemy3D(i32, i32), // (horizontal range, vertical range)
    SingleAlly(i32),
    AreaOfEffect(AoeShape),
    Self,
}

enum AoeShape {
    Circle { center: HexCoord, radius: i32 },
    Circle3D { center: HexPosition, radius: i32, vertical_spread: i32 },
    Cone { origin: HexCoord, direction: Direction, length: i32 },
    Line { start: HexCoord, end: HexCoord, width: i32 },
    Cylinder { center: HexPosition, radius: i32, height: i32 },  // Full vertical column
}
```

**3D Area Effects**:

```rust
fn get_targets_in_3d_aoe(shape: &AoeShape, map: &HexMap) -> Vec<HexPosition> {
    match shape {
        AoeShape::Circle3D { center, radius, vertical_spread } => {
            let mut targets = Vec::new();
            
            for hex in hexes_in_range(center.hex, *radius) {
                for elev in (center.elevation - vertical_spread)..=(center.elevation + vertical_spread) {
                    let pos = HexPosition { hex, elevation: elev };
                    
                    // Check 3D distance
                    if distance_3d(*center, pos) <= *radius as f32 {
                        targets.push(pos);
                    }
                }
            }
            
            targets
        },
        AoeShape::Cylinder { center, radius, height } => {
            // Hit all units in radius from base_elevation to base_elevation + height
            let mut targets = Vec::new();
            
            for hex in hexes_in_range(center.hex, *radius) {
                for elev in center.elevation..=(center.elevation + height) {
                    targets.push(HexPosition { hex, elevation: elev });
                }
            }
            
            targets
        },
        // ... other shapes
    }
}
```

**Examples**:
- **Fireball**: `Circle3D` - Spherical explosion, hits adjacent elevations
- **Lightning Strike**: `Cylinder` - Hits entire vertical column
- **Earthquake**: `Circle` (2D only) - Ground-based effect, single elevation
- **Meteor**: Originates high, impacts ground with splash (elevation-based damage falloff)

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

### Falling Damage

Units that move or are pushed off cliffs take falling damage:

```rust
fn calculate_fall_damage(fall_distance: i32) -> i32 {
    match fall_distance {
        0..=1 => 0,                    // No damage from 1 level
        2 => roll_dice("1d6"),         // Minor fall
        3 => roll_dice("2d6"),         // Moderate fall
        4 => roll_dice("3d6"),         // Serious fall
        d if d >= 5 => roll_dice("4d6") + (d - 5) * 6,  // Deadly fall
        _ => 0,
    }
}

fn apply_knockback_with_fall(
    unit: &mut Unit,
    direction: HexDirection,
    distance: i32,
    map: &HexMap,
) {
    let mut current_pos = unit.position;
    let mut highest_elevation = current_pos.elevation;
    
    for step in 0..distance {
        let next_hex = current_pos.hex.neighbor(direction);
        let next_tile = map.get_tile(next_hex);
        
        // Check if we fall off an edge
        if next_tile.base_elevation < current_pos.elevation {
            // Unit is knocked off a cliff
            let fall_damage = calculate_fall_damage(current_pos.elevation - next_tile.base_elevation);
            unit.take_damage(fall_damage, DamageType::Falling);
            
            // Land at lower elevation
            current_pos = HexPosition {
                hex: next_hex,
                elevation: next_tile.base_elevation,
            };
            break;
        } else {
            current_pos.hex = next_hex;
            current_pos.elevation = next_tile.base_elevation;
            highest_elevation = highest_elevation.max(current_pos.elevation);
        }
    }
    
    unit.position = current_pos;
}
```

**Abilities that cause falling**:
- **Knockback**: Push enemy off cliff
- **Bull Rush**: Charge through enemy, potentially off edge
- **Gravity Well**: Pull all units to center, falling from elevated positions
- **Shockwave**: Radial knockback from caster

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
