# G4 Induction Cooker - Full Codebase Map

**Product:** Burn Manufacturing G4 Induction Cooker (Paygo-enabled)
**MCU:** STM32G071CB (Cortex-M0+, 128KB Flash, 36KB RAM)
**Build:** CMake + STM32CubeMX
**Date:** 2026-03-06

## Architecture
- **MVP pattern** for UI: UIPresenter (state machine) -> UIModel (data) -> UIView (display)
- **Driver abstraction** for power boards: PowerBoard dispatches to Highway/CHK/Aifer
- **Cooperative main loop**: `_Tick()` (1ms ISR) + `_Process()` (main loop)
- **Session pipeline**: CookerState -> CookingSession -> SessionAggr -> Flash

## Data Flow
```
Keypad -> KeypadEvents -> UIView -> UIPresenter -> UIModel -> IndCooker -> PowerBoard -> [Highway/CHK/Aifer]Driver
                                                                               |
                                                     CookerState -> CookingSession -> SessionAggr -> Flash
```

---

## Active Source Files

### Core/ (CubeMX-generated, modify only in USER CODE sections)
- `Core/Src/main.c` - Entry point, super-loop, init all modules, tick/keypad callbacks
- `Core/Src/gpio.c` - GPIO pin config
- `Core/Src/i2c.c` - I2C init (Display VK16K33)
- `Core/Src/spi.c` - SPI init (W25Qxx flash)
- `Core/Src/usart.c` - UART init (PowerBoard comms + Debug)
- `Core/Src/tim.c` - Timer init (buzzer PWM)
- `Core/Src/rtc.c` - RTC init
- `Core/Src/crc.c` - CRC peripheral init
- `Core/Src/stm32g0xx_it.c` - Interrupt handlers

### ExternalPeripherals/PowerBoard/ (UART comms with power board)
- `PowerBoard.h` - Central interface: CookerPower_e, CookerError_e, callback types
- `PowerBoard.c` - Dispatcher, routes to driver by PowerBoardType (hardcoded HIGHWAY)
- `HighwayDriver.c` - Highway UART protocol (9600 baud, 0x30 0xCF SOF)
- `CHKDriver.c` - CHK UART protocol (4800 baud, AA 55 SOF)
- `AiferDriver.c` - Aifer driver (stub/minimal)
- `HighwayTempLookUp.h` - NTC ADC-to-Celsius table

### ExternalPeripherals/InductionCooker/ (cooking business logic)
- `IndCooker.c` - Main controller, timer mgmt, PowerBoard callback handler
- `CookerState.c` - On/Off state pattern, starts/stops CookingSession
- `CookingSession.c` - Accumulates power (kWh) per session, saves to flash on stop
- `CookerModel.c` - CookerPower_e enum to watt conversion

### ExternalPeripherals/Display/ (VK16K33 7-segment over I2C)
- `Display.c` - Show strings/ints/floats, power levels, light bars, icons, blinking
- `DisplayTest.c` - Test routines
- `CharacterMap.h` / `NewShineCharMap.h` / `BaoLaiYaCharMap.h` - Segment maps per board

### ExternalPeripherals/Keypad/ (capacitive touch)
- `Keypad.c` - Raw key scanning + debounce (8 command + 6 numerical keys)
- `KeypadEvents.c` - Short/long press detection, fires callback to UIView
- `X12ACapTouch.c` - X12A cap touch I2C driver (BaoLaiYa/NewShine variants)
- `KeypadTest.c` - Test routines

### ExternalPeripherals/Led/
- `Led.c` - 6 LEDs: Pay Green/Red/Amber, Indicator Blue/Red, Cmd Button

### ExternalPeripherals/Flash/ (W25Qxx SPI flash)
- `w25qxx.c` - Low-level flash driver (third-party)
- `ringfs.c` - Circular log filesystem on flash
- `FlashData.c` - Session data buffer (CookerSessionData_t records)
- `SessionAggr.c` - Daily/monthly power aggregation
- `SpiFlashMemoryMap.h` - Flash sector allocation
- `RtcBackup.c` - STM32 RTC backup register access

### ExternalPeripherals/RTC/
- `UserRtc.c` - TIME_T struct, get/set time, Unix conversion, time diff

### ExternalPeripherals/UserSystick/
- `UserSystick.c` - 1ms tick from SysTick, second counter

### ExternalPeripherals/SerialDebug/
- `Debug.c` - printf-style debug over UART4

### ProductFeatures/UI/ (MVP pattern)
- `UIPresenter.c` - State machine with state table, timeout/key event handling
- `UIModel.c` - UIModelData_t (PowerSetting, ErrorStatus, CookTime), wraps IndCooker
- `UIView.c` - 21+ view types, display rendering, icon mgmt, beep engine

### ProductFeatures/DeviceSettings/
- `DeviceSettings.c` - Hardware version params from flash

### ProductFeatures/DeviceStats/
- `DeviceStats.c` - Temp stats (paygo/surface/IGBT avg/max)

### ProductFeatures/CRC/
- `CrcWrapper.c` - HAL CRC wrapper (16-bit + 32-bit)

---

## Stub/Future Directories (empty or inactive)
- `ExternalPeripherals/Bluetooth/` - BT module
- `ExternalPeripherals/RFComms/` - RF comms (At, BT, BTModemDriver, Payload)
- `ExternalPeripherals/Temperature/` - Temp sensing
- `ExternalPeripherals/TempControl/` - Temp control
- `ExternalPeripherals/FlagsSubsystem/` - Flags
- `ProductFeatures/BackgroundOta/` - OTA updates
- `ProductFeatures/CreditManagement/` - Paygo credit
- `ProductFeatures/DeviceErrors/` - Error tracking
- `ProductFeatures/Encryption/` - Encryption
- `ProductFeatures/STM32_Cryptographic/` - Crypto lib
- `ProductFeatures/SecData/` - Secure data
- `ProductFeatures/Shadow/` - Config sync
- `ProductFeatures/Stubs/` - Stubs
- `ProductFeatures/TamperProofing/` - Tamper detection
- `ProductFeatures/Token/` - Paygo tokens
- `ProductFeatures/UartCommMessaging/` - UART messaging
- `Utils/` - Calc, Json, Queue, SharedBuffer
- `GlobalDefs/` - Global type defs
- `Drivers/Nexus/` - Nexus paygo channel protocol

## Vendor Drivers (DO NOT MODIFY)
- `Drivers/CMSIS/` - ARM Cortex-M0+ core headers
- `Drivers/STM32G0xx_HAL_Driver/` - STM32G0 HAL & LL drivers
