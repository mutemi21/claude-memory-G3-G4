# Memory Reminders

After completing any of the following, ask: "Would you like me to save this to memory?"
- Bug fix — save to `bugfixes.md`
- New feature — save to `features.md`
- Architectural decision or important pattern discovery
- Any workaround or non-obvious solution that took multiple attempts

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
