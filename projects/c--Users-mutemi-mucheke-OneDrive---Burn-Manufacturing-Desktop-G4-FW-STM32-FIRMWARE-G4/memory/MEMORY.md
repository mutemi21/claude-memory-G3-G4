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
- **PAYGO/Credit**: See [paygo_system.md](paygo_system.md) for full details. Nexus Keycode lib validates 15-digit tokens. Two modes: RTC (time-based due date) and Device On/Off (server-controlled). Token → Nexus → Shadow_SetDueDate → CreditManagement → UI credit check.

## Critical Gotchas
- **Highway SetLevel() triggers StatusCallback** (starts session via CookerState_On). PowerOff() does NOT trigger callback.
- **CHK PowerOff() DOES trigger StatusCallback** (unlike Highway). New PowerOffWithFan must NOT trigger callback.
- **UIPresenter state table**: `ShortPressNextStateID` is set BEFORE `ActionFunctionShortPress` runs. Use `NO_SUPPORTED_STATE` to let action function set state dynamically.
- **UIPresenter_Process()** always resets `StateTimeOutCounter` after key events, overwriting any value set by the action function.
- **CookingSession_Stop()** resets CurrentSession - must get duration BEFORE stopping.
- **IndCooker_Tick() timer** calls `IndCooker_PowerOff()` on expiry - must stop timer during shutdown to prevent killing fan.
- Pre-existing `-Wc23-extensions` warnings on variadic macros throughout codebase - not our issue.
- **`CookerShouldBeLocked()` is disabled** — hardcoded `return false` with TODO. Credit enforcement is only at UI level via `UIModel_IsUserInCredit()`.
- **Nexus `credit_set` callback sends UI feedback directly** — not via `nxp_keycode_feedback_start()`, because Nexus ignores the return value.
- **Token type is SET (absolute), not ADD** — credit = seconds since production date, not seconds to add.

## File Locations
- PowerBoard enum/callbacks: `ExternalPeripherals/PowerBoard/Inc/PowerBoard.h`
- Highway control bytes: `ExternalPeripherals/PowerBoard/Src/HighwayDriver.c` (CookerControl array)
- UI State machine: `ProductFeatures/UI/Src/UIPresenter.c`
- Session aggregation: `ExternalPeripherals/Flash/Src/SessionAggr.c`
- Token processing: `ProductFeatures/Token/Src/Token.c` (queue, Nexus init/churn)
- Token Nexus callbacks: `ProductFeatures/Token/Src/TokenKeycode.c` (credit_set, unlock, feedback)
- Token NV storage: `ProductFeatures/Token/Src/TokenNv.c` (SPI flash persistence)
- Credit decision: `ProductFeatures/CreditManagement/Src/CreditManagement.c`
- Shadow/device twin: `ProductFeatures/Shadow/Src/Shadow.c` (server JSON, due date)
- Security data (keys, prod date): `ProductFeatures/SecData/Src/SecDataStore.c` (MCU flash page 63)
- Device settings (PAYGO serial/version): `ProductFeatures/DeviceSettings/Src/DeviceSettings.c`
