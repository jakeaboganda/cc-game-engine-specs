# Data-Driven Content System

## Design Philosophy

Game content should be defined in **data files** (RON/TOML/JSON), not hardcoded in Rust. This enables:
- Non-programmers to create content
- Hot-reloading during development
- Easy modding support
- Rapid iteration and balancing

## Content Types

### Unit Definitions

```toml
# content/units/warrior.toml
[unit]
id = "warrior"
name = "Warrior"
sprite = "units/warrior.png"

[stats]
max_hp = 30
armor_class = 16
speed = 5
initiative_bonus = 1

[attack]
name = "Longsword"
range = "melee"
hit_bonus = 5
damage = "1d8+3"
damage_type = "slashing"

[[abilities]]
id = "power_attack"
name = "Power Attack"
cooldown = 3
action_cost = "standard"

[[resistances]]
damage_type = "fire"
resist_type = "half"
```

### Ability Templates

```toml
# content/abilities/fireball.toml
[ability]
id = "fireball"
name = "Fireball"
description = "Hurl a ball of flame that explodes on impact"
action_cost = "standard"
cooldown = 0
uses_per_encounter = 3

[targeting]
type = "area_of_effect"
range = 10
shape = "circle"
radius = 2
requires_los = true

[[effects]]
type = "damage"
dice = "8d6"
damage_type = "fire"
allow_save = true
save_stat = "dexterity"
save_dc = 15
on_save = "half_damage"

[[effects]]
type = "apply_status"
status_id = "burning"
duration_turns = 2
chance = 0.5
```

### Map/Scenario Files

```toml
# content/maps/forest_ambush.toml
[map]
name = "Forest Ambush"
width = 15
height = 12
default_terrain = "grass"
default_elevation = 0

[[terrain]]
q = 3
r = 4
type = "tree"
blocks_los = true
blocks_movement = true

[[terrain]]
q = 7
r = 2
type = "difficult"
movement_cost = 2

[encounter]
description = "Bandits ambush the party"

[[spawn_points]]
team = "player"
coords = [
    { q = 1, r = 1 },
    { q = 2, r = 1 },
    { q = 1, r = 2 },
    { q = 2, r = 2 },
]

[[enemy_groups]]
unit_type = "bandit"
count = 4
spawn_area = { q = 12, r = 6, radius = 2 }

[[enemy_groups]]
unit_type = "bandit_captain"
count = 1
spawn_coords = { q = 13, r = 7 }

[victory]
type = "eliminate_all"
target_team = "enemy"
```

### Status Effect Definitions

```toml
# content/effects/poisoned.toml
[status]
id = "poisoned"
name = "Poisoned"
icon = "icons/poison.png"
duration_type = "turns"
default_duration = 3

[[on_turn_start]]
type = "damage"
dice = "1d4"
damage_type = "poison"
message = "{target} takes poison damage!"

[[stat_modifiers]]
stat = "attack_bonus"
modifier = -2
```

## Asset Organization

```
content/
├── units/           # Unit definitions
├── abilities/       # Ability templates
├── effects/         # Status effects
├── maps/            # Map layouts
├── scenarios/       # Full encounter definitions
├── items/           # Equipment and consumables
└── campaigns/       # Series of linked scenarios

assets/
├── sprites/
│   ├── units/
│   ├── terrain/
│   ├── effects/
│   └── ui/
├── audio/
│   ├── sfx/
│   └── music/
└── fonts/
```

## Hot Reload System

Watch for file changes in `content/` and `assets/`:
```rust
// Pseudo-code
on_file_change(path) {
    match path.extension() {
        "toml" | "ron" | "json" => reload_content(path),
        "png" | "jpg" => reload_texture(path),
        "ogg" | "wav" => reload_audio(path),
    }
    emit_event(ContentReloaded { path })
}
```

**Developer experience**:
1. Edit `warrior.toml`, change HP from 30 → 35
2. Save file
3. Game instantly reflects new HP (even mid-combat for testing)

## Validation & Error Handling

Content files should be validated on load:
- Check required fields exist
- Verify references (ability IDs, unit types) exist
- Validate numeric ranges (HP > 0, etc.)
- Provide helpful error messages:
  ```
  Error in content/units/warrior.toml:
  - Missing required field: stats.max_hp
  - Invalid reference: abilities.unknown_ability (not found)
  ```

## Scripting Layer (Optional)

For complex behaviors that don't fit templates, embed Lua/Rhai:

```toml
[ability]
id = "chain_lightning"
name = "Chain Lightning"
script = """
function on_activate(caster, target, world)
    local targets = { target }
    for i = 1, 3 do
        local next = find_nearest_enemy(targets[#targets], world, 3)
        if next then
            table.insert(targets, next)
        else
            break
        end
    end
    
    for i, t in ipairs(targets) do
        apply_damage(t, roll_dice("4d6") / i)
    end
end
"""
```

Provides escape hatch for unique mechanics without recompiling Rust.

## Modding Support

To enable community content:
1. **Mod directory**: `~/.cc-game-engine/mods/`
2. **Mod manifest**: Each mod has `mod.toml` describing contents
3. **Load order**: Core content → mods (alphabetical or priority-based)
4. **Overrides**: Mods can override core content by matching IDs
5. **Safety**: Validate mod content, sandbox scripts

Example mod structure:
```
~/.cc-game-engine/mods/my-mod/
├── mod.toml
├── content/
│   ├── units/new_unit.toml
│   └── abilities/custom_spell.toml
└── assets/
    └── sprites/custom_sprite.png
```

## Performance

- **Lazy loading**: Don't load all content at startup, load on-demand
- **Caching**: Parse each file once, cache in memory
- **Incremental reload**: Only reload changed files, not entire content tree
- **Background loading**: Load content on separate thread to avoid frame drops
