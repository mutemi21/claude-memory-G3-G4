You are picking up a firmware port task on the G3 STM32 induction-cooker codebase.

## Goal
Port the "Usage Feature" from the `Prescan_New_Display` branch onto the
`Usage-Feature` branch (which was cut from `Develop`). The feature is:
  - Realtime kWh display during cooking when INFO is pressed
  - Last-session kWh display during STANDBY when INFO is pressed
  - Automatic session-usage display at end of cook (USED label + value)
  - Full compatibility with locked-cooking (LOCK button behaviour preserved)
  - E0 voltage-fault must PAUSE the session, not end it

## Required reading (read in this order BEFORE writing any code)
1. `projects/.../memory/UsageFeaturePort.md` — the master spec. The 12 source
   commits, target architecture, Develop-specific integration notes, and
   bench-test acceptance criteria are all here. Read every section.
2. `projects/.../memory/MEMORY.md` — index of project memory.
3. `projects/.../memory/features.md` — see "Energy (kWh) accounting" section.
4. `projects/.../memory/bugfixes.md` — entries on B5/B6/B7 error recovery,
   A2 timer-in-place edit, F20 auto-off lock preservation. These behaviours
   must be preserved in the port.
5. `projects/.../memory/codebase_summary.md` — file-by-file map.
6. Optional: `projects/.../memory/chk.md` for power-board protocol context.

(The memory repo is github.com/mutemi21/claude-memory-G3-G4. The firmware repo
is the G3 STM32_FIRMWARE_G3 repo — branch `Usage-Feature` for the work,
`Prescan_New_Display` for reference commits, `Develop` is the merge target.)

## Branch state
- Work on: `Usage-Feature` (branched from `Develop` at ddb049f, currently empty
  of feature changes — only contains the spec doc).
- Reference: `Prescan_New_Display` — contains the 12 commits whose hashes
  are listed in UsageFeaturePort.md.
- Cherry-pick is NOT viable. Develop has diverged (Token system, Tamper, Pay,
  restructured PowerBoardStatusCb, API renames). Re-implement against Develop's
  current state using the spec as the design and Prescan as the reference
  implementation.

## CRITICAL: Borrow the concept, don't copy the code

Prescan's implementation is REFERENCE ONLY. Do not copy code verbatim.
Develop has diverged significantly:
  - Token system, Tamper handling, and Pay flow exist on Develop but not in
    the same form on Prescan.
  - `PowerBoardStatusCb` has been restructured on Develop — the spec shows
    the adapted shape; preserve Develop's voltage-fault zero-power routing.
  - Several APIs were renamed between branches.
  - The keypad and presenter state machines have different state sets and
    transition guards.

For each piece of functionality you port:
  1. Read the Prescan commit to understand WHAT it does and WHY.
  2. Look at the equivalent surface on Develop (same file, same function,
     same state) and understand its CURRENT shape and constraints.
  3. Re-implement the behaviour in a way that fits Develop's existing
     structure — adding states, helpers, and view cases as the spec
     describes, but written against Develop's APIs, callbacks, and state
     enums, not Prescan's.
  4. Preserve all Develop-only behaviours: token states, tamper, pay,
     voltage-fault zero-power routing, any other diverged logic.

If you find yourself blindly transcribing a Prescan diff, STOP and re-read
this section. A clean compile against Develop is a necessary but not
sufficient signal — the integrated behaviour must respect Develop's
existing flows.

## Coding conventions (strict — match existing code exactly)
- CamelCase for variables, functions, and filenames. UPPERCASE for macros.
- 4 spaces per indent, no tabs.
- Braces on same line as condition/declaration; always brace single-line bodies.
- Public symbols (callable from other .c files) prefixed `<Filename>_`.
  Private symbols are `static` and get NO prefix.
- `<stdint.h>` types (`uint8_t` etc).
- No magic numbers — define named constants.
- Struct: `typedef struct { ... } ExampleStruct_t;`

## Workflow
1. Read all required-reading docs first. Then share back to the user a
   high-level plan (which states/files/helpers you're adding, in what order)
   and wait for sign-off before writing code.
2. Implement in the phased order from UsageFeaturePort.md:
     Phase A -> locked-timer plumbing
     Phase B -> end-of-session display + E0 auto-shutdown
     Phase C -> E0 pause-not-end
     Phase D -> info button (realtime + standby)
   One commit per phase. Commit message style: match `git log --oneline -10`
   on the firmware repo.
3. After each phase, build the project and report any errors/warnings before
   moving on. Ask the user to bench-test that phase.
4. Do NOT push to remote without explicit user confirmation.

## Acceptance criteria
The bench tests listed in section 6 of UsageFeaturePort.md must all pass on
real hardware before the port is considered complete. Do not mark "done"
based on a clean build alone.

## Collaboration style
- Before any detailed implementation, share the high-level plan and wait.
- When troubleshooting, do NOT guess. Ask the user to run a specific bench
  test to gather data.
- For each non-trivial decision: state the tradeoff in 2-3 sentences and
  recommend a path; let the user redirect.
- Consider how each change affects both Highway and CHK power-board drivers.

Begin by reading UsageFeaturePort.md and the four other required docs, then
report back with your high-level implementation plan.
