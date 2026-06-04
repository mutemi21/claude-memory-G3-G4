# Working Style

- Before generating a detailed answer, generate high-level answers first to ensure we are on the same page.
- When troubleshooting, do not guess if you are not sure. Ask me to run a specific test to help gather enough data to solve the problem.

# Before starting any non-trivial task

If the project's memory directory contains `bugfixes.md` and/or `features.md`,
scan them at the start of any non-trivial task — they hold historical context
that may surface relevant prior bugs/decisions for the area you're touching.
Focus on entries near the modules you'll be modifying. This is in addition to
any per-project required reading (e.g. `codebase_summary.md`).

# Memory Reminders

After completing any of the following, ask: "Would you like me to save this to memory?"
- Bug fix — save to `bugfixes.md`
- New feature — save to `features.md`
- Architectural decision or important pattern discovery
- Any workaround or non-obvious solution that took multiple attempts

## CHK power board reference

The CHK 2 kW power board is shared between G3 and G4. The corrected/extended
protocol reference lives at `<project>/memory/chk.md` in **both** the G3 and G4
project memory directories. Before doing any CHK-protocol or CHK-driver work,
read the `chk.md` in the project you're currently in. The original supplier
doc has gaps — undocumented exception codes, no glass-over-temp signal, an
incorrect example value for `RETURNED_DATA_LENGTH`, etc. — and `chk.md`
documents what we have actually observed on the wire.

**When updating `chk.md`, update both copies** so the two projects stay in
sync:
- `~/.claude/projects/c--Users-...-G3-STM32-FIRMWARE-G3/memory/chk.md`
- `~/.claude/projects/c--Users-...-G4-FW-STM32-FIRMWARE-G4/memory/chk.md`

When saving, use this format and append to the existing file:

```
## [Short title of the fix/feature]
- **Product:** [G3 / G4 / Both]
- **Date:** [today's date]
- **Branch:** [branch name]
- **Files changed:** [list the files modified]
- **Root cause / Purpose:** Why this change was needed
- **What we tried that didn't work:** Any failed approaches (if applicable)
- **Final solution:** What we actually did, with file names and line-level detail — enough that another Claude instance on a different product codebase could reproduce it
- **Verification:** How to test that it works
- **Status:** [Untested / Verified / Failed]
```

When the user confirms a fix works, update its Status to "Verified". If it failed, update to "Failed" and note what went wrong.

After saving to memory, ask: "Would you like me to commit and push the memory update to GitHub?"
If yes, run: `cd ~/.claude && git add -A && git commit -m "<short description of what was saved>" && git push`

# Global Coding Guidelines

## C/C++ Firmware

### Naming
- **CamelCase** for variables, functions, and file names.
- **UPPERCASE** for macros.
- **Meaningful names** for all variables, constants, and functions — no abbreviations that obscure intent.
- Any function or variable that can be called/accessed from **outside its source file** must be prefixed with the **full filename (without extension)** followed by an underscore — no abbreviations. This makes the origin immediately clear to the caller.
  - e.g., `bool HighwayDriver_FanOn(void)` — callable from other files, prefix shows it lives in `HighwayDriver.c`.
- Functions used **only inside their own source file** must be declared `static` and do not get the module prefix.
  - e.g., `static uint8_t ConvertPotTemperatureADCReadingToCelsius(uint8_t ADCReading)` — private to `HighwayDriver.c`.

### Types
- Use `<stdint.h>` types for all integer types (`uint8_t`, `uint16_t`, `int32_t`, etc.).

### Formatting
- **4 spaces** per indentation level — never tab characters.
- One space after commas and semicolons in function calls and declarations.
- Braces always on the **same line** as the condition or function declaration.
- Always use braces even for single-line bodies.
- Use parentheses for clarity in complex expressions.

### Constants & Macros
- Never use magic numbers in expressions — define named constants with `#define`.

### Structure Declarations
```c
typedef struct
{
    // members
}
ExampleStruct_t;
```

### General Rules
- Avoid unnecessary global variables.
- Avoid code duplication — call functions rather than copy-pasting.
- Do not use `goto`.

### Examples
```c
// Public function in HighwayDriver.c — callable from other files, module prefix required
bool HighwayDriver_FanOn(void);

// Private function in HighwayDriver.c — only used internally, static, no module prefix
static uint8_t ConvertPotTemperatureADCReadingToCelsius(uint8_t ADCReading);

// Macro
#define UART_BUFFER_SIZE    128
#define PI                  3.1415

// Variables
uint8_t *LocalBuffer;
uint16_t BufferSize;

// Braces & conditionals
void CheckNetworkRegistration(void) {
    if (MyGSM.CheckNetworkRegistration()) {
        NetRegFailCnt = 0;
    } else {
        NetRegFailCnt++;
    }
}

// Single-line body still gets braces
if (SimModuleOnCount > 8) {
    SimModuleOnCount = 0;
}

// Complex expressions — use parentheses
Result = ((Param1 * Param2) - (Param3 * Param4)) / (Param1 + Param2 + Param3 + Param4);
```
