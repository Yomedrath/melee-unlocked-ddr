@AGENTS.md

## Claude-specific notes

- The decomp submodule is usually empty in a fresh session. Run
  `git submodule update --init sourceport/extern/melee` before searching gameplay code, then
  `python tools/prepare_native_sources.py` if you need the patched (shipping) sources.
- Searching the decomp: it has about 850 files under `src/melee/`. Start from
  `docs/gameplay-function-map.md` instead of broad greps. Useful entry points:
  `gm/gmscdata.c` (mode and scene tables), `mn/mnmain.c` (main menu),
  `ft/fighter.c` (per-frame fighter pipeline), `ft/kinds/ftCommon/` (shared moves).
- You cannot build or run the game here (Windows + MSVC + ISO). Say so when a change is
  unverified at run time, and name the check the user should run.
- Work for the DDR mode follows `docs/ddr-mode-plan.md`. Update its phase checklist when a phase
  lands.
