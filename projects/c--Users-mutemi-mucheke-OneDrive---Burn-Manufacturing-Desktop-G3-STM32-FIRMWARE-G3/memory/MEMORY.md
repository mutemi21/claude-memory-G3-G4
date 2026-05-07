# G3 STM32 Firmware - Project Memory

## Working Style
- Before generating a detailed answer, generate high-level answers first to ensure we are on the same page.
- When troubleshooting, do not guess if you are not sure. Ask me to run a specific test to help gather enough data to solve the problem.
- Always consider how a bug fix or feature will affect both CHK and Highway drivers.

## IMPORTANT: Before Starting Any Task
**Always read `codebase_summary.md` in this memory directory before solving any task.** It contains a complete file-by-file breakdown of the entire codebase, data flow diagrams, and a "Where to Start" table for common tasks. If the information needed is not there or appears outdated, scan the relevant parts of the codebase first, then update `codebase_summary.md` with the new information before proceeding.

**For anything CHK-protocol or CHK-driver related, also read [`chk.md`](chk.md) first.** The original supplier doc (`UART_Protocol_CHK_V2 - EngVer 2.docx`) has gaps and ambiguities — undocumented exception codes, no glass-over-temp signal, an `0x20` returned-data-length value that contradicts what the board actually emits, etc. `chk.md` is the corrected/extended reference combining the supplier doc with everything we have observed in firmware testing. The same CHK power board is used on G4, so this file is mirrored at `~/.claude/projects/c--Users-...-G4-FW-STM32-FIRMWARE-G4/memory/chk.md`. **When updating `chk.md`, update both copies.**

## Project Overview
STM32G0 induction cooker firmware with credit/token-based usage system.
Branch: EHD-1537-G3-Custom-Display-Mapping

## Architecture: MVP Pattern
- **Model**: `ProductFeatures/UI/Src/UIModel.c` — power, timer, error state
- **Presenter**: `ProductFeatures/UI/Src/UIPresenter.c` — 19-state state machine
- **View**: `ProductFeatures/UI/Src/UIView.c` — renders to 7-segment display
- **Display driver**: `ExternalPeripherals/Display/Src/Display.c` — VK16K33 via I2C (0xE0)

## Key Files
- UI states/transitions: `ProductFeatures/UI/Docs/UITransition.md`
- Character maps: `ExternalPeripherals/Display/Inc/NewShineCharMap.h`, `BaoLaiYaCharMap.h`
- Power levels enum: `CookerPower_e` in `ExternalPeripherals/PowerBoard/Inc/PowerBoard.h`
- Error codes enum: `CookerError_e` in `ExternalPeripherals/PowerBoard/Inc/PowerBoard.h`
- Keypad buttons: `ExternalPeripherals/Keypad/Inc/Keypad.h`

## UI States (19 total)
START_UP → STANDBY → SHOW_ECOA_MESSAGE → SHOW_ON_MESSAGE → SHOW_ZERO_POWER → USER_COOKING
- From USER_COOKING: timer, energy info, child lock, error screens

## Display
- 7-segment, max 7 chars visible
- Power bars: 0–10 (mapping COOKER_POWER_0W=0 to COOKER_POWER_2000W=10)
- Icons: HOUR, LOCK, KWH, MONTH, MIN

## Main Loop (Core/Src/main.c)
IndCooker_Process → UIPresenter_Process → UIView_Process → Keypad_Process → Bluetooth/GSM
Tick (~100ms): IndCooker_Tick, UIPresenter_Tick, Led_Tick, SessionSummary_Tick

## Details
See `codebase_summary.md` for complete file-by-file breakdown of every module.
See `ui_architecture.md` for full UI details.
See `features.md` for implemented features.
See `bugfixes.md` for bug fixes.
- [CHK Driver Protocol](chk_driver_protocol.md) — UART protocol, wake-up sequence, commanded vs reported state
