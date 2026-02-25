# Rendering System

## Overview

**2.5D isometric rendering** for tactical hex-based games. Combines the visual depth of 3D with the performance and art workflow of 2D sprites. Supports elevation, shadows, and depth-sorted rendering.

## Design Philosophy

**2.5D approach**:
- Isometric or dimetric projection (angled view, not top-down)
- Sprites rendered with depth sorting based on Y-position and elevation
- Visual height differences (units on cliffs, multi-level terrain)
- Shadows cast on ground plane
- Option for 3D models rendered in isometric view OR pre-rendered 2D sprites

**Why 2.5D**:
- Better spatial awareness than flat 2D
- Easier to read elevation and unit positioning
- More immersive tactical feel
- Simpler art pipeline than full 3D
- Classic tactical RPG aesthetic (XCOM, Tactics Ogre, FFT)

## Projection System

### Isometric vs Dimetric

```rust
enum ProjectionMode {
    Isometric,      // 1:2 ratio (26.565° angle), true isometric
    Dimetric,       // 1:1.5 ratio (~30° angle), slightly less steep
    Custom(f32),    // Custom angle in degrees
}

struct ProjectionMatrix {
    mode: ProjectionMode,
    tile_width: f32,   // Base hex width in pixels
    tile_height: f32,  // Base hex height in pixels
    elevation_scale: f32,  // Pixels per elevation level
}
```

**Standard isometric (recommended)**:
- 2:1 pixel ratio for tiles
- 30° camera angle from horizontal
- Clean, readable, proven in many games

### Coordinate Conversion

```rust
// Hex cube coords + elevation → isometric screen position
fn hex_to_iso_pixel(hex: HexCoord, elevation: i32, projection: &ProjectionMatrix) -> Vec3 {
    // First convert hex to flat 2D position
    let flat_x = projection.tile_width * (3.0_f32.sqrt() * hex.q as f32 
                                          + 3.0_f32.sqrt() / 2.0 * hex.r as f32);
    let flat_y = projection.tile_width * (3.0 / 2.0 * hex.r as f32);
    
    // Apply isometric transformation
    let iso_x = (flat_x - flat_y) * 0.5;
    let iso_y = (flat_x + flat_y) * 0.25;
    
    // Add elevation (higher = render higher on screen)
    let screen_y = iso_y - (elevation as f32 * projection.elevation_scale);
    
    Vec3::new(iso_x, screen_y, elevation as f32)  // Z stored for depth sorting
}

// Screen pixel → hex coord (for mouse picking)
fn iso_pixel_to_hex(screen_pos: Vec2, elevation: i32, projection: &ProjectionMatrix) -> HexCoord {
    // Reverse isometric transformation
    let adjusted_y = screen_pos.y + (elevation as f32 * projection.elevation_scale);
    
    let flat_x = screen_pos.x + adjusted_y * 2.0;
    let flat_y = adjusted_y * 2.0 - screen_pos.x;
    
    // Convert flat coords to hex
    let q = (flat_x * 3.0_f32.sqrt() / 3.0 - flat_y / 3.0) / projection.tile_width;
    let r = flat_y * 2.0 / 3.0 / projection.tile_width;
    
    hex_round(q, r)
}
```

## Depth Sorting

Critical for correct visual layering:

```rust
struct DepthSortedSprite {
    sprite: Sprite,
    position: Vec3,      // x, y, elevation
    depth_bias: f32,     // Manual adjustment for edge cases
}

impl DepthSortedSprite {
    fn depth_key(&self) -> i32 {
        // Primary: Y position (back-to-front)
        // Secondary: Elevation (higher objects in front when at same Y)
        // Tertiary: Bias (manual override)
        
        let y_priority = (self.position.y * 1000.0) as i32;
        let elevation_priority = (self.position.z * 10.0) as i32;
        let bias_priority = (self.depth_bias * 100.0) as i32;
        
        y_priority - elevation_priority + bias_priority
    }
}

fn sort_sprites(sprites: &mut Vec<DepthSortedSprite>) {
    sprites.sort_by_key(|s| s.depth_key());
}
```

**Sorting rules**:
1. Objects further back (higher Y) render first
2. At same Y, higher elevation renders in front
3. Manual bias for special cases (flying units, effects)

**Optimization**: Use spatial hashing to avoid sorting all sprites every frame—only sort within visible chunks.

## Elevation System

Multi-level terrain and units:

```rust
struct ElevationConfig {
    levels: Vec<f32>,          // [0.0, 1.0, 2.0, 3.0] etc.
    height_per_level: f32,     // Pixels of vertical offset per level
    transition_slope: bool,    // Smooth ramps vs hard cliffs
}

struct TerrainTile {
    hex: HexCoord,
    elevation: i32,            // Base elevation level
    terrain_type: TerrainType,
    corners: Option<[i32; 6]>, // Elevation of each hex corner (for slopes)
}
```

**Visual representation**:
- **Flat terrain**: All corners same elevation
- **Slopes**: Gradual corner transitions (walkable)
- **Cliffs**: Sharp drops (require climbing or flying)
- **Elevation walls**: Render cliff faces between levels

### Cliff Rendering

```rust
fn render_cliff_face(tile: &TerrainTile, neighbor: &TerrainTile) {
    if tile.elevation > neighbor.elevation {
        let height_diff = tile.elevation - neighbor.elevation;
        
        // Render vertical wall sprite
        let wall_sprite = get_cliff_sprite(tile.terrain_type, height_diff);
        let position = calculate_wall_position(tile, neighbor);
        
        render_sprite(wall_sprite, position);
    }
}
```

**Cliff tiles**:
- Different sprites for 1-level, 2-level, 3+ level drops
- Corner pieces for inside/outside corners
- Match terrain type (grass cliff, stone cliff, etc.)

## Shadow System

Ground shadows for depth perception:

```rust
struct ShadowRenderer {
    shadow_offset: Vec2,     // Direction and distance of shadow
    shadow_alpha: f32,       // Opacity (0.0-1.0)
    elevation_multiplier: f32,  // Higher units = longer shadows
}

fn render_unit_shadow(unit: &Unit, config: &ShadowRenderer) {
    let shadow_pos = Vec2::new(
        unit.position.x + config.shadow_offset.x,
        unit.position.y + config.shadow_offset.y + (unit.elevation as f32 * config.elevation_multiplier),
    );
    
    let shadow_sprite = Sprite {
        texture: unit.sprite.texture,
        tint: Color::rgba(0.0, 0.0, 0.0, config.shadow_alpha),
        // Squash shadow vertically for perspective
        scale: Vec2::new(1.0, 0.5),
        ..unit.sprite
    };
    
    render_sprite_at_ground_level(shadow_sprite, shadow_pos);
}
```

**Types**:
- **Blob shadows**: Simple circular shadow under units
- **Sprite shadows**: Silhouette of unit sprite (more realistic)
- **Dynamic shadows**: Change with lighting direction (advanced)

## Camera System

```rust
struct IsometricCamera {
    position: Vec3,          // World position (x, y, elevation)
    zoom: f32,               // Scale factor
    pitch: f32,              // Vertical angle (fixed for pure iso, adjustable for 2.5D)
    yaw: f32,                // Rotation around vertical axis (usually fixed)
    viewport: Rect,
    projection: ProjectionMatrix,
}
```

### Camera Controls

**Movement**:
- Pan in isometric space (WASD moves along iso axes, not screen axes)
- Zoom in/out (scale entire view)
- Optional rotation (rotate world in 90° increments for different viewing angles)

**Focus modes**:
- Free-cam: Player controls
- Unit-follow: Camera tracks selected unit
- Cinematic: Scripted camera paths for abilities

### Multi-Camera (Split-Screen)

```rust
struct IsoCameraManager {
    cameras: Vec<IsometricCamera>,
    layout: CameraLayout,
}

enum CameraLayout {
    Single,
    Horizontal2,
    Vertical2,
    Quad,
}
```

Each camera has independent position/zoom but shares projection settings.

## Rendering Pipeline

### Render Passes

```
1. Terrain Base
   ├─ Ground tiles (elevation 0)
   ├─ Elevated terrain (elevation 1+)
   └─ Cliff faces

2. Ground Shadows
   └─ All unit/object shadows on ground plane

3. Depth-Sorted Objects
   ├─ Terrain features (trees, rocks)
   ├─ Units (sorted by Y + elevation)
   ├─ Effects (particles, spells)
   └─ Projectiles

4. Overlays (no depth)
   ├─ Selection highlights
   ├─ Range indicators
   ├─ Health bars
   └─ Status icons

5. UI Layer
   ├─ HUD
   ├─ Menus
   └─ Tooltips
```

### Batching Strategy

```rust
struct IsometricRenderBatch {
    texture: TextureHandle,
    sprites: Vec<(Sprite, Vec3)>,  // (sprite, world_pos)
    depth_sorted: bool,
}

fn prepare_batches(entities: &[Entity]) -> Vec<IsometricRenderBatch> {
    let mut batches = HashMap::new();
    
    for entity in entities {
        let key = (entity.sprite.texture, entity.render_layer);
        batches.entry(key)
            .or_insert_with(Vec::new)
            .push((entity.sprite, entity.position));
    }
    
    // Sort each batch by depth
    for batch in batches.values_mut() {
        batch.sort_by_key(|(_, pos)| depth_key(pos));
    }
    
    batches.into_iter()
        .map(|((texture, _), sprites)| IsometricRenderBatch {
            texture,
            sprites,
            depth_sorted: true,
        })
        .collect()
}
```

## 3D Models Option

Support actual 3D models rendered in isometric view:

```rust
enum RenderMode {
    Sprites2D,           // Pre-rendered sprite sheets
    Models3D,            // Real-time 3D models
    Hybrid,              // Mix both (3D units, 2D terrain)
}

struct Model3DRenderer {
    mesh: MeshHandle,
    materials: Vec<MaterialHandle>,
    animation_state: AnimationState,
}

fn render_3d_in_isometric_view(model: &Model3DRenderer, position: Vec3, camera: &IsometricCamera) {
    // Set orthographic projection matching isometric angle
    let ortho_matrix = create_orthographic_projection(
        camera.projection.mode,
        camera.zoom,
    );
    
    // Render 3D model with fixed camera angle
    let view_matrix = create_isometric_view_matrix(camera.pitch, camera.yaw);
    
    render_mesh(model.mesh, ortho_matrix * view_matrix, position);
}
```

**Hybrid approach (recommended)**:
- 3D models for units (easier animation, rotation)
- 2D sprites for terrain (faster, stylized look)
- Depth buffer ensures correct layering

## Hex Grid Rendering (Isometric)

```rust
struct IsometricHexRenderer {
    tile_sprites: HashMap<TerrainType, TextureHandle>,
    elevation_sprites: HashMap<(TerrainType, i32), TextureHandle>,  // Cliff sprites
    grid_overlay: bool,
}

fn render_hex_tile(hex: &TerrainTile, renderer: &IsometricHexRenderer) {
    let screen_pos = hex_to_iso_pixel(hex.hex, hex.elevation, &projection);
    
    // Base tile sprite
    let sprite = renderer.tile_sprites.get(&hex.terrain_type);
    render_sprite(sprite, screen_pos);
    
    // Elevation walls (if higher than neighbors)
    for direction in HexDirection::all() {
        let neighbor = get_neighbor(hex, direction);
        if hex.elevation > neighbor.elevation {
            render_cliff_face(hex, &neighbor);
        }
    }
}
```

**Grid overlay**:
- Outline each hex with subtle lines
- Highlight on hover
- Show movement/attack range with colored overlays

## Sprite Art Requirements

For 2.5D sprites:

**Units**:
- 8 directions (N, NE, E, SE, S, SW, W, NW) OR
- 4 directions (NE, SE, SW, NW) with flipping
- Multiple animation frames per direction
- Consistent lighting angle (top-left for isometric)

**Terrain**:
- Hex tiles fitting isometric grid
- Cliff edge sprites (straight, inside corner, outside corner)
- Transition tiles between terrain types
- Elevation variations (base, level 1, level 2, etc.)

**Effects**:
- Particles rendered as billboards (always face camera)
- Spell effects with consistent perspective

## Shaders

```rust
// Vertex shader: Transform to isometric space
fn vertex_main(position: Vec3, tex_coords: Vec2) -> VertexOutput {
    let iso_pos = apply_isometric_projection(position, projection_matrix);
    
    VertexOutput {
        position: iso_pos,
        tex_coords: tex_coords,
        elevation: position.z,
    }
}

// Fragment shader: Apply tint, shadows, effects
fn fragment_main(input: VertexOutput) -> Color {
    let base_color = sample_texture(input.tex_coords);
    
    // Elevation-based darkening (lower = darker)
    let elevation_tint = 1.0 - (input.elevation * 0.05);
    
    // Apply fog of war
    let fog_amount = sample_fog_map(input.world_pos);
    
    base_color * elevation_tint * fog_amount
}
```

**Custom shaders**:
- Water ripple (offset UV coordinates)
- Fire/glow (additive blending + animation)
- Outline (edge detection for selection)
- Fog of war (darken/desaturate unexplored areas)

## Lighting (Optional Advanced Feature)

Dynamic lighting in 2.5D:

```rust
struct Light {
    position: Vec3,
    color: Color,
    intensity: f32,
    radius: f32,
}

fn apply_lighting(sprite_pos: Vec3, lights: &[Light]) -> Color {
    let mut total_light = Color::BLACK;
    
    for light in lights {
        let distance = (light.position - sprite_pos).length();
        if distance < light.radius {
            let attenuation = 1.0 - (distance / light.radius).powi(2);
            total_light += light.color * light.intensity * attenuation;
        }
    }
    
    total_light.clamp(0.0, 1.0)
}
```

**Use cases**:
- Torch light in dark dungeons
- Spell effects casting light
- Day/night cycle
- Fire/lava glow

## Performance Targets

- **60 FPS** on mid-range hardware
- **100+ units** on screen with elevation
- **50x50 hex map** with multi-level terrain
- **Smooth depth sorting** (< 2ms per frame)

### Optimization Strategies

1. **Spatial partitioning**: Only sort sprites in visible chunks
2. **Frustum culling**: Don't render off-screen objects
3. **Occlusion culling**: Higher terrain blocks lower objects (optional)
4. **Instancing**: Batch identical sprites at different positions
5. **Level-of-detail**: Simpler sprites for distant objects
6. **Pre-sorted layers**: Static terrain doesn't need per-frame sorting

## Debug Rendering

```rust
struct IsometricDebugger {
    show_hex_coords: bool,
    show_elevation_numbers: bool,
    show_depth_order: bool,       // Color-code by render order
    show_bounding_boxes: bool,
    show_shadows: bool,
    wireframe_mode: bool,
}
```

**Visualizations**:
- Elevation heat map (color by height)
- Depth sort order (numbers on sprites)
- Hex grid overlay
- Shadow direction indicators

## Accessibility

- **Camera rotation**: Allow 90° increments for different view angles
- **Elevation indicators**: Visual markers for height differences
- **High contrast mode**: Stronger outlines, clearer shadows
- **Zoom range**: 0.5x to 3.0x for better visibility

## Platform Considerations

- **Mobile**: Larger touch targets, pinch-to-zoom, two-finger pan
- **Desktop**: Mouse-based camera, hover tooltips, scroll-wheel zoom
- **Console**: Gamepad cursor, snap-to-unit navigation

## Example Art Pipeline

1. **3D modeling**: Create unit in Blender
2. **Render**: Export 8-direction sprite sheets at isometric angle
3. **Post-process**: Add outlines, optimize palette
4. **Import**: Load sprite sheet with animation frames
5. **In-engine**: Render with depth sorting and shadows

OR use real-time 3D with orthographic camera for dynamic angles/animations.

## Future Extensions

- **Dynamic weather**: Rain, snow, fog affecting visibility
- **Destructible terrain**: Cliffs crumble, walls collapse
- **Volumetric effects**: Smoke, magic clouds
- **Normal mapping**: Fake lighting detail on sprites
- **Parallax backgrounds**: Multi-layer scrolling for depth
