# Features

## Info Button (MODE key) — Last-Session kWh Recall from Standby
- **Product:** G4
- **Date:** 2026-05-13
- **Branch:** Rendering-Grouped-Power-Levels (commit f046d2b)
- **Source files:**
  - `ExternalPeripherals/Keypad/Inc/Keypad.h` — `MODE_BUTTON` enum
  - `ExternalPeripherals/Keypad/Src/X12ACapTouch.c` — physical-key bitmask → MODE_BUTTON
  - `ExternalPeripherals/Keypad/Src/KeypadEvents.c` — short/long press classification
  - `ProductFeatures/UI/Src/UIPresenter.c` — state machine (STANDBY → INFO_USED_LABEL → INFO_USED_VALUE → STANDBY)
  - `ProductFeatures/UI/Src/UIView.c` — `ShowStringUsed()`, `ShowSessionPower()`
  - `ExternalPeripherals/Flash/Src/SessionAggr.c` — `SessionAggr_GetLatestSessionPowerUsage()`
- **Purpose:** Lets the user recall how much energy the last cooking session consumed without powering the cooker back on. Surface for paygo customers to check per-session kWh usage at a glance.

### Behaviour
- Active only from `UI_PRESENTER_STATE_STANDBY` (blinking-dashes screen). MODE is `NO_ACTION_ON_KEYPRESS` in every other state.
- Short press of `MODE_BUTTON` in STANDBY → `UI_PRESENTER_STATE_INFO_USED_LABEL`.
- The two info states are dedicated to this feature; no other path enters them.

### State machine (UIPresenter.c)
```
STANDBY ──(MODE short press)──▶ INFO_USED_LABEL
                                   │  view  = UI_VIEW_DISPLAY_STRING_USED → "USED"
                                   │  timeout = INFO_LABEL_TIMEOUT (9 ticks)
                                   │  MODE or ON/OFF (short/long) → STANDBY (immediate)
                                   ▼ (timeout)
                                INFO_USED_VALUE
                                   │  view  = UI_VIEW_DISPLAY_SESSION_POWER → last kWh + KWH icon
                                   │  timeout = INFO_VALUE_TIMEOUT (9 ticks)
                                   │  MODE or ON/OFF (short/long) → STANDBY (immediate)
                                   ▼ (timeout)
                                STANDBY
```

### Hardware mapping (X12A cap-touch IC, I2C read at `0x40`)
- BaoLaiYa board: bit `0x2000` → MODE_BUTTON
- NewShine board: bit `0x0080` → MODE_BUTTON
- Selected via compile-time `#ifdef TOUCHPAD_BAOLAIYA` in `X12ACapTouch.c`.

### Data source
`ShowSessionPower()` calls `SessionAggr_GetLatestSessionPowerUsage()`, which returns
`SessionAggrFlashStoreRAMCpy.SessionAggrData.LatestSessionData.CookerPower` — a RAM mirror of the
last-session kWh value persisted to SPI flash on `CookingSession_Stop()`. The Info value therefore
survives a reboot. If no session has ever been recorded the displayed value is 0.00.

### Rendering
- "USED" label: `ShowStringUsed()` → `Display_ShowString("USED")` (UIView.c:508).
- kWh value: `ShowSessionPower()` (UIView.c:379) — rounds up to 2 decimals via `ceilf(v * 100) / 100`,
  formats as `"%2d%02d"`, overlays `KWH_ICON` and the decimal dot, pushes the OR'd buffer to the VK16K33.

### Known oddity / open question
```c
#define INFO_LABEL_TIMEOUT   9   // comment: "5 seconds effective (fires on tick N+1)"
#define INFO_VALUE_TIMEOUT   9   // comment: "10 seconds effective (fires on tick N+1)"
```
Both macros are `9`, but their comments claim different durations. `UIPresenter_Tick()` decrements
`StateTimeOutCounter` at a single fixed rate, so in practice both screens display for the same time.
Either the LABEL or the VALUE macro has the wrong numeric value, or the comments are stale. Worth
stopwatch-checking on the bench and either correcting the macro or pruning the comment.

- **Verification:** Documented from source on `Rendering-Grouped-Power-Levels` (HEAD `f046d2b`).
  Not bench-tested in this session.
- **Status:** Documented (not bench-verified)