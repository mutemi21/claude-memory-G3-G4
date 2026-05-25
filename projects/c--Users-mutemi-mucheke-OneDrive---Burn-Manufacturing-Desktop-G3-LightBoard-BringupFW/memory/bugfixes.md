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
