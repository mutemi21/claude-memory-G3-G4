---
name: features
description: Per-project record of completed features and bring-up milestones. Append new entries; do not rewrite history.
type: project
originSessionId: 0f927519-2240-4f06-ab0d-226cc3a19677
---
## G4-LightBoard-BringupFW — full bring-up Steps 1-5 + flash JEDEC test
- **Product:** G4 (G4-LightBoard-BringupFW)
- **Date:** 2026-05-26
- **Branch:** `master` (commit `5f07a4e`, pushed to `origin/master`)
- **Files changed:** `Core/Src/main.c` (final cleanup commit). The full feature also touched `Core/Src/rtc.c`, `Core/Src/gpio.c`, `Core/Src/usart.{c,h}`, `Core/Src/main.c` SystemClock_Config, `ExternalPeripherals/RFComms/BTModemDriver/Src/ModemHw.c`, `ExternalPeripherals/RFComms/BT/Src/BTStateMachine.c`, `ExternalPeripherals/Bluetooth/Src/Bluetooth.c`, `ProductFeatures/DeviceSettings/Src/DeviceSettings.c`, `ExternalPeripherals/Led/Src/Led.c`, `ExternalPeripherals/Keypad/Src/X12ACapTouch.c` — see earlier commits on this branch.
- **Root cause / Purpose:** Build a G4 production hardware bring-up firmware mirroring the G3 equivalent. Five sequenced steps the operator can watch + a runtime stage that simultaneously exercises power-board UART and Bluetooth. Plus a flash-presence check on boot so a dead/missing W25Qxx flash is detected immediately.
- **What we tried that didn't work:**
  - Chasing the "BT silent after init" symptom through many wrong theories: flash inits drawing current, AT+TS=1 transparent-streaming persisting across resets, AT command timing, TS mode getting stuck. None of these were the cause.
  - Trying to make Step 1 (RTC) work by tweaking persistence/CheckTimeDifference logic — the RTC ops themselves were fine.
  - Inverting both BT_PWR_SW and BL_RESET polarities based on stale `gpio.c` comments — needed the actual schematic to confirm BT_PWR_SW (PB1) is P-FET gate active-LOW and BL_RESET (PA4) is direct trace active-HIGH.
  - Writing the user to the wrong BLE characteristic UUID (0xAE00 instead of the correct 0xFFF2) for LOCK/UNLOCK testing.
  - Porting the G3 inline pattern write/read-back flash test (commit `6318c3e` and follow-ups). G3 itself abandoned that in commit `b1ef82c` — JEDEC-ID check inside `W25qxx_Init` is sufficient.
- **Final solution:**
  1. **Boot-time BT power gate (load-bearing fix).** First action in `USER CODE BEGIN 2` in [Core/Src/main.c](Core/Src/main.c) is `HAL_GPIO_WritePin(BT_PWR_SW_GPIO_Port, BT_PWR_SW_Pin, GPIO_PIN_SET)` — keeps the HLK-B25 unpowered until `Step5_BtStartup()`'s `ModemHw_PowerOn` so the module doesn't run autonomously through pre-BT init / Steps 1-4 delays and drift into a non-AT-responsive state. This is what made every other symptom go away.
  2. **Step sequence (in `main` after init):**
     - `Step1_RtcTest()` — sets RTC to `26/02/15,12:34:00` via `UserRtc_ParseNewTime`, waits 5 s, reads back. Target time chosen to exceed `UserRtc_ParseNewTime`'s internal 120 s `CheckTimeDifference` tolerance vs any plausible already-set value, otherwise the write is silently skipped. RTC source is LSE 32.768 kHz crystal (set in `rtc.c` MspInit + `SystemClock_Config` LSE oscillator config).
     - `Step2_VisualTest()` — `AllLedsOn()` + `DisplayTest_EverythingOn()`, hold 5 s for operator inspection. Note: `LED_IND_BLUE` and `LED_IND_RED` are active-LOW, other LEDs active-HIGH — `Led.c` polarity was fixed for this.
     - `Step3_KeypadFeedbackTest()` — press each key, display increments. Required enabling the `PAY_BUTTON` case at address `0x8000` in `X12ACapTouch.c` (was stubbed with a "dummy" comment).
     - Buzzer-stop hooks (`UIView_StopEndOfCookingBeeps()` + `PowerBoard_BuzzerOff()`) — known not fully effective; see Status/outstanding.
     - `Step4_PowerDown()` — turn off all LEDs + display to reduce current draw for BT.
     - `Step5_BtStartup()` — `ModemHw_PowerOn` toggles BT_PWR_SW LOW then pulses BL_RESET HIGH→LOW with the timing in `ModemHw.c`. BT stack inits, advertises, LOCK/UNLOCK works over BLE characteristic `0xFFF2` (service `0xFFF0`).
  3. **Flash test (G3-style, JEDEC-only).** Just before `DebugUart_Init()` in `USER CODE BEGIN 2`:
     ```c
     DebugOut("--- Flash test: probing SPI flash via JEDEC ID ---\n");
     if (W25qxx_Init()) {
         DebugOut("Flash test PASS: SPI flash initialised successfully.\n");
     } else {
         DebugOut("Flash test FAIL: SPI flash init failed (JEDEC ID not recognised).\n");
     }
     ```
     `W25qxx_Init` reads the chip's JEDEC ID over SPI and returns `bool`. No pattern write/read-back — G3 evolved through that and abandoned it in commit `b1ef82c`.
  4. **Critical supporting fixes in BT path (from earlier on this branch):**
     - `Gsm_Tick` in `BTStateMachine.c:300` now calls `GsmUart_Tick()` — without it, BT UART RX frames are never drained and BT stays "silent" even when the module is fine.
     - `Bluetooth.c` gained `COMMAND_MESSAGE_ID (0x06)` + sub-IDs `0x01`/`0x02` for UNLOCK/LOCK, plus a `HandleCommandMessage` handler.
     - `DeviceSettings.c` got a `DEFAULT_BLUETOOTH_DEVICE_ID "DEVICE_UNDER_TST"` fallback when `GetPaygoSerial` checksum fails (otherwise BT advertises with no name).
     - `usart.c` gained a `BleUart_SendString` wrapper required at link time.
     - SystemClock_Config: added `HAL_PWR_EnableBkUpAccess()`, `__HAL_RCC_LSEDRIVE_CONFIG(RCC_LSEDRIVE_LOW)`, and LSE in `RCC_OscInitStruct.OscillatorType`.
  5. **Runtime stage (while-loop).** All subsystems run unconditionally: `IndCooker_Process / UIPresenter_Process / UIView_Process / Keypad_Process / Bluetooth_Churn / Gsm_Churn` in the main loop, plus `Gsm_Tick / IndCooker_Tick / Led_Tick / UIPresenter_Tick` on `TickFlag`. This is the simultaneous power-board + BT runtime test the bring-up is meant to prove.
  6. **Cleanup before commit.** The `BRINGUP_BT_ISOLATION` debug switch (a bisect tool) was removed entirely along with its three `#if` blocks; comments rewritten to describe the established fix rather than an in-progress bisect.
- **Verification:**
  - Flash on G4 hardware. Console shows:
    - "Flash test PASS: SPI flash initialised successfully."
    - Step 1 sets `26/02/15,12:34:00`, after 5 s reads ~`26/02/15,12:34:05` (verified ±0 s drift on LSE).
    - Step 2 lights all 6 LEDs + every segment for ~5 s.
    - Step 3 increments display on each keypress (14 keys including PAY).
    - Step 4 turns everything off.
    - Step 5 BT comes up, advertises as the configured device name; phone connects and LOCK/UNLOCK on characteristic `0xFFF2` toggles the GUI/state correctly.
    - Main loop continues running power-board + BT in parallel.
- **Status:** Verified.
- **Outstanding:** Buzzer continues to beep after Step 3 despite `UIView_StopEndOfCookingBeeps()` + `PowerBoard_BuzzerOff()`. The htim6-driven `KeypadEvents_MsTick` path re-arms via UIView keypress callbacks. Not investigated — deferred.

### G3-vs-G4 hardware gotchas locked in by this bring-up
- **BT_PWR_SW (PB1) — P-FET (NTR4502PT1G) high-side gate, active-LOW.** MCU LOW = module powered. Pin must be HIGH at boot to keep module off through pre-BT init.
- **BL_RESET (PA4) — direct trace, active-HIGH.** MCU HIGH = reset asserted. Pulse HIGH 100 ms, then LOW, then wait 200 ms before AT commands.
- **`LED_IND_BLUE` and `LED_IND_RED` are active-LOW**, unlike the other four LEDs on the board which are active-HIGH. Polarity fix lives in `Led.c`.
- **`PAY_BUTTON` keypad address is `0x8000`** on this hardware. The previous bring-up had stubbed it with a "dummy" comment.
- **HLK-B25 service UUID `0xFFF0`, write characteristic `0xFFF2`** for the LOCK/UNLOCK protocol (byte 0 = `0x06` = `COMMAND_MESSAGE_ID`, byte 1 = `0x02` LOCK / `0x01` UNLOCK).

## G3-R3-BringUp — G071->G0B0 MCU migration + full Steps 1-6 bring-up
- **Product:** G3 R3 board (new revision). Repo: github.com/ecoa-dev-team/G3-R3-BringUp. Started by cloning AddingMoreTests branch from G3-LightBoard-BringupFW as master.
- **Date:** 2026-06-03
- **Branch:** `master` (commit `8cfca92`, pushed)
- **Root cause / Purpose:** R3 board replaces STM32G071CBT6 (R2's MCU) with STM32G0B0CETx (more flash, more RAM, USB/UCPD pins). Need a clean bring-up firmware that exercises Steps 1-5 (RTC, LEDs+segments, keypad, power-down, BT) plus a new Step 6 BLE LOCK/UNLOCK verification prompt, with consistent debug logging.

### Final firmware shape
1. **MCU migration done surgically** — see bugfixes entry "G3-R3 BringUp -- surgical G071->G0B0 migration on already-regenerated subfolder". CubeMX wouldn't honor Generate Under Root; used its misplaced output as a donor source for chip-specific files.
2. **R3-specific GPIO changes:**
   - PA9 = `TOUCH_PWR_SW` (output) — new, replaces GPS USART1_TX on R2.
   - PA12 = `LED_CTRL_PAY` (output) — single PAY LED on R3 (R2 had PA11). PA11 left unused; future RGB board will add green/blue.
   - PB2 = `LED_CTRL_INFO` — preserved from R2.
   - Led module enum: `INFO_PAY_LED` renamed to `INFO_LED`; new `PAY_LED` enum drives PA12.
3. **BT path (load-bearing fix preserved from G4):**
   - First action in USER CODE BEGIN 2: `HAL_GPIO_WritePin(BT_PWR_SW_GPIO_Port, BT_PWR_SW_Pin, GPIO_PIN_SET)` so HLK-B25 stays unpowered until Step 5.
   - `Gsm_Tick()` in `BTStateMachine.c` calls `GsmUart_Tick()` first.
   - `Bluetooth.c` has `COMMAND_MESSAGE_ID 0x06` + LOCK/UNLOCK sub-IDs + `HandleCommandMessage`.
   - `DeviceSettings.c` has `DEFAULT_BLUETOOTH_DEVICE_ID = "860626040000331"` fallback when `GetPaygoSerial` checksum fails.
4. **LSE for RTC.** `SystemClock_Config` has `HAL_PWR_EnableBkUpAccess() + __HAL_RCC_LSEDRIVE_CONFIG(RCC_LSEDRIVE_LOW)` and `OscillatorType = HSE|LSE` with `LSEState = LSE_ON`. `rtc.c` `RTCClockSelection = RCC_RTCCLKSOURCE_LSE`. LSI removed.
5. **Production hardware test (Steps 1-6 in main.c):**
   - **Step 1 — RTC:** sets clock to `26/01/01 00:00:00`, waits 5 s, reads back `26/01/01 00:00:05` exactly (LSE accurate to 0 s).
   - **Step 2 — Visual:** `Led_AllOn()` + `Display_SetLEDs(true)` + `DisplayTest_AllSegmentsOn(5000)`. `DisplayTest_AllSegmentsOn` Pass 2 (driver-mapped 8888888 + icons) is `#if 0`-disabled — only the raw all-FFs pass runs. Holds for 5 seconds.
   - **Step 3 — Keypad:** `KeyPadFeedBackTest()` with operator prompt "PRESS EACH KEY ONCE. Display counts up as keys are detected."
   - **Step 4 — Power down:** all LEDs off, display cleared. Required before Step 5 because power-board can't drive both UI + BT simultaneously.
   - **Step 5 — BT bring-up:** `GsmUart_Reset() + BTStateMachine_Init() + BTStateMachine_TurnOn()`. AT init runs through `AT+TS=1`. Device advertises as `EC_0000331`.
   - **Step 6 — BLE LOCK/UNLOCK prompt:** main loop polls `BTStateMachine_IsReady()` (reads `CurrentGsmState->StateID` directly — direct query, not a SetSequence hook), then waits 500 ms for AT response print pipeline to flush, then prints the operator-facing nRF Connect instructions. Writes `06 02 ...` to characteristic `0xFFF2` -> LOCK; `06 01 ...` -> UNLOCK.
6. **Debug log conventions established:**
   - `[Flash]` prefix for `w25qxx.c` flash detection lines (4-line summary: Probing / JEDEC ID / Chip / Capacity / PASS).
   - `[Step N]` prefix for bring-up step messages, with blank-line separators between steps.
   - ASCII dashes (`--`) not em-dashes (`—`) to avoid CP-1252 mojibake in serial terminals.
   - `DebugUart_Init()` must be the FIRST call in USER CODE BEGIN 2 — see bugfixes entry.

### Verification
- Clean CMake build (RAM 11.76% / FLASH 21.13% of G0B0CETx).
- Boot log shows full Step 1-6 sequence with operator prompts at correct moments.
- Step 5 AT init completes: `AT+VER=?` -> module identity `HLK-B25(b.2.16.120250506103252)`, then `AT+NAME=EC_0000331`, `AT+RFPOWER=10`, `AT+SLEEPEN=1`, `AT+CONNI=12`, `AT+ADVI=800`, `AT+ADVDATA=68696C696E6B`, `AT+ROLE=1`, `AT+UUIDS=fff0`, `AT+UUIDR=fff1`, `AT+UUIDW=fff2`, `AT+TS=1` — each followed by `OK`.
- Step 6 prompt prints cleanly as the last block.
- nRF Connect on phone: device shown as `EC_0000331`, connect succeeds, write `0x06 02 ...` to FFF2 -> `Received lock command` on Debug UART; write `0x06 01 ...` -> `Received unlock command`.

### Status
**Verified end-to-end. Production-ready bring-up firmware for the R3 board.**

### Open follow-ups
- `.ioc` PA11 still labeled `GREEN_LED_CTL_PAY` even though main.h and gpio.c dropped it. Next CubeMX regen will re-emit. Either clear the label in CubeMX or accept the regen churn.
- `.cproject` references G071 in 4 spots (Debug + Release × MCU value + linker path). Doesn't affect the CMake build flow used here; only matters if someone opens the project in STM32CubeIDE.
- Display Pass 2 driver-mapped test is `#if 0`-disabled in `DisplayTest_AllSegmentsOn`. Pass 1 (raw all-FFs) alone is sufficient as a production all-segments check; Pass 2 was useful only for driver-mapping diagnostics during early bring-up. Keep disabled unless changing the segment map.
- `Stuck timeout: 6` line appears in the log before each AT response, same off-by-one in `AtTimeoutCount` as documented in G4 bugfix. Non-blocking, deferred.
