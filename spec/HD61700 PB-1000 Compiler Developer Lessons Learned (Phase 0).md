**Title: `HD61700 PB-1000 Compiler Developer Lessons Learned (Phase 0).md`**

**Introduction:**

This document summarizes the critical lessons learned during the extensive empirical validation performed in Phase 0 of the PB-1000 C Compiler project, particularly through the iterative development required to successfully complete Prompt 1 (Signed 8-bit comparison validation). Significant effort and time were invested to overcome initial assumptions and uncover the specific nuances and constraints of the Casio PB-1000 environment and its HD61700 CPU/Assembler. Adhering strictly to these findings has proven essential for reducing AI generation errors. This guide is intended to provide subsequent AI development phases (or other AI models) with this hard-won knowledge, preventing repetition of the same pitfalls and ensuring generated code is compliant and functional from the outset.

**Core Principle:** Standard assumptions about CPU architecture, assembly syntax, or BASIC interaction models **do not apply**. Adhere strictly to empirically validated behavior and documented constraints for this specific target.

**I. PB-1000 Built-in Assembler Constraints (Mandatory):**

1.  **Labels:**
    *   **Length:** Maximum 5 characters.
    *   **Syntax:** Must end with a colon (`:`) when defined (e.g., `LOOP1:`, `MYVAR: DS 1`).
    *   **`EQU` Labels:** Also require a colon (e.g., `ROMFUNC: EQU &H9664`).
    *   **Operand Usage:** Labels are **ONLY** valid as operands for `JP`, `JR`, and `CAL`. **Cannot** be used as immediate numeric values or address substitutes in `LD`, `ST`, `LDW`, `STW`, `AD`, `SB`, `AN`, `PRE`, `EQU` RHS, etc.
2.  **Literals:**
    *   **Valid:** Decimal (e.g., `10`), Hexadecimal (`&HFF`, `&h0a`). Case-insensitive prefix.
    *   **Invalid:** Binary literals (`&B...`) are **not supported**.
3.  **Instruction Set & Addressing Modes:**
    *   Use **only** instructions and addressing modes explicitly listed as valid formats in **Table 5.2 ("Full Instruction Formats and Byte Sizes")** of the `HD61700 PB-1000 Assembly Guide.md`. Do not infer modes.
    *   Specifically **avoid undocumented instructions/modes** (e.g., `LDM`, `STM`, `SX`/`SY`/`SZ` registers, stack pointer relative addressing `(US+/-offset)` for LD/ST).
4.  **Directives:**
    *   **Supported:** `ORG`, `START`, `EQU`, `DB`, `DS`.
    *   **Unsupported:** `DW` (use two `DB`s for words, respecting little-endian), `#IF`, `#INCLUDE`, macros, etc.
    *   **`DS` Behavior:** Reserves space but **does not initialize** memory (contents undefined). C semantics require zero-initialization for static/global data; compiler must generate explicit zeroing code (e.g., loop with `ST $31, ...` or `CLRME` ROM call if appropriate).
5.  **Source Structure:**
    *   `EQU` definitions must appear at the **top** of the file, before `ORG`.
    *   `START label` must typically follow `ORG address` immediately.

**II. HD61700 CPU Behavior & Comparison Implementation (Mandatory):**

1.  **Comparison Instructions:**
    *   Use **`SBC`** (8-bit) or **`SBCW`** (16-bit) for all comparisons (`==`, `!=`, `<`, `<=`, `>`, `>=`).
    *   `SBC`/`SBCW` perform `A - B` **only setting Flags (Z, C)**; they **do not modify** the destination register.
2.  **Subtraction Instruction:**
    *   Use **`SB`** (8-bit) or **`SBW`** (16-bit) when the **result** of `A - B` is required.
    *   `SB`/`SBW` **modify the destination register** and set Flags (Z, C).
3.  **Flag Interpretation (after `SBC`/`SBCW` or `SB`/`SBW`):**
    *   Capture flags immediately via `GFL $FlagsReg` *only if the raw flag byte itself is needed*. For comparisons, branch directly.
    *   `Z=0` (Zero Result / Equality): `($FlagsReg & &H80) == 0`. Test with `JR Z`, `JR NZ`.
    *   `C=1` (Borrow Occurred / Unsigned <): `($FlagsReg & &H40) != 0`. Test with `JR C`, `JR NC`.
4.  **Comparison Implementation (Use Conditional Branching Directly):**
    *   Branch immediately after `SBC`/`SBCW` using `JR Z/NZ/C/NC`. **Do not use `GFL`** followed by manual flag masking for comparison branches.
    *   **Unsigned:** Branch based directly on Z/C flag conditions (e.g., `A >= B` is `JR NC`).
    *   **Signed:** Requires checking operand signs *before* `SBC`/`SBCW` if signs might differ. If signs differ, branch based on signs alone. If signs are the same, perform `SBC`/`SBCW` and branch based on Z/C flags according to derived rules (Refer to Comparison Addendum v3 for full logic).
    *   **Prefer `JR`** (Relative Jump, range -127 to +127 bytes) over `JP` (Absolute Jump) for efficiency. Use `JP` only if the target is out of `JR` range.
5.  **Register `$30`/`$31`:** System ROM routines expect `$30=1` and `$31=0`. User code can read these but should avoid modifying them, especially around `CAL` instructions. If modified, they must be restored before calling ROM.

**III. BASIC/Assembly Interface Methodology (Mandatory):**

1.  **`CALL "filename.EXE"` Behavior:** This command loads the `.EXE` starting at the address specified by its **`ORG`** directive.
2.  **Memory Conflict:** If `ORG` is set low (e.g., `&H7000`) and BASIC uses `POKE` on the same addresses before `CALL`, the `POKE`d data **will be overwritten**.
3.  **Adopted Solution (`ORG` Adjustment):**
    *   Set **`ORG &H7004`** (or higher if more I/O space needed) in Assembly files.
    *   Set **`START MAIN`** pointing to the first executable code label.
    *   BASIC performs `POKE`/`PEEK` operations in the reserved low memory area (e.g., `&H7000`-`&H7003`).
    *   Assembly code reads inputs from and writes outputs to this low memory area using **literal addresses**.
4.  **Alternative (`CALCBUF`):** For larger data or specific needs, use `CALCBUF` (`&H6D1A`, 258 bytes) via literal addresses, ensuring CALC mode is not active.

**IV. PB-1000 BASIC Constraints (for Test Harness Generation):**

1.  **Variable Names:** No underscores (`_`). Use valid alphanumeric names starting with a letter.
2.  **Syntax:** Line numbers required. Use `:` before `REM` on the same line.
3.  **Literals:** Decimal and Hex (`&H...`) acceptable in `DATA`, `POKE`.
4.  **Filenames:** Use lowercase `.md` extension for log files.
5.  **Functions/Operators:** Assume standard set (`OPEN`, `PRINT#`, `CLOSE`, `DATA`, `READ`, `RESTORE`, `POKE`, `PEEK`, `CALL "filename"`, `HEX$`, `AND`, `/`, `INT`, `MOD`, `IF`, `GOTO`, `GOSUB`, `RETURN`, `END`, `DIM`, `ASC`, `MID$`, etc.). Use `INT(/)` for reliable integer division results. Be aware of `HEX$` padding.

**V. General Development Methodology:**

1.  **Raw Data First:** When validating CPU/ROM behavior, focus assembly tests on capturing raw outputs (register values, flag registers) with minimal processing.
2.  **Defer Interpretation:** Perform complex analysis and interpretation of raw data after collection.
3.  **Iterative Testing:** Expect errors and rely on testing in the target environment (emulator/hardware) and iterative refinement.
4.  **Constraint Adherence:** Meticulously follow all documented target constraints during code generation.

**VI. General PB-1000 Constraints**

1.  **ASCII Character Set** PB-1000 supports a character set similar to 7-bit ASCII.  Never generate PB-1000 code listings (Assembler or BASIC) with non ASCII characters.  For example, standard hypens only.  Non-ASCII dashes are forbidden.
2.  **Defer Interpretation:** Perform complex analysis and interpretation of raw data after collection.
3.  **Iterative Testing:** Expect errors and rely on testing in the target environment (emulator/hardware) and iterative refinement.
4.  **Constraint Adherence:** Meticulously follow all documented target constraints during code generation.

