---
name: Silent thermal regulator (Highway + CHK) — port guide
description: Replicate the thermal-control feature on a divergent branch. Covers the 3-state pot machine, ADC-indexed Highway lookup, OFF-during-throttle guard, and debug-flag cleanup. Concept-first, not copy-paste.
type: reference
originSessionId: 280775ba-8ead-4e17-8f7b-8895c4e7ddc2
---
# Silent thermal regulator (Highway + CHK) — port guide

**Source commits (G3, on `Fix/Highway-TIM14-OnlyWriter` after merge of `Feature/Temperature-Control`):**

| Commit | Subject | Scope |
|---|---|---|
| [`c67c50a`](https://github.com/ecoa-dev-team/STM32_FIRMWARE_G3/commit/c67c50a) | Add silent thermal regulator with PL1 throttle and ADC-indexed Highway lookup | new module + driver wiring + lookup rewrite + PowerBoard.c consolidation |
| [`9042c28`](https://github.com/ecoa-dev-team/STM32_FIRMWARE_G3/commit/9042c28) | Guard ThermalRegulator THROTTLED path from throttling up when user is off | safety follow-up |
| [`f98ff19`](https://github.com/ecoa-dev-team/STM32_FIRMWARE_G3/commit/f98ff19) | Silence default-off debug prints for thermal control and AT sender | flag flips + AtSender pattern |

The TIM14-OnlyWriter prerequisite (commit `7879299`) is documented separately in `Tim14SoleUartWriterPortGuide.md` — port that first if the target branch doesn't have it.

---

## 1. The feature in one paragraph

A silent, driver-agnostic soft-throttle that sits between the user's commanded power level and the wire-level command to the power board. When the pot/surface NTC crosses 175 °C the regulator overrides the wire-level command to either fully off (HARD_CUT, returns `COOKER_POWER_NO_POWER_FAN_ON`) or PL1 200 W (SOFT_CUT, returns `COOKER_POWER_200W`). The UI never sees the override — it reads the user's commanded level unchanged. The goal is to make the power board's onboard E3 trip rare enough that, when it does fire, users treat it as a real hazard rather than a routine annoyance.

---

## 2. Why each piece exists

### 2.1 Why a 3-state pot machine instead of bang-bang
The most natural design — "cut at 175, resume at 165" — works but bench evidence (Run 3 in the bench log series) showed that PL1 alone *can* hold steady-state temperatures in the 165–175 °C band, more gracefully than a full cut. So the desire was: gentle 200 W throttle when possible, hard cut only when necessary.

But the first overshoot per power-up is special. The cooker has the most pent-up thermal energy then, and PL1 alone may not be enough to bring it back down — it can climb past 175 to 185 even while throttled. With the power board's onboard E3 at 188 °C, a HARD_CUT at 185 leaves only 3 °C of margin, which is too tight in high-ambient conditions.

So the design becomes asymmetric:
- **First overshoot per power-up** → straight to HARD_CUT at 175. 13 °C margin from E3.
- **Subsequent overshoots** → SOFT_CUT (PL1) at 175. Thermal mass has settled; PL1 is sufficient.
- **Safety floor at 185** → if PL1 ever proves insufficient in steady state, HARD_CUT kicks in. Same 3 °C margin as before, but rare.

The "first overshoot consumed by HARD_CUT" decision is reset only on `Init` (i.e., power-cycle). UI on/off doesn't reset it.

### 2.2 Why HARD_CUT exit is origin-aware
Each HARD_CUT entry path has its own resume threshold so each path has its own 10 °C hysteresis:
- HARD_CUT entered from NORMAL (first overshoot, entry at 175) → exits at 165 directly to NORMAL. 10 °C hysteresis.
- HARD_CUT entered from THROTTLED (PL1 escape, entry at 185) → exits at 175 to THROTTLED, then THROTTLED → NORMAL at 165. 10 °C hysteresis on the 185-trip leg.

Without origin tracking, a single fixed exit threshold (e.g. 175) caused a one-frame HARD_CUT for the first-overshoot path: entry at 175, exit at 175 the next frame. Bench-observed in Run 5 before the fix.

### 2.3 Why a 256-entry ADC-indexed Highway lookup table
The original Highway lookup was indexed by *computed* NTC resistance from `R = 5.1 * 255 / ADC - 5.1` (POT) or `R = 10 * 255 / ADC - 10` (IGBT), cast to `uint16_t`, then used to index a 326-entry table. Integer truncation collapsed many ADC values into table index 0 or 1:

| ADC | Computed R (kΩ) | Index after cast | Old table reads |
|---|---|---|---|
| 192–213 | 1.0–1.7 | 1 | **165 °C** |
| 214–254 | <1.0 | 0 | **255 °C** (sentinel) |

At real cooktop temperatures of 168–175 °C (ADC ~210–215), the firmware oscillated between "165 °C" and "255 °C" frame-to-frame with single-LSB ADC noise. The G4 firmware solved this by precomputing a 256-entry table indexed directly by ADC value. We ported that table verbatim. Validation in Run 4 confirmed: with regulator output bypassed, the board's onboard E3 fired at ADC = 224 → 188 °C per our table → matches the documented Highway trip point with `DEFAULT_FURNACE_HIGH_TEMP_PROTECTION_SETTING = 0xE0`.

### 2.4 Why a single `POWER_BOARD_TYPE` define
`PowerBoard.c` previously hardcoded `PowerBoardType = COOKER_POWER_BOARD_HIGHWAY;` in *two* places (`PowerBoard_Init` and `PowerBoard_SetBaudRate`). Easy to update one and forget the other → mismatched dispatch and baud rate. Replaced with a single `#define POWER_BOARD_TYPE COOKER_POWER_BOARD_HIGHWAY` consumed by both sites.

### 2.5 Why the "never throttle UP" guard (commit `9042c28`)
The original `ApplyToLevel` returned `COOKER_POWER_200W` unconditionally when `PotState == POT_THROTTLED`. If the user pressed OFF mid-throttle, `CurrentPowerLevel` became `NO_POWER` but `ApplyToLevel(NO_POWER) → 200W`. The cooker kept heating at 200 W against the user's explicit off command until pot temp dropped to 165 °C and THROTTLED cleared.

Fix: when the user has commanded no heat (`NO_POWER`, `0W`, or `NO_POWER_FAN_ON`), the THROTTLED branch returns the requested level unchanged. The throttle never raises power above what the user asked for. The HARD_CUT path is naturally safe (always returns `NO_POWER_FAN_ON` which is fan-only / no-heat).

### 2.6 Why a separate debug flag for the temperature diagnostic (commit `f98ff19`)
The per-frame `Highway frame: ...` diagnostic is gated by its own `HIGHWAY_TEMP_DEBUG_ENABLED` flag, separate from the existing `HIGHWAY_DRIVER_DEBUG_ENABLED`. Reason: we want to be able to toggle the temperature diagnostic during bench work without re-enabling all the noisy per-tick HighwayDriver prints (`Tick`, `Valid data received`, `Unknown parameter`, `PowerOn/Off/SetLevel`, etc.). Two independent gates, default both off.

AtSender.c was retrofitted with the same module-debug pattern so the "Failed to get bluetooth device name" and "Invalid command null" prints can be compile-time silenced. Convention used across HighwayDriver / CHKDriver / etc.

---

## 3. Files involved

| Path | Change type |
|---|---|
| `ExternalPeripherals/ThermalRegulator/Inc/ThermalRegulator.h` | **new** |
| `ExternalPeripherals/ThermalRegulator/Src/ThermalRegulator.c` | **new** |
| `ExternalPeripherals/PowerBoard/Inc/HighwayTempLookUp.h` | full rewrite (resistance- → ADC-indexed) |
| `ExternalPeripherals/PowerBoard/Src/HighwayDriver.c` | regulator wiring, lookup conversion simplify, debug flag |
| `ExternalPeripherals/PowerBoard/Src/CHKDriver.c` | regulator wiring |
| `ExternalPeripherals/PowerBoard/Src/PowerBoard.c` | single `POWER_BOARD_TYPE` define, regulator init |
| `ExternalPeripherals/RFComms/At/Src/AtSender.c` | module debug-flag pattern |
| `CMakeLists.txt` | add source + include path |
| `.cproject` | add include path |

---

## 4. Implementation guide

### 4.1 `ThermalRegulator.h` (new)

A minimal, driver-agnostic API. Three public functions and the existing `CookerPower_e` / `CookerError_e` types from `PowerBoard.h`:

```c
void ThermalRegulator_Init(void);
void ThermalRegulator_Update(uint8_t PotTempC, uint8_t IgbtTempC, CookerError_e SensorError);
CookerPower_e ThermalRegulator_ApplyToLevel(CookerPower_e Requested);
bool ThermalRegulator_IsThermalCut(void);  /* optional convenience query */
```

- `Init` resets state. Safe to call before any temperature is known.
- `Update` is called by the driver's RX-frame parser once per status frame. It runs the state machine.
- `ApplyToLevel` is called by the driver's TX path each frame, immediately before sending. It resolves the requested level into the wire-level command. **This is the only function safe to call from ISR context**; the rest are main-loop.
- `IsThermalCut` returns true if any limit is currently active. Useful for "are we doing anything?" queries from outside.

Header guard, `#include <stdbool.h>`, `<stdint.h>`, and `PowerBoard.h`. Match the project's existing comment style.

### 4.2 `ThermalRegulator.c` (new)

This is the meat. Structure:

#### Debug pattern (match HighwayDriver / CHKDriver convention)

```c
#define THERMAL_REGULATOR_DEBUG_ENABLED 0   /* set to 1 to enable edge prints */

#if THERMAL_REGULATOR_DEBUG_ENABLED == 1
#include "Debug.h"
#define DebugOut(fmt, ...) DebugPrint(fmt, ##__VA_ARGS__)
#else
#define DebugOut(fmt, ...)
#endif
```

#### Compile-time toggles

```c
/* 0 = regulator output bypassed (state machine still runs and logs, but
 * ApplyToLevel always returns Requested). Used to characterise the power
 * board's own E3 trip. Set to 1 for normal operation. */
#define THERMAL_REGULATOR_ENABLED  1

/* 0 = baseline 2-state machine (pot HARD_CUT at 175, NORMAL at 165).
 * 1 = the 3-state PL1 design described in this guide. */
#define POT_THROTTLE_MODE_PL1      1
```

#### Thresholds

```c
#define POT_OVERTEMP_OFF_C    175  /* SOFT_CUT trip / HARD_CUT relax dest */
#define POT_OVERTEMP_ON_C     165  /* THROTTLED → NORMAL exit / first-overshoot HARD_CUT exit */
#define POT_HARD_CUT_C        185  /* THROTTLED → HARD_CUT (PL1 escape) */
#define POT_HARD_CUT_ON_C     175  /* PL1-escape HARD_CUT → THROTTLED */
#define IGBT_OVERTEMP_OFF_C   70
#define IGBT_OVERTEMP_ON_C    65
```

#### State

```c
typedef enum { POT_NORMAL, POT_THROTTLED, POT_HARD_CUT } PotThermalState_e;

static volatile PotThermalState_e PotState = POT_NORMAL;
static volatile bool IgbtThermalCut = false;
static volatile bool FirstOvershootHandled = false;
static volatile bool HardCutFromThrottled = false;
```

`volatile` is required: `PotState` and `IgbtThermalCut` are written from main-loop context (RX parser) and read from ISR context (TX path, via `ApplyToLevel`). On Cortex-M0+, byte-sized reads/writes are atomic, but `volatile` prevents the compiler from caching across the ISR boundary.

`FirstOvershootHandled` is set true on the first HARD_CUT entry per power-up. Reset only by `Init` — UI on/off doesn't reset it.

`HardCutFromThrottled` tracks how HARD_CUT was entered so the exit logic can pick the right threshold.

#### State machine in `Update`

The shape, in pseudocode:

```
on each Update(potT, igbtT, sensorErr):
    PotSensorFault = (sensorErr is SURFACE_OPEN/SHORT/FAILURE)
    IgbtSensorFault = (sensorErr is IGBT_OPEN/SHORT)
    PrevPotState = PotState
    PrevIgbtCut = IgbtThermalCut

    -- Pot side
    if PotSensorFault:
        PotState = HARD_CUT
    elif POT_THROTTLE_MODE_PL1:
        switch PotState:
            NORMAL:
                if potT >= 185: PotState=HARD_CUT, FirstOvershootHandled=true, HardCutFromThrottled=false
                elif potT >= 175:
                    if not FirstOvershootHandled:
                        PotState=HARD_CUT, FirstOvershootHandled=true, HardCutFromThrottled=false
                    else:
                        PotState=THROTTLED
            THROTTLED:
                if potT >= 185: PotState=HARD_CUT, HardCutFromThrottled=true
                elif potT <= 165: PotState=NORMAL
            HARD_CUT:
                if HardCutFromThrottled:
                    if potT <= 175: PotState=THROTTLED
                else:
                    if potT <= 165: PotState=NORMAL
    else:
        -- baseline 2-state
        switch PotState:
            NORMAL: if potT >= 175: PotState=HARD_CUT
            HARD_CUT: if potT <= 165: PotState=NORMAL
            THROTTLED: PotState=HARD_CUT  -- defensive; unreachable in baseline

    -- IGBT side (bang-bang)
    if IgbtSensorFault: IgbtThermalCut = true
    elif !IgbtThermalCut and igbtT >= 70: IgbtThermalCut = true
    elif IgbtThermalCut and igbtT <= 65: IgbtThermalCut = false

    -- Edge debug prints (only when state changes)
```

Translate that pseudocode literally into C. The key invariants:
- **Sensor fault forces HARD_CUT** (or IGBT cut). Defence in depth — fault first, threshold logic second.
- **Within the threshold checks**, mutually exclusive `if / else if` so only one transition per frame.
- **HARD_CUT exit threshold consults `HardCutFromThrottled`** — this is the origin-aware hysteresis that fixes the Run 5 one-frame bug.
- **The PL1 escape path sets `HardCutFromThrottled = true`**; the first-overshoot path sets it `false`. Both also set `FirstOvershootHandled = true` so subsequent overshoots use the SOFT_CUT branch.

#### `ApplyToLevel` (the resolver — important for E1 safety)

```c
CookerPower_e ThermalRegulator_ApplyToLevel(CookerPower_e Requested) {
#if THERMAL_REGULATOR_ENABLED
    /* IGBT cut and pot HARD_CUT both demand a full hard cut. Safe to apply
     * unconditionally — NO_POWER_FAN_ON only enables the fan, no heat. */
    if (IgbtThermalCut || (PotState == POT_HARD_CUT)) {
        return COOKER_POWER_NO_POWER_FAN_ON;
    }
    if (PotState == POT_THROTTLED) {
        /* Never throttle UP. If the user has commanded no heat (off, 0W
         * standby, or cool-down) the THROTTLED substitution to 200W would
         * actively heat a cooker the user has turned off — keep their
         * level instead. */
        if (Requested == COOKER_POWER_NO_POWER ||
            Requested == COOKER_POWER_0W ||
            Requested == COOKER_POWER_NO_POWER_FAN_ON) {
            return Requested;
        }
        return COOKER_POWER_200W;
    }
#endif
    return Requested;
}
```

The "never throttle UP" check is `9042c28`. Don't skip it — without it, pressing OFF mid-throttle leaves the cooker heating at 200 W until pot temp drops below 165 °C.

#### Edge debug prints

After the state machine block, if `PotState != PrevPotState` emit a single line. Include a `PotActionDescription(Prev, Curr)` helper that returns a human-readable label:

- `Curr == HARD_CUT && Prev == NORMAL`: `"pot HARD CUT (initial overshoot, wire=OFF+FAN)"`
- `Curr == HARD_CUT && Prev == THROTTLED`: `"pot HARD CUT (PL1 not enough, wire=OFF+FAN)"`
- `Curr == THROTTLED && Prev == HARD_CUT`: `"pot RELAX (hard cut -> soft cut, wire=PL1 200W)"`
- `Curr == THROTTLED && Prev == NORMAL`: `"pot SOFT CUT (wire=PL1 200W)"`
- `Curr == NORMAL && Prev == HARD_CUT`: `"pot RESUME (from hard cut direct, wire=user level)"`
- `Curr == NORMAL && Prev == THROTTLED`: `"pot RESUME (wire=user level)"`

Format: `ThermalRegulator: <action> [<PrevName> -> <CurrName>] (temp=NC, fault=N)`. Edge-only: don't print every frame, only on transitions.

IGBT side gets two edge prints: `IGBT HARD CUT (wire=OFF+FAN)` on rising edge, `IGBT RESUME` on falling.

### 4.3 `HighwayTempLookUp.h` (full rewrite)

Replace the resistance-indexed `NTCTempValues[326]` table with two 256-entry ADC-indexed tables:

```c
static const uint8_t PotTempFromADC[256] = { /* 256 values */ };
static const uint8_t IgbtTempFromADC[256] = { /* 256 values */ };
```

- Index 0 and 246–255 map to the **sentinel value 255**, meaning "invalid / saturated / NTC short".
- Other entries are temperatures in °C, computed from the manufacturer's NTC characteristic table for each ADC value.
- The POT table assumes a 5.1 kΩ pull-down divider; IGBT assumes 10 kΩ.
- Mark both arrays `static const` so the header is safe to include from multiple TUs.

**Don't hand-compute these values.** Copy them from G4's `STM32_FIRMWARE_G4/origin/Temp-Control` branch at `ExternalPeripherals/PowerBoard/Inc/HighwayTempLookUp.h`. The values were derived once, validated empirically (Run 4 confirmed ADC=224 → 188 °C trip), and shouldn't be re-derived ad-hoc.

The header should also remove `NUM_NTC_RESISTANCE_VALUES` since it's no longer used.

### 4.4 `HighwayDriver.c` changes

Five distinct edits in this file.

#### 4.4.1 Add the new include
After the existing `#include "HighwayTempLookUp.h"`, add `#include "ThermalRegulator.h"`.

#### 4.4.2 Add the new debug flag pair
Replace the existing single-flag debug block with:

```c
#define HIGHWAY_DRIVER_DEBUG_ENABLED 0
#define HIGHWAY_TEMP_DEBUG_ENABLED   0

#if (HIGHWAY_DRIVER_DEBUG_ENABLED == 1) || (HIGHWAY_TEMP_DEBUG_ENABLED == 1)
#include "Debug.h"
#endif

#if HIGHWAY_DRIVER_DEBUG_ENABLED == 1
#define DebugOut(fmt, ...) DebugPrint(fmt, ##__VA_ARGS__)
#else
#define DebugOut(fmt, ...)
#endif

#if HIGHWAY_TEMP_DEBUG_ENABLED == 1
#define TempDebugOut(fmt, ...) DebugPrint(fmt, ##__VA_ARGS__)
#else
#define TempDebugOut(fmt, ...)
#endif
```

#### 4.4.3 Simplify the conversion functions
Replace the two `Convert*TemperatureADCReadingToCelsius` functions with one-liners:

```c
static uint8_t ConvertPotTemperatureADCReadingToCelsius(uint8_t ADCReading) {
    return PotTempFromADC[ADCReading];
}
static uint8_t ConvertIgbtTemperatureADCReadingToCelsius(uint8_t ADCReading) {
    return IgbtTempFromADC[ADCReading];
}
```

Delete the now-unused `static uint8_t LookUpTempValInCelsius(uint16_t NTCResistanceVal);` along with its prototype, and the three defines `POT_TEMP_NTC_BOTTOM_RESISTOR_VAL`, `IGBT_TEMP_NTC_BOTTOM_RESISTOR_VAL`, `MAX_NTC_ADC_READING`.

The `if (ADCReading == 0) return 0;` divide-by-zero guard from the old version is no longer needed — the new table's index 0 maps to 255 (sentinel) without any computation.

#### 4.4.4 Wire the regulator into `UpdateCurrentStatus`
After the existing temperature decode lines:

```c
CurrentError = (CookerError_e)Data[0];
CurrentPotTemperature = ConvertPotTemperatureADCReadingToCelsius(Data[1]);
CurrentIGBTTemperature = ConvertIgbtTemperatureADCReadingToCelsius(Data[2]);
```

Add immediately after:

```c
ThermalRegulator_Update(CurrentPotTemperature, CurrentIGBTTemperature, CurrentError);
```

Then (optionally) add the throttled per-frame diagnostic — gated by `HIGHWAY_TEMP_DEBUG_ENABLED`, decimated 1-in-50 to give ~1 Hz at 50 Hz status frames:

```c
static uint8_t FrameLogCounter = 0;
if (++FrameLogCounter >= 50) {
    FrameLogCounter = 0;
    CookerPower_e EffectiveLevel = ThermalRegulator_ApplyToLevel(CurrentPowerLevel);
    TempDebugOut("Highway frame: err=%u potADC=%u potT=%uC igbtADC=%u igbtT=%uC sysFlag=0x%02X user=%u eff=%u\n",
                 (unsigned)CurrentError, (unsigned)Data[1], (unsigned)CurrentPotTemperature,
                 (unsigned)Data[2], (unsigned)CurrentIGBTTemperature, (unsigned)Data[3],
                 (unsigned)CurrentPowerLevel, (unsigned)EffectiveLevel);
}
```

The `user=N eff=M` field is the single most useful debug aid — `user != eff` flags an active throttle/cut and shows what the wire is actually carrying.

#### 4.4.5 Wire the regulator into `SendCurrentPowerCommandToCooker`
Replace the line(s) that read `CurrentPowerLevel` for `FireGearByte` / `SystemFlagByte` lookups with a resolved local:

```c
CookerPower_e EffectiveLevel = ThermalRegulator_ApplyToLevel(CurrentPowerLevel);
DataOut[2] = CookerControl[EffectiveLevel].FireGearByte;
/* ... */
DataOut[4] = CookerControl[EffectiveLevel].SystemFlagByte;
```

**Critical:** `CurrentPowerLevel` is never written to — only read. The regulator's effect is purely on the wire. The UI / kWh accumulator / timer all keep reading `CurrentPowerLevel` and so observe the user's commanded level. This is what makes the throttle "silent".

The beep path (`if (Beep) DataOut[2] += BEEP_COMMAND;`) stays unchanged and works regardless of `EffectiveLevel` because the beep bit is OR'd in after the FireGear lookup.

### 4.5 `CHKDriver.c` changes

Three edits.

#### 4.5.1 Add include
`#include "ThermalRegulator.h"` after the existing `#include "CHKDriver.h"`.

#### 4.5.2 Update call in `UpdateCurrentStatus`
After the existing temperature decode lines (find where `NewIgbtTemp` and `NewSurfaceTemp` are computed), add:

```c
ThermalRegulator_Update(NewSurfaceTemp, NewIgbtTemp, NewError);
```

The CHK driver uses local `NewSurfaceTemp`/`NewIgbtTemp` before assigning to the `Current*` statics. Use the locals — `NewError` is already computed by `DetectError(CookerStatusByte)` earlier in the function.

#### 4.5.3 Wire `ApplyToLevel` in `SendCurrentPowerCommandToCooker`
Replace the `Command[8] = ConvertPowerLevelToFireGearByte(CurrentPowerLevel);` line with:

```c
CookerPower_e EffectiveLevel = ThermalRegulator_ApplyToLevel(CurrentPowerLevel);
Command[8] = ConvertPowerLevelToFireGearByte(EffectiveLevel);
```

CHK's `ConvertPowerLevelToFireGearByte` switch statement doesn't explicitly handle `COOKER_POWER_NO_POWER_FAN_ON` — it falls to the `default: return 0;` case, which is correct (FireGear 0 = no heat). The CHK Switch byte at `Command[7]` stays driven by `CurrentPowerState`, so the board stays "powered on, FireGear 0" during a cut — exactly the right behaviour (peripherals/fan live, no heating).

### 4.6 `PowerBoard.c` changes

Three edits.

#### 4.6.1 Add include
`#include "ThermalRegulator.h"` near the existing `#include "CHKDriver.h"`.

#### 4.6.2 Add the consolidation define
At the top of the "Local Definitions" block, add:

```c
/* Single source of truth for which power board is fitted. Both the dispatch
 * table and the UART baud rate read this; the previous code hardcoded
 * HIGHWAY in two separate places, so changing board meant remembering to
 * edit both. */
#define POWER_BOARD_TYPE COOKER_POWER_BOARD_HIGHWAY
```

(Or `CHK` / `AIFER` if the target product needs that board.)

#### 4.6.3 Replace the two hardcoded assignments
In `PowerBoard_Init`, replace `PowerBoardType = COOKER_POWER_BOARD_HIGHWAY;` with `PowerBoardType = POWER_BOARD_TYPE;`.

In `PowerBoard_SetBaudRate`, replace the same hardcoded assignment with the same `POWER_BOARD_TYPE` reference.

Then **add the regulator init call** inside `PowerBoard_Init`, after the `PowerBoardType` assignment and before the dispatch chain:

```c
ThermalRegulator_Init();
```

### 4.7 `AtSender.c` retrofit (debug-flag pattern)

Remove the unconditional `#include "Debug.h"` from the top includes. Add the standard module-debug pattern instead:

```c
#define AT_SENDER_DEBUG_ENABLED 0

#if AT_SENDER_DEBUG_ENABLED == 1
#include "Debug.h"
#define DebugOut(fmt, ...) DebugPrint(fmt, ##__VA_ARGS__)
#else
#define DebugOut(fmt, ...)
#endif
```

The existing `DebugOut(...)` call sites in the file stay unchanged — they now expand to nothing when the flag is 0.

### 4.8 Build system

#### `CMakeLists.txt`
Add the source file to the project's source list:

```cmake
ExternalPeripherals/ThermalRegulator/Src/ThermalRegulator.c
```

(next to the other `ExternalPeripherals/PowerBoard/Src/*.c` lines)

And add the include path to `target_include_directories`:

```cmake
ExternalPeripherals/ThermalRegulator/Inc
```

#### `.cproject`
Two include-path blocks (Debug and Release configurations). Add to **both**:

```xml
<listOptionValue builtIn="false" value="../ExternalPeripherals/ThermalRegulator/Inc"/>
```

next to the existing `../ExternalPeripherals/PowerBoard/Inc` entries. The source path entry (`<entry ... name="ExternalPeripherals"/>`) automatically picks up `.c` files in any subfolder so you don't need to add the source explicitly there.

---

## 5. Verification

Per the G3 bench-test plan in `features.md`, the validated states are:
- **Run 6** (final form on G3): clean first-overshoot HARD_CUT, all subsequent overshoots handled by PL1 SOFT_CUT, zero E3 trips, all transitions have proper 10 °C hysteresis.
- **Run 4** confirmed the lookup table calibration (E3 fires at ADC = 224 → 188 °C with `0xE0` protection setting).

On a fresh port, the minimal verification I'd want:
1. **Build clean.**
2. **Enable both debug flags** (`THERMAL_REGULATOR_DEBUG_ENABLED 1`, `HIGHWAY_TEMP_DEBUG_ENABLED 1`).
3. **One short cook** — let pot reach 175 °C. Expect:
   - `Highway frame:` line every ~1 s with sensible `potT` values.
   - `ThermalRegulator: pot HARD CUT (initial overshoot, ...) [NORMAL -> HARD_CUT] (temp=175C, ...)` on first crossing.
   - `RESUME (from hard cut direct)` around 165 °C.
   - On the next overshoot, `SOFT CUT (wire=PL1 200W) [NORMAL -> THROTTLED]`.
4. **OFF mid-throttle test (E1)** — during a SOFT_CUT, press OFF. Watch the next `Highway frame:` log shows `user=0 eff=0`. Pre-fix would show `user=0 eff=2`.
5. **Flip debug flags back to 0** after bench work.

---

## 6. Gotchas / divergence handling

### 6.1 `DEFAULT_FURNACE_HIGH_TEMP_PROTECTION_SETTING` on Prescan
The `Prescan` branch set this to `0xD3` in its own development. We set it to `0xE0`. These produce different E3 trip points on the power board. **Default to `0xE0` when porting alongside the thermal regulator** — `0xE0` is what the regulator's 175/185/188 design assumes. If the Prescan team wants `0xD3` for some other reason, that needs a conversation, not an automatic merge.

### 6.2 CHK-Bring-Up has its own CHK thermal protection
Commit `44a93b3` on `CHK-Bring-Up` added `IgbtThermalCut`/`GlassThermalCut` latches inside `CHKDriver.c` plus a `CHKTempLookUp.h` and `UpdateThermalProtection()`. **All of that is superseded by the shared regulator** and should be deleted during the port:
- Remove `IgbtThermalCut` and `GlassThermalCut` statics.
- Remove `UpdateThermalProtection` (function + prototype + call site).
- Remove `CHKTempLookUp.h` include and use of `CHKNTCTempFromAdc[]` — actually wait, the CHK NTC lookup is product-specific. **Keep `CHKTempLookUp.h` and `ConvertNTCAdcToCelsius`** — they're the temperature-conversion path. Only delete the *thermal-cut logic* layered on top.
- Remove the error-injection block in `UpdateCurrentStatus` that surfaces the CHK thermal cuts as `IGBT_SENSOR_OVER_TEMPERATURE` / `SURFACE_TEMP_SENSOR_OVER_TEMPERATURE` errors — the new regulator is silent.
- Remove the `if (ThermalCut)` block in `SendCurrentPowerCommandToCooker` that sets `POWER_SWITCH_PAUSE` / forces `FireGear` to 0 — replaced by the `ApplyToLevel` substitution.
- Wire the shared regulator in instead (steps 4.5.1–4.5.3 above).

Functionally similar but the new regulator is silent (no UI error injection), uses the asymmetric first-overshoot HARD_CUT design, and shares the lookup module pattern with Highway.

### 6.3 Boot with hot pot
If the pot/cooktop is already ≥ 175 °C on the first status frame after boot (e.g. residual heat from a previous session), the regulator immediately triggers its first-overshoot HARD_CUT and holds until pot temp drops to 165 °C. The cooker won't heat during that window even if the user turns on. **This is correct safety behaviour** but worth flagging to a product owner — could surprise users who think the cooker is broken when it's actually just waiting for safety.

### 6.4 IGBT calibration is unverified
Across all bench runs on G3 Highway, IGBT temps sat at 28–42 °C — never close to the 70 °C trigger. Three possibilities: (a) the IGBT is genuinely well-cooled and reads correctly, (b) the G4-ported IGBT lookup uses a different NTC β than the actual G3 IGBT NTC part, or (c) `Data[2]` isn't quite the IGBT ADC byte we think it is. The 70 °C threshold is so far above observed values that it's effectively untested. Not a blocker — pot side does all the safety work — but worth a heatsink touch-check during port verification if you have bench time.

### 6.5 `volatile` is required
Don't drop the `volatile` on `PotState`, `IgbtThermalCut`, `FirstOvershootHandled`, or `HardCutFromThrottled`. They're written from main-loop context and read from ISR context. Without `volatile`, GCC at `-Og` may CSE the reads inside the ISR.

### 6.6 No copy-paste of the lookup table
The values in `PotTempFromADC[256]` and `IgbtTempFromADC[256]` are not derivable from first principles — they come from the NTC manufacturer's characteristic curve. **Always copy from G4 `STM32_FIRMWARE_G4/origin/Temp-Control`'s `HighwayTempLookUp.h`.** Re-deriving by hand will likely produce subtly different values that won't match the empirical 188 °C trip validation.

### 6.7 The TIM14-OnlyWriter prerequisite
The thermal regulator's `ApplyToLevel` is read from inside the TIM14 ISR's TX path. If the target branch has main-loop callers that also write the UART directly (i.e. the TIM14-OnlyWriter fix from `Tim14SoleUartWriterPortGuide.md` is missing), the regulator's level resolution may race with those callers and produce inconsistent wire frames. **Port `Tim14SoleUartWriterPortGuide.md` first if the target branch doesn't already have it.**

### 6.8 Don't reintroduce the divide-by-zero guard
The old `Convert*TemperatureADCReadingToCelsius` had a `if (ADCReading == 0) return 0;` guard to avoid divide-by-zero in the resistance computation. The new ADC-indexed lookup doesn't divide at all — index 0 is just the sentinel 255. Don't reintroduce the guard. If you see it in a merge conflict, delete it.
