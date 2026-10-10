# DDR mode: plan

A rhythm game mode built on Melee's engine. You pick any character and stage. Arrows scroll to
the stage's music, and you must hit **exactly** the charted input on the beat. Each correct step
makes your fighter perform a real Melee move, or one of a set of **extra moves** ("dance moves")
that normal play can't reach. Free movement is limited: the fighter dances on a spot instead of
running around the stage.

Status: **planning**. Background: `docs/gameplay-function-map.md`, `docs/adding-a-1p-mode.md`.

---

## Goals

1. **All characters and all stages.** Reuse the normal CSS and SSS. Every fighter the CSS offers
   works.
2. **Real engine.** Moves are the fighters' own motion states, animations, hitboxes and physics,
   entered through the engine (`Fighter_ChangeMotionState` and the moves' own enter functions). No
   prerecorded video and no fake models.
3. **Limited movement.** The player never drives the fighter freely. Only charted steps produce
   motion, and the fighter stays near its dance spot.
4. **Exact inputs.** A note is hit only if the exact input is new on a frame inside the timing
   window, with no extra buttons and the stick in the right zone. Melee's own leniency (buffering,
   Z as LR+A, tap-jump) does not apply to judging.
5. **Extra moves.** Input sequences unlock states the fighter can't normally do on demand:
   taunts, dizzy, teeter, victory poses, item swings, and so on.
6. **Modified graphics.** A rhythm overlay with lanes, arrows, judgements, combo and life bar,
   and options for the scene (dim or spotlight the stage, fixed camera, hide the percent HUD).
7. **Off means retail.** With the option off, nothing changes. The mode never runs online, in a
   pending online match, or during replay playback.

## Not goals (for now)

- Online play. The judge is game state, but Slippi peers wouldn't run it.
- Shipping music or art from the game. Charts are our own data and refer to the BGM on the
  user's ISO by id.
- New 3D models or animations that aren't already in the user's ISO.

---

## Architecture

```
GAME DLL (deterministic, 60 Hz)                         HOST (display only)
-----------------------------------------------------   ---------------------------------
mu_ddr.c        mode table on GM_HANYU_SSS,             gx/ddr_view.cpp
                CSS -> SSS -> match -> results            ImGui lanes, arrows, judgement,
                rules.on_frame_start / on_frame_end       combo, life (sub-frame scroll)
mu_ddr_chart.c  chart bytes -> fixed arrays             scene options: dim, spotlight,
                (beats -> frames)                         beat pulse
mu_ddr_judge.c  raw pad -> exact-match judge   == ABI   PC settings: DDR option,
                -> score / combo / life        mu_ddr_    audio offset
mu_ddr_moves.c  judged step / sequence ->      view.h ==>
                real engine move
mu_ddr_gate.c   input gate + dance-spot anchor   <== chart file bytes at mode start
                (mu_lab_input hook)
```

### Where each piece hooks in

| Piece | Hook | Why there |
|---|---|---|
| Mode | Runtime rewrite of the `GM_HANYU_SSS` entry in `gm_GetAllGameModes()`, done like `mu_slippi_scene_install()` | No enum change, `GM_COUNT` stays put |
| Menu row | `mn_80229938` / `mn_8022D7F4` / `gmmenumode.c onEnter`, inside `#ifdef MU_NATIVE` | The only way into a 1P mode. See open question 1 |
| Per-frame logic | `StartMeleeRules.on_frame_start` and `on_frame_end`, set in the match state's `on_enter` | Runs at the top and bottom of every match frame with no decomp edit (`gm_Scene_Vs_OnFrame`) |
| Judge input | `HSD_PadGameStatus[port]` read raw in `on_frame_start` | Exact-input judging needs the pad before the game's deadzones and Z-to-LR+A mapping |
| Fighter input | `mu_lab_input()` in `Fighter_Spaghetti_8006AD10`, extended with a DDR branch | The engine's own input path. Returning neutral input stops free movement |
| Moves | Each move's enter function (e.g. `ftCo_Squat_Enter`, `ftCo_KneeBend_Enter`; some, like the forward smash, are still unnamed and are entered from their `_CheckInput`) or `Fighter_ChangeMotionState(msid)`, and `ftData_Special*[kind]` for B moves | Real states, so the real animations and hitboxes |
| Display | New ABI struct `port/runtime/abi/mu_ddr_view.h`, filled by the game each frame and drawn by the host | Same pattern as `mu_lcancel_view.h` and the Lab view. The display only reads |
| Music | `lbAudioAx_80023F28(bgm_id)` | The game's own BGM start. The chart names the id |

### Timing and judging

- Time is **game frames since the chart started**: `gm_GetFrameCount()` minus the start frame.
  It is deterministic, so savestates and the Training Lab's snapshot keep working.
- A chart is written in beats: `frame = offset_frames + beat × 3600 / bpm`. It is converted once
  at load into an integer frame and a fraction.
- Windows, in frames at 60 Hz. These are starting values to tune:

  | Judgement | Window | Approx. |
  |---|---|---|
  | Marvelous | ±1 | ±17 ms |
  | Perfect | ±2 | ±33 ms |
  | Great | ±5 | ±83 ms |
  | Good | ±8 | ±133 ms |
  | Miss | outside, or wrong input | |

- An **exact input** is the set of buttons newly pressed on that frame plus the stick zone. It
  must equal the note's set:
  - Buttons: A, B, X, Y, Z, L, R (digital), D-pad ×4.
  - Stick zones: neutral, or 8 directions. A direction counts when the stick is past 0.8 on its
    axis and under 0.3 on the other (cardinals). Diagonals have their own thresholds. Values
    between zones count as no direction, so a sloppy diagonal is never read as a cardinal.
  - Any extra new press inside the window is a miss ("only exact inputs").
- A player audio offset (in whole frames, PC settings) shifts all note frames. It is a constant,
  so the judge stays deterministic.

### Movement limits

The DDR branch of the input gate makes the human fighter's `fp->input` **neutral every frame**.
The only exception is the one frame on which a judged step fires a move. Without input, the
fighter can't walk, dash, jump or shield on its own. Moves are started directly (next section),
so the input gate doesn't need to fake multi-frame inputs such as smash taps.

The **dance spot** is set at match start (the spawn point). After a step, a soft anchor in
`on_frame_end` pulls the fighter back to the spot over a few frames, and it is never allowed to
leave the stage's ground. Knockback still happens if something hits the fighter, but in the solo
setup nothing does.

### Steps and extra moves

A data table (`mu_ddr_moves.c`) maps inputs to moves. There are three layers:

1. **Basic steps.** These are what most notes are.

   | Note | Move |
   |---|---|
   | ← / → | Turn to face, then a short step (walk state, a few frames) |
   | ↑ | Jump in place (`ftCo_KneeBend_Enter` → `ftCo_MS_JumpF`) |
   | ↓ | Crouch (`Squat`) |
   | A | Jab |
   | A + direction | Tilt |
   | B, B + direction | Special |
   | Z, L, R | Grab, spot dodge, shield pose |

2. **Extra moves**, triggered by **exact sequences** of judged notes (for example ↓ ↓ A, all
   Perfect or better). They enter states normal play only reaches by circumstance:
   `ftCo_MS_AppealSR/SL` (taunts), `ftCo_MS_Furafura` (dizzy), `ftCo_MS_Ottotto` (teeter), `Squat` hold, `ftCo_MS_Sleep`,
   swing states (`ftswing.c`, `ftlipstickswing.c`, `ftstarrodswing.c`) and victory poses
   (`ft/ftdemo.c`). Each one needs testing on every character: some states expect an item, a
   ledge or a partner and must be given a safe setup or left out.
3. **Per-character overrides.** These handle fighters whose moves don't fit the general table:
   Ice Climbers (Nana follows Popo), Zelda/Sheik (down-B transforms), Kirby (neutral-B swallows),
   Game & Watch, and Peach (float).

### Graphics

- **Overlay** (host, ImGui, like `lab_view.cpp`). Lanes, scrolling arrows, a target line, the
  judgement and combo pop-ups, the life bar and the score. Arrows are drawn as shapes, so no art
  ships. The scroll position is interpolated at sub-frame time from the latest game frame, so it
  is smooth at 144/240 Hz without touching the simulation.
- **Scene options** (host renderer, display only). Dim the stage, a spotlight on the dancer, a
  beat pulse on the clear colour, and hiding the percent HUD. A fixed camera (as in
  `gmfixedcamera.c`) is a gameplay-side setting, made in the match's rules.
- **Results.** A host-drawn summary when the chart ends: the count of each judgement, max combo
  and grade. Writing to the memory card comes later, if ever.

### Charts and music

- One file per song, in a `DDR/` folder next to the game (as with `Lab/`). Format (JSON or a
  simple text format, to decide):
  `{ "bgm": 23, "title": "...", "bpm": 150.0, "offset_frames": 42, "difficulty": "basic",
     "notes": [ { "beat": 4.0, "input": "A" }, { "beat": 4.5, "input": "DOWN+B" } ] }`
- Which chart plays: the one for the stage's BGM by default, so all stages work as soon as a
  chart exists for their song. A song-select screen comes later.
- The host reads the file and passes its bytes to the game through the host API at mode start.
  The game parses them into fixed arrays, so no host pointers reach game memory.
- A later tool, `tools/ddr_autochart.py`, could estimate the BPM and onsets from the user's own
  ISO locally. Charts made from it contain no audio, so they can be shared.

---

## Phases

Each phase ends with a check that doesn't depend on the next phase. Tick a phase here when it
lands.

- [x] **P0: Docs and plan.** `AGENTS.md`, `CLAUDE.md`, the function map, the 1P mode guide and
      this plan.
- [ ] **P1: Skeleton mode.** Option bit, `mu_ddr.c` installs the mode on `GM_HANYU_SSS`, a 1P
      menu row, CSS → SSS → match (one human, no CPU, no time limit, items off, stock 1) → back
      to SSS. B on the CSS returns to the menu with the cursor on the row.
      *Check:* option off: `validate_native.py` passes and the 1P menu is retail. Option on: the
      flow works on three stages; Training and Online Play still work.
- [ ] **P2: Movement limits.** DDR branch in the input gate, dance-spot anchor.
      *Check:* on every character, holding any input does nothing. A scripted run over all
      characters and legal stages doesn't crash or fall off.
- [ ] **P3: Chart and judge (no visuals).** Chart parser, beat → frame conversion, exact-input
      judge, score, combo and life, written to the log.
      *Check:* unit tests in `sourceport/game/tests/` feed synthetic pad sequences to the judge.
      They are pure C and run with plain GCC, on Linux too. Edge cases: two notes in one window,
      chords, extra presses, stick between zones.
- [ ] **P4: Steps → moves.** Basic step table, then extra-move sequences, then per-character
      overrides.
      *Check:* a scripted chart drives every step on every character. Record a per-character
      table of works / fallback / excluded in this doc.
- [ ] **P5: Overlay.** `mu_ddr_view.h` ABI, `gx/ddr_view.cpp` lanes and judgements, sub-frame
      scroll, and scene options.
      *Check:* screenshots at 60 and 240 Hz. The overlay is drawn on both D3D12 and D3D11.
- [ ] **P6: Music sync.** Start the BGM at chart frame 0 and add an offset calibration screen.
      **Measure drift** between the 60 Hz simulation and the audio clock over a whole song.
      *Check:* a clap track at a known BPM stays within ±1 frame of its notes after 3 minutes.
      If it doesn't, judge against the audio position the host reports (`mu_audio`) and record
      that decision here.
- [ ] **P7: Content.** Starter charts for the six legal stages' songs at two difficulties, plus
      the chart format doc.
- [ ] **P8: Polish.** Results screen, a two-human dance-off (P1 and P2 on two spots), launcher
      and PC settings integration, release notes.

---

## Open questions

1. **Menu entry.** The hidden 1P row is Slippi's "Online Play" when Slippi menus are on. The
   choices are: (a) use that row only when Slippi menus are off (fastest, fine for a
   prototype); (b) a submenu like Slippi's online page; (c) a sixth row, which needs new menu art
   shipped as a game-file diff. See `docs/adding-a-1p-mode.md` step 3.
2. **Arrow inputs.** D-pad (closest to DDR, and free during a match except the taunt), the
   control stick (uses Melee's analog feel), or both as separate difficulty styles?
3. **Opponent.** Solo first (recommended). Later, a CPU "backup dancer" or a two-human
   dance-off. Should hitboxes connect between dancers?
4. **Song choice.** The stage decides the song (simplest, all stages at once), or a separate song
   list?
5. **Failing.** Does an empty life bar end the song (DDR style), or does the song always finish?

## Risks

| Risk | Mitigation |
|---|---|
| Audio clock drifts from the 60 Hz simulation over long songs | Measure in P6 before writing many charts. Fall back to audio-position judging |
| Some states crash or soft-lock outside their normal context (item states, Ice Climbers, transformations) | Per-character table in P4, safe setups, exclusion list |
| Menu art for a new row can't ship | Start with option 1(a), decide later |
| Can't build or run on Linux or in CI | Keep the judge and parser as pure C with unit tests. Runtime checks need a Windows build with an ISO |
| Upstream decomp bumps change function names | All decomp edits stay small hooks inside `melee-native.patch`. Logic lives in shims |
