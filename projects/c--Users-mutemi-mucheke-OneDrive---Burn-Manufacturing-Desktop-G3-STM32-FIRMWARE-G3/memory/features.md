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

## Energy (kWh) accounting — end-to-end reference
- **Product:** G3 (Highway and CHK; same accumulator code runs for both)
- **Date:** 2026-06-04
- **Branch:** Prescan (production single-burner). Section 11 covers an *experimental* 4-burner lockstep variant on `G3-HWY-4-Burner` (commit b47a089) — not production.
- **Files involved:** `ExternalPeripherals/InductionCooker/Src/CookingSession.c`, `ExternalPeripherals/InductionCooker/Src/CookerState.c`, `ExternalPeripherals/InductionCooker/Src/IndCooker.c`, `ExternalPeripherals/Flash/Src/SessionAggr.c`, `ProductFeatures/UI/Src/UIView.c`. (`GlobalDefs/Inc/BurnerConfig.h` is experimental-branch-only.)
- **Purpose:** Reference doc for how every kWh value in the firmware is computed, persisted, and rendered. Use this to trace a displayed kWh number back to its source variable, or to decide where to add a new feature that touches energy data. Captures the integrator + state pattern + display paths + 4-burner lockstep scaling in one place.

### 1. The single source of truth

All energy in the firmware lives in one place: the static `CookingSession_t CurrentSession` in `CookingSession.c`, specifically the field:

```c
uint32_t AccumulatedPower;   // watt-seconds (W·s)
```

`uint32_t` gives a headroom of ~1193 kWh — fine for any plausible session. Conversion to kWh happens at read time via `WattsToKWh()` which divides by 3600 (s/h) and 1000 (W/kW).

### 2. The integrator

`UpdateSessionPower()` in `CookingSession.c` is the only function that writes to `AccumulatedPower` (other than zero-init in `CookingSession_Start` / `Init`). The math:

```c
int32_t timeDiff = UserRtc_CalcTimeDiffInSeconds(&CurrentSession.End, CurrentTime);
if (timeDiff < 1) return;            // (1) coalesce sub-second calls
uint32_t watts = CookerModel_PowerToWatt(&CurrentSession.CurrentUserPowerSetting);
                                     // (or .ActualPower if USE_POWER_BOARD_MEASURED_POWER == 1)
CurrentSession.AccumulatedPower += (watts * timeDiff);   // production: per-burner
// experimental G3-HWY-4-Burner branch only:
//   CurrentSession.AccumulatedPower += (watts * timeDiff * NUM_BURNERS_IN_LOCKSTEP);
CurrentSession.End = *CurrentTime;
```

Key properties:
- **`UserRtc_CalcTimeDiffInSeconds` has 1-second resolution** — sub-second calls return early without advancing `End`, so the integrator naturally steps at 1 Hz max even if upstream status frames arrive at 10 Hz.
- **`USE_POWER_BOARD_MEASURED_POWER` is `0` in the build.** `watts` comes from a fixed power-level → wattage table (`CookerModel_PowerToWatt`), so the accumulator is intent-based, not a true measurement. Flip the macro to use real measurements from the power board.
- **Rectangle rule integration.** Each interval contributes `watts × Δt`. If the user changes power mid-interval, `CookingSession_Update` updates `CurrentUserPowerSetting` first, so the *next* interval picks up the new wattage.
- **`NUM_BURNERS_IN_LOCKSTEP`** *(experimental — only present on the `G3-HWY-4-Burner` branch; not in production)*. When defined (in `GlobalDefs/Inc/BurnerConfig.h`) it scales the integrator contribution; the experimental branch sets it to `4`. Production builds don't include the header and the multiplier doesn't appear in the code. Mentioned here only because it lives at the single source of truth — see section 11.

### 3. Session lifecycle — the State pattern

`CookerState.c` is a thin state machine with two states (`Off`, `On`), each holding three function pointers:

| State | `StateActionOff` | `StateActionOn` | `StateActionTickTimeout` |
|---|---|---|---|
| `Off` | `NULL` | `StartSession` (transitions to On) | `NULL` |
| `On` | `StopSession` (transitions to Off) | `UpdateSession` (integrates) | `BackupSession` (writes to RTC backup) |

Dispatch is via `CookerState_Off()` / `CookerState_On()` / `CookerState_BackupTimeout()`. Each forwards to the current state's pointer. So calling `CookerState_On()` does different things depending on state:
- While Off → `StartSession` zeroes the accumulator, sets isActive=true, captures Start/End, transitions to On.
- While On → `UpdateSession` runs the integrator (one tick of the rectangle rule).

`Start` / `Update` / `Stop` / `Backup` wrappers in `CookerState.c` fetch `Now` via `UserRtc_GetRtcTime` and forward to the matching `CookingSession_*` function.

### 4. What drives the lifecycle

The state transitions all come from one upstream source: `PowerBoardStatusCb` in `IndCooker.c`. Triggered on every UART status frame from the (primary) power board (~10 Hz Highway, ~9 Hz CHK):

```c
if (!IsVoltageOk || PowerState != COOKER_POWER_STATE_POWER_ON) {
    CookerState_Off();                       // session ends via this path
} else if (PotPresence == COOKER_POT_PRESENCE_ABSENT) {
    SessionPaused = true;                    // skip frame entirely
} else {
    if (SessionPaused) {
        CookingSession_ResumeFromPause(&Now);
        SessionPaused = false;
    }
    CookerState_On(&PowerLevel, &CurrentActualPower);   // → StartSession (first time) or UpdateSession (every time after)
}
```

The `timeDiff < 1` guard inside `UpdateSessionPower` is what reconciles the ~10 Hz status-frame rate with the 1 Hz accumulator step rate.

`BackupSession` (the third state action) is wired through `CookerState_BackupTimeout()` from a slower periodic timer — its purpose is to snapshot `AccumulatedPower` to the STM32's RTC backup registers periodically for crash recovery.

### 5. Pause semantics (pot lift)

When the user lifts the pot mid-cook, `PowerBoardStatusCb` sees `PotPresence == ABSENT` and skips the entire body — no `CookerState_On()` call, no `UpdateSession`, accumulator pauses naturally. The user-facing E0 error is raised separately via `PowerBoardErrorCb` → `UIPresenter_HandleError`.

When the pot returns, `CookingSession_ResumeFromPause(&Now)` runs:

```c
void CookingSession_ResumeFromPause(const TIME_T* CurrentTime) {
    if (!CurrentSession.isActive) return;
    CurrentSession.End = *CurrentTime;    // realign End to now, discard the paused interval
}
```

Without this realignment, the next `UpdateSessionPower` would compute `timeDiff = (long pause duration)` and credit the entire pause as cooking energy. Resetting `End` to the resume timestamp throws away the paused interval cleanly.

### 6. Stop and persistence

When the cooker is powered off (user OFF press, E0 30-second auto-shutdown, voltage fault, any path that calls `CookerState_Off()`), `StopSession` → `CookingSession_Stop()` runs the canonical finalisation:

1. One last `CookingSession_Update` to flush the final interval into the accumulator.
2. `GetCurrentCookingSession()` builds a `CookerSessionData_t` with `.CookerPower = WattsToKWh(AccumulatedPower)` (the ×NUM_BURNERS_IN_LOCKSTEP scaling is baked into AccumulatedPower already).
3. If non-zero:
   - `FlashData_SessionDataBufferAppend` — circular session log on SPI flash.
   - `AppendCookingSessionToFlash` — duplicate entry + `SessionSummary_UpdateUsage` for cloud reporting.
   - `SessionAggr_UpdatePowerUsage(sessionData.CookerPower, sessionData.Time0)` — **this is the key call** for the post-session UI. Writes the kWh into `SessionAggrFlashStoreRAMCpy.SessionAggrData.LatestSessionData.CookerPower` and persists to flash. Also accumulates daily/monthly totals (RTC-gated).
4. `CurrentSession` is zeroed; `isActive = false`. The live integrator stops.
5. RTC backup is cleared.

### 7. Crash recovery via RTC backup

`BackupSession` (called periodically while On from `CookerState_BackupTimeout`) writes `{Start, End, kWh}` to the STM32's RTC backup registers via `RtcBackup_Write`. On boot, `CookingSession_Init` reads it; if `Backup.Power != 0`:
- Logs the lost session as a historical entry in `FlashData_SessionDataBufferAppend`.
- Updates `SessionAggr` via `SessionAggr_UpdatePowerUsage` so the post-cycle "last session" display shows it.
- Does **not** resume the live session. `CurrentSession.isActive = 0` after init. Cooker boots to STANDBY cleanly. The "missing" energy from the crash is preserved as the most recent session.

### 8. The five display paths — all read from the single source

| # | Where it shows up | Read path | Source variable |
|---|---|---|---|
| 1 | Realtime view (mid-cook): `INFO_REALTIME[_TIMED]` and locked variants | `ShowRealtimePower` / `ShowRealtimeTimedPower` → `CookingSession_GetCurrentSession(&Session)` → `Session.CookerPower` | live `CurrentSession.AccumulatedPower` |
| 2 | Post-cook "USED" → final-kWh shutdown sequence (`SHUTDOWN_USED_LABEL` → `SHUTDOWN_SESSION_USAGE`) | `ShowSessionPower` → `SessionAggr_GetLatestSessionPowerUsage` | `SessionAggrFlashStoreRAMCpy.SessionAggrData.LatestSessionData.CookerPower` (RAM mirror of flash, set in step 6.3) |
| 3 | STANDBY → INFO short-press "last session" screen (currently routed via `INFO_LAST_SESSION_*` because the RTC isn't populated; will become daily/monthly via the disabled `INFO_DAILY_*` / `INFO_MONTHLY_*` states once RTC arrives) | `ShowSessionPower` → `SessionAggr_GetLatestSessionPowerUsage` (same as #2) | same as #2 |
| 4 | Daily and monthly aggregates (RTC-gated, currently `#if 0`'d but in the code) | `ShowDailyPowerUsage` / `ShowMonthlyPowerUsage` → `SessionAggr_GetDailyPowerUsage` / `GetMonthlyPowerUsage` | `SessionAggrFlashStoreRAMCpy.SessionAggrData.DailyPowerUsage` / `.MonthlyPowerUsage`, accumulated by `SessionAggr_UpdatePowerUsage` per session stop |
| 5 | Cloud / MQTT reporting (`SessionSummary`, `MqttPayload`) | `SessionSummary_UpdateUsage(Session->CookerPower)` called from `AppendCookingSessionToFlash` | finalised `sessionData.CookerPower` from `GetCurrentCookingSession()` |

All five trace back to `AccumulatedPower` directly or via SessionAggr populated by it. There is exactly one writer (`UpdateSessionPower`) and every consumer reads from that writer's output, so any future scaling change (such as the experimental 4-burner multiplier in section 11) is applied once and propagated everywhere automatically — no double-multiplication, no conversion drift between paths.

### 9. Display formatting (`RenderKwhFloat` and `ShowSessionPower`)

Both renderers convert a `float` kWh to a 4-digit display string with 2-decimal precision, capping at 99.99 kWh:

```c
uint16_t Hundredths = (uint16_t)ceilf(Value * 100.0f);
if (Hundredths > 9999) Hundredths = 9999;
uint8_t Tens=(.../1000)%10, Ones=(.../100)%10, Tenths=(.../10)%10, Hundths=...%10;
char ValueStr[8];
if (Tens == 0) snprintf(ValueStr, 8, "    %d%d%d", Ones, Tenths, Hundths);    // " X.XX"
else           snprintf(ValueStr, 8, "   %d%d%d%d", Tens, Ones, Tenths, Hundths);  // "XX.XX"
```

The decimal point is NOT in the string; it's overlaid as a separate display segment via `Display_GetDotData()` (fixed position between display digits 5 and 6 on the VK16K33). With 4-burner scaling enabled and the cap at 99.99 kWh, a sustained max-power session would lock the display at "99.99" after ~12.5 hours at 8 kW — operationally unreachable.

Icons overlaid on the buffer:
- `KWH_ICON` always.
- `MONTH_ICON` only on monthly view (`RenderKwhFloat`'s `IncludeMonthIcon` parameter).
- `LOCK_ICON` overlaid on realtime views when `UIPresenter_IsRealtimeLocked()` returns true.

### 10. Refresh cadences

Three independent rates govern the visible cadence of the kWh display:

| Layer | Rate | Mechanism |
|---|---|---|
| Power-board status frames | ~10 Hz (Highway) / ~9 Hz (CHK) | UART RX, driver-driven |
| `AccumulatedPower` increments | **~1 Hz** | `timeDiff < 1` early-return in `UpdateSessionPower` |
| View re-render | varies by view | `UIView_Process` tick handler |

Per-view re-render rates:
- `INFO_REALTIME` (untimed) — 1 Hz, via a static counter in the case (10 ticks of the 100 ms UI tick).
- `INFO_REALTIME_TIMED` and `INFO_REALTIME_TIMED_LOCKED` — every tick (100 ms), so the blinking colon can toggle visibly. kWh digits stay stable between integrator updates because the underlying value only changes at 1 Hz.
- `INFO_REALTIME_TIMED_ACCEPTANCE` — every tick, driven by the 5× flash counter (`ACCEPTANCE_FLASH_ON_TICKS` = 8 ticks ON, `ACCEPTANCE_FLASH_OFF_TICKS` = 2 ticks OFF).
- Post-shutdown `SHUTDOWN_SESSION_USAGE` and `INFO_LAST_SESSION_VALUE` — render once on entry, no per-tick refresh (the underlying value doesn't change while displayed).

### 11. 4-burner lockstep details *(experimental — prototype branch only)*

> ⚠ **Experimental.** Lives on the `G3-HWY-4-Burner` prototype branch only. Not merged to Prescan, CHK-Bring-Up, or any production build. Treat as forward-looking design notes rather than stable feature reference. The production accumulator described in sections 1-10 has no multiplier — single-board accounting throughout.

`NUM_BURNERS_IN_LOCKSTEP` (in `GlobalDefs/Inc/BurnerConfig.h`, prototype branch only) is the deployment-time knob. The branch sets it to `4` to support an industrial 4-burner appliance built from four standard 2 kW Highway power boards driven from a single MCU UART. The multiplier scales every accumulator contribution by 4; all five display paths then show total appliance energy.

Topology assumption: MCU broadcasts TX to all 4 power boards; only the primary board's TX is wired back to MCU RX. The MCU has visibility into one board's state only, but trusts that all four are in lockstep because they receive the same commands.

Known risks (why this stays experimental until bench-validated):
- Silent failures on cookers 2-4 — no per-board feedback. A blown fuse / lifted pot / bad harness on any of those three is invisible to the firmware.
- Displayed kWh over-reports by 25-100 % if lockstep is violated for any reason during a session.
- Buzzer fires on all 4 boards simultaneously per UART command change.
- A primary-board UART-silence watchdog is the next refinement; without it, a stuck primary plus three invisible heating burners is a hazardous failure mode.

### Reproduction recipe for future Claude instances

- "Where is kWh accumulated?" → `CookingSession.c::UpdateSessionPower`, one write to `AccumulatedPower`.
- "Where is it converted to kWh?" → `WattsToKWh()` divides W·s by 3600 × 1000. Conversion happens at read time via `GetCurrentCookingSession()`.
- "Why does the realtime display only change once per second?" → `timeDiff < 1` guard in `UpdateSessionPower`. The integrator is bounded by the RTC's 1-second resolution.
- "Why does the post-cook kWh persist across reboots?" → `SessionAggr_UpdatePowerUsage` is called from `CookingSession_Stop` and writes to SPI flash; on boot `SessionAggr_Init` loads `LatestSessionData.CookerPower` from flash.
- "Why is the kWh resetting to 0 mid-cook?" → almost certainly the side-channel `CookerState_Off` path firing (voltage fault via `IsCookerVoltageOK` or `PowerState != POWER_ON` in `PowerBoardStatusCb`) — `CookingSession_Stop` ran and cleared `CurrentSession`. Look at `IsCookerVoltageOK` and the upstream `CurrentErrorCode` to find which error tripped the cooker.
- "How do I add a new kWh display somewhere?" → make it read from `CookingSession_GetCurrentSession` (for live mid-cook) or `SessionAggr_GetLatestSessionPowerUsage` (for finalised last-session). Don't try to recompute — there's exactly one accumulator.
- "How do I scale for N burners?" *(experimental, prototype-branch path only)* → change `NUM_BURNERS_IN_LOCKSTEP` in `BurnerConfig.h` (defined only on the `G3-HWY-4-Burner` branch). Production branches don't include this header and the multiplier doesn't appear in the integrator. Single source of truth either way.

- **Status:** Reference doc — describes existing production behaviour. Section 11 (4-burner lockstep) is experimental on the `G3-HWY-4-Burner` prototype branch only; not bench-validated yet.


## G3 R2 → R3 hardware revision support
- **Product:** G3 (R3 hardware revision)
- **Date:** 2026-06-10
- **Branch:** R3
- **Files changed:** `STM32_FIRMWARE_G3.ioc`, `Core/Inc/main.h`, `Core/Src/main.c`, `Core/Src/gpio.c`, `Core/Src/rtc.c`, `Core/Src/stm32g0xx_it.c`, `Core/Src/usart.c`, `Core/Startup/startup_stm32g0b0cetx.s` (new), `STM32G0B0CETX_FLASH.ld` (new), `Drivers/STM32G0xx_HAL_Driver/*` (regenerated for G0B0), `ExternalPeripherals/RFComms/BTModemDriver/Src/ModemHw.c`, `ExternalPeripherals/Led/Inc/Led.h`, `ExternalPeripherals/Led/Src/Led.c`, `cmake/stm32cubemx/CMakeLists.txt`. Reference doc: `R2_TO_R3_CHANGES.md` in project root; MCU-swap mechanics in `MIGRATION_GUIDE.md`.
- **Purpose:** Bring the G3 firmware up on the R3 board revision. R3 changes the MCU (G071 → G0B0), flips BT module power polarity, adds a touch-driver LDO, and dual-colors the PAY LED. Five firmware deltas to make R3 work, plus one bugfix that fell out of bring-up.

### Hardware deltas R2 → R3
1. **MCU swap:** STM32G071CBT6 (128K Flash, 36K RAM, RTC on LSI) → STM32G0B0CETx (512K Flash, 144K RAM, RTC on LSE 32.768 kHz crystal). Pinout LQFP-48 compatible.
2. **BT_PWR_SW polarity flip (PB1):** R2 P-FET active-LOW gate (LOW = BT ON) → R3 LP5907MFX-3.3 LDO (U204) active-HIGH EN (HIGH = BT ON). Firmware now writes RESET=OFF, SET=ON for BT_PWR_SW.
3. **Touch driver LDO (NEW, U205 LP5907MFX-3.3):** PA9 (`TOUCH_PWR_SW`) active-HIGH EN drives +3V3_Touch rail, powering XW12A cap-touch ICs U500/U501. Requires 100 ms settle delay before I2C2 access.
4. **PAY LED dual-color:** R2 single yellow on PA12 → R3 TZ-LAMP03RG4HD-A with green on PA11 + red on PA12 (both N-FET low-side drive, active-HIGH at MCU).
5. **USART IRQ vector layout:** G0B0 combines `USART3_4_5_6_IRQHandler` (vs G071's `USART3_4_IRQHandler`). USART2 still has its own dedicated `USART2_IRQHandler`.

### Firmware changes for R3
- **MCU migration:** new startup file, linker script, HAL drivers, CMSIS headers, `STM32G0B0xx` build define. CubeMX regeneration with "Keep User Code" + "Delete Previous" required. See `MIGRATION_GUIDE.md`.
- **LSE crystal:** `__HAL_RCC_LSEDRIVE_CONFIG(RCC_LSEDRIVE_LOW)` + `LSEState = RCC_LSE_ON` in `SystemClock_Config()`. `RTCClockSelection = RCC_RTCCLKSOURCE_LSE` in `HAL_RTC_MspInit()`.
- **BT_PWR_SW polarity:** all writes flipped — `main.c` USER CODE BEGIN 2 boot-safety pin write is `GPIO_PIN_RESET` (BT off until Step 5); `ModemHw_PowerOn` writes SET; `ModemHw_PowerOff` writes RESET.
- **Touch LDO enable:** `HAL_GPIO_WritePin(TOUCH_PWR_SW_*, GPIO_PIN_SET)` + `HAL_Delay(100)` in `main.c` USER CODE BEGIN 2, before any I2C2 traffic.
- **PAY LED dual-color:** `main.h` splits into `GREEN_LED_CTL_PAY` (PA11) + `RED_LED_CTL_PAY` (PA12); `gpio.c` adds PA11 to the GPIOA output bitmask; `Led.h` splits `PAY_LED` enum into `RED_PAY_LED` + `GREEN_PAY_LED`; `Led.c` adds per-color switch cases everywhere. `INFO_PAY_LED` enum renamed to `INFO_LED` to disambiguate.
- **USART IRQ dispatch:** `stm32g0xx_it.c` `USART3_4_5_6_IRQHandler` USER CODE block calls `PowerBoardUart_IrqHandler()` + `DebugUart_IrqHandler()` (replaces the `USART3_4_IRQHandler` from G071).

### Bug found during R3 bring-up (also affects future hardware revisions with floating UART RX)
- **USART RX ORE storm in `BleUartInit` on cold reboot.** Floating PA3 (BT unpowered on R3) latched ORE between MX_USART2_UART_Init and BleUartInit. BleUartInit's enable of RXNEIE triggered an unhandled IRQ storm that starved USART4 TX, truncating boot logs at `[Flash] C`. Fix: clear `USARTx->ICR` error flags + drain RDR before enabling RXNEIE, in all three UART init functions (BleUartInit/PowerBoardUart_Init/DebugUart_Init); defensive ICR clear in all three IRQ handlers. Full investigation in `bugfixes.md` "R3 cold-reboot hang truncated at '[Flash] C'".

### Production hardware test harness (boot-time Steps 0–6)
| Step | Test                                           | Validates                              |
|------|------------------------------------------------|----------------------------------------|
| 0    | Control-board buzzer (3 beeps, 1s on/off)      | PD3 + 2N7002, unchanged from R2        |
| 1    | RTC set + 5s wait + read-back                  | LSE crystal (NEW on R3)                |
| 2    | All LEDs + 7-seg segments on (5s visual)       | Dual PAY LED wiring + display          |
| 3    | Keypad feedback (press each key once)          | Touch LDO + I2C2 + cap-touch ICs       |
| 4    | Power down LEDs/display                        | Current-budget prep for Step 5         |
| 5    | BT bring-up (BTStateMachine_TurnOn)            | BT_PWR_SW polarity + AT init           |
| 6    | BLE LOCK/UNLOCK prompt (nRF Connect, char 0xFFF2) | End-to-end BT comms                  |

### Pending items
- BT_PWR_SW polarity is hard-coded for R3. If a unified build needs to support both R2 and R3 in the field, this needs a hardware-revision compile flag.
- Stage A/B debug-bisection scaffolding (`STAGE_A_FLASH_ONLY` + `STAGE_B_*` macros in `main.c`) left in-tree dormant; useful for future peripheral-interaction debugging.
- ORE-storm bugfix should be backported to the G4 unified-firmware USART init/IRQ handlers if any of their RX lines can float in a similar topology.

## Usage Feature port: Prescan → Develop
- **Product:** G3 (Develop branch family — applies to Highway and CHK drivers equally; same accumulator code runs for both)
- **Date:** 2026-06-09 to 2026-06-10
- **Branch:** Usage-Feature, off Develop HEAD `ddb049f`, rebased onto `8465389`. Two commits ahead of `origin/Develop` after rebase: `005ae3e` (Phase C — pause across pot-lift) and `726f63a` (Phase D — INFO button).
- **Files changed:** `ExternalPeripherals/InductionCooker/Inc/CookingSession.h`, `Src/CookingSession.c`, `Src/IndCooker.c` (Phase C); `ProductFeatures/UI/Inc/UIPresenter.h`, `Inc/UIView.h`, `Src/UIPresenter.c`, `Src/UIView.c` (Phase D).
- **Purpose:** Re-implement the Usage Feature on the `Develop` branch using `UsageFeaturePort.md` as the spec. The spec was written assuming all 12 source commits from Prescan would need re-applying; in practice only 2 phases (≈ 60 % of the line count) were genuinely new work on Develop. This entry documents what was actually needed, what was already there, and the Develop-specific traps. See the Prescan port entry above ("Info button — realtime current-session + last-session usage screens", 2026-05-14) for the feature behaviour itself.

### What was already on Develop (do not re-port)
Bench-verified A1–A3 and B1–B3 against the pre-port build (commit `ddb049f`) before writing any code. Both phases passed end-to-end without any changes:

- **Phase A — Locked-timer integrity** (3 source commits `3d69214` / `0a4fdcd` / `bc3a8d3`): present in `UIView.c` (`Led_On/Off(LOCK_LED)` calls embedded in every locked-variant render function; colon-blink in `UI_VIEW_DISPLAY_CHILD_LOCK` / `_POWER_WITH_LOCK` tick cases) and in `UIPresenter.c` (`PendingLockActivation`, `QueueLockActivation`, `LockDisplayCycleCounter`, `UIPresenter_IsLockPending`, `UIPresenter_IsLockPhaseLoc`, `LOCK_PHASE_LOC_SECONDS`, `LOCK_PHASE_CYCLE_SECONDS`; LOCK long-press bound on `TIME_SETTING_SCREEN` and `TIMER_ACCEPTANCE`).
- **Phase B — End-of-session UX + E0 auto-shutdown** (4 source commits `eac762f` / `a70a5aa` / `5f37234` / `bf54aec`): present as `IsStateCooking(UIPresenterStateID_e State)` parameterised helper, `HandleErrorShutdown` routing via `IsStateCooking(SavedPreErrorState)` → `HandleShutdownSequence`, `ErrorAutoShutdownCounter` ticking in `UIPresenter_Tick`, `ERROR_AUTO_SHUTDOWN_SECONDS = 30`, and PWR+/- already bound on `TIME_SETTING_SCREEN` and `TIMER_ACCEPTANCE` in the state table.

If a similar port is ever attempted to another long-lived branch, run the bench tests against the pre-port head **first** — the spec lists 22 tests across 4 phases, and any subset that already passes is wasted re-port effort.

### What was new work (Phases C and D)
- **Phase C — Pause-not-end on pot lift** — `CookingSession_ResumeFromPause(const TIME_T*)` prototype + 4-line body in `CookingSession.{h,c}`; restructured `PowerBoardStatusCb` in `IndCooker.c` with a static `SessionPaused` bool and the pause branch. **Critical Develop-specific adjustment** — exclude `COOKER_ERROR_NO_POT_ERROR` from the `ErrorBlocking` check at the call site (see "Phase C pot-lift pause naive port terminated the session on first lift" entry in `bugfixes.md` for the callback-ordering rationale). Develop's voltage-fault zero-power routing (E1/E2 → `CookerState_On` with `COOKER_POWER_NO_POWER` instead of `CookerState_Off`) is preserved.
- **Phase D — INFO button** — 7 new active presenter states + 4 dormant daily/monthly states (`#if 0` with `TODO(RTC)`), 3 new active views + 2 dormant. New helpers: `RealtimePwrPlusExit/MinusExit`, `RealtimeTimerIncShort/Long/DecShort/Long`, `RealtimeEngageLock`, `RealtimeDisengageLock`, `StartRealtimeTimerAfterAcceptance`, plus public `UIPresenter_IsRealtimeLocked` getter. View-side: new `RenderKwhFloat(value, includeMonthIcon, includeLockIcon)` shared helper (ShowSessionPower refactored to use it; ShowDailyPowerUsage/ShowMonthlyPowerUsage left alone — they use `Display_ShowFloat` directly, refactoring them is not in scope). Extended `IsStateCooking`, `IsTimerActiveState` (unconditional for `_TIMED[_LOCKED]` per fb9c6e8's rationale, gated on `!RealtimeTimerEditPending`), `HandleTimerExpiry` (LOCK_LED cleanup for locked realtime variants), `UIPresenter_HandleError` (whitelist + defensive `Led_Off(LOCK_LED)` in the STANDBY fallback), `HandleAutoOff` (lock-detection branch). Tick-driven edit-counter commit logic in `UIPresenter_Tick`.

### Rebase outcome (onto `8465389`, the RTC drift workaround merge)
Two textual conflicts during Phase C replay, both at the same insertion point in `CookingSession.{h,c}` — the upstream RTC fix added `bool CookingSession_IsActive(void)` immediately after `CookingSession_GetCurrentSession`, exactly where Phase C added `CookingSession_ResumeFromPause`. Resolution: keep both functions side-by-side. The two functions don't interact — `CookingSession_IsActive` is a read-only getter, `CookingSession_ResumeFromPause` realigns `CurrentSession.End`. The drift-corrected `UserRtc_GetRtcTime` only makes the pause realignment more accurate. Phase D replayed cleanly with zero conflicts.

### FLASH footprint
- Before port (`ddb049f` build): 97.05 % of 128 KB (≈ 124.2 KB / 130 016 B)
- After Phase C: 97.07 %
- After Phase D: 99.16 %
- After rebase (includes RTC drift workaround): **99.39 %** (130 272 B / 131 072 B)

Roughly 800 bytes of headroom on the G071 build. Future material additions to this branch family will need either size optimisation or the R3 MCU bump (G0B0 has 512 KB Flash — see the R3 features.md entry from 2026-06-10). The 4 dormant daily/monthly states / 2 dormant views are `#if 0`'d and contribute zero bytes; re-enabling them when the RTC arrives will cost a few hundred bytes more.

### Acceptance verdict
All 14 bench tests in `UsageFeaturePort.md` §6 passed: A1–A3, B1–B3, C1–C3, D1–D8. C2 initially failed on the naive port — see the dedicated bugfix entry. C2 retest passed after the `NO_POT_ERROR` exclusion fix was folded into the Phase C commit (squashed before the rebase). All other tests passed on first run.

### Guidance for future ports of this feature
1. **Cherry-pick is non-viable across diverged branches.** The Develop port took ≈ 1 day of focused work; a cherry-pick attempt against Develop's current shape would conflict on nearly every file.
2. **Run the bench tests against the pre-port head FIRST.** Several phases may already be merged via other PRs.
3. **For the pause logic specifically, run C1–C4** (NOT just C1/C2/C3). C2 exposes the callback-ordering bug; **C4 (lift then OFF before pot returns) exposes the pause-accumulator freeze bug** discovered in Copilot PR review — see "Pot-lift pause didn't freeze the accumulator" in bugfixes.md. The original 3-test bench plan from the spec is incomplete.
4. **The Develop `PowerBoardStatusCb` already has voltage-fault zero-power routing** that doesn't exist on Prescan. Don't remove it during the port; layer pause on top.
5. **Develop has Token / Tamper / Pay states adjacent to USER_COOKING.** Verify ENERGY_INFO_BUTTON stays inert in those (default for missing state-table entries is NO_ACTION, so as long as nothing is added there, it's safe).
6. **The realtime in-place TIMR edit needs a tick-driven `else` (not `else if !Pending`)** to clear the pending flag on non-LOCK exits — see "RealtimeTimerEditPending stuck after non-LOCK exit" in bugfixes.md. Mistake is easy to make; bench coverage won't catch it without a dedicated test.

### Post-merge review findings (Copilot, 2026-06-10)
The first commits that shipped the port (`005ae3e`, `726f63a`) bench-passed all A1–A3, B1–B3, C1–C3, D1–D8 but still had two non-obvious bugs that AI code review caught:
1. **Pause-accumulator freeze bug** — `PowerBoardStatusCb` pause branch didn't freeze `End` / `ActualPower`; stop-during-pause overcounted kWh. Detailed entry in `bugfixes.md`. Fixed in `247bb08` via new `CookingSession_PauseAccumulation`. Added bench test C4.
2. **Sticky `RealtimeTimerEditPending`** — `UIPresenter_Tick`'s `else if (!Pending)` only cleared the counter when the flag was already false, leaving it sticky if the user exited `INFO_REALTIME_TIMED` mid-edit via PWR+/-, ON/OFF, or INFO. Detailed entry in `bugfixes.md`. Fixed in `247bb08` by changing `else if` to plain `else`.

Both bugs survived a clean A1–D8 bench pass. **Adding code review (human or AI) to the port workflow caught them in roughly one round of review and a single follow-up commit.** Treat this as a data point: bench-test coverage alone is insufficient for integrator/freeze patterns and pending-flag state machinery.

Other (smaller) Copilot findings in the same review round:
3. `Hundths` typo → renamed to `HundredthsDigit` in both `RenderKwhFloat` and `ShowRealtimeTimedPower`.
4. `INFO_REALTIME_TIMED_ACCEPTANCE` was missing from the `UIPresenter_HandleError` whitelist — E0 during the 5× flash would have dropped to STANDBY on clear. Added.
5. `USE_POWER_BOARD_MEASURED_POWER` inline rationale expanded so the macro choice is discoverable from the source, not just PR comments.
6. `SessionAggr_Tick` docstring updated to point at `main.c`'s tick block (the actual call site) rather than the original `UIPresenter_Tick` draft.

- **Status:** Verified (all 15 bench tests pass on bench hardware post-`247bb08`: A1–A3, B1–B3, C1–C4, D1–D8). PR #34 open against `Develop`, ready for merge from the implementer's side.

## Silent Thermal Regulator (PL1-throttle + ADC-indexed Highway lookup)
- **Product:** G3
- **Date:** 2026-06-19
- **Branch:** Feature/Temperature-Control
- **Commit:** c67c50a
- **Files changed:**
  - `ExternalPeripherals/ThermalRegulator/Inc/ThermalRegulator.h` (new)
  - `ExternalPeripherals/ThermalRegulator/Src/ThermalRegulator.c` (new)
  - `ExternalPeripherals/PowerBoard/Inc/HighwayTempLookUp.h` (rewrite: ADC-indexed)
  - `ExternalPeripherals/PowerBoard/Src/HighwayDriver.c` (regulator wiring, lookup simplify, debug)
  - `ExternalPeripherals/PowerBoard/Src/CHKDriver.c` (regulator wiring)
  - `ExternalPeripherals/PowerBoard/Src/PowerBoard.c` (single `POWER_BOARD_TYPE` define)
  - `CMakeLists.txt` (sources + include path)
  - `.cproject` (include path)

### Purpose
Eliminate user-visible E3 (`COOKER_ERROR_SURFACE_TEMP_SENSOR_OVER_TEMPERATURE`) trips during normal cooking. Pre-feature, users hitting the power board's onboard 188 °C trip would see E3 and try to clear it by toggling ON/OFF — a UX wart. The new regulator silently soft-throttles the wire-level command so E3 only ever surfaces as a real hazard signal (regulator wasn't enough) rather than as routine.

### Design

**Shared driver-agnostic module.** `ThermalRegulator.c/.h` lives under `ExternalPeripherals/ThermalRegulator/`. Both `HighwayDriver` and `CHKDriver` feed it via `ThermalRegulator_Update(potT, igbtT, sensorError)` once per RX status frame, and resolve their TX wire-level command via `ThermalRegulator_ApplyToLevel(CurrentPowerLevel)`. The regulator never mutates `CurrentPowerLevel`, so the UI shows the user's selected level throughout — it is a *silent* throttle.

**Pot/surface NTC: 3-state machine.**

```
First overshoot per power-up:
  NORMAL --175C--> HARD_CUT --165C--> NORMAL  (direct, 10 C hysteresis)

Subsequent overshoots:
  NORMAL --175C--> THROTTLED (PL1 200 W) --165C--> NORMAL

PL1 escape safety floor (rare in steady state):
  THROTTLED --185C--> HARD_CUT --175C--> THROTTLED --165C--> NORMAL
```

Why the asymmetry: the first overshoot per power-up is when the cooker has the most pent-up thermal energy and PL1 alone may not be enough (verified in Run 3 where PL1 escaped to HARD_CUT). Going straight to HARD_CUT on the first overshoot opens a **13 °C margin** from E3 (175 vs 188), instead of the tight 3 °C margin a `THROTTLED -> HARD_CUT` escalation would leave. After the first overshoot has been fully cleared, the thermal mass has settled and PL1 alone holds the pot in the 165–175 °C band reliably (verified in Run 6).

**HARD_CUT exit threshold is origin-aware.** The fix that came out of Run 5: HARD_CUT entered from NORMAL (first overshoot, at 175) exits at 165 directly to NORMAL. HARD_CUT entered from THROTTLED (PL1 escape, at 185) exits at 175 back to THROTTLED, then to NORMAL at 165. Each entry has its own 10 °C hysteresis. Tracked by `HardCutFromThrottled` bool. Without this, Run 5 showed a degenerate one-frame HARD_CUT (entry at 175, exit at 175 same frame).

**IGBT NTC: simple bang-bang**, 70/65 thresholds, always hard cut. Sensor open/short/failure forces the matching latch ON as defence-in-depth. No PL1 path on the IGBT side — IGBT only ever does hard cut.

**Compile-time toggles in `ThermalRegulator.c`:**
- `THERMAL_REGULATOR_ENABLED` (default 1): set to 0 to bypass `ApplyToLevel` output while keeping the state machine and its debug prints active. Used to characterise the power board's onboard E3 trip without firmware interference.
- `POT_THROTTLE_MODE_PL1` (default 1): set to 0 to revert pot side to a simple 2-state bang-bang (HARD_CUT at 175, NORMAL at 165) — the original baseline.

### Highway ADC-indexed lookup port (the upstream fix that unblocked everything)

Pre-feature, `HighwayTempLookUp.h` was indexed by *computed* NTC resistance from the formula `R = 5.1 * 255 / ADC - 5.1` (POT) or `R = 10 * 255 / ADC - 10` (IGBT), then cast to `uint16_t` to index a 326-entry table. Integer truncation collapsed many ADC values into table index 0 or 1:

| ADC | Computed R (kΩ) | Index (int cast) | G3 reads as |
|---|---|---|---|
| 192–213 | 1.0–1.7 | 1 | 165 °C |
| 214–254 | <1.0 | 0 | 255 °C (sentinel) |

At real cooktop temperatures of 168–175 °C (ADC ~210–215), the firmware oscillated between 165 °C and 255 °C frame-to-frame with single-LSB ADC noise. Run 1 showed the regulator firing 10+ cuts/resumes per second on this bogus data.

Ported G4's ADC-indexed `PotTempFromADC[256]` and `IgbtTempFromADC[256]` tables from `STM32_FIRMWARE_G4` Temp-Control branch. Direct indexing, no float math, no integer-truncation collapse, 1 °C resolution across the whole ADC range. ADC=0 and ADC=246–255 map to the 255 sentinel (invalid). The conversion functions in `HighwayDriver.c` collapsed to single-line table lookups. Validation in Run 2 immediately showed real temperatures (e.g. ADC=215 → 175 °C exactly), and Run 4 verified the lookup is calibrated to the Highway board: with regulator disabled, E3 fired at ADC=224 → 188 °C — matches the power board's onboard trip point with `DEFAULT_FURNACE_HIGH_TEMP_PROTECTION_SETTING=0xE0`.

### PowerBoard.c consolidation
The driver selection (`PowerBoardType = COOKER_POWER_BOARD_HIGHWAY`) was hardcoded in two separate places (`PowerBoard_Init` and `PowerBoard_SetBaudRate`). Replaced both with a single `#define POWER_BOARD_TYPE COOKER_POWER_BOARD_HIGHWAY`. Changing board now means editing one line, not remembering two.

### Debug instrumentation
- `THERMAL_REGULATOR_DEBUG_ENABLED 1` in `ThermalRegulator.c`: edge-only control-action prints. Examples:
  - `ThermalRegulator: pot HARD CUT (initial overshoot, wire=OFF+FAN) [NORMAL -> HARD_CUT] (temp=175C, fault=0)`
  - `ThermalRegulator: pot SOFT CUT (wire=PL1 200W) [NORMAL -> THROTTLED] (temp=175C, fault=0)`
  - `ThermalRegulator: pot RESUME (from hard cut direct, wire=user level) [HARD_CUT -> NORMAL] (temp=165C, fault=0)`
  - `ThermalRegulator: pot RELAX (hard cut -> soft cut, wire=PL1 200W) [HARD_CUT -> THROTTLED] (temp=175C, fault=0)`
  - `ThermalRegulator: IGBT HARD CUT (wire=OFF+FAN) (temp=70C, fault=0)`
- `HIGHWAY_TEMP_DEBUG_ENABLED 1` in `HighwayDriver.c` (separate from `HIGHWAY_DRIVER_DEBUG_ENABLED`): per-frame diagnostic throttled to 1 in every 50 RX frames (~1 Hz), e.g. `Highway frame: err=0 potADC=214 potT=173C igbtADC=42 igbtT=41C sysFlag=0x88 user=11 eff=11`. `user=N eff=M` shows user-requested vs regulator-resolved wire level; `user != eff` flags an active throttle/cut.
- Highway error transitions logged with `*** E3 FIRED ***` markers for trip-point characterisation work.

### Verification (bench runs)
- **Run 1:** Original (resistance-indexed) lookup table — exposed integer-truncation collapse. Pot temps stuck at 165/255 alternation.
- **Run 2:** Post-lookup-port, pre-regulator. Pot reads accurate temperatures (175 °C at ADC=215 etc.). NTC oscillation gone.
- **Run 3:** Regulator on (3-state PL1 mode, original symmetric hysteresis) — confirmed PL1 escape to HARD_CUT on first overshoot, then steady-state ~50 % duty cycle between NORMAL and THROTTLED.
- **Run 4:** Regulator output bypassed (`THERMAL_REGULATOR_ENABLED 0`) — E3 fired at exactly ADC=224 / pot=188 °C. Confirms lookup table calibration and gives the 13/3 °C margin numbers used in the asymmetric design.
- **Run 5:** First-overshoot HARD_CUT logic enabled but exposed a one-frame HARD_CUT bug (entry 175, exit 175, no hysteresis). Fixed with origin-aware exit threshold.
- **Run 6:** Final form. Initial HARD_CUT at 175 holds for ~30 s until 165, then transitions to NORMAL directly. All subsequent overshoots are SOFT_CUT → THROTTLED → 165 → NORMAL with proper 10 °C hysteresis. **Zero HARD_CUT escapes after the first overshoot. Zero E3 trips. Clean control behaviour.**

### Boot-with-hot-pot caveat (known, accepted)
If the pot is residually hot from a previous session and ≥ 175 °C on the first status frame after boot, the regulator will trigger its "first overshoot" HARD_CUT immediately and hold it until the pot cools to 165 °C. The cooker will not heat during that window even if the user turns on. Defensible as a conservative safety behaviour, but worth noting. Could be refined later by adding a "warm-up below threshold seen" gate before counting an overshoot as "first".

### IGBT calibration caveat (open question)
IGBT temperature reads consistently low across all runs (range 28–42 °C during active cooking on Run 2/3). Three possibilities: (a) the IGBT is genuinely well-cooled and 30–50 °C is correct, (b) the G4-ported IGBT lookup uses a different NTC β than the actual G3 IGBT NTC part, or (c) `Data[2]` is not quite the IGBT ADC byte we think it is. Not blocking — regulator's 70 °C IGBT threshold is far above observed values. Worth physical heatsink touch-check next time the cooker is on bench.

- **Status:** Verified (Run 6 bench test on Highway with real pot — clean first-overshoot HARD_CUT, all subsequent overshoots handled by PL1 SOFT_CUT, zero E3 trips, all transitions have proper 10 °C hysteresis).

### Post-feature sanity-check fix (commit `9042c28`)
Code-review pass after Run 6 caught a regression where `ApplyToLevel`'s THROTTLED branch returned `COOKER_POWER_200W` unconditionally, including when the user had commanded OFF / 0 W / NO_POWER_FAN_ON. Pressing OFF mid-throttle would have continued heating the pot at PL1 until the temp dropped below 165 °C. Fixed by guarding the THROTTLED substitution against no-heat user requests — see "ThermalRegulator throttled OFF still heats at PL1" in bugfixes.md. Bench test E1 to confirm pending next cooker session.
