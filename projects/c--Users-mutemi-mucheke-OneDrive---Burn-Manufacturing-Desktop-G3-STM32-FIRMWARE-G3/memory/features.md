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

### Known residual — cannot be fixed in firmware (2026-04-21 investigation)
**Symptom:** When mains is cut **mid-cook**, the display briefly shows all segments on then flickers off. Not present in ON / USED / USAGE / OFF / STANDBY — only cooking states.
**Why cooking-specific:** IGBT tank collapse on mains-off produces a noisy, non-monotonic 3.3V rail drop (unlike the clean monotonic decay in idle states). The VK16K33 sits on that rail.
**Why all segments:** The chip's entire internal state — segment RAM + display-enable bit + oscillator-enable bit — is SRAM. Under the noisy Vcc sag, 6T SRAM cells collapse toward 1. All-bits-1 = all segments lit.
**What we tried (all ineffective):**
- PVD at level 6 (~2.75V falling) with ISR sending `CMD_DISPLAY_SETUP | DISPLAY_OFF`. The display-enable bit is itself SRAM and re-flips to 1 during the collapse — the driver comes back ON before we're done.
- Added `CMD_SYSTEM_SETUP` (oscillator off) in the same ISR. Oscillator-enable bit is also SRAM → same failure.
**Verified independent:** LEDs do not flicker during the same event → MCU is not resetting and re-running startup. The glitch is purely chip-internal.
**Only fix is hardware:**
1. **Bulk cap (22–100 µF) on VK16K33 Vcc** → clean monotonic decay; SRAM stays stable until chip browns out cleanly.
2. **Power-gate VK16K33 Vcc through a FET** driven by a GPIO; cut via PVD ISR. Eliminates the glitch window entirely.
**Code left as-is:** PVD scaffolding was added and then reverted (commits visible in git history). Do not re-add without hardware change — it's provably ineffective alone.

### Later polish (commit 03ce001 / d684d41)
- `ShowStartupAllOn` now explicitly turns `INFO_PAY_LED` and `BT_LED` off after `Led_AllOn`. Those are excluded from the startup visual by design.

## CHK Power Board Wake-Up
- **Product:** G3
- **Date:** 2026-05-06
- **Branch:** CHK-Wakeup-Test
- **Files changed:** `ExternalPeripherals/PowerBoard/Src/CHKDriver.c`, new file `ExternalPeripherals/PowerBoard/Docs/CHK_WakeUp.md`
- **Root cause / Purpose:** New G3 hardware revision has CHK power board that enters 0.3 W sleep after ~100 ms of UART silence. Once asleep its MCU stops responding, including to subsequent commands — any TIM7 interruption silently bricks the cooker until power-cycle. Need a deliberate wake mechanism + auto-recovery watchdog.
- **What we tried that didn't work:**
  - Supplier-doc HIGH→LOW pulse on both PB10/PB11 at any width (150 ns, 5 µs, 200 ms, 20-pulse burst). All FAIL.
  - Hold PB11 LOW alone (1500 ms) without TX activity. FAIL — proves TX is required.
  - TX frames alone, no PB11 manipulation. FAIL — proves hold is required.
  - TX BEFORE the hold. FAIL — proves coincidence is required.
  - TX AFTER hold release. FAIL — proves coincidence is required.
  - Drive PB11 HIGH (instead of LOW) with TX during. FAIL — polarity matters.
  - Hold > 3000 ms with TX at start. FAIL — TX→release gap exceeds the wake-detect's ~3 s recency window.
- **Final solution:** Non-blocking 3-state machine in `CHKDriver.c` — `WakePowerBoard_Begin()` kicks off (drives PB11 LOW via GPIO, queues one control frame on PB10 in USART AF), `WakePowerBoard_Tick()` runs every `CHKDriver_Process` iteration and advances state on elapsed-time thresholds (TX_IN_FLIGHT for 30 ms, then SETTLE for 10 ms, then back to IDLE). Main loop never blocks — UI/keypad/BLE stay responsive throughout recovery. TIM7 stops for the whole 40 ms wake window (below the board's 100 ms sleep threshold). Triggered by a 500 ms RX-timeout watchdog gated on `WakeState == IDLE`. Critical impl detail: pre-clear PB11 ODR via BSRR BEFORE flipping MODER from AF to OUTPUT, otherwise the mode change produces a ~544 ns spurious LOW edge before the deliberate one. Test harness was removed post-validation; a 200+ cycle scenario stress-test characterised the operational envelope and is preserved in git history on branch `CHK-Wakeup-Test`.
- **Hardware dependency:** 1 kΩ series resistor on the powerboard's TX line (= STM32 RX = PB11) limits contention current when our GPIO push-pull drives the line LOW against the powerboard's own TX driver. Without this resistor, future hardware would risk driver damage.
- **Verification:** 38/38 PASS across 8 silence depths from 50 ms to 5 minutes. 19/19 PASS across 5 settle durations from 0 to 100 ms. Production-mode disconnect-test (manually unplug TX briefly) recovers cleanly every time.
- **Status:** Verified
- **Detailed write-up:** `ExternalPeripherals/PowerBoard/Docs/CHK_WakeUp.md` — full technical doc covering mechanism, what failed, operational envelope, and known limitations.

## Info button — realtime current-session + last-session usage screens
- **Product:** G3
- **Date:** 2026-05-14
- **Branch:** Prescan (commit 6862220) + ported to CHK-Bring-Up (commit 8d87b80)
- **Files changed:** `ProductFeatures/UI/Inc/UIPresenter.h`, `ProductFeatures/UI/Inc/UIView.h`, `ProductFeatures/UI/Src/UIPresenter.c`, `ProductFeatures/UI/Src/UIView.c`
- **Purpose:** Wire the dedicated `ENERGY_INFO_BUTTON` (X12A cap-touch bit `0x0020` on I2C addr 2, already declared in `Keypad.h` but inert in the UI) to two parallel UX flows: glance at the **last completed session's kWh** from STANDBY, and watch the **current session's kWh live** during cooking. Required so paygo users can check usage without leaving the cooking screen.

### Hardware mapping (already in place, no change)
`ENERGY_INFO_BUTTON` is declared in `Keypad.h` and mapped in `X12ACapTouch.c` at `X12A_CAP_TOUCH_KEY_ENERGY_INFO_BUTTON = 0x0020`. Prior to this feature it was wired in code but no UI state acted on it.

### State machine additions (`UIPresenter.c`)
Seven new active states + two dormant RTC-blocked pairs:
- `INFO_LAST_SESSION_LABEL` (view `UI_VIEW_DISPLAY_STRING_USED` → "USED", 3 s) → `INFO_LAST_SESSION_VALUE` (view `UI_VIEW_DISPLAY_SESSION_POWER` → last-session kWh + KWH icon, 10 s) → STANDBY. Reached from `STANDBY[ENERGY_INFO_BUTTON]` short press.
- `INFO_REALTIME` (view `UI_VIEW_DISPLAY_REALTIME_POWER`, no timeout) — entered from `USER_COOKING[ENERGY_INFO_BUTTON]`. Shows live `CookingSession.AccumulatedPower` via `CookingSession_GetCurrentSession()` + KWH icon, refreshed at 1 Hz from a new case in `UIView_Process`.
- `INFO_REALTIME_TIMED` (view `UI_VIEW_DISPLAY_REALTIME_TIMED_POWER`) — entered from `USER_COOKING_WITH_TIMER[ENERGY_INFO_BUTTON]`. Shows timer (DIGIT_1-3) + HOUR/MIN icons + blinking colon (`COLON_BLINK_TICKS = 10`) + kWh value at DIGIT_4-7 with dot + KWH icon. Re-renders every tick (100 ms) for the colon; kWh source updates ~1 Hz.
- `INFO_REALTIME_LOCKED` / `INFO_REALTIME_TIMED_LOCKED` — same views, with `LOCK_ICON` overlaid via `UIPresenter_IsRealtimeLocked()` check inside `ShowRealtimePower` (third arg to new `RenderKwhFloat` helper) and inside `ShowRealtimeTimedPower`. Both also `Led_On(LOCK_LED)` on every render so the LED stays asserted.
- `INFO_DAILY_LABEL`, `INFO_DAILY_VALUE`, `INFO_MONTHLY_LABEL`, `INFO_MONTHLY_VALUE` — **wrapped in `#if 0 … #endif`** with a `TODO(RTC)` comment. The hardware lacks a populated RTC, so `SessionAggr_UpdatePowerUsage()`'s daily/monthly rollover cannot work. To re-enable: remove the `#if 0`, change `STANDBY.ActionsOnKeyPress[ENERGY_INFO_BUTTON].ShortPressNextStateID` from `UI_PRESENTER_STATE_INFO_LAST_SESSION_LABEL` back to `UI_PRESENTER_STATE_INFO_DAILY_LABEL`, optionally remove the LAST_SESSION states.

### Key matrix on the realtime states
| State | LOCK short | LOCK long | INFO short | PWR+/-, TIMR+/- short | ON/OFF short | Auto-timeout |
|---|---|---|---|---|---|---|
| `INFO_REALTIME[_TIMED]` (unlocked) | inert | `RealtimeEngageLock` → LOCKED variant | `ManageTransitionFromPowerUsageDisplay` (toggle off → cooking) | `RealtimePwr/TimrPlus/MinusExit` (apply + transition) | `HandleShutdownSequence` | none — persists |
| `INFO_REALTIME[_TIMED]_LOCKED` | inert | `RealtimeDisengageLock` → unlocked variant | inert | inert | inert | none |

### Helper functions added (`UIPresenter.c`, static)
`RealtimePwrPlusExit`, `RealtimePwrMinusExit`, `RealtimeTimrPlusExit`, `RealtimeTimrMinusExit`, `RealtimeEngageLock`, `RealtimeDisengageLock`. The TIMR helpers call `UIModel_Increment/DecrementTimerSetting` only when `UIModel_IsCookTimerRunning()` is true (mirrors the difference between USER_COOKING and USER_COOKING_WITH_TIMER's TIMR+/- behaviour). `RealtimeEngageLock` sets `SavedPreLockState` to the *cooking* state (not the realtime state) so a later unlock returns the user to cooking, not realtime; it also honours `LockToggleCooldown` to match `EnterChildLock`'s debounce.

### Views (`UIView.c`)
- New: `ShowString1D`, `ShowString30D` (dormant, registered for the disabled daily/monthly path), `ShowRealtimePower`, `ShowRealtimeTimedPower`.
- New helper `RenderKwhFloat(float Value, bool IncludeMonthIcon, bool IncludeLockIcon)` consolidates the digit-formatting, dot overlay, KWH icon, optional MONTH icon, optional LOCK icon logic — caps at 99.99 kWh with two-decimal precision, same digit alignment as the existing `ShowSessionPower`. Called by `ShowDailyPowerUsage`, `ShowMonthlyPowerUsage`, `ShowRealtimePower`.
- `ShowRealtimeTimedPower` constructs a single 7-char `snprintf` string `"%d%02d %d%d%d"` (or `"%d%02d%d%d%d%d"` for ≥ 10 kWh) so timer occupies DIGIT_1-3 and kWh occupies DIGIT_5-7 (or 4-7), with the colon between digits 1 and 2 (gated by `ColonVisible`) and the dot at the fixed position 5↔6.
- `UIView_Process` got a new `UI_VIEW_DISPLAY_REALTIME_POWER` case that re-calls `ShowRealtimePower` every 10 ticks (1 s) via a static counter, and a new `UI_VIEW_DISPLAY_REALTIME_TIMED_POWER` case that re-renders every tick (matching the existing `TIMED_COOKING_WITH_POWER` cadence so the colon visibly blinks).

### Integration with existing helpers
- `IsStateCooking()` — added `INFO_REALTIME`, `INFO_REALTIME_TIMED`, `INFO_REALTIME_LOCKED`, `INFO_REALTIME_TIMED_LOCKED`. Keeps the 4-hour auto-off counter ticking and pauses the fan cooldown while realtime is showing mid-cook.
- `IsTimerActiveState()` — added the same four states, all guarded by `UIModel_IsCookTimerRunning()`. Ensures `HandleTimerExpiry` still fires the shutdown sequence if the cooking timer hits zero while the user is viewing realtime.
- `HandleTimerExpiry()` LOCK_LED cleanup — added the two `_LOCKED` realtime states alongside the existing `CHILD_LOCK[_SHOW_POWER]` check, so `LOCK_LED` is turned off when timer-expiry transitions to `SHUTDOWN_USED_LABEL`.
- New public getter `UIPresenter_IsRealtimeLocked()` (declared in `UIPresenter.h`) used by `ShowRealtimePower`/`ShowRealtimeTimedPower` to decide whether to overlay `LOCK_ICON` and assert `Led_On(LOCK_LED)`. UIView already includes `UIPresenter.h` for the existing `UIPresenter_IsLockPending` / `UIPresenter_IsLockPhaseLoc` calls.

### Constants added
- `STANDBY_INFO_LABEL_TIMEOUT = 2` (→ 3 s effective, fires on tick N+1). Existing `INFO_LABEL_TIMEOUT = 4` (5 s) is reserved for the `SHUTDOWN_USED_LABEL` path and left alone. Existing `INFO_VALUE_TIMEOUT = 9` (10 s) reused for `INFO_LAST_SESSION_VALUE` and the realtime states' (unused) timeout value.

### Behaviour summary
```
STANDBY ──INFO short──▶ "USED" (3s) ─auto─▶ last-session kWh (10s) ─auto─▶ STANDBY
                          │ INFO or ON/OFF → STANDBY

USER_COOKING ──INFO short──▶ INFO_REALTIME (full-screen kWh, persists)
                                │ LOCK long → INFO_REALTIME_LOCKED (kWh + LOCK_ICON)
                                │ INFO → cooking; PWR+/- → cooking + apply; ON/OFF → shutdown

USER_COOKING_WITH_TIMER ──INFO──▶ INFO_REALTIME_TIMED (timer + blinking colon + kWh)
                                  │ LOCK long → INFO_REALTIME_TIMED_LOCKED
                                  │ (other keys same as above)
```

- **Verification:** Bench-tested on Prescan against the live cooker. Confirmed: STANDBY INFO short → "USED" then kWh; cooking INFO short → live kWh updating ~1 Hz; timed cooking INFO short → timer + kWh together with blinking colon; LOCK long on realtime → LOCK_ICON appears + screen persists + all other keys inert; LOCK long again → unlocks + still on realtime; INFO toggle and any non-LOCK key exit + apply normal action from unlocked realtime. Daily/monthly states confirmed unreachable (RTC limitation), STANDBY INFO short routes to LAST_SESSION instead.
- **Status:** Verified

