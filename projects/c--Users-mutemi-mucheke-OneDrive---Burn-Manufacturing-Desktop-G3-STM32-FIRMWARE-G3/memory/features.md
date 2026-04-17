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
