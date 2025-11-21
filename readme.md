# Astonia Community Client

This is the community client for Astonia 3. It is based on, and compatible
with, the [client maintained by Daniel Brockhaus](https://github.com/DanielBrockhaus/astonia_client).

The main goal is to allow community driven development.

Further goals:

- Maintain backwards compatibility with the original game server.
- Maintain compatibility with all servers currently using the client.
- Keep it simple. No additional libraries, frameworks or tools unless absolutely needed.
- Whenever possible, put bugfixes in single commits for easy cherry-picking.
- Always have a downloadable Windows release with sensible hardware requirements and no software requirements.
- Work towards supporting more / all of the servers.

## Build

### Windows

Install [MSYS2](https://www.msys2.org/). It comes with three shells, launch the clang 64 one (not aarch64).

Install dependencies:
```bash
pacman -Syu
pacman -Sy mingw-w64-clang-x86_64-clang mingw-w64-clang-x86_64-SDL2 mingw-w64-clang-x86_64-libpng mingw-w64-clang-x86_64-SDL2_mixer mingw-w64-clang-x86_64-libzip make zip mingw-w64-clang-x86_64-dwarfstack mingw-w64-clang-x86_64-zig mingw-w64-clang-x86_64-rustup
```

After installing dependencies, set up Rust (only needs to be done once):
```bash
# IMPORTANT: Install and use the GNU toolchain (not MSVC)
# This avoids requiring Visual Studio C++ Build Tools
rustup toolchain install stable-x86_64-pc-windows-gnu
rustup default stable-x86_64-pc-windows-gnu

# Alternatively, just override for this project directory:
# cd /path/to/astonia_client
# rustup override set stable-x86_64-pc-windows-gnu
```

**Note:** You can build from any Windows shell (CMD, PowerShell, or MSYS2). The Makefile will automatically detect and use the CLANG64 libraries from your MSYS2 installation. If you have MSYS2 installed in a non-standard location, set the `CLANG64_PREFIX` environment variable to your clang64 path (e.g., `C:\msys64\clang64`).

**Note:** This build uses the GNU toolchain and does **not** require Visual Studio or C++ Build Tools.

### Linux

Install dependencies:
```bash
sudo pacman -S base-devel sdl3 sdl2-compat sdl2_mixer libpng libzip zlib zig rust
```

Or use Docker (no dependencies needed):
```bash
make docker-linux
```

### macOS

Install Xcode Command Line Tools:
```bash
xcode-select --install
```

Install dependencies:
```bash
brew install zig sdl2 sdl2_mixer libpng zlib libzip rust
```

## Commands

```bash
make            # Build for current platform (requires MSYS2 only)
make zig-build  # Build using Zig (optional, requires Visual Studio C++ Build Tools)
make distrib    # Create distribution package
make amod       # Build mod (src/amod/)
make clean      # Clean up build assets
```

**Note:** The standard `make` build uses the GNU toolchain and works without Visual Studio. The `make zig-build` option is available for developers who prefer Zig, but requires Visual Studio C++ Build Tools to be installed.

## Troubleshooting (Windows)

If the build fails with linker errors, run the diagnostic script:

```powershell
.\diagnose_rust.ps1            # Check Rust toolchain configuration
```

This will check if you're using the correct GNU toolchain and provide fix instructions.

### Testing Without Visual Studio

If you have Visual Studio installed and want to test that the build works **without** Visual Studio (as your users would experience):

**PowerShell:**
```powershell
.\test_without_msvc.ps1        # Test build with Visual Studio paths removed
.\verify_build_tools.ps1       # Verify which tools/libraries are being used
```

**CMD:**
```cmd
test_without_msvc.bat          # Test build with Visual Studio paths removed
```

These scripts temporarily remove Visual Studio from PATH to simulate a clean environment, helping ensure the build works with only MSYS2 dependencies.
