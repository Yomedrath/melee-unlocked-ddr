# Adding a game mode to the 1P menu

How to add a new mode, reached from the 1-Player menu, to the Source Port. It assumes you have
read sections 1, 2 and 5 of `docs/gameplay-function-map.md`. The running example is the rhythm
mode from `docs/ddr-mode-plan.md`, called `DDR` here.

A mode needs four things:

1. A **mode table entry**: a `GameMode` with a `GameModeState` list (CSS → SSS → match).
2. **State callbacks** that fill each scene's enter data and read its exit data.
3. A **menu row** that sets `pending_mode` and ends the menu scene.
4. A **way back**: the menu puts the cursor back on your row when the mode exits.

Everything below runs only when the feature's option bit is set (see step 0). With the bit
off, the game behaves exactly like retail.

---

## Step 0: an option bit

Add a flag to `sourceport/game/include/mu_native.h` next to the other `MU_OPTION_*` values, in
whichever of `mu_game_options()` / `mu_game_options2()` has a free bit. Then add a predicate:

```c
/* mu_native.h */
int mu_ddr_enabled(void);   /* host set the DDR option at boot */
```

The host side (PC settings or launcher, under `port/runtime/` and `port/app/`) sets the bit.
Until that exists, an environment variable read once in the shim is fine for development, as
`MELEE_TEST_CPU_FALL` does in `mu_lab.c`.

## Step 1: pick a mode id

There are two ways. **Prefer A.**

**A. Repurpose an unused debug major at run time.** This is what Slippi does with
`GM_HANYU_CSS` (`mu_slippi_scene_install`). It needs no enum change, does not shift `GM_COUNT`,
and touches nothing that compares mode numbers. `GM_HANYU_SSS` (0x09) is free at the time of
writing. Grep `sourceport/game` and `port/runtime` to confirm before you claim it.

```c
#define GM_DDR GM_HANYU_SSS   /* repurposed while mu_ddr_enabled() */
```

**B. Add a new enum value** `GM_DDR` just before `GM_COUNT` in `gm/forward.h`, and add a row to
`modes[]` in `gm/gmscdata.c`. This is cleaner to read, but `GM_COUNT` moves from 0x2D to 0x2E.
Before you do it, audit every array sized or terminated by `GM_COUNT` (`lbdvd.static.h`,
`lbCardGame_DecideGameMode`, the shims' `!= GM_COUNT` loops) and any literal `0x2D`.

## Step 2: the mode (new file `sourceport/game/shim/mu_ddr.c`)

Copy the shape of `gm/gmtrainingmode.c`: CSS, then SSS, then the match. Use `GS_VS` for the
match, not `GS_TRAINING`, because the training scene brings its own pause menu and HUD.

```c
static CSSData ddr_css;
static SSSData ddr_sss;
static StartMeleeData ddr_start;
static MatchExitInfo ddr_exit;

static GameModeState ddr_states[] = {
    { 0, lbDvdPreload_3, 0, ddr_css_enter,   ddr_css_exit,   { GS_CSS, &ddr_css,   &ddr_css } },
    { 1, lbDvdPreload_3, 0, ddr_sss_enter,   ddr_sss_exit,   { GS_SSS, &ddr_sss,   &ddr_sss } },
    { 2, lbDvdPreload_3, 0, ddr_match_enter, ddr_match_exit, { GS_VS,  &ddr_start, &ddr_exit } },
    { -1 },
};

void mu_ddr_install(void)   /* once at boot: call it next to mu_slippi_scene_install() in the patched gm_801A4510 */
{
    GameMode* m;
    if (!mu_ddr_enabled()) return;
    for (m = gm_GetAllGameModes(); m->kind != GM_COUNT; m++) {
        if (m->kind == GM_DDR) {
            m->preloaded = true;
            m->on_init = ddr_on_init;  /* defaults, like gm_Mode_Training_OnInit */
            m->on_load = ddr_on_load;
            m->on_unload = NULL;
            m->states = ddr_states;
            break;
        }
    }
}
```

What each callback does. Model it on the Training function named in brackets:

| Callback | Job |
|---|---|
| `ddr_css_enter` (`gm_801B1B74`) | Set up the CSS: match type (start with `TRAINING_MODE` 0x17: one human plus a CPU slot), the saved fighter picks, preload cache |
| `ddr_css_exit` (`gm_801B1C24`) | If `css->pending_scene_change == 2` (B pressed), `gm_ChangeGameModeAfterCurrentScene(GM_MENU)` and return. Otherwise copy the picks into your saved `VsModeData` and preload fighters and audio |
| `ddr_sss_enter` / `ddr_sss_exit` (`gm_801B1EB8` / `gm_801B1EEC`) | If `!sss->start_game`, `gm_SetNextGameModeStateId(0)` (back to CSS). Otherwise store `sss->vs.start.rules.stkind` |
| `ddr_match_enter` (`gm_801B1F70`) | `gm_SetupRulesDefaults`, `gm_SetupAllPlayerDefaults`, `gm_SetupHumanPlayer` / `gm_SetupCpuPlayer`, then set `rules.stkind`, time/stock and items off. **Install your per-frame logic**: `rules.on_match_start`, `rules.on_frame_start`, `rules.on_frame_end`, `rules.on_match_end` |
| `ddr_match_exit` (`gm_801B2204`) | Read `MatchExitInfo`, store the score, then `gm_SetNextGameModeStateId(0)` to go back to the SSS. To leave the mode instead, `gm_ChangeGameModeAfterCurrentScene(GM_MENU)` |

The CSS and SSS are reused unchanged. To restrict picks (stage list, a fixed CPU), change the
data you hand them in the `_enter` callbacks. Don't fork `mncharsel.c`.

Add `shim/mu_ddr.c` to the game DLL's source list in `sourceport/game/CMakeLists.txt`.

## Step 3: the menu row (decomp edit, `mn/mnmain.c`)

The 1P page has five rows in data (`SEL_1P_REG`, `SEL_1P_EVENT`, `SEL_1P_2`, `SEL_1P_STADIUM`,
`SEL_1P_TRAINING`). Row 2 is hidden in retail. The menu's art (`MnMaMain.dat`, from the user's ISO)
only has panels for these five rows.

**Row 2 is already taken.** With Slippi menus on, it is "Online Play". You have these choices:

| Option | Work | Notes |
|---|---|---|
| Row 2 when Slippi menus are off | Small | Good for a prototype. Can't be used together with Slippi online menus |
| A sub-page, like Slippi's online page | Medium | Reuse an unnamed `MenuKindData` slot such as `MENU_KIND_10` or `MENU_KIND_11` (no data, NULL think; grep first to confirm nothing uses it). Fill its preview animations and description ids from existing tables, as `mu_mnmain_install_online_menu()` does. Still needs a row to open it from |
| A sixth 1P row | Large | Needs new art in the menu archive, which this repo can't ship. It would need a game-file diff (see how `Sys/` GameFiles diffs are applied) |

Whichever you pick, the edit has the same three parts, each in `#ifdef MU_NATIVE`. The
prototype version:

```c
/* mn_80229938: is the row visible */
if (menu_kind == MENU_KIND_1P && selection == SEL_1P_2) {
#ifdef MU_NATIVE
    return mu_slippi_menus_enabled() || mu_ddr_enabled();
#else
    return false;
#endif
}

/* mn_8022D7F4: 1P menu input, inside switch (hovered_selection) */
#ifdef MU_NATIVE
case SEL_1P_2:
    if (mu_slippi_menus_enabled()) { /* existing Online Play branch */ }
    else if (mu_ddr_enabled()) {
        sfxForward();
        data = gm_GetCurrentSceneExitData();
        data->pending_mode = GM_DDR;
        gm_801A4B60();
        return;
    }
    break;
#endif
```

The description line under the menu comes from `mn_803EB6B0[MENU_KIND_1P].description_indices`
(text ids in `SdMenu.usd`). A new string needs a game-file diff, or you can draw the text yourself
with `HSD_SisLib_803A5ACC` and `HSD_SisLib_803A6368` the way `mn_80229A7C` does.

## Step 4: the way back (`gm/gmmenumode.c` `onEnter`)

Add a case so the cursor goes back to your row when the mode exits:

```c
#ifdef MU_NATIVE
    case GM_DDR:
        if (mu_ddr_enabled()) {
            data->menu_kind = MENU_KIND_1P;
            data->hovered_selection = SEL_1P_2;
            return;
        }
        break;
#endif
```

With option A, `GM_DDR` is `GM_HANYU_SSS`, so guard the case with `mu_ddr_enabled()`.

## Step 5: regenerate the patch, build, check

1. `git -C sourceport/extern/melee diff > sourceport/patches/melee-native.patch`. Mark new
   decomp files with `add -N` first.
2. Build `melee_game.dll` and `melee_source.exe` (`docs/build-source-port.md`).
3. Check with the option **off**: the 1P menu looks retail, and `tools/validate_native.py` still
   matches all 2400 checkpoints.
4. Check with the option **on**: the row appears, CSS → SSS → match → back to SSS works, B on
   the CSS returns to the menu with the cursor on the row, and Training and Online Play still work.
5. Make sure the mode never starts during online play or replay playback.

## Decisions to make for each new mode

- **Does it count as a 1P mode?** `gm_IsCurrently1PMode_inline()` in `gm/gmvs.c` decides whether
  the C-stick works, and some HUD and camera behaviour. A repurposed id is not in that list, so the
  C-stick works unless you add it.
- **Records/save data.** Training writes stats through `gm_80162968` and similar functions. A new
  mode should not write to the memory card unless you deliberately add a record format.
- **Pause.** `rules.on_pause_override` and `check_for_pauser_override` control pausing in the match.
