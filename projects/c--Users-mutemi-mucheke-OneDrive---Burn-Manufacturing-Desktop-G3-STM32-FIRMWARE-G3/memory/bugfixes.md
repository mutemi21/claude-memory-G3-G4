# Bug Fixes

## "USED" displays as "USE" on NewShine 7-segment display
- **Product:** G3
- **Date:** 2026-03-05
- **Branch:** BugFix/Timer-Feature
- **Files changed:** `ExternalPeripherals/Display/Src/Display.c`
- **Root cause:** `CharMapAsciiCharLookup()` mapped 'D'/'d' to `CAP_D`, but the NewShine character map (`NewShineCharMap.h`) does not define `CAP_D` — only `SMALL_D`. The `UpperCharSegments[CAP_D]` lookup read zero/uninitialized data, rendering 'D' as blank.
- **What we tried that didn't work:** N/A — root cause was identified directly by comparing the enum entries in `NewShineCharMap.h` vs `CharacterMap.h`.
- **Final solution:** Changed `Display.c` line ~423: `RetVal = CAP_D` → `RetVal = SMALL_D`. Lowercase 'd' (segments B,C,D,E,G) is the standard 7-segment representation since uppercase 'D' is indistinguishable from '0'. "USED" now renders as "USEd".
- **Verification:** Power off cooker after cooking — display should show "USEd" (not "USE") during shutdown sequence.
- **Status:** Untested

## UART command flooding causes power output failure after off/on cycle
- **Product:** G3
- **Date:** 2026-03-05
- **Branch:** BugFix/Timer-Feature
- **Files changed:** `ExternalPeripherals/PowerBoard/Src/HighwayDriver.c`
- **Root cause:** `HighwayDriver_SetLevel()` sent an immediate UART command AND the periodic tick also sent commands. This dual-path flooding overwhelmed the power board, causing it to stop responding after an off/on cycle. Same bug as G4.
- **What we tried that didn't work:** N/A — known fix from G4.
- **Final solution:** Removed `SendCurrentPowerCommandToCooker(false)` from `HighwayDriver_SetLevel()`. Added TIM14 ISR (~20ms) for periodic command sends instead of the 1-second tick. Registered TIM14 callback and started timer in `HighwayDriver_Init()`. Removed send from `HighwayDriver_Tick()`.
- **Verification:** Turn cooker on, change power levels, turn off, turn on again — power levels should respond correctly.
- **Status:** Untested
