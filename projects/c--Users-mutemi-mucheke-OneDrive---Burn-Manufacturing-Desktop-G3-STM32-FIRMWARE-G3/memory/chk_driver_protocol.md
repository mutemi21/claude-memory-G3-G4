---
name: CHK Power Board Protocol & Learnings
description: CHK power board UART protocol details, wake-up sequence, and refactoring notes from bring-up testing
type: reference
---

## CHK Power Board Communication

### UART Config
- **USART3**: PB10 (TX), PB11 (RX)
- **Baud rate**: 4800
- **Format**: 8N1

### TX Frame (STM32 → Power Board) — 13 bytes
```
AA 55 09 10 10 14 [ControlSet] [PowerSwitch] [FireGear] FF [PowerMarker] [Checksum] FF
```
- ControlSet: Fan/Buzzer control (0x87 = Fan ON, Buzzer OFF)
- PowerSwitch: 0x10 = ON, 0x00 = OFF
- FireGear: power/25 (e.g., 200W = 0x08, 400W = 0x10, 2000W = 0x50)
- PowerMarker: 0xE0 = ON, 0x3C = OFF
- Checksum: modulo-256 sum of bytes [0..10]

### RX Frame (Power Board → STM32) — 26 bytes
```
AA 55 16 10 [StatusByte] ... [Checksum] FF
```
- Length field = 0x16 (22), NOT 0x14 (20) as SIZE_OF_STATUS_MESSAGE_PAYLOAD suggests
- StatusByte[3:0] = error bits, StatusByte[4] = pot presence (1=absent, 0=present)
- Data[3] = IGBT temperature (raw ADC)
- Data[4] = Surface temperature (raw ADC)

### Sleep & Wake-Up
- Power board sleeps after ~100ms of no commands
- TIM7 sends keep-alive every 30ms (prevents sleep during normal operation)
- Wake-up sequence (from supplier PDF):
  1. Reconfigure PB10/PB11 as GPIO output
  2. Drive both HIGH for ~185ns (5 NOPs)
  3. Drive both LOW for ~185ns (5 NOPs)
  4. Wait 40ms (HAL_Delay)
  5. Restore PB10/PB11 to USART3 AF4
  6. Re-init UART, start TIM7

### Main Loop Tick Rate
- `TickFlag` fires once per SECOND (not 100ms as comments suggest)
- Chain: SysTick (1ms) → UserSystick_Handler → counts to 1000 → sets TickFlag
- `CHKDriver_Tick()` is called via: main tick → IndCooker_Tick → PowerBoard_Tick → CHKDriver_Tick

## Refactoring Notes

### Commanded vs Reported State (CRITICAL)
- **Commanded state** (set by our code): `CurrentPowerState`, `CurrentPowerLevel`, `FanOn`
- **Reported state** (parsed from RX): pot presence, errors, temperatures
- **NEVER let RX parsing overwrite commanded state.** The original code had `NewPowerState = POWER_OFF` hardcoded in `UpdateCurrentStatus()` which overwrote `CurrentPowerState` on every received frame, killing power commands.

### PowerBoard Type Selection
- `PowerBoard_SetBaudRate()` and `PowerBoard_Init()` both hardcode `PowerBoardType`
- Both locations must match — there are TWO assignments to update
- TODO: Read from flash instead of hardcoding

### Dual Command Paths
- TIM7 ISR sends commands every 30ms (main keep-alive)
- `PowerOn()`/`SetLevel()` also call `SendCurrentPowerCommandToCooker()` immediately
- This is intentional for CHK — the 30ms timer is the primary mechanism
