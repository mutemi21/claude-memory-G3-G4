# Features

## Timer UI Overhaul
- **Product:** G3
- **Date:** 2026-03-05
- **Branch:** BugFix/Timer-Feature
- **Files changed:** `ProductFeatures/UI/Src/UIPresenter.c`, `ProductFeatures/UI/Src/UIView.c`, `ProductFeatures/UI/Inc/UIView.h`, `ProductFeatures/UI/Src/UIModel.c`, `ExternalPeripherals/Display/Src/Display.c`, `ExternalPeripherals/Display/Inc/Display.h`
- **Purpose:** Complete rework of timer display behavior for better UX

### Changes Made

**New presenter states:**
- `UI_PRESENTER_STATE_TIMER_ACCEPTANCE` — 5x flash sequence after timer value confirmed

**New view enums:**
- `UI_VIEW_TIMER_ACCEPTANCE_FLASH` — 5x flash (800ms on, 200ms off), power stays static
- `UI_VIEW_TIMED_COOKING_WITH_POWER` — timer left (DIGIT_1-3), power right (DIGIT_6-7), colon blinks 1Hz

**Timer setting flow:**
- USER_COOKING → TIMR+/- → TIME_SETTING_SCREEN (static display showing time + colon + HOUR + MIN icons + power level)
- First press from USER_COOKING shows 0:00 (no increment) — ActionFunctionShortPress is NULL
- First press from USER_COOKING_WITH_TIMER increments immediately — ActionFunctionShortPress is UIModel_IncrementTimerSetting
- 2s idle timeout with value=0:00 → back to USER_COOKING (ignore)
- 2s idle timeout with value>0:00 → TIMER_ACCEPTANCE (5x flash) → USER_COOKING_WITH_TIMER
- Decrementing active timer to 0:00 → deactivates timer, returns to USER_COOKING

**Timer limits:**
- Max time: 4:00 (240 minutes), defined as `MAX_TIMER_MINUTES` in UIModel.c
- Increment past 4:00 wraps to 0:00, decrement below 0:00 wraps to 4:00
- Long press increments/decrements by 15 minutes (direct arithmetic, not loop)

**Display compositing:**
- `Display_GetPowerLevelBuffer()` added to Display.c — returns power level data in buffer without sending to I2C, enabling buffer compositing
- Timer + power displayed simultaneously by OR'ing separate buffers (timer uses DIGIT_1-3, power uses DIGIT_6-7, no overlap)

**Remaining time display:**
- `UIModel_GetRemainingCookTime()` rounds up to nearest minute: `(RemainingTime + 59) / 60` — prevents premature minute drop (e.g., 179 seconds still shows 3 minutes)

**Fan cooldown after timed cooking:**
- Timer expiry goes directly to SHUTDOWN_USED_LABEL (no blank screen from END_OF_SET_COOK_TIME)
- `HandleShutdownSequence()` preserves existing fan cooldown if `FanCooldownCounter > 0`
- `END_OF_SET_COOK_TIME` removed from `IsInCookingState()` — prevents fan cooldown counter from being zeroed

**Child lock with active timer:**
- `HandleTimerExpiry()` now also works during CHILD_LOCK/CHILD_LOCK_SHOW_POWER states when SavedPreLockState is USER_COOKING_WITH_TIMER
- `ShowPowerLevelWithLockIcon()` checks if timer values exist in model data; if so, renders timer + power + lock icon together

**Removed/replaced:**
- `ShowTimeLeftPeriodically()` — replaced by `ShowTimedCookingWithPower()`
- TIME_SETTING_SCREEN_IDLE state no longer used (direct transition to TIME_SETTING_SCREEN)
- Old periodic display logic (2s on/6s off) replaced by static timer with colon-only blink

- **Verification:** Set timer from cooking, verify 5x flash, verify timer+power display, verify child lock preserves timer, verify 0:00 handling, verify fan runs after timed session
- **Status:** Verified

## Error Handling Rewrite — Pause/Resume Cooking on Error
- **Product:** G3
- **Date:** 2026-03-06
- **Branch:** BugFix/Timer
- **Files changed:** `ProductFeatures/UI/Src/UIPresenter.c`, `ProductFeatures/UI/Src/UIModel.c`, `ProductFeatures/UI/Inc/UIModel.h`, `ExternalPeripherals/InductionCooker/Src/IndCooker.c`, `ExternalPeripherals/InductionCooker/Inc/IndCooker.h`, `ProductFeatures/UI/Src/UIView.c`
- **Purpose:** Rewrite error handling so cooking pauses on error and resumes when error clears

### Error Handling Logic
1. **Error occurs** → save pre-error state, power level, and timer status → pause timer if running → show error on display → set `ErrorActive` flag
2. **Error persists** → error display persists, auto-off counter paused
3. **Error clears** → restore power level, resume timer if it was running, return to saved state
4. **ON button during error** → go directly to STANDBY (no shutdown sequence), stop timer, reset everything
5. **Only ON/OFF button works during error state** — all other buttons ignored

### New Functions Added
- `IndCooker_PauseTimer()` — stops timer countdown without resetting the value
- `IndCooker_ResumeTimer()` — resumes timer if remaining time > 0
- `UIModel_RestorePowerLevel(CookerPower_e)` — restores power setting and sends to power board

### New Static Variables in UIPresenter.c
- `ErrorActive` — tracks whether an error is currently active
- `SavedPreErrorState` — UI state before error occurred
- `SavedPreErrorPowerLevel` — power level before error occurred
- `TimerWasRunningBeforeError` — whether timer was counting down

### States That Resume Cooking After Error
- USER_COOKING, USER_COOKING_WITH_TIMER, TIME_SETTING_SCREEN, CHILD_LOCK, CHILD_LOCK_SHOW_POWER
- All other states → go to STANDBY on error clear

### Error Display Position
- Error codes (E0-E11, EC) displayed on digits 5-6 of the 7-segment display (right side)
- Single-digit errors: 4 leading spaces ("    E1")
- Two-digit errors: 3 leading spaces ("   E10")

- **Verification:** Trigger errors during cooking — should pause and resume. Press ON during error — should go to standby. Check error codes appear on digits 5-6.
- **Status:** Untested

## Lock during timer setting/acceptance (queue-lock flow)
- **Product:** G3
- **Date:** 2026-04-17
- **Branch:** Prescan (commit bc3a8d3), also on CHK-Bring-Up (commit 8ee0e7a)
- **Files changed:** `ProductFeatures/UI/Inc/UIPresenter.h`, `ProductFeatures/UI/Src/UIPresenter.c`, `ProductFeatures/UI/Src/UIView.c`
- **Purpose:** Previously, long-pressing LOCK during `TIME_SETTING_SCREEN` or `TIMER_ACCEPTANCE` called `EnterChildLock` which jumped the state machine to `CHILD_LOCK` and aborted the timer flow — the timer never got accepted and never counted down. Users had to exit lock and re-adjust. New behavior: lock engages immediately (LED, icon, LOC on the right), timer flow continues in parallel (flash plays, timer starts counting down), and on flash completion the cooker is already in `CHILD_LOCK_SHOW_POWER`.

### Mechanism
- New `PendingLockActivation` flag + `QueueLockActivation` handler bound to LOCK long-press in `TIME_SETTING_SCREEN` and `TIMER_ACCEPTANCE`.
- While flag is set, `ProcessCommandKeyPress` early-returns on non-LOCK keys (in those two states only) — disables `TIMR+/−`, `PWR+/−`, `ON/OFF` so the user can't interrupt the queued flow.
- `StartTimerAfterAcceptance` checks the flag: if set, clears it, sets `SavedPreLockState = USER_COOKING_WITH_TIMER`, transitions to `CHILD_LOCK_SHOW_POWER`. Otherwise transitions to `USER_COOKING_WITH_TIMER` as before.
- `UIPresenter_IsLockPending()` and `UIPresenter_IsLockPhaseLoc()` getters exported for view layer.
- `UIView_UpdateCurrentView` tracks `LastLockPendingState` and force-redraws when it flips — gives immediate visual feedback on lock press without waiting for a view transition.
- `RenderTimeSettingScreen` and `ShowTimerAcceptanceFlash` render lock icon + LOC/power in the right section when `IsLockPending()` is true.

### LOC/power cycling driven from lock-press instant
- `LockDisplayCycleCounter` (in `UIPresenter.c`) ticks every second while `PendingLockActivation` or state is `CHILD_LOCK_SHOW_POWER`. Wraps at 12. Phase: `LOC` for counter 0-2, `power level` for 3-11.
- Reset to 0 in `QueueLockActivation` and `EnterChildLock` — so the 3s LOC window starts from the instant of press, not from the (later) state transition into `CHILD_LOCK_SHOW_POWER`.
- Consolidated lock destination: `EnterChildLock` and the queue-lock path both land directly in `CHILD_LOCK_SHOW_POWER` (the intermediate `CHILD_LOCK` state is now unreachable but left in the state table for safety). `CHILD_LOCK_SHOW_POWER` no longer cycles back — the view `ShowPowerLevelWithLockIcon` checks `IsLockPhaseLoc()` itself to decide between LOC and power bars.
- Timer section (time + colon + hour/min icons + lock icon) persists throughout locked timed cooking; only the power section alternates.

### Cleanup paths for the flag
- `HandleErrorShutdown` clears `PendingLockActivation` and LOCK_LED so a shutdown from the error screen doesn't leak a pending lock into the next session.

### Verification
1. TIMR+ to `0:01`, long-press lock → instant LOCK_LED + lock icon + LOC. Flash still plays. Other buttons inert until flash finishes. After flash, state lands in locked timed cooking; LOC visible for 3s total from press, then 9s power bars, cycle repeats.
2. Long-press lock during active timed cooking → same LOC/power cycling, timer section persists, timer keeps counting down.
3. Long-press lock during non-timed cooking → LOC/power cycling (same 3s/9s), no timer section (unchanged from before).
4. Exit lock at any point → returns to SavedPreLockState, LOCK_LED off.
- **Status:** Verified

## Startup all-segments/all-LEDs-on state (doubles as residual-display mitigation)
- **Product:** G3
- **Date:** 2026-04-17
- **Branch:** Prescan (commit 53c86f6), also on CHK-Bring-Up (commit 1712796)
- **Files changed:** `Core/Src/main.c`, `ExternalPeripherals/Display/Inc/Display.h`, `ExternalPeripherals/Display/Src/Display.c`, `ExternalPeripherals/Led/Inc/Led.h`, `ExternalPeripherals/Led/Src/Led.c`, `ProductFeatures/UI/Inc/UIView.h`, `ProductFeatures/UI/Src/UIPresenter.c`, `ProductFeatures/UI/Src/UIView.c`
- **Purpose:** On mains plug-in the cooker now holds every segment and every LED on for 2 seconds before proceeding to STANDBY. All UI buttons are inactive during this window. Doubles as the fix for the residual-display bug: on re-plug the VK16K33 had been powered through the dip by bulk caps and retained its RAM ("USEd", power level, etc.) — that retained image used to flash briefly before the startup state kicked in.

### Key insight (after iteration)
The residual flash has a hard software floor of a few ms (HAL_Init + SystemClock_Config + I²C setup is unavoidable). Instead of trying to hide it, **match** it: the first I²C write after `MX_I2C1_Init` now paints the VK16K33 RAM with all-0xFF. Whatever the display was retaining gets instantly replaced with the same all-segments-on image that the startup state is about to render — so the transition is invisible. For cold boot (POR default = display OFF), the paint goes into a dark chip and becomes visible only once `Display_Init` enables output, so no transient there either.

### Mechanism
- New `UI_VIEW_STARTUP_ALL_ON` view + `ShowStartupAllOn` which calls `Led_AllOn()` and writes all 0xFF to the display.
- `UI_PRESENTER_STATE_START_UP` now uses this view, `TimeOutValue = 2`, and on timeout runs the new `EndStartupSequence` action which calls `Led_AllOff()` then `CheckCookerLoanStatus` (which transitions to STANDBY).
- All key bindings on START_UP are `NO_ACTION_ON_KEYPRESS` — buttons are inert.
- `UIPresenter_Init` now starts in `UI_PRESENTER_STATE_START_UP` (was STANDBY). `UIView.c` static `CurrentUIView` defaults to `UI_VIEW_STARTUP_ALL_ON` so the very first `UIView_RunScreen` inside `UIView_Init` already paints all-on.
- `Led_AllOff()` added as mirror of `Led_AllOn()`.
- `Display_PaintAllSegmentsOn()` (replaces the earlier `Display_OutputsOff`) writes the 17-byte all-0xFF buffer via I²C. Called from `main.c` right after `MX_I2C1_Init()` — this is what masks the retained image.
- The existing 300 ms startup beep (in the `UI_VIEW_BLANK_SCREEN` case of `UIView_Process`) is preserved via a shared `case UI_VIEW_STARTUP_ALL_ON:` fall-through.

### Verification
- Unplug during "USEd" screen, wait several seconds, plug back in → no visible "USEd" flash. Display goes straight to all-on for 2 s, then normal STANDBY with blinking PWR LED.
- Cold boot (fully drained) → display starts all-on, no random segments.
- Buttons pressed during the 2 s window do nothing.
- Startup beep still plays once on boot.
- **Status:** Verified

### Known residual (not addressed)
- "Segments flicker on mains unplug" — remains. That's a different root cause (VK16K33 driver instability as Vcc sags during mains-off). Fixes: bulk cap on display rail, or PVD-driven pre-brown-out display shutdown in firmware. Left for hardware revision.

### Later polish (commit 03ce001 / d684d41)
- `ShowStartupAllOn` now explicitly turns `INFO_PAY_LED` and `BT_LED` off after `Led_AllOn`. Those are excluded from the startup visual by design.
