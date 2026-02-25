# AI System

## Overview

Behavior system for computer-controlled units in tactical combat. Focus on providing challenging, believable opponents without requiring frame-perfect execution.

## Design Approach

**Turn-based AI** is different from real-time:
- Can take time to "think" (within reason)
- Perfect information vs fog of war changes strategy
- Mistakes make AI more human-like
- Readable behavior is better than optimal play

## AI Architecture

### Behavior Trees

Hierarchical decision-making:

```rust
enum BehaviorNode {
    // Composite nodes
    Sequence(Vec<BehaviorNode>),      // Execute children until one fails
    Selector(Vec<BehaviorNode>),      // Execute children until one succeeds
    Parallel(Vec<BehaviorNode>),      // Execute all children
    
    // Decorator nodes
    Inverter(Box<BehaviorNode>),      // Flip success/failure
    Repeater(Box<BehaviorNode>, u32), // Repeat N times
    UntilFail(Box<BehaviorNode>),     // Repeat until failure
    
    // Leaf nodes (actions & conditions)
    Condition(ConditionFn),
    Action(ActionFn),
}

enum BehaviorResult {
    Success,
    Failure,
    Running,  // Still executing (multi-turn action)
}
```

### Example Behavior Tree

**Aggressive melee fighter**:
```
Selector
├─ Sequence: Attack if possible
│  ├─ Condition: EnemyInMeleeRange?
│  └─ Action: Attack nearest enemy
├─ Sequence: Move to attack
│  ├─ Condition: Can reach enemy this turn?
│  ├─ Action: Move toward nearest enemy
│  └─ Action: Attack if now in range
├─ Sequence: Defensive fallback
│  ├─ Condition: Low health?
│  └─ Action: Move toward allies
└─ Action: Move toward nearest enemy (default)
```

**Ranged support**:
```
Selector
├─ Sequence: Heal wounded ally
│  ├─ Condition: Ally below 50% HP?
│  ├─ Condition: In heal range?
│  └─ Action: Cast heal on most wounded
├─ Sequence: Attack from range
│  ├─ Condition: Enemy in weapon range?
│  ├─ Condition: Have line of sight?
│  └─ Action: Attack weakest visible enemy
├─ Action: Reposition to safe distance
└─ Action: Follow strongest ally
```

### Utility-Based AI

For more nuanced decisions, score multiple options:

```rust
struct UtilityAction {
    name: String,
    consider_fn: Box<dyn Fn(&GameState, EntityId) -> f32>,
    execute_fn: Box<dyn Fn(&mut GameState, EntityId)>,
}

struct UtilityAI {
    actions: Vec<UtilityAction>,
    randomness: f32,  // 0.0 = always pick best, 1.0 = weighted random
}
```

**Scoring example**:
```rust
// Attack scoring (with elevation awareness)
fn score_attack_target(state: &GameState, attacker: &Entity, target: &Entity) -> f32 {
    let mut score = 0.0;
    
    // Prefer low-health targets (can finish them off)
    let health_percent = target.hp as f32 / target.max_hp as f32;
    score += (1.0 - health_percent) * 30.0;
    
    // Prefer high-threat targets (damage dealers)
    score += get_threat_level(state, target) * 20.0;
    
    // Prefer closer targets (less movement needed, 3D distance)
    let distance = distance_3d(attacker.position, target.position);
    score += (10.0 - distance) * 5.0;
    
    // Penalize heavily-armored targets
    score -= target.armor as f32 * 2.0;
    
    // ELEVATION BONUS: High ground advantage
    let elevation_diff = attacker.position.elevation - target.position.elevation;
    score += match elevation_diff {
        d if d >= 2 => 15.0,   // Major high ground - big bonus
        1 => 10.0,             // Minor high ground - good bonus
        0 => 0.0,              // Even ground - neutral
        -1 => -5.0,            // Shooting uphill - penalty
        _ => -10.0,            // Major disadvantage - big penalty
    };
    
    // Bonus if target is near cliff edge (can knock them off)
    if is_near_cliff_edge(target.position, 1) && attacker.has_knockback_ability() {
        score += 12.0;  // Opportunity for massive fall damage
    }
    
    // Penalty if we don't have line of sight (elevation can block)
    if !has_line_of_sight(attacker.position, target.position, state.map) {
        score -= 100.0;  // Can't shoot what you can't see
    }
    
    score
}

// Movement position scoring (elevation-aware)
fn score_movement_position(pos: HexPosition, unit: &Entity, state: &GameState) -> f32 {
    let mut score = 0.0;
    
    // High ground preference
    let enemies = state.get_enemies_in_range(pos, 6);
    for enemy in enemies {
        let elev_diff = pos.elevation - enemy.position.elevation;
        score += elev_diff as f32 * 3.0;  // +3 per level advantage
    }
    
    // Cover bonus
    if provides_cover(pos, state.map) {
        score += 8.0;
    }
    
    // Avoid cliff edges (unless we can fly)
    if is_near_cliff_edge(pos, 1) && !unit.can_fly {
        score -= 10.0;
    }
    
    // Formation bonus (stay near allies)
    let nearby_allies = state.count_allies_in_range(pos, 3);
    score += nearby_allies as f32 * 2.0;
    
    // Line of sight to enemies
    let visible_enemies = enemies.iter()
        .filter(|e| has_line_of_sight(pos, e.position, state.map))
        .count();
    score += visible_enemies as f32 * 4.0;
    
    score
}
```

Pick highest-scoring action, or weighted random for variety.

## AI Personalities

Define different playstyles via data:

```toml
# content/ai/aggressive.toml
[personality]
name = "Aggressive"
risk_tolerance = 0.8      # Willing to take risky moves
focus_damage = 0.9        # Prioritize damage over survival
focus_defense = 0.1
value_positioning = 0.5   # Doesn't care much about formation

[behavior]
tree = "aggressive_melee.tree"
retreat_threshold = 0.2   # Only retreat below 20% HP
```

```toml
# content/ai/tactical.toml
[personality]
name = "Tactical"
risk_tolerance = 0.3
focus_damage = 0.6
focus_defense = 0.7
value_positioning = 0.9   # Cares about flanking, cover

[behavior]
tree = "tactical_mixed.tree"
retreat_threshold = 0.5
prefers_cover = true
flanking_bonus = 2.0
```

## Decision-Making Factors

### Tactical Considerations

**Target selection**:
- Health remaining (finish weak enemies)
- Threat level (prioritize dangerous foes)
- Distance (3D distance accounting for elevation)
- Armor/resistances (avoid ineffective attacks)
- Position (can allies follow up?)
- **Elevation advantage**: Prefer targets at lower elevation (high ground bonus)
- **Elevation disadvantage**: Avoid attacking uphill unless necessary

**Movement**:
- **Seek high ground**: Move to elevated positions for combat advantage
- Stay in cover when possible (elevation-based cover)
- Maintain formation with allies
- Control chokepoints (especially elevated choke points)
- Avoid area hazards (fire, poison clouds)
- Flanking opportunities (attack from behind or from higher elevation)
- **Avoid cliff edges**: Don't position where knockback would cause fall damage
- **Control ramps/stairs**: Defend access points to elevated positions

**Elevation-specific tactics**:
```rust
fn score_position_for_combat(pos: HexPosition, enemies: &[Entity], allies: &[Entity]) -> f32 {
    let mut score = 0.0;
    
    // Bonus for elevation over enemies
    for enemy in enemies {
        let elevation_diff = pos.elevation - enemy.position.elevation;
        match elevation_diff {
            d if d >= 2 => score += 8.0,   // Major high ground
            1 => score += 4.0,             // Minor high ground
            0 => score += 0.0,             // Even ground
            -1 => score -= 3.0,            // Shooting uphill penalty
            _ => score -= 6.0,             // Major disadvantage
        }
    }
    
    // Bonus for line of sight to enemies
    for enemy in enemies {
        if has_line_of_sight(pos, enemy.position) {
            score += 2.0;
        }
    }
    
    // Penalty for being at cliff edge (risky)
    if is_near_cliff_edge(pos, 1) {
        score -= 5.0;
    }
    
    // Bonus for cover
    if has_cover_from_elevation(pos) {
        score += 3.0;
    }
    
    score
}
```

**Ability usage**:
- Use AoE when hitting 2+ enemies (account for vertical spread)
- Save powerful abilities for high-value targets
- Use buffs early in combat
- Use heals when ally below threshold
- **Use knockback** to push enemies off cliffs (massive damage from fall)
- **Meteor/bombardment abilities** from high ground for maximum effect

### Information & Uncertainty

**Perfect information** (AI knows everything):
- Exact HP of all units
- All abilities and cooldowns
- Future turn order
- Hidden units (no fog of war for AI)

**Limited information** (more realistic):
- Estimates HP from visual damage
- Doesn't know cooldowns
- Surprised by new abilities
- Respects fog of war

**Recommendation**: Start with perfect info for consistent behavior, add uncertainty for difficulty modes.

## Difficulty Levels

Scale AI competence without just boosting stats:

```rust
enum AIDifficulty {
    Easy,
    Normal,
    Hard,
    Expert,
}

struct DifficultySettings {
    reaction_time: f32,        // Delay before acting (Easy = slow)
    mistake_chance: f32,       // % chance of suboptimal move
    planning_depth: u32,       // How many turns ahead to simulate
    use_abilities: bool,       // Easy AI may ignore complex abilities
    coordinate_attacks: bool,  // Focus fire vs spread damage
}
```

**Easy**:
- Makes obvious mistakes (attacks wrong target)
- Doesn't use abilities well
- No coordination between units
- Ignores flanking/positioning

**Normal**:
- Competent individual unit play
- Uses abilities appropriately
- Basic coordination (focus fire)
- Respects positioning

**Hard**:
- Optimal target selection
- Advanced ability combos
- Strong coordination (baiting, pincer moves)
- Exploits player mistakes

**Expert**:
- Simulates future turns (minimax-style)
- Perfect ability timing
- Coordinated multi-turn strategies
- Adapts to player patterns

## Group Tactics

Coordination between AI units:

```rust
struct AISquad {
    leader: EntityId,
    members: Vec<EntityId>,
    formation: FormationType,
    objective: SquadObjective,
}

enum FormationType {
    Line,        // Horizontal spread
    Column,      // Single file
    Wedge,       // V-formation
    Circle,      // Surround target
    Skirmish,    // Loose spacing
}

enum SquadObjective {
    Attack(EntityId),        // Focus this target
    Defend(HexCoord),        // Hold this position
    Retreat(HexCoord),       // Fall back to position
    Support(EntityId),       // Assist this ally
}
```

**Squad behaviors**:
- Leader makes decisions, members follow
- Maintain formation when moving
- Coordinated attacks (stagger to maximize damage)
- Protect weak members (tanks block for mages)

## Performance Budget

Turn-based allows more thinking time:

- **Per-unit budget**: 100-500ms per AI turn
- **Total budget**: 2-5 seconds for all AI units
- **Show "AI thinking" indicator** if > 1 second

**Optimization**:
- Cache pathfinding results
- Limit search depth for complex decisions
- Use early-exit heuristics (good enough vs perfect)
- Parallelize AI calculations across units

## Debugging AI

Tools for understanding AI decisions:

```rust
struct AIDebugger {
    show_considerations: bool,   // Display all options scored
    show_behavior_tree: bool,    // Visualize tree traversal
    show_threat_map: bool,       // Color code threat levels
    log_decisions: bool,         // Text log of reasoning
}
```

**Debug overlay**:
- Show considered movement options with scores
- Highlight selected target and why
- Display current behavior tree node
- Log: "Attacking Goblin (HP: 5/12, Score: 42.3) instead of Orc (HP: 20/30, Score: 38.1)"

## Special AI Types

### Boss AI

More complex behaviors for unique enemies:

```rust
struct BossAI {
    phase_thresholds: Vec<f32>,  // HP % to change phase
    current_phase: u32,
    enrage_at_low_health: bool,
    summon_minions: bool,
}
```

**Phase-based behavior**:
- Phase 1 (100-70% HP): Normal attacks
- Phase 2 (70-40% HP): Start using AoE abilities
- Phase 3 (40-0% HP): Enrage, summon adds, desperate attacks

### Neutral AI

Non-hostile units with conditional aggression:

```rust
struct NeutralAI {
    aggressive_if_attacked: bool,
    defends_territory: Option<Vec<HexCoord>>,
    flees_when_damaged: bool,
}
```

**Example**: Guard won't attack unless you enter their zone or attack first.

## AI Testing

Automated scenarios to verify AI competence:

- **Target selection test**: Given 3 enemies at different HP/threat, picks the right one
- **Pathfinding test**: Finds optimal path around obstacles
- **Ability usage test**: Uses heal when ally is low, not full
- **Coordination test**: Multiple AI units focus-fire effectively

```rust
#[test]
fn ai_prioritizes_low_health_targets() {
    let mut state = create_test_scenario();
    // Enemy A: 100 HP, Enemy B: 5 HP
    let action = ai.choose_action(&state);
    assert_eq!(action.target, enemy_b);
}
```

## Future Extensions

- **Learning AI**: Adapts to player strategies over campaign
- **Scripted sequences**: Pre-programmed moves for tutorials/setpieces
- **Personality adaptation**: AI becomes more aggressive if player is defensive
- **Emergent behavior**: Simple rules leading to complex tactics
