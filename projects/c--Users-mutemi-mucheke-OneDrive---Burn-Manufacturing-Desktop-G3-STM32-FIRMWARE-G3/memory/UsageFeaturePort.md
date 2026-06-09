# Usage Feature — Port Specification

> **Purpose of this document:** a portable specification of the "usage feature" originally developed on the `Prescan` and `CHK-Bring-Up` branches, written for an implementer (human or AI) who will re-apply it on `Develop`. Develop has diverged substantially from the source branches, so cherry-picking the 12 source commits produces conflicts on nearly every file. The right approach is to *re-implement* the feature against Develop's current state machine and APIs using this document as the spec.
>
> **Source branches:** `Prescan` (commit `92bf53f`), `CHK-Bring-Up` (commit `0bc27e6`).
> **Target branch:** `Usage-Feature` (currently off `Develop` HEAD `ddb049f`).
> **Author of spec:** initial Claude Code session, 2026-06-05.
> **Status:** draft spec — implementation pending.

---

## 1. What "the usage feature" means

The user-visible feature has four parts:

1. **Realtime usage during cooking.** When the user presses the **INFO button** (capacitive-touch key, `ENERGY_INFO_BUTTON` in the keypad enum) while a cooking session is active, the display shows the live kWh accumulated so far in the session. The screen updates as the kWh advances (~1 Hz cadence). Persists until dismissed.

2. **Last-session usage from STANDBY.** When the user presses INFO from STANDBY, the display briefly shows the word "USED" then the kWh value of the last completed session. Auto-returns to STANDBY after a short timeout.

3. **Automatic session-usage display at the end of every cook.** When a cooking session ends (user pressed OFF, timer expired, E0 auto-shutdown), the display plays a **"USED" → final-kWh → STANDBY** sequence as part of the shutdown flow. This happens regardless of how the session ended.

4. **Locked + timed cooking interactions work cleanly.** All the small UX fixes that were needed to make the realtime / last-session views co-exist with the existing child-lock and timer features. Includes:
   - LOCK long-press during realtime engages child lock without dismissing the realtime view.
   - LOCK_LED tracks lock state correctly across timer expiry, error recovery, auto-off warning.
   - Colon blinks during locked timed cooking.
   - PWR+/- and TIMR+/- still work in `TIME_SETTING_SCREEN` and `TIMER_ACCEPTANCE`.

The feature **must** also include the E0 / pot-lift behaviour change:

5. **E0 (pot-lift) PAUSES the session instead of ending it.** Lifting the pot mid-cook used to terminate the session via `PowerBoardStatusCb → CookerState_Off → CookingSession_Stop`, which overwrote `LatestSessionData.CookerPower` with just the last segment. After the fix, pot-absent is treated as a pause — the session stays active, accumulation skips that frame, and on pot return the session resumes from where it was (with `End` realigned via `CookingSession_ResumeFromPause` so the paused interval isn't credited as cooking energy). E0 is still raised via the `ErrorCallback` path. The session only terminates on real events: voltage fault, user OFF press, or the 30-second E0 auto-shutdown.

---

## 2. High-level architecture summary

### New presenter states (7 + 4 commented-out daily/monthly)

To be added to `UIPresenterStateID_e` in `ProductFeatures/UI/Inc/UIPresenter.h`:

```
UI_PRESENTER_STATE_INFO_LAST_SESSION_LABEL    // "USED" text, 3 s, then →
UI_PRESENTER_STATE_INFO_LAST_SESSION_VALUE    // last-session kWh, 10 s, → STANDBY

UI_PRESENTER_STATE_INFO_REALTIME              // full-screen kWh, no timeout
UI_PRESENTER_STATE_INFO_REALTIME_TIMED        // timer + kWh together, no timeout
UI_PRESENTER_STATE_INFO_REALTIME_LOCKED       // realtime view + LOCK_ICON
UI_PRESENTER_STATE_INFO_REALTIME_TIMED_LOCKED // timed realtime + LOCK_ICON
UI_PRESENTER_STATE_INFO_REALTIME_TIMED_ACCEPTANCE // 5x flash of timer-only digits during in-place TIMR commit
```

Plus 4 dormant states for daily/monthly aggregation (wrapped in `#if 0` until RTC hardware lands):

```
UI_PRESENTER_STATE_INFO_DAILY_LABEL    // "1D" text
UI_PRESENTER_STATE_INFO_DAILY_VALUE    // daily kWh + KWH icon
UI_PRESENTER_STATE_INFO_MONTHLY_LABEL  // "30D" text
UI_PRESENTER_STATE_INFO_MONTHLY_VALUE  // monthly kWh + KWH icon + MONTH icon
```

### New view types (3 + 2 commented-out)

To be added to `UIView_e` in `ProductFeatures/UI/Inc/UIView.h`:

```
UI_VIEW_DISPLAY_REALTIME_POWER            // kWh + KWH icon, full-screen
UI_VIEW_DISPLAY_REALTIME_TIMED_POWER      // timer + colon + kWh + KWH/HOUR/MIN icons
UI_VIEW_DISPLAY_REALTIME_TIMED_FLASH      // same layout but timer digits flash
```

Plus dormant (`#if 0`):
```
UI_VIEW_DISPLAY_STRING_1D
UI_VIEW_DISPLAY_STRING_30D
```

### New view render functions (in UIView.c)

- `ShowRealtimePower()` — reads `CookingSession_GetCurrentSession`, renders kWh value via existing digit-pack helper + KWH icon.
- `ShowRealtimeTimedPower()` — composes timer (DIGIT_1-3) + colon + HOUR/MIN icons + kWh (DIGIT_5-7 or 4-7) + KWH icon. Optionally overlays LOCK_ICON when `UIPresenter_IsRealtimeLocked()` is true.
- `ShowRealtimeTimedFlash()` — same layout but during the OFF phase only kWh + dot + KWH icon are visible (timer digits + colon + HOUR/MIN blanked).
- `RenderKwhFloat(float Value, bool IncludeMonthIcon, bool IncludeLockIcon)` — shared helper for digit-packing kWh into the display buffer. Caps at 99.99 kWh, two-decimal precision, dot at the fixed display-segment position.

### New presenter helpers (in UIPresenter.c)

State-transition helpers used by the realtime states:
- `RealtimePwrPlusExit`, `RealtimePwrMinusExit` — exit realtime, apply +/-1 level.
- `RealtimeTimerIncShort/Long`, `RealtimeTimerDecShort/Long` — in-place edit the timer (stages value in UIModel, marks pending flag, no state transition).
- `RealtimeEngageLock` — transition to the locked-realtime variant, set `SavedPreLockState` to the cooking state (not the realtime state), turn on `LOCK_LED`.
- `RealtimeDisengageLock` — transition back to the unlocked realtime variant, turn off `LOCK_LED`.
- `StartRealtimeTimerAfterAcceptance` — commit the staged timer value to IndCooker (`SetInductionCookerTimerFromUIModel`), return to plain `INFO_REALTIME_TIMED`.
- `UIPresenter_IsRealtimeLocked` — public getter exposed in `UIPresenter.h` so the view layer can check whether to overlay LOCK_ICON.

Static module state:
- `RealtimeTimerEditPending` (bool) — set true when TIMR+/- pressed on realtime; cleared on commit or lock-engage.
- `RealtimeTimerEditCounter` (uint8_t) — counts down 3 seconds in `UIPresenter_Tick`; on reaching zero, fires the commit logic.
- `#define REALTIME_TIMER_EDIT_TIMEOUT 3` (seconds).
- `#define STANDBY_INFO_LABEL_TIMEOUT 2` (about 3 s effective, for the "USED" label timeout).

### Integrations with existing helpers

- `IsStateCooking()` — add all four realtime variants + the realtime acceptance state, so auto-off counter / fan-cooldown / error-recovery whitelist all treat realtime as a cooking state.
- `IsTimerActiveState()` — add `INFO_REALTIME_TIMED` and `INFO_REALTIME_TIMED_LOCKED` *unconditionally* (do **not** gate on `UIModel_IsCookTimerRunning()` — see commit `fb9c6e8`'s rationale). Also gate via `!RealtimeTimerEditPending` to suppress timer expiry during in-place edit.
- `HandleTimerExpiry()` — LOCK_LED cleanup branch should include the two `_LOCKED` realtime variants so the LED turns off when the timer expires while locked.
- `UIPresenter_HandleError()` — NO_ERROR branch's restore whitelist must include all four realtime variants so error-recovery returns to the realtime view instead of falling through to STANDBY.
- `HandleAutoOff()` — at the warning threshold, if currently in any locked variant (CHILD_LOCK / CHILD_LOCK_SHOW_POWER / INFO_REALTIME_LOCKED / INFO_REALTIME_TIMED_LOCKED), transition to `CHILD_LOCK_SHOW_POWER` with `SavedPreLockState = USER_COOKING_WITH_TIMER` instead of bare `USER_COOKING_WITH_TIMER`. Preserves the lock through the auto-off warning.

### kWh data source for the new views

All new views read the same accumulator that the existing post-cook "USED" screen uses:

- Realtime views call `CookingSession_GetCurrentSession(&Session)` and use `Session.CookerPower` (live, in-progress kWh).
- Last-session view calls `SessionAggr_GetLatestSessionPowerUsage()` (final kWh from the most recent session, persisted to flash by `SessionAggr_UpdatePowerUsage` at session stop).

No new accumulator. No new conversion. Both already exist in the firmware.

### View refresh cadences (in `UIView_Process` tick switch)

- `UI_VIEW_DISPLAY_REALTIME_POWER` — re-render once per second (10 ticks of the 100 ms UI tick, via a local static counter in the case).
- `UI_VIEW_DISPLAY_REALTIME_TIMED_POWER` — re-render every tick (100 ms) so the blinking colon toggles visibly. kWh source only changes ~1 Hz so digits stay stable between updates.
- `UI_VIEW_DISPLAY_REALTIME_TIMED_FLASH` — drives the 5× flash counter using the same `AcceptanceFlash*` static vars as the existing `UI_VIEW_TIMER_ACCEPTANCE_FLASH` (only one is active at a time, no conflict). On the 5th flash completion, calls `UIPresenter_ForceStateTimeout` which fires `StartRealtimeTimerAfterAcceptance`.

---

## 3. The 12 source commits

These are the original commits on Prescan. The implementer should **read each one's diff** with `git show <sha>` (or use the SHAs to look them up on GitHub) for full context. The summaries below capture intent and integration notes for the Develop port.

### Order matters — apply in chronological order

The commits build on each other. Listed below in chronological order (oldest first). Within each, the rough phase is shown so the implementer can group them sensibly if doing the port in stages.

---

### Phase A — Locked-timer integrity (foundation)

#### `3d69214` — Fix lock LED staying on after exiting child lock in timed cooking
*1 file, 5 insertions*

Drive `LOCK_LED` from the active view (symmetric with `PWR_LED`) so the LED state follows the view. Closes a race where a second LONG_PRESS during `LockToggleCooldown` caused `ExitChildLock` to return early without calling `Led_Off`, leaving the LED on while the state transitioned back to `USER_COOKING_WITH_TIMER`.

**Develop integration:** Develop's `UIView.c` has a renamed API (`Display_SetLEDs` → `Display_EnableDisableNumpadLEDs`). When re-implementing, the lock-LED assertion lines should be added at the start of each render function that displays a locked variant — there's nothing special about how the assertion is done, just that it's done.

#### `0a4fdcd` — Blink colon during child lock in timed cooking session
*1 file, 22 changes*

Tick `ColonBlinkCounter`/`ColonVisible` during `UI_VIEW_DISPLAY_CHILD_LOCK` and `UI_VIEW_DISPLAY_POWER_WITH_LOCK` so the colon blink continues at 1 Hz when locked. `ShowPowerLevelWithLockIcon` now gates the colon on `ColonVisible` (previously always solid).

**Develop integration:** Develop has the same views (`UI_VIEW_DISPLAY_CHILD_LOCK`, `UI_VIEW_DISPLAY_POWER_WITH_LOCK`). Add the colon-blink switch cases in `UIView_Process`.

#### `bc3a8d3` — Allow locking during timer setting/acceptance; fix LOCK_LED stuck on expiry
*3 files, 184 / 29 = 213 lines changed*

Pressing lock during `TIME_SETTING_SCREEN` or `TIMER_ACCEPTANCE` no longer aborts the timer flow. Instead the timer flash plays, the timer starts, and the cooker lands in `CHILD_LOCK_SHOW_POWER` — while the lock icon and "LOC" appear in the power section the instant the long-press fires. The power section cycles "LOC" (3 s) / power level (9 s) driven by a counter that starts at the moment of lock press, decoupled from the state machine. The timer section persists through locked timed cooking. `LOCK_LED` now turns off when the cooking timer expires while locked.

**New helpers introduced here:** `QueueLockActivation` (queues a lock to be activated after timer acceptance completes), `PendingLockActivation` flag, `LockDisplayCycleCounter` static, `LOCK_PHASE_LOC_SECONDS` / `LOCK_PHASE_CYCLE_SECONDS` constants, `UIPresenter_IsLockPending()` / `UIPresenter_IsLockPhaseLoc()` public getters used by the view layer.

**Develop integration:** Develop has all the prerequisite states (`TIME_SETTING_SCREEN`, `TIMER_ACCEPTANCE`, `CHILD_LOCK[_SHOW_POWER]`). The new helpers and flags can be added cleanly. The view-layer cycling of "LOC" vs power level uses the `LockDisplayCycleCounter` — `UIPresenter_IsLockPending()` and `_IsLockPhaseLoc()` are the bridge between the presenter's lock state and the view's render decisions.

---

### Phase B — End-of-session usage display + E0 auto-shutdown

#### `eac762f` — Show usage screen when shutting down from a mid-cook error
*1 file, 23 / 14 lines*

`HandleErrorShutdown` used to jump straight to `STANDBY`, skipping `SHUTDOWN_USED_LABEL → session-kWh → OFF`. When an error fires mid-cook and the user presses ON/OFF (or auto-shutdown fires), the usage screen now plays, mirroring the normal shutdown flow including fan cooldown. Errors that fired *outside* a cooking state still go directly to STANDBY.

**Refactor:** introduces `IsStateCooking(UIPresenterStateID_e)` parameterised helper (previously `IsInCookingState` was self-bound). The parameter form is needed so `HandleErrorShutdown` can query `SavedPreErrorState`.

**Develop integration:** `HandleErrorShutdown` exists on Develop. Refactor it to call `IsStateCooking(SavedPreErrorState)` (introduce the helper alongside the existing `IsInCookingState`); if cooking, run `HandleShutdownSequence` (which produces the USED → final-kWh → STANDBY sequence); else `UIModel_PowerOffPowerBoard` + reset.

#### `a70a5aa` — Auto-shutdown after 60 s of persistent E0 mid-cook
*1 file, 22 lines added*

E0 used to sit on the display indefinitely; only a manual ON/OFF press triggered the usage/shutdown flow. Now a dedicated 60-second counter (`ErrorAutoShutdownCounter`, with `#define ERROR_AUTO_SHUTDOWN_SECONDS 60`) ticks while E0 is active and the pre-error state was a cooking state; on expiry it runs `HandleErrorShutdown`, producing the same "USED" → kWh → OFF → STANDBY sequence (with fan cooldown for sessions >= 60 s).

Also suppress E0 entirely when it fires outside a cooking state — no `DISPLAY_ERROR` entry in that case. The new counter is independent of `StateTimeOutCounter`, so keypresses during the error do not extend the 60 s window.

**Develop integration:** `HandleAutoOff` / `HandleTimerExpiry` already use independent counters — same pattern works here. Add `ErrorAutoShutdownCounter`, increment it in `UIPresenter_Tick` if `ErrorActive && CurrentError == COOKER_ERROR_NO_POT_ERROR && IsStateCooking(SavedPreErrorState)`; on threshold, call `HandleErrorShutdown`.

#### `5f37234` — Enable PWR+/- during TIME_SETTING_SCREEN and TIMER_ACCEPTANCE
*1 file, 30 / 10 lines*

Power level was locked for the whole ~7 s window between pressing TIMR+ and the timer actually starting. Now PWR+/- adjust the power level in both the static time-setting screen and the acceptance flash, same bindings as `USER_COOKING`. Pressing PWR+/- during `TIME_SETTING_SCREEN` also resets its 2 s "stop adjusting" timeout.

**Develop integration:** Both states exist on Develop. In their state-table entries, replace `NO_ACTION_ON_KEYPRESS` for PWR+/- with `{ ShortPressNextStateID = NO_SUPPORTED_STATE, ActionFunctionShortPress = UIModel_IncrementPowerSetting }` (and the corresponding decrement). For `TIME_SETTING_SCREEN` only, the action also needs to reset `StateTimeOutCounter` — this happens automatically because every keypad event resets the counter (see `UIPresenter_Process`).

#### `bf54aec` — Reduce E0 auto-shutdown timeout from 60 s to 30 s
*1 file, 1 / 1 line*

Single-line tweak: `#define ERROR_AUTO_SHUTDOWN_SECONDS 30` (was 60). User-experience refinement — 60 s was too long.

**Develop integration:** trivial follow-up to `a70a5aa`.

---

### Phase C — E0 pause (the user's explicit ask)

#### `083b57d` — Preserve cooking session across pot-lift pauses
*Touches `IndCooker.c` and `CookingSession.c` (+`.h`)*

**This is the single most important behavioural change in the feature.** Lifting the pot mid-cook was ending the session via `PowerBoardStatusCb → CookerState_Off → CookingSession_Stop`, which called `SessionAggr_UpdatePowerUsage` and overwrote `LatestSessionData.CookerPower` with just the segment that had run up to the lift. A lift-return-lift sequence therefore showed only the last small segment on the shutdown usage screen (~0.01 kWh) even though the true session was much longer.

Fix: treat pot-absent as a **PAUSE** rather than session end. `PowerBoardStatusCb` no longer calls `CookerState_Off` on pot-absent while the cooker is still commanded ON; the session stays active and simply skips accumulation for that frame. E0 is still raised via the `ErrorCallback` path (unchanged).

Added `CookingSession_ResumeFromPause(const TIME_T* CurrentTime)` which realigns `CurrentSession.End = now` when the pot returns, so the paused window isn't attributed as cooking energy on the next `UpdateSessionPower` call.

Session now only terminates on real events — voltage fault, user OFF press, or E0 auto-shutdown.

**Develop integration — biggest care point.** Develop's `PowerBoardStatusCb` has been restructured significantly. The Develop version routes voltage faults through `CookerState_On` with zero power (rather than `CookerState_Off`) and uses a `CookerErrorExceptVoltageIssue()` helper. The implementer must:

1. Understand Develop's current logic for pot-absent (does it currently end the session, or does it already pause?).
2. If it currently ends, add a `SessionPaused` static + early-return on pot-absent + call to the new `CookingSession_ResumeFromPause` on pot-return.
3. Add the new public `CookingSession_ResumeFromPause(const TIME_T*)` function to `CookingSession.c` + `.h`.

The new function is trivial:
```c
void CookingSession_ResumeFromPause(const TIME_T* CurrentTime) {
    if (!CurrentSession.isActive) return;
    CurrentSession.End = *CurrentTime;
}
```

**Verification critical**: a lift-return-lift sequence + a manual OFF must show the full cook kWh on the shutdown screen, not just the last segment.

---

### Phase D — The Info button itself (the core feature)

#### `6862220` — Info button: realtime + last-session usage screens
*Large commit touching `UIPresenter.h`, `UIPresenter.c`, `UIView.h`, `UIView.c`*

The core feature. Adds 7 new presenter states (5 realtime variants + LAST_SESSION_LABEL/VALUE), 3 new view types (REALTIME_POWER, REALTIME_TIMED_POWER, REALTIME_TIMED_FLASH not yet — that comes in 25df579), 4 dormant daily/monthly states wrapped in `#if 0`.

**Behaviour:**

*From STANDBY:*
- INFO short-press → `INFO_LAST_SESSION_LABEL` (shows "USED" for 3 s) → auto-transition to `INFO_LAST_SESSION_VALUE` (shows last-session kWh for 10 s) → `STANDBY`.
- ON/OFF from either screen → STANDBY (does NOT power on cooker).
- INFO from either screen → STANDBY (effectively dismisses).

*From USER_COOKING:*
- INFO short-press → `INFO_REALTIME` (full-screen kWh + KWH icon, refreshes 1 Hz, no timeout).
- Non-INFO keys exit with normal action applied (PWR+ → USER_COOKING + power increment, ON/OFF → `HandleShutdownSequence`, TIMR+/- → `TIME_SETTING_SCREEN` + increment, LOCK long → `INFO_REALTIME_LOCKED`).
- INFO short while on realtime → toggle off back to USER_COOKING.

*From USER_COOKING_WITH_TIMER:*
- INFO short-press → `INFO_REALTIME_TIMED` (timer + colon + HOUR/MIN + kWh + KWH icon). Same key handling as untimed.
- LOCK long → `INFO_REALTIME_TIMED_LOCKED`.

*Locked variants (`INFO_REALTIME_LOCKED`, `INFO_REALTIME_TIMED_LOCKED`):*
- All keys except LOCK long are inert.
- LOCK long → unlock back to corresponding unlocked realtime variant.
- LOCK_ICON overlaid via `UIPresenter_IsRealtimeLocked()` check in the view's render path; `LOCK_LED` asserted every render.

**RTC limitation:** the daily/monthly aggregation states (`INFO_DAILY_LABEL/VALUE`, `INFO_MONTHLY_LABEL/VALUE`) and their views (`UI_VIEW_DISPLAY_STRING_1D`, `_30D`) exist in code but are wrapped in `#if 0` because the hardware lacks a populated RTC. STANDBY INFO short-press is routed to `INFO_LAST_SESSION_LABEL` instead. When the RTC arrives, the implementer can re-enable the disabled block and re-target STANDBY's INFO transition.

**Develop integration:**
- Develop has Token / Pay / Tamper states adjacent to the relevant cooking states. Make sure INFO short-press is **inert** in those new states (Token entry, Tamper) — the realtime/last-session views should not be reachable from those flows.
- The view enum on Develop has more entries; add the new view enum values near the existing kWh-related ones for readability.
- Check that the keypad enum on Develop has `ENERGY_INFO_BUTTON` and the X12A capacitive touch mapping is correct (bitmask `0x0020` on I2C addr 2). If missing, the keypad hardware mapping must be added first.

#### `b1f620e` — Enable TIMR+/- during timer acceptance flash
*1 file, 16 / 2 lines*

Pressing TIMR+ or TIMR− while `UI_PRESENTER_STATE_TIMER_ACCEPTANCE` is showing the 5× acceptance flash now cancels the flash, returns to `TIME_SETTING_SCREEN` with the increment/decrement applied, and restarts the 3 s idle timeout. Previously these buttons were `NO_ACTION_ON_KEYPRESS` during the ~5 s flash window, forcing the user to wait.

Mirrors `USER_COOKING_WITH_TIMER`'s TIMR+/- wiring: short press uses `UIModel_Increment/DecrementTimerSetting`, long press uses the `*Fast` variants (±15 min jumps).

**Develop integration:** small wiring change to the `TIMER_ACCEPTANCE` state-table entry. `PendingLockActivation` is preserved across the round trip — no lock-flow regression.

#### `fb9c6e8` — Fix timer-expiry shutdown skipped when info button is showing
*1 file, 9 / 5 lines*

`IsTimerActiveState()` previously gated `INFO_REALTIME[_TIMED][_LOCKED]` on `UIModel_IsCookTimerRunning()`. IndCooker clears the running flag the instant the timer hits zero (via the power-board status callback path), so `HandleTimerExpiry` would see the guard fail and return early — the cooker had already stopped via the side channel but the UI never transitioned to `SHUTDOWN_USED_LABEL`.

Fix: make `INFO_REALTIME_TIMED` and `INFO_REALTIME_TIMED_LOCKED` return true **unconditionally** from `IsTimerActiveState()` (matches the existing `USER_COOKING_WITH_TIMER` pattern). `INFO_REALTIME` and `INFO_REALTIME_LOCKED` are removed from the list — they're only reachable from `USER_COOKING` and never have a running timer.

**Develop integration:** include this in the initial implementation of `IsTimerActiveState` for the new states. Don't make this a separate later fix — gate the wiring correctly from the start.

#### `25df579` — Info-button bundle: error recovery, in-place TIMR edit, lock-preserving auto-off
*3 files, 260 / 6 lines* — biggest commit, three sub-features bundled

Three related fixes that emerged from bench testing the info button feature:

**B5/B6/B7 — error recovery for realtime states.** `UIPresenter_HandleError`'s NO_ERROR branch had a hard-coded whitelist of states to restore to. The four realtime variants weren't in it, so when an error cleared while on a realtime screen the UI fell to STANDBY and the LOCK_LED was left on if the realtime state was locked. Fix: add `INFO_REALTIME[_TIMED][_LOCKED]` to the whitelist; defensive `Led_Off(LOCK_LED)` in the STANDBY fallback.

**A2 — TIMR+/- on realtime swapped screens.** Pressing TIMR+/- on `INFO_REALTIME_TIMED` used to exit to `TIME_SETTING_SCREEN` (which shows power level), violating the spec that the realtime view should stay on while editing. Fix: TIMR+/- now edits the timer **in-place** — kWh keeps updating, timer digits change. After 3 s of idle the staged value commits via a new `INFO_REALTIME_TIMED_ACCEPTANCE` state using a new `UI_VIEW_DISPLAY_REALTIME_TIMED_FLASH` view (5× flash of timer digits only, kWh stays solid). Decrement to 0:00 cancels the timer and falls through to `INFO_REALTIME` (untimed). New static `RealtimeTimerEditPending` flag + `RealtimeTimerEditCounter`; `IsTimerActiveState` suppresses timer-expiry while the flag is set so the old timer hitting zero doesn't lose the user's edit (same protection `TIME_SETTING_SCREEN` already gets by being omitted from that helper).

**F20 — auto-off warning bypassed child lock.** `HandleAutoOff`'s warning threshold forced `CurrentUIPresenterState = USER_COOKING_WITH_TIMER` unconditionally, silently dropping any active child lock. Fix: if currently in any locked variant (CHILD_LOCK / CHILD_LOCK_SHOW_POWER / INFO_REALTIME_LOCKED / INFO_REALTIME_TIMED_LOCKED), transition to `CHILD_LOCK_SHOW_POWER` instead of bare `USER_COOKING_WITH_TIMER` and set `SavedPreLockState = USER_COOKING_WITH_TIMER` so a subsequent unlock returns the user to the cooking screen with the auto-off timer running.

**Develop integration:**
- B5/B6/B7 fix: `UIPresenter_HandleError` exists on Develop and has the same whitelist pattern. Just add the realtime states.
- A2 fix: requires the new state `INFO_REALTIME_TIMED_ACCEPTANCE`, the new view `UI_VIEW_DISPLAY_REALTIME_TIMED_FLASH`, the helper functions, and the `RealtimeTimerEditPending` / `RealtimeTimerEditCounter` tracking in `UIPresenter_Tick`.
- F20 fix: `HandleAutoOff` exists on Develop; add the lock-detection branch.

---

## 4. Develop-specific integration notes

### File-level changes on Develop vs Prescan

- `Display_SetLEDs(bool)` → renamed to `Display_EnableDisableNumpadLEDs(bool)`. Any view function that called the old name needs the new name.
- `LockStringBuffer` (variable) → renamed to `LocStringBuffer` in `ShowChildLock`. Mostly an internal-name change; affects only one function.
- `Buzzer.h` is its own module on Develop. Buzzer calls may use a different API path.
- `UIView_e` has 10+ new view entries on Develop (Token, Tamper, Pay, all-segments display). Add new realtime view entries after the existing usage-related ones (DISPLAY_SESSION_POWER, DISPLAY_STRING_USED, etc.).
- `UIPresenterStateID_e` has 10+ new state entries on Develop (token / tamper). Add new realtime/last-session state entries in a clean block, typically before the SPECIAL_TEST_STATE / NUM_STATES sentinels.

### Power-board callback structure (the `083b57d` integration point)

Develop's `PowerBoardStatusCb` is restructured:

```c
void PowerBoardStatusCb(CookerPowerState_e PowerState, CookerPotPresence_e PotPresence, CookerPower_e PowerLevel) {
    DebugOut(...);

    if (!CookerErrorExceptVoltageIssue() && (PowerState == COOKER_POWER_STATE_POWER_ON) && (PotPresence != COOKER_POT_PRESENCE_ABSENT)) {
        if (!IsCookerVoltageOK()) {
            // route voltage faults through CookerState_On with zero power
            PowerLevel = COOKER_POWER_NO_POWER;
        }
        CookerState_On(&PowerLevel, &CurrentActualPower);
    } else {
        CookerState_Off();    // ← currently this path runs on pot-absent
    }

    CurrentPowerState = PowerState;
    CurrentPotPresence = PotPresence;
    CurrentPowerLevel = PowerLevel;
}
```

To implement the `083b57d` pause behaviour, this becomes:

```c
void PowerBoardStatusCb(CookerPowerState_e PowerState, CookerPotPresence_e PotPresence, CookerPower_e PowerLevel) {
    DebugOut(...);

    static bool SessionPaused = false;
    bool ErrorBlocking = CookerErrorExceptVoltageIssue();
    bool CookerOn = (PowerState == COOKER_POWER_STATE_POWER_ON);
    bool PotPresent = (PotPresence != COOKER_POT_PRESENCE_ABSENT);

    if (ErrorBlocking || !CookerOn) {
        CookerState_Off();
        SessionPaused = false;
    } else if (!PotPresent) {
        // pause — session stays active, accumulation skipped this frame
        SessionPaused = true;
    } else {
        if (SessionPaused) {
            TIME_T Now;
            UserRtc_GetRtcTime(&Now);
            CookingSession_ResumeFromPause(&Now);
            SessionPaused = false;
        }
        if (!IsCookerVoltageOK()) {
            PowerLevel = COOKER_POWER_NO_POWER;
        }
        CookerState_On(&PowerLevel, &CurrentActualPower);
    }

    CurrentPowerState = PowerState;
    CurrentPotPresence = PotPresence;
    CurrentPowerLevel = PowerLevel;
}
```

Note: the voltage-fault routing (zero-power) is preserved — that's a Develop-specific enhancement that should stay. The pause behaviour layers on top: pot-absent doesn't call `CookerState_Off` anymore.

### Keypad — ENERGY_INFO_BUTTON

The Prescan/CHK-Bring-Up keypad has a dedicated `ENERGY_INFO_BUTTON` enum member in `CommandKeys_e` (`Keypad.h`) and is mapped on the X12A capacitive-touch IC at bitmask `0x0020` for I2C address 2 (in `X12ACapTouch.c`). **Verify these exist on Develop** before implementing. If missing, add them first.

### Keypad race (independent open bug, not in scope)

The Prescan/CHK-Bring-Up branches have a known race in `Keypad_Process` (the two-I²C-address `GetKeysFromDriver` sequence opens a phantom NOT_PRESSED window, allowing the keypad ISR to emit duplicate SHORT_PRESS events). This bug almost certainly exists on Develop too. **Not in scope for the usage-feature port** — flag separately. Symptom: a single physical INFO press feels like a double-press, with the screen briefly toggling through two states. If you see this during bench testing, it's a separate fix.

---

## 5. Suggested implementation order

If implementing in phases:

1. **Phase A — Locked-timer integrity.** Apply the three locked-timer fixes (3d69214, 0a4fdcd, bc3a8d3). Verify lock LED behaves correctly across timer expiry and timed cooking. ~half a day. *Stable checkpoint.*

2. **Phase B — End-of-session UX + E0 auto-shutdown.** Apply eac762f, a70a5aa, 5f37234, bf54aec. Verify the USED → kWh → STANDBY sequence plays on error shutdown and E0 30 s auto-shutdown. ~half a day. *Stable checkpoint.*

3. **Phase C — E0 pause.** Apply 083b57d (the critical session-preservation change). Bench-test the lift-return-lift-OFF sequence and confirm the shutdown screen shows the full cook kWh. **Most critical bench test of the whole port.** ~half a day. *Stable checkpoint.*

4. **Phase D — Info button.** Apply 6862220 + b1f620e + fb9c6e8 + 25df579 together (they're tightly coupled). Bench-test all the INFO press scenarios from the bench-test list (section 6). ~1-2 days. *Final.*

If preferred, the implementer can land it all in a single PR — but the four-phase split gives clean intermediate stable points for bench-testing.

---

## 6. Acceptance criteria / bench tests

Each test below must pass before the PR is merged. Reference: the original 22-test bench plan that the source branches verified against.

### Locked-timer integrity (Phase A)

- A1. Set a 0:05 timer, lock the cooker (LOCK long). Wait for timer to expire. Confirm LOCK_LED turns off as part of the shutdown sequence.
- A2. During TIME_SETTING_SCREEN (mid-edit), long-press LOCK. Confirm the timer flash plays, the timer starts, and the cooker ends up in CHILD_LOCK_SHOW_POWER. The lock icon should appear instantly on the long-press.
- A3. Locked timed cooking: colon should blink at 1 Hz.

### End-of-session usage display (Phase B)

- B1. Cook for 1 min at level 5. Press OFF. Screen shows "USED" → final kWh → STANDBY.
- B2. Cook, lift pot mid-cook (E0 fires). Wait 30 s without replacing. Screen shows USED → kWh → STANDBY (auto-shutdown path).
- B3. During TIME_SETTING_SCREEN, press PWR+. Power level adjusts; the 2 s "stop adjusting" timeout resets.

### E0 pause (Phase C — the critical one)

- C1. Cook at level 5 for 1 min. Lift pot. After E0 displays, replace pot. Confirm cooking resumes at level 5.
- C2. Same as C1, but lift the pot multiple times during a longer cook. Manually press OFF after all the lifts/returns. The shutdown screen kWh must show the **full cook total**, not just the last segment.
- C3. Lift pot, wait for 30 s E0 auto-shutdown. Shutdown screen plays USED → final kWh → STANDBY.

### Info button (Phase D)

- D1. STANDBY → INFO short-press. Screen: USED (3 s) → last-session kWh (10 s) → STANDBY.
- D2. Cook, INFO short-press. Realtime kWh appears, updates ~1 Hz. Persists indefinitely. Press INFO again → back to USER_COOKING.
- D3. Cook with timer, INFO short-press. Realtime + timer + blinking colon. Persists.
- D4. From realtime, press TIMR+. Timer digits change in place; kWh keeps updating; layout unchanged. Wait 3 s. Timer digits flash 5×, kWh stays solid. Then returns to plain timed-realtime with timer counting down from the new value.
- D5. From timed realtime, LOCK long-press. LOCK_ICON appears; screen stays on realtime view. Other keys go inert. LOCK long again → unlocks back to realtime.
- D6. E0 (pot lift) during realtime. Error displays. Replace pot — screen returns to **realtime view** (not STANDBY).
- D7. E0 30-second auto-shutdown from locked realtime. LOCK_LED turns off during the transition.
- D8. Auto-off warning (3h45m) from locked realtime — screen transitions to `CHILD_LOCK_SHOW_POWER` with timer counting down, lock stays engaged.

---

## 7. Files that will be modified

Approximate (final implementation will determine exact lines):

```
ExternalPeripherals/InductionCooker/Inc/CookingSession.h   (add ResumeFromPause prototype)
ExternalPeripherals/InductionCooker/Src/CookingSession.c   (add ResumeFromPause)
ExternalPeripherals/InductionCooker/Src/IndCooker.c        (PowerBoardStatusCb pause logic)
ProductFeatures/UI/Inc/UIPresenter.h                       (new state enum values + IsRealtimeLocked prototype)
ProductFeatures/UI/Src/UIPresenter.c                       (new states + helpers + edit-tracking + tick logic)
ProductFeatures/UI/Inc/UIView.h                            (new view enum values)
ProductFeatures/UI/Src/UIView.c                            (new view functions + tick handler + RenderKwhFloat helper)
```

No new modules required. Everything lives in existing files.

---

## 8. References

- **kWh accounting end-to-end:** the `~/.claude/projects/.../memory/features.md` entry titled "Energy (kWh) accounting — end-to-end reference" covers the integrator, state pattern, persistence, display paths, and refresh cadences. Read it before implementing — it's the single source of truth for how kWh flows through the firmware.
- **Lab test for kWh accuracy:** `ExternalPeripherals/InductionCooker/Docs/KwhAccuracyComparison.md` documents the modelled-vs-measured kWh comparison test. Separate from the usage-feature port.
- **Source-branch commit history:** the 12 commits are on the `Prescan` branch (commits `3d69214` through `25df579`, chronological order) and have been cherry-picked to `CHK-Bring-Up` too. `git show <sha>` on either branch for full diffs.
- **UI state transitions (broader context):** `ProductFeatures/UI/Docs/UITransition.md` (existing file in the repo) for the wider state-machine overview.

---

## 9. Change log

- 2026-06-05 — initial draft. Spec captures the 12 source commits and the Develop integration considerations identified during a failed cherry-pick attempt. Implementation pending — file will be updated as the port proceeds.
