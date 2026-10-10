# Gameplay function map

A guide to the parts of the Melee decomp that decide how the game plays. It covers how the game
moves between menus and matches, how a fighter turns controller input into a move, and where this
project's own code hooks in. Use it to find a starting point, then read the source.

All paths are relative to `sourceport/extern/melee/src/melee/` unless they start with
`sourceport/` or `port/`. Initialise the submodule first (see `AGENTS.md`).

**Reading decomp names.** `gm_801A4B60` is a function the decomp has not named. The hex is its
address in the retail DOL. Prefixes give the module:

| Prefix | Dir | Module |
|---|---|---|
| `gm` | `gm/` | Game modes ("majors"), scenes ("minors"), match flow, results, records |
| `mn` | `mn/` | Menus: main menu, CSS (`mncharsel`), SSS (`mnstagesel`), rules |
| `ft`, `ftCo`, `ftMr`, `ftFx`… | `ft/` | Fighters. `ftCo` is shared ("common") code, `ftMr` is Mario, `ftFx` is Fox, and so on |
| `pl` | `pl/` | Player slots: stocks, percent, stale moves, bonuses, stats |
| `it` | `it/` | Items and projectiles |
| `gr`, `Stage_` | `gr/` | Stages (grounds): per-stage logic, collision lines, BGM |
| `cm` | `cm/` | Camera |
| `ef` | `ef/` | Visual effects (particles, sparks) |
| `if` | `if/` | In-match interface: percent display, timer, stocks |
| `lb` | `lb/` | Library: collision maths, audio (`lbAudioAx`), DVD preload, memory card, commands |
| `mp` | `mp/` | Map (collision geometry) |
| `vi`, `ty` | `vi/`, `ty/` | Cutscenes, trophies |
| `HSD_` | `../sysdolphin/baselib/` | HAL's engine: GObj process scheduler, JObj scene graph, pads, text |

---

## 1. Program flow: modes and scenes

Melee runs one **GameMode** (a "major scene") at a time. Each mode is a list of
**GameModeState** entries, each pointing at one **GameScene** (a "minor scene"), for example
CSS → SSS → match → results.

```
gm_RunGameMode(kind)                 gm/gm_1A3F.c   load mode, loop its states, return next mode
 └─ gm_801A4014(mode)                gm/gm_1A3F.c   one state: on_enter → run scene → on_exit
     ├─ GameModeState.on_enter        fills the scene's enter_data (e.g. CSSData, StartMeleeData)
     ├─ gm_FindGameSceneHandler(k)   gm/gm_1A45.c   looks up the GameScene by GameSceneKind
     ├─ gm_801A4D34(on_frame, info)  gm/gm_1A45.c   per-frame scene loop until gm_801A4B60()
     └─ GameModeState.on_exit         reads exit_data, picks the next state or mode
```

| Symbol | File | Role |
|---|---|---|
| `GameModeKind` (`GM_*`) | `gm/forward.h` | Enum of every mode, `GM_TITLE` (0x00) to `GM_SINGLE_BUTTON_VS` (0x2C), then `GM_COUNT` |
| `GameSceneKind` (`GS_*`) | `gm/forward.h` | Enum of every scene: `GS_CSS`, `GS_SSS`, `GS_VS`, `GS_TRAINING`, `GS_RESULTS`… |
| `struct GameMode`, `GameModeState`, `GameScene` | `gm/types.h` | The three table types above |
| `modes[]`, `scenes[]` | `gm/gmscdata.c` | The global tables. `gm_GetAllGameModes()` and `gm_GetAllGameScenes()` return them |
| `gm_ChangeGameModeAfterCurrentScene(m)` | `gm/gm_1A3F.c` | Leave the current mode for `m` once this scene ends |
| `gm_SetPendingGameMode(m)` + `gm_SetNewGameModePending()` | `gm/gm_1A3F.c` | The same thing in two steps (the menu uses this) |
| `gm_SetNextGameModeStateId(i)` | `gm/gm_1A3F.c` | Inside a mode, jump to state `i + 1` next (`gm_SetNextGameModeStateId(0)` goes back to state 1, e.g. SSS) |
| `gm_GetGameModeStateEnterData/ExitData` | `gm/gm_1A3F.c` | Inside `on_enter`/`on_exit`: the scene's data block |
| `gm_GetCurrentGameMode()`, `gm_GetPreviousGameMode()` | `gm/gm_1A3F.c` | What is running, and what ran before (the menu uses it to put the cursor back) |
| `gm_801A4B60()` | `gm/gm_1A45.c` | Ends the current scene. Its exit handler runs next |
| `gm_GetCurrentSceneExitData()` | `gm/gm_1A45.c` | The running scene's exit data block, e.g. `MenuExitData` |

### Mode example: Training (`gm/gmtrainingmode.c`)

The smallest complete 1P mode. Read it before writing a new one.

| State | Scene | `on_enter` | `on_exit` |
|---|---|---|---|
| 0 | `GS_CSS` | `gm_801B1B74`: sets up the CSS with match type `TRAINING_MODE` (0x17) | `gm_801B1C24`: B on the CSS returns to `GM_MENU`. Otherwise it copies the picks and the CPU and preloads the fighters |
| 1 | `GS_SSS` | `gm_801B1EB8` | `gm_801B1EEC`: Back returns to CSS (`gm_SetNextGameModeStateId(0)`). Otherwise it stores the stage and preloads its BGM |
| 2 | `GS_TRAINING` | `gm_801B1F70`: builds `StartMeleeData`: rules, P1 human, CPUs, pause override | `gm_801B2204`: records stats, then returns to the SSS |

`gm_Mode_Training_OnInit` (once per entry: defaults) and `gm_Mode_Training_OnLoad` are its
`GameMode` callbacks. Other modes to compare: `gmhomerun.c`, `gmmultiman.c`, `gmstamina.c`,
`gmcameramode.c` (that one uses the per-frame callbacks).

### Match setup structures (`mn/types.h`, `gm/gm_1B03.c`)

| Symbol | Role |
|---|---|
| `StartMeleeData` | `rules` + `players[6]`. The VS/Training scene's enter data |
| `StartMeleeRules` | Stage (`stkind`), time/stock, items, flags, and **callbacks**: `on_match_start`, `on_frame_start`, `on_frame_end`, `on_match_end(outcome)`, `on_pause_override`… |
| `PlayerInitData` | One slot: `ckind`, `slot_type` (human/CPU/none), stocks, color, team, handicap, flags |
| `gm_SetupRulesDefaults`, `gm_SetupAllPlayerDefaults` | Zero/default a `StartMeleeData` |
| `gm_SetupHumanPlayer(p, ckind, color, stocks, port)` / `gm_SetupCpuPlayer(...)` | Fill one slot |
| `VsModeData` (`gmMainLib_804D3EE0->modes.unk_D10`) | Saved CSS/SSS picks shared between VS-like modes |
| `CSSData`, `SSSData` | CSS/SSS enter/exit data (`match_type`, `pending_scene_change`, `start_game`, `force_stage_id`) |

### The match loop (`gm/gmvs.c`)

| Symbol | Role |
|---|---|
| `gm_Scene_Vs_OnEnter`, `gm_Scene_Training_OnEnter` | Start a match from `StartMeleeData`: create the stage, players, HUD |
| `gm_Scene_Vs_OnFrame` | Every match frame. Calls `rules.on_frame_start()`, then runs the match (`fn_8016CFE0` while playing, `fn_8016D634` at the end), then calls `rules.on_frame_end()` |
| `gm_Scene_Vs_OnExit` | Fills `MatchExitInfo` (standings, frame count) for the mode's `on_exit` |
| `gm_GetFrameCount()` | Match frame counter (60 per second). Deterministic, so a rhythm judge can use it |
| `gm_IsCurrently1PMode_inline()` | True for Classic, Adventure, All-Star, Stadium, Training and Event. **In these modes the C-stick reads as zero** (see §3) |
| `MatchOutcome` (`OUTCOME_*`) | `gm/forward.h`. Passed to `on_match_end` |

---

## 2. Menus

| Symbol | File | Role |
|---|---|---|
| `MenuKind` (`MENU_KIND_*`) | `mn/forward.h` | Every main-menu page: `MENU_KIND_1P`, `MENU_KIND_REG`, `MENU_KIND_STADIUM`… |
| `OnePlayerMenuSelection` | `mn/forward.h` | 1P rows: `SEL_1P_REG`, `SEL_1P_EVENT`, `SEL_1P_2` (hidden), `SEL_1P_STADIUM`, `SEL_1P_TRAINING` |
| `mn_803EB6B0[]` (`MenuKindData`) | `mn/mnmain.c` | Per page: preview animation, description text ids, row count, and the page's input "think" function |
| `mn_80229938(kind, sel)` | `mn/mnmain.c` | Is a row unlocked/visible. The cursor skips rows that return false |
| `mn_8022D7F4` | `mn/mnmain.c` | **1P menu input.** A on a row either opens a sub-page or sets `MenuExitData.pending_mode = GM_…` and calls `gm_801A4B60()` |
| `mn_8022DB10` | `mn/mnmain.c` | Main (top) menu input |
| `mnMain_Scene_OnEnter/OnFrame` | `mn/mnmain.c` | The `GS_MENU` scene |
| `gmmenumode.c` `onEnter` / `onExit` | `gm/gmmenumode.c` | `onEnter` picks which page and row the cursor starts on from `gm_GetPreviousGameMode()`. `onExit` hands `pending_mode` to the mode runner |
| `mncharsel.c`, `mnstagesel.c` | `mn/` | CSS and SSS scenes |

---

## 3. Fighters: from controller to move

Each fighter is an `HSD_GObj` whose `user_data` is a `Fighter` (`ft/types.h`, `GET_FIGHTER`).
`Fighter_Create` in `ft/fighter.c` registers its per-frame processes (in `Fighter_Create_Inline2`). They run in priority order:

| Prio | Process | What happens |
|---|---|---|
| 0 | `Fighter_8006A1BC` | Hitlag/timers, mushroom effects |
| 1 | `Fighter_8006A360` | Position delta, item/metal timers, **animation advance** (`ftAnim_8006EBA4`), then the state's `anim_cb` |
| 2 | `Fighter_8006ABA0` | CPU only: runs the AI (`ftCo_800B3900`) |
| 3 | `Fighter_Spaghetti_8006AD10` | **Input.** Reads the pad (or the CPU), applies deadzones, sets `fp->input.*`, then runs the state's `input_cb` (the "interrupt"/IASA) |
| 4 | `Fighter_procUpdate` | **Physics**: the state's `phys_cb`, then velocity integration |
| 6 | `Fighter_procMap` | **Collision with the stage**: the state's `coll_cb` |
| 7–9 | `Fighter_8006C5F4`, `Fighter_CallAcessoryCallbacks_8006C624`, `Fighter_8006C80C` | Accessory callbacks, model update |
| 0xC–0xE | `Fighter_UnkProcessGrab_8006CA5C`, `Fighter_8006CB94`, `Fighter_ProcessHit_8006D1EC` | Grabs, then **hit resolution**: damage, knockback, reactions |
| 0x10–0x16 | `Fighter_8006D9AC`, `Fighter_UnkCallCameraCallback_8006D9EC`, `Fighter_8006DA4C` | Camera box, end of frame |

### Input (`Fighter_Spaghetti_8006AD10`)

- Human source: `HSD_PadGameStatus[fp->x618_player_id]` (`nml_stickX/Y`, `nml_subStickX/Y`,
  `nml_analogL/R`, `button`).
- CPU source: `ftCo_800A2040(fp)` is true for a CPU. Its input then comes from `ftCo_GetCpu*`.
- Result: `fp->input.lstick[0]`, `cstick[0]`, `triggers[0]`, `held_buttons[0]`. Index `[1]` holds
  the previous frame. Pressed and released buttons are derived from the two.
  Deadzones come from `p_ftCommonData`. Z also counts as LR+A.
- **1P modes zero the C-stick** (`gm_IsCurrently1PMode_inline()`), so a C-stick smash or aerial
  is impossible there. Decide whether a new mode joins that list.
- Project hook: the patched tree calls `mu_lab_input(fp, &lab)` here first
  (`sourceport/game/shim/mu_lab.c`). When it returns 1, that function's stick/trigger values
  replace the pad. This is how the Training Lab plays back recorded input. **It is the place to
  filter or rewrite a fighter's input.**

### Motion states (action states)

| Symbol | File | Role |
|---|---|---|
| `struct MotionState` | `ft/types.h` | One state: `anim_id`, flags, `move_id`, and the five callbacks `anim_cb`, `input_cb`, `phys_cb`, `coll_cb`, `cam_cb` |
| `ftCommon_MotionState` (`ftCo_MS_*`) | `ft/kinds/ftCommon/forward.h` | Common state ids: `ftCo_MS_Wait`, `ftCo_MS_Attack11`, `ftCo_MS_AttackS4S`, `ftCo_MS_AppealSR`… (0x155 of them) |
| `ftData_MotionStateList[]` | `ft/ftdata.c` | Table for the common states |
| `ftData_CharacterStateTables[kind]` | `ft/ftdata.c` | Per-character tables. A `msid >= 0x155` indexes these (specials, character-only states) |
| `Fighter_ChangeMotionState(gobj, msid, flags, anim_start, anim_speed, anim_blend, …)` | `ft/fighter.c` | **Enter a state**: set the animation, install the five callbacks, reset or keep hitboxes per `flags` |
| `ftData_80085E50(fp, msid)` | `ft/ftdata.c` | The animation (FigaTree) for a state |
| `ftData_SpecialS[kind]` (and `SpecialN/Hi/Lw`, `SpecialAir*`) | `ft/ftdata.c` | Per-character entry functions for B moves |

**How a move is chosen.** In a standing state the `input_cb` is an "IASA" function that tries
each move's check in priority order, for example `ftCo_Wait_IASA` in
`ft/kinds/ftCommon/ftCo_Wait.c`:

```
SpecialS → Attack100 → … → Catch → AttackS4 → AttackHi4 → AttackLw4 → AttackS3 → AttackHi3
→ AttackLw3 → Attack1 → … → AppealS → … → Jump → Dash → … → Turn → Walk
```

Each `ftCo_<Move>_CheckInput(gobj)` reads `fp->input` and the frame timers
(`x670_timer_lstick_tilt_x`, `x671…_y`, `x688`…: frames since the stick or button changed). When
it matches, it calls the move's enter function (usually `ftCo_<Move>_Enter`; some are still
unnamed, e.g. the forward smash's), which calls `Fighter_ChangeMotionState`, and it returns true.
Every common move lives in `ft/kinds/ftCommon/ftCo_<Move>.c` with enter, `_Anim`, `_IASA`,
`_Phys` and `_Coll` functions. Character files are in `ft/kinds/ft<Name>/`.

### Other fighter code

| File | Role |
|---|---|
| `ft/ftcommon.c` | Shared helpers: facing (`ftCommon_UpdateFacing`), velocity clamps, gravity, ground/air transitions |
| `ft/ftaction.c`, `ft/ftcmdscript.c` | Subaction scripts: hitbox timing, SFX, GFX, flags, driven by each animation's command list |
| `ft/ftcoll.c` | Hitboxes/hurtboxes (`ftColl_*`), hit detection between fighters and items |
| `ft/ftanim.c` | Animation playback and blending |
| `ft/ftlib.c` | Queries other modules use: `ftLib_*` (position, percent, facing, is-dead) |
| `ft/ftcpuattack.c`, `ft/kinds/ftCommon/ftCo_0A01.c` | CPU AI |
| `ft/ftcamera.c` | Fighter camera box |
| `ft/ftdata.c` | Data load per character (`Pl<Xx>.dat`), state tables, special-move tables |
| `ft/forward.h` | `FighterKind` (`Ft_Kind_*`, internal ids) and `CharacterKind` (`CKind_*`, CSS ids) are **different orders** |

---

## 4. Players, items, stages, camera, HUD, audio

| Area | Key symbols | Role |
|---|---|---|
| Player slots | `pl/player.c` `Player_*` | Per-slot state outside the fighter: character, costume, stocks, controller index (`Player_GetControllerIndex`), facing, spawn points |
| Stats/staling | `pl/plstale.c`, `pl/plattack.c`, `pl/plbonus.c` | Stale-move queue, attack stats, end-of-match bonuses |
| Items | `it/item.c` `Item_80268B18(SpawnItem*)` | Spawn an item or projectile. `Item_80268E5C` changes an item's state. Per-item code is in `it/kinds/` |
| Stage | `gr/stage.c` `Stage_*`, `gr/ground.c`, `gr/gr<Name>.c` | Stage load, per-stage objects, blast zones, camera limits. `StKind`/`GrKind` are in `gr/forward.h` |
| Collision geometry | `mp/mplib.c` | Floor, wall and ceiling lines and platforms |
| Camera | `cm/camera.c` | Match camera: follows the fighters' camera boxes. `gmcameramode.c` and `gmfixedcamera.c` override it |
| HUD | `if/ifstatus.c` (percent), `if/iftime.c` (timer), `if/ifstock.c`, `if/textlib.c` | The in-match interface. `HSD_SisLib_*` (sysdolphin) draws menu and HUD text |
| Effects | `ef/eflib.c`, `ef/efsync.c` | Particle and sprite effects |
| Music | `lb/lbaudio_ax.c` `lbAudioAx_80023F28(bgm_id)` | Starts a BGM by id (0–0x61, an `.hps` file on the disc). `Stage_*` in `gr/stage.c` calls it when a match starts |
| Sound effects | `lb/lbaudio_ax.c`, `sfxForward/sfxBack/sfxMove` | Menu and game SFX |
| Collision maths | `lb/lbcollision.c` | Hitbox shapes against hurtboxes |
| Engine | `sysdolphin/baselib/gobj.c` (`GObj_Create`, `HSD_GObj_SetupProc`), `jobj.c`, `controller.c` (`HSD_PadGameStatus`) | Process scheduler, scene graph, pads |

---

## 5. Where this project hooks in

These are the existing extension patterns, best first:

1. **Runtime table rewrite (no enum change).** A shim finds a `GameMode` in
   `gm_GetAllGameModes()` and replaces its `states`, `on_load` and `on_unload`.
   - `sourceport/game/shim/mu_slippi_menu.c` `mu_slippi_scene_install()` turns the unused debug
     mode `GM_HANYU_CSS` into Slippi's online flow.
   - `sourceport/game/shim/mu_tmce_event.c` `mu_tmce_ev_scene_lists()` swaps `GM_EVENT` and
     `GM_MENU` state lists while TM-CE is on, and puts them back when it is off.
2. **`#ifdef MU_NATIVE` hook in decomp code that calls a shim predicate.** For example
   `mn_80229938` shows the hidden `SEL_1P_2` row when `mu_slippi_menus_enabled()`, and
   `mn_8022D7F4` routes it to the online submenu. See `sourceport/patches/melee-native.patch`.
3. **Fighter input override.** `mu_lab_input()` (§3).
4. **Match callbacks.** Set `StartMeleeRules.on_frame_start` and the other callbacks from a
   mode's `on_enter`. These need no decomp edit.
5. **Host display.** The game DLL fills a plain C struct (`port/runtime/abi/mu_lcancel_view.h`,
   `mu_native_pose.h`), and the host draws it with ImGui or the renderer. See
   `port/runtime/gx/lab_view.cpp`, `net_overlay.cpp`, `replay_bar.cpp`.
6. **Option flags.** Each feature is gated by a bit in `mu_game_options()`/`mu_game_options2()`
   (`sourceport/game/include/mu_native.h`). The host sets it and it is latched at boot.

Debug majors that no shim uses yet (as of this writing): `GM_HANYU_SSS` (0x09),
`GM_DEBUG_CUTSCENE` (0x19), `GM_DEBUG_GOVER` (0x1A). `GM_HANYU_CSS` belongs to the Slippi online
flow. `GM_DEBUG_VS` (0x0E) is where Slippi and the P2P launcher boot, so it is taken too.

For a step-by-step guide to a new 1P mode, see `docs/adding-a-1p-mode.md`.
