---
name: bugfixes
description: Per-project record of bugs found and how they were fixed. Append new entries; do not rewrite history.
type: project
originSessionId: 05e5f167-f52e-4896-b977-9acdd1c7733c
---
## STM32G071 -> STM32G0B0 migration + LSE/MCO bringup
- **Product:** G3 (LightBoard bringup repo). Pattern also applies to any G07x/G08x -> G0B0/G0B1/G0C1 migration in other repos — see WIP note at end.
- **Date:** 2026-05-25
- **Branch:** `LSE-Crystal-Bringup`
- **Files changed:** `Core/Src/main.c`, `Core/Src/rtc.c`, `Core/Src/gpio.c`, `Core/Src/stm32g0xx_it.c`, `Core/Inc/stm32g0xx_hal_conf.h`, `ExternalPeripherals/Led/Src/Led.c`, `cmake/gcc-arm-none-eabi.cmake`, `cmake/stm32cubemx/CMakeLists.txt`, `.cproject`, `STM32_FIRMWARE_G3.ioc`, all CubeMX-regenerated peripheral files, plus a brand-new linker script `STM32G0B0XX_FLASH.ld` and startup `Core/Startup/startup_stm32g0b0xx.s`.
- **Root cause / Purpose:** Swap the soldered MCU from STM32G071CBT6 (128 KB / 36 KB, no MCO2) to STM32G0B0CETx (512 KB / 144 KB, has MCO2) so we can output the HSE main crystal on PB2 and verify it on a scope. Also enable the LSE 32.768 kHz crystal as the RTC source and output it on PA2 via LSCO. Bluetooth (USART2 on PA2/PA3) is disabled during bringup since PA2 is reused as LSCO.

### Pre-flight gotchas to know before regenerating

1. **CubeMX may write a fresh project to a subfolder instead of regenerating in place.** First regen produced `G3-LightBoard-BringupFW/G3-LightBoard-BringupFW.ioc` with a fresh empty Core/ tree. Cause: Project Manager → Project Name didn't match the original (`STM32_FIRMWARE_G3`), and/or "Generate Under Root" was unchecked. Fix before regen: in **Project Manager**, set Project Name = original (`STM32_FIRMWARE_G3`), Project Location = parent folder, **Generate Under Root** = checked, Application Structure = Advanced.
2. **CubeMX rewrites the `.ioc` filename when you save.** The toolchain dropdown also changed from `STM32CubeIDE` to `CMake` on first save (which CubeMX picked up from the .vscode CMake setup). Result: a new `.ioc` filename + only `cmake/stm32cubemx/CMakeLists.txt` is maintained, no longer the `.cproject`. We renamed the new ioc back to the original name and also patched ProjectName/ProjectFileName fields inside the ioc to match.
3. **Always set `Toolchain Folder Location` to project root** so generated files land beside the original ones.

### What CubeMX got wrong / failed to carry over (most important section)

These are the issues that bit us in this exact order. Watch for the same pattern in other repos:

#### 1. Shared UART IRQ name changed -> empty handler -> CPU wedged
- G07x: `USART3_4_LPUART1_IRQHandler` (handles USART3+4+LPUART1)
- G0B0: `USART3_4_5_6_IRQHandler` (handles USART3+4+5+6, **no LPUART1** on G0B0)
- G0B1/G0C1: `USART3_4_5_6_LPUART1_IRQHandler`

CubeMX produced an **empty** new handler in `stm32g0xx_it.c`. The original project had a custom IRQ body that dispatched to `DebugUart_IrqHandler()` (USART4) and `PowerBoardUart_IrqHandler()` (USART3), defined in `usart.c`. Both `*Init()` functions enable RXNEIE on their UART. As soon as any byte arrived on RX (terminal noise, normal traffic), the IRQ would fire, do nothing, return, and re-fire forever because RXNE was never cleared.

Symptom: firmware froze somewhere inside the first `DebugPrint`. Debugger pause showed the call stack stuck at the empty IRQ handler.

Fix: inside `/* USER CODE BEGIN USART3_4_5_6_IRQn 0 */` block, add `PowerBoardUart_IrqHandler(); DebugUart_IrqHandler();`. Each handler reads its own UART's ISR/RDR which clears the flag.

**For other repos:** grep for `*_IrqHandler` in the app's `usart.c` (or similar) to find any custom UART handlers. Find the shared IRQ name in the new chip's `stm32g0xx_it.c`. Wire each app handler into the USER CODE block of the new IRQ.

#### 2. PB2 (MCO2) reconfigured as GPIO by `Led_Init()`
`Led_Init()` (in `ExternalPeripherals/Led/Src/Led.c`) calls `MX_GPIO_Init_InfoLED()`, which lives in a USER CODE block in `gpio.c` and reconfigures PB2 as `GPIO_MODE_OUTPUT_PP` for the LED_CTRL_INFO LED. This overrides the `GPIO_AF3_MCO2` mode that `MX_GPIO_Init()` set. After `Led_Init()` the GPIO mux is GPIO, not AF, so MCO2 internally toggles but nothing reaches the pad.

Symptom: PA2 (LSCO) showed clean 32.768 kHz, but PB2 (MCO2) showed only "some activity" — the brief AF state during init, then GPIO writes from Led code.

Why LSCO worked but MCO didn't: **LSCO bypasses the GPIO mux** on STM32G0 (similar to OSC pins). MCO does NOT — it requires AF mode on the pin.

Fix: comment out the `MX_GPIO_Init_InfoLED()` call in `Led_Init()` (preserve the function definition for later). Done as a temporary bringup measure since LED_CTRL_INFO and MCO2 conflict on PB2.

**For other repos:** any time you repurpose a GPIO-labelled pin to an AF (MCO, LSCO-only-mux pins, USART AF, etc.), grep for the pin's label macro and confirm nothing else calls `HAL_GPIO_Init` on it after `MX_GPIO_Init`. CubeMX won't catch this because the conflicting init lives in USER CODE.

#### 3. MCO2 source defaulted to SYSCLK (PLL/64 MHz), not HSE (8 MHz)
The `.ioc` doesn't have an explicit MCO2 source line by default, so CubeMX generates `HAL_RCC_MCOConfig(RCC_MCO_PB2, RCC_MCO2SOURCE_SYSCLK, RCC_MCO2DIV_1)`. Watching PB2 then shows the PLL output, not the actual crystal — defeating the purpose of using MCO to verify the crystal.

Symptom: would have been a 64 MHz wave (or distorted slew at low GPIO speed) on PB2 instead of 8 MHz. We caught this before flashing only by reading the generated code.

Fix (code-level): change SystemClock_Config's MCO call to `RCC_MCO2SOURCE_HSE`. Fix (.ioc-level, for persistence across regens): in CubeMX RCC config -> MCO2 Source = HSE.

#### 4. RTC auto-initialised the clock every boot, fighting the marker-based persist logic
After regen, CubeMX placed `HAL_RTC_SetTime(...)` and `HAL_RTC_SetDate(...)` calls in `MX_RTC_Init()` outside any USER CODE block. These run on every boot and reset the time to 00/01/01 00:00:00. Combined with our marker check in `main.c` (which skips setting RTC if BKP_DR1 == 0x32F2), this would have read back 00:00:00 instead of the persisted time.

Fix: in CubeMX RTC config, **un-check "Activate Calendar"** so only "Activate Clock Source" stays checked. Then `MX_RTC_Init` no longer contains the SetTime/SetDate calls. User did this in CubeMX, regenerated, and the calls disappeared.

#### 5. USER CODE WHILE / 3 block in `main.c` got wiped
After one of the regens, the entire body of the main loop disappeared — `Keypad_Process()`, `UIPresenter_Process()`, `UIView_Process()`, the `if (TickFlag) { IndCooker_Tick(); UIPresenter_Tick(); ... }` block, and the 5-second RTC printout. Other USER CODE blocks in the same file survived (USER CODE Init, USER CODE 2, USER CODE 4, etc.) — so this wasn't a global "user code wiped" event, just this specific block.

Suspected cause: at some point during the regen back-and-forth, the user opened CubeMX from a different working copy of the .ioc and saved/generated. CubeMX's preservation of USER CODE blocks depends on matching block IDs between old and new generated files; if the IDs differ, content gets discarded.

Symptom: firmware booted, ran through all init, then sat in an empty `while(1) {}`. Only the BT LED was lit (its state was set during init). Keypad, RTC print, IndCooker - all silent.

Fix: hand-restore the loop body in `main.c`. To prevent recurrence: **always commit before any CubeMX regen**, and after each regen diff `main.c` against the prior commit specifically looking at the WHILE / 3 blocks.

#### 6. `USE_HAL_TIM_REGISTER_CALLBACKS` reset to 0 by HAL conf regen
The original project used `HAL_TIM_RegisterCallback()` in four places (Keypad.c, CHKDriver.c, HighwayDriver.c, UIView.c). That HAL feature requires `#define USE_HAL_TIM_REGISTER_CALLBACKS 1u` in `stm32g0xx_hal_conf.h`. CubeMX regen reset this to `0u`, so the build failed with `implicit declaration of HAL_TIM_RegisterCallback` and `'HAL_TIM_PERIOD_ELAPSED_CB_ID' undeclared`.

Fix (one-shot): edit `Core/Inc/stm32g0xx_hal_conf.h` and set the flag to `1u`. Fix (persistent across regens): in CubeMX Project Manager -> Advanced Settings -> "Register Callback" column for TIM -> ENABLE.

#### 7. Startup file location depends on toolchain choice
With `Toolchain = CMake` in the .ioc, CubeMX places `startup_stm32g0b0xx.s` at the **project root** and `cmake/stm32cubemx/CMakeLists.txt` references it there. With `Toolchain = STM32CubeIDE`, the startup goes to `Core/Startup/` and is discovered via the CubeIDE source folders list.

To support both build flows simultaneously (CMake from terminal/VSCode + STM32CubeIDE GUI), we moved the startup file to `Core/Startup/` and patched `cmake/stm32cubemx/CMakeLists.txt` to point to the new path. This breaks on the next regen with CMake toolchain — CubeMX will recreate the file at root and overwrite the CMakeLists.txt reference. Re-do the move + patch after each regen, OR set toolchain to STM32CubeIDE in .ioc (loses auto-CMake regen).

#### 8. Linker script + `.cproject` reference G07x parts
- `cmake/gcc-arm-none-eabi.cmake` hand-references `STM32G071CBUX_FLASH.ld`. CubeMX produces a new linker script with a different name (`STM32G0B0XX_FLASH.ld` for G0B0CET) — must be hand-patched.
- `.cproject` (STM32CubeIDE) has G071 references in multiple places: MCU value `STM32G071CBUx`, preprocessor define `STM32G071xx`, linker path `STM32G071CBUX_FLASH.ld`. All appear twice (Debug + Release configs). Hand-patch each occurrence to the G0B0 equivalents.

#### 9. Eclipse project name mismatch between `.project` and `.cproject`
`.project` had `<name>STM32_FIRMWARE_G3</name>`. `.cproject` had build paths like `${workspace_loc:/G3-LightBoard-BringupFW}/Debug` (using the folder name instead of the project name). Eclipse couldn't resolve the build output dir on import. Fix: replace `G3-LightBoard-BringupFW` -> `STM32_FIRMWARE_G3` in `.cproject`.

#### 10. PB2 LQFP-48 conflict — info LED vs MCO2
PB2 was assigned to `LED_CTRL_INFO` in the original board design (defined in `main.h` USER CODE Private defines + `gpio.c` USER CODE 2 `MX_GPIO_Init_InfoLED`). It's also the **only LQFP-48 MCO2-capable pin** on G0B0 that's still free (PA8 is `LED_CTRL_LOCK`, PA9/PA10/PA15 are available alternatives if you can't sacrifice the info LED). On G071 PB2 was NOT MCO-capable at all (G071 has no MCO2), which is why it was free to be used as a GPIO — this only became a conflict after the G0B0 migration.

### What we tried that didn't work

- **First regen attempt:** CubeMX put everything in `<root>/G3-LightBoard-BringupFW/` as a fresh empty project. Caused by wrong Project Name + missing Generate Under Root tick. Deleted the subfolder and re-did Project Manager settings before re-saving.
- **Trying to build with VSCode CMake Tools before installing STM32CubeCLT:** failed with "ninja.exe not found" at `C:/ST/STM32CubeCLT_1.19.0/...` because only STM32CubeIDE was installed. Workaround was terminal builds with full PATH override. Real fix: install STM32CubeCLT (latest, currently 1.21.0).
- **Trying to verify with debugger before ST-Link driver:** "No debug probe detected" / "error 138". Fixed by checking USB connection and driver.
- **Looking at PB2 thinking the HSE crystal was bad:** spent time worrying about the crystal until we realised `Led_Init()` was overriding the AF mux. The crystal was fine all along — the firmware path was wrong.

### Final solution (concrete steps to reproduce on another repo)

Assuming starting state: G07x project on a board, want to migrate to G0B0 (or G0B1/G0C1) with similar features.

1. **Commit current work** on a fresh branch (`git checkout -b <chip>-migration`).
2. **In CubeMX, before changing device:**
   - Project Manager: confirm Project Name matches existing `.ioc`, Project Location = parent folder, Generate Under Root = checked, Application Structure = Advanced.
   - Project Manager -> Advanced Settings: enable "Register Callback" for any peripheral the app uses callbacks on (TIM at minimum).
3. **Change device:** File -> Change Device -> new G0B0/G0B1/G0C1 part number matching package.
4. **Reconfigure pins/clocks:**
   - RCC: enable HSE Crystal/Resonator, LSE Crystal/Resonator. Set LSCO source = LSE, MCO2 source = HSE (explicitly — default is SYSCLK).
   - Clock tree: RTC mux = LSE.
   - RTC: enable Clock Source, **disable Activate Calendar** if you have app-level RTC set logic.
   - Disable any USART that was on PA2 (now needed for LSCO).
5. **Generate Code.**
6. **Hand-fix after regen:**
   - `Core/Src/stm32g0xx_it.c`: wire app's `*Uart_IrqHandler()` calls into the new shared USART IRQ USER CODE block. New IRQ name on G0B0 is `USART3_4_5_6_IRQHandler`; on G0B1/G0C1 it's `USART3_4_5_6_LPUART1_IRQHandler`.
   - `Core/Inc/stm32g0xx_hal_conf.h`: confirm `USE_HAL_TIM_REGISTER_CALLBACKS = 1u` (should be permanent if you ticked it in CubeMX, but check).
   - `cmake/gcc-arm-none-eabi.cmake` (if you use CMake): update linker script reference (`STM32G07*XX_FLASH.ld` -> `STM32G0B0XX_FLASH.ld` or equivalent).
   - `.cproject` (if you use CubeIDE): replace MCU value, preprocessor define, linker script path. All appear twice (Debug + Release). Also fix any `workspace_loc:/<wrong-name>` references to match `.project` name.
   - Move `startup_stm32g0b*xx.s` to `Core/Startup/` and patch the CMakeLists.txt reference if you want both CMake and CubeIDE to work.
   - For any pin reassigned from GPIO to peripheral AF, grep the codebase for the pin's label and check no USER CODE `HAL_GPIO_Init` overrides it.
7. **Build and confirm no errors.** Watch for `implicit declaration of HAL_*` — that means a HAL feature flag is off.
8. **Flash and step through main()** to confirm nothing hangs in init. Watch IRQs especially.

### Verification

- **Build:** RAM 6.6% / FLASH 16.2% of G0B0CETx (was 26% / 65% on G071CBT6).
- **Scope PA2:** clean 32.768 kHz square wave (LSE -> LSCO).
- **Scope PB2:** clean 8 MHz square wave (HSE -> MCO2). After Led_Init patch.
- **Debug log on USART4:** init messages + `[LSE] RTC: yy/mm/dd,HH:MM:SS` every 5 seconds.
- **Debugger:** call stack does not get stuck in `USART3_4_5_6_IRQHandler` after the IRQ dispatch fix.
- **RTC persistence:** reset the board with NRST (not power-cycle unless VBAT is wired); time should continue ticking from where it left off.

### Status

**WIP.** Verified on the G3 LightBoard with STM32G0B0CETx soldered.

**Still applies to:**
- Any other repo using G07x/G08x that needs to migrate to G0Bx/G0Cx. Specifically check:
  - G3-STM32-FIRMWARE-G3 (full G3 firmware) if it's still on G071
  - G4-FW-STM32-FIRMWARE-G4 (G4 firmware) — different chip family, but the CubeMX workflow gotchas (USER CODE preservation, MCO2 source default, RTC auto-set, USE_HAL_TIM_REGISTER_CALLBACKS flag) apply identically
  - G4-LightBoard-BringupFW — same family situation
  - Any custom board firmware that uses MCO2 or LSCO

When migrating each of these, expect to repeat the same fix-after-regen ritual until ST changes how CubeMX preserves user customisations (don't hold your breath).

**Open follow-ups in this repo:**
- LED_CTRL_INFO on PB2 is disabled in `Led_Init()` while MCO2 is needed. Re-enable for production once crystal verification is done — either by uncommenting the call (loses MCO2) or by moving LED_CTRL_INFO to a different pin (PD0/PD8/etc are free on LQFP-48).
- The `.ioc` still has MCO2 source unset (defaults to SYSCLK). Patched manually in `SystemClock_Config()` to HSE. Fix permanently in CubeMX RCC config.

---

## G4 BRING-UP — Bluetooth (HLK-B25) silent on UART despite advertising
- **Product:** G4 — specifically the **G4-LightBoard-BringupFW** repo (ecoa-dev-team/G4-LightBoard-BringupFW). Hardware is the new G4 light board with STM32G071CBUx + HLK-B25 BLE module.
- **Date:** 2026-05-21
- **Branch:** `master`
- **Files changed:**
  - `ExternalPeripherals/RFComms/BTModemDriver/Src/ModemHw.c` (pin polarities)
  - `ExternalPeripherals/RFComms/BT/Src/BTStateMachine.c` (add `GsmUart_Tick()` to `Gsm_Tick`)
  - `ProductFeatures/DeviceSettings/Src/DeviceSettings.c` (add `DEFAULT_BLUETOOTH_DEVICE_ID` fallback in `DeviceSettings_GetPaygoSerial`)
  - `ExternalPeripherals/Bluetooth/Src/Bluetooth.c` (add `COMMAND_MESSAGE_ID 0x06` + `HandleCommandMessage` for LOCK/UNLOCK)
  - `Core/Src/main.c` (Step 5 BT startup wiring + `BRINGUP_BT_ISOLATION` compile switch)

### Root cause / Purpose
The G4 bring-up firmware was derived from `STM32_FIRMWARE_G4 @ Rendering-Grouped-Power-Levels` (a thinner branch with no BT code) and we pulled the BT stack (`Bluetooth/`, `RFComms/BT/`, `RFComms/BTModemDriver/`, `Token/`, `Encryption/`, `Shadow/`, etc.) wholesale from `STM32_FIRMWARE_G4 @ Develop`. With all that wired into the bring-up's existing test harness (Steps 1–4 + cooker/keypad/UI), the HLK-B25 booted and advertised over BLE just fine, but:
1. The firmware sent `AT+VER=?` repeatedly with no response.
2. NRFConnect writes to the configured write characteristic never produced `Received lock/unlock command` on the Debug UART.

Four distinct issues had to be fixed before LOCK/UNLOCK worked end-to-end:

#### Issue 1 — `ModemHw_PowerOn` polarity vs G4 schematic
Develop's `ModemHw.c` assumed a particular polarity (`BT_PWR_SW` LOW = power, `BL_RESET` HIGH→LOW pulse) that's *exactly inverted* from what `Core/Src/gpio.c` comments on G4 suggested, and neither of them actually matched the schematic. After tracing the schematic:
- **`BT_PWR_SW` (PB1) is active-LOW.** It drives the gate of P-FET `Q400` (NTR4502PT1G high-side load switch) via `R404` 100R, with `R403` 100k pull-up to +3V3. MCU LOW → P-FET ON → `BL_VDD` powered.
- **`BL_RESET` (PA4) is a direct trace** to the HLK-B25 RST pin. Per the HLK-B25 datasheet ("Pull high >4s to reset"), RST is active-HIGH at the module. So MCU HIGH = reset asserted, LOW = running.
- Conclusion: the G3 sequence (`PWR_SW` LOW, then `RESET` HIGH→LOW pulse) is actually correct for the G4 wiring. The `gpio.c` comments ("PB1 LOW - BLE power off", "PA4 HIGH - BLE module not held in reset") are **stale and wrong**; ignore them.

The DebugOut in `ModemHw_PowerOn` documents the correct polarities now.

#### Issue 2 — Missing `GsmUart_Tick()` at the top of `Gsm_Tick`
G3 bring-up had a commit `96c92c6bc5a7b647a3ea5d02243019cf369ac5c1` ("Add GsmUart_Tick call in Gsm_Tick for improved UART handling") that adds `GsmUart_Tick();` as the very first line of `Gsm_Tick`. **This commit was never picked up into `STM32_FIRMWARE_G4 @ Develop`**, so when we pulled the BT stack from there, we inherited the broken version. Without `GsmUart_Tick()`, the partial UART frames from the HLK-B25 never get assembled into complete events. AT responses *do* arrive at USART2 RX, but `Gsm_Churn` never sees them because the tick-driven RX timer never fires. End result: the state machine retries `AT+VER=?` forever.

Fix: port the single-line change from G3 into G4's `BTStateMachine.c`. **This is the most important fix** — without it, even the AT init sequence can't complete.

#### Issue 3 — `DeviceSettings_GetPaygoSerial` returns false on unprovisioned boards
G3's bring-up version of `DeviceSettings_GetPaygoSerial` *always* returns the hard-coded `DEFAULT_BLUETOOTH_DEVICE_ID` and *always* returns true (it bypasses flash/checksum entirely). G4's version checks the flash checksum and returns false on mismatch.

On a fresh G4 light board (no PaygoSerial provisioned), G4's `GetPaygoSerial` returns false → `AtSender_SendSetBtNameCommand` bails with `Failed to get bluetooth device name` → returns -1 → state machine treats this command as failed → falls back to `GSM_STATE_ID_BLUETOOTH_RESET` → retries forever, never reaches `BLUETOOTH_CONNECT` / `BLUETOOTH_RECEIVE` (transparent-transmission mode). So even with Issue 2 fixed, the module never gets configured for transparent passthrough; BLE writes from the phone don't reach the MCU UART.

Fix: in G4's `DeviceSettings.c`, when the flash PaygoSerial checksum doesn't match, fall back to `#define DEFAULT_BLUETOOTH_DEVICE_ID "DEVICE_UNDER_TST"` and return true. Mirrors G3 bring-up. Side effect: device now advertises as `EC_NDER_TST` (chars 8–15 of `"DEVICE_UNDER_TST"` is `"NDER_TST"`, prefixed with `"EC_"` by `AtSender`).

Note: there's a `#define BLUETOOTH_DEVICE_ID "DEVICE_UNDER_TST"` in G3's `Bluetooth.c:30` — it's **dead code** (defined but never referenced). The actual name source is `DEFAULT_BLUETOOTH_DEVICE_ID` in `DeviceSettings.c`.

#### Issue 4 — `Bluetooth.c` had no `COMMAND_MESSAGE_ID` (0x06) handler
G3's `Bluetooth.c` has:
```c
#define COMMAND_MESSAGE_ID            0x06
#define UNLOCK_COMMAND_SUB_MESSAGE_ID 0x01
#define LOCK_COMMAND_SUB_MESSAGE_ID   0x02
```
and a `case COMMAND_MESSAGE_ID:` in `Bluetooth_DataReceivedCallback`'s switch that calls `HandleCommandMessage(Packet)` (prints `Received lock/unlock command`). **Develop dropped this** before we cloned, so the G4 bring-up Bluetooth.c only handles `AUTHENTICATION`, `COOKER_DATA`, `OTA`, `DEVICE_SHADOW`, `TIME_SYNC`. Writes with MessageID 0x06 silently fall into `default: break;`.

Fix: re-add the three defines, the case, the forward declaration, and the `HandleCommandMessage` function body. G3's `CreditManagement_SetCreditStatusOkay(true/false)` calls inside the handler can't be ported — G4's `CreditManagement` API has no setter — left as a `TODO` comment. For bring-up testing, the Debug-UART print is what we need.

#### Issue 5 — REAL root cause: BT module powered up too early at boot
**Update 2026-05-26:** the "current/peripheral interference" framing was wrong. The actual root cause is:

`Core/Src/gpio.c` drives `BT_PWR_SW` (PB1) LOW at `MX_GPIO_Init()`. PB1 is the active-low gate of the P-FET high-side load switch that controls `BL_VDD` (the HLK-B25's VBAT). So the **HLK-B25 is powered up the moment the MCU runs `MX_GPIO_Init()`**, before any of our application init runs.

Once the module is powered, it boots and starts BLE advertising on its own. After even a few seconds of being awake (somewhere ≤ 5 s — exact threshold unmeasured) it reaches an internal state where `Step5_BtStartup`'s reset pulse + first `AT+VER=?` are ignored. The persisted `AT+SLEEPEN=1` (saved to module flash from a previous successful init) is suspected to be what tips it over, but we haven't proven it.

This retroactively explains the earlier "flash inits break BT" finding: `FlashData_SessionDataBufferInitialize()` and friends do a full ringfs sector walk at boot which simply takes long enough for the BT module to enter the bad state. The flash itself wasn't the problem; the *delay* was.

**Fix (one line):** at the very top of `Core/Src/main.c` `USER CODE BEGIN 2`, drive `BT_PWR_SW` HIGH so the P-FET stays OFF and the module is unpowered until `Step5_BtStartup`'s `ModemHw_PowerOn` is the first thing to apply power.

```c
/* Keep BT module unpowered at boot. Step 5 will be its first power-on.
 * Otherwise the module runs autonomously during pre-BT init and ends
 * up in a state that won't respond to AT commands. */
HAL_GPIO_WritePin(BT_PWR_SW_GPIO_Port, BT_PWR_SW_Pin, GPIO_PIN_SET);
```

With this in place: flash inits, RTC verification, LED + segment visual test, keypad test, settle delays — all run safely before BT. The `BRINGUP_BT_ISOLATION` switch is still useful as a quick "BT-only" debug mode, but it's no longer required for BT to actually work.

The `gpio.c` comment ("PB1 LOW - BLE power off") that misled me earlier remains stale — see "G4 BT module UART RX deaf while full bring-up runs" → root cause section above for the actual schematic polarity (P-FET active-LOW gate via R404 with R403 100 k pull-up to +3V3).

#### Issue 6 — LSE-driven RTC (and Step 1 verification gotcha)
Brought up on 2026-05-26 alongside the BT_PWR_SW fix. Two parts:

- **LSE config:** `Core/Src/main.c` `SystemClock_Config` now enables LSE alongside HSE/LSI: `HAL_PWR_EnableBkUpAccess()` + `__HAL_RCC_LSEDRIVE_CONFIG(RCC_LSEDRIVE_LOW)` + `OscillatorType |= RCC_OSCILLATORTYPE_LSE` + `LSEState = RCC_LSE_ON`. `rtc.c` `HAL_RTC_MspInit` switches `RTCClockSelection` from `RCC_RTCCLKSOURCE_LSI` to `RCC_RTCCLKSOURCE_LSE`. The existing `AsynchPrediv=127` / `SynchPrediv=255` are already correct for a 32.768 kHz source (÷32768 → 1 Hz).
- **Test target time:** `Step1_RtcTest` used to set RTC to `26/01/01,00:00:00`. `UserRtc_ParseNewTime` calls `CheckTimeDifference` and *silently skips* `SetRtcTime` if the delta vs current RTC ≤ 120 s. Since the LSE-backed RTC keeps ticking across resets, after the first successful boot the RTC was always close to 00:00:0X — so subsequent boots never actually re-wrote it, and Step 1 was only verifying "RTC is ticking" rather than "RTC can be set". Changed the target to `26/02/15,12:34:00` so the delta is guaranteed > 120 s and the write always happens. Read-back of `26/02/15,12:34:05` confirms both the SetRtcTime path and accurate ±0 s LSE tick.

### What we tried that didn't work
- **Inverting BT_PWR_SW and BL_RESET polarities** (assumed gpio.c comments were correct + assumed an inverter on each line). Module still silent. The gpio.c comments were stale — schematic confirmed direct trace for RESET and active-low P-FET for PWR_SW.
- **Hard-coding the device name in `AtSender_SendSetBtNameCommand` alone** (fallback to `EC_BRINGUP` when no serial). Worked for the name but didn't unblock the rest of the init sequence cleanly. Switching the fallback to `DeviceSettings_GetPaygoSerial` is the correct fix because every other module that depends on `PaygoSerial` (Token, Shadow, etc.) also gets a consistent value.
- **Just enabling more debug flags.** Useful for visibility but not the actual fix.

### Verification
- Reflash with `BRINGUP_BT_ISOLATION=1`.
- Debug UART shows the full AT init sequence reaching `AT+TS=1` (transparent streaming enabled): `AT+VER=?`, `AT+NAME=EC_NDER_TST`, `AT+RFPOWER=10`, `AT+SLEEPEN=1`, `AT+CONNI=12`, `AT+ADVI=800`, `AT+ADVDATA=68696C696E6B`, `AT+ROLE=1`, `AT+UUIDS=fff0`, `AT+UUIDR=fff1`, `AT+UUIDW=fff2`, `AT+TS=1` — each followed by `OK`.
- BLE scan shows device named `EC_NDER_TST`.
- Connect via NRFConnect → service `0xFFF0`, write characteristic `0xFFF2`.
- Writing `0x06 02 00 00 00 00` produces on Debug UART:
  ```
  Bluetooth connection state changed: 1
  received: <ACK><STX>
  Received lock command
  ```
- Writing `0x06 01 00 00 00 00` produces:
  ```
  received: <ACK><SOH>
  Received unlock command
  ```

### Status
**Verified — LOCK/UNLOCK working end-to-end on G4 light board with `BRINGUP_BT_ISOLATION=1`.**

### Open follow-ups
- **Find the subsystem(s) that break BT when `BRINGUP_BT_ISOLATION=0`.** Suspects in order of likelihood: `Keypad_Process` (continuous I2C scans of the cap-touch IC over the same I2C bus the display uses), `IndCooker_Process` + `IndCooker_Tick` (continuous UART chatter to the cooker power board on USART3), `UIPresenter` / `UIView` (display I2C writes), `Led_Tick`. Bisect by re-enabling them one at a time.
- **`Stuck timeout: 6`** appears in the log right before each AT-init command response — non-blocking but the state machine is timing out before it gets the response. Probably an off-by-one in `AtTimeoutCount` accounting in `BTStateMachine.c`. Worth fixing for cleaner logs.
- **`Paygo Serial: DEVICE_UNDER_TSTø`** debug print in `main.c` shows a trailing junk character because the local `PaygoSerial` array is `[DEVICE_SETTINGS_PAYGO_SERIAL_LENGTH]` (16 bytes) and `strncpy` fills all 16 with no null. Trivial fix: `[LENGTH + 1]`. Cosmetic only — the actual `AT+NAME=` command uses a properly-sized buffer in `AtSender`.
- **`CreditManagement_SetCreditStatusOkay()` equivalent on G4.** G4's CreditManagement only exposes `IsCreditStatusOkay` and `UpdateDeadline`. For the LOCK/UNLOCK handler to actually do something (vs just print), a setter or equivalent path needs to be added. Currently a `TODO` in `HandleCommandMessage`.
- **`gpio.c` polarity comments are wrong.** They contradict the schematic. Should be fixed at the CubeMX `.ioc` level so the comments are regenerated correctly — at minimum, hand-fix them in `gpio.c` and add a CHANGELOG note so we don't trust them next time.
- **PAY_BUTTON address (`0x8000`).** Earlier in this bring-up we also confirmed the PAY button bitmask via NRFConnect testing. It's in `X12ACapTouch.c` (BaoLaiYa block); the dummy "Hardware issue" comment has been removed.

---

## G3-R3 BringUp — surgical G071->G0B0 migration on already-regenerated subfolder
- **Product:** G3 (G3-R3-BringUp — new repo at github.com/ecoa-dev-team/G3-R3-BringUp, cloned from G3-LightBoard-BringupFW AddingMoreTests branch as master).
- **Date:** 2026-06-03
- **Branch:** `master` (commit `8cfca92`)
- **Files changed:** `Core/Src/main.c`, `Core/Src/gpio.c`, `Core/Src/i2c.c`, `Core/Src/rtc.c`, `Core/Src/tim.c`, `Core/Src/usart.c`, `Core/Src/stm32g0xx_it.c`, `Core/Src/stm32g0xx_hal_msp.c`, `Core/Inc/main.h`, `Core/Inc/stm32g0xx_it.h`, `Core/Inc/stm32g0xx_hal_conf.h`, `cmake/stm32cubemx/CMakeLists.txt`, `cmake/gcc-arm-none-eabi.cmake` (gitignored), plus new `Core/Startup/startup_stm32g0b0cetx.s` + `STM32G0B0CETX_FLASH.ld` + regenerated `Drivers/STM32G0xx_HAL_Driver/*` from CubeMX donor folder.
- **Root cause / Purpose:** Migrate G3 firmware from STM32G071CBT6 (R2 board, G070 .ioc actually) to STM32G0B0CETx (R3 board) while preserving the AddingMoreTests bring-up harness (Steps 1-5). Goal: clean CMake build for G0B0 + retain USER CODE.

### Key trap: CubeMX "Generate Under Root" can silently fail
Even with **Project Name = STM32_FIRMWARE_G3**, **Project Location pointing at the repo itself**, **Generate Under Root = checked**, and **`ProjectManager.UnderRoot=true` confirmed in the .ioc** — CubeMX still emitted a fresh project tree into a `STM32_FIRMWARE_G3/` subfolder of the repo, fully empty USER CODE blocks. The "Toolchain Folder Location" field is a misleading read-only display that shows the would-be path with the Project Name appended regardless of the UnderRoot flag. Don't trust it.

### The fix that worked: surgical donor-file salvage (instead of re-running CubeMX)
Rather than fighting CubeMX, treat the misplaced subfolder as a **donor source**:
1. Copy chip-specific files out of the subfolder onto the existing tree (overwriting):
   - `Drivers/STM32G0xx_HAL_Driver/` (replaces G071 set)
   - `Drivers/CMSIS/Device/ST/STM32G0xx/Include/stm32g0b0xx.h` (new header)
   - `Core/Startup/startup_stm32g0b0cetx.s` (new startup)
   - `STM32G0B0CETX_FLASH.ld` (new linker -- note chip-specific filename, not the family-generic STM32G0B0XX we'd expected)
   - `.ioc` (now contains G0B0 + new GPIOs)
2. For files that touch both chip-specific stuff AND USER CODE (gpio.c, main.h), take the donor copy then sed-replace specific tokens (e.g. `LED_CTRL_INF` -> `LED_CTRL_INFO`) and hand-restore USER CODE blocks (e.g. `MX_GPIO_Init_InfoLED`).
3. For Core/Src files where USER CODE is the bulk of value (main.c, usart.c, rtc.c, etc.), DO NOT copy from donor -- apply surgical patches instead:
   - IRQ renames: `USART3_4_LPUART1_IRQHandler` -> `USART3_4_5_6_IRQHandler`, `TIM6_DAC_LPTIM1_IRQn` -> `TIM6_IRQn`, `TIM7_LPTIM2_IRQn` -> `TIM7_IRQn`.
   - `stm32g0xx_hal_msp.c` HAL_MspInit: add `HAL_SYSCFG_StrobeDBattpinsConfig(SYSCFG_CFGR1_UCPD1_STROBE | SYSCFG_CFGR1_UCPD2_STROBE)` -- G0B0 has UCPD dead-battery pins on PA11/PA12, must be detached if used as GPIO.
   - `i2c.c` HAL_I2C_MspInit I2C2 branch: add `PeriphClkInit.PeriphClockSelection = RCC_PERIPHCLK_I2C2; PeriphClkInit.I2c2ClockSelection = RCC_I2C2CLKSOURCE_PCLK1; HAL_RCCEx_PeriphCLKConfig(&PeriphClkInit);`.
   - `rtc.c` HAL_RTC_MspInit: switch `RTCClockSelection` LSI -> LSE.
   - `main.c` SystemClock_Config: prepend `HAL_PWR_EnableBkUpAccess(); __HAL_RCC_LSEDRIVE_CONFIG(RCC_LSEDRIVE_LOW);` and change `OscillatorType = HSE|LSE`, drop LSI references.
4. `cmake/stm32cubemx/CMakeLists.txt`: `STM32G071xx` -> `STM32G0B0xx`, startup path -> `${CMAKE_SOURCE_DIR}/Core/Startup/startup_stm32g0b0cetx.s`.
5. `cmake/gcc-arm-none-eabi.cmake` (gitignored, local): linker -> `${CMAKE_SOURCE_DIR}/STM32G0B0CETX_FLASH.ld` (chip-specific suffix).
6. Delete old G071 artifacts.

### Verification
- Clean CMake build (RAM 11.76%, FLASH 21.13% of G0B0CETx; was ~26%/65% on G071CBT6).
- `grep -c "PRODUCTION HARDWARE TEST\|KeyPadFeedBackTest\|BTStateMachine_TurnOn" Core/Src/main.c` returns 3+.
- Steps 1-6 verified on R3 hardware end-to-end.

### Status
**Verified.**

### Open follow-ups
- `.ioc` PA11 label `GREEN_LED_CTL_PAY` was hand-removed from main.h/gpio.c but NOT from the .ioc. Next CubeMX regen will re-emit it.
- `.cproject` still references G071 in 4 spots. Doesn't affect CMake; only CubeIDE GUI.

---

## G3-R3 BringUp — DebugUart_Init() ordering truncates flash boot log
- **Product:** G3 (G3-R3-BringUp). Pattern applies to any project using the same DebugUart driver.
- **Date:** 2026-06-03
- **Branch:** `master`
- **Files changed:** `Core/Src/main.c` USER CODE BEGIN 2 ordering, `ExternalPeripherals/Flash/Src/w25qxx.c` debug strings.
- **Symptom:** Boot log shows truncated flash debug -- e.g. `[Flash] Chip: W25Q32` prints fine but `[Flash] Capacity: 4096 KB` truncates to `[Flash] C` followed immediately by the next print like `Display_Init`. The pattern of "first few lines clean, then last line cut mid-byte followed by next module's print" is the tell.
- **Root cause:** `DebugUart_Init()` zeroes `Fifo4.tri/twi/tct/rri/rwi/rct` (in `Core/Src/usart.c:673`) -- i.e. wipes the UART4 TX FIFO state. In the original AddingMoreTests main.c init sequence, `W25qxx_Init()` was called BEFORE `DebugUart_Init()`. `W25qxx_Init` prints debug lines; those bytes get queued in `Fifo4.tbuf[]` and the IRQ-driven TX is in-flight when `DebugUart_Init()` is called next. `DebugUart_Init()` zeros the FIFO indices mid-transmission, so the partially-sent flash bytes are dropped on the floor and subsequent prints start fresh from index 0.
- **What we tried that didn't work:** Shortening the chip+capacity line from 53 chars to two shorter lines (20 + 24 chars). Truncation just moved to the new line -- the issue was FIFO state reset, not per-line length.
- **Final solution:** Move `DebugUart_Init()` to be the FIRST call in USER CODE BEGIN 2, BEFORE `W25qxx_Init()` or any `DebugOut`:
  ```c
  /* USER CODE BEGIN 2 */
      HAL_GPIO_WritePin(BT_PWR_SW_GPIO_Port, BT_PWR_SW_Pin, GPIO_PIN_SET);
      DebugUart_Init();   // MUST be before first DebugOut
      if (!W25qxx_Init()) { DebugOut("[Flash] Continuing despite flash failure...\n"); }
  ```
- **Verification:** Boot log shows all 5 `[Flash]` lines + `[Flash] PASS` intact, no truncation.
- **Status:** Verified.

---

## G3-R3 BringUp — BT state-machine: SetSequence hook misses success-path transitions
- **Product:** G3 (G3-R3-BringUp). Applies to any project using the shared `BTStateMachine.c`.
- **Date:** 2026-06-03
- **Branch:** `master`
- **Files changed:** `ExternalPeripherals/RFComms/BT/Src/BTStateMachine.c`, `ExternalPeripherals/RFComms/BT/Inc/BTStateMachine.h`, `Core/Src/main.c`.
- **Symptom:** Tried to set a `static bool BtInitComplete` flag inside `SetSequence()` when called with `GSM_STATE_ID_BLUETOOTH_CONNECT`, so main.c could poll and react to BT init completion. The flag never got set, prompt never printed.
- **Root cause:** `SetSequence(StateID)` is only called for (a) initial state setup in `ResetGsmStateSequence()`, (b) failure-path retries in `HandleFailedCommand()`. The success-path state transitions bypass `SetSequence()` entirely -- they directly assign `CurrentGsmState = GetStateInfo(CurrentGsmState->NextStateIDOnSuccess)`:
  - `BTStateMachine.c:424` inside `SendNextCommand()` when CurrentCommandIndex reaches end of a state's command sequence
  - `BTStateMachine.c:384` inside `AreNextStatesDone()` while iterating skip-conditional states
- **Final solution:** Don't hook `SetSequence`. Read `CurrentGsmState->StateID` directly:
  ```c
  bool BTStateMachine_IsReady(void) {
      if (CurrentGsmState == NULL) return false;
      GsmStateID_e id = CurrentGsmState->StateID;
      return (id != GSM_STATE_ID_BLUETOOTH_INIT && id != GSM_STATE_ID_BLUETOOTH_RESET);
  }
  ```
  Prototype in BTStateMachine.h requires `#include <stdbool.h>`.
- **Status:** Verified.

---

## G3-R3 BringUp — IsReady true != AT responses fully flushed to debug UART
- **Product:** G3 (G3-R3-BringUp).
- **Date:** 2026-06-03
- **Branch:** `master`
- **Files changed:** `Core/Src/main.c` main loop Step 6 logic.
- **Symptom:** Step 6 BLE prompt printed BETWEEN `Sent: AT+TS=1` and `received: OK` in the debug log, even though the state-machine advance only happens after AT+TS=1 OK is parsed. The prompt appeared in the middle of the AT chatter instead of at the end. Pattern: `Sent: AT+TS=1` -> [Step 6 block] -> `received: AT+TS=1` (echo) -> `received: OK`.
- **Root cause / Working theory:** The AT response handler (`At_HandleResponse` at `At.c:60`) prints `received:` lines into the UART4 TX FIFO at the same time the state-machine advances. The FIFO drains at 115200 baud (~10 kB/sec). State advance happens before all `received:` bytes have transmitted, so `BTStateMachine_IsReady()` returns true while `received:` text is still in flight. The main-loop's Step 6 check fires immediately, queuing the prompt behind the still-draining text -- but because of interleaved FIFO fill/drain over many main-loop iterations, the prompt ends up appearing before some of the AT text in the rendered output.
- **Final solution:** Add a 500 ms debounce delay between `IsReady()` becoming true and the Step 6 print. This gives the UART4 TX FIFO ample time to fully drain all in-flight AT response lines:
  ```c
  static bool Step6Printed = false;
  static uint32_t Step6ReadyTickStart = 0;
  if (!Step6Printed && BTStateMachine_IsReady()) {
      if (Step6ReadyTickStart == 0) {
          uint32_t Now = HAL_GetTick();
          Step6ReadyTickStart = (Now == 0) ? 1 : Now;
      } else if ((HAL_GetTick() - Step6ReadyTickStart) >= 500) {
          Step6Printed = true;
          DebugOut("\n[Step 6] BLE LOCK/UNLOCK test...\n");
      }
  }
  ```
- **Verification:** Step 6 prompt now lands cleanly as the last block in the boot log, after the final `received: OK`.
- **Status:** Verified. Pattern is reusable for any bring-up prompt that needs to land after a state machine completes its UART-driven init.
