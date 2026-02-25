# Animation System

## Overview

2D sprite animation and tweening system for tactical games. Emphasis on clarity, readability, and smooth transitions.

## Animation Types

### 1. Sprite Animation

Traditional frame-by-frame animation:

```rust
struct SpriteAnimation {
    frames: Vec<TextureHandle>,
    frame_duration: f32,     // seconds per frame
    loop_mode: LoopMode,
    current_frame: usize,
    elapsed: f32,
}

enum LoopMode {
    Once,           // Play once, stop on last frame
    Loop,           // Repeat forever
    PingPong,       // Forward then backward
    LoopN(u32),     // Loop N times then stop
}
```

**Common animations**:
- Idle: Gentle breathing, slight movement
- Walk/run: Stepping animation
- Attack: Wind-up → strike → recovery
- Hit: Flinch reaction
- Death: Collapse
- Spell cast: Charging → release

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

## Animation Controller

State machine for unit animations:

```rust
struct AnimationController {
    current_state: AnimationState,
    animations: HashMap<AnimationState, SpriteAnimation>,
    transitions: Vec<Transition>,
}

enum AnimationState {
    Idle,
    Moving,
    Attacking,
    Casting,
    TakingDamage,
    Dying,
    Dead,
}

struct Transition {
    from: AnimationState,
    to: AnimationState,
    condition: TransitionCondition,
    blend_duration: f32,  // cross-fade time
}
```

**Example flow**:
1. Unit idle → Player clicks to move → transition to Moving
2. Moving → Reaches destination → transition to Idle
3. Idle → Takes damage → flash to TakingDamage → back to Idle
4. Any state → HP = 0 → Dying → Dead (stop)

## Movement Animation

### Hex-to-Hex Movement

Smooth movement between hexes:

```rust
struct MovementAnimation {
    path: Vec<HexCoord>,
    current_segment: usize,
    segment_progress: f32,  // 0.0 to 1.0
    speed: f32,             // hexes per second
}

fn update_movement(anim: &mut MovementAnimation, dt: f32) {
    anim.segment_progress += dt * anim.speed;
    
    if anim.segment_progress >= 1.0 {
        anim.segment_progress = 0.0;
        anim.current_segment += 1;
    }
    
    if anim.current_segment >= anim.path.len() - 1 {
        // Reached destination
    }
}

fn current_position(anim: &MovementAnimation) -> Vec2 {
    let start = hex_to_pixel(anim.path[anim.current_segment]);
    let end = hex_to_pixel(anim.path[anim.current_segment + 1]);
    lerp(start, end, ease_out(anim.segment_progress))
}
```

**Features**:
- Smooth easing (not linear)
- Face direction of movement
- Pause on obstacles (if path becomes blocked)
- Speed up/down based on terrain

### Projectile Animation

For ranged attacks:

```rust
struct ProjectileAnimation {
    start: Vec2,
    end: Vec2,
    arc_height: f32,       // Parabolic arc
    speed: f32,
    sprite: SpriteAnimation,
    trail_effect: Option<ParticleEmitter>,
}

fn projectile_position(anim: &ProjectileAnimation, t: f32) -> Vec2 {
    let x = lerp(anim.start.x, anim.end.x, t);
    let y = lerp(anim.start.y, anim.end.y, t);
    
    // Add vertical arc
    let arc = anim.arc_height * (4.0 * t * (1.0 - t));  // Parabola peaks at t=0.5
    
    Vec2::new(x, y - arc)
}
```

**Variants**:
- **Arrow**: Fast, shallow arc
- **Fireball**: Medium speed, flaming trail
- **Lightning**: Instant hit with branching effect
- **Thrown weapon**: Spinning rotation

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
