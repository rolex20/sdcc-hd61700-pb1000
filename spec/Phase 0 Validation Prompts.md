**Title: Phase 0 Validation Prompts**

**Introduction for Target AI:**

*You are tasked with generating pairs of programs for Phase 0 ("Foundational Validation") of the HD61700 C Cross Compiler project. Each pair consists of:*
    1.  *A **Casio PB-1000 BASIC program** (C61 dialect, line numbers).*
    2.  *A corresponding **HD61700 assembly program** (for PB-1000 built-in assembler).*

*The goal is to capture **raw, essential data** about specific CPU instructions or ROM calls to create an **"HD61700 Behavior Reference Guide"**. The BASIC program is the test harness: it uses `POKE` to set up inputs (typically starting at `&H7000`), uses `CALL "TEST<N>.EXE"` to execute the assembly routine, uses `PEEK` to retrieve raw results, and logs these results into a structured **Markdown file** (e.g., `log<N>.md`, **lowercase extension**).*

*The assembly program performs the **minimal core operation** being tested, reads inputs from the location POKEd by BASIC, and stores the **raw results** (e.g., operation result byte/word, Flags Register content) back into memory for BASIC to PEEK.*

* **General Instructions:** Generate BASIC/Assembly code using only valid PB-1000 constructs (Ref: Table 5.2, BASIC syntax). Ensure correct structure (ORG &H7004 / START MAIN, I/O layout at &H7000-&H7003 or as specified) and apply **ALL lessons learned** (incl. 5-char labels, literal addresses [non-branch], no binary literals, SBC vs SB, JR pref.). Use v<M> comment format (`10 REM ' TEST<N>.BAS (v<M>)` / `; TEST<N>.ASM (v<M>)`).*

*Use the **Reference Example Programs (Prompt 1)** below as the definitive template.*

---

**Prompt 1: Signed 8-bit Subtraction Raw Data Capture (Reference Example)**

**(Objective):** Capture raw 8-bit subtraction result and Flag Register content after the `SB` instruction for various signed 8-bit inputs.

**(Implementation Note & Context):**
*   *This prompt's corresponding BASIC and Assembly programs (`TEST1.BAS v9`, `TEST1.ASM v11`) were successfully developed and executed through an iterative process. The resulting raw data (`log1.md`) was critical in establishing the definitive ground truth for HD61700 flag behavior after subtraction (which needs to be used for comparison, particulary 'SBC/SBCW').*
*   *This verified flag behavior allowed for the creation of the section **"Implementing Comparisons on the HD61700 using SBC/SBCW"** in the assembly language guide document, which details the correct logic for all signed/unsigned comparisons.*
*   *The final, working versions of `TEST1.BAS (v9)` and `TEST1.ASM (v11)` are shown below as reference templates for subsequent prompts.*
*   *This iterative validation process generated a detailed list of "Lessons Learned" regarding PB-1000 constraints and development methodology, which will be provided separately to the AI developing the compiler.*


**(Reference BASIC Program - `TEST1.BAS (v9)`):**
```basic
10 REM ' TEST1.BAS (v9) - Signed 8-bit SB Test Driver
20 CLS : PRINT "Running Signed 8bit SB Raw Data Capture..."
30 OPEN "log1.md" FOR OUTPUT AS #1 : REM ' Overwrite/Create log file .md (lowercase)
40 PRINT #1,"# Signed 8-bit Subtraction (`SB`) Raw Results"
50 PRINT #1,""
60 PRINT #1,"| Input A (Signed) | Input B (Signed) | SB Result (Signed Dec) | Flags Register (Dec) | Notes                       |"
70 PRINT #1,"| :--------------- | :--------------- | :---------------------- | :------------------- | :-------------------------- |"
80 CLOSE #1
90 GOSUB 1000 : REM ' Run the tests from DATA
100 PRINT "Tests complete. Check log1.md"
110 END

1000 REM ' Subroutine to run tests from DATA
1100 RESTORE 2031 : REM ' Point READ DIRECTLY to the first DATA line number
1110 READ A, B, N$ : REM ' Read pair AND Note String
1120 IF A = 999 THEN RETURN : REM ' 999 is end-of-data marker
1130 REM ' Convert raw inputs A, B to signed decimal DA, DB for status/log
1131 IF A > 127 THEN DA = A - 256 ELSE DA = A
1132 IF B > 127 THEN DB = B - 256 ELSE DB = B
1135 PRINT "Testing A="; DA; ", B="; DB; " ("; N$; ")"
1140 POKE &H7000, A : REM ' POKE A (Raw byte value, handles hex input)
1150 POKE &H7001, B : REM ' POKE B (Raw byte value)
1170 CALL "TEST1.EXE" : REM ' CALL the assembly program directly
1180 SBCRESRAW = PEEK(&H7002) : REM ' Read RAW 8-bit SUBTRACTION result byte
1185 FLAGSREG = PEEK(&H7003) : REM ' Read 8-bit FLAGS register content
1186 REM ' Convert raw SB result byte to signed decimal for display
1187 IF SBCRESRAW > 127 THEN DSBCRES = SBCRESRAW - 256 ELSE DSBCRES = SBCRESRAW
1190 OPEN "log1.md" FOR APPEND AS #1 : REM ' Append results to .md file (lowercase)
1290 REM ' --- Format Markdown Table Row (Use Signed values for display) ---
1310 PRINT #1,"| "; DA; " | "; DB; " | "; DSBCRES; " | "; FLAGSREG; " | "; N$; " |"
1320 CLOSE #1
1330 GOTO 1110 : REM ' Read next pair

2000 REM ' --- TEST DATA (Using &H for negative numbers raw byte values) ---
2010 REM ' Format: DATA NumberA, NumberB, Note$
2020 REM ' Marker for end of data is A=999
2030 REM ' Pos vs Pos (A < B)
2031 DATA 5, 10, "Pos vs Pos (<)"
... (rest of DATA lines as before) ...
2310 REM ' End of Data Marker
2311 DATA 999, 0, "End"
```

**(Reference Assembly Program - `TEST1.ASM (v11)`):**
```assembly
; TEST1.ASM (v11)
; Performs Signed 8-bit SB (Subtract) and stores raw results.
; Uses SB instruction which MODIFIES destination register.
; Input:  &H7000 = Signed Byte A
;         &H7001 = Signed Byte B
; Output: &H7002 = Raw 8-bit result of SB $2, $3 (A - B)
;         &H7003 = Raw 8-bit content of Flags register after SB
; IMPORTANT: ORG &H7004, Uses SB, LITERAL addresses, HEX/DEC literals, 5-char labels.

      ORG   &H7004       ; Program IMAGE starts at &H7004
      START MAIN         ; Execution starts at the MAIN label

      ; --- Start of Executable Code ---
MAIN:
      ; --- Load Operands ---
      LDW   $0, &H7000   ; Load LITERAL ADDRESS of Input A
      LD    $2, ($0)     ; Load DATA Byte A into $2
      LDW   $0, &H7001   ; Load LITERAL ADDRESS of Input B
      LD    $3, ($0)     ; Load DATA Byte B into $3

      ; --- Perform Subtraction & Get Flags ---
      SB    $2, $3       ; Calculate $2 = $2 - $3. MODIFIES $2. Affects Z, C.
      GFL   $4           ; Copy Flag register F into $4 immediately after SB

      ; --- Store Raw Results ---
      LDW   $0, &H7002   ; Load LITERAL ADDRESS for SB result output
      ST    $2, ($0)     ; Store raw SB result (NOW MODIFIED $2) to &H7002

      LDW   $0, &H7003   ; Load LITERAL ADDRESS for Flags result output
      ST    $4, ($0)     ; Store raw Flags register ($4) to &H7003

      RTN                ; Return to BASIC
```

**(Generation Request - *For Reference Only*):** *Generate the BASIC and Assembly programs as shown above.*

**(Critical BASIC Dialect Assumptions):**
*   File I/O: `OPEN "filename" FOR OUTPUT/APPEND AS #1`, `PRINT #1`, `CLOSE #1`.
*   Data Handling: `DATA value1, value2, str$`, `RESTORE linenum`, `READ var1, var2, var$`. Accepts decimal and `&Hxx`.
*   Memory: `POKE address, value`, `PEEK(address)`.
*   Execution: `CALL "filename"`.
*   String/Math: `HEX$(value)`, `AND` (bitwise), `/` (division), `INT(value)`, `+`, `-`, `MOD`. String concatenation with `;`.
*   Control Flow: `IF condition THEN statement`, `GOTO linenum`, `GOSUB linenum`, `RETURN`, `END`, `DIM`.
*   Syntax: Line numbers required. `:` separates statements from `REM` on same line. Valid variable names (no `_`).

---
*(Prompts 2, 3, 4 are excluded)*
---

**Prompt 5: Arithmetic/Logical Flag Raw Data Capture**

```prompt
Generate two programs: `TEST5.BAS (v2)` (generates `log5.md`) and `TEST5.ASM (v2)` (compiles to `TEST5.EXE`).

**Objective:** Capture raw Flag Register content after various 8-bit arithmetic (`AD`, `SB`) and logical (`AN`, `OR`, `XR`) instructions, plus complement (`CMP`, `INV`).

**BASIC Program (`TEST5.BAS v2`):**
1.  `10 REM ' TEST5.BAS (v2)'`
2.  Use `DATA` for 8-bit pairs (A, B, Note$) testing zero results, carries, bit patterns. Use Hex (`&Hxx`). End marker (999, 0, "End").
3.  Set up Markdown output to `log5.md` (lowercase): `| Input A (Hex) | Input B (Hex) | Operation | Flags Register (Dec) | Notes |`.
4.  `DIM OPS(7)`. Assign names `OPS(1)="AD": OPS(2)="SB": ... OPS(7)="INV A"`.
5.  Loop: `RESTORE` (to first data line), `READ A, B, N$`, check marker.
6.  Inside loop: `POKE &H7000, A`, `POKE &H7001, B`.
7.  `CALL "TEST5.EXE"`.
8.  Open `log5.md` for APPEND.
9.  `PRINT #1, "--- Inputs A="; HEX$(A); ", B="; HEX$(B); " ("; N$; ") ---"`
10. Add Markdown table header/separator: `| Operation | Flags Register (Dec) | Notes |` / `| :-------- | :------------------- | :---- |`
11. Loop I from 1 to 7:
    *   `FLAGSREG = PEEK(&H7001 + I)` (Results &H7002 to &H7008).
    *   `PRINT #1, "| "; OPS(I); " | "; FLAGSREG; " | |"`
12. End Inner Loop. `CLOSE #1`, loop back.
13. Add global Markdown header `# 8-bit Arithmetic/Logical Flag Raw Results`.

**Assembly Program (`TEST5.ASM v2`):**
1.  `; TEST5.ASM (v2)`
2.  Use `ORG &H7004`, `START MAIN`.
3.  `MAIN:` label.
4.  **Test AD:** Load A(&H7000) to $2, B(&H7001) to $3. `AD $2, $3`. `GFL $4`. `ST $4, (&H7002)`. (Use literal addresses).
5.  **Test SB:** Load A to $2, B to $3 (Reload!). `SB $2, $3`. `GFL $4`. `ST $4, (&H7003)`.
6.  **Test AN:** Load A to $2, B to $3 (Reload!). `AN $2, $3`. `GFL $4`. `ST $4, (&H7004)`.
7.  **Test OR:** Load A to $2, B to $3 (Reload!). `OR $2, $3`. `GFL $4`. `ST $4, (&H7005)`.
8.  **Test XR:** Load A to $2, B to $3 (Reload!). `XR $2, $3`. `GFL $4`. `ST $4, (&H7006)`.
9.  **Test CMP:** Load A to $2 (Reload!). `CMP $2`. `GFL $4`. `ST $4, (&H7007)`.
10. **Test INV:** Load A to $2 (Reload!). `INV $2`. `GFL $4`. `ST $4, (&H7008)`.
11. `RTN`.
12. **Constraint Check:** Generate BASIC/Assembly code using only valid PB-1000 constructs (Ref: Table 5.2, BASIC syntax). Ensure correct structure (ORG/START, I/O layout) and apply ALL lessons learned (incl. label rules, literal usage, SBC/SB, JR pref.). Use v<M> comment.

**Critical BASIC Dialect Assumptions:**
*   File I/O, Data Handling, Memory, Execution (`CALL`), String/Math (`HEX$`, `AND`, `/`, `INT`, etc.), Control Flow, Syntax (Line numbers, `:`, no `_`), **`DIM`** (for `OPS()` array).
```

---

**Prompt 6: Shift/Rotate Flag Raw Data Capture**

```prompt
Generate two programs: `TEST6.BAS (v2)` (generates `log6.md`) and `TEST6.ASM (v2)` (compiles to `TEST6.EXE`). Use Prompt 1 Reference Example structure/style/methodology/constraints.

**Objective:** Capture raw Flag Register content after 8-bit (`ROU`, `ROD`, `BIU`, `BID`) and 16-bit (`ROUW`, `RODW`, `BIUW`, `BIDW`) shift/rotate instructions.

**BASIC Program (`TEST6.BAS v2`):**
1.  `10 REM ' TEST6.BAS (v2)'`
2.  Use `DATA` for 8-bit A, 16-bit B, Note$. Include cases testing carry in/out, zero/non-zero results, patterns. Use Hex (`&Hxx`, `&Hxxxx`). End marker (999, 99999, "End").
3.  Set up Markdown output to `log6.md` (lowercase): `| Input A (Hex) | Input B (Hex) | Operation | Flags Register (Dec) | Notes |`.
4.  `DIM OPS(8)`. Assign names `OPS(1)="ROU A": ... OPS(8)="BIDW B"`.
5.  Loop: `RESTORE` (to first data line), `READ A, B, N$`, check marker.
6.  Inside loop: `POKE &H7000, A`. `BL=B MOD 256`, `BH=INT(B/256)`. `POKE &H7001, BL`, `POKE &H7002, BH`.
7.  `CALL "TEST6.EXE"`.
8.  Open `log6.md` for APPEND.
9.  `PRINT #1, "--- Inputs A="; HEX$(A); ", B="; HEX$(B); " ("; N$; ") ---"`
10. Add Markdown table header/separator.
11. Loop I from 1 to 8:
    *   `FLAGSREG = PEEK(&H7003 + I)` (Results &H7004 to &H700B).
    *   `PRINT #1, "| "; HEX$(A); " | "; HEX$(B); " | "; OPS(I); " | "; FLAGSREG; " | |"`
12. End Inner Loop. `CLOSE #1`, loop back.
13. Add global Markdown header `# Shift/Rotate Flag Raw Results`.

**Assembly Program (`TEST6.ASM v2`):**
1.  `; TEST6.ASM (v2)`
2.  Use `ORG &H7004`, `START MAIN`.
3.  `MAIN:` label.
4.  **Test ROU:** Load A from `&H7000` to $2. `ROU $2`. `GFL $4`. `ST $4, (&H7004)`. (Use literal addresses).
5.  **Test ROD:** Load A from `&H7000` to $2 (Reload!). `ROD $2`. `GFL $4`. `ST $4, (&H7005)`.
6.  **Test BIU:** Load A from `&H7000` to $2 (Reload!). `BIU $2`. `GFL $4`. `ST $4, (&H7006)`.
7.  **Test BID:** Load A from `&H7000` to $2 (Reload!). `BID $2`. `GFL $4`. `ST $4, (&H7007)`.
8.  **Test ROUW:** Load B from `&H7001`/`&H7002` to $4/$5 (Reload!). `ROUW $4`. `GFL $6`. `ST $6, (&H7008)`.
9.  **Test RODW:** Load B from `&H7001`/`&H7002` to $4/$5 (Reload!). `RODW $4`. `GFL $6`. `ST $6, (&H7009)`.
10. **Test BIUW:** Load B from `&H7001`/`&H7002` to $4/$5 (Reload!). `BIUW $4`. `GFL $6`. `ST $6, (&H700A)`.
11. **Test BIDW:** Load B from `&H7001`/`&H7002` to $4/$5 (Reload!). `BIDW $4`. `GFL $6`. `ST $6, (&H700B)`.
12. `RTN`.
13. **Constraint Check:** Generate BASIC/Assembly code using only valid PB-1000 constructs (Ref: Table 5.2, BASIC syntax). Ensure correct structure (ORG/START, I/O layout) and apply ALL lessons learned (incl. label rules, literal usage, SBC/SB, JR pref.). Use v<M> comment.

**Critical BASIC Dialect Assumptions:** (List same assumptions as Prompt 1, plus `DIM` for string array `OPS()`)
```

---

**Prompt 7: Addressing Mode Raw Data Capture**

```prompt
Generate two programs: `TEST7.BAS (v2)` (generates `log7.md`) and `TEST7.ASM (v2)` (compiles to `TEST7.EXE`). Use Prompt 1 Reference Example structure/style/methodology/constraints.

**Objective:** Verify basic functionality of register indirect (`($n)`) and indexed (`(IX+offset)`, `(IZ+offset)`) addressing modes by storing and reloading known values. Use `CALCBUF` for target memory.

**BASIC Program (`TEST7.BAS v2`):**
1.  `10 REM ' TEST7.BAS (v2)'`
2.  Define test params: `PTR1=&H6D1A`, `PTR2=&H6D2A`, `OFF=5`, `VAL8=&HAA`, `VAL16=&H1234`.
3.  Set up Markdown output to `log7.md` (lowercase). Header: `| Operation | Address (Hex) | Expected (Hex) | Actual Written (Hex) | Read Back (Hex) | Status |`.
4.  Zero out CALCBUF areas: `FOR I=0 TO 3 : POKE PTR1+I, 0 : NEXT I`, `FOR I=0 TO 15 : POKE PTR2+I, 0 : NEXT I`.
5.  `POKE` Ptr1(L/H), Ptr2(L/H), Off, Val8, Val16(L/H) into setup area `&H7000`-`&H7007`.
6.  `CALL "TEST7.EXE"`.
7.  Open `log7.md` for APPEND.
8.  **Verify Stores:** PEEK memory locations `PTR1`, `PTR1+1`, `PTR2+OFF`, `PTR2+10`, `PTR2+11`. Store as `MWINDL`, `MWINDH`, `MWIDX8`, `MWIDX16L`, `MWIDX16H`. (No underscores).
9.  **Verify Loads:** PEEK results from `&H7010`-`&H7018`. Store as `RLIND8`, `RLIND16L`, `RLIND16H`, `RLIDX8IX`, `RLIDX16LIX`, `RLIDX16HIX`, `RLIDX8IZ`, `RLIDX16LIZ`, `RLIDX16HIZ`. Reconstruct 16-bit values `RLIND16 = RLIND16L+RLIND16H*256`, etc.
10. **Log Table Rows:** Print rows comparing Expected (`VAL8`, `VAL16`) vs Actual Memory Writes (`MW...`) and Actual Register Reads (`RL...`). Use HEX$ for values. Add OK/FAIL status. Handle ST/STW overwrite note.
11. `CLOSE #1`.

**Assembly Program (`TEST7.ASM v2`):**
1.  `; TEST7.ASM (v2)`
2.  Use `ORG &H7004`, `START MAIN`.
3.  `MAIN:` label.
4.  Load Ptr1 L/H from literal `&H7000`/`&H7001` into `$10/$11`.
5.  Load Ptr2 L/H from literal `&H7002`/`&H7003` into `$12/$13`.
6.  Load Offset from literal `&H7004` into `$14`.
7.  Load Val8 from literal `&H7005` into `$2`.
8.  Load Val16 L/H from literal `&H7006`/`&H7007` into `$4/$5`.
9.  **Reg Indirect:** `LDW $0, $10` (Load Ptr1 value into $0/$1). `ST $2, ($0)` (Store Val8 - overwritten). `STW $4, ($0)` (Store Val16). `LD $6, ($0)` (Read byte after STW). `LDW $8, ($0)` (Read word after STW). `ST $6, (&H7010)`. `STW $8, (&H7011)`. (Store results using literal addresses `&H7010` and `&H7011`/`&H7012`).
10. **Indexed IX:** `PRE IX, $12` (Use Ptr2 value). `ST $2, (IX+$14)` (Use offset register). `STW $4, (IX+10)` (Use immediate offset). `LD $6, (IX+$14)`. `LDW $8, (IX+10)`. `ST $6, (&H7013)`. `STW $8, (&H7014)`. (Store results using literal addresses `&H7013` and `&H7014`/`&H7015`).
11. **Indexed IZ:** `PRE IZ, $12`. `ST $2, (IZ+$14)`. `STW $4, (IZ+10)`. `LD $6, (IZ+$14)`. `LDW $8, (IZ+10)`. `ST $6, (&H7016)`. `STW $8, (&H7017)`. (Store results using literal addresses `&H7016` and `&H7017`/`&H7018`).
12. `RTN`.
13. **Constraint Check:** Generate BASIC/Assembly code using only valid PB-1000 constructs (Ref: Table 5.2, BASIC syntax). Ensure correct structure (ORG/START, I/O layout) and apply ALL lessons learned (incl. label rules, literal usage, SBC/SB, JR pref.). Use v<M> comment.

**Critical BASIC Dialect Assumptions:** (List same assumptions as Prompt 1)
```

---

**Prompt 8: ROM Call Register Clobber Raw Data Capture (PRLB1 &h9664)**

```prompt
Generate two programs: `TEST8.BAS (v3)` (generates `log8.md`) and `TEST8.ASM (v3)` (compiles to `TEST8.EXE`). Use Prompt 1 Reference Example structure/style/methodology/constraints.

**Objective:** Capture initial and final raw register values ($0-$31) AND initial/final Flags register content around a call to the `PRLB1` (&h9664) ROM routine. Use `CALCBUF` for the test string.

**BASIC Program (`TEST8.BAS v3`):**
1.  `10 REM ' TEST8.BAS (v3)'`
2.  Define string params: `STRADDR = &H6D1A`, `STRLEN = 4`, `S$ = "Test"`.
3.  `DIM RINIT(31), RFINAL(31)`.
4.  `POKE` initial values (0-31) into memory `&H7000 - &H701F`. Store in `RINIT()`.
5.  `POKE` StrAddr L/H into `&H7020`/`&H7021`.
6.  `POKE` StrLen L/H into `&H7022`/`&H7023`.
7.  `POKE` string S$ byte-by-byte to STRADDR (&H6D1A...), add null terminator (0).
8.  `POKE &H690C, 0` (Set OUTDV=Display).
9.  `CALL "TEST8.EXE"`.
10. `FLAGSINIT = PEEK(&H7024)`.
11. `PEEK` final register values from memory `&H7030 - &H704F`. Store in `RFINAL()`.
12. `FLAGSFINAL = PEEK(&H7050)`.
13. Open `log8.md` (lowercase) FOR OUTPUT.
14. `PRINT #1, "# ROM Call Clobber Test: PRLB1 (&h9664)"`
15. `PRINT #1, "| Item      | Initial (Dec) | Final (Dec) | Status    |"`
16. `PRINT #1, "| :-------- | :------------ | :---------- | :-------- |"`
17. `PRINT #1, "| Flags Reg | "; FLAGSINIT; " | "; FLAGSFINAL; " | ";`
18. `IF FLAGSINIT <> FLAGSFINAL THEN PRINT #1, "CLOBBERED |" ELSE PRINT #1, "Preserved |"`
19. Loop I from 0 to 31:
    *   `PRINT #1, "| $"; I; " | "; RINIT(I); " | "; RFINAL(I); " | ";`
    *   `IF RINIT(I) <> RFINAL(I) THEN PRINT #1, "CLOBBERED |" ELSE PRINT #1, "Preserved |"`
20. End Loop.
21. `IF RFINAL(30)<>1 THEN PRINT #1, "\n**WARNING:** $30 final value is not 1!"`.
22. `IF RFINAL(31)<>0 THEN PRINT #1, "\n**WARNING:** $31 final value is not 0!"`.
23. `CLOSE #1`.

**Assembly Program (`TEST8.ASM v3`):**
1.  `; TEST8.ASM (v3)`
2.  Use `ORG &H7004`, `START MAIN`.
3.  `PRLB1: EQU &H9664` (Define ROM addr with colon).
4.  `MAIN:` label.
5.  **Load Initial Registers:** Use explicit `LD`s or loop: Load $0-$31 from literal addresses `&H7000`-`&H701F`. (e.g., `LDW $0,&H7000; LD $0,($0)` ...).
6.  **Store Initial Flags:** `GFL $4`. `ST $4, (&H7024)` (Use literal address).
7.  **Preserve/Set $30/$31:** AFTER load, ensure `$30=1`, `$31=0`: `LD $30, 1`, `LD $31, 0`.
8.  **Prepare PRLB1 Inputs:** Load StrAddr word from literal `&H7020` into `$15/$16`. Load StrLen word from literal `&H7022` into `$17/$18`.
9.  **Call ROM:** `CAL PRLB1`.
10. **Store Final Flags:** `GFL $4`. `ST $4, (&H7050)` (Use literal address).
11. **Store Final Registers:** Use explicit `ST`s or loop: Store $0-$31 into literal addresses `&H7030`-`&H704F`. (e.g., `LDW $0,&H7030; ST $0,($0)` ...).
12. `RTN`.
13. **Constraint Check:** Generate BASIC/Assembly code using only valid PB-1000 constructs (Ref: Table 5.2, BASIC syntax). Ensure correct structure (ORG/START, I/O layout) and apply ALL lessons learned (incl. label rules, literal usage, SBC/SB, JR pref.). Use v<M> comment.

**Critical BASIC Dialect Assumptions:** (List assumptions as in Prompt 1, plus `DIM`, `MID$`, `ASC`)
```

---

**Prompt 9: Comparison Logic Branch Validation (16-bit)**

```prompt
Generate two programs: `TEST9.BAS (v2)` (generates `log9.md`) and `TEST9.ASM (v2)` (compiles to `TEST9.EXE`). Use Prompt 1 Reference Example structure/style/methodology/constraints.

**Objective:** Validate the *branching logic* implementation (from Addendum v3) for **unsigned 16-bit A >= B** and **signed 16-bit A < B** comparisons using `SBCW`.

**BASIC Program (`TEST9.BAS v2`):**
1.  `10 REM ' TEST9.BAS (v2)'`
2.  Use `DATA` statements with **16-bit** pairs (A, B) and Notes$. Reuse relevant cases from TEST3/TEST4 data. Use Hex (`&Hxxxx`). End marker (99999, 0, "End").
3.  Set up Markdown output to `log9.md` (lowercase). Header: `| Input A | Input B | Condition Tested | Branch Outcome (1=TRUE, 0=FALSE) | Notes |`.
4.  Loop: `RESTORE` (to first data line), `READ A, B, N$`, check marker.
5.  Inside loop: Calc `AL, AH, BL, BH`. `POKE &H7000, AL`, `POKE &H7001, AH`. `POKE &H7002, BL`, `POKE &H7003, BH`.
6.  `CALL "TEST9.EXE"`.
7.  `RUGE = PEEK(&H7004)` (Unsigned >= result).
8.  `RSLT = PEEK(&H7005)` (Signed < result).
9.  Open `log9.md` for APPEND.
10. Convert A, B to signed DA, DB for display if needed.
11. `PRINT #1, "| "; A; " | "; B; " | "; "Unsigned A>=B"; " | "; RUGE; " | "; N$; " |"` (Use raw A, B)
12. `PRINT #1, "| "; DA; " | "; DB; " | "; "Signed A<B"; " | "; RSLT; " | "; N$; " |"` (Use signed DA, DB)
13. `CLOSE #1`, loop back.

**Assembly Program (`TEST9.ASM v2`):**
1.  `; TEST9.ASM (v2)`
2.  Use `ORG &H7004`, `START MAIN`.
3.  `MAIN:` label.
4.  Load Word A from literal `&H7000` into `$2/$3`. Load Word B from literal `&H7002` into `$4/$5`.

5.  **Implement Unsigned 16-bit >= Logic:**
    *   `SBCW  $2, $4`
    *   `JR    NC, UGETR`  ; Jump if A >= B (unsigned)
    *   ; False case
    *   `LD    $5, 0`
    *   `JR    UGEDN`     ; (Use JP if needed for range -127 to +127)
    *   `UGETR: LD $5, 1`    ; True case
    *   `UGEDN: LDW $0, &H7004` ; Literal address for OUT_UGE
    *   `ST    $5, ($0)`     ; Store 0 or 1

6.  **Implement Signed 16-bit < Logic:**
    *   ; Operands $2/$3 (A) and $4/$5 (B) are already loaded.
    *   ; Check Signs using HIGH bytes ($3, $5)
    *   `LD    $6, $3`      ; Copy High Byte A
    *   `LD    $7, $5`      ; Copy High Byte B
    *   `AN    $6, &H80`    ; Isolate SignA bit
    *   `AN    $7, &H80`    ; Isolate SignB bit
    *   `LD    $8, $6`
    *   `XR    $8, $7`      ; Test if signs same (Z=0)
    *   `JR    Z, SGNSE`    ; Jump if signs are SAME
    *   ; Signs are DIFFERENT
    *   `JR    Z, ISFAL`    ; If SignA ($6) is zero, A is pos -> LT is FALSE
    *   ; A is neg, B is pos -> LT is TRUE
    *   `JR    ISTRU`     ; (Use JP if needed for range -127 to +127)
    *   `SGNSE:`
    *   ; Signs are SAME
    *   `SBCW  $2, $4`      ; Use SBCW here
    *   `JR    C, ISTRU`    ; If C=1, LT is TRUE
    *   ; No carry -> LT is FALSE
    *   `ISFAL:`
    *   `LD    $5, 0`       ; Set result FALSE
    *   `JR    SLTDN`     ; (Use JP if needed for range -127 to +127)
    *   `ISTRU:`
    *   `LD    $5, 1`       ; Set result TRUE
    *   `SLTDN:`
    *   `LDW   $0, &H7005`   ; Literal address for OUT_SLT
    *   `ST    $5, ($0)`     ; Store 0 or 1

7.  `RTN`.
8.  **Constraint Check:** Generate BASIC/Assembly code using only valid PB-1000 constructs (Ref: Table 5.2, BASIC syntax). Ensure correct structure (ORG/START, I/O layout) and apply ALL lessons learned (incl. label rules, literal usage, SBC/SB, JR pref.). Use v<M> comment.

**Critical BASIC Dialect Assumptions:** (List same assumptions as Prompt 1)
```

---

This revised set of prompts (labeled Final v4 internally) incorporates all the fixes and requested changes.

Would you like me to execute the plan again now to review this final set of prompts (1, 5, 6, 7, 8, 9) for any remaining errors?