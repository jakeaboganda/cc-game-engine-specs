# Hex Grid System (2.5D)

## Coordinate System

**3D hex coordinates**: Cube coordinates for horizontal position + elevation for vertical.

```rust
struct HexCoord {
    q: i32,  // column (axial)
    r: i32,  // row (axial)
    s: i32,  // derived: -q - r
}

struct HexPosition {
    hex: HexCoord,
    elevation: i32,  // Vertical level (0 = ground, negatives for pits/water)
}
```

**Constraint**: `q + r + s = 0` always.

### Why Cube Coordinates?
- Symmetric operations (all 6 directions treated equally)
- Simpler distance calculation: `(|q| + |r| + |s|) / 2`
- Cleaner neighbor math
- Easy extension to 3D with elevation

## Core Operations

### Horizontal Distance
```rust
fn hex_distance(a: HexCoord, b: HexCoord) -> i32 {
    (a.q - b.q).abs() + (a.r - b.r).abs() + (a.s - b.s).abs()) / 2
}
```

### 3D Distance
Include elevation in distance calculations:

```rust
fn distance_3d(a: HexPosition, b: HexPosition) -> f32 {
    let hex_dist = hex_distance(a.hex, b.hex) as f32;
    let elevation_diff = (a.elevation - b.elevation).abs() as f32;
    
    // Pythagorean: horizontal + vertical
    (hex_dist.powi(2) + elevation_diff.powi(2)).sqrt()
}
```

**Use cases**:
- Projectile range (accounts for height)
- Visual distance (rendering priorities)
- Area effects with vertical falloff

### Neighbors
Six horizontal directions:
```rust
const HEX_DIRECTIONS: [(i32, i32, i32); 6] = [
    (+1,  0, -1),  // E
    (+1, -1,  0),  // NE
    ( 0, -1, +1),  // NW
    (-1,  0, +1),  // W
    (-1, +1,  0),  // SW
    ( 0, +1, -1),  // SE
];

fn get_neighbor(hex: HexCoord, direction: usize) -> HexCoord {
    let (dq, dr, ds) = HEX_DIRECTIONS[direction];
    HexCoord {
        q: hex.q + dq,
        r: hex.r + dr,
        s: hex.s + ds,
    }
}

fn get_neighbors(pos: HexPosition) -> Vec<HexPosition> {
    HEX_DIRECTIONS.iter()
        .map(|&(dq, dr, ds)| HexPosition {
            hex: HexCoord {
                q: pos.hex.q + dq,
                r: pos.hex.r + dr,
                s: pos.hex.s + ds,
            },
            elevation: pos.elevation,  // Same elevation
        })
        .collect()
}
```

### Range Query
All hexes within N steps (same elevation):
```rust
fn hexes_in_range(center: HexCoord, range: i32) -> Vec<HexCoord> {
    let mut results = Vec::new();
    
    for q in -range..=range {
        for r in (-range).max(-q-range)..=(range).min(-q+range) {
            let s = -q - r;
            results.push(HexCoord { q, r, s });
        }
    }
    
    results
}
```

**3D range** (include elevation):
```rust
fn positions_in_3d_range(center: HexPosition, range: i32, vertical_range: i32) -> Vec<HexPosition> {
    let mut results = Vec::new();
    
    for hex in hexes_in_range(center.hex, range) {
        for elev in (center.elevation - vertical_range)..=(center.elevation + vertical_range) {
            results.push(HexPosition {
                hex,
                elevation: elev,
            });
        }
    }
    
    results
}
```

## Elevation & Multi-Level Terrain

### Terrain Representation

```rust
struct HexTile {
    coord: HexCoord,
    base_elevation: i32,        // Base height of this tile
    terrain_type: TerrainType,
    corners: [i32; 6],          // Elevation of each corner (for slopes)
    blocking: bool,             // Blocks movement/LOS
    cover_level: CoverLevel,
}

enum TerrainType {
    Grass,
    Stone,
    Water,
    Mountain,
    Void,  // Empty space (between levels)
}

enum CoverLevel {
    None,
    Half,      // Low wall, rock
    Full,      // High wall, building
}

impl HexTile {
    fn is_flat(&self) -> bool {
        self.corners.iter().all(|&c| c == self.base_elevation)
    }
    
    fn is_sloped(&self) -> bool {
        !self.is_flat() && self.max_corner_diff() <= 1
    }
    
    fn is_cliff(&self) -> bool {
        self.max_corner_diff() > 1
    }
    
    fn max_corner_diff(&self) -> i32 {
        let min = self.corners.iter().min().unwrap();
        let max = self.corners.iter().max().unwrap();
        max - min
    }
}
```

### Elevation Types

**Flat terrain**:
- All corners at same elevation
- Easy movement
- Example: Plains, floors

**Slopes**:
- Gradual elevation change (1 level difference max between corners)
- Walkable but may cost extra movement
- Example: Hills, ramps

**Cliffs**:
- Sharp drops (2+ levels between corners)
- Not walkable without special abilities
- Render vertical cliff face
- Example: Mountains, plateaus

### Movement Costs

```rust
fn calculate_movement_cost(from: &HexTile, to: &HexTile) -> Option<i32> {
    if to.blocking {
        return None;  // Impassable
    }
    
    let elevation_diff = to.base_elevation - from.base_elevation;
    
    match elevation_diff {
        // Same level or downhill
        d if d <= 0 => Some(1),
        
        // Gentle uphill (slope)
        1 if to.is_sloped() => Some(2),
        
        // Steep climb (flat elevated tile)
        1 => Some(3),
        
        // Cliff (requires climbing ability)
        d if d >= 2 => None,
        
        _ => Some(1),
    }
}
```

**Special movement modes**:
- **Flying**: Ignores elevation, can cross cliffs
- **Climbing**: Can move up cliffs at high cost
- **Jumping**: Can leap across 1-hex gaps with elevation changes

## Pathfinding (3D)

A* with elevation awareness:

```rust
struct PathfindingNode {
    position: HexPosition,
    g_cost: i32,      // Actual cost from start
    h_cost: i32,      // Heuristic to goal
    parent: Option<HexPosition>,
}

fn find_path(
    start: HexPosition,
    goal: HexPosition,
    map: &HexMap,
    movement_type: MovementType,
) -> Option<Vec<HexPosition>> {
    let mut open_set = BinaryHeap::new();
    let mut closed_set = HashSet::new();
    
    open_set.push(PathfindingNode {
        position: start,
        g_cost: 0,
        h_cost: heuristic_3d(start, goal),
        parent: None,
    });
    
    while let Some(current) = open_set.pop() {
        if current.position == goal {
            return Some(reconstruct_path(current));
        }
        
        closed_set.insert(current.position);
        
        for neighbor in get_walkable_neighbors(current.position, map, movement_type) {
            if closed_set.contains(&neighbor) {
                continue;
            }
            
            let move_cost = calculate_movement_cost_3d(
                current.position,
                neighbor,
                map,
                movement_type,
            );
            
            let g_cost = current.g_cost + move_cost;
            let h_cost = heuristic_3d(neighbor, goal);
            
            open_set.push(PathfindingNode {
                position: neighbor,
                g_cost,
                h_cost,
                parent: Some(current.position),
            });
        }
    }
    
    None  // No path found
}

fn heuristic_3d(a: HexPosition, b: HexPosition) -> i32 {
    let hex_dist = hex_distance(a.hex, b.hex);
    let elevation_diff = (a.elevation - b.elevation).abs();
    
    // Manhattan distance in 3D
    hex_dist + elevation_diff
}
```

**Flying pathfinding**:
```rust
fn find_flying_path(start: HexPosition, goal: HexPosition) -> Vec<HexPosition> {
    // Direct line through 3D space, ignoring terrain
    // Only blocked by tall obstacles (towers, mountains)
}
```

## Line of Sight (3D)

Height-aware LOS with blocking:

```rust
fn has_line_of_sight(
    from: HexPosition,
    to: HexPosition,
    map: &HexMap,
) -> bool {
    let path = line_between_hexes(from.hex, to.hex);
    
    for hex in path {
        let tile = map.get_tile(hex);
        
        // Calculate height at this point along the line
        let t = calculate_interpolation_factor(from.hex, to.hex, hex);
        let height_at_point = lerp(from.elevation as f32, to.elevation as f32, t);
        
        // Check if terrain blocks at this height
        if tile.blocks_los_at_height(height_at_point as i32) {
            return false;
        }
    }
    
    true
}

impl HexTile {
    fn blocks_los_at_height(&self, height: i32) -> bool {
        match self.cover_level {
            CoverLevel::None => false,
            CoverLevel::Half => height <= self.base_elevation,
            CoverLevel::Full => height <= self.base_elevation + 1,
        }
    }
}
```

**High ground advantage**:
```rust
fn has_high_ground_advantage(attacker: HexPosition, target: HexPosition) -> bool {
    attacker.elevation > target.elevation
}

fn calculate_los_bonus(attacker: HexPosition, target: HexPosition) -> i32 {
    let elevation_diff = attacker.elevation - target.elevation;
    
    match elevation_diff {
        d if d >= 2 => 3,   // Major high ground (+3 to hit)
        1 => 2,             // Minor high ground (+2 to hit)
        0 => 0,             // Even ground
        -1 => -1,           // Shooting uphill (-1 to hit)
        _ => -2,            // Shooting way uphill (-2 to hit)
    }
}
```

**Cover from elevation**:
```rust
fn has_cover_from_elevation(
    target: HexPosition,
    attacker: HexPosition,
    map: &HexMap,
) -> bool {
    // If target is lower and attacker must shoot over terrain
    if target.elevation < attacker.elevation {
        return false;
    }
    
    // Check if any intervening hex provides cover
    let path = line_between_hexes(attacker.hex, target.hex);
    for hex in path {
        let tile = map.get_tile(hex);
        if tile.base_elevation >= target.elevation - 1 {
            return true;  // Intervening terrain provides cover
        }
    }
    
    false
}
```

## Area Effects with Elevation

```rust
fn get_area_effect_targets(
    center: HexPosition,
    radius: i32,
    vertical_spread: i32,
    map: &HexMap,
) -> Vec<HexPosition> {
    let mut targets = Vec::new();
    
    for hex in hexes_in_range(center.hex, radius) {
        for elevation in (center.elevation - vertical_spread)..=(center.elevation + vertical_spread) {
            let pos = HexPosition { hex, elevation };
            
            // Check if this position is valid and in range
            if map.is_valid_position(pos) {
                targets.push(pos);
            }
        }
    }
    
    targets
}
```

**Example**: Fireball explodes at elevation 1, affects units at elevation 0, 1, and 2 within radius.

## Map Storage

```toml
[map]
width = 20
height = 15
default_terrain = "grass"
default_elevation = 0

# Flat tile
[[tiles]]
q = 5
r = 3
terrain = "stone"
elevation = 2
corners = [2, 2, 2, 2, 2, 2]  # All corners at elevation 2

# Sloped tile (ramp)
[[tiles]]
q = 6
r = 3
terrain = "stone"
elevation = 1
corners = [1, 1, 2, 2, 1, 1]  # Slope from 1 to 2

# Cliff
[[tiles]]
q = 7
r = 3
terrain = "mountain"
elevation = 3
blocking = false
cover = "full"

# Water (below ground level)
[[tiles]]
q = 8
r = 3
terrain = "water"
elevation = -1
```

## Coordinate Conversion (Isometric)

See rendering.md for full isometric projection details:

```rust
fn hex_to_iso_pixel(pos: HexPosition, projection: &ProjectionMatrix) -> Vec3 {
    // Convert hex to flat 2D
    let flat = hex_to_flat_2d(pos.hex);
    
    // Apply isometric transformation
    let iso_x = (flat.x - flat.y) * 0.5;
    let iso_y = (flat.x + flat.y) * 0.25;
    
    // Add elevation offset (higher = further up on screen)
    let screen_y = iso_y - (pos.elevation as f32 * projection.elevation_scale);
    
    Vec3::new(iso_x, screen_y, pos.elevation as f32)
}
```

## Performance Considerations

### Spatial Hashing (3D)

```rust
struct SpatialHashMap3D {
    cells: HashMap<(i32, i32, i32), Vec<EntityId>>,  // (q, r, elevation) → entities
    cell_size: i32,
}

impl SpatialHashMap3D {
    fn get_cell_key(&self, pos: HexPosition) -> (i32, i32, i32) {
        (
            pos.hex.q / self.cell_size,
            pos.hex.r / self.cell_size,
            pos.elevation,
        )
    }
    
    fn get_nearby_entities(&self, pos: HexPosition, range: i32) -> Vec<EntityId> {
        // Query cells in range, including elevation
        // Much faster than checking all entities
    }
}
```

### Pre-computed Data

```rust
struct HexMapCache {
    neighbor_cache: HashMap<HexCoord, [HexCoord; 6]>,
    distance_cache: HashMap<(HexCoord, HexCoord), i32>,
    los_cache: HashMap<(HexPosition, HexPosition), bool>,
}
```

**Cache invalidation**: Clear LOS cache when terrain changes (destructible terrain).

### Level-of-Detail

```rust
fn should_calculate_detailed_path(distance: i32) -> bool {
    distance < 20  // Only calculate full path for nearby goals
}

fn get_approximate_path(start: HexPosition, goal: HexPosition) -> Vec<HexPosition> {
    // Simple straight-line path for distant targets
    // AI refines as it gets closer
}
```

## Debug Visualization

```rust
struct HexGridDebugger {
    show_coordinates: bool,
    show_elevation_numbers: bool,
    show_movement_costs: bool,
    show_los_rays: bool,
    highlight_elevation: Option<i32>,  // Highlight all tiles at this elevation
}
```

**Visualizations**:
- Elevation heat map (color gradient by height)
- Movement cost overlay (numbers on each hex)
- LOS rays (lines from selected unit)
- Reachable hexes (within movement range, respecting elevation)

## Future Extensions

- **Destructible terrain**: Cliffs can be destroyed, creating new paths
- **Dynamic elevation**: Terrain that rises/falls (lava, water levels)
- **Bridges**: Walkable hexes above other hexes
- **Tunnels**: Multiple elevations at same horizontal hex
- **Volumetric fog**: Fog at specific elevation ranges
