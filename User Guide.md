# Label BASIC Pro Language & Transpiler User Guide  
**Version:** v1.0 RC1

---

## Contents
1. [Introduction](#1-introduction)  
2. [Installation & Running](#2-installation--running)  
   - [Requirements](#21-requirements)  
   - [Running LBP](#22-running-lbp)  
3. [Writing LBP Source Code](#3-writing-lbp-source-code)  
4. [Constants & Enums](#4-constants--enums)  
5. [Enums](#5-enums)  
6. [Structured Conditionals](#6-structured-conditionals)  
   - [Nested IFs](#62-nested-ifs)  
   - [ELSEIF Style](#63-elseif-style)  
7. [Looping Constructs](#7-looping-constructs)  
   - [WHILE/WEND](#71-while--wend)  
   - [REPEAT/UNTIL](#72-repeat--until)  
   - [EXIT and CONTINUE](#73-exit-and-continue)  
8. [Options (Directives)](#8-options-directives)  
9. [Error Handling](#9-error-handling)  
10. [Example LBP Program](#10-example-lbp-program)  
11. [Output Files](#11-output-files)  
12. [Limitations of v1.0 RC1](#12-limitations-of-v10-rc1)  
13. [Notes for Advanced Users](#13-notes-for-advanced-users)  
14. [Troubleshooting](#14-troubleshooting)  
15. [Performance Notes](#15-performance-notes)  
16. [Summary](#16-summary)

---

# 1. Introduction

LBP is a transpiler that converts **Label BASIC Pro (LBP)** — a structured, more expressive dialect of Label BASIC (LB) — into LB suitable for the Commander X16’s **BASLOAD** tool.

LBP adds modern conveniences:

- Multi-line `IF / ELSE / END IF`
- `WHILE / WEND`
- `REPEAT / UNTIL`
- `EXIT / CONTINUE LOOP`, `EXIT WHILE`, `EXIT REPEAT`
- `CONSTANT` and `ENUM`
- Strict structural error checking

LBP source is readable and maintainable; the final output becomes valid BASIC V2, tokenised by BASLOAD.

**IMPORTANT:**  
LB and LBP are **case-insensitive** for:

- Keywords  
- Statements  
- Labels  
- Variable names  

This matches the behaviour of BASIC V2 and BASLOAD.

---

# 2. Installation & Running

## 2.1 Requirements

- Commander X16 (emulator or real hardware)  
- BASIC environment with BASLOAD  
- `LBP.PRG` placed on your disk/device

## 2.2 Running LBP

At the BASIC prompt:

```
LOAD "LBP.PRG"
RUN
```

You will be prompted for:

1. Device number (RETURN = default 8)  
2. Input filename (.LBP)  
3. Output filename  
   - RETURN → same as input but `.BAS` extension  
   - If existing: LBP asks whether to overwrite; if **no**, it will ask for another filename.

LBP then transpiles and writes the output LB file.

---

# 3. Writing LBP Source Code

LBP accepts ASCII files with **LF**, **CRLF**, or **CR** endings.

General rules:

- Case-insensitive for identifiers and keywords  
- One structured construct per line  
- Labels must appear alone ending with `:`  
- Variables/labels may be up to 64 chars, must begin with a letter  
- Allowed characters: `A–Z`, `0–9`, `_`, `.`  
- Do **not** start identifiers with `Z__` (reserved by LBP)  
- Variable name cannot duplicate a label name  
- Reserved X16 variables: `TI$`, `TI`, `DA$`, `ST`, `MX`, `MY`, `MB`

Structured constructs must appear alone (may have `: REM something` after):

- `IF`  
- `ELSE`  
- `END IF`  
- `WHILE`  
- `WEND`  
- `REPEAT`  
- `UNTIL`  
- `CONSTANT`, `ENDCONST`  
- `ENUM`, `END ENUM`  
- `REM ##OPTION …`

Other lines may contain any valid LB code.

---

# 4. Constants & Enums

### 4.1 Single-line constant

```
CONSTANT MAX.SIZE = 100
```

### 4.2 Block constants

```
CONSTANT
  BUF.LEN = 256
  TITLE$  = "HELLO"
ENDCONST
```

Notes:

- Supports string and numeric constants  
- With `DEFINE ON`, numeric constants in 0–65535 emit as `#DEFINE`  
- Strings always become assignments  
- Case-insensitive identifiers  

### Built-in System Constants (always emitted)

```
FALSE = 0
TRUE  = -1
```

Reason for `TRUE = -1`:

- BASIC treats TRUE as all bits set (16-bit)
- Ensures correct operation with `NOT`, `AND`, `OR`

---

# 5. Enums

Example:

```
ENUM COLOUR
  RED            : REM 1
  GREEN = 5      : REM 5
  BLUE           : REM 6
END ENUM
```

Behaviour:

- First unassigned member = 1  
- Unassigned values auto-increment  
- Emitted as `#DEFINE` or `NAME = value` depending on options  

---

# 6. Structured Conditionals

## 6.1 Multi-line IF

```
IF SCORE >= 50 THEN
  PRINT "PASS"
ELSE
  PRINT "FAIL"
END IF
```

Rules:

- `IF <expr> THEN` must be alone  
- `ELSE` must be alone  
- `END IF` must be alone  

LBP lowers this into labels + GOTOs.

---

## 6.2 Nested IFs

Nested IF blocks are fully supported.

### Example 1

```
IF A THEN
  IF B THEN
    PRINT "A AND B"
  END IF
ELSE
  PRINT "NOT A"
END IF
```

### Example 2

```
IF A > 0 THEN
  IF B = 1 THEN
    PRINT "CASE 1"
  ELSE
    IF C$ = "YES" THEN
      PRINT "CASE 2"
    END IF
  END IF
END IF
```

---

## 6.3 ELSEIF Style (Not Supported)

### Invalid:

```
IF X < 0 THEN
  PRINT "NEG"
ELSEIF X = 0 THEN      : REM INVALID
  PRINT "ZERO"
ELSE
  PRINT "POS"
END IF
```

Or:

```
ELSE IF X = 0 THEN     : REM INVALID
```

### Valid LBP alternative:

```
IF X < 0 THEN
  PRINT "NEG"
ELSE
  IF X = 0 THEN
    PRINT "ZERO"
  ELSE
    PRINT "POS"
  END IF
END IF
```

---

# 7. Looping Constructs

## 7.1 WHILE / WEND

```
WHILE COUNT < 10
  PRINT COUNT
  COUNT = COUNT + 1
WEND
```

---

## 7.2 REPEAT / UNTIL

```
REPEAT
  INPUT "VALUE:"; V%
UNTIL V% >= 0
```

Post-condition looping fully supported.

---

## 7.3 EXIT and CONTINUE

LBP supports structured loop control:

```
EXIT WHILE
CONTINUE WHILE

EXIT REPEAT
CONTINUE REPEAT

EXIT LOOP       : REM nearest loop
CONTINUE LOOP   : REM nearest loop
```

### Example: EXIT WHILE

```
WHILE X < 100
  IF X = 50 THEN EXIT WHILE
  X = X + 1
WEND
```

### Example: CONTINUE LOOP

```
REPEAT
  X = X + 1
  IF X MOD 2 = 1 THEN CONTINUE LOOP
  PRINT "EVEN:"; X
UNTIL X > 20
```

---

# 8. Options (Directives)

Options are written as:

```
REM ##OPTION NAME VALUE
```

---

## DEFINE ON|OFF

Controls numeric constant and enum emission.

### Example

```
REM ##OPTION DEFINE ON
```

---

## REMPASS ON|OFF

Controls whether the option line is emitted into the LB output.

---

## STRICT_ERR ON|OFF

### Example that triggers strict mode abort:

```
REM ##OPTION STRICT_ERR ON
ELSE     : REM invalid, triggers abort
```

---

# 9. Error Handling

LBP uses IF and loop stacks to ensure structural correctness.

Below are all errors implemented in v1.0 RC1.

---

## 9.1 ELSE Without IF

```
ELSE
```

---

## 9.2 Multiple ELSE for IF

```
IF A THEN
ELSE
ELSE   : REM invalid
END IF
```

---

## 9.3 END IF Without IF

```
END IF
```

---

## 9.4 Missing END IF

```
IF A THEN
  PRINT "OPEN BLOCK"
```

---

## 9.5 WEND Without WHILE

```
WEND
```

---

## 9.6 UNTIL Without REPEAT

```
UNTIL X > 0
```

---

## 9.7 UNTIL for Non-REPEAT Loop

```
WHILE A < 10
UNTIL A = 5
```

---

## 9.8 Missing WEND/UNTIL

```
REPEAT
  PRINT "OPEN LOOP"
```

---

## 9.9 Colon Not Allowed on Structured Line

```
IF X THEN: PRINT "NO"
```

---

# 10. Example LBP Program

```
REM ##OPTION DEFINE OFF

CONSTANT
  MAX.SCORE = 100
  PASS.MARK = 60
ENDCONST

INPUT "ENTER SCORE:"; S%

IF S% < 0 THEN
  PRINT "INVALID"
ELSE
  IF S% > MAX.SCORE THEN
    PRINT "TOO HIGH"
  ELSE
    IF S% >= PASS.MARK THEN
      PRINT "PASS"
    ELSE
      PRINT "FAIL"
    END IF
  END IF
END IF

END
```

---

# 11. Output Files

LBP produces:

- `.LBP` (your source)  
- `.BAS` (LB output for BASLOAD)  

---

# 12. Limitations of v1.0 RC1

(Same as previous)

---

# 13. Notes for Advanced Users

- Output is plain LB  
- Labels start with `Z__`  
- TRUE = -1, FALSE = 0  
- Blank lines preserved  

---

# 14. Troubleshooting

(Same as previous)

---

# 15. Performance Notes

LBP is implemented in LB and thus runs at BASIC V2 speed.

Performance expectations:

- Files **≤ 250 lines** → generally fast  
- Files **> 250 lines** → slower; you may want to grab a cup of tea ☕  

Slow operations include:

- Reading input lines  
- Printing diagnostics  
- Writing large output files  
- Huge CONSTANT/ENUM blocks  

Future versions will have a **compiled machine-code** version (LLVM-MOS), dramatically improving speed.

---

# 16. Summary

LBP v1.0 RC1 provides a reliable structured superset of LB that lowers cleanly to BASIC V2.  
It is stable, predictable, and ready for real-world use and community feedback.
