# Project Notes: Custom Handheld Tracker

Personal working notes for this fork of [polyend/tracker-ce](https://github.com/polyend/tracker-ce).
First written 2026-11-02 from a read of the codebase. Nothing has been built or tested yet.

> **For Claude (or anyone) picking this up:** read this file first. It records what was learned
> on the first pass, the decisions so far and the open questions. Build and run work happens
> only on the home machine.

---

## Goal

1. Build my own battery-powered handheld that runs Tracker CE as a Tracker Mini equivalent.
   It should be more portable than the Mini, possibly with a Game Boy-style form factor.
2. Use this firmware as the base for my own music tool. The focus is **sampling and the tracker
   workflow/layout**, not synth engines. I'll find or write synth engines separately.

What I own: an **original Polyend Tracker** (MK66). No Mini or Tracker+.

---

## Home machine setup

Clone somewhere **not** synced by Box, OneDrive or Dropbox; cloud sync and `.git` folders don't mix well.

```bash
git clone -b dev https://github.com/jrb102292/tracker-ce.git C:/dev/tracker-ce
cd C:/dev/tracker-ce
git remote add upstream https://github.com/polyend/tracker-ce.git
```

**Branches:** `main` mirrors Polyend's code. `dev` is my work branch.
To pull in Polyend's updates later:

```bash
git fetch upstream
git checkout main && git merge upstream/main && git push
git checkout dev && git merge main
```

**Toolchain (Windows):**
1. Install Git and Python 3.
2. Download **GNU Arm Embedded Toolchain 10.3-2021.10** (Windows ZIP; link in README.md) and
   extract it so that `agt/10.3/bin/arm-none-eabi-gcc.exe` exists inside the repo. The Makefile
   picks it up automatically, and `agt/` should be git-ignored; check `.gitignore` and add it if not.
3. Install **make** via MSYS2 (`pacman -S make`). The Makefile uses `find`, `cp`, `nproc` and
   `date`, so run builds from an MSYS2 or Git Bash shell, not PowerShell.

On macOS or Linux, just extract the toolchain and run make.

**Build and flash on my original Tracker:**
```bash
make original      # -> build/PolyendTracker_CE_<ver>.ptf
```
Copy the `.ptf` to the SD card's `/Firmware/` folder, then go to **Config > Firmware > Firmware update**.
Keep `.ptf` filenames at 31 characters or fewer. If a build won't boot, put a known-good `.ptf` on the
card and press the recessed reset/bootloader pinhole button.

**First milestone:** build stock firmware, flash it, confirm it boots. Then make one small visible
change (e.g. a text string) to prove the edit, build and flash loop works.

---

## How the code is organized

| Path | What it is |
|---|---|
| `core/` | The whole Tracker app (sequencer, audio engine, UI, file manager). **Shared by all 3 devices.** ~230 .cpp files. |
| `core/include/Platform.h` | **The hardware boundary.** `core/` reaches hardware only through these interfaces (`Battery`, `SwitchGroup`, `Encoder`, `DisplayDriver`, `Sgtl5000`, `Timer`, MIDI, USB, `FileSystem`). A new board means implementing these. |
| `platforms/mk66/` | Driver layer for the original Tracker (Kinetis MK66, Cortex-M4F @ 180 MHz, 16 MB SDRAM). |
| `platforms/imx/` | Driver layer for Tracker+ and Mini (i.MX RT1051, Cortex-M7 @ 600 MHz, 64 MB SDRAM). |
| `platforms/imx/source/hardware.cpp` | Board bring-up: which chip sits on which bus, and the Mini vs Plus differences. |
| `platforms/imx/board/pin_mux.c` / `PlatformTrackerIMX.mex` | Full pin map. The `.mex` file opens in NXP MCUXpresso Config Tools. |
| `synth/` | Synth SDK headers (open) plus **closed-source** `libPolySynth.a` / `libPolyDSP.a`. |
| `editor-fx-sdk/` | `.ptfx` Sample Editor plugin SDK (MIT licensed). |
| `docs/developer_guide.md` | **Read this.** Memory budgets, gotchas, conditional compilation. |

**Mini vs Plus** is just the compile flag `-DTRACKER_MINI` (set in `platforms/imx/makefile`).
Both use the same `pin_mux.c`. Mini drops the pads, LEDs, jog wheel and FM radio, and adds a
battery gauge and extra buttons.

**Platform macros in `core/`:** use `#ifdef PLATFORM_IMX`, `#ifdef TRACKER_MINI` and `#if GUI_HANDLE_SYNTH`.
Do **not** use `__MK66FX1M0__`; it's defined on both platforms for legacy reasons.

---

## Tracker Mini hardware (as the code shows it)

| Function | Part | Bus / pins |
|---|---|---|
| Processor | NXP **MIMXRT1051CVL5B** (Cortex-M7, 600 MHz) | — |
| Code flash | 16 MB QSPI NOR | FlexSPI A (GPIO_SD_B1_06..11) |
| Sample RAM | **64 MB SDRAM** | SEMC (GPIO_EMC_00..41) |
| Display | **FT812** graphics controller, 800×480 RGB panel (`kConfig800_480`, WF70A2TIAGDNN0) | LPSPI4 (GPIO_B1_05/06/07), CS = GPIO2_IO20, PD = GPIO2_IO30 |
| Audio codec | **SGTL5000** | SAI1 I2S (GPIO_B0_13, B1_00..03) + I2C on LPI2C2 |
| Buttons | **TCA8418** keypad scanner (I2C 0x34, up to 80 keys) | LPI2C2 (GPIO_B0_04/05), INT = GPIO2_IO25 |
| Battery gauge | **MAX17262** fuel gauge (firmware assumes 4310 mAh, 3.0 V cutoff) | LPI2C4 (GPIO_AD_B0_12/13) |
| Power | Latching power switch: POWER_SET / POWER_RESET / POWER_BUTTON | GPIO1 |
| SD card | microSD, 4-bit | USDHC1 (GPIO_SD_B0_00..05) |
| MIDI | UART MIDI | LPUART5 (GPIO_B1_12/13) |
| USB | Device: audio, MIDI, mass storage, CDC | USB OTG1 |
| Other | Audio output enable GPIO2_IO24, bootloader reset button GPIO3_IO27, status LED GPIO3_IO26 | |

The charger IC isn't visible in the code; firmware only reads charging state through the MAX17262.

Memory map (`platforms/imx/linker/TrackerMini/memory.ld`): app flash starts at **0x60020000**
for the Mini and 0x60030000 for the Plus. The README only mentions 0x60030000.

---

## Blockers and constraints

1. **Closed-source synth libraries can't be used on my hardware.** `synth/lib/LICENSE-PROPRIETARY.txt`
   allows `libPolySynth.a` / `libPolyDSP.a` only on *authentic Polyend hardware*. A custom board
   needs a build without them. `FEATURE_SYNTH` is set for every IMX build in `core/makefile:57`, and
   the libraries link in `platforms/imx/makefile:83`. Turning that off, or keeping only open-source
   engines (see `core/synth/engines/`, e.g. CHIP), is a required firmware task. Sampling, sequencing
   and FX are open source and unaffected.
2. **Polyend's i.MX bootloader is not in the repo.** App images assume it sits at the bottom of flash.
   A custom board needs its own FlexSPI boot header (FCB/IVT) and either a small bootloader that reads
   `.ptf` from SD, or direct flashing over SWD with the app relinked to 0x60000000.
3. **License: PolyForm Noncommercial.** Personal and hobby builds and modifications are fine, and so is
   making or selling *music* with it. Selling boards or firmware based on this code is not allowed.
   Keep the `Required Notice:` line and the license with any copy I share.

---

## Hardware direction (not decided)

- **Least firmware work:** clone the Mini's parts and pins as closely as possible
  (RT1051 or RT1062 + SGTL5000 + FT812 + TCA8418 + MAX17262).
- **Screen:** the UI is drawn at **fixed 800×480 coordinates** everywhere (e.g. `800 / 8` column
  tables in `core/interface/patternEditor/gPatternEdit.cpp`). Rewriting it for a smaller resolution
  means redoing every screen. Plan: keep 800×480 on a **4.3–5" panel**, which still fits a Game
  Boy-sized body. Small fonts will be tiny; test readability early. There's an unused
  `kConfig240_320` profile in `FT81x_Config.h` (not a realistic UI target).
- **Buttons:** Mini firmware expects about 43 logical buttons (`interfaceDefs.h`, ids 0–42). The TCA8418
  can scan 80 keys, so physical buttons are cheap. The trade-off is face buttons vs shift layers.
  All physical-to-logical mapping is one table: `convertToControlButtons` in
  `platforms/imx/source/ButtonsImpl.cpp`.
- **Prototype before designing a PCB:** NXP RT1050 or RT1060 EVK + SGTL5000 breakout + FT812
  display module + TCA8418 and MAX17262 breakouts. EVKs have 32 MB SDRAM (not 64 MB), so lower
  `SAMPLE_MEMORY_SIZE` in `core/source/mtStructs.h` (see `docs/developer_guide.md` §1 for the rules).

---

## Firmware ideas (sampling and tracker focus)

All of these can be developed and tested on my **original Tracker** today, because `core/` is shared.

### More lanes per step
Current step = note, instrument, volume, micromove, **2 FX slots** (`kStepFxSlots = 2`,
`core/source/mtSequencer.h:36`). 8 sample tracks + 8 MIDI tracks (`core/source/mtStructs.h:87-90`),
up to 128 steps.

- **Option A, widen the step to 3–4 FX.** Simple constant change, but patterns are saved as raw
  structs, so it breaks project compatibility (needs format versioning and a converter). It also
  squeezes the 100 px track columns and costs RAM, which is tight (only ~22 KB free in IMX DTCM;
  less headroom on MK66).
- **Option B (preferred), separate per-track automation lanes.** Drawn parameter curves (cutoff,
  pan, sample start, etc.) stored as a new block next to patterns. Old projects still load, it's
  small (~128 B per lane per pattern) and it gets its own screen. It's also musically bigger than a
  third FX column.
- Free screen slot: `modules[11]` in `core/interface/core/interface.cpp` is a reserved `nullptr`
  (the removed NES emulator), and `interfaceButtonGame` is unused. Good home for a lane editor.
  Don't shift `modules[]` indices.

### Sampling
- Samples ≥ 256 KB already stream from SD (`core/AudioEngine/VoiceStreamManager.cpp`), which opens
  the door to long-sample features (resample a whole song, tape-style recording).
- `.mod` / `.it` import exists (`core/FileManager/fileManagerImportMod.cpp`, `...ImportIt.cpp`), so a
  sample-chain or "kit from folder" importer would sit next to it.
- Slice detection exists (`core/source/mtSliceDetector.cpp`, `mtSliceManager.cpp`), so step-locked
  slice select and slice randomization can build on it.
- `.ptfx` Sample Editor plugins (`editor-fx-sdk/`, `docs/modular_effects_system.md`) are the
  lowest-friction way to add offline sample processing without touching core.

---

## Open questions

- [ ] Handheld orientation: portrait (DMG-style) or landscape (GBA-style)?
- [ ] Screen: which 4.3–5" 800×480 panel, and does it work with the FT812 timings?
- [ ] Physical button count vs shift layers. Map out which of the ~43 logical buttons matter most.
- [ ] Battery and charger IC choice (MAX17262 gauge + which charger?), and target runtime.
- [ ] RT1051 vs RT1062 (the RT1062 is easier to buy and has more internal RAM; needs pin and clock config checks).
- [ ] How to boot: custom bootloader that reads `.ptf` from SD, or SWD flashing only?
- [ ] Automation lanes: which parameters, what resolution, interpolation modes.

## Log

- **2026-11-02:** Forked repo, read the architecture and the IMX driver layer, and wrote these notes. No builds yet.
