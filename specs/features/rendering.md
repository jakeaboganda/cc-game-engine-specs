# Rendering System

## Overview

2D rendering focused on tactical hex-based games with support for layered sprites, animations, and effects.

## Design Choice: Renderer Abstraction

The engine should support multiple rendering backends through a trait-based system:

```rust
trait Renderer {
    fn render_sprite(&mut self, sprite: &Sprite, transform: Transform);
    fn render_hex_grid(&mut self, grid: &HexGrid, camera: &Camera);
    fn render_text(&mut self, text: &str, font: &Font, position: Vec2);
    fn render_effect(&mut self, effect: &ParticleEffect);
    fn present(&mut self);
}
```

**Primary backend**: wgpu (cross-platform, modern, Rust-native)
**Alternative**: Bevy renderer (if using Bevy ECS)

## Rendering Pipeline

### Layer System

Render order (back to front):
1. **Background** - Skybox, far background
2. **Terrain** - Hex tiles, ground textures
3. **Terrain Features** - Trees, rocks, water
4. **Ground Effects** - AoE indicators, selection highlights
5. **Units** - Characters, enemies
6. **Unit Effects** - Status icons, health bars
7. **Projectiles** - Arrows, spells in flight
8. **Particles** - Explosions, magic effects
9. **UI Overlays** - Turn indicators, range displays
10. **UI** - Menus, dialogs, HUD

Each layer has a z-index for fine control within that layer.

### Sprite System

```rust
struct Sprite {
    texture: TextureHandle,
    source_rect: Option<Rect>,  // for sprite sheets
    tint: Color,
    flip_x: bool,
    flip_y: bool,
}

struct SpriteSheet {
    texture: TextureHandle,
    frame_width: u32,
    frame_height: u32,
    frames: Vec<Rect>,
}
```

**Features**:
- Sprite batching for performance (batch identical textures)
- Automatic atlasing for small sprites
- 9-slice scaling for UI panels
- Multi-sprite rendering (composite units with equipment)

### Hex Grid Rendering

```rust
struct HexRenderer {
    tile_radius: f32,
    flat_top: bool,  // vs pointy-top
}
```

**Rendering modes**:
1. **Textured** - Each hex has terrain sprite
2. **Colored** - Flat colors with borders
3. **Procedural** - Generated textures based on terrain type
4. **Mixed** - Blend textures at borders (grass → sand transition)

**Grid overlays**:
- Movement range (blue highlight)
- Attack range (red highlight)
- AoE preview (semi-transparent danger zone)
- Pathfinding visualization (arrow trail)

### Coordinate Conversion

```rust
// Hex (cube coords) to screen pixels
fn hex_to_pixel(hex: HexCoord, radius: f32) -> Vec2 {
    let x = radius * (3.0_f32.sqrt() * hex.q as f32 
                      + 3.0_f32.sqrt() / 2.0 * hex.r as f32);
    let y = radius * (3.0 / 2.0 * hex.r as f32);
    Vec2::new(x, y)
}

// Screen pixels to hex coords (for mouse picking)
fn pixel_to_hex(pos: Vec2, radius: f32) -> HexCoord {
    let q = (pos.x * 3.0_f32.sqrt() / 3.0 - pos.y / 3.0) / radius;
    let r = pos.y * 2.0 / 3.0 / radius;
    hex_round(q, r)
}
```

## Camera System

```rust
struct Camera {
    position: Vec2,        // world position
    zoom: f32,             // 1.0 = normal, 2.0 = 2x zoom
    rotation: f32,         // radians (typically 0 for top-down)
    viewport: Rect,        // screen area to render to
}
```

### Camera Features

**Movement**:
- Pan (WASD or edge scrolling)
- Zoom (mouse wheel, pinch-to-zoom)
- Snap to unit
- Follow selected unit
- Smooth interpolation (ease in/out)

**Multi-camera** (for split-screen co-op):
```rust
struct CameraManager {
    cameras: Vec<Camera>,
    layout: CameraLayout,
}

enum CameraLayout {
    Single,                    // Full screen
    Horizontal2,               // Top/bottom split
    Vertical2,                 // Left/right split
    Quad,                      // 4-way split
    Custom(Vec<Rect>),         // Arbitrary layouts
}
```

**Culling**:
- Only render hexes visible in camera frustum
- Cull sprites outside view
- Distance-based LOD (optional, for large sprites)

## Shaders

Basic shader pipeline:

1. **Sprite shader**: Standard textured quad with tint
2. **Hex outline shader**: Draw hex borders
3. **Highlight shader**: Animated glow for selection
4. **Fog of war shader**: Darken/desaturate unexplored areas
5. **Status effect shader**: Color overlays (poisoned = green tint)

Custom shaders for special effects:
- Ripple effect for water
- Fire/glow animation
- Screen shake on impact
- Flash on damage

## Performance Targets

- **60 FPS** on mid-range hardware (5-year-old laptop)
- **1000+ sprites** on screen simultaneously
- **50x50 hex grid** rendered with no frame drops

### Optimization Strategies

1. **Batching**: Group draw calls by texture
2. **Culling**: Don't render off-screen objects
3. **Dirty rectangles**: Only redraw changed areas (for static grids)
4. **Texture atlases**: Combine small textures
5. **Instancing**: Render many identical sprites in one call
6. **Asset streaming**: Load/unload textures based on camera position

## Debug Rendering

Toggle-able debug overlays:
- Hex coordinate labels
- FPS counter
- Draw call count
- Culling visualization (show frustum)
- Collision bounds
- Pathfinding costs (color-coded)
- LOS rays

```rust
struct DebugRenderer {
    show_hex_coords: bool,
    show_fps: bool,
    show_bounds: bool,
    show_pathfinding: bool,
}
```

## Resolution & Scaling

Support multiple resolutions:
- **Native rendering**: Pick target resolution (1920x1080, etc.)
- **Scaling modes**:
  - Pixel-perfect (integer scaling for retro look)
  - Smooth (bilinear/trilinear filtering)
  - Adaptive (scale to fit window, maintain aspect ratio)

**HiDPI support**: Detect and render at native resolution on retina displays.

## Accessibility

- **Color-blind modes**: Alternative color palettes for team/range indicators
- **High contrast mode**: Stronger outlines, bolder colors
- **Zoom limits**: Allow zooming closer for better visibility
- **Icon clarity**: Large, clear icons with text labels

## Future Extensions

- **Lighting system**: Dynamic lights, shadows from tall units
- **Weather effects**: Rain, fog, snow overlays
- **3D isometric**: Transition to 2.5D while keeping hex grid
- **Shader effects**: More advanced post-processing (bloom, chromatic aberration)
