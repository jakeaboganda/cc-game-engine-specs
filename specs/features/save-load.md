# Save/Load System

## Overview

Persistent game state for mid-combat saves, campaign progress, and settings. Support multiple save slots and cloud sync (future).

## Architecture

```rust
struct SaveManager {
    save_directory: PathBuf,      // e.g., ~/.cc-game-engine/saves/
    current_save_slot: Option<u32>,
    auto_save_enabled: bool,
    auto_save_interval: f32,       // seconds
}

struct SaveFile {
    metadata: SaveMetadata,
    game_state: GameState,
    session_state: SessionState,
}

struct SaveMetadata {
    version: String,               // Game version
    timestamp: SystemTime,
    slot_id: u32,
    scenario_name: String,
    turn_number: u32,
    playtime: Duration,
    thumbnail: Option<Vec<u8>>,    // Screenshot
}
```

## Save File Format

### RON (Rust Object Notation)

Human-readable, Rust-native:

```ron
SaveFile(
    metadata: SaveMetadata(
        version: "0.1.0",
        timestamp: SystemTime(...),
        slot_id: 1,
        scenario_name: "Forest Ambush",
        turn_number: 5,
        playtime: Duration(seconds: 342),
    ),
    game_state: GameState(
        entities: [
            Entity(
                id: EntityId(0),
                position: HexCoord(q: 3, r: 4, s: -7),
                hp: 25,
                max_hp: 30,
                // ...
            ),
            // ...
        ],
        turn_order: [EntityId(0), EntityId(2), EntityId(1)],
        current_turn_index: 1,
        round_number: 2,
    ),
    session_state: SessionState(
        current_scenario: "forest_ambush",
        completed_scenarios: ["tutorial", "first_battle"],
        // ...
    ),
)
```

**Pros**: Readable, native Rust types, good for debugging
**Cons**: Larger file size than binary

### Alternative: Binary (MessagePack, Bincode)

```rust
use bincode::{serialize, deserialize};

fn save_binary(save: &SaveFile, path: &Path) -> Result<()> {
    let bytes = serialize(save)?;
    fs::write(path, bytes)?;
    Ok(())
}

fn load_binary(path: &Path) -> Result<SaveFile> {
    let bytes = fs::read(path)?;
    Ok(deserialize(&bytes)?)
}
```

**Pros**: Compact, fast
**Cons**: Not human-readable, harder to debug

**Recommendation**: RON for dev builds, binary for release (with compression).

## What to Save

### Full Game State

```rust
#[derive(Serialize, Deserialize)]
struct GameState {
    // Entities
    entities: Vec<Entity>,
    next_entity_id: u32,
    
    // Combat state
    turn_manager: TurnManager,
    action_queue: Vec<GameAction>,
    
    // Map
    map: HexMap,
    terrain_state: HashMap<HexCoord, TerrainState>,  // Dynamic changes (fire, ice)
    
    // Status effects
    active_effects: Vec<ActiveStatusEffect>,
    
    // RNG state (for deterministic replays)
    rng_seed: u64,
    
    // Event history (optional, for replay)
    event_log: Vec<GameEvent>,
}
```

### Session/Campaign State

```rust
#[derive(Serialize, Deserialize)]
struct SessionState {
    // Campaign
    current_scenario: ScenarioId,
    completed_scenarios: Vec<ScenarioId>,
    campaign_flags: HashMap<String, bool>,
    
    // Persistent units
    player_roster: Vec<PersistentUnit>,
    
    // Progression
    unlocked_abilities: HashSet<AbilityId>,
    unlocked_units: HashSet<UnitTypeId>,
    
    // Resources
    gold: i32,
    items: Vec<ItemInstance>,
}

#[derive(Serialize, Deserialize)]
struct PersistentUnit {
    unit_type: UnitTypeId,
    level: u32,
    experience: i32,
    equipment: Vec<ItemId>,
    learned_abilities: Vec<AbilityId>,
}
```

### Settings

Separate from save files (shared across all saves):

```rust
#[derive(Serialize, Deserialize)]
struct GameSettings {
    // Display
    resolution: (u32, u32),
    fullscreen: bool,
    vsync: bool,
    
    // Audio
    master_volume: f32,
    music_volume: f32,
    sfx_volume: f32,
    
    // Gameplay
    difficulty: Difficulty,
    combat_speed: f32,  // Animation speed
    auto_end_turn: bool,
    
    // Input
    keybindings: HashMap<GameAction, Vec<InputSource>>,
    mouse_sensitivity: f32,
    
    // Accessibility
    subtitles: bool,
    colorblind_mode: ColorblindMode,
}
```

**Location**: `~/.cc-game-engine/settings.ron`

## Save Slots

```rust
impl SaveManager {
    fn list_saves(&self) -> Vec<SaveMetadata> {
        // Scan save directory for .save files
        let mut saves = Vec::new();
        
        for entry in fs::read_dir(&self.save_directory)? {
            let path = entry?.path();
            if path.extension() == Some("save") {
                if let Ok(metadata) = self.load_metadata(&path) {
                    saves.push(metadata);
                }
            }
        }
        
        saves.sort_by_key(|m| m.timestamp);
        saves.reverse();  // Most recent first
        saves
    }
    
    fn save(&mut self, slot: u32, state: &GameState) -> Result<()> {
        let filename = format!("save_{}.save", slot);
        let path = self.save_directory.join(filename);
        
        let save_file = SaveFile {
            metadata: self.create_metadata(slot, state),
            game_state: state.clone(),
            session_state: self.session_state.clone(),
        };
        
        // Write to temporary file first
        let temp_path = path.with_extension("tmp");
        self.write_save(&save_file, &temp_path)?;
        
        // Atomic rename (prevents corruption if interrupted)
        fs::rename(temp_path, path)?;
        
        Ok(())
    }
    
    fn load(&mut self, slot: u32) -> Result<SaveFile> {
        let filename = format!("save_{}.save", slot);
        let path = self.save_directory.join(filename);
        
        self.read_save(&path)
    }
    
    fn delete(&mut self, slot: u32) -> Result<()> {
        let filename = format!("save_{}.save", slot);
        let path = self.save_directory.join(filename);
        fs::remove_file(path)?;
        Ok(())
    }
}
```

## Auto-Save

```rust
struct AutoSaveManager {
    enabled: bool,
    interval: f32,         // Save every N seconds
    last_save_time: f32,
    save_on_turn_end: bool,
    max_auto_saves: u32,   // Keep last N auto-saves
}

impl AutoSaveManager {
    fn update(&mut self, dt: f32, state: &GameState) {
        if !self.enabled {
            return;
        }
        
        self.last_save_time += dt;
        
        if self.last_save_time >= self.interval {
            self.auto_save(state);
            self.last_save_time = 0.0;
        }
    }
    
    fn on_turn_end(&mut self, state: &GameState) {
        if self.save_on_turn_end {
            self.auto_save(state);
        }
    }
    
    fn auto_save(&mut self, state: &GameState) {
        // Use special auto-save slot (e.g., 999)
        save_manager.save(AUTO_SAVE_SLOT, state);
        
        // Rotate old auto-saves
        self.cleanup_old_auto_saves();
    }
}
```

## Quick Save/Load

```rust
impl SaveManager {
    fn quick_save(&mut self, state: &GameState) {
        // Always save to slot 0
        self.save(QUICK_SAVE_SLOT, state);
    }
    
    fn quick_load(&mut self) -> Result<SaveFile> {
        self.load(QUICK_SAVE_SLOT)
    }
}
```

**Keybindings**:
- F5: Quick save
- F9: Quick load

## Version Compatibility

Handle saves from older game versions:

```rust
struct VersionMigrator {
    migrations: HashMap<String, Box<dyn Fn(SaveFile) -> SaveFile>>,
}

impl VersionMigrator {
    fn migrate(&self, save: SaveFile, target_version: &str) -> Result<SaveFile> {
        let mut current = save;
        let mut version = current.metadata.version.clone();
        
        while version != target_version {
            if let Some(migration) = self.migrations.get(&version) {
                current = migration(current);
                version = current.metadata.version.clone();
            } else {
                return Err("No migration path");
            }
        }
        
        Ok(current)
    }
}

// Example migration: v0.1.0 → v0.2.0
fn migrate_v01_to_v02(mut save: SaveFile) -> SaveFile {
    // Added new field: status_effect_resistance
    for entity in &mut save.game_state.entities {
        entity.status_effect_resistance = 0.0;  // Default value
    }
    
    save.metadata.version = "0.2.0".to_string();
    save
}
```

**Version policy**:
- **Patch versions** (0.1.0 → 0.1.1): Full compatibility
- **Minor versions** (0.1.0 → 0.2.0): Automatic migration
- **Major versions** (0.x → 1.x): Manual migration or incompatible

## Corruption Detection

```rust
use sha2::{Sha256, Digest};

#[derive(Serialize, Deserialize)]
struct SignedSaveFile {
    save: SaveFile,
    checksum: Vec<u8>,
}

impl SaveManager {
    fn write_save(&self, save: &SaveFile, path: &Path) -> Result<()> {
        let data = ron::to_string(&save)?;
        
        // Calculate checksum
        let mut hasher = Sha256::new();
        hasher.update(&data);
        let checksum = hasher.finalize().to_vec();
        
        let signed = SignedSaveFile {
            save: save.clone(),
            checksum,
        };
        
        fs::write(path, ron::to_string(&signed)?)?;
        Ok(())
    }
    
    fn read_save(&self, path: &Path) -> Result<SaveFile> {
        let data = fs::read_to_string(path)?;
        let signed: SignedSaveFile = ron::from_str(&data)?;
        
        // Verify checksum
        let save_data = ron::to_string(&signed.save)?;
        let mut hasher = Sha256::new();
        hasher.update(&save_data);
        let computed = hasher.finalize().to_vec();
        
        if computed != signed.checksum {
            return Err("Save file corrupted");
        }
        
        Ok(signed.save)
    }
}
```

**Fallback**: If save is corrupted, try loading backup (previous auto-save).

## Backup System

```rust
impl SaveManager {
    fn save_with_backup(&mut self, slot: u32, state: &GameState) -> Result<()> {
        let filename = format!("save_{}.save", slot);
        let path = self.save_directory.join(&filename);
        
        // Backup existing save before overwriting
        if path.exists() {
            let backup_path = self.save_directory.join(format!("save_{}.backup", slot));
            fs::copy(&path, backup_path)?;
        }
        
        self.save(slot, state)?;
        Ok(())
    }
    
    fn restore_from_backup(&mut self, slot: u32) -> Result<()> {
        let backup_path = self.save_directory.join(format!("save_{}.backup", slot));
        let save_path = self.save_directory.join(format!("save_{}.save", slot));
        
        if backup_path.exists() {
            fs::copy(backup_path, save_path)?;
            Ok(())
        } else {
            Err("No backup found")
        }
    }
}
```

## Screenshot Thumbnails

```rust
fn create_save_thumbnail(renderer: &Renderer) -> Vec<u8> {
    // Capture current frame at low resolution
    let screenshot = renderer.capture_screenshot(256, 144);  // 16:9 thumbnail
    
    // Encode as JPEG
    encode_jpeg(&screenshot, 80)  // Quality 80%
}

impl SaveManager {
    fn create_metadata(&self, slot: u32, state: &GameState) -> SaveMetadata {
        SaveMetadata {
            version: env!("CARGO_PKG_VERSION").to_string(),
            timestamp: SystemTime::now(),
            slot_id: slot,
            scenario_name: state.scenario_name.clone(),
            turn_number: state.turn_manager.round_number,
            playtime: state.total_playtime,
            thumbnail: Some(create_save_thumbnail(&self.renderer)),
        }
    }
}
```

Display thumbnails in save/load menu for visual identification.

## Cloud Save (Future)

```rust
trait CloudSaveProvider {
    fn upload(&self, slot: u32, save: &SaveFile) -> Result<()>;
    fn download(&self, slot: u32) -> Result<SaveFile>;
    fn list_cloud_saves(&self) -> Result<Vec<SaveMetadata>>;
    fn delete(&self, slot: u32) -> Result<()>;
}

struct SteamCloudSave;
impl CloudSaveProvider for SteamCloudSave {
    // Steam Cloud integration
}

struct ManualCloudSave {
    sync_directory: PathBuf,  // e.g., Dropbox, Google Drive folder
}
impl CloudSaveProvider for ManualCloudSave {
    fn upload(&self, slot: u32, save: &SaveFile) -> Result<()> {
        let filename = format!("save_{}.save", slot);
        let path = self.sync_directory.join(filename);
        // Write to synced folder
        fs::write(path, ron::to_string(save)?)?;
        Ok(())
    }
    // ...
}
```

**Conflict resolution**: If local and cloud saves differ, show both timestamps and let player choose.

## Performance

- **Async saving**: Save on background thread to avoid frame drops
- **Compression**: gzip save files (especially for binary format)
- **Incremental saves**: Only save changed data (complex, maybe v2.0)

```rust
use async_std::task;

fn save_async(save: SaveFile, path: PathBuf) {
    task::spawn(async move {
        // Serialize and write on background thread
        let data = ron::to_string(&save).unwrap();
        fs::write(path, data).unwrap();
    });
}
```

## Debug Tools

```rust
struct SaveDebugger {
    show_save_contents: bool,
    verify_saves: bool,
}

impl SaveDebugger {
    fn inspect_save(&self, path: &Path) {
        let save: SaveFile = ron::from_str(&fs::read_to_string(path).unwrap()).unwrap();
        
        println!("=== Save File ===");
        println!("Version: {}", save.metadata.version);
        println!("Turn: {}", save.metadata.turn_number);
        println!("Entities: {}", save.game_state.entities.len());
        println!("================");
    }
    
    fn test_save_load_cycle(&self, state: &GameState) {
        // Save, then immediately load and compare
        let temp_path = PathBuf::from("/tmp/test_save.save");
        save_manager.save(999, state);
        let loaded = save_manager.load(999).unwrap();
        
        assert_eq!(state, &loaded.game_state);
    }
}
```

## Best Practices

1. **Save often**: Auto-save every turn, never lose progress
2. **Atomic writes**: Use temp file + rename to prevent corruption
3. **Validate on load**: Check checksums, version compatibility
4. **Keep backups**: Previous save always available
5. **Test migrations**: Every version change, test old saves load correctly
6. **Compress large saves**: gzip for >1MB files
7. **Don't save UI state**: Only game logic, UI rebuilds on load
