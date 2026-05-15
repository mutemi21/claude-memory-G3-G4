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

## ON/OFF button inert on SHOW_ON_MESSAGE screen
- **Product:** G3
- **Date:** 2026-04-17
- **Branch:** Prescan (commit 03ce001), also on CHK-Bring-Up (commit d684d41)
- **Files changed:** `ProductFeatures/UI/Src/UIPresenter.c`
- **Root cause / Purpose:** `UI_PRESENTER_STATE_SHOW_ON_MESSAGE` had `.ActionsOnKeyPress[ON_OFF_BUTTON] = NO_ACTION_ON_KEYPRESS`. During the ~5 s "ON" splash after pressing ON from STANDBY, pressing ON/OFF again did nothing — user couldn't cancel.
- **Final solution:** Bound ON/OFF short press to `UI_PRESENTER_STATE_SHOW_OFF_MESSAGE` with action `UIModel_PowerOffPowerBoard`, mirroring the existing `SHOW_ZERO_POWER` pattern.
- **Verification:** Press ON from STANDBY → "ON" shows. Press ON/OFF within the window → cooker powers off, "OFF" shows, returns to STANDBY.
- **Status:** Verified

## Usage screen skipped when shutting down from mid-cook error
- **Product:** G3
- **Date:** 2026-04-20
- **Branch:** Prescan (commit eac762f), also on CHK-Bring-Up (commit a442ea7)
- **Files changed:** `ProductFeatures/UI/Src/UIPresenter.c`
- **Root cause / Purpose:** E0 (pot removed) mid-cooking puts the presenter in `DISPLAY_ERROR`. Pressing ON/OFF in that state used to run `HandleErrorShutdown` which did a minimal teardown and let the state table transition directly to STANDBY — the `SHUTDOWN_USED_LABEL → session-kWh → OFF` usage sequence (which ON/OFF from `USER_COOKING` would normally trigger via `HandleShutdownSequence`) was skipped, so the user never saw their session's kWh.
- **Final solution:** `HandleErrorShutdown` now inspects `SavedPreErrorState`. If that state was a cooking state, it delegates to `HandleShutdownSequence` (same flow as ON/OFF from USER_COOKING — stops session, records kWh, fan cooldown if session ≥60s, transitions to SHUTDOWN_USED_LABEL). Otherwise it keeps the old minimal teardown path. Also refactored `IsInCookingState()` into a parameterised `IsStateCooking(State)` helper so the pre-error state can be queried.
- **Verification:** Start cooking, remove pot → E0 displayed. Press ON/OFF → "USEd" shows, then session kWh, then "OFF", then STANDBY. Fan runs if session was ≥60s. Errors that fire outside a cooking state (startup/standby) still jump straight to STANDBY with no usage screen.
- **Status:** Verified

## Persistent E0 (pot removed) never auto-shuts down
- **Product:** G3
- **Date:** 2026-04-21
- **Branch:** Prescan (commit a70a5aa, timeout later reduced to 30s in bf54aec), also on CHK-Bring-Up (commits 4ec65fa + 44efd5c)
- **Files changed:** `ProductFeatures/UI/Src/UIPresenter.c`
- **Root cause / Purpose:** `DISPLAY_ERROR` state had `NextStateID = NO_SUPPORTED_STATE` and `ActionFunction = NULL` on its state timeout — with `TimeOutValue = DEFAULT_STATE_TIMEOUT` (1s) it re-armed every second and did nothing. If the user removed the pot mid-cook and walked away, E0 sat on the display indefinitely; only a manual ON/OFF press would trigger the shutdown flow.
- **Final solution:** Added a dedicated `ErrorAutoShutdownCounter` (bumped each `UIPresenter_Tick` while `ErrorActive && CurrentError == COOKER_ERROR_NO_POT_ERROR && IsStateCooking(SavedPreErrorState)`). At `ERROR_AUTO_SHUTDOWN_SECONDS = 30` it calls `HandleErrorShutdown` — which routes through `HandleShutdownSequence` for cooking pre-error states, so "USEd" → kWh → OFF → STANDBY plays (with fan cooldown if the session was ≥60s). New `CurrentError` static variable tracks the active error code. Also suppressed E0 entirely when it fires outside a cooking state — no `DISPLAY_ERROR` entry in that case. The counter lives outside `StateTimeOutCounter` so key presses during the error do not extend the 60s window.
- **Verification:** Start cooking, remove pot, do nothing → after 30s cooker auto-shuts down and displays usage. Press ON/OFF before 30s → immediate shutdown (existing behavior). Trigger E0 from STANDBY → no error screen. Re-insert pot before 30s → error clears, counter resets, cooker resumes.
- **Status:** Verified

## PWR+/- disabled during TIME_SETTING_SCREEN and TIMER_ACCEPTANCE
- **Product:** G3
- **Date:** 2026-04-21
- **Branch:** Prescan (commit 5f37234), also on CHK-Bring-Up (commit f78a528)
- **Files changed:** `ProductFeatures/UI/Src/UIPresenter.c`
- **Root cause / Purpose:** In `UI_PRESENTER_STATE_TIME_SETTING_SCREEN` and `UI_PRESENTER_STATE_TIMER_ACCEPTANCE` the `PWR_PLUS_BUTTON` and `PWR_MINUS_BUTTON` entries were `NO_ACTION_ON_KEYPRESS`. So once the user pressed `TIMR+` to begin setting a timer, the power level was frozen until the timer actually started counting down (through the 2s idle + ~5s acceptance flash window).
- **Final solution:** Bound both buttons in both states to `UIModel_IncrementPowerSetting` / `UIModel_DecrementPowerSetting` (same as `USER_COOKING`). `NextStateID = NO_SUPPORTED_STATE` so the state stays; pressing PWR+/- during `TIME_SETTING_SCREEN` also resets the state's 2s timeout naturally (via `UIPresenter_Process`), which matches the user's intent of still adjusting.
- **Verification:** Press `TIMR+` to begin setting a timer. While the static time-setting screen is up, press PWR+ → power level bars increment. Same for PWR-. Wait for acceptance flash to begin, press PWR+ during flash → power level bars update (visible during the flash's on-half). Timer still starts correctly at the end of the flash.
- **Status:** Verified

## All buttons beep in STANDBY
- **Product:** G3
- **Date:** 2026-04-21
- **Branch:** Prescan (commit 743ec31), also on CHK-Bring-Up (commit 46c0503)
- **Files changed:** `ProductFeatures/UI/Src/UIView.c`, `ProductFeatures/UI/Src/UIPresenter.c`
- **Root cause / Purpose:** `UIView_UpdateWithKeyPadEvent` fired `PowerBoard_SingleBeep()` unconditionally on every keypad event. In STANDBY all non-ON/OFF keys are `NO_ACTION_ON_KEYPRESS` yet still produced a beep. Gave the impression buttons were doing something when they weren't.
- **Final solution:** Moved the beep into `ProcessCommandKeyPress` and gated it on the pressed key having a mapped `NextStateID != NO_SUPPORTED_STATE` or non-NULL action function for the current `(state, duration)`. Generalises the STANDBY fix: any state now beeps only for its mapped bindings (e.g. BT button no longer beeps in USER_COOKING, TIMR+/- silent on SHOW_OFF_MESSAGE, etc.). Beep also sits after the `PendingLockActivation` guard so non-LOCK keys are silent during the queue-lock window.
- **Verification:** STANDBY: only ON/OFF beeps. USER_COOKING: PWR+/-, TIMR+/-, LOCK, ON/OFF beep; BT doesn't. Queue-lock window (TIME_SETTING_SCREEN with pending lock): only LOCK beeps.
- **Status:** Verified

## CHK power board goes silent mid-cook after rapid +/- presses (beep/TIM7 UART race)
- **Product:** G3 (CHK power board only)
- **Date:** 2026-04-23
- **Branch:** CHK-Bring-Up (commit 4a37f73, plus prerequisite 30 ms timer fix and init-settle gate on same commit)
- **Files changed:** `Core/Src/tim.c`, `ExternalPeripherals/PowerBoard/Src/CHKDriver.c`, `ExternalPeripherals/PowerBoard/Inc/CHKDriver.h`
- **Root cause / Purpose:** Field failure: after rapid power +/- presses during cooking (any gear, worse near 1800 W), the CHK power board stopped responding on the A-channel UART. B-channel (STM32 TX) kept running. Only a mains power cycle recovered it. Eight rounds of forward bisection (stripping production subsystems one at a time) all failed to find a single culprit; the failure reproduced in every build that had the full production stack on top of the minimal firmware. Inverse bisection from a minimal test-runner (19-pattern Python-suite replica driving only USART3 + TIM7) was conclusive: the test runner at Python's exact cadence with real heating PASSED, but when the same runner also fired `CHKDriver_SingleBeep()` before each level change (mimicking the UI's per-keypress beep path), the failure reproduced identically to the field.
  - Mechanism: `CHKDriver_SingleBeep()`, `CHKDriver_BuzzerOn()`, `CHKDriver_BuzzerOff()`, `CHKDriver_PowerOn()` called `SendCurrentPowerCommandToCooker()` synchronously from main-loop context. That function builds a 13-byte CHK frame on the stack and enqueues it byte-by-byte via `PowerBoardUart_PutByte()`. `PutByte` holds `__disable_irq()` only around the `Fifo3.tct++` increment — not around the whole frame. TIM7's 30 ms ISR calls the same function on its periodic cadence. Two things went wrong concurrently:
    1. **Race**: When TIM7 ISR preempted main-loop mid-frame, the two frames' bytes interleaved in Fifo3 and a mixed frame hit the wire.
    2. **Overflow**: At rapid-press cadence (~30 ms between beeps from UI), total enqueue rate reached ~866 B/s vs UART3's 480 B/s capacity. Fifo3 (32 bytes) destructively overflowed every cycle — indices zeroed mid-transmit, rest of in-progress frame becomes garbage.
  - Corrupt-frame count in a failing full-suite run: ~2850 lines with invalid FireGear/tail bytes vs ~3 in a clean run. Enough bad frames put the power board into a silent safe-state from which only a mains cycle recovers.
- **What we tried that didn't work:** (a) Python bench test suite at 4800 baud with exact replay of field patterns — 29/29 passed even with a pot heating, because USB-TTL is the only sender (no race). (b) Forward bisection tests #3–#8 removing flash writes, I²C1/display writes, LED GPIO writes, individual tick callbacks, UserSystick — each still failed because the beep-path race was still intact in each build. (c) Focused 3× staircase + 5× beep-staircase script on a fresh board — passed, because it didn't accumulate enough beep events to corrupt frames (43 events vs the full suite's ~6 600). (d) An earlier mitigation — `LEVEL_CHANGE_HOLD_MS = 150` throttle on FireGear changes — appeared to help because it slowed main-loop enqueue pressure, but didn't remove the race; it was a band-aid and has been removed.
- **Final solution:** Make TIM7 ISR the **only** writer to Fifo3. In `CHKDriver.c`:
  - `CHKDriver_PowerOn()` — set `CurrentPowerState = COOKER_POWER_STATE_POWER_ON` only; removed the sync `SendCurrentPowerCommandToCooker()` call.
  - `CHKDriver_BuzzerOn()` — set `CurrentBuzzerDuration = BUZZER_800MS` only.
  - `CHKDriver_BuzzerOff()` — set `CurrentBuzzerDuration = BUZZER_OFF` only.
  - `CHKDriver_SingleBeep()` — set `CurrentBuzzerDuration = BUZZER_100MS` only.
  TIM7 ISR (30 ms) picks up the new state variables on its next tick. Buzzer latency changes from 0 ms to ≤ 30 ms — imperceptible.
  Also in this commit: TIM7 period set to 30 ms (`htim7.Init.Period = 1499`, was 999 which gave ~20 ms — too fast for a 13-byte frame @ 4800 baud which needs 27 ms + ~3 ms gap). And an init-settle gate in the TIM7 ISR skips control commands for `INIT_SETTLE_TIME_MS = 100 ms` after init frame start, so the first TIM7 tick doesn't enqueue a control frame while the 18-byte init frame is still being drained out of Fifo3 (would send back-to-back with no inter-frame gap, which the board rejects).
  Dropped mitigations from earlier troubleshooting that were symptoms-only (not committed): `LEVEL_CHANGE_HOLD_MS` throttle, `CommittedPowerLevel`/`LastLevelChangeTick` variables, `ThrottleBypass` test hook and `CHKDriver_SetThrottleBypass()` setter.
- **Verification:** Same abuse script that reliably failed (full 19-pattern suite with `SingleBeep()` before each level change — ~380 s runtime) now runs clean for ≥ 3 minutes. B-channel responses continuous, max gap 187 ms (normal cadence), zero silence events, corrupt-frame decode count drops from ~2850 to ~1, actual power reaches 2050 W on the B-channel status. Field validation: rapid +/- presses during production cooking should no longer put the board silent.
- **Reproduction recipe for future Claude instances:** Dual-sniffer on USART3 (PB10 STM32→board, PB11 board→STM32). Run the full production build with a pot on the cooker. Go through credit entry, start cooking, mash +/- rapidly through power levels up to 1800 W. A-channel goes silent within ~380 s. Without the fix, every run ends identically (silence ~100 ms after hitting 1800 W, STM32 keeps TX-ing fine). With the fix, same abuse never triggers silence.
- **Status:** Verified on bench. Field validation pending.

## CHK usage stays at 0.01 kWh + no E0 on pot removal (RX frame parsing broken)
- **Product:** G3 (CHK power board only)
- **Date:** 2026-04-24
- **Branch:** CHK-Bring-Up (commit f0a95c6)
- **Files changed:** `ExternalPeripherals/PowerBoard/Src/CHKDriver.c`
- **Root cause / Purpose:** Two stacked bugs, both caused by CHKDriver never successfully parsing RX status frames from the power board.
  1. **Preamble mismatch**: the CHK board's status responses (board → MCU) do NOT include the 0xAA preamble byte. They start directly with `0x55 0x22 0x10 <ihstatus> ...` — 15 bytes total, no 0xFF EOF, and a non-standard trailing byte (often 0x00 or 0x01). The firmware's `ParseData` required `0xAA` as byte 0 and rejected every first-byte otherwise, so the buffer was almost never primed. Rare exceptions occurred when a data byte in the previous frame happened to equal 0xAA (probability ~1/256), after which the following `55 22 10 ...` was accepted — so usage accumulation appeared to work *very* slowly (~0.01 kWh per minute instead of the expected ~0.03+).
  2. **Length-byte mismatch**: the board's length byte reads `0x22` (= 34) in these short frames, which per the nominal formula (`4 + MessageLength`) means a 38-byte frame. `ParseData` waited for 38 bytes, which in practice spanned ~2.5 real frames; the EOF-at-position-37 check then failed on mid-stream garbage and the buffer was discarded. Result: `UpdateCurrentStatus` never ran, so `DetectPotPresence` never reported ABSENT, `StatusCallback` never fired the session-accumulator, and `ErrorCallback` never produced `COOKER_ERROR_NO_POT_ERROR`. Usage stuck at ~0.01 and E0 never displayed on pot removal.
  3. **Error-code nibble has no no-pot encoding**: CHK signals "no pot" via bit 4 of the ihstatus byte (the pot-presence bit), not via the low-nibble error code. So even with RX parsing working, `DetectError` (which only maps nibble values 0x02/0x04/0x06/0x07) would return `COOKER_ERROR_NO_ERROR` when the pot was removed. The UI's E0 display is gated on `CookerError_e == COOKER_ERROR_NO_POT_ERROR`, so pot removal was silent at the UI layer.
- **What we tried that didn't work:** (a) Unqating `StatusCallback` to fire on every parse — had no effect because `UpdateCurrentStatus` was never reached. (b) Adding `COOKER_ERROR_NO_POT_ERROR` synthesis in `UpdateCurrentStatus` — had no effect for the same reason. (c) Diagnostic edge-triggered beep inside the synthesis branch never fired, which led to the frame-parsing investigation.
- **How we found it:** cross-referencing raw `A RX` bytes in the dual-sniffer capture (`c:/Users/mutemi.mucheke/OneDrive - Burn Manufacturing/Desktop/G4 LAB Testing/CHK Test/chk_dual_20260424_*.txt`) against the Python sniffer's decoder in `c:/tmp/chk_test.py`. The Python decoder has a special case at line ~355 that synthesizes the missing `0xAA` preamble when it sees `0x55 {0x09|0x22|0x16|0x0D} 0x10` — confirming the board omits the preamble. Real CHK status frame byte layout (with virtual AA):
  - `[0] aa  [1] 55  [2] 22  [3] 10  [4] ihstatus  [5] voltage  [6] current  [7] igbt_temp  [8] surface_temp  [9..10] padding  [11] actual_power ÷ 25  [12..14] trailer`
  - Bit 4 of `[4]` clear → pot present; set → pot absent.
- **Final solution:** in `CHKDriver.c`:
  1. `ParseData` first-byte logic: accept either `0xAA` (normal) or `0x55` (CHK status peculiarity). When the first byte is `0x55`, synthesize the missing `0xAA` into `DataBuffer[0]` and jump `DataIndex` to 2 so the rest of the parser sees a uniform AA-55 preamble.
  2. After reading the header (`DataIndex >= 4`), branch on `MessageIDByte`: for `MSG_TYPE_READ_STATUS_DATA`, wait for exactly `CHK_STATUS_FRAME_LEN = 15` bytes, call `UpdateCurrentStatus`, and skip EOF + checksum validation (the board doesn't use them for status responses). Other frame types keep the original length-byte + EOF + checksum path.
  3. In `UpdateCurrentStatus`: after `DetectError`, synthesize `NewError = COOKER_ERROR_NO_POT_ERROR` whenever `NewPotPresence == ABSENT && CurrentPowerState == POWER_ON`. Needed because the board's error nibble doesn't carry a no-pot code.
  4. Side finding: `BUZZER_100MS` / `BUZZER_800MS` enum labels are inverted vs actual wire behaviour (`0x01` → long tone, `0x03` → short). `CHKDriver_SingleBeep` was changed to use `BUZZER_800MS` to get the short click the UX wants; labels kept but documented as inverted.
- **Verification:** usage accumulated at expected rate on first cook (~0.01 → 0.06 after 2 min at PL5). E0 displays within one status cycle (~150 ms) on pot lift, clears when pot replaced. User confirmed: "The usage has updated" and after the frame-parse fix "It works."
- **Reproduction recipe for future Claude instances:** dual-sniffer capture on USART3 while cooking with a pot. If raw A-RX bytes show frames starting with `55 22 10 XX ...` (no 0xAA preceding), and the firmware's usage is stuck at ~0.01 kWh, this bug is live. Check `DataBuffer[4]` value in a debugger at `UpdateCurrentStatus` entry — if never seen at all, the preamble/length mismatch is the cause.
- **Status:** Verified on bench — CHK cooker now reports usage correctly and shows E0 on pot removal.

## CHK/Highway shutdown-usage screen shows only the last pot-lift segment, not the full cook
- **Product:** G3 (both CHK and Highway — lives in shared `IndCooker` / `CookingSession` code)
- **Date:** 2026-04-24
- **Branch:** CHK-Bring-Up (commit 43e0bbf), cherry-picked to Prescan (commit 083b57d)
- **Files changed:** `ExternalPeripherals/InductionCooker/Inc/CookingSession.h`, `ExternalPeripherals/InductionCooker/Src/CookingSession.c`, `ExternalPeripherals/InductionCooker/Src/IndCooker.c`
- **Root cause / Purpose:** `IndCooker.PowerBoardStatusCb` treated every pot-absent event as a session end. Each pot lift while cooking → `CookerState_Off` → `CookingSession_Stop` → `SessionAggr_UpdatePowerUsage(segment_kWh)`, which both added the segment to the daily total AND **overwrote `LatestSessionData.CookerPower = segment_kWh`** ([SessionAggr.c:113](ExternalPeripherals/Flash/Src/SessionAggr.c#L113)). Returning the pot started a fresh session (`AccumulatedPower = 0`). If the user lifted → returned → lifted again → let E0 auto-shutdown fire, the shutdown usage screen (which reads `LatestSessionData.CookerPower` via [SessionAggr_GetLatestSessionPowerUsage](ExternalPeripherals/Flash/Src/SessionAggr.c#L144-L146), rendered by [ShowSessionPower](ProductFeatures/UI/Src/UIView.c#L416)) displayed only the last tiny segment (~0.01 kWh), not the full cook. Daily total stayed correct; only the per-session readout was wrong.
- **What we tried that didn't work:** N/A — root cause identified by tracing SessionAggr_UpdatePowerUsage call sites and the Latest session overwrite.
- **Final solution:** treat pot-absent as a PAUSE, not a session end.
  1. `IndCooker.PowerBoardStatusCb`: only call `CookerState_Off` on power-off or voltage-fault. On `PotPresence == ABSENT` while `PowerState == POWER_ON`, skip both `On` and `Off` calls — session stays active but accumulation is skipped for that frame. Track pause state with a static `SessionPaused` bool.
  2. On pot-absent → pot-present transition: call new `CookingSession_ResumeFromPause(now)` which sets `CurrentSession.End = now`. Without this, the next `UpdateSessionPower` would attribute the entire paused window as cooking energy (`watts × timeDiff` with large `timeDiff`).
  3. Session now only terminates via the UI's `IndCooker_StopSession` paths (user OFF, E0 auto-shutdown, power-off button) — at which point `LatestSessionData.CookerPower` reflects the full `AccumulatedPower`.
- **Verification:** user tested lift → return → lift → E0-30s-timeout sequence; shutdown usage screen now shows the full cook kWh instead of the last segment. User confirmed: "The bug is resolved."
- **Status:** Verified on bench (CHK). Highway path unchanged architecturally — same code drives both. Highway validation still TODO.

## CHK status frames missing AA preamble + truncated — wrong RETURNED_DATA_LENGTH and TX cadence
- **Product:** G3 (CHK power board only)
- **Date:** 2026-04-29
- **Branch:** CHK-Bring-Up (commit 104ac14)
- **Files changed:** `Core/Src/tim.c`, `ExternalPeripherals/PowerBoard/Src/CHKDriver.c`
- **Root cause / Purpose:** Cross-referenced the supplier's PC tool serial logs (`C:\Users\mutemi.mucheke\Downloads\IDCSerialLogs\IDCSerialLogs\*.txt`) and the CHK V2 protocol doc (`C:\Users\mutemi.mucheke\Downloads\UART_Protocol_CHK_V2 - EngVer 2.docx`) against our wire traffic. Two settings on the firmware side were forcing the board into a truncated/non-spec response format. The 26-byte frames the PC sees come back as ~14–15 byte mangled fragments to us. Two contributing factors:
  1. **Control TX byte 5 (`RETURNED_DATA_LENGTH`)**: firmware sent `0x20` (= 32 data bytes); PC tool sends `0x14` (= 20). At `0x20` the board returns a frame that's missing the leading `0xAA`, the trailing `0xFF` EOF, and the checksum byte. At `0x14` the board returns a clean 26-byte frame (length byte `0x16`) that includes all three.
  2. **TIM7 period (TX cadence)**: firmware sent control frames every 30 ms; PC tool runs at ~110 ms. A 26-byte response at 4800 baud takes ~54 ms, so a new 30 ms TX would arrive while the board was still mid-reply, forcing it to abort. The earlier "the board doesn't include AA" assumption (and the parser hack to accept `0x55`-without-AA + treat status frames as fixed 15-byte with no validation, commit f0a95c6) was a workaround for this self-inflicted condition.
- **What we tried that didn't work:** First fix attempt (commit f0a95c6) accepted `0x55` without `AA` and skipped EOF + checksum validation for status frames. Worked, but only because we were processing fragmented responses — back-half data (PowerStatus byte, version, ADDxL low-nibble for 12-bit temps, etc.) was simply absent.
- **How we found it:** user opened a PC-tool serial log (`1800W_SerialLog_2026-02-06_1414-37-33.txt`) and noticed every RX line had a clean `AA 55 16 10 20 ...` preamble. Comparing TX byte 5 between PC and firmware, and timing between consecutive TXs, exposed both differences.
- **Final solution:** four changes:
  1. `CHKDriver.c`: `#define RETURNED_DATA_LENGTH 0x14` (was `0x20`). Match PC tool — board now responds with proper 26-byte frames including AA + checksum + EOF.
  2. `tim.c`: TIM7 `Period = 5499` (was `1499`). Same prescaler 1279 → 110 ms cadence (was 30 ms). Matches PC tool. Note: `tim.c` is CubeMX-generated; if the .ioc gets regenerated this revert needs reapplying — TODO update the .ioc itself.
  3. `CHKDriver.c` ParseData: reverted the `0x55`-without-AA tolerance and the 15-byte status-frame override. Back to plain length-byte-driven parsing with full EOF + checksum validation.
  4. `CHKDriver.c` `IsDataValid`: fixed off-by-one. Was `CalculateChecksum(Data, Size-1)`; now `CalculateChecksum(Data, Size)`. The protocol-defined checksum covers all bytes from index 0 up to (but not including) the checksum byte at index `Size`, so the sum needs `Size` iterations, not `Size-1`. Old code happened to work in some cases because the byte just before the checksum was often `0x00` so adding it changed nothing, but for any frame where that byte was non-zero (e.g. on the new clean 26-byte frames), the old check would have rejected legitimate data.
- **Verification:** dual-sniffer log `chk_dual_20260429_154946.txt` (103 s, 942 board frames):
  - Every RX is `AA 55 16 10 20 ... <cksum> FF`, 26 bytes ✓
  - TX byte 5 = `0x14` ✓
  - TX cadence 109 ms (one timer-aliasing tick off the requested 110) ✓
  - Pot present → ABSENT → present transitions cleanly handled
  - Max actual power 2100 W reported normally
  - Zero silence events >1 s
  Functional regression on the cooker (heating, usage accumulation, pot-lift E0, E0 auto-shutdown) all still pass.
- **Reproduction recipe for future Claude instances:** if a CHK installation shows truncated/malformed RX frames in the dual-sniffer logs (no `AA`, no `FF` EOF, intermittent 14–15 byte chunks), check (a) the `RETURNED_DATA_LENGTH` value the firmware writes into TX byte 5, and (b) the TIM7 period. PC-tool reference values are `0x14` and ~110 ms respectively. Anything tighter than that (faster TX cadence) will collide with the board's reply and produce fragments.
- **Followups (out of scope for this commit):** decode the PowerStatus byte (Byte 16 of the now-complete frame) so brown-out / current-surge / voltage-surge events surface as UI errors. Add `0x03` (IGBT NTC open) and `0x05` (Surface NTC open) cases to `DetectError` — the doc lists those nibbles as "Reserved" but the supplier's fault-injection logs prove the board uses them in practice. Add UI E-code mappings in `ShowErrorStatus` for every `CookerError_e` value currently used (only E0 is wired today; wiring/IGBT-short/surface-short/surface-failure all detect but never display).
- **Status:** Verified on bench. Did NOT cherry-pick to Prescan — every change is CHK-specific (Highway uses TIM14 + its own driver, doesn't touch TIM7 or CHKDriver.c at runtime).

## TIMR+/- inert during timer-acceptance flash
- **Product:** G3
- **Date:** 2026-05-14
- **Branch:** Prescan (commit b1f620e) + ported to CHK-Bring-Up (commit 393aa95, clean cherry-pick)
- **Files changed:** `ProductFeatures/UI/Src/UIPresenter.c` (only the `UI_PRESENTER_STATE_TIMER_ACCEPTANCE` state-table entry)
- **Root cause / Purpose:** When the user finished editing a cook timer in `TIME_SETTING_SCREEN` and the 3 s idle timeout transitioned the UI to `TIMER_ACCEPTANCE` (the 5× flash before timer activation), the `TIMR_PLUS_BUTTON` and `TIMR_MINUS_BUTTON` entries were both `NO_ACTION_ON_KEYPRESS`. The flash window is ~5 s long, so any user attempt to keep adjusting the timer during that window did nothing — they had to wait for the flash to finish, watch the timer become active, then press TIMR+/- (which re-routes back to `TIME_SETTING_SCREEN` with the increment applied). Felt broken because PWR+/- already worked in this state (enabled by an earlier commit `5f37234`) but TIMR+/- did not.
- **Final solution:** wired `TIMR_PLUS_BUTTON` and `TIMR_MINUS_BUTTON` on `TIMER_ACCEPTANCE` identically to how they're wired on `USER_COOKING_WITH_TIMER`:
  ```c
  .ActionsOnKeyPress[TIMR_PLUS_BUTTON] = {
      .LongPressNextStateID = UI_PRESENTER_STATE_TIME_SETTING_SCREEN,
      .ActionFunctionLongPress = UIModel_IncrementTimerSettingFast,
      .ShortPressNextStateID = UI_PRESENTER_STATE_TIME_SETTING_SCREEN,
      .ActionFunctionShortPress = UIModel_IncrementTimerSetting
  }
  ```
  (and the mirror for `TIMR_MINUS_BUTTON` with the Decrement variants). The dispatcher applies the action first, then transitions to `TIME_SETTING_SCREEN`, and `UIPresenter_Process` resets `StateTimeOutCounter` to that state's `TimeOutValue` (= 2 → 3 s effective) after the keypad event. Result: pressing TIMR+/- during the flash cancels the flash, applies the increment/decrement, and gives the user another 3 s of editing time. If they then release, `HandleTimerSettingTimeout` runs again and re-enters `TIMER_ACCEPTANCE` — the existing tick-handler at `UIView.c` detects `AcceptancePreviousView != UI_VIEW_TIMER_ACCEPTANCE_FLASH` and restarts the 5× flash counters from scratch, so the visual is a clean fresh flash, not a continuation.
- **PendingLockActivation preservation:** the round trip through `TIME_SETTING_SCREEN` → `TIMER_ACCEPTANCE` does not touch `PendingLockActivation`, so a lock queued via LOCK long-press during `TIME_SETTING_SCREEN` still activates inside `StartTimerAfterAcceptance` when the final acceptance completes. No additional code needed for this.
- **Verification:** bench-tested on Prescan against the live cooker. Pressing TIMR+ during the flash → screen returns to TIME_SETTING_SCREEN at value+1; holding TIMR+ → +15 min jump; releasing and waiting ~3 s → fresh 5× flash, then timer becomes active normally. User confirmed.
- **Reproduction recipe for future Claude instances:** if any state in the UI presenter is missing key bindings (`NO_ACTION_ON_KEYPRESS`) for keys that should naturally interrupt that state, check whether the missing-key behaviour is intentional (a deliberate inert window) or an oversight (key was forgotten when the state was added). The `TIMER_ACCEPTANCE` PWR+/- entries had been added in commit `5f37234` for the same UX reason — TIMR+/- in this fix was the matching twin.
- **Status:** Verified

## Timer-expiry shutdown sequence skipped on INFO_REALTIME_TIMED
- **Product:** G3
- **Date:** 2026-05-14
- **Branch:** Prescan (commit fb9c6e8) + ported to CHK-Bring-Up (commit 0ef30ad, clean cherry-pick)
- **Files changed:** `ProductFeatures/UI/Src/UIPresenter.c` (only `IsTimerActiveState`)
- **Root cause / Purpose:** Reported by user: set timer 0:02, start cooking, press INFO to enter `INFO_REALTIME_TIMED`. Timer counted down 0:02 → 0:01, kWh accumulated live. When the 2 min elapsed: heater stopped correctly, **but** the timer display froze at 0:01, kWh display dropped to 0.00, and the normal SHUTDOWN_USED_LABEL → SHUTDOWN_SESSION_USAGE → STANDBY sequence never ran. UI stayed in `INFO_REALTIME_TIMED` indefinitely (would have hit 4-hour auto-off eventually).

  The bug was introduced when I added `INFO_REALTIME[_TIMED][_LOCKED]` to `IsTimerActiveState()` while wiring up the realtime feature — the addition was gated on `UIModel_IsCookTimerRunning()`:
  ```c
  if ((CurrentUIPresenterState == UI_PRESENTER_STATE_INFO_REALTIME ||
       CurrentUIPresenterState == UI_PRESENTER_STATE_INFO_REALTIME_TIMED ||
       CurrentUIPresenterState == UI_PRESENTER_STATE_INFO_REALTIME_LOCKED ||
       CurrentUIPresenterState == UI_PRESENTER_STATE_INFO_REALTIME_TIMED_LOCKED) &&
      UIModel_IsCookTimerRunning()) {
      return true;
  }
  ```
  The guard intent was to filter out non-timed realtime entries. But the moment the timer hits zero, `IndCooker_Tick()` clears `TimerRunning` and the power-board status callback runs `CookerState_Off` (which finalizes kWh to flash + zeros `CurrentSession`). All this happens *before* `HandleTimerExpiry()` next runs from `UIPresenter_Tick`. By then `UIModel_IsCookTimerRunning()` returns false → guard fails → `HandleTimerExpiry` returns early — the shutdown transition is skipped.

  Why the cooker still goes off: the side-channel `PowerBoardStatusCb → CookerState_Off → StopSession → CookingSession_Stop` path is what actually stops the heater + finalizes the session. That's why the heater goes silent but the UI is frozen. And why kWh shows 0.00: `CookingSession_GetCurrentSession()` returns -1 (no active session), so `ShowRealtimeTimedPower` falls back to `Value = 0.0f` per its existing logic.
- **Final solution:** removed the `UIModel_IsCookTimerRunning()` guard for the timed-realtime variants — they now return true unconditionally, matching the `USER_COOKING_WITH_TIMER` pattern. Non-timed `INFO_REALTIME` / `INFO_REALTIME_LOCKED` were dropped from the list entirely (they're only reachable from `USER_COOKING` where no timer can be running, so they never needed to be there). Resulting code:
  ```c
  if (CurrentUIPresenterState == UI_PRESENTER_STATE_INFO_REALTIME_TIMED ||
      CurrentUIPresenterState == UI_PRESENTER_STATE_INFO_REALTIME_TIMED_LOCKED) {
      return true;
  }
  ```
  With this, `HandleTimerExpiry`'s `UIModel_HasCookTimerExpired` check fires correctly, `CurrentUIPresenterState` transitions to `SHUTDOWN_USED_LABEL`, and the standard "USED" → final-kWh → STANDBY sequence plays. The double-stop / double-PowerOff calls inside `HandleTimerExpiry` are no-ops at this point (already done by the side channel) — safe. The "USED" / final-kWh screens read from `SessionAggr.LatestSessionData.CookerPower` which was populated by the side-channel `CookingSession_Stop`, so the user sees the correct cook total.
- **Verification:** bench-test recipe — set timer 0:01, start cooking, press INFO to enter `INFO_REALTIME_TIMED`, wait for expiry. Expect: heater stops, screen advances to "USED" (5 s), then final kWh (10 s), then STANDBY. Also confirm the locked variant: same recipe but LOCK long-press before expiry to be on `INFO_REALTIME_TIMED_LOCKED` — LOCK_LED should turn off as part of the transition (already covered by the existing LOCK_LED cleanup branch in `HandleTimerExpiry` which I had added when wiring the locked-realtime states).
- **Reproduction recipe for future Claude instances:** when adding a new UI state that should preserve timer-active behaviour, check both `IsStateCooking()` AND `IsTimerActiveState()` — but for `IsTimerActiveState` specifically, do NOT gate on `UIModel_IsCookTimerRunning()` for states reached from `USER_COOKING_WITH_TIMER`. Timer-running flag racing the expiry tick is a documented hazard. The safe pattern is "this state inherits the timer-active status of its entry context unconditionally" (same as `USER_COOKING_WITH_TIMER` itself, which doesn't self-check).
- **Status:** Verified
