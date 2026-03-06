# G4 Induction Cooker - STM32 Firmware (STM32G071CB, Cortex-M0+)

## Before Starting Any Task
1. **Read `CODEBASE_SUMMARY.md`** in the project root first for full file-by-file context, architecture diagrams, and callback chains.
2. If the info you need is **not there**, scan the relevant source files, then **update `CODEBASE_SUMMARY.md`** with the new information.
3. After adding/modifying files, update their entries in `CODEBASE_SUMMARY.md`.

## Full Codebase Summary
- See [codebase_map.md](codebase_map.md) for memory copy of codebase breakdown
- See `CODEBASE_SUMMARY.md` in the repo root for the canonical version

## Architecture
- **MVP pattern**: UIPresenter (state machine) -> UIModel (data) -> UIView (display)
- **PowerBoard dispatch**: PowerBoard.c dispatches to HighwayDriver/CHKDriver/AiferDriver based on `PowerBoardType` (currently hardcoded HIGHWAY)
- **Session tracking**: CookerState (On/Off state pattern) -> CookingSession (accumulates power) -> SessionAggr (daily/monthly flash storage)
- **Callback chain**: PowerBoard drivers fire StatusCallback -> IndCooker's PowerBoardStatusCb -> CookerState_On/Off

## Critical Gotchas
- **Highway SetLevel() triggers StatusCallback** (starts session via CookerState_On). PowerOff() does NOT trigger callback.
- **CHK PowerOff() DOES trigger StatusCallback** (unlike Highway). New PowerOffWithFan must NOT trigger callback.
- **UIPresenter state table**: `ShortPressNextStateID` is set BEFORE `ActionFunctionShortPress` runs. Use `NO_SUPPORTED_STATE` to let action function set state dynamically.
- **UIPresenter_Process()** always resets `StateTimeOutCounter` after key events, overwriting any value set by the action function.
- **CookingSession_Stop()** resets CurrentSession - must get duration BEFORE stopping.
- **IndCooker_Tick() timer** calls `IndCooker_PowerOff()` on expiry - must stop timer during shutdown to prevent killing fan.
- Pre-existing `-Wc23-extensions` warnings on variadic macros throughout codebase - not our issue.

## File Locations
- PowerBoard enum/callbacks: `ExternalPeripherals/PowerBoard/Inc/PowerBoard.h`
- Highway control bytes: `ExternalPeripherals/PowerBoard/Src/HighwayDriver.c` (CookerControl array)
- UI State machine: `ProductFeatures/UI/Src/UIPresenter.c`
- Session aggregation: `ExternalPeripherals/Flash/Src/SessionAggr.c`
