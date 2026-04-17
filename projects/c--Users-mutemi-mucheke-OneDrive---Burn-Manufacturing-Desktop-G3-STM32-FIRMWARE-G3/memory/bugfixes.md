# Bug Fixes

## BT, LOCK, LED_0, LED_1 light up on power-up before Led_Init runs
- **Product:** G3
- **Date:** 2026-04-13
- **Branch:** Prescan
- **Files changed:** `Core/Src/gpio.c`, `STM32_FIRMWARE_G3.ioc`, `ExternalPeripherals/Led/Src/Led.c`, `ExternalPeripherals/Led/Inc/Led.h`
- **Root cause / Purpose:** When the cooker was plugged in, the Bluetooth LED, Lock LED, and two board LEDs (LED_0 on PC6, LED_1 on PC7) came on at power-up and only turned off ~18 init steps later when `Led_Init()` ran. Expectation was all LEDs off except the PWR LED toggling in standby. Root cause was that `MX_GPIO_Init()` (CubeMX-generated) wrote these pins to the wrong initial level given each LED's polarity:
  - BT_LED (PB12, active-HIGH): written SET → ON
  - LOCK_LED (PA8, active-HIGH): written SET → ON
  - LED_0 (PC6, active-LOW): written RESET → ON
  - LED_1 (PC7, active-LOW): written RESET → ON
  - Both BT and LOCK also had `GPIO_PuPd=GPIO_PULLUP` on push-pull outputs, weakly holding them high during the init window.
- **What we tried that didn't work:** N/A — identified by tracing WritePin calls in `MX_GPIO_Init` against the polarity in `Led.c` `Led_On/Led_Off` functions.
- **Final solution:** In `Core/Src/gpio.c` `MX_GPIO_Init()`:
  - Line 61: `HAL_GPIO_WritePin(LED_CRTL_BLE_GPIO_Port, LED_CRTL_BLE_Pin, GPIO_PIN_SET)` → `GPIO_PIN_RESET`
  - Line 64: `HAL_GPIO_WritePin(LED_CTRL_LOCK_GPIO_Port, LED_CTRL_LOCK_Pin, GPIO_PIN_SET)` → `GPIO_PIN_RESET`
  - Line 67: `HAL_GPIO_WritePin(GPIOC, LED_0_Pin|LED_1_Pin, GPIO_PIN_RESET)` → `GPIO_PIN_SET` (both are active-LOW)
  - Line 89: BLE pin `GPIO_PuPd=GPIO_PULLUP` → `GPIO_NOPULL`
  - Line 102: LOCK pin `GPIO_PuPd=GPIO_PULLUP` → `GPIO_NOPULL`
  Mirrored the same changes in `STM32_FIRMWARE_G3.ioc` (PA8, PB12, PC6, PC7) so CubeMX regeneration produces the fixed code — set `PinState` and `GPIO_PuPd` on each pin to match.
  Also added `LED_0` to the `Led_e` enum in `Led.h` and added matching cases to `Led_On`/`Led_Off`/`Led_Toggle` and `Led_Init`/`Led_AllOn` in `Led.c` so the pin is controllable from firmware (was defined in GPIO but never referenced).
- **Verification:** Power-cycle the cooker. At startup only the PWR LED should toggle in standby — BT, LOCK, LED_0, and LED_1 should stay off. User confirmed on hardware.
- **Status:** Verified (LED startup behavior). The `LED_0` enum/switch additions are untested — exercise by calling `Led_On(LED_0)` / `Led_Off(LED_0)` and checking the physical PC6 LED responds.

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

## CHKDriver UpdateCurrentStatus overwrites CurrentPowerState to POWER_OFF
- **Product:** G3
- **Date:** 2026-03-30
- **Branch:** CHK-Bring-Up
- **Files changed:** `ExternalPeripherals/PowerBoard/Src/CHKDriver.c`
- **Root cause / Purpose:** `UpdateCurrentStatus()` declared `NewPowerState = COOKER_POWER_STATE_POWER_OFF` as a local variable but never parsed it from the RX data. On every received status frame, it compared `NewPowerState != CurrentPowerState` and overwrote `CurrentPowerState` back to `POWER_OFF`. This meant `CHKDriver_PowerOn()` would set `CurrentPowerState = POWER_ON`, but the very next RX frame would reset it to `POWER_OFF`, causing `SendCurrentPowerCommandToCooker()` to send power-off bytes (Command[7]=0x00, Command[10]=0x3C) instead of power-on bytes (Command[7]=0x10, Command[10]=0xE0).
- **What we tried that didn't work:** Initially thought the wake-up sequence was failing. Tested with a 30s send / 30s silent / wake cycle. The wake-up GPIO pulse worked correctly, but after waking the board only entered standby instead of resuming 200W.
- **Final solution:** Removed `NewPowerState` variable entirely from `UpdateCurrentStatus()`. `CurrentPowerState` is a commanded state (set by `PowerOn`/`PowerOff`), not reported by the power board. The status callback now only fires on pot presence changes. Added a comment explaining why `CurrentPowerState` must not be overwritten by RX data.
- **Verification:** Wake-up test cycle: 30s sending at 200W → 30s silent → wake → should resume 200W (not standby).
- **Status:** Untested
- **Key lesson for refactoring:** In the CHK driver, clearly separate *commanded state* (what we tell the power board to do) from *reported state* (what the power board tells us). `CurrentPowerState` and `CurrentPowerLevel` are commanded. Pot presence, errors, and temperatures are reported. Never let RX parsing overwrite commanded state.

## Lock LED stays on after exiting child lock in timed cooking
- **Product:** G3
- **Date:** 2026-04-17
- **Branch:** Prescan (also ported to CHK-Bring-Up as commit ff6bf10)
- **Files changed:** `ProductFeatures/UI/Src/UIView.c`
- **Root cause / Purpose:** In a timed cooking session, long-pressing the lock button enters child lock (LOCK_LED on) and pressing again exits (state goes back to `USER_COOKING_WITH_TIMER`) — but the LOCK_LED stayed on. Normal (non-timed) cooking worked fine. The LED was managed only in `EnterChildLock`/`ExitChildLock` in `UIPresenter.c`, gated by `LockToggleCooldown` (set to 2 ticks ≈ 2s on entry). Because `UIPresenter_Tick` runs every 1s and the keypad generates repeated LONG_PRESS events while the button is held, a second LONG_PRESS could arrive while cooldown was still > 0. `ExitChildLock` then returned early without calling `Led_Off(LOCK_LED)`. The asymmetry with normal cooking came from the LED being controlled purely by the presenter state transition — there was no view-level safety net.
- **What we tried that didn't work:** Traced the full state-machine flow first (UIPresenter_Process → ProcessCommandKeyPress → ExitChildLock → Led_Off). On paper the LED should turn off. The race only manifests because `LockToggleCooldown` can still be non-zero when a held-button fires a second LONG_PRESS.
- **Final solution:** Made LOCK_LED a function of the active view in `ProductFeatures/UI/Src/UIView.c`, symmetric with how PWR_LED is handled:
  - `ShowCurrentPowerSetting` (line ~163): added `Led_Off(LOCK_LED)`
  - `ShowTimedCookingWithPower` (line ~317): added `Led_On(PWR_LED)` + `Led_Off(LOCK_LED)` at function top. This view re-renders every 100 ms tick, so the LED cannot linger even if `ExitChildLock`'s `Led_Off` is skipped by cooldown.
  - `ShowChildLock` (line ~460): added `Led_On(LOCK_LED)`
  - `ShowPowerLevelWithLockIcon` (line ~474): added `Led_On(LOCK_LED)`
  `EnterChildLock`/`ExitChildLock` still drive the LED too — the view calls are a defensive second path.
- **Verification:** Start a timed cooking session. Long-press lock → LOCK_LED on + lock screen. Long-press again → LOCK_LED off + back to timed cooking screen with timer and power level. Repeat several times quickly to exercise the cooldown race. Also verify normal cooking still works (lock on/off).
- **Status:** Verified

## Colon doesn't blink in child lock during timed cooking
- **Product:** G3
- **Date:** 2026-04-17
- **Branch:** Prescan (commit 0a4fdcd), also ported to CHK-Bring-Up (commit 292c821)
- **Files changed:** `ProductFeatures/UI/Src/UIView.c`
- **Root cause / Purpose:** In a timed cooking session, after long-pressing lock the display cycles LOC (2s) → time+power+lock icon (9s) → LOC → …. In the timed cooking view (`UI_VIEW_TIMED_COOKING_WITH_POWER`) the colon blinks at 1 Hz, but in the lock view the colon sat solid. Two causes: (1) `ShowPowerLevelWithLockIcon` always called `Display_GetColonData` unconditionally, not gated on `ColonVisible`. (2) `UIView_Process` only ticked `ColonBlinkCounter` / toggled `ColonVisible` for `UI_VIEW_TIMED_COOKING_WITH_POWER` — during lock views the counter was frozen and the screen wasn't re-rendered on tick.
- **What we tried that didn't work:** N/A.
- **Final solution:** Two edits in `ProductFeatures/UI/Src/UIView.c`:
  - `ShowPowerLevelWithLockIcon` (line ~496): replaced the unconditional `Display_GetColonData(ColonBuffer)` with the same `if (ColonVisible) { Display_GetColonData(...) } else { memset(ColonBuffer, 0, ...) }` pattern already used by `ShowTimedCookingWithPower`.
  - `UIView_Process` (line ~791): added cases for `UI_VIEW_DISPLAY_CHILD_LOCK` and `UI_VIEW_DISPLAY_POWER_WITH_LOCK`. Both increment `ColonBlinkCounter` and toggle `ColonVisible` at `COLON_BLINK_TICKS`. CHILD_LOCK just ticks the counter (LOC screen has no colon to render). POWER_WITH_LOCK also calls `ShowPowerLevelWithLockIcon()` so the display actually redraws each tick. Not resetting the counter on entry/exit keeps the blink phase continuous across the lock/unlock transitions (user explicitly wanted "resume where it was").
- **Verification:** Start a timed cooking session — confirm colon blinks at 1 Hz. Long-press lock. During the time+power+lock view the colon should continue to blink at 1 Hz. Long-press again to unlock — colon phase should be continuous (no visible jump).
- **Status:** Verified

## Lock LED stays on after timer expires while locked
- **Product:** G3
- **Date:** 2026-04-17
- **Branch:** Prescan (commit bc3a8d3), also on CHK-Bring-Up (commit 8ee0e7a) — bundled with the "Lock during timer setting/acceptance" feature
- **Files changed:** `ProductFeatures/UI/Src/UIPresenter.c`
- **Root cause / Purpose:** `HandleTimerExpiry` transitioned to `SHUTDOWN_USED_LABEL` without turning off `LOCK_LED` when the cooking timer elapsed while the cooker was in `CHILD_LOCK`/`CHILD_LOCK_SHOW_POWER`. Exit from the lock normally goes through `ExitChildLock` which turns the LED off — but timer expiry bypasses that path.
- **Final solution:** In `HandleTimerExpiry`, before setting state to `SHUTDOWN_USED_LABEL`, check if current state is `CHILD_LOCK` or `CHILD_LOCK_SHOW_POWER` and call `Led_Off(LOCK_LED)`.
- **Verification:** Start a short timed cooking session, long-press lock, wait for the timer to expire. LOCK_LED should turn off as the cooker enters the shutdown "USED" sequence.
- **Status:** Verified
