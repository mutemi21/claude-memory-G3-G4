# Timed Cooking Logic — Implementation Guide

This document describes the complete timed cooking system from the STM32_FIRMWARE_G4 codebase. It is intended as a reference for implementing equivalent functionality on a **different product with a different UI**. The architecture, data flows, state transitions, and critical gotchas are all documented here so you can adapt the logic without reading the original source.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [System Tick and Main Loop](#2-system-tick-and-main-loop)
3. [Button / Key Event System](#3-button--key-event-system)
4. [UI State Machine Design](#4-ui-state-machine-design)
5. [Timed Cooking States — Full Detail](#5-timed-cooking-states--full-detail)
6. [Timer Engine (IndCooker Timer)](#6-timer-engine-indcooker-timer)
7. [Power Control and Callback Chain](#7-power-control-and-callback-chain)
8. [Session Tracking (CookerState / CookingSession / SessionAggr)](#8-session-tracking)
9. [Auto-Off Safety System](#9-auto-off-safety-system)
10. [Shutdown Sequence and Fan Cooldown](#10-shutdown-sequence-and-fan-cooldown)
11. [Complete End-to-End Flow](#11-complete-end-to-end-flow)
12. [Critical Gotchas](#12-critical-gotchas)
13. [Constants and Tunables](#13-constants-and-tunables)
14. [API Reference Summary](#14-api-reference-summary)

---

## 1. Architecture Overview

The firmware follows an **MVP (Model-View-Presenter)** pattern:

```
┌────────────┐     ┌────────────┐     ┌────────────┐
│ UIPresenter │────>│  UIModel   │────>│  UIView    │
│ (State      │     │ (Data      │     │ (Display   │
│  Machine)   │<────│  Store)    │     │  Rendering)│
└──────┬──────┘     └──────┬─────┘     └────────────┘
       │                   │
       │    ┌──────────────┴──────────────┐
       │    │        IndCooker            │
       │    │  (Timer + Power Control)    │
       │    └──────────────┬──────────────┘
       │                   │
       │    ┌──────────────┴──────────────┐
       │    │        PowerBoard           │
       │    │  (Hardware Abstraction)     │
       │    │  Dispatches to Driver       │
       │    └──────────────┬──────────────┘
       │                   │
       │    ┌──────────────┴──────────────┐
       │    │      CookerState            │
       │    │  (On/Off state pattern)     │
       │    │  ┌─> CookingSession         │
       │    │  │   (Power accumulation)   │
       │    │  │   ┌─> SessionAggr        │
       │    │  │   │   (Flash persistence)│
       │    └──┴───┴──────────────────────┘
       │
  Keypad Events (via callbacks)
```

**Separation of concerns:**
- **UIPresenter** — State machine. Handles user input, drives state transitions, calls UIModel/IndCooker APIs. Knows nothing about display hardware.
- **UIModel** — Pure data store. Holds power level, timer hours/minutes, error status. Provides getters/setters. Thin wrapper around IndCooker calls.
- **UIView** — Rendering only. Given a view enum + model data, renders to the display. Manages animations (blink, flash). Knows nothing about state logic.
- **IndCooker** — Timer countdown engine + power control orchestrator. Owns the countdown variable. Coordinates with PowerBoard.
- **PowerBoard** — Hardware abstraction. Dispatches to a specific driver (e.g., HighwayDriver). Fires status callbacks on power changes.
- **CookerState** — Two-state (On/Off) pattern. Starts/stops CookingSessions in response to PowerBoard callbacks.
- **CookingSession** — Accumulates watt-seconds during cooking. Converts to kWh on stop. Backs up to RTC every second for crash recovery.
- **SessionAggr** — Persists daily/monthly power totals to SPI flash with CRC32.

---

## 2. System Tick and Main Loop

### Tick Generation

A 1ms SysTick interrupt (from STM32 HAL) drives `UserSystick`. Every 1000 ticks (1 second), a callback sets `TickFlag = true` in `main.c`.

### Main Loop Structure

```c
while (1) {
    // Fast path — runs every loop iteration (no delay)
    IndCooker_Process();     // Check PowerBoard status
    UIPresenter_Process();   // Read key events, update state machine, refresh display
    UIView_Process();        // Run display animations (blink, flash timers)
    Keypad_Process();        // I2C keypad communication

    // Slow path — runs once per second
    if (TickFlag) {
        TickFlag = false;
        IndCooker_Tick();       // Decrement cooking timer, fire expiry
        UIPresenter_Tick();     // Decrement state timeout, handle auto-off, handle timer expiry detection
        Led_Tick();             // LED animations
    }
}
```

**Key insight:** The "tick" is 1 second. All timeout values in the state table are in **seconds**. UIView animations (blink, flash) use their own ms-level counters inside `UIView_Process()`.

---

## 3. Button / Key Event System

### Physical Buttons

| Button ID | Enum Name | Purpose |
|-----------|-----------|---------|
| 0 | `BT_BUTTON` | Bluetooth |
| 1 | `PWR_PLUS_BUTTON` | Power increment (+) |
| 2 | `PWR_MINUS_BUTTON` | Power decrement (-) |
| 3 | `TIMR_PLUS_BUTTON` | Timer increment (+) |
| 4 | `TIMR_MINUS_BUTTON` | Timer decrement (-) |
| 5 | `MODE_BUTTON` | Mode / Info |
| 6 | `ON_OFF_BUTTON` | Power on/off toggle |
| 7 | `PAY_BUTTON` | Pay |

### Press Duration Detection

- **Short press:** Button released before 1000ms hold time
- **Long press:** Button held >= 1000ms; fires once, then auto-repeats while held
- Sampling rate: 5ms (200 samples = 1000ms threshold)

### Event Delivery Pipeline

```
GPIO/I2C Matrix → Keypad_GetKeypadMatrixState()
    → KeypadEvents_MsTick() [every 5ms]
        → DeterminePressDuration() → SHORT_PRESS or LONG_PRESS
            → Callback → UIView_UpdateWithKeyPadEvent(Event)
                → Stored as CurrentKeyPadEvent in UIView
                    → UIPresenter_Process() reads via UIView_GetKeyPadEvent()
                        → State table lookup → action execution
```

### Key Event Structure

```c
typedef struct KeyPadEvent {
    KeyType_e KeyType;           // KEY_TYPE_COMMAND or NO_KEY_TYPE
    union {
        KeypadControlEvent_t CommandKeyEvent;   // .Key + .KeyPressDuration
        NumericalKeys_e NumericalKeyPressed;
    } KeyInputEvents;
} KeyPadEvent_t;
```

---

## 4. UI State Machine Design

### State Table Structure

Each state is defined by a struct with:

```c
typedef struct {
    UIPresenterStateID_e StateID;          // This state's ID
    UIView_e             StateUIView;      // Which view to render
    KeyPressAction_t     ActionsOnKeyPress[NUM_COMMAND_KEYS]; // Per-button actions
    TimeOutAction_t      StateTimeOut;     // What happens on timeout
    uint32_t             TimeOutValue;     // Seconds until timeout fires
} UIPresenterStateInfo_t;
```

### Per-Button Action Structure

```c
typedef struct {
    UIPresenterStateID_e LongPressNextStateID;     // State to go to on long press
    void (*ActionFunctionLongPress)(void);          // Function to call on long press
    UIPresenterStateID_e ShortPressNextStateID;     // State to go to on short press
    void (*ActionFunctionShortPress)(void);         // Function to call on short press
} KeyPressAction_t;
```

### Timeout Action Structure

```c
typedef struct {
    UIPresenterStateID_e NextStateID;       // State to transition to on timeout
    void (*ActionFunction)(void);           // Function to call on timeout
} TimeOutAction_t;
```

### Processing Logic

**UIPresenter_Process() (every main loop iteration):**
1. Read key event via `UIView_GetKeyPadEvent()`
2. If key pressed:
   - Cancel auto-off warning if active
   - Reset auto-off counter to 0
   - Look up `ActionsOnKeyPress[KeyID]` in current state
   - Set `CurrentState = ShortPressNextStateID` (or LongPressNextStateID) — **BEFORE** calling action function
   - Call the action function (which may override the state)
   - Reset `StateTimeOutCounter` to new state's `TimeOutValue`
3. Read UIModel data
4. Pass (view enum, model data) to UIView for rendering

**UIPresenter_Tick() (every 1 second):**
1. Decrement `StateTimeOutCounter`
2. If counter reaches 0 → call `ProcessStateTimeOut()`:
   - Set `CurrentState = TimeOut.NextStateID` (if not `NO_SUPPORTED_STATE`)
   - Call `TimeOut.ActionFunction` (if not NULL)
   - Reset counter for new state
3. Call `HandleTimerExpiry()` — detect cooking timer expiry
4. Call `HandleAutoOff()` — increment auto-off counter, check thresholds
5. Decrement `ShutdownFanCountdown` if > 0

### Special Value: NO_SUPPORTED_STATE

When `NextStateID = NO_SUPPORTED_STATE`, the state machine does NOT change state. This lets the action function dynamically decide the next state by writing to `CurrentUIPresenterState` directly.

### CRITICAL: State Set BEFORE Action Function

The state table's `ShortPressNextStateID` is applied **before** the action function runs. If the action function also sets state, it overwrites the table value. To let the action function have full control, set `ShortPressNextStateID = NO_SUPPORTED_STATE`.

### CRITICAL: Timeout Counter Reset After Key Events

After any key event, `UIPresenter_Process()` resets `StateTimeOutCounter` to the **new** state's timeout value. This means any timeout counter set by an action function will be overwritten.

---

## 5. Timed Cooking States — Full Detail

### State Flow Diagram

```
                    ┌─────────────────────┐
                    │   USER_COOKING      │ (no timer active)
                    │   View: power bars  │
                    │   Timeout: none     │
                    └─────────┬───────────┘
                              │ TIMR_PLUS or TIMR_MINUS (short press)
                              ▼
                    ┌─────────────────────┐
                    │ TIME_SETTING_IDLE   │ (blinking time display)
                    │ View: blink HH:MM  │
                    │ Timeout: 4 seconds  │──timeout──→ back to USER_COOKING
                    └─────────┬───────────┘
                              │ TIMR_PLUS or TIMR_MINUS (short press within 4s)
                              ▼
                    ┌─────────────────────┐
                    │ TIME_SETTING_SCREEN │ (solid time display, user editing)
                    │ View: solid HH:MM  │
                    │ Timeout: 2 seconds  │──timeout──→ SetInductionCookerTimerFromUIModel()
                    └─────────┬───────────┘
                              │ (timeout fires, timer value > 0, IndCooker accepts)
                              ▼
                    ┌─────────────────────┐
                    │ TIMER_ACCEPT_FLASH  │ (5 flashes confirming timer set)
                    │ View: flash on/off  │
                    │ Timeout: 7 seconds  │──timeout──→ USER_COOKING_WITH_TIMER
                    └─────────┬───────────┘
                              │ (auto-transitions after flash animation)
                              ▼
                    ┌─────────────────────────────┐
                    │ USER_COOKING_WITH_TIMER     │ (active countdown display)
                    │ View: HH:MM + blinking colon│
                    │ Timeout: default (1 tick)   │
                    └─────────┬───────────────────┘
                              │ Timer reaches 0 (detected by HandleTimerExpiry)
                              ▼
                    ┌─────────────────────┐
                    │ END_OF_SET_COOK_TIME│ (3 beeps alerting user)
                    │ View: beeping       │
                    │ Timeout: 15 seconds │──timeout──→ SHUTDOWN_USED_LABEL
                    └─────────────────────┘
                              │
                              ▼
                    (Shutdown sequence — see Section 10)
```

### State Details

#### TIME_SETTING_SCREEN_IDLE (Blinking Entry)

- **Purpose:** User just pressed timer button from cooking state. Show blinking time to indicate edit mode.
- **Display:** HH:MM with blinking colon (500ms on/off via `ShowTimeLeftPeriodically()`)
- **Timeout:** 4 seconds → return to `USER_COOKING` (user didn't commit)
- **Key actions:**
  - TIMR_PLUS short: `UIModel_IncrementTimerSetting()` (+1 min), transition to `TIME_SETTING_SCREEN`
  - TIMR_PLUS long: `UIModel_IncrementTimerSettingFast()` (+15 min), stay
  - TIMR_MINUS short: `UIModel_DecrementTimerSetting()` (-1 min), transition to `TIME_SETTING_SCREEN`
  - TIMR_MINUS long: `UIModel_DecrementTimerSettingFast()` (-15 min), stay
  - ON_OFF: `HandleShutdownSequence()` → shutdown

#### TIME_SETTING_SCREEN (Active Editing)

- **Purpose:** User is actively adjusting the timer value.
- **Display:** HH:MM with solid colon (static display via `RenderTimeSettingScreen()`)
- **Timeout:** 2 seconds → `SetInductionCookerTimerFromUIModel()` (apply timer to hardware)
- **Key actions:** Same as IDLE but transitions stay in `TIME_SETTING_SCREEN`
- **Critical:** Each key press resets the 2-second inactivity timeout. Timer is only applied when the user stops pressing buttons for 2 seconds.

#### TIMER_ACCEPT_FLASH (Visual Confirmation)

- **Purpose:** Confirm that timer has been accepted with 5 visual flashes.
- **Display:** Alternates between full timer screen (ON, 500ms) and power-only screen (OFF, 500ms), 5 times.
- **Timeout:** 7 seconds (5 flashes + buffer) → auto-transition to `USER_COOKING_WITH_TIMER`
- **Key actions:**
  - TIMR_PLUS: `IncrementTimerAndReflash()` — increment timer, re-apply to hardware, restart flash
  - TIMR_MINUS: `DecrementTimerAndReflash()` — decrement, re-apply, restart flash
- **Animation mechanism:** Module-level statics in UIView: `TimerFlashCounter`, `TimerFlashCount`, `TimerFlashOn`, `TimerFlashDone`. When `TimerFlashDone = true`, calls `UIPresenter_ForceStateTimeout()` to immediately trigger state transition.

#### USER_COOKING_WITH_TIMER (Active Countdown)

- **Purpose:** Cooking is in progress with the timer counting down.
- **Display:** HH:MM with blinking colon (1 second period), timer icon, power level bars
- **Timeout:** Default (1 tick) — state persists indefinitely; transition driven by timer expiry detection
- **Key actions:**
  - PWR_PLUS short: `UIModel_IncrementPowerSetting()` (adjust power mid-cook)
  - PWR_MINUS short: `UIModel_DecrementPowerSetting()`
  - TIMR_PLUS: `UIModel_IncrementTimerSetting()` → transition to `TIME_SETTING_SCREEN` (edit timer mid-cook)
  - TIMR_MINUS: `UIModel_DecrementTimerSetting()` → transition to `TIME_SETTING_SCREEN`
  - ON_OFF: `HandleShutdownSequence()` → manual shutdown
- **Timer expiry detection:** `HandleTimerExpiry()` runs every tick. When `IndCooker_IsTimerRunning()` returns false, it triggers:
  1. `UIModel_StopSession()` — save kWh to flash
  2. `UIModel_PowerOffWithFan()` — enable fan cooldown
  3. `ShutdownFanCountdown = 60`
  4. Transition to `END_OF_SET_COOK_TIME`

#### END_OF_SET_COOK_TIME (Beeping Alert)

- **Purpose:** Alert user that their set cooking time has elapsed.
- **Display:** 3 beeps (buzzer ON 300ms, OFF 600ms between beeps)
- **Timeout:** 15 seconds → transition to `SHUTDOWN_USED_LABEL`
- **Key actions:**
  - ON_OFF: `SkipToShutdownUsage()` — skip beeps and go to usage display immediately
- **Beep mechanism:** `HandleEndOfCookingBeeps()` in UIView. Uses module-level statics for beep count and timing. After 3 beeps, calls `UIPresenter_ForceStateTimeout()`.

---

## 6. Timer Engine (IndCooker Timer)

### Data

```c
static uint32_t TimerCountDownVal = 0;  // Remaining seconds
static bool TimerRunning = false;       // Is timer active?
```

### API

| Function | Signature | Behavior |
|----------|-----------|----------|
| **SetTimer** | `bool IndCooker_SetTimer(uint16_t Duration)` | Set countdown to `Duration` seconds, set `TimerRunning = true`. Returns false if cooker is off or locked. |
| **StopTimer** | `void IndCooker_StopTimer(void)` | Set `TimerRunning = false`, `TimerCountDownVal = 0`. Does NOT power off. |
| **IsTimerRunning** | `bool IndCooker_IsTimerRunning(void)` | Returns `TimerRunning` flag. |
| **GetRemainingTimerTime** | `uint32_t IndCooker_GetRemainingTimerTime(void)` | Returns `TimerCountDownVal` in seconds. |

### Tick Countdown (called every 1 second)

```c
void IndCooker_Tick(void) {
    PowerBoard_Tick();           // Hardware communication tick
    if (TimerRunning) {
        if (TimerCountDownVal > 0) {
            TimerCountDownVal--;
            if (TimerCountDownVal == 0) {
                IndCooker_PowerOff();   // Power off the cooker hardware
                TimerRunning = false;
            }
        }
    }
}
```

### Timer Value Conversion

UIModel stores timer as separate hours and minutes. Conversion to seconds for IndCooker:

```c
uint16_t TimerValue = (SetCookTimeHours * 3600) + (SetCookTimeMins * 60);
```

### Timer Increment/Decrement (UIModel Layer)

| Function | Step | Min | Max |
|----------|------|-----|-----|
| `UIModel_IncrementTimerSetting()` | +1 minute | — | 4:00 (240 min) |
| `UIModel_DecrementTimerSetting()` | -1 minute | 0:00 | — |
| `UIModel_IncrementTimerSettingFast()` | +15 minutes | — | 4:00 |
| `UIModel_DecrementTimerSettingFast()` | -15 minutes | 0:00 | — |

Increment wraps minutes at 60 → increments hours. Decrement borrows from hours at 0 minutes. Capped at 4:00 max, 0:00 min.

### Remaining Time Display Update

`UIModel_GetRemainingCookTime()` reads `IndCooker_GetRemainingTimerTime()` (in seconds) and converts back to hours:minutes for the UIModel data, using **ceiling division** so that 1-59 remaining seconds display as "0:01" (never shows "0:00" until truly expired).

---

## 7. Power Control and Callback Chain

### Power Levels

The system uses 13 discrete power levels:

| Enum | Watts | Notes |
|------|-------|-------|
| `COOKER_POWER_NO_POWER` | 0W | Idle / off |
| `COOKER_POWER_0W` | 200W | Starting level |
| `COOKER_POWER_200W` – `COOKER_POWER_2000W` | 200W–2000W | 200W steps |
| `COOKER_POWER_NO_POWER_FAN_ON` | 0W | Fan-only cooldown mode |

### SetLevel Trigger Chain

```
UIPresenter action (e.g., PWR_PLUS short press)
    → UIModel_IncrementPowerSetting()
        → Updates UIModelData.PowerSetting
        → IndCooker_SetLevel(newLevel)
            → PowerBoard_SetLevel(newLevel)    [dispatch]
                → HighwayDriver_SetLevel(newLevel) [hardware]
                    → Sets CurrentPowerLevel = newLevel
                    → **IMMEDIATELY** fires StatusCallback(PowerState, PotPresence, PowerLevel)
                        → PowerBoardStatusCb() in IndCooker.c
                            → CookerState_On(&PowerLevel, &ActualPower) [starts/updates session]
```

**CRITICAL:** `SetLevel()` fires the status callback **synchronously and immediately**. This starts or updates the cooking session.

### PowerOff vs PowerOffWithFan

| Function | Effect | Callback? | Use Case |
|----------|--------|-----------|----------|
| `PowerBoard_PowerOff()` | Full power off, no fan | NO (Highway) | Short sessions < 60s |
| `PowerBoard_PowerOffWithFan()` | Fan only, no heating | NO | Long sessions >= 60s, timer expiry |

**CRITICAL:** Neither PowerOff nor PowerOffWithFan triggers the status callback on Highway driver. The session stop is handled explicitly by the UI layer calling `UIModel_StopSession()` before issuing the power off.

---

## 8. Session Tracking

### Three-Layer Architecture

```
CookerState (On/Off pattern)
    → CookingSession (power accumulation)
        → SessionAggr (flash persistence)
```

### CookerState — On/Off State Machine

Two states with action functions:

| State | On CookerState_On() | On CookerState_Off() | On Tick Timeout |
|-------|---------------------|----------------------|-----------------|
| **Off** | `StartSession()` → go to On | (no-op, already off) | (no-op) |
| **On** | `UpdateSession()` (update power) | `StopSession()` → go to Off | `BackupSession()` every 1s |

### CookingSession — Energy Accumulation

```c
typedef struct CookingSession {
    int32_t isActive;
    TIME_T Start;                        // Session start timestamp
    TIME_T End;                          // Last update timestamp
    CookerPower_e CurrentUserPowerSetting;
    uint32_t ActualPower;                // Current watts
    uint32_t AccumulatedPower;           // Running total in watt-seconds
} CookingSession_t;
```

**Lifecycle:**
1. `CookingSession_Start()` — Initialize: `isActive=true`, `Start=now`, `End=now`, `AccumulatedPower=0`
2. `CookingSession_Update()` — Every second: `AccumulatedPower += ActualPower × timeDelta`; `End=now`
3. `CookingSession_Stop()` — Finalize:
   - Call `Update()` one last time (captures final second)
   - Convert to kWh: `AccumulatedPower / 3600 / 1000`
   - Append to flash buffer via `FlashData_SessionDataBufferAppend()`
   - Update aggregation via `SessionAggr_UpdatePowerUsage(kWh, &startTime)`
   - Clear RTC backup
   - Reset session to inactive

**CRITICAL:** `CookingSession_Stop()` resets the session data. You must call `CookingSession_GetDurationInSeconds()` **BEFORE** stopping the session.

### CookingSession Backup (Crash Recovery)

Every second during cooking, `CookerState_BackupTimeout()` calls `BackupSession()`:
- Calls `CookingSession_Update()` to accumulate latest power
- Writes snapshot to RTC backup memory
- On startup, `CookingSession_Init()` checks for non-zero backup and recovers

### SessionAggr — Flash Persistence

```c
typedef struct SessionAggrData {
    float DailyPowerUsage;               // kWh today
    float MonthlyPowerUsage;             // kWh this month
    SessionData_t LatestSessionData;     // Last session info
    uint32_t NextDayMidnight;            // UTC timestamp
    uint32_t NextMonthFirstDayMidnight;  // UTC timestamp
} SessionAggrData_t;
```

**On `SessionAggr_UpdatePowerUsage(power_kWh, &startTime)`:**
1. Check if session start crosses day boundary → reset daily
2. Check if session start crosses month boundary → reset monthly
3. Add power to daily and monthly totals
4. Compute CRC32, erase flash sector, write updated data

---

## 9. Auto-Off Safety System

Prevents indefinite cooking. If the user cooks without setting a timer, the system enforces a **4-hour maximum**.

### Constants

```c
#define AUTO_OFF_TOTAL_SECONDS      14400  // 4 hours
#define AUTO_OFF_WARNING_SECONDS    13500  // 3 hours 45 minutes
```

### Mechanism

`HandleAutoOff()` runs every tick (1 second) in `UIPresenter_Tick()`:

```
AutoOffCounter increments every second while in a cooking state

At 3:45 (13500s):
    → 2 warning beeps
    → Create "synthetic" 15-minute timer: IndCooker_SetTimer(remainingSeconds)
    → Transition to USER_COOKING_WITH_TIMER (shows countdown)
    → Set AutoOffWarningActive = true

At 4:00 (14400s):
    → HandleShutdownSequence() — forced shutdown
```

### User Override

Any key press during the warning phase:
- Resets `AutoOffCounter = 0` (restarts the 4-hour window)
- Sets `AutoOffWarningActive = false`
- Stops the synthetic timer: `IndCooker_StopTimer()`
- Returns to `USER_COOKING` state

### User-Set Timer Supersedes Auto-Off

If the user has set their own timer and `AutoOffWarningActive` is false, the auto-off counter resets to 0. The user's timer takes priority.

---

## 10. Shutdown Sequence and Fan Cooldown

### HandleShutdownSequence() — The Universal Shutdown Entry Point

Called by: ON_OFF button press, auto-off trigger, or timer expiry.

```c
static void HandleShutdownSequence(void) {
    int32_t duration = UIModel_GetSessionDuration();  // MUST read BEFORE stopping
    UIModel_StopSession();       // Save kWh to flash
    UIModel_ResetTimerSetting(); // Clear HH:MM display
    IndCooker_StopTimer();       // Stop hardware timer (prevents race condition)

    if (duration >= 60) {
        UIModel_PowerOffWithFan();    // Fan runs for 60s cooldown
        ShutdownFanCountdown = 60;
    } else {
        UIModel_PowerOffPowerBoard(); // Immediate full off
    }

    CurrentUIPresenterState = UI_PRESENTER_STATE_SHUTDOWN_USED_LABEL;
}
```

### Timer-Expiry Shutdown (Slightly Different)

When timer expires (detected by `HandleTimerExpiry()`), the sequence is:
1. `UIModel_StopSession()` — save kWh
2. `UIModel_PowerOffWithFan()` — fan cooldown (always, since timed cook implies long session)
3. `ShutdownFanCountdown = 60`
4. Transition to `END_OF_SET_COOK_TIME` (beeping) — NOT directly to USED_LABEL

### Shutdown Display Sequence

```
END_OF_SET_COOK_TIME (15s, 3 beeps)   ← Only for timer expiry
    ↓
SHUTDOWN_USED_LABEL (5s)              ← Shows "USED" label
    ↓
SHUTDOWN_SESSION_USAGE (10s)          ← Shows session kWh value
    ↓
SHUTDOWN_OFF (1-2s)                   ← Shows "OFF"
    ↓
STANDBY                               ← Blinking dashes, waiting for ON_OFF press
```

### Fan Cooldown (ShutdownFanCountdown)

- Set to 60 when `PowerOffWithFan()` is called
- Decremented every tick (1 second) in `UIPresenter_Tick()`
- When reaches 0: calls `UIModel_PowerOffPowerBoard()` to fully cut power
- Runs concurrently with the shutdown display sequence (fan keeps cooling while user sees usage info)
- If countdown finishes during a cooking state (shouldn't happen), it won't fire

---

## 11. Complete End-to-End Flow

### Scenario: User powers on, sets power to 5, sets 30-minute timer

```
USER ACTION: Press ON/OFF
    → PowerBoard fires StatusCallback(ON, PRESENT, LEVEL_5)
    → CookerState_On() → StartSession()
    → CookingSession_Start(now, LEVEL_5, 3000W)
    → UI state: USER_COOKING, display shows power level 5

USER ACTION: Press TIMR_PLUS
    → UI state: TIME_SETTING_SCREEN_IDLE
    → Display: blinking "0:00"
    → UIModel_IncrementTimerSetting() → SetCookTimeMins = 1

USER ACTION: Long-press TIMR_PLUS (multiple times to reach 0:30)
    → UI state: TIME_SETTING_SCREEN
    → Display: solid "0:30"
    → Each long press: +15 minutes

USER STOPS PRESSING (2 second inactivity)
    → Timeout fires: SetInductionCookerTimerFromUIModel()
    → Converts: 0 hours × 3600 + 30 mins × 60 = 1800 seconds
    → IndCooker_SetTimer(1800) → TimerCountDownVal=1800, TimerRunning=true
    → UI state: TIMER_ACCEPT_FLASH
    → Display: 5 flashes (timer screen ON 500ms / power-only OFF 500ms)

FLASH ANIMATION COMPLETES (7 seconds)
    → UI state: USER_COOKING_WITH_TIMER
    → Display: "0:30" with blinking colon every 1 second

EVERY SECOND DURING COOKING:
    IndCooker_Tick():
        → TimerCountDownVal-- (1799, 1798, ... 1, 0)
        → CookerState_BackupTimeout() → BackupSession()
            → CookingSession_Update(): AccumulatedPower += 3000 × 1
            → Write backup to RTC

    UIPresenter_Tick():
        → HandleTimerExpiry(): UIModel_GetRemainingCookTime() updates display
        → HandleAutoOff(): counter resets (user timer supersedes)

    UIView_Process():
        → Toggle ColonBlinkFlag every 1000ms
        → Render "0:29", "0:28", ..., "0:01"

TIMER REACHES 0 (IndCooker_Tick):
    → IndCooker_PowerOff() → PowerBoard powers off hardware
    → TimerRunning = false

NEXT UIPresenter_Tick():
    → HandleTimerExpiry() detects: PrevTimerRunning=true, IsTimerRunning=false
    → UIModel_StopSession():
        → CookingSession_Update(now) — final power accumulation
        → CookingSession_Stop() — converts to kWh, saves to flash
        → SessionAggr_UpdatePowerUsage(1.5 kWh)
    → UIModel_PowerOffWithFan() — fan runs for cooldown
    → ShutdownFanCountdown = 60
    → UI state: END_OF_SET_COOK_TIME

END_OF_SET_COOK_TIME (15 seconds):
    → 3 beeps: buzzer ON 300ms, OFF 600ms, 3 times
    → After beeps done: UIPresenter_ForceStateTimeout()
    → Or timeout at 15s
    → UI state: SHUTDOWN_USED_LABEL

SHUTDOWN DISPLAY SEQUENCE:
    → USED_LABEL (5s) → SESSION_USAGE (10s) → OFF (1s) → STANDBY

FAN COOLDOWN (parallel):
    → ShutdownFanCountdown: 60, 59, 58, ... 0
    → At 0: UIModel_PowerOffPowerBoard() → full hardware off
```

---

## 12. Critical Gotchas

### 1. State is set BEFORE action function runs

The state table's `ShortPressNextStateID` is applied **before** calling `ActionFunctionShortPress`. If your action function needs to set a different state, use `NO_SUPPORTED_STATE` in the table and set it in the function.

### 2. Timeout counter is reset AFTER key events

`UIPresenter_Process()` always resets `StateTimeOutCounter` after processing a key event. Any timeout value you set inside an action function will be overwritten.

### 3. Session duration must be read BEFORE stopping

`CookingSession_Stop()` resets `CurrentSession`. Call `CookingSession_GetDurationInSeconds()` or `UIModel_GetSessionDuration()` before stopping.

### 4. IndCooker_StopTimer() during shutdown is mandatory

If you don't stop the timer during shutdown, `IndCooker_Tick()` may fire `IndCooker_PowerOff()` during the fan cooldown period, killing the fan prematurely.

### 5. Timer can expire while user is editing

If the user is in `TIME_SETTING_SCREEN` adjusting the timer and the original timer expires, `HandleTimerExpiry()` detects this via the `JustExpired` flag (was running → now stopped) and forces transition to `END_OF_SET_COOK_TIME`.

### 6. SetLevel triggers session callbacks; PowerOff does not (Highway)

`SetLevel()` immediately fires `StatusCallback`, which starts/updates the cooking session via `CookerState_On()`. `PowerOff()` and `PowerOffWithFan()` do NOT fire the callback. Session stop must be handled explicitly.

### 7. Remaining time uses ceiling display

`UIModel_GetRemainingCookTime()` uses ceiling division so 1-59 remaining seconds display as "0:01". The display never shows "0:00" until the timer is truly expired.

### 8. Auto-off creates a synthetic timer

At the 3:45 mark, auto-off creates a real timer on IndCooker for the remaining 15 minutes. This synthetic timer is indistinguishable from a user-set timer. If the user presses any key, this synthetic timer is cancelled and the auto-off counter resets.

### 9. ForceStateTimeout bypasses counter

`UIPresenter_ForceStateTimeout()` immediately calls `ProcessStateTimeOut()`, which transitions the state and calls the timeout action. Used by UIView animations (flash done, beeps done) to signal completion to the state machine.

### 10. Fan countdown runs independently

`ShutdownFanCountdown` decrements in `UIPresenter_Tick()` independently of the state machine. It can complete before, during, or after the shutdown display sequence.

---

## 13. Constants and Tunables

| Constant | Value | Unit | Purpose |
|----------|-------|------|---------|
| Max cook time | 4:00 | hours:min | Maximum user-settable timer |
| Max cook time (auto-off) | 4 hours (14400) | seconds | Auto-off threshold |
| Auto-off warning | 3:45 (13500) | seconds | Synthetic timer trigger |
| Timer increment (short) | 1 | minute | Per short press |
| Timer increment (fast/long) | 15 | minutes | Per long press |
| Time setting idle timeout | 4 | seconds | Return to cooking if no action |
| Time setting edit timeout | 2 | seconds | Apply timer after inactivity |
| Timer accept flash count | 5 | flashes | Visual confirmation |
| Timer accept flash ON | 500 | ms | Flash ON duration |
| Timer accept flash OFF | 500 | ms | Flash OFF duration |
| Timer accept state timeout | 7 | seconds | Max time in flash state |
| End of cooking beep count | 3 | beeps | Alert beeps |
| End of cooking beep ON | 300 | ms | Buzzer ON duration |
| End of cooking beep interval | 600 | ms | Time between beeps |
| End of cooking timeout | 15 | seconds | Max beeping time |
| Fan cooldown duration | 60 | seconds | Post-cooking fan run time |
| Fan cooldown threshold | 60 | seconds | Min session for fan cooldown |
| Long press threshold | 1000 | ms | Short vs long press |
| Colon blink rate | 1000 | ms | Active timer colon blink |
| Time blink rate | 500 | ms | Idle timer time blink |
| USED label display | 5 | seconds | Shutdown label time |
| Usage display | 10 | seconds | Shutdown kWh display time |
| Tick interval | 1000 | ms | System tick period |
| Keypad sample interval | 5 | ms | Button polling rate |

---

## 14. API Reference Summary

### UIModel Timer APIs

```c
void UIModel_IncrementTimerSetting(void);     // +1 min, cap at 4:00
void UIModel_DecrementTimerSetting(void);     // -1 min, floor at 0:00
void UIModel_IncrementTimerSettingFast(void); // +15 min, cap at 4:00
void UIModel_DecrementTimerSettingFast(void); // -15 min, floor at 0:00
void UIModel_ResetTimerSetting(void);         // Reset to 0:00
void UIModel_GetRemainingCookTime(void);      // Sync from IndCooker to UIModel (ceiling display)
bool UIModel_HasCookTimerExpired(void);       // Returns !IndCooker_IsTimerRunning()
int32_t UIModel_GetSessionDuration(void);     // Session duration in seconds
UIModelData_t UIModel_GetUIModelData(void);   // Get full model snapshot
```

### UIModel Power APIs

```c
void UIModel_IncrementPowerSetting(void);     // +1 level (skips 0), wraps 9→1
void UIModel_DecrementPowerSetting(void);     // -1 level (skips 0), wraps 1→9
void UIModel_StopSession(void);               // Stop CookingSession, save kWh
void UIModel_PowerOffWithFan(void);           // Fan-only mode for cooldown
void UIModel_PowerOffPowerBoard(void);        // Full power off
```

### IndCooker Timer APIs

```c
bool IndCooker_SetTimer(uint16_t Duration);        // Set timer (seconds), returns false if off/locked
void IndCooker_StopTimer(void);                     // Cancel timer without power off
bool IndCooker_IsTimerRunning(void);                // Check if timer is active
uint32_t IndCooker_GetRemainingTimerTime(void);     // Get remaining seconds
```

### IndCooker Power APIs

```c
void IndCooker_SetLevel(CookerPower_e Level);       // Set power level
void IndCooker_PowerOff(void);                       // Full power off
void IndCooker_PowerOffWithFan(void);                // Fan-only cooldown
```

### UIPresenter APIs

```c
void UIPresenter_Process(void);            // Main loop: key events + display update
void UIPresenter_Tick(void);               // 1-second tick: timeouts + timer expiry + auto-off
void UIPresenter_ForceStateTimeout(void);  // Immediately trigger current state's timeout action
```

### Session APIs

```c
void CookingSession_Start(TIME_T *now, CookerPower_e *level, uint32_t *watts);
void CookingSession_Update(TIME_T *now, CookerPower_e *level, uint32_t *watts);
void CookingSession_Stop(TIME_T *now);
void CookingSession_BackupSession(TIME_T *now);
int32_t CookingSession_GetDurationInSeconds(void);
```

---

## Adapting for a Different Product

When implementing this for a new product with a different UI:

1. **Keep the IndCooker timer engine as-is** — it's UI-agnostic (just seconds countdown)
2. **Keep CookerState/CookingSession/SessionAggr as-is** — they're business logic, not UI
3. **Replace UIView entirely** — your display hardware will be different
4. **Replace UIPresenter's state table** — add/remove states for your UI flow, but keep the same state machine processing logic
5. **Adapt UIModel** — may need different data fields for your display, but keep the same IndCooker wrapper pattern
6. **Adapt the keypad system** — different buttons, but keep the event-driven architecture
7. **Keep auto-off constants** — safety feature, unlikely to change between products
8. **Keep the fan cooldown logic** — thermal safety, hardware-dependent but logic is reusable
9. **Keep the session tracking chain** — business requirement, not UI-dependent
