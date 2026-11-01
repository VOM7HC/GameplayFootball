# PR #2: Mac into Windows main

Prepared 2026-11-01 from `mac` / `9bc4eda` and `origin/main` / `4c9b7bb`.
Local integration branch: `codex/prepare-mac-windows-merge`.
This branch merges main into the PR history so it can later fast-forward `mac`.
No remote branch or PR has been changed by this preparation.

## Conflict decisions

- CMake: retain Windows console/GUI selection, MSVC optimization flags, explicit
  dependency propagation to object libraries, SDL2main, and post-link asset copy.
  Add Mac static SDL image/TTF targets, vcpkg SQLite, and platform dynamic-loader
  libraries. Use the Mac manifest baseline and OpenAL override on both platforms;
  this changes Windows dependency versions and requires an MSVC rebuild.
- Keep Windows removal of SDL2_gfx and the SDL surface resize implementation.
  Remove the now-unused dependency from the Mac manifest.
- OpenGL: core profile on Apple, compatibility profile elsewhere. Retain Windows
  initialization errors and adapter diagnostics, plus Mac context handoff and
  main-thread event pumping. Correct misleading threading comments.
- SQLite: use the external header/library consistently. The automatic merge
  otherwise added a reference to the deleted `libs/sqlite3/sqlite3.h`.
- Shutdown: remove duplicate overlay/quad buffer deletion introduced by combining
  the two branches' independently relocated cleanup calls.
- Keep the union of build ignores and update the README for both platforms.

## VLA audit

All 14 Windows array-to-vector replacements are retained:

| Source | Storage |
| --- | --- |
| `src/loaders/aseloader.cpp` | vertex_cache, tvertex_cache |
| `src/onthepitch/match.cpp` | lightTest |
| `src/onthepitch/player/humanoid/animcollection.cpp` | origPositions |
| `src/onthepitch/player/humanoid/humanoid.cpp` | touchIDs |
| `src/onthepitch/proceduralpitch.cpp` | ynoise |
| `src/onthepitch/teamAIcontroller.cpp` | r |
| `src/systems/graphics/rendering/opengl_renderer3d.cpp` | targets, samples (two scopes), candidates, SSAO_kernel |
| `src/utils/gui2/widgets/grid.cpp` | widths, heights |

`SetRenderTargets` passes `targets.data()` instead of `&targets[0]`, avoiding
invalid indexing when the attachment list is empty. Fixed arrays sized by
`playerNum` remain valid: it is the compile-time constant `11` in gamedefines.hpp.

CMake requires C++14, disables GNU language extensions, and adds `-Werror=vla`
for C++ on GCC/Clang. This is a continuing build guard, not a regex-only audit.
It covers compiled platform paths; MSVC still needs to compile Windows paths.

## Verification

- Apple Silicon Release configure and full build passed using the installed
  vcpkg toolchain at `/Users/khoi/.local/share/vcpkg`.
- All 177 C++ entries in `build/mac-vcpkg/compile_commands.json` contain both
  `-std=c++14` and `-Werror=vla`; no VLA errors occurred. Existing unrelated
  compiler warnings remain.
- Startup from the staged build directory initialized OpenGL 4.1 Metal and audio.
  The process remained running for 12 seconds before the test sent termination.
  A texture-unloadable warning remains. No visual/gameplay validation is claimed.
- Local diagnostic logs: `/tmp/gameplay-merge-configure.log`,
  `/tmp/gameplay-merge-build.log`, `/tmp/gameplay-merge-startup.log`, and
  `build/mac-vcpkg/log.txt`. These are local artifacts, not committed evidence.
- Added Windows Win32 configure/build presets and a GitHub Actions workflow that
  builds both console and GUI subsystem variants. The workflow has not run yet.

## Before merging on GitHub

1. Publish the reviewed integration commit to the PR's `mac` branch (no force
   push is needed while its head is still `9bc4eda`).
2. Require the Windows build workflow to pass for both executable variants.
3. On Windows, launch from the staged Release directory; check menus, a match,
   keyboard/gamepad input, audio, and exit. Check the OpenGL adapter/version log.
4. On Mac, inspect the texture warning and verify menus, a match, input, and exit.

The integration is prepared locally; Windows compilation/runtime and full Mac
visual validation remain outstanding. Do not treat the Mac build as MSVC proof.
