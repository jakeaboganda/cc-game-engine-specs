# Hex Grid System

## Coordinate System

**Use cube coordinates** for all internal calculations (converts to axial or offset for storage/display).

```rust
struct HexCoord {
    q: i32,  // column (axial)
    r: i32,  // row (axial)
    s: i32,  // derived: -q - r
}
```

**Constraint**: `q + r + s = 0` always.

### Why Cube Coordinates?
- Symmetric operations (all 6 directions treated equally)
- Simpler distance calculation: `(|q| + |r| + |s|) / 2`
- Cleaner neighbor math

## Core Operations

### Distance
```
distance(a, b) = (|a.q - b.q| + |a.r - b.r| + |a.s - b.s|) / 2
```

### Neighbors
Six directions defined as:
```
directions = [
    (+1, 0, -1),  // E
    (+1, -1, 0),  // NE
    (0, -1, +1),  // NW
    (-1, 0, +1),  // W
    (-1, +1, 0),  // SW
    (0, +1, -1),  // SE
]
```

### Range Query
All hexes within N steps:
```
for q in -N..=N:
    for r in max(-N, -q-N)..=min(N, -q+N):
        s = -q - r
        yield (q, r, s)
```

### Line Drawing
Use linear interpolation for lines (ray casting, projectile paths):
```
lerp_hex(a, b, t) = round(a + (b - a) * t)
```

## Elevation & 3D

Support vertical layers for multi-level maps:
```rust
struct HexTile {
    coord: HexCoord,
    elevation: i32,  // 0 = ground level
    terrain: TerrainType,
}
```

**Movement cost**: Uphill costs more, downhill may cost less or provide bonuses.

## Pathfinding

A* implementation with:
- **Cost function**: Movement points based on terrain + elevation change
- **Heuristic**: Hex distance to goal
- **Obstacles**: Impassable tiles, enemy units (may be passable with rules)
- **Caching**: Store recently calculated paths for AI reuse

## Line of Sight

Raycasting algorithm:
1. Draw line from source to target using `lerp_hex`
2. Check each hex along path for blocking terrain/units
3. Account for elevation (higher ground can see over obstacles)

**Optimization**: Pre-calculate LOS maps for common positions.

## Rendering

Convert cube coordinates to pixel positions:
```
size = hex_radius
x = size * (3/2) * q
y = size * sqrt(3) * (r + q/2)
```

**Flat-top vs pointy-top**: Above formula is for flat-top. Swap x/y calculations for pointy-top.

## Map Storage

Store maps as:
```toml
[map]
width = 20
height = 15
default_terrain = "grass"

[[tiles]]
q = 5
r = 3
terrain = "mountain"
elevation = 2

[[tiles]]
q = 6
r = 3
terrain = "water"
elevation = -1
```

Or use 2D array with offset coordinates for compact storage, convert to cube on load.

## Performance Considerations

- **Spatial hashing**: For large maps, partition hexes into chunks for faster neighbor queries
- **Pre-computed tables**: Store neighbor offsets, distance lookup tables
- **Cache terrain data**: Don't recalculate movement costs repeatedly
