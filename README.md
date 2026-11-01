# Gameplay Football

A C++ football game based on the discontinued
[Gameplay Football](https://github.com/BazkieBumpercar/GameplayFootball) by
[Bastiaan Konings Schuiling](http://www.properlydecent.com/).

This fork aims to modernize the original game while retaining its menus, audio,
and match simulation. It also draws on changes from
[Google Research Football](https://github.com/google-research/football), the
reinforcement learning environment derived from the game in 2019. The
`google_brain` branch contains related reference work.

## Build status

| Target | Verification |
| --- | --- |
| macOS Apple Silicon | Release build passed with vcpkg; startup initialized OpenGL 4.1 and audio. Full gameplay has not been verified. |
| macOS Intel | Build commands provided below; not verified. |
| Linux | Not verified with the current vcpkg configuration. |
| Windows | Work in progress; not verified with the current vcpkg configuration. |

The macOS startup check emitted a texture warning. A successful build and context
initialization do not establish that every scene renders correctly.

## Requirements

- Git and CMake 3.21 or newer.
- A C++ compiler and platform build tools.
- [vcpkg](https://github.com/microsoft/vcpkg) to build third-party dependencies.

The root `vcpkg.json` declares SDL2, SDL2_image (including JPEG support), SDL2_ttf,
Boost components, OpenAL Soft, and SQLite. Its baseline pins dependency
versions. CMake installs these automatically when configured with the vcpkg
toolchain; a separate `vcpkg install` command is unnecessary.

OpenGL comes from the platform SDK or system development packages. On macOS,
third-party libraries are linked statically and OpenGL uses the system framework.
Homebrew dependency libraries are not required.

## macOS

### Set up the tools

Install Apple's Command Line Tools if needed:

```bash
xcode-select --install
```

Install CMake and make sure `cmake` is available in your shell. Then set up vcpkg
once, choosing any location for its checkout:

```bash
git clone https://github.com/microsoft/vcpkg.git "$HOME/vcpkg"
"$HOME/vcpkg/bootstrap-vcpkg.sh" -disableMetrics
export VCPKG_ROOT="$HOME/vcpkg"
```

If vcpkg is already installed, set `VCPKG_ROOT` to that checkout instead. Set this
variable again in each new terminal session, or add it to your shell profile.

### Build for Apple Silicon

Run from the GameplayFootball repository root:

```bash
cmake --preset mac-vcpkg
cmake --build --preset mac-vcpkg --parallel "$(sysctl -n hw.logicalcpu)"
```

The preset creates a Release build in `build/mac-vcpkg` using `arm64-osx`.
The first configure takes longer because vcpkg downloads and builds dependencies.
Later builds reuse installed dependencies and vcpkg's binary cache.

### Run

Copy the game data into the build directory, then run from that directory:

```bash
cp -R data/. build/mac-vcpkg/
cd build/mac-vcpkg
./gameplayfootball
```

The working directory must contain `football.config`, `databases/`, and `media/`.
CMake also stages these assets beside the executable after linking. Copy them again after changing
files under `data/`; this also replaces the staged configuration file.

### Build for Intel

Use a separate directory and override the preset's triplet:

```bash
cmake --preset mac-vcpkg -B build/mac-vcpkg-intel -DVCPKG_TARGET_TRIPLET=x64-osx
cmake --build build/mac-vcpkg-intel --parallel "$(sysctl -n hw.logicalcpu)"
cp -R data/. build/mac-vcpkg-intel/
cd build/mac-vcpkg-intel
./gameplayfootball
```

Intel builds and execution have not been verified.

### Compatibility notes

- macOS creates the window and pumps events on the main thread; OpenGL rendering runs on its renderer thread.
- OpenAL Soft is pinned to `1.23.1#2`. Version 1.25.2 failed to compile with the
  selected Apple Command Line Tools because their C++ library lacked
  `std::views::join`.
- The build still produces warnings from existing game code.

## Linux

The following is a starting point for Debian/Ubuntu; this path has not been
verified with the current manifest. Install the compiler, CMake, Git, and system
OpenGL development files:

```bash
sudo apt-get update
sudo apt-get install build-essential cmake git curl zip unzip tar pkg-config libgl1-mesa-dev
```

Bootstrap vcpkg as in the macOS setup above, set `VCPKG_ROOT`, then run from the
repository root:

```bash
cmake -S . -B build/linux-vcpkg \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_TOOLCHAIN_FILE="$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake"
cmake --build build/linux-vcpkg --parallel "$(nproc)"
cp -R data/. build/linux-vcpkg/
cd build/linux-vcpkg
./gameplayfootball
```

SDL's window and audio backends may require additional system development
packages for your distribution and desktop environment.

## Windows

Windows support is a work in progress. These commands describe the manifest
workflow; the merged code still needs an MSVC build and runtime check. Install Visual Studio 2022 with the **Desktop
development with C++** workload, Git, and CMake. Use a Visual Studio Developer
Command Prompt.

```bat
git clone https://github.com/microsoft/vcpkg.git C:\dev\vcpkg
call C:\dev\vcpkg\bootstrap-vcpkg.bat -disableMetrics
set VCPKG_ROOT=C:\dev\vcpkg
```

From the GameplayFootball repository root, configure the original 32-bit target:

```bat
cmake --preset windows-vcpkg
cmake --build --preset windows-vcpkg --parallel
cd build\windows-vcpkg\Release
gameplayfootball.exe
```

The default Windows executable keeps a console for diagnostics. To build the GUI
subsystem variant, configure with `-DGAMEPLAYFOOTBALL_WINDOWS_SUBSYSTEM=ON`.
Both variants link SDL2main. The Windows CI workflow builds both variants; a
passing compile does not replace a GPU, input, audio, and shutdown smoke test.

## Merge portability checks

C++ targets use standard C++14, and GCC/Clang builds reject variable-length arrays
with `-Werror=vla`. Keep dynamically sized storage in `std::vector` so MSVC can
compile the same source. The Windows replacements in loaders, animation,
renderer buffers, pitch generation, team AI, and GUI layout are retained.
See [the merge review](docs/mac-windows-merge.md) for evidence and remaining checks.

## Troubleshooting

- **CMake cannot find SDL2 or another package:** check `VCPKG_ROOT` and configure
  with the preset or explicit vcpkg toolchain shown above.
- **Changing toolchain or architecture:** use a fresh build directory. An existing
  CMake cache can retain paths and architecture settings from an earlier build.
- **Missing configuration, database, or media:** stage `data/` and run with the
  build directory as the working directory.
- **Dependency build failure:** inspect the log path printed by vcpkg. Include the
  failing package, compiler version, target triplet, and relevant errors in an issue.
- **Rendering or startup issue:** include `log.txt` from the run directory and
  terminal output, along with the OS and GPU details.

There is no automated test suite. Validate changes by building and running the
game, and distinguish startup checks from full gameplay testing.

## Source layout

| Path | Purpose |
| --- | --- |
| `src/main.cpp` | Application entry point and task setup |
| `src/base/`, `src/framework/`, `src/scene/`, `src/systems/` | Blunted2 engine and platform systems |
| `src/onthepitch/` | Match simulation, players, teams, and AI |
| `src/menu/`, `src/hid/` | Menus and input handling |
| `data/` | Configuration, databases, shaders, and other game assets |
| `CMakeLists.txt`, `sources.cmake` | Build targets and source lists |
| `vcpkg.json`, `CMakePresets.json` | Dependency versions and platform build presets |

## Contributing

Issues and pull requests are welcome. Include reproducible steps and state which
platforms you built and tested. See [LICENSE](LICENSE) for licensing information.

## Support the original author

Bastiaan's Bitcoin donation address: `1JHnTe2QQj8RL281fXFiyvK9igj2VhPh2t`.
