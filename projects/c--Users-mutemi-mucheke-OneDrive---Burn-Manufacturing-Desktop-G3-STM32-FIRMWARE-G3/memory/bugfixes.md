# Bug Fixes

## "USED" displays as "USE" on NewShine 7-segment display
- **Product:** G3
- **Date:** 2026-03-05
- **Branch:** BugFix/Timer-Feature
- **Files changed:** `ExternalPeripherals/Display/Src/Display.c`
- **Root cause:** `CharMapAsciiCharLookup()` mapped 'D'/'d' to `CAP_D`, but the NewShine character map (`NewShineCharMap.h`) does not define `CAP_D` — only `SMALL_D`. The `UpperCharSegments[CAP_D]` lookup read zero/uninitialized data, rendering 'D' as blank.
- **What we tried that didn't work:** N/A — root cause was identified directly by comparing the enum entries in `NewShineCharMap.h` vs `CharacterMap.h`.
- **Final solution:** Changed `Display.c` line ~423: `RetVal = CAP_D` → `RetVal = SMALL_D`. Lowercase 'd' (segments B,C,D,E,G) is the standard 7-segment representation since uppercase 'D' is indistinguishable from '0'. "USED" now renders as "USEd".
- **Verification:** Power off cooker after cooking — display should show "USEd" (not "USE") during shutdown sequence.
- **Status:** Untested

## UART command flooding causes power output failure after off/on cycle
- **Product:** G3
- **Date:** 2026-03-05
- **Branch:** BugFix/Timer-Feature
- **Files changed:** `ExternalPeripherals/PowerBoard/Src/HighwayDriver.c`
- **Root cause:** `HighwayDriver_SetLevel()` sent an immediate UART command AND the periodic tick also sent commands. This dual-path flooding overwhelmed the power board, causing it to stop responding after an off/on cycle. Same bug as G4.
- **What we tried that didn't work:** N/A — known fix from G4.
- **Final solution:** Removed `SendCurrentPowerCommandToCooker(false)` from `HighwayDriver_SetLevel()`. Added TIM14 ISR (~20ms) for periodic command sends instead of the 1-second tick. Registered TIM14 callback and started timer in `HighwayDriver_Init()`. Removed send from `HighwayDriver_Tick()`.
- **Verification:** Turn cooker on, change power levels, turn off, turn on again — power levels should respond correctly.
- **Status:** Untested

## Timer bugs: info button clears timer, stuck on power level after timer elapses, no fan cooldown
- **Product:** G3
- **Date:** 2026-03-06
- **Branch:** BugFix/Timer
- **Files changed:** `ProductFeatures/UI/Src/UIModel.c`, `ProductFeatures/UI/Src/UIPresenter.c`
- **Root cause / Purpose:** Three related timer bugs all stemming from `UIModel_IsCookTimerRunning()` being a placeholder (`return false`). During timed cooking: (1) pressing info button showed usage then returned to `USER_COOKING` instead of `USER_COOKING_WITH_TIMER`, clearing the timer display; (2) when the timer elapsed, `HandleTimerExpiry()` never fired because `IsTimerActiveState()` returned false (user was in wrong state), so the UI stayed stuck showing power level even though the power board turned off; (3) fan cooldown logic in `HandleTimerExpiry()` was never reached, so the fan shut off immediately with the power board.
- **What we tried that didn't work:** N/A — root cause chain was traced directly.
- **Final solution:**
  1. `UIModel.c:167` — Changed `UIModel_IsCookTimerRunning()` from `return false` to `return IndCooker_IsTimerRunning()` so `ManageTransitionFromPowerUsageDisplay()` returns to the correct cooking state.
  2. `UIPresenter.c:372` — Disabled `ENERGY_INFO_BUTTON` in `USER_COOKING_WITH_TIMER` state (set to `NO_ACTION_ON_KEYPRESS`) to prevent the state corruption path during cooking sessions.
  3. `UIPresenter.c:471` — Added `ManageTransitionFromPowerUsageDisplay` as the timeout action for `MONTHLY_POWER_VALUE` state (was `NULL`), so it returns to cooking instead of staying forever.
- **Verification:** Set a timer, let it elapse — cooker should show shutdown sequence ("USEd" + usage) and fan should run for 60s. Info button should do nothing during cooking.
- **Status:** Untested

## Keypad not working in unified firmware — PB15 (CAPTOUCH_INT) stuck LOW permanently
- **Product:** G4 (unified firmware repo)
- **Date:** 2026-03-13
- **Branch:** unified-firmware (tested on G4 hardware)
- **Files changed:** `Core/Src/gpio.c`, `ExternalPeripherals/Keypad/Src/Keypad.c`, `ExternalPeripherals/Keypad/Src/KeypadEvents.c`, `ExternalPeripherals/InductionCooker/Src/IndCooker.c`, `ExternalPeripherals/Keypad/Src/X12ACapTouch.c`
- **Root cause / Purpose:** The X12A capacitive touch controller performs auto-calibration during its first ~2 seconds after power-up, holding INT (PB15) LOW during calibration. In the unified firmware, the CubeMX-generated `gpio.c` had BLE_RESET (PA4) and GPS_RESET (PA11) set to GPIO_PIN_SET (HIGH) at boot, immediately releasing both modules from reset. These modules starting up created electrical activity/EMI during the X12A's calibration window, causing calibration to fail and INT to stay LOW permanently. In the working G4 FW, both pins were GPIO_PIN_RESET (LOW), keeping BLE and GPS held in reset during boot, giving the X12A a quiet environment to calibrate.
- **What we tried that didn't work:**
  1. Adding initial I2C2 read in Keypad_Init to clear pending INT — PB15 stayed LOW
  2. Removing initial I2C2 reads to match G4 exactly — PB15 stayed LOW
  3. Commenting out MX_TIM6_Init() (only peripheral init difference) — PB15 stayed LOW
  4. Disabling IndCooker debug output (massive PowerBoard UART flooding) — PB15 stayed LOW
  5. Register dump confirmed GPIO/EXTI/NVIC/I2C2 all correctly configured
- **Final solution:**
  1. `gpio.c` line 56: Changed `HAL_GPIO_WritePin(BL_RESET_GPIO_Port, BL_RESET_Pin, GPIO_PIN_SET)` to `GPIO_PIN_RESET` (hold BLE in reset during boot)
  2. `gpio.c` line 72: Changed `HAL_GPIO_WritePin(GPS_RESET_GPIO_Port, GPS_RESET_Pin, GPIO_PIN_SET)` to `GPIO_PIN_RESET` (hold GPS in reset during boot)
  3. `gpio.c` line 73: Changed `HAL_GPIO_WritePin(GPS_GEO_MCU_GPIO_Port, GPS_GEO_MCU_Pin, GPIO_PIN_RESET)` to `GPIO_PIN_SET` (match G4 FW)
  4. `Keypad.c`: Replaced TIM6 polling approach with EXTI interrupt callbacks (`HAL_GPIO_EXTI_Rising_Callback` / `HAL_GPIO_EXTI_Falling_Callback` on CAPTOUCH_INT_Pin)
  5. `KeypadEvents.c`: Set `LONG_PRESS_THRESHOLD` to 40 (2 seconds at 50ms interval), added 50ms rate limiting via `HAL_GetTick()`
  6. `IndCooker.c`: Set `IND_COOKER_DEBUG_ENABLED` to 0 (was flooding UART with ~20 PowerBoard callback prints/sec)
  7. `X12ACapTouch.c`: Set `X12A_CAP_TOUCH_DEBUG_ENABLED` to 0
- **Verification:** Flash unified firmware to G4 hardware. PB15 starts LOW at init, recovers to HIGH after ~3 seconds. EXTI interrupts fire on button press. All CMD and NUM keys detected correctly.
- **Key lesson:** Cap touch controllers (X12A) are extremely sensitive to EMI during power-up calibration. When unifying repos, always match GPIO initial output levels for reset/power pins to the working reference, especially for peripherals near the cap touch sensor. The CubeMX .ioc file may generate different initial states than the reference.
- **Status:** Verified
