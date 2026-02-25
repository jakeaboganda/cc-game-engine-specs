# Animation System (2.5D Isometric)

## Overview

Sprite animation and tweening system for 2.5D isometric tactical games. Emphasis on directional animations, depth sorting, and elevation-aware movement.

## Animation Types

### 1. Isometric Sprite Animation

Frame-by-frame animation with directional support:

```rust
struct IsometricSpriteAnimation {
    directions: DirectionSet,
    frames_per_direction: Vec<TextureHandle>,
    frame_duration: f32,     // seconds per frame
    loop_mode: LoopMode,
    current_direction: Direction,
    current_frame: usize,
    elapsed: f32,
}

enum DirectionSet {
    Single,            // No direction (particles, effects)
    Four,              // N, E, S, W (with mirroring for NE, SE, SW, NW)
    Eight,             // Full 8 directions (no mirroring needed)
}

enum Direction {
    North,      // 0°
    NorthEast,  // 45°
    East,       // 90°
    SouthEast,  // 135°
    South,      // 180°
    SouthWest,  // 225°
    West,       // 270°
    NorthWest,  // 315°
}

impl Direction {
    fn from_angle(radians: f32) -> Self {
        let degrees = radians.to_degrees().rem_euclid(360.0);
        match degrees as i32 {
            0..=22 | 338..=360 => Direction::North,
            23..=67 => Direction::NorthEast,
            68..=112 => Direction::East,
            113..=157 => Direction::SouthEast,
            158..=202 => Direction::South,
            203..=247 => Direction::SouthWest,
            248..=292 => Direction::West,
            293..=337 => Direction::NorthWest,
            _ => Direction::North,
        }
    }
    
    fn to_sprite_index(&self, use_mirroring: bool) -> (usize, bool) {
        // Returns (sprite_index, should_flip_horizontal)
        match (self, use_mirroring) {
            (Direction::NorthEast, true) => (0, false),   // NE sprite
            (Direction::East, true) => (1, false),         // E sprite
            (Direction::SouthEast, true) => (2, false),    // SE sprite
            (Direction::South, true) => (3, false),        // S sprite
            (Direction::SouthWest, true) => (2, true),     // SE sprite flipped
            (Direction::West, true) => (1, true),          // E sprite flipped
            (Direction::NorthWest, true) => (0, true),     // NE sprite flipped
            (Direction::North, true) => (3, false),        // Use S facing backwards (or separate N)
            
            // 8-direction (no flipping)
            (Direction::North, false) => (0, false),
            (Direction::NorthEast, false) => (1, false),
            (Direction::East, false) => (2, false),
            (Direction::SouthEast, false) => (3, false),
            (Direction::South, false) => (4, false),
            (Direction::SouthWest, false) => (5, false),
            (Direction::West, false) => (6, false),
            (Direction::NorthWest, false) => (7, false),
        }
    }
}
```

**Common animations** (per direction):
- Idle: Gentle breathing, slight movement
- Walk/run: Stepping animation (4-8 frames)
- Attack: Wind-up → strike → recovery (6-10 frames)
- Hit: Flinch reaction (2-4 frames)
- Death: Collapse (6-12 frames, ends with corpse sprite)
- Spell cast: Charging → release (8-16 frames)
- Jump/Fall: For elevation changes

**Art asset organization**:
```
sprites/
  warrior/
    idle/
      NE_0.png, NE_1.png, NE_2.png, NE_3.png
      E_0.png, E_1.png, E_2.png, E_3.png
      SE_0.png, SE_1.png, SE_2.png, SE_3.png
      S_0.png, S_1.png, S_2.png, S_3.png
    walk/
      NE_0.png...NE_7.png
      E_0.png...E_7.png
      ...
    attack/
      NE_0.png...NE_9.png
      ...
```

### 2. Tweening (Interpolation)

Smooth property changes over time:

```rust
struct Tween {
    start_value: f32,
    end_value: f32,
    duration: f32,
    elapsed: f32,
    easing: EasingFunction,
}

enum EasingFunction {
    Linear,
    EaseIn,         // Slow start, fast end
    EaseOut,        // Fast start, slow end
    EaseInOut,      // Slow both ends
    Bounce,         // Spring-like overshoot
    Elastic,        // Rubber-band effect
}
```

**Tweenable properties**:
- Position (movement)
- Scale (grow/shrink effects)
- Rotation (spin, face direction)
- Alpha (fade in/out)
- Color/tint (damage flash, status effects)

### 3. Particle Effects

Procedural visual effects:

```rust
struct ParticleEmitter {
    position: Vec2,
    emission_rate: f32,         // particles per second
    particle_lifetime: f32,
    initial_velocity: Vec2,
    velocity_randomness: f32,
    gravity: Vec2,
    color_over_lifetime: Gradient,
    size_over_lifetime: Curve,
}
```

**Use cases**:
- Hit impact: Spark burst
- Magic spells: Glowing trails, explosions
- Healing: Rising sparkles
- Poison: Dripping green goo
- Fire: Flickering flames
- Smoke: Rising wisps

## Animation Controller (Isometric)

State machine for unit animations with directional support:

```rust
struct IsometricAnimationController {
    current_state: AnimationState,
    current_direction: Direction,
    animations: HashMap<(AnimationState, Direction), IsometricSpriteAnimation>,
    transitions: Vec<Transition>,
    direction_set: DirectionSet,  // 4-way or 8-way
}

enum AnimationState {
    Idle,
    Moving,
    Attacking,
    Casting,
    TakingDamage,
    Dying,
    Dead,
    Jumping,    // For elevation changes
    Falling,    // For being pushed/falling off cliffs
}

struct Transition {
    from: AnimationState,
    to: AnimationState,
    condition: TransitionCondition,
    blend_duration: f32,  // cross-fade time
}

impl IsometricAnimationController {
    fn get_current_animation(&self) -> &IsometricSpriteAnimation {
        let key = (self.current_state, self.current_direction);
        self.animations.get(&key)
            .or_else(|| {
                // Fallback to closest direction if exact match not found
                self.find_closest_direction_animation(self.current_state, self.current_direction)
            })
            .expect("Animation not found")
    }
    
    fn set_direction(&mut self, new_direction: Direction) {
        if self.current_direction != new_direction {
            self.current_direction = new_direction;
            // Reset animation to start with new direction
            if let Some(anim) = self.animations.get_mut(&(self.current_state, new_direction)) {
                anim.current_frame = 0;
                anim.elapsed = 0.0;
            }
        }
    }
    
    fn transition_to(&mut self, new_state: AnimationState) {
        if self.current_state != new_state {
            self.on_exit_state(self.current_state);
            self.current_state = new_state;
            self.on_enter_state(new_state);
        }
    }
    
    fn update(&mut self, dt: f32) {
        if let Some(anim) = self.animations.get_mut(&(self.current_state, self.current_direction)) {
            anim.elapsed += dt;
            
            if anim.elapsed >= anim.frame_duration {
                anim.elapsed = 0.0;
                anim.current_frame += 1;
                
                match anim.loop_mode {
                    LoopMode::Once => {
                        if anim.current_frame >= anim.frames_per_direction.len() {
                            anim.current_frame = anim.frames_per_direction.len() - 1;
                            self.on_animation_complete();
                        }
                    },
                    LoopMode::Loop => {
                        if anim.current_frame >= anim.frames_per_direction.len() {
                            anim.current_frame = 0;
                        }
                    },
                    // ... other loop modes
                }
            }
        }
    }
    
    fn on_animation_complete(&mut self) {
        // Auto-transition based on state
        match self.current_state {
            AnimationState::Attacking => self.transition_to(AnimationState::Idle),
            AnimationState::TakingDamage => self.transition_to(AnimationState::Idle),
            AnimationState::Dying => self.transition_to(AnimationState::Dead),
            AnimationState::Jumping => self.transition_to(AnimationState::Idle),
            AnimationState::Falling => self.transition_to(AnimationState::Idle),
            _ => {}
        }
    }
}
```

**Example flow**:
1. Unit idle (facing SE) → Player clicks to move NE → transition to Moving + update direction to NE
2. Moving (NE animation plays) → Reaches destination → transition to Idle (NE)
3. Idle → Takes damage → flash to TakingDamage (same direction) → back to Idle
4. Idle → Moves up cliff → Jumping animation → Idle at new elevation
5. Any state → HP = 0 → Dying → Dead (final frame is corpse sprite)

**Direction persistence**: When transitioning states, keep the same direction unless explicitly changed (e.g., idle warrior facing NE transitions to attack NE).

## Movement Animation (Isometric 3D)

### Hex-to-Hex Movement with Elevation

Smooth movement between hexes in isometric space:

```rust
struct IsometricMovementAnimation {
    path: Vec<HexPosition>,     // 3D path including elevation changes
    current_segment: usize,
    segment_progress: f32,       // 0.0 to 1.0
    speed: f32,                  // hexes per second
    current_direction: Direction,
    sprite_controller: AnimationController,
    jump_arc_height: f32,        // For elevation transitions
}

fn update_movement(anim: &mut IsometricMovementAnimation, dt: f32) {
    anim.segment_progress += dt * anim.speed;
    
    if anim.segment_progress >= 1.0 {
        anim.segment_progress = 0.0;
        anim.current_segment += 1;
        
        // Update facing direction for next segment
        if anim.current_segment < anim.path.len() - 1 {
            anim.update_direction();
        }
    }
    
    if anim.current_segment >= anim.path.len() - 1 {
        // Reached destination
        anim.sprite_controller.transition_to(AnimationState::Idle);
    }
}

impl IsometricMovementAnimation {
    fn current_position(&self) -> Vec3 {
        let start = &self.path[self.current_segment];
        let end = &self.path[self.current_segment + 1];
        
        // Convert to isometric screen space
        let start_screen = hex_to_iso_pixel(*start, &projection);
        let end_screen = hex_to_iso_pixel(*end, &projection);
        
        let t = ease_out(self.segment_progress);
        
        // Interpolate position
        let x = lerp(start_screen.x, end_screen.x, t);
        let y = lerp(start_screen.y, end_screen.y, t);
        
        // Add jump arc if changing elevation
        let elevation_diff = end.elevation - start.elevation;
        let jump_offset = if elevation_diff != 0 {
            self.calculate_jump_arc(t, elevation_diff)
        } else {
            0.0
        };
        
        Vec3::new(x, y - jump_offset, lerp(start.elevation as f32, end.elevation as f32, t))
    }
    
    fn calculate_jump_arc(&self, t: f32, elevation_change: i32) -> f32 {
        // Parabolic arc for jumps/falls
        if elevation_change > 0 {
            // Jumping up: arc peaks midway
            self.jump_arc_height * (4.0 * t * (1.0 - t))
        } else {
            // Falling: steeper descent
            let fall_curve = 1.0 - (1.0 - t).powi(2);
            -elevation_change.abs() as f32 * fall_curve * 10.0
        }
    }
    
    fn update_direction(&mut self) {
        let current = &self.path[self.current_segment];
        let next = &self.path[self.current_segment + 1];
        
        // Calculate 2D direction (ignoring elevation)
        let dx = next.hex.q - current.hex.q;
        let dy = next.hex.r - current.hex.r;
        let angle = (dy as f32).atan2(dx as f32);
        
        self.current_direction = Direction::from_angle(angle);
        
        // Update sprite animation direction
        self.sprite_controller.set_direction(self.current_direction);
    }
}
```

**Features**:
- Smooth easing (not linear)
- Automatic direction updates per segment
- Jump/fall arcs for elevation changes
- Pause on obstacles (if path becomes blocked mid-animation)
- Speed variations based on terrain
- Footstep particles aligned to isometric ground

**Elevation transition animations**:
- **Level terrain**: Standard walk/run cycle
- **Uphill**: Slower animation, lean forward slightly
- **Downhill**: Faster animation, lean back
- **Jumping up 1 level**: Jump animation with arc
- **Falling 2+ levels**: Freefall animation → landing impact

### Projectile Animation (3D)

### Projectile Animation (3D)

For ranged attacks with elevation:

```rust
struct ProjectileAnimation3D {
    start: HexPosition,
    end: HexPosition,
    arc_height: f32,         // Additional arc beyond elevation
    speed: f32,
    elapsed: f32,
    sprite: SpriteAnimation,
    trail_effect: Option<ParticleEmitter>,
    rotation_speed: f32,     // For spinning projectiles
}

fn projectile_position(anim: &ProjectileAnimation3D, t: f32) -> Vec3 {
    let start_screen = hex_to_iso_pixel(anim.start, &projection);
    let end_screen = hex_to_iso_pixel(anim.end, &projection);
    
    // Horizontal interpolation
    let x = lerp(start_screen.x, end_screen.x, t);
    let base_y = lerp(start_screen.y, end_screen.y, t);
    
    // Elevation interpolation
    let elevation = lerp(anim.start.elevation as f32, anim.end.elevation as f32, t);
    
    // Add parabolic arc (peaks at t=0.5)
    let arc_offset = anim.arc_height * (4.0 * t * (1.0 - t));
    
    Vec3::new(x, base_y - arc_offset, elevation)
}

impl ProjectileAnimation3D {
    fn calculate_flight_time(&self) -> f32 {
        // Account for 3D distance
        let distance = distance_3d(self.start, self.end);
        distance / self.speed
    }
    
    fn facing_angle(&self, t: f32) -> f32 {
        // Projectile rotates to face flight direction
        let next_t = (t + 0.01).min(1.0);
        let current_pos = self.projectile_position(t);
        let next_pos = self.projectile_position(next_t);
        
        (next_pos.y - current_pos.y).atan2(next_pos.x - current_pos.x)
    }
}
```

**Projectile types**:
- **Arrow**: Fast, low arc, points toward target
  - Adjusts angle based on elevation difference
  - Uphill shots arc higher, downhill shots arc lower
  
- **Fireball**: Medium speed, moderate arc, leaves flame trail
  - Trail particles rendered at each frame position
  - Explosion effect on impact (spherical AoE)
  
- **Lightning**: Instant hit with branching effect
  - No arc, straight line from caster to target
  - Branch particles spawn along path
  - Accounts for elevation (can hit units on different levels)
  
- **Thrown weapon**: Slow, high arc, spinning
  - Rotates around center during flight
  - Clinks on ground if misses (physics-based bounce)

- **Meteor**: Falls from above, impacts ground
  - Starts at elevation + 10, falls to target
  - Acceleration (gravity simulation)
  - Large impact crater effect

## Combat Animations

### Attack Sequence

```rust
struct AttackAnimation {
    stages: Vec<AttackStage>,
    current_stage: usize,
}

struct AttackStage {
    duration: f32,
    animation: AnimationType,
    on_complete: Option<Callback>,
}

// Example: Melee attack
let attack = AttackAnimation {
    stages: vec![
        AttackStage {
            duration: 0.2,
            animation: AnimationType::WindUp,  // Pull back weapon
            on_complete: None,
        },
        AttackStage {
            duration: 0.1,
            animation: AnimationType::Strike,  // Swing forward
            on_complete: Some(deal_damage),    // Damage on impact
        },
        AttackStage {
            duration: 0.3,
            animation: AnimationType::Recovery, // Return to stance
            on_complete: None,
        },
    ],
    current_stage: 0,
};
```

### Damage Flash

Visual feedback for taking damage:

```rust
fn apply_damage_flash(sprite: &mut Sprite, damage: i32) {
    let flash_color = Color::rgb(1.0, 0.5, 0.5);  // Red tint
    let flash_duration = 0.15;
    
    tween_color(
        sprite,
        sprite.tint,
        flash_color,
        flash_duration,
        EasingFunction::Linear,
    );
    
    // Then fade back to normal
    after(flash_duration, || {
        tween_color(
            sprite,
            flash_color,
            Color::WHITE,
            0.15,
            EasingFunction::EaseOut,
        );
    });
}
```

**Additional feedback**:
- Shake unit sprite
- Pop-up damage number
- Screen shake (for big hits)
- Particles (blood, sparks, magic)

## UI Animations

### Menu Transitions

```rust
enum MenuAnimation {
    SlideIn { from: Direction, duration: f32 },
    FadeIn { duration: f32 },
    Scale { from_scale: f32, duration: f32 },
}

fn animate_menu(menu: &mut UIPanel, anim: MenuAnimation) {
    match anim {
        MenuAnimation::SlideIn { from, duration } => {
            let offset = match from {
                Direction::Left => Vec2::new(-menu.width, 0.0),
                Direction::Right => Vec2::new(menu.width, 0.0),
                // ...
            };
            tween_position(menu, menu.position + offset, menu.position, duration);
        },
        // ...
    }
}
```

### Button Hover/Click

```rust
struct ButtonAnimation {
    idle_scale: f32,
    hover_scale: f32,
    click_scale: f32,
}

fn on_hover_start(button: &mut Button) {
    tween_scale(button, button.scale, 1.1, 0.1, EasingFunction::EaseOut);
}

fn on_click(button: &mut Button) {
    tween_scale(button, button.scale, 0.95, 0.05, EasingFunction::EaseIn);
    after(0.05, || {
        tween_scale(button, 0.95, 1.0, 0.1, EasingFunction::Bounce);
    });
}
```

## Timing & Sequencing

### Animation Queue

Chain animations sequentially:

```rust
struct AnimationSequence {
    steps: VecDeque<AnimationStep>,
}

enum AnimationStep {
    PlayAnimation(SpriteAnimation),
    Tween(Tween),
    Wait(f32),
    Parallel(Vec<AnimationStep>),  // Play multiple at once
    Callback(Box<dyn Fn()>),
}

// Example: Move → Attack → Return
let sequence = AnimationSequence::new()
    .then(move_to(target_hex))
    .then(play_animation(AttackAnimation))
    .then(spawn_particles(hit_effect))
    .wait(0.2)
    .then(move_to(original_hex));
```

### Timeline System

For complex choreography:

```rust
struct Timeline {
    tracks: Vec<Track>,
    current_time: f32,
}

struct Track {
    entity: EntityId,
    keyframes: Vec<Keyframe>,
}

struct Keyframe {
    time: f32,
    property: Property,
    value: Value,
    easing: EasingFunction,
}

// Example: Multi-unit ability
timeline.add_keyframe(caster, 0.0, Property::Animation, "cast_start");
timeline.add_keyframe(caster, 0.5, Property::Animation, "cast_release");
timeline.add_keyframe(projectile, 0.5, Property::Position, spell_target);
timeline.add_keyframe(target1, 1.0, Property::TakeDamage, 50);
timeline.add_keyframe(target2, 1.0, Property::TakeDamage, 50);
timeline.add_keyframe(camera, 0.5, Property::Shake, 0.3);
```

## Camera Effects

### Screen Shake

```rust
struct ScreenShake {
    intensity: f32,
    duration: f32,
    frequency: f32,
    decay: f32,
}

fn apply_screen_shake(camera: &mut Camera, intensity: f32) {
    let offset = random_direction() * intensity;
    camera.position += offset;
    
    // Gradually reduce intensity
    intensity *= 0.9;
}
```

**Use cases**:
- Light shake: Normal hit
- Heavy shake: Critical hit, explosion
- Continuous shake: Earthquake, boss stomp

### Camera Zoom

```rust
fn focus_on_action(camera: &mut Camera, position: Vec2) {
    // Smooth pan to action
    tween_position(camera, camera.position, position, 0.5);
    
    // Slight zoom in
    tween_zoom(camera, camera.zoom, 1.2, 0.3);
    
    // Zoom back out after action
    after(2.0, || {
        tween_zoom(camera, 1.2, 1.0, 0.5);
    });
}
```

## Performance

### Optimization Strategies

1. **Object pooling**: Reuse animation instances
2. **Culling**: Don't update off-screen animations
3. **LOD**: Simpler animations at distance
4. **Batching**: Group similar particle effects
5. **Throttling**: Skip frames for background effects

```rust
struct AnimationManager {
    active_animations: Vec<Animation>,
    animation_pool: Vec<Animation>,
    cull_distance: f32,
}

fn update_animations(&mut self, camera: &Camera, dt: f32) {
    for anim in &mut self.active_animations {
        if camera.can_see(anim.position, self.cull_distance) {
            anim.update(dt);
        } else {
            // Skip or update at lower rate
            anim.update_slow(dt * 0.1);
        }
    }
}
```

## Accessibility

- **Speed control**: Global animation speed multiplier
- **Reduce motion**: Option to disable non-essential animations
- **Flash reduction**: Tone down intense effects
- **Skip animations**: Fast-forward combat (keep logic, skip visuals)

```rust
struct AnimationSettings {
    speed: f32,              // 0.5 = half speed, 2.0 = double
    reduce_motion: bool,     // Disable screen shake, particles
    skip_combat: bool,       // Instant resolution
}
```

## Debug Tools

- **Animation inspector**: View current state, transitions
- **Timeline scrubber**: Manually advance/rewind animations
- **Hitbox visualization**: Show attack/damage frames
- **Performance overlay**: Animation count, frame time
