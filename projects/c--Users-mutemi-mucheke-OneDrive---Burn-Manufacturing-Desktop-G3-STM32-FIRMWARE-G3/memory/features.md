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
