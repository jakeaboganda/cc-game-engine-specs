# Asset Pipeline

## Overview

System for loading, caching, and managing game assets (sprites, audio, data files). Support hot-reloading for rapid iteration.

## Asset Types

```rust
enum AssetType {
    Texture,
    SpriteSheet,
    Audio,
    Font,
    DataFile(DataFormat),
    Shader,
}

enum DataFormat {
    Ron,
    Toml,
    Json,
}
```

## Asset Manager

```rust
struct AssetManager {
    textures: HashMap<AssetId, TextureAsset>,
    audio: HashMap<AssetId, AudioAsset>,
    fonts: HashMap<AssetId, FontAsset>,
    data: HashMap<AssetId, DataAsset>,
    
    asset_paths: HashMap<AssetId, PathBuf>,
    load_queue: VecDeque<AssetLoadRequest>,
    file_watcher: Option<FileWatcher>,  // For hot-reload
}

type AssetId = String;  // e.g., "sprites/warrior"

struct AssetLoadRequest {
    id: AssetId,
    path: PathBuf,
    asset_type: AssetType,
    priority: LoadPriority,
}

enum LoadPriority {
    Critical,   // Block until loaded (UI, core assets)
    High,       // Load soon (current scenario assets)
    Normal,     // Load when convenient
    Low,        // Background (future scenario assets)
}
```

## Asset Loading

### Synchronous Loading

```rust
impl AssetManager {
    fn load_texture(&mut self, id: &str, path: &Path) -> Result<TextureHandle> {
        // Load image from disk
        let image = image::open(path)?;
        
        // Upload to GPU
        let texture = self.renderer.create_texture(&image);
        
        // Cache
        self.textures.insert(id.to_string(), TextureAsset {
            handle: texture,
            path: path.to_path_buf(),
            last_modified: fs::metadata(path)?.modified()?,
        });
        
        Ok(texture)
    }
    
    fn load_audio(&mut self, id: &str, path: &Path) -> Result<AudioHandle> {
        let bytes = fs::read(path)?;
        let sound = self.audio_engine.load_from_bytes(&bytes)?;
        
        self.audio.insert(id.to_string(), AudioAsset {
            handle: sound,
            path: path.to_path_buf(),
        });
        
        Ok(sound)
    }
}
```

### Asynchronous Loading

```rust
use async_std::task;

impl AssetManager {
    fn load_texture_async(&mut self, id: String, path: PathBuf) -> AssetFuture<TextureHandle> {
        task::spawn(async move {
            let image = image::open(&path).await?;
            // Upload to GPU (must happen on main thread)
            Ok(image)
        })
    }
    
    fn process_load_queue(&mut self) {
        // Process N assets per frame to avoid stuttering
        const MAX_LOADS_PER_FRAME: usize = 5;
        
        for _ in 0..MAX_LOADS_PER_FRAME {
            if let Some(request) = self.load_queue.pop_front() {
                match request.asset_type {
                    AssetType::Texture => {
                        self.load_texture(&request.id, &request.path);
                    },
                    // ... other types
                }
            } else {
                break;
            }
        }
    }
}
```

## Hot Reloading

Monitor files for changes and reload automatically:

```rust
use notify::{Watcher, RecursiveMode, Event};

impl AssetManager {
    fn setup_hot_reload(&mut self, asset_directory: &Path) -> Result<()> {
        let (tx, rx) = std::sync::mpsc::channel();
        
        let mut watcher = notify::recommended_watcher(tx)?;
        watcher.watch(asset_directory, RecursiveMode::Recursive)?;
        
        self.file_watcher = Some(FileWatcher {
            watcher,
            receiver: rx,
        });
        
        Ok(())
    }
    
    fn check_for_changes(&mut self) {
        if let Some(ref watcher) = self.file_watcher {
            while let Ok(event) = watcher.receiver.try_recv() {
                if let Ok(event) = event {
                    for path in event.paths {
                        self.reload_asset(&path);
                    }
                }
            }
        }
    }
    
    fn reload_asset(&mut self, path: &Path) {
        // Find asset by path
        if let Some((id, _)) = self.asset_paths.iter()
            .find(|(_, p)| *p == path) 
        {
            println!("Reloading asset: {}", id);
            
            match path.extension().and_then(|s| s.to_str()) {
                Some("png") | Some("jpg") => {
                    self.load_texture(id, path);
                    self.event_bus.publish(AssetReloaded { id: id.clone() });
                },
                Some("ogg") | Some("wav") => {
                    self.load_audio(id, path);
                },
                Some("toml") | Some("ron") => {
                    self.load_data(id, path);
                },
                _ => {}
            }
        }
    }
}
```

**Use case**: Edit sprite in Photoshop → save → see changes in-game immediately.

## Asset Packing

Bundle assets for release builds:

```rust
struct AssetPack {
    files: HashMap<String, Vec<u8>>,  // path → compressed data
}

impl AssetPack {
    fn build(source_dir: &Path, output_path: &Path) -> Result<()> {
        let mut pack = AssetPack {
            files: HashMap::new(),
        };
        
        // Walk directory, compress and pack files
        for entry in WalkDir::new(source_dir) {
            let entry = entry?;
            if entry.file_type().is_file() {
                let path = entry.path();
                let relative = path.strip_prefix(source_dir)?;
                let bytes = fs::read(path)?;
                
                // Compress with gzip
                let compressed = compress_gzip(&bytes);
                
                pack.files.insert(
                    relative.to_string_lossy().to_string(),
                    compressed,
                );
            }
        }
        
        // Serialize pack
        let packed = bincode::serialize(&pack)?;
        fs::write(output_path, packed)?;
        
        Ok(())
    }
    
    fn load(path: &Path) -> Result<Self> {
        let bytes = fs::read(path)?;
        Ok(bincode::deserialize(&bytes)?)
    }
    
    fn read_file(&self, path: &str) -> Option<Vec<u8>> {
        self.files.get(path).map(|compressed| {
            decompress_gzip(compressed)
        })
    }
}
```

**Usage**:
- **Dev mode**: Load directly from `assets/` directory
- **Release mode**: Load from `assets.pak` file

## Asset Resolution

Support multiple resolutions for textures:

```rust
struct AssetResolver {
    base_path: PathBuf,
    resolution_scales: Vec<f32>,  // [1.0, 1.5, 2.0] for @1x, @1.5x, @2x
}

impl AssetResolver {
    fn resolve_texture(&self, logical_path: &str, dpi_scale: f32) -> PathBuf {
        // Find closest resolution
        let closest_scale = self.resolution_scales.iter()
            .min_by_key(|&&s| ((s - dpi_scale).abs() * 100.0) as i32)
            .unwrap();
        
        // assets/sprite.png → assets/sprite@2x.png
        let filename = Path::new(logical_path);
        let stem = filename.file_stem().unwrap().to_str().unwrap();
        let ext = filename.extension().unwrap().to_str().unwrap();
        
        if *closest_scale == 1.0 {
            self.base_path.join(logical_path)
        } else {
            let scaled_name = format!("{}@{}x.{}", stem, closest_scale, ext);
            self.base_path.join(filename.parent().unwrap()).join(scaled_name)
        }
    }
}
```

## Reference Counting

Track asset usage, unload when not needed:

```rust
struct AssetHandle<T> {
    id: AssetId,
    inner: Arc<T>,
}

impl<T> Clone for AssetHandle<T> {
    fn clone(&self) -> Self {
        Self {
            id: self.id.clone(),
            inner: Arc::clone(&self.inner),
        }
    }
}

impl<T> Drop for AssetHandle<T> {
    fn drop(&mut self) {
        if Arc::strong_count(&self.inner) == 1 {
            // Last reference dropped, can unload
            asset_manager.mark_for_unload(&self.id);
        }
    }
}
```

**Unload policy**:
- Keep critical assets always loaded
- Unload unused assets after 60 seconds
- Unload all non-critical assets on scenario transition

## Asset Manifest

Define asset dependencies:

```toml
# content/scenarios/forest_ambush/manifest.toml
[scenario]
name = "Forest Ambush"

[[assets.textures]]
id = "terrain/grass"
path = "textures/terrain/grass.png"
priority = "high"

[[assets.textures]]
id = "units/warrior"
path = "textures/units/warrior.png"
priority = "high"

[[assets.audio]]
id = "music/combat"
path = "audio/music/combat.ogg"
streaming = true

[[assets.data]]
id = "map"
path = "maps/forest_ambush.toml"
format = "toml"
```

**Preloading**:
```rust
fn preload_scenario(&mut self, scenario_id: &str) -> Result<()> {
    let manifest_path = format!("content/scenarios/{}/manifest.toml", scenario_id);
    let manifest: ScenarioManifest = self.load_data(&manifest_path)?;
    
    for asset in &manifest.assets.textures {
        self.load_queue.push_back(AssetLoadRequest {
            id: asset.id.clone(),
            path: PathBuf::from(&asset.path),
            asset_type: AssetType::Texture,
            priority: asset.priority,
        });
    }
    
    // Start loading
    self.process_load_queue();
    
    Ok(())
}
```

## Sprite Sheet Handling

```rust
struct SpriteSheetAsset {
    texture: TextureHandle,
    atlas: SpriteAtlas,
}

struct SpriteAtlas {
    frames: HashMap<String, Rect>,  // name → UV coords
}

// Load from JSON (e.g., TexturePacker output)
fn load_sprite_sheet(texture_path: &Path, atlas_path: &Path) -> Result<SpriteSheetAsset> {
    let texture = load_texture(texture_path)?;
    
    let atlas_json = fs::read_to_string(atlas_path)?;
    let data: TexturePackerFormat = serde_json::from_str(&atlas_json)?;
    
    let mut frames = HashMap::new();
    for (name, frame) in data.frames {
        frames.insert(name, Rect {
            x: frame.x,
            y: frame.y,
            w: frame.w,
            h: frame.h,
        });
    }
    
    Ok(SpriteSheetAsset {
        texture,
        atlas: SpriteAtlas { frames },
    })
}

// Usage
fn get_sprite(&self, sheet_id: &str, sprite_name: &str) -> Sprite {
    let sheet = self.sprite_sheets.get(sheet_id).unwrap();
    let frame = sheet.atlas.frames.get(sprite_name).unwrap();
    
    Sprite {
        texture: sheet.texture,
        source_rect: Some(*frame),
        // ...
    }
}
```

## Error Handling

Graceful fallback for missing assets:

```rust
impl AssetManager {
    fn get_texture(&self, id: &str) -> TextureHandle {
        self.textures.get(id)
            .map(|asset| asset.handle)
            .unwrap_or(self.fallback_texture)
    }
    
    fn create_fallback_assets(&mut self) {
        // 16x16 magenta checkerboard for missing textures
        let fallback_image = create_checkerboard(16, 16, Color::MAGENTA, Color::BLACK);
        self.fallback_texture = self.renderer.create_texture(&fallback_image);
        
        // Silent audio for missing sounds
        self.fallback_audio = self.audio_engine.create_silent_sound(1.0);
    }
}
```

**Error reporting**:
```rust
fn load_texture(&mut self, id: &str, path: &Path) -> Result<TextureHandle> {
    match image::open(path) {
        Ok(image) => {
            // Success
        },
        Err(e) => {
            eprintln!("Failed to load texture '{}' from {:?}: {}", id, path, e);
            // Use fallback, but log error for developer
            Ok(self.fallback_texture)
        }
    }
}
```

## Memory Management

Track memory usage:

```rust
struct AssetMemoryTracker {
    texture_memory: usize,
    audio_memory: usize,
    total_limit: usize,
}

impl AssetMemoryTracker {
    fn on_load_texture(&mut self, texture: &TextureAsset) {
        let size = texture.width * texture.height * 4;  // RGBA
        self.texture_memory += size;
        
        if self.texture_memory > self.total_limit {
            // Trigger garbage collection
            self.evict_lru_textures();
        }
    }
    
    fn evict_lru_textures(&mut self) {
        // Unload least-recently-used textures until under limit
    }
}
```

## Debug Tools

```rust
struct AssetDebugger {
    show_loaded_assets: bool,
    show_memory_usage: bool,
}

impl AssetDebugger {
    fn render_ui(&self, ui: &mut UI) {
        ui.window("Asset Debugger", || {
            ui.text(format!("Textures loaded: {}", asset_manager.textures.len()));
            ui.text(format!("Audio loaded: {}", asset_manager.audio.len()));
            ui.text(format!("Memory usage: {:.2} MB", self.total_memory() / 1_000_000.0));
            
            ui.separator();
            
            for (id, asset) in &asset_manager.textures {
                ui.text(format!("{}: {}x{}", id, asset.width, asset.height));
            }
        });
    }
}
```

## Build Pipeline

Automate asset processing:

```bash
# scripts/build_assets.sh
#!/bin/bash

# Optimize images
for img in assets/textures/**/*.png; do
    optipng -o7 "$img"
done

# Pack sprite sheets
texture-packer --input assets/sprites --output assets/packed/sprites.png --atlas assets/packed/sprites.json

# Encode audio
for ogg in assets/audio/**/*.wav; do
    ffmpeg -i "$ogg" -c:a libvorbis -q:a 5 "${ogg%.wav}.ogg"
done

# Build asset pack
cargo run --bin pack-assets -- assets/ release/assets.pak
```

**Integration**: Run as pre-build step in CI/CD.

## Platform Considerations

Different platforms have different constraints:

```rust
struct PlatformAssetConfig {
    max_texture_size: u32,      // e.g., 2048 on mobile
    audio_format: AudioFormat,  // OGG on desktop, M4A on iOS
    compress_textures: bool,    // Use DXT/ETC compression
}

impl AssetManager {
    fn load_for_platform(&mut self, config: PlatformAssetConfig) {
        // Automatically downscale textures if too large
        // Convert audio formats
        // Apply platform-specific compression
    }
}
```
