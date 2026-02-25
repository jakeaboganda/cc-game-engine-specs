# Build & Deployment

## Build Pipeline

### Development Builds

Fast iteration with debug symbols:

```bash
# Quick build (debug)
cargo build

# Run game
cargo run

# Run with editor
cargo run --features editor

# Build specific crate
cargo build -p cc-tactics

# Run tests
cargo test

# Run with logging
RUST_LOG=debug cargo run
```

### Release Builds

Optimized for distribution:

```bash
# Full optimization
cargo build --release

# Strip debug symbols
cargo build --release --config profile.release.strip=true

# With LTO (slower build, smaller binary)
cargo build --release --config profile.release.lto=true

# Build all platforms (using cross)
cross build --release --target x86_64-pc-windows-gnu
cross build --release --target x86_64-unknown-linux-gnu
cross build --release --target x86_64-apple-darwin
```

### Profile.toml Configuration

```toml
# Cargo.toml

[profile.dev]
opt-level = 1          # Some optimizations for playable frame rates
debug = true           # Full debug symbols

[profile.dev.package."*"]
opt-level = 3          # Optimize dependencies even in debug

[profile.release]
opt-level = 3
lto = "fat"            # Full link-time optimization
codegen-units = 1      # Single codegen unit for max optimization
strip = true           # Remove debug symbols
panic = "abort"        # Smaller binary, no unwinding

[profile.release-with-debug]
inherits = "release"
strip = false
debug = true           # Debug symbols for profiling

[profile.bench]
inherits = "release"
debug = true           # Keep symbols for flame graphs
```

---

## Asset Pipeline

### Asset Organization

```
assets/
├── textures/
│   ├── terrain/
│   │   ├── grass.png
│   │   ├── stone.png
│   │   └── water.png
│   ├── units/
│   │   ├── warrior/
│   │   │   ├── idle_NE.png
│   │   │   ├── idle_E.png
│   │   │   └── ...
│   │   └── goblin/
│   └── ui/
├── audio/
│   ├── music/
│   │   ├── menu_theme.ogg
│   │   └── combat_theme.ogg
│   └── sfx/
│       ├── sword_hit.wav
│       └── footstep.wav
├── fonts/
│   └── main.ttf
└── shaders/
    ├── sprite.wgsl
    └── terrain.wgsl

content/
├── units/
│   ├── warrior.toml
│   └── goblin.toml
├── abilities/
│   ├── power_attack.toml
│   └── fireball.toml
└── maps/
    └── forest_ambush.toml
```

### Asset Processing

```bash
#!/bin/bash
# tools/process_assets.sh

# Optimize PNG images
for img in assets/textures/**/*.png; do
    optipng -o7 "$img"
done

# Convert WAV to OGG (smaller, compressed)
for wav in assets/audio/sfx/*.wav; do
    ffmpeg -i "$wav" -c:a libvorbis -q:a 5 "${wav%.wav}.ogg"
    rm "$wav"  # Remove original
done

# Pack sprite sheets with TexturePacker
texture-packer \
    --input assets/textures/units/ \
    --output assets/packed/units.png \
    --atlas assets/packed/units.json \
    --trim \
    --algorithm MaxRects

echo "Asset processing complete!"
```

### Asset Packing for Release

```bash
# tools/pack_assets.sh

cargo run --bin pack-assets -- \
    --input assets/ \
    --output release/assets.pak \
    --compress gzip

cargo run --bin pack-assets -- \
    --input content/ \
    --output release/content.pak \
    --compress gzip
```

**pack-assets binary**:
```rust
// crates/cc-game/src/bin/pack-assets.rs

use std::fs;
use std::path::PathBuf;
use walkdir::WalkDir;
use flate2::write::GzEncoder;
use flate2::Compression;

fn main() -> anyhow::Result<()> {
    let args = parse_args();
    
    let mut pack = AssetPack::new();
    
    for entry in WalkDir::new(&args.input) {
        let entry = entry?;
        if entry.file_type().is_file() {
            let path = entry.path();
            let relative = path.strip_prefix(&args.input)?;
            let bytes = fs::read(path)?;
            
            let compressed = if args.compress {
                compress_gzip(&bytes)
            } else {
                bytes
            };
            
            pack.add_file(relative.to_string_lossy().to_string(), compressed);
        }
    }
    
    pack.write(&args.output)?;
    
    Ok(())
}
```

---

## Platform-Specific Builds

### Windows

```bash
# Native Windows build
cargo build --release --target x86_64-pc-windows-msvc

# Create installer with WiX
candle installer.wxs
light -out game_installer.msi installer.wixobj
```

**Installer.wxs** (WiX XML):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Wix xmlns="http://schemas.microsoft.com/wix/2006/wi">
  <Product Id="*" Name="CC Game" Version="0.1.0" Manufacturer="YourCompany">
    <Package InstallerVersion="200" Compressed="yes" />
    
    <Directory Id="TARGETDIR" Name="SourceDir">
      <Directory Id="ProgramFilesFolder">
        <Directory Id="INSTALLFOLDER" Name="CC Game">
          <Component Id="MainExecutable">
            <File Source="target/release/cc-game.exe" KeyPath="yes" />
          </Component>
          
          <Component Id="Assets">
            <File Source="release/assets.pak" />
            <File Source="release/content.pak" />
          </Component>
        </Directory>
      </Directory>
    </Directory>
    
    <Feature Id="Complete" Level="1">
      <ComponentRef Id="MainExecutable" />
      <ComponentRef Id="Assets" />
    </Feature>
  </Product>
</Wix>
```

### Linux

```bash
# Build
cargo build --release --target x86_64-unknown-linux-gnu

# Create AppImage
linuxdeploy --executable target/release/cc-game \
            --appdir AppDir \
            --output appimage

# Or create .deb package
cargo deb --target x86_64-unknown-linux-gnu
```

**Cargo.toml** (for cargo-deb):
```toml
[package.metadata.deb]
name = "cc-game"
maintainer = "Your Name <you@example.com>"
copyright = "2024, Your Name"
license-file = ["LICENSE", "4"]
depends = "$auto"
section = "games"
priority = "optional"
assets = [
    ["target/release/cc-game", "usr/bin/", "755"],
    ["release/assets.pak", "usr/share/cc-game/", "644"],
    ["release/content.pak", "usr/share/cc-game/", "644"],
    ["desktop/cc-game.desktop", "usr/share/applications/", "644"],
    ["desktop/icon.png", "usr/share/pixmaps/cc-game.png", "644"],
]
```

### macOS

```bash
# Build universal binary (Intel + Apple Silicon)
cargo build --release --target x86_64-apple-darwin
cargo build --release --target aarch64-apple-darwin

lipo -create \
    target/x86_64-apple-darwin/release/cc-game \
    target/aarch64-apple-darwin/release/cc-game \
    -output cc-game-universal

# Create .app bundle
./tools/create_macos_app.sh
```

**create_macos_app.sh**:
```bash
#!/bin/bash

APP_NAME="CC Game"
APP_DIR="$APP_NAME.app"
CONTENTS="$APP_DIR/Contents"

mkdir -p "$CONTENTS/MacOS"
mkdir -p "$CONTENTS/Resources"

# Copy executable
cp cc-game-universal "$CONTENTS/MacOS/cc-game"
chmod +x "$CONTENTS/MacOS/cc-game"

# Copy assets
cp -r release/assets.pak "$CONTENTS/Resources/"
cp -r release/content.pak "$CONTENTS/Resources/"

# Create Info.plist
cat > "$CONTENTS/Info.plist" <<EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>CFBundleName</key>
    <string>$APP_NAME</string>
    <key>CFBundleExecutable</key>
    <string>cc-game</string>
    <key>CFBundleIdentifier</key>
    <string>com.yourcompany.ccgame</string>
    <key>CFBundleVersion</key>
    <string>0.1.0</string>
    <key>CFBundleIconFile</key>
    <string>icon.icns</string>
</dict>
</plist>
EOF

# Copy icon
cp desktop/icon.icns "$CONTENTS/Resources/"

# Create DMG
hdiutil create -volname "$APP_NAME" -srcfolder "$APP_DIR" -ov -format UDZO "$APP_NAME.dmg"
```

---

## Continuous Integration

### GitHub Actions Workflow

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  build-windows:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Install Rust
        uses: actions-rs/toolchain@v1
        with:
          toolchain: stable
          target: x86_64-pc-windows-msvc
      
      - name: Build
        run: cargo build --release --target x86_64-pc-windows-msvc
      
      - name: Package
        run: |
          mkdir release
          cp target/x86_64-pc-windows-msvc/release/cc-game.exe release/
          cp -r assets release/
          cp -r content release/
      
      - name: Create ZIP
        run: |
          cd release
          7z a ../cc-game-windows-x64.zip *
      
      - name: Upload artifact
        uses: actions/upload-artifact@v3
        with:
          name: windows-build
          path: cc-game-windows-x64.zip
  
  build-linux:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Install dependencies
        run: |
          sudo apt-get update
          sudo apt-get install -y libasound2-dev libudev-dev
      
      - name: Install Rust
        uses: actions-rs/toolchain@v1
        with:
          toolchain: stable
      
      - name: Build
        run: cargo build --release
      
      - name: Package
        run: |
          mkdir -p release
          cp target/release/cc-game release/
          cp -r assets release/
          cp -r content release/
      
      - name: Create tarball
        run: tar -czf cc-game-linux-x64.tar.gz -C release .
      
      - name: Upload artifact
        uses: actions/upload-artifact@v3
        with:
          name: linux-build
          path: cc-game-linux-x64.tar.gz
  
  build-macos:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Install Rust
        uses: actions-rs/toolchain@v1
        with:
          toolchain: stable
          target: x86_64-apple-darwin
      
      - name: Install Apple Silicon target
        run: rustup target add aarch64-apple-darwin
      
      - name: Build Intel
        run: cargo build --release --target x86_64-apple-darwin
      
      - name: Build Apple Silicon
        run: cargo build --release --target aarch64-apple-darwin
      
      - name: Create universal binary
        run: |
          lipo -create \
            target/x86_64-apple-darwin/release/cc-game \
            target/aarch64-apple-darwin/release/cc-game \
            -output cc-game-universal
      
      - name: Create .app bundle
        run: ./tools/create_macos_app.sh
      
      - name: Upload artifact
        uses: actions/upload-artifact@v3
        with:
          name: macos-build
          path: CC Game.dmg
  
  release:
    needs: [build-windows, build-linux, build-macos]
    runs-on: ubuntu-latest
    steps:
      - name: Download Windows build
        uses: actions/download-artifact@v3
        with:
          name: windows-build
      
      - name: Download Linux build
        uses: actions/download-artifact@v3
        with:
          name: linux-build
      
      - name: Download macOS build
        uses: actions/download-artifact@v3
        with:
          name: macos-build
      
      - name: Create Release
        uses: softprops/action-gh-release@v1
        with:
          files: |
            cc-game-windows-x64.zip
            cc-game-linux-x64.tar.gz
            CC Game.dmg
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## Distribution

### Steam

**Depot structure**:
```
depot_1001/  (Windows)
├── cc-game.exe
├── assets.pak
└── content.pak

depot_1002/  (Linux)
├── cc-game
├── assets.pak
└── content.pak

depot_1003/  (macOS)
└── CC Game.app/
```

**Upload via SteamPipe**:
```bash
steamcmd +login <username> +run_app_build depot_build_script.vdf +quit
```

### Itch.io

```bash
# Install butler
butler push release/cc-game-windows-x64.zip yourname/cc-game:windows-x64
butler push release/cc-game-linux-x64.tar.gz yourname/cc-game:linux-x64
butler push release/cc-game-macos.dmg yourname/cc-game:macos
```

### GOG

Submit builds via GOG Galaxy Pipeline.

---

## Version Management

### Semantic Versioning

Format: `MAJOR.MINOR.PATCH`

- **MAJOR**: Breaking changes (save incompatibility, major features)
- **MINOR**: New features (backward compatible)
- **PATCH**: Bug fixes

### Version Bumping

```bash
# Bump version in all crates
./tools/bump_version.sh 0.2.0

# Create git tag
git tag -a v0.2.0 -m "Release v0.2.0"
git push origin v0.2.0
```

**bump_version.sh**:
```bash
#!/bin/bash

NEW_VERSION=$1

if [ -z "$NEW_VERSION" ]; then
    echo "Usage: $0 <version>"
    exit 1
fi

# Update workspace version
sed -i.bak "s/^version = .*/version = \"$NEW_VERSION\"/" Cargo.toml

# Update all crate versions
for crate in crates/*/Cargo.toml; do
    sed -i.bak "s/^version = .*/version = \"$NEW_VERSION\"/" "$crate"
done

# Clean up backups
find . -name "*.bak" -delete

echo "Bumped version to $NEW_VERSION"
```

---

## Deployment Checklist

Before releasing:

- [ ] All tests pass (`cargo test --workspace`)
- [ ] No clippy warnings (`cargo clippy --all-targets`)
- [ ] Code formatted (`cargo fmt --all`)
- [ ] Benchmarks run without regression
- [ ] Manual playtesting complete
- [ ] Save/load tested across versions
- [ ] Performance profiled (60 FPS target)
- [ ] Assets optimized (images, audio compressed)
- [ ] Content validated (no broken references)
- [ ] Changelog updated
- [ ] Version bumped in all crates
- [ ] Git tagged
- [ ] Release notes written
- [ ] Builds created for all platforms
- [ ] Installers tested on clean machines
- [ ] Save migration tested (if applicable)

---

## Post-Release

### Monitoring

Track crashes via crash reporting (e.g., Sentry):

```toml
[dependencies]
sentry = "0.32"
```

```rust
fn main() {
    let _guard = sentry::init((
        "https://your-dsn@sentry.io/project",
        sentry::ClientOptions {
            release: sentry::release_name!(),
            ..Default::default()
        },
    ));
    
    // Run game
    run_game();
}
```

### Analytics

Optional telemetry (with user consent):

```rust
// Track general usage patterns (no PII)
analytics::track_event("game_started", hashmap! {
    "version" => env!("CARGO_PKG_VERSION"),
    "platform" => std::env::consts::OS,
});
```

### Update Distribution

```bash
# Push update via Steam
butler push release/update steamuser/game:channel

# Create auto-updater manifest
cat > update_manifest.json <<EOF
{
  "version": "0.2.0",
  "url": "https://yoursite.com/downloads/cc-game-v0.2.0.zip",
  "checksum": "sha256:abcdef..."
}
EOF
```

---

## Development vs Production

### Environment Variables

```rust
// src/config.rs

pub struct Config {
    pub debug_mode: bool,
    pub log_level: String,
    pub asset_path: PathBuf,
}

impl Config {
    pub fn load() -> Self {
        if cfg!(debug_assertions) {
            // Development
            Self {
                debug_mode: true,
                log_level: "debug".to_string(),
                asset_path: PathBuf::from("assets/"),
            }
        } else {
            // Production
            Self {
                debug_mode: false,
                log_level: "info".to_string(),
                asset_path: get_asset_install_path(),
            }
        }
    }
}
```

### Conditional Compilation

```rust
#[cfg(debug_assertions)]
fn enable_debug_tools() {
    // Only in debug builds
    ui.add_debug_panel();
}

#[cfg(not(debug_assertions))]
fn enable_debug_tools() {
    // No-op in release
}
```

---

## Docker (Optional)

For reproducible builds:

```dockerfile
# Dockerfile
FROM rust:1.75 AS builder

WORKDIR /build
COPY . .

RUN cargo build --release

FROM debian:bookworm-slim

COPY --from=builder /build/target/release/cc-game /usr/local/bin/
COPY --from=builder /build/release/*.pak /usr/share/cc-game/

CMD ["cc-game"]
```

```bash
docker build -t cc-game:latest .
docker run -v ~/.cc-game:/root/.cc-game cc-game:latest
```
