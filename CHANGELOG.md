# Changelog

All notable changes to the LBP → LB transpiler will be documented in this file.

The project is still in an experimental / RC phase. Version numbers below describe
feature milestones rather than strict semantic-versioning guarantees.

---

## [1.0.0-rc1] – First public release candidate

> **Core behaviour:** single-pass, line-oriented LBP-to-Label BASIC transpiler,
> lowering a small set of structured constructs into plain LB.

### Added

- **Structured IF / ELSE / END IF lowering**
  - Recognises `IF <cond> THEN` when used as a structured header (no body on the same line).
  - Emits guarded jump and labels of the form:
    - `Z__IFn_ELSE`
    - `Z__IFn_END`
  - Supports:
    - `IF ... THEN`
    - `ELSE`
    - `END IF`
  - Tracks IF nesting using an internal stack with depth checking.
  - Detects and reports:
    - `ELSE` without a matching `IF`
    - `END IF` without a matching `IF`
    - Multiple `ELSE` blocks for the same `IF`
    - Unterminated `IF` blocks at EOF.

- **Structured loops with EXIT / CONTINUE**
  - Shared loop stack for:
    - `WHILE ... WEND` (pre-test loop)
    - `REPEAT ... UNTIL <cond>` (post-test loop)
  - Each loop instance uses labels of the form:
    - `Z__WHn_TOP`
    - `Z__WHn_CHK`
    - `Z__WHn_END`
  - Lowerings:
    - `WHILE <cond>` → top label + guard (`IF (<cond>)=0 THEN Z__WHn_END`).
    - `WEND` → `GOTO Z__WHn_TOP` then `Z__WHn_END:`.
    - `REPEAT` → marks `Z__WHn_TOP:`.
    - `UNTIL <cond>` → emits `Z__WHn_CHK:` and
      `IF (<cond>)=0 THEN Z__WHn_TOP` then `Z__WHn_END:`.
  - Loop stack treats nesting correctly and reports unterminated loops at EOF.

- **Loop control statements**
  - `EXIT WHILE` / `CONTINUE WHILE`
    - Search upwards on loop stack for nearest WHILE/WEND loop.
    - `EXIT WHILE` → jump to that loop’s `Z__WHn_END`.
    - `CONTINUE WHILE` → jump to that loop’s `Z__WHn_TOP`.
  - `EXIT REPEAT` / `CONTINUE REPEAT`
    - Search upwards for nearest REPEAT/UNTIL loop.
    - `EXIT REPEAT` → jump to that loop’s `Z__WHn_END`.
    - `CONTINUE REPEAT` → jump to that loop’s `Z__WHn_CHK` (i.e. perform UNTIL test next).
  - `EXIT LOOP` / `CONTINUE LOOP`
    - Search upwards for **nearest loop of any type** (WHILE or REPEAT).
    - `EXIT LOOP` → jump to that loop’s `Z__WHn_END`.
    - `CONTINUE LOOP` →  
      - WHILE → `GOTO Z__WHn_TOP`  
      - REPEAT → `GOTO Z__WHn_CHK`
  - All loop-control handlers detect and report usage when there is no matching loop,
    respecting strict-error mode (see below).

- **Constants and enums with optional #DEFINE lowering**
  - `CONSTANT` handling
    - Single-line form: `CONSTANT NAME = expr`.
    - Block form:
      ```basic
      CONSTANT
        NAME1 = expr1
        NAME2 = expr2
      ENDCONST
      ```
  - Type detection:
    - String constants recognised via `$` suffix or quoted literal on RHS.
    - Numeric constants parsed via `VAL()`.
  - Symbol table:
    - Keeps track of names, types (CONST / ENUM) and numeric or string values.
    - Not yet exposed to the user (no symbol-table dump/export in this RC).
  - Lowering rules:
    - String constants are always emitted as assignments:
      - `NAME$ = "string"`
    - Numeric constants are:
      - Emitted as `#DEFINE NAME value` when:
        - `##OPTION DEFINE ON` (default), and
        - value fits in 0..65535.
      - Otherwise emitted as assignment:
        - `NAME = expr`.

  - `ENUM` handling
    - Block form:
      ```basic
      ENUM
        NAME1          : REM auto = 1
        NAME2          : REM auto = 2
        NAME3 = 10     : REM explicit
        NAME4          : REM auto = 11
      END ENUM
      ```
    - Automatically assigns increasing integer values starting at 1.
    - Explicit values via `NAME = expr` reset the current value.
    - Lowering rules mirror numeric constants:
      - Prefer `#DEFINE` when `DEFINE` is ON and value in 0..65535.
      - Otherwise emit `NAME = value`.

- **Built-in logical constants**
  - Always emitted at the start of the output:
    - `FALSE = 0`
    - `TRUE = 65535` (all bits set for 16-bit logical operations).

- **Transpiler options via comment directives**
  - Directives are recognised in the form:
    - `REM ##OPTION NAME VALUE`
  - Options are case-insensitive for both name and value.
  - Values recognised (case-insensitive):
    - ON, OFF
    - TRUE, FALSE
    - ENABLE, DISABLE
    - 1, 0
  - Implemented options:
    - `DEFINE` (default ON)
      - Controls whether suitable numeric constants/enums are emitted as `#DEFINE` or as assignments.
    - `REMPASS` (default OFF)
      - When ON, the original option lines are also written to the output.
      - When OFF, option lines are only echoed to the screen.
    - `STRICT_ERR` (default OFF)
      - When ON, the transpiler sets an internal abort flag on structural errors
        (e.g. `ELSE` without `IF`, `WEND` without `WHILE`, unterminated loops/ifs at EOF).
      - The main loop checks `ABORT.FLAG%` and terminates cleanly via `LBP.ABORT` / `LBP.CLEANUP`.

- **Disk I/O and EOF handling**
  - Uses X16/C64-style DOS channel 15 (`ST.CH%`) for status:
    - Helper `LBP.GET.DOS.STATUS` to parse error code, message, track and sector.
    - `LBP.SHOW.DISK.ERROR` prints human-readable diagnostics.
  - Input and output file handling:
    - Safe open of input as `",S,R"` (does not create if missing).
    - Output is opened as `"@:<name>,S,W"` (create/overwrite with scratch semantics).
    - Output filename defaults to input basename plus `.BAS` if not specified.
    - Overwrite prompt if output file already exists.
  - EOF and EOI logic for line reads:
    - `LBP.READ_LINE` deals with CR, LF, NUL and the EOI bit in `ST`.
    - Returns empty buffer with `EOF.FLAG%` set for end-of-file conditions.
    - Treats blank lines correctly as present/real lines.
  - All open files are closed in `LBP.CLEANUP`, even on error/abort paths.

- **Case-insensitive keyword handling**
  - Internally normalises buffer copies via `LBP.TO_UPPER` for keyword detection.
  - Original input line (`BUF$`) is emitted unchanged to preserve case where the
    transpiler is not transforming structure.

- **Miscellaneous helpers**
  - `LBP.TRIM.BOTH` – trims leading/trailing spaces in `TMP$`.
  - `LBP.PARSE.NAME.EQ.EXPR` – splits `"name = expr"` pairs used by CONSTANT/ENUM logic.
  - `LBP.MAKE.IF.LABELS` / `LBP.MAKE.WH.LABELS` – generate unique internal label names.

### Behavioural notes / limitations for 1.0.0-rc1

- The program banner still prints:
  - `LBP IDENTITY TRANSPILER V0.2`
  - This is cosmetic; users can update the string in `LBP.MAIN` when finalising 1.0.
- Lines not recognised as structured constructs are passed through unchanged
  (echo to screen + write to output).
- Only one logical statement per physical line is currently supported for transformation:
  - No special handling yet for multiple statements separated by `:`.
- No `ELSEIF` support – such constructs are currently treated as normal lines.
- Symbol table is kept in memory but not exposed (no `##SYMBOLS` facility yet).
- No optimisation/compression of output:
  - No statement-joining, line-continuation or line-splitting logic.
- No `ASSERT`, `REM_MAP` or macro/include system yet – those belong to a planned v1.5.

---

## [0.2.0] – Identity transpiler with I/O & scaffolding

_This describes the internal stage that the current code evolved from;
it is not necessarily a separately distributed version._

### Added

- Basic “identity” behaviour:
  - Read source file line by line.
  - Echo each line to the screen.
  - Write each line unchanged to an output file.
- Initial DOS error-handling wrappers:
  - Status reading from channel 15.
  - Simple file-open failure reporting and cleanup.
- Default output filename logic:
  - If no explicit output name is given, use input name with `.BAS` extension.
- Skeleton/layout for future features:
  - Placeholder comments describing intended uppercase output behaviour.
  - Early structure for main loop, EOF handling and cleanup.

---

## [0.1.0] – Initial experiments (unreleased / prototype)

_Informal, local prototypes only._

- Rough test harnesses for:
  - Reading sequential text from device.
  - Primitive identity copy of BASIC source.
- No options, no structured constructs, minimal error handling.

---

### Planned (not yet implemented, for v1.5+)

These features are **not** in 1.0.0-rc1, but are design candidates for later versions:

- Warning system (unused constants, suspicious shadows, deep nesting, etc.).
- Better error reporting with context buffers.
- Line continuation with `\` at end of line.
- Symbol-table dump and/or export.
- ASSERT support and optional removal in the output.
- REM-mapping (`REM LBP line #n`) for cross-referencing.
- Statement compression/expansion (one statement per line vs. compacted lines).
- Macro facilities and simple include system (`#INCLUDE`, `#MACRO`, etc.).
