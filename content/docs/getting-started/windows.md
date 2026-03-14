---
title: "Windows Installation"
weight: 1
---

# Installing Toona on Windows

## System Requirements

- Windows 10 or later
- Rust 1.79+
- Visual Studio Build Tools or Visual Studio 2019+

## Installation Steps

### 1. Install Rust

Download and install Rust from [rustup.rs](https://rustup.rs/):

```powershell
# Download and run rustup-init.exe
# Follow the installation prompts
```

### 2. Install Visual Studio Build Tools

Download from [Visual Studio Downloads](https://visualstudio.microsoft.com/downloads/):
- Select "Desktop development with C++"
- Install the required components

### 3. Clone and Build Toona

```powershell
# Clone the repository
git clone https://github.com/toona/toona.git
cd toona

# Build in release mode
cargo build --release

# Run Toona
cargo run --release
```

### 4. Create Desktop Shortcut (Optional)

```powershell
# Package the application
cargo packager --release
```

The installer will be created in `target/release/bundle/`.

## Troubleshooting

### Build Errors

If you encounter build errors:

```powershell
# Update Rust
rustup update

# Clean and rebuild
cargo clean
cargo build --release
```

### Missing Dependencies

Ensure Visual Studio Build Tools are properly installed with C++ support.

## Next Steps

- [Configuration]({{< relref "../configuration" >}})
- [User Guide]({{< relref "/docs/user-guide" >}})
