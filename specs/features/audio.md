# Audio System

## Overview

Audio playback for music, sound effects, and voice (if applicable). Support spatial audio, mixing, and dynamic music systems.

## Audio Architecture

```rust
struct AudioEngine {
    music_player: MusicPlayer,
    sfx_player: SfxPlayer,
    mixer: AudioMixer,
    settings: AudioSettings,
}

struct AudioSettings {
    master_volume: f32,     // 0.0 to 1.0
    music_volume: f32,
    sfx_volume: f32,
    voice_volume: f32,
    mute: bool,
}
```

## Audio Categories

### 1. Music

Background music with looping and transitions:

```rust
struct MusicPlayer {
    current_track: Option<MusicTrack>,
    next_track: Option<MusicTrack>,
    fade_duration: f32,
    loop_mode: MusicLoopMode,
}

struct MusicTrack {
    file_path: String,
    intro: Option<AudioSegment>,  // Play once before loop
    loop_segment: AudioSegment,    // Repeated section
    outro: Option<AudioSegment>,   // Plays on stop
}

enum MusicLoopMode {
    Loop,              // Seamless loop
    LoopWithIntro,     // Intro → loop
    Sequential,        // Next track when done
}
```

**Music transitions**:
- **Cross-fade**: Fade out current, fade in next
- **Hard cut**: Instant switch
- **Stinger**: Short transition sound between tracks
- **Layer-based**: Add/remove instrument layers dynamically

**Example**:
```toml
# content/audio/music/combat.toml
[track]
name = "Battle Theme"
intro = "audio/music/combat_intro.ogg"
loop = "audio/music/combat_loop.ogg"
bpm = 140
loop_start = 4.8    # seconds
loop_end = 28.8
```

### 2. Sound Effects

Short, one-shot sounds:

```rust
struct SfxPlayer {
    channels: Vec<AudioChannel>,
    max_simultaneous: usize,  // Limit polyphony
}

struct SoundEffect {
    file_path: String,
    volume: f32,
    pitch_variation: f32,     // Randomize pitch ±%
    cooldown: f32,            // Prevent spam
    priority: Priority,
}

enum Priority {
    Critical,   // Always play (UI clicks, important events)
    High,       // Usually play (attacks, abilities)
    Medium,     // Can be dropped if too many sounds
    Low,        // Background, ambient
}
```

**Common SFX**:
- UI: Click, hover, open/close menu
- Movement: Footsteps, running
- Combat: Sword swing, hit, block, dodge
- Abilities: Cast, impact, status effect applied
- Environmental: Door open, chest unlock, item pickup

**Variations**:
```rust
struct SfxVariations {
    sounds: Vec<String>,  // ["hit1.ogg", "hit2.ogg", "hit3.ogg"]
    mode: VariationMode,
}

enum VariationMode {
    Random,      // Pick random each time
    Sequential,  // Cycle through list
    RoundRobin,  // Avoid immediate repeats
}
```

### 3. Spatial Audio

3D positioning for immersion (even in 2D games):

```rust
struct SpatialSound {
    position: Vec2,          // World position
    max_distance: f32,       // Falloff distance
    attenuation: Attenuation,
}

enum Attenuation {
    Linear,
    Exponential,
    InverseSquare,  // Realistic physics
}

fn calculate_volume(
    sound_pos: Vec2,
    listener_pos: Vec2,
    max_distance: f32,
) -> f32 {
    let distance = (sound_pos - listener_pos).length();
    if distance > max_distance {
        return 0.0;
    }
    1.0 - (distance / max_distance).powf(2.0)
}
```

**Use cases**:
- Footsteps from off-screen units
- Environmental sounds (waterfall, fire)
- Combat sounds positioned at combatants

### 4. Voice/Dialogue (Optional)

Character voice lines:

```rust
struct VoiceLine {
    character_id: String,
    line_id: String,
    audio_file: String,
    subtitle: String,
    duration: f32,
}

struct DialoguePlayer {
    current_line: Option<VoiceLine>,
    queue: VecDeque<VoiceLine>,
    interrupt_priority: bool,
}
```

**Dialogue system**:
- Auto-advance after line completes
- Skip to next line
- Subtitles always visible
- Interrupts lower-priority lines

## Audio Mixer

Multi-channel mixing with ducking:

```rust
struct AudioMixer {
    buses: HashMap<String, AudioBus>,
}

struct AudioBus {
    name: String,
    volume: f32,
    parent: Option<String>,  // Hierarchical mixing
    effects: Vec<AudioEffect>,
}

enum AudioEffect {
    Reverb { room_size: f32, damping: f32 },
    LowPass { cutoff_freq: f32 },
    HighPass { cutoff_freq: f32 },
    Compression { threshold: f32, ratio: f32 },
}
```

**Bus hierarchy**:
```
Master
├─ Music
├─ SFX
│  ├─ Combat
│  ├─ UI
│  └─ Ambient
└─ Voice
```

**Ducking**: Lower music volume when voice plays:
```rust
fn on_voice_start(&mut self) {
    self.mixer.duck_bus("Music", 0.3, 0.5);  // Drop to 30% over 0.5s
}

fn on_voice_end(&mut self) {
    self.mixer.unduck_bus("Music", 1.0);  // Restore to 100%
}
```

## Dynamic Music

Adaptive music that responds to gameplay:

### Layer System

```rust
struct DynamicMusic {
    base_track: MusicTrack,
    layers: HashMap<String, MusicLayer>,
}

struct MusicLayer {
    audio: AudioHandle,
    active: bool,
    fade_time: f32,
}

// Example: Combat intensity
fn update_combat_music(&mut self, intensity: f32) {
    if intensity > 0.7 {
        self.activate_layer("drums");
        self.activate_layer("brass");
    } else if intensity > 0.3 {
        self.activate_layer("drums");
        self.deactivate_layer("brass");
    } else {
        self.deactivate_layer("drums");
        self.deactivate_layer("brass");
    }
}
```

**States**:
- Exploration: Ambient, soft
- Combat start: Add percussion
- Combat high stakes: Full orchestra
- Victory: Triumphant stinger
- Defeat: Somber

### Horizontal Re-sequencing

Switch music segments based on events:

```rust
struct SegmentedMusic {
    segments: HashMap<GameState, MusicSegment>,
    current_segment: GameState,
}

fn on_game_state_change(&mut self, new_state: GameState) {
    if let Some(segment) = self.segments.get(&new_state) {
        // Wait for musical bar to finish, then transition
        self.queue_transition(segment, TransitionMode::OnBar);
    }
}
```

## Audio Events

Event-driven sound triggering:

```rust
enum AudioEvent {
    UnitMove(EntityId),
    UnitAttack(EntityId, EntityId),
    UnitTakeDamage(EntityId, i32),
    AbilityUsed(AbilityId),
    UIClick,
    MenuOpen,
    GameStart,
    Victory,
    Defeat,
}

struct AudioEventHandler {
    mappings: HashMap<AudioEvent, Vec<SoundEffect>>,
}

fn trigger_event(&mut self, event: AudioEvent) {
    if let Some(sounds) = self.mappings.get(&event) {
        for sound in sounds {
            self.play_sfx(sound);
        }
    }
}
```

**Data-driven mapping**:
```toml
# content/audio/events.toml
[[event]]
trigger = "unit_attack"
sound = "audio/sfx/sword_swing.ogg"
volume = 0.8
pitch_variation = 0.1

[[event]]
trigger = "unit_attack"
unit_type = "mage"
sound = "audio/sfx/staff_whoosh.ogg"
```

## Asset Management

Lazy loading and streaming:

```rust
struct AudioAssetManager {
    loaded: HashMap<String, AudioBuffer>,
    streaming: HashMap<String, StreamingSource>,
}

enum AudioLoadMode {
    Preload,    // Load entire file into memory
    Stream,     // Stream from disk (for music)
}
```

**Loading strategy**:
- **Music**: Stream (large files)
- **Common SFX**: Preload (low latency)
- **Rare SFX**: Load on-demand

**File formats**:
- **OGG Vorbis**: Compressed, good quality (music, voice)
- **WAV**: Uncompressed, instant playback (short SFX)
- **MP3**: Alternative compression (compatibility)

## Performance

### Polyphony Limiting

Prevent audio overload:

```rust
fn play_sfx(&mut self, sound: SoundEffect) -> Option<AudioHandle> {
    if self.active_sounds.len() >= self.max_simultaneous {
        // Drop lowest priority sound
        self.stop_lowest_priority();
    }
    
    self.internal_play(sound)
}
```

**Limits**:
- UI sounds: 5 simultaneous
- Combat SFX: 16 simultaneous
- Total: 32 active sounds

### Distance Culling

Don't play sounds too far away:

```rust
fn should_play_spatial_sound(
    sound_pos: Vec2,
    listener_pos: Vec2,
    max_distance: f32,
) -> bool {
    let distance = (sound_pos - listener_pos).length();
    distance <= max_distance
}
```

## Accessibility

- **Subtitles**: Always show for voice/important sounds
- **Visual cues**: Icon indicators for off-screen sounds
- **Mono mode**: Collapse stereo for hearing-impaired
- **Audio descriptions**: Optional narration of visual events

```rust
struct AudioAccessibility {
    subtitles_enabled: bool,
    visual_sound_indicators: bool,
    mono_output: bool,
    screen_reader_support: bool,
}
```

## Platform Considerations

### Audio Backend

Use abstraction layer for cross-platform:
- **Windows**: XAudio2 or WASAPI
- **Linux**: PulseAudio or ALSA
- **macOS**: CoreAudio
- **Web**: Web Audio API

**Rust libraries**:
- **rodio**: Simple, cross-platform
- **kira**: Game-focused, good features
- **oddio**: Spatial audio support
- **cpal**: Low-level, max control

## Debug Tools

- **Audio inspector**: View active sounds, volumes, positions
- **Mute individual buses**: Test mix balance
- **Visualize spatial sounds**: Draw sound radius on map
- **Performance stats**: Active sounds, memory usage, CPU %

```rust
struct AudioDebugger {
    show_active_sounds: bool,
    show_spatial_ranges: bool,
    mute_buses: HashSet<String>,
    volume_meters: bool,
}
```

## Example Audio Config

```toml
# config/audio.toml
[settings]
master_volume = 1.0
music_volume = 0.7
sfx_volume = 0.9

[mixer]
max_simultaneous_sfx = 32
voice_ducking_amount = 0.3
voice_ducking_speed = 0.5

[spatial]
max_distance = 15.0  # hexes
attenuation = "inverse_square"

[music]
fade_duration = 2.0
loop_mode = "seamless"

[[buses]]
name = "Music"
volume = 1.0
effects = [
    { type = "reverb", room_size = 0.3 }
]

[[buses]]
name = "Combat_SFX"
parent = "SFX"
volume = 1.0
effects = [
    { type = "compression", threshold = -6.0, ratio = 4.0 }
]
```
