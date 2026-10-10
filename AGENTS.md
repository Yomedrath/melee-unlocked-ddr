# AGENTS.md

Guide for coding agents (and humans) working in this repository. `CLAUDE.md` imports this file.

## What this is

A native Windows build of Super Smash Bros. Melee (NTSC 1.02, game ID `GALE01`) with two engines:

- **Source Port** (main build): the Melee decompilation (`sourceport/extern/melee`, a pinned
  submodule of doldecomp/melee) plus native adaptations, compiled with MinGW GCC 14+ into
  `melee_game.dll`. The MSVC host (`melee_source.exe`) supplies rendering, audio, pads, Slippi.
- **Static Recomp**: the retail PowerPC code translated ahead of time to C++ (`port/recomp/`,
  output in `port/generated/`, never committed). Kept for mods that need original code.

Gameplay work (new modes, moves, rules) belongs in the **Source Port**: it is readable C.

This fork (`melee-unlocked-ddr`) adds a rhythm game mode. See `docs/ddr-mode-plan.md`.

## Layout

| Path | What |
|---|---|
| `sourceport/extern/melee/` | Decomp submodule, pinned at `039c4bf4…`. Empty until initialised. |
| `sourceport/patches/melee-native.patch` | Every edit this project makes to the decomp, as one `git diff`. |
| `sourceport/game/shim/` | Native C that the game DLL links: `mu_*.c` (Slippi menus, TM-CE, 20XX TE, Training Lab, replay, online). |
| `sourceport/game/include/` | Shim headers. `mu_native.h` is force-included in every game TU (option flags, hook prototypes). |
| `sourceport/game/CMakeLists.txt` | Builds `melee_game.dll` and the native unit tests (`sourceport/game/tests/`). |
| `port/runtime/` | Host runtime: `gx/` renderers and overlays (D3D12/D3D11, ImGui, Lab view), `hle/` SDK stand-ins, `host/`, `mu_net/` netcode, `abi/` game↔host structs. |
| `port/recomp/` | Static Recomp translator (Python). |
| `port/app/launcher.cpp` | Optional launcher. |
| `tools/` | Validation, packaging, diagnostics scripts. |
| `docs/` | Feature and build docs. Start with `docs/gameplay-function-map.md` for gameplay code. |

## Getting the decomp sources

```sh
git submodule update --init sourceport/extern/melee
python tools/prepare_native_sources.py   # applies melee-native.patch onto the pinned commit
```

Read the decomp **with the patch applied** when you need to know what actually ships; many hooks
(`#ifdef MU_NATIVE`) only exist in the patched tree.

### Changing decomp code

1. Prepare the sources as above, edit files under `sourceport/extern/melee/src/`.
2. Wrap every change in `#ifdef MU_NATIVE … #endif` with a one-line comment saying why.
3. Regenerate the patch from the submodule diff (new files need `git -C sourceport/extern/melee add -N <file>` first):
   `git -C sourceport/extern/melee diff > sourceport/patches/melee-native.patch`
4. Commit only the patch; never commit the submodule's working tree or move its pinned commit.

Prefer adding new code as a shim in `sourceport/game/shim/` and keeping the decomp edit to a
small hook that calls it. That is how Slippi menus, TM-CE and the Training Lab are done.

## Building and testing

The full build needs Windows x64 (MSVC 2022 + MinGW GCC 14+) and a user-supplied ISO. Steps are in
`README.md` ("Build from source") and `docs/build-source-port.md`. On Linux you can still:

- Run the CI checks (`.github/workflows/source-checks.yml`), e.g.
  `python -m unittest discover -s tools -p release_integration_test.py`
- Build the small native tests in the root `CMakeLists.txt` (`native_animation_test`, etc.).

Verification that needs Windows and an ISO:

- `ctest --test-dir build-review -C Release`
- `python tools/validate_native.py --iso <iso>`: 2400 simulation checkpoints must match.
- `python tools/online_pair.py --script port/scripts/online_bot.txt`: no `DESYNC` in the log.

## Rules that are easy to break

- **No Nintendo data in the repo.** No ISO, DOL, extracted `.dat`, textures, music, or
  `port/generated/`. Assets are always read from the user's ISO at run time.
- **Determinism.** The simulation runs at 60 Hz and must stay bit-exact with Slippi Dolphin for
  online play and replays. New gameplay features must be off in online matches, pending online
  matches and replay playback (see the guards at the top of `sourceport/game/shim/mu_lab.c`).
  Never read wall-clock time, host floats or unseeded randomness from game-side code.
- **Display vs. simulation.** Anything purely visual goes in the host (`port/runtime/gx/`) and
  must only *read* game state (see `port/runtime/gx/lab_view.h`). Anything that changes outcomes
  goes in the game DLL.
- **Options are latched at boot** through `mu_game_options()` / `MU_OPTION_*` in `mu_native.h`.
  With an option off, hook sites must run the original code path unchanged.
- **Pointers are 32-bit slots.** The game image lives below 4 GB; don't store host pointers
  in game structures.
- **License**: GPL-3.0-or-later. Keep third-party notices (`NOTICE`).

## Conventions

- C in the game DLL follows the decomp's style (4-space indent, `clang-format` config in the
  submodule). Name new shim files `mu_<feature>.c`, public functions `mu_<feature>_<verb>`.
- Comments explain *why*, cite the original function address or code name they stand for.
- Commit subjects follow the existing style: `Area: what changed, in plain words`
  (e.g. `Display: CRT display option (studio monitor, home television)`).
- Decomp function names like `gm_801A4B60` are unnamed functions: the hex is the retail
  address. `docs/gameplay-function-map.md` gives the roles of the ones that matter for gameplay.
