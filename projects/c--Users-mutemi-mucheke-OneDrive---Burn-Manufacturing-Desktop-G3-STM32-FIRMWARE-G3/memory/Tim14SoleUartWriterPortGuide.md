---
name: TIM14 Sole UART Writer (Highway) — port guide
description: Replicate the Highway TIM14-only-writer fix on a divergent branch. Concept-first, not copy-paste.
type: reference
originSessionId: 280775ba-8ead-4e17-8f7b-8895c4e7ddc2
---
# TIM14 Sole UART Writer (Highway) — port guide

**Source commit (G3):** [`7879299`](https://github.com/ecoa-dev-team/STM32_FIRMWARE_G3/commit/7879299) "Make TIM14 ISR sole UART writer and raise furnace high-temp protection to 0xE0"

This commit is two logically distinct changes that landed together:

1. **The race-condition fix** (the main concern of this guide).
2. **`DEFAULT_FURNACE_HIGH_TEMP_PROTECTION_SETTING` raised from `0xC8` to `0xE0`** — bench-validated to trip Highway's onboard E3 at 188 °C. Verified empirically in Run 4 of the temperature-control bench tests (see `ThermalControlPortGuide.md` for context).

Both live in `ExternalPeripherals/PowerBoard/Src/HighwayDriver.c`.

---

## 1. The race condition

### Symptom
Highway power-board UART TX FIFO occasionally carried corrupted frames, especially when key presses (beeps) coincided with power-level changes or fan toggles. Symptoms downstream were a silent power board (didn't react to commands) or intermittent E2/E4 errors caused by garbage being interpreted as malformed protocol.

### Root cause
Two distinct contexts were writing to the Highway power-board UART:

- **TIM14 ISR** — running every ~20 ms, called `SendCurrentPowerCommandToCooker(false)` to emit the 10-byte status command frame.
- **Main-loop public API** — `HighwayDriver_PowerOn`, `HighwayDriver_PowerOff`, `HighwayDriver_FanOn`, `HighwayDriver_FanOff`, `HighwayDriver_BuzzerOn`, `HighwayDriver_SingleBeep` each also called `SendCurrentPowerCommandToCooker(...)` synchronously.

If a main-loop caller hit `SendCurrentPowerCommandToCooker` while the TIM14 ISR was mid-frame, the two TX writes interleaved into the underlying UART DMA/FIFO and produced a malformed frame on the wire.

### The fix concept
**TIM14 ISR is the only path that writes the UART.** Public API setters update *state variables* only; the ISR materialises those variables onto the wire on the next ~20 ms tick. Worst-case latency from API call to wire is one TIM14 tick (≤ 20 ms), which is well within the cooker's expected command cadence.

For single-shot beeps, the new state is a pending flag (`PendingBeep`). The ISR reads it, sets it false in the same critical section, and OR's the BEEP bit into that frame.

---

## 2. Implementation steps

All edits are inside `ExternalPeripherals/PowerBoard/Src/HighwayDriver.c`. The file is structured into Include / Debug / Defines / Local variables / Local prototypes / Private functions / Public functions. Match the existing sections — do not move them around.

### Step 1: Add the `PendingBeep` state variable

Near the existing `static CookerError_e CurrentError = COOKER_ERROR_NO_ERROR;` declarations (the file's "Local Variables" block), add:

```c
/* Single-shot beep request. Public API setters mark it true; TIM14 ISR
 * consumes it on the next tick and clears it. Volatile because it's
 * written from main-loop context and read from ISR. */
static volatile bool PendingBeep = false;
```

The `volatile` is load-bearing — without it, the compiler may cache the read inside the ISR. Don't drop it.

### Step 2: Restructure the TIM14 ISR

Find `static void TimerIsr(TIM_HandleTypeDef *TimerHandle)` and rewrite its body so that it:

1. Snapshots `PendingBeep` into a local `bool BeepThisFrame`.
2. If `BeepThisFrame` is true, clears `PendingBeep`.
3. Calls `SendCurrentPowerCommandToCooker(BeepThisFrame)`.

The reason for the snapshot-and-clear pattern is to ensure the beep fires for exactly one TIM14 frame even if a main-loop setter writes `PendingBeep = true` while we're already inside the ISR — `volatile bool` reads/writes are byte-atomic on Cortex-M0+ so this works as a simple test-and-set.

### Step 3: Strip synchronous UART sends from public API setters

For each of `HighwayDriver_PowerOn`, `HighwayDriver_PowerOff`, `HighwayDriver_FanOn`, `HighwayDriver_FanOff`, `HighwayDriver_BuzzerOn`, `HighwayDriver_BuzzerOff`, `HighwayDriver_SingleBeep`:

- **Remove** the `SendCurrentPowerCommandToCooker(...)` call inside the function body.
- **Keep** the state mutation (`CurrentPowerLevel = COOKER_POWER_0W;` etc.) — that's what the ISR will materialise on the next tick.
- For `BuzzerOn` and `SingleBeep`, replace the `SendCurrentPowerCommandToCooker(true)` with `PendingBeep = true;`.
- For `BuzzerOff`, the function becomes a near-no-op — Highway beeps are single-shot; the beep bit auto-clears in the ISR after one frame. Keep the function for API symmetry but document with a one-line comment that it's a no-op by design.

Update the doxygen `\brief` of each affected function so it reads "Set commanded X. TIM14 ISR transmits on next tick." or similar — the old wording lies if it says "Send the serial command to ...".

`HighwayDriver_SetLevel` already didn't make a synchronous send (it called `SetLevel` which only updated state), so it stays untouched.

### Step 4: Update the `TimerIsr` doxygen

Document the invariant explicitly so future maintainers don't reintroduce a synchronous send. Phrase it as:

> Sole writer to the power-board UART TX path: this prevents main-loop `Send*` calls (PowerOn/Off, Fan, Buzzer, SingleBeep) from racing this ISR mid-frame and corrupting Fifo3. Public API setters update state variables (`CurrentPowerLevel`, `PendingBeep`); this ISR materialises them on the next 20 ms tick.

### Step 5: Furnace high-temp protection setting

In the file's defines block, change:

```c
#define DEFAULT_FURNACE_HIGH_TEMP_PROTECTION_SETTING 0xC8
```

to

```c
#define DEFAULT_FURNACE_HIGH_TEMP_PROTECTION_SETTING 0xE0
```

This byte is sent as `DataOut[5]` in `SendCurrentPowerCommandToCooker`. It tells the Highway board where to set its onboard surface-temperature E3 trip. `0xE0` produces a trip at approximately 188 °C, validated empirically (Run 4 of the temperature-control bench). `0xC8` was an earlier value that may trip too early under some loads.

**Note about divergent branches:** the `Prescan` branch has independently set this to `0xD3` (commit a7634be on 2026-06-11). If you're porting to a Prescan that has `0xD3`, treat that as a textual conflict that needs human review — `0xE0` is our validated value for the new thermal regulator's 13/3 °C margin design, but a Prescan team may have characterised `0xD3` for their use case. Default to keeping `0xE0` when porting alongside the thermal regulator changes; otherwise check with the Prescan owner.

---

## 3. Verification

This is hard to bench-test directly because the race is intermittent. The proxies that confirm it works:

1. **No corrupted Highway frames** — run a long cook session with many key presses (timer up/down, level up/down) and watch the power board's responses for any malformed-frame errors. Pre-fix would occasionally drop frames; post-fix should not.
2. **Beeps still fire on key press** — confirms `PendingBeep` path works.
3. **Power level changes still take effect within ~20 ms** — visually verify the cooker responds to up/down presses without perceptible lag.

A code-review check: grep `HighwayDriver.c` for `SendCurrentPowerCommandToCooker`. There should be **exactly two** call sites — the function definition itself, and the call from `TimerIsr`. If any public API function still calls it, the fix is incomplete.

---

## 4. Gotchas

- **`HighwayDriver_BuzzerOff` becoming an "obvious no-op"** is the correct outcome. Don't delete the function — UI code or other modules may call it for API symmetry. Just let it return `false` (or whatever the original return convention was) without doing anything material.
- **CHK driver has the same general pattern** but is separately fixed in commit `4a37f73` ("Fix CHK power-board silence from beep/TIM7 UART race") on the `CHK-Bring-Up` branch family. If you're porting to a branch that already includes `4a37f73`, the CHK side is independent and you don't need to touch it. If you're porting to a branch *without* it, the analogous fix is needed there but is outside the scope of this guide.
- **`PendingBeep` must be `volatile`** — without it, GCC at `-Og` may CSE the read across the ISR boundary. Confirmed required on Cortex-M0+ STM32G0.
- **Don't move state mutations into the ISR** — keep them in the main-loop setters. The ISR's only job is to read state and emit a frame. If you accidentally start writing `CurrentPowerLevel` inside the ISR, you've reintroduced multi-writer state which is what we just fixed.
