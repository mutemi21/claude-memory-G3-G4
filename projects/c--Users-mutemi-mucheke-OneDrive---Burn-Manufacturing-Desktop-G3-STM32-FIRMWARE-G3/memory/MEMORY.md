# G3 STM32 Firmware - Project Memory

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
