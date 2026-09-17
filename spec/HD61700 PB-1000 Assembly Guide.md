# HD61700 PB-1000 Assembly Guide

# 1. Introduction

This guide details the necessary technical information to guide the development of a C cross compiler for the Casio PB-1000. The compiler's goal is to compile a C language program into HD61700 assembly instructions in a text file. This generated assembly instructions must be syntactically correct and functionally compatible with the assembler built into the Casio PB-1000 handheld computer.


The original manuals from the CASIO PB-1000 are available only in physical paperback format.  There are only some bad quality PDF scanned and there is a concern that uploading those documents might exhaust the token context size of the AI model in charge of developing the actual C Cross compiler.

**Constraint:** Features specific only to the "HD61 PC-based cross-assembler" (like enhanced directives, expression evaluation, longer labels, undocumented instructions/modes, and any other feature tagged as undocumented) must be avoided by the C cross compiler.

# 2. Target Environment: Casio PB-1000

*   **CPU:** Hitachi HD61700.
*   **C61-BASIC:** Built-in BC61-BASIC interpreter dialect based on Japan Industrial Standard (JIS) BASIC (C6207).
*   **Assembler:** Built-in assembler within the PB-1000. This assembler has limitations compared to modern or even the documented HD61 PC cross-assembler.
    *   **Syntax:** Strict format required.
    *   **Labels:** Limited to 5 characters (`@`, `_`, A-Z, a-z, 0-9; cannot start with a digit). Case insensitive.  Labels can be freely declared, but only JP, JR and CAL commands may be used as operands with label names.
    *   **Expressions:** Not supported, only pre-calculate values.
    *   **Directives:** Supports a limited set (`ORG`, `START`, `EQU`, `DB`, `DS`). `DW` is *not* supported.
    *   **Addressing Modes:** Do not use undocumented features, they work only on the cross assembler program available by third parties for the PC.  The internal or built-in assembler in the PB-1000 does not support any undocumented features and this is the target assembler. Stick to documented, common modes verified by examples or basic documentation.
*   **Memory:**
    *   User machine code programs typically reside starting at address `&H7000`. This can be the default `ORG` address unless overridden.
    *   The memory map shows system variables below `&H7000`. The compiler should generally avoid generating code that accesses addresses below `&H7000` unless interfacing with specific system routines or variables intentionally.
    *   Stack pointers (`SS`, `US`) manage specific memory regions (see Architecture section).
    *   RAM structure includes areas for BASIC programs, variables, stacks, and I/O buffers. The compiler needs to manage its own data and stack usage within the available user RAM, typically growing downwards from near the top of RAM or upwards from `&H7000`/`IOBF`.
	*   The Casio PB-1000 has an ASCII table very similar Intel 8088.
	*   Memory capacity (user area, excluding ROM): 36864 bytes with the always included and preinstalled RAM expansion pack.
	*   The maximum machine language program binary is 4096 bytes.
	*   System stack: 255 bytes starts at `&H6A4B`.
	*   User stack: 249 bytes starts at `&H6952`.
*   Built-in clock function.
*   Built-in screen: LCD 32 columns by 4 line (192 x 32 dots).  Be **very** **concise** with text messages on the screen.


# 3. HD61700 CPU Architecture (Relevant for Compiler)

*   **Registers:**
    *   **Main Registers ($0 - $31):** 32 x 8-bit general-purpose registers residing in internal RAM. These are the primary work registers for data manipulation. They can be used individually (8-bit) or in adjacent pairs (16-bit).
    *   **16-bit Operations:** Use register pairs `$n` and `$n+1`. The byte order is **little-endian** (e.g., for `LDW $0, &H1234`, register `$0` gets `&H34` (low byte) and register `$1` gets `&H12` (high byte)). This is crucial for representing 16-bit data types.
    *   **Index Registers (IX, IY, IZ):** 16-bit registers used primarily for memory addressing (pointers). `IX` and `IZ` support indexed addressing with offsets. `IY` is mainly used as the end-pointer for block transfer/search operations (`BUP`, `BDN`, `SUP`, `SDN`).
    *   **Stack Pointers (SS, US):** 16-bit registers.
        *   `SS` (System Stack Pointer): Used by hardware for calls (`CAL`) and interrupts. Can also be used via `PHS`/`PPS` instructions. Grows *downwards* (pre-decrement push, post-increment pop).  
        *   `US` (User Stack Pointer): Available for general-purpose stack operations via `PHU`/`PPU`. Grows *downwards* (pre-decrement push, post-increment pop). The compiler can use `US` for function call frames, local variables, etc.
    *   **Program Counter (PC):** 16-bit, points to the *next* instruction to be fetched. Modified by jumps, calls, returns.

    *   **Flag Register (F):** 8-bit. Contains status flags affected by arithmetic, logical and system operations.
        *   `Z` Zero Flag.  Bit 7.  Reset to 0 when all the bits of a calculation result are 0 (Z), and set to 1 when data are present. (NZ)
        *   `C` Carry Flag.  Bit 6.  Set to 1 when a carry or borrow occurs (C), and reset to 0 when a carry or borrow does not occur. (NC)
        *   `LZ` Lower Digit Zero Flag.  Bit 5.  Reset to 0 when the low-order 4 bits are 0 due to a calculation result (LZ), and set to 1 when data are present.
        *   `UZ` Upper Digit Zero Flag.  Bit 4.  Reset to 0 when the high-order 4 bits are 0 (UZ), and set to 1 when data are present.
        *   `SW` Power Switch State Flag.  Bit 3.  Indicates the ON/OFF status of the power switch. This flag is set to 1 when power is ON, and reset to 0 when power is OFF.
        *   `APO` Auto Power Off State Flag. Bit 2.  Set to 1 when an OFF command is executed while the power switch is ON, and reset to 0 when the power switch is OFF.
		*    Bits 0 and 1 are not used in the system.
		*    Note the inverted logic description for the 'Z' Zero Flag.  This behavior has been verified, it is consistent and crucial for equality checks.
		*    The `APO` and `SW` hardware flags can only be referenced by user programs, they cannot be modified.
		*    When working directly with the Flags register you can copy the Flags register with `GFL`, **you could test Z or C** like in the folowing example, however for simple Zero, or Carry tests, it is better to use `CAL Z`, `JR NZ`, `JP C`, etc.

```assembly
GFL $4      ; copy F
    AN  $4,&H80 ; keep only bit 7 → test Z  
    AN  $4,&H40 ; keep only bit 6 → test C
```

    *   **Specific Index Registers (SX, SY, SZ):** *AVOID USING THESE.* They are undocumented for the PB-1000 platform, provide shortcuts for `$31`, `$30`, `$0` respectively, and are an optimization feature of a third party, open-source HD61 cross-assembler for DOS/PC. Generating code relying on them will fail on the built-in PB-1000's assembler.
    *   **Other Registers (IE, IA, UA, PE, PD, TM, IB, KY):** Primarily for hardware control (Interrupts, I/O, Banking, Timer, Keyboard). A compiler will typically not generate code to manipulate any these directly, not even `UA` for banking since the system internally handles banking.

*   **Memory Addressing Modes (PB-1000 Compatible):**
    *   **Register Direct:** `$n` (for operands where registers are allowed).
    *   **Immediate:** `value` (e.g., `LD $0, 10`, `AD $1, &H20`). Used for 8-bit (`IM8`) and 16-bit (`IM16`, e.g., `LDW $0, &H1234`, `PRE IX, &H7000`).
    *   **Register Indirect:** `($n)` - Uses the 16-bit value in register pair `$n`/`$n+1` as a memory address (e.g., `LD $2, ($0)`, `STW $4, ($6)`).
    *   **Indexed:** `(IX+offset)`, `(IX-offset)`, `(IZ+offset)`, `(IZ-offset)`.
        *   `offset` can be an 8-bit immediate value (`IM8`): `(IX+10)`, `(IZ-&H20)`.
        *   `offset` can be an 8-bit register (`$m`): `(IX+$0)`, `(IZ-$5)`.
        *   *Avoid* modes using SX/SY/SZ as offsets (e.g., `(IX+$SX)`).
    *   **Stack Addressing:** Implicit via `PHS/PPS/PHU/PPU` and their word/multi-byte variants, using `SS` or `US`.
	*   **No need for alignment**.

*   **Instruction Types:** The CPU supports both 8-bit and 16-bit operations, clearly distinguished by mnemonics (e.g., `LD` vs `LDW`, `AD` vs `ADW`).

*   **System Constant Registers:** The PB-1000 system maintains specific registers as constants required by ROM routines:
    *   **`$30`**: Holds the decimal value **1**.
    *   **`$31`**: Holds the decimal value **0**.
    System calls made via `CAL` **expect** these registers to hold these values. User programs *can* modify `$30` and `$31`, but doing so will cause subsequent system `CAL` calls to malfunction unless the correct values are restored beforehand.  If your user program never modify $30 or $31 then you can safetly use them for their respective constant values and you don't need to restore their values before ROM calls.
*   **Using `$30`/`$31` in User Code:** User code *can* read `$30` and `$31` to obtain the constants `1` and `0`.  Simply avoid changing `$30` or `$31`.

*   **ROM Calls (`CAL`):**
    *   If your code modifies `$30` or `$31`, you **must** save their original values (e.g., using `PHU`/`PHS`) before the modification, and restore the correct system values (`1` and `0`) before executing any `CAL` instruction.


# 4. Assembly Language Syntax and Directives (PB-1000)

*   **Statement Format:** `[LABEL:] MNEMONIC [OPERAND1 [, OPERAND2...]] [; COMMENT]`
    *   Labels must end with a colon `:` when defined.
    *   Mnemonics are instruction codes (e.g., `LD`, `ADW`, `JR`). Use uppercase.
    *   Operands follow rules described in Addressing Modes. Separate multiple operands with commas.
    *   Comments start with `;` and extend to the end of the line.
*   **Numeric Literals:**
    *   Decimal: `123`
    *   Hexadecimal: `&H7A` (`H` in caps only)
    *   Character/String: `"A"` (=&H41), `"AB"` (=&H4241, little-endian in memory). Only usable within `DB` directive.
*   **Essential Pseudo-instructions:**
    *   `LABEL: EQU value`: Assigns a numeric `value` (must be pre-calculable) to `LABEL`. `LABEL` must end with a colon.
    *   `ORG address`: Specifies the object program start address.  Multiple ocurrences may be present in a single program, but a newly specified address must be larger than the address specified by the ORG command immediately preceding.  Typically `ORG &H7000`. `address` must be a 16-bit value.
    *   `START address`: Specifies the execution entry point for the program loader. `address` must be a 16-bit value. Can only appear once.  Normally the address specified by this command is the same as that specified by the ORG command.  START is not explicitely required but it is a good practice to use it.
    *   `[LABEL:] DB byte1, byte2, "string", ...`: Define byte data. Stores 8-bit values consecutively. String literals are expanded into their ASCII byte values.  Character literals (`A`) are only supported in `DB` directives; character literals (`A`) are not valid as instruction operands.  DB string escape sequences are not supported (no escape interpretation, the assembler treats everything between quotes literally.
    *   `[LABEL:] DS size`: Define Storage. Reserves `size` bytes of memory. The PB-1000 assembler *does not* initialize this memory (contents are undefined). The compiler must explicitly store initial values if required (e.g., for zero-initialized variables).
*	 The built-in assembler maximum line size is: 255 characters.  The file size of the source assembly program to be assembled by the built-in assembler in the PB-1000 is only limited by the free available RAM in the PB-1000.
*   **Excluded HD61 Features:** The generated code *must not* use `DW`, `LEVEL`, `#IF`, `#ELSE`, `#ENDIF`, `#INCLUDE`, `#INCBIN`, `#NOLIST`, `#LIST`, `#EJECT`, `#KC`, `#AI`, or rely on advanced expression evaluation, long labels, or specific index registers (SX/SY/SZ). Undocumented jump extensions on non-jump instructions (e.g., `LD $2, $0, JR LABEL`) should also be avoided.

# 5. Instruction Set

## 5.1. Introduction and Summary

- This section contains a brief summary of the instruction set with description and notes.
- The table in the section `Full Instruction Set Reference Guide` must be prioritized when selecting valid PB-1000 assembler instructions.  Do not assume as valid, common formats from other CPU instruction sets.

### Brief-notation-style conventions in this section

* `$reg`: Refers only to one of the 32 main registers ($0-$31).
* `$reg_pair`: Refers to an adjacent pair ($n, $n+1) used for 16-bit operations.  Only main registers.
* `IM8`/`IM16`: Represent 8-bit/16-bit immediate literal values.
* `dst`/`src`: Standard destination/source operand placeholders.
* `(IX/IZ+/- offset)`: Indexed addressing using IX or IZ plus an 8-bit $reg or IM8 as offset.
* `(IX/IZ+/- $reg)`: Indexed addressing using IX or IZ plus an 8-bit $reg.

### Data Transfer (8-bit):

|Instruction|Summary|
|---|---|
|`LD dst, src`|Load 8-bit value. `src` can be `$reg`, `IM8`, `($reg_pair)`, `(IX/IZ+/-offset)`. `dst` is `$reg`.|
|`ST src, dst`|Store 8-bit value. `src` is `$reg`. `dst` can be `($reg_pair)`, `(IX/IZ+/-offset)`.|
|`LDI dst, (IX/IZ+/-offset)`|Load 8-bit and increment pointer (`IX/IZ` = `IX/IZ` +/- offset + 1).|
|`STI src, (IX/IZ+/-offset)`|Store 8-bit and increment pointer (`IX/IZ` = `IX/IZ` +/- offset + 1).|
|`PHU $reg`|Push 8-bit to User Stack (`US`).|
|`PPU $reg`|Push/Pop 8-bit from User Stack (`US`).|
| `PHS $reg`  | Push 8-bit to System Stack (`SS`).  |
| `PPS $reg`  | Pop 8-bit from System Stack (`SS`). |
|`GFL $reg`|Transfers the flag resiter contents to the main register specified by the first operand.|
|`PFL $reg`|Transfers the contents of the main register specified by the first operand to the flag register (high-order 4 bits only).  Affects Z,C,LZ and UZ.|


### Data Transfer (16-bit):

|Instruction|Summary|
|---|---|
|`LDW dst_pair, src`|Load 16-bit value. `src` can be `$reg_pair`, `IM16`, `($reg_pair)`, `(IX/IZ+/-offset)`. `dst_pair` is `$reg`.|
|`STW src_pair, dst`|Store 16-bit value. `src_pair` is `$reg`. `dst` can be `($reg_pair)`, `(IX/IZ+/-offset)`.|
|`LDIW dst_pair, (IX/IZ+/- $reg)`|Load 16-bit and post-increment pointer by 2 + $reg (`IX/IZ` = `IX/IZ` +/- $reg + 2).|
|`STIW src_pair, (IX/IZ+/- $reg)`|Store 16-bit and post-increment pointer by 2 + $reg (`IX/IZ` = `IX/IZ` +/- $reg + 2).|
|`PRE {IX\IY\IZ\US\SS}, src`|Put Register. `src` can be `$reg_pair` or `IM16`. Loads the specified 16-bit register.  Only GRE and PRE are allowed to manipulate IX, IZ, IY, SS, US and KY.|
|`GRE {IX\IY\IZ\US\SS\KY}, dst_pair`|Get Register. Stores the specified 16-bit register into `$dst_pair`.  Only GRE and PRE are allowed to manipulate IX, IZ, IY, SS, US and KY.|
|`PHUW $reg_pair`|Pushes 16-bit data from registers, `$reg` first then `$reg-1` onto the **User Stack** (little-endian). `US` decrements by 2.|
|`PPUW $reg_pair`| Pops 16-bit data into registers `$reg` first then `$reg+1` (high byte) from the **User Stack**. `US` increments by 2.|
| `PHSW $reg_pair` | Pushes 16-bit data onto System Stack (`SSP`) (high-byte then low-byte).  Identical logic as PHUW. |
| `PPSW $reg_pair` | Pops 16-bit data from System Stack (`SSP`) into register-pair (low-byte then high-byte). Identical logic as PPUW.|


- **Crucial detail**: PHUW/PPUW and PHSW/PPSW operate on register-pairs in little-endian format. Pushing uses the higher register (e.g., $5), while popping uses the lower register (e.g., $4) to reconstruct the original word.  Critical clarification example: If your intent is to PUSH the word in the register pair $4 and $5 then do `PHUW $5` and to correctly restore you should do `PPUW $4`.  Nothing prevents from popping to another register pair, except the logic of your solution, so if you do $PPUW $1, then $4/5 will be restored to $1/2.

### Arithmetic (8-bit):

|Instruction|Summary|
|---|---|
|`AD dst, src`|Add (`dst = dst + src`). `src` can be `$reg` or `IM8`.  `dst` can be `$reg` or (IX/IZ+/-offset)`. Affects Z, C flags.|
|`SB dst, src`|Subtract (`dst = dst - src`). `src` can be `$reg` or `IM8`.  `dst` can be `$reg` or (IX/IZ+/-offset)`. Affects Z, C flags.|
|`ADC dst, src`|Add Check (like `AD` but only affects flags).|
|`SBC dst, src`|Subtract Check (like `SB` but only affects flags). Use `SBC` for comparisons.|
|`CMP $reg`|Two's complement (`$reg = -$reg`). Affects Z, C.|
|`INV $reg`|One's complement (`$reg = ~$reg`). Affects Z, C=1.|
|`ADB dst, src`|Performs Binary Coded Decimal Addition (BCD). Add-BCD (`dst = dst + src`). `src` can be `$reg` or `IM8`. `dst` is `$reg`. Affects Z, C flags.|
|`SBB dst, src`|Performs Binary Coded Decimal Substraction (BCD). Subtract-BCD (`dst = dst - src`). `src` can be `$reg` or `IM8`. `dst` is `$reg`. Affects Z, C flags.|

- `Binary Coded Decimal Instructions (BCD)`: The high-order and low-order 4 bits are treated as binary coded decimal.


### Arithmetic (16-bit):

|Instruction|Summary|
|---|---|
|`ADW dst_pair, src_pair`|Add 16-bit (`dst = dst + src`). `src_pair` is `$reg_pair`. `dst_pair` can be `$reg_pair` or `(IX/IZ+/-$reg)`. Affects Z, C flags.|
|`SBW dst_pair, src_pair`|Subtract 16-bit (`dst = dst - src`). `src_pair` is `$reg_pair`. `dst_pair` can be `$reg_pair` or `(IX/IZ+/-$reg)`. Affects Z, C flags.|
|`ADCW dst_pair, src_pair`|Add Check 16-bit (flags only).|
|`SBCW dst_pair, src_pair`|Subtract Check 16-bit (flags only). Use `SBCW` for 16-bit comparisons.|
|`CMPW $reg_pair`|Two's complement 16-bit. Affects Z, C.|
|`INVW $reg_pair`|One's complement 16-bit. Affects Z, C=1.|
|`ADBW dst_pair, src_pair`|Add-BCD 16-bit (`dst = dst + src`). `src_pair` is `$reg`. `dst` is `$reg`. Affects Z, C flags.|
|`SBBW dst_pair, src_pair`|Substract-BCD 16-bit (`dst = dst + src`). `src_pair` is `$reg`. `dst` is `$reg`. Affects Z, C flags.|



### Logical (8-bit):

|Instruction|Summary|
|---|---|
|`AN dst, src`|Bitwise AND (`dst = dst & src`). `src` can be `$reg` or `IM8`. `dst` is `$reg`. Affects Z, C=0.|
|`OR dst, src`|Bitwise OR (`dst = dst | src`). `src` can be `$reg` or `IM8`. `dst` is `$reg`. Affects Z, C=1.|
|`XR dst, src`|Bitwise XOR (`dst = dst ^ src`). `src` can be `$reg` or `IM8`. `dst` is `$reg`. Affects Z, C=0.|
|`ANC dst, src`|Bitwise AND-Check version (flags only).|
|`ORC dst, src`|Bitwise OR-Check version (flags only).|
|`XRC dst, src`|Bitwise XOR-Check version (flags only).|
|`NA dst, src`|Calculates the NAND (inverted AND between dst and src. `src` can be `$reg` or `IM8`. `dst` is `$reg`. Affects Z, C=1.|
|`NAC dst, src`|Check version of NA (flags only).|


### Logical (16-bit):

|Instruction|Summary|
|---|---|
|`ANW dst_pair, src_pair`|Bitwise AND 16-bit. Affects Z, C=0.|
|`ORW dst_pair, src_pair`|Bitwise OR 16-bit. Affects Z, C=1.|
|`ANCW dst_pair, src_pair`|Bitwise AND-word-check version (flags only). |
|`ORCW dst_pair, src_pair`|Bitwise OR-word-check version (flags only).  |
|`XRCW dst_pair, src_pair`|Bitwise XOR-word-check version (flags only). |
|`NAW dst_pair, src_pair`|Calculates the 16-bit NAND (inverted AND between dst and src). Affects Z, C=1.|
|`NACW dst_pair, src_pair`|Check version of NAW (flags only).|


### Shift/Rotate Instructions** (operand is a general-purpose register):

|Instruction|Summary|
|---|---|
|`ROU reg`| Rotate Left 8-bit (`Carry←MSB←…←LSB←Carry`).|
|`ROUW reg_pair`| Rotate Left 16-bit (`Carry←MSB←…←LSB←Carry`).|
|`ROD reg`| Rotate Right 8-bit (`Carry→MSB→…→LSB→Carry`).|
|`RODW reg_pair`| Rotate Right 16-bit (`Carry→MSB→…→LSB→Carry`).|
|`BIU reg`| Shift Left 8-bit (`0→LSB`, `Carry←MSB`).|
|`BIUW reg_pair`| Shift Left 16-bit (`0→LSB`, `Carry←MSB`).|
|`BID reg`| Shift Right 8-bit (`0→MSB`, `Carry←LSB`).|
|`BIDW reg_pair`| Shift Right 16-bit (`0→MSB`, `Carry←LSB`).|
|`DIU reg`| Shift Left 4 bits 8-bit (`0→lower nibble`).|
|`DIUW reg_pair`| Shift Left 4 bits 16-bit (`0→lower nibble`).|
|`DID reg`| Shift Right 4 bits 8-bit (`0→upper nibble`).|
|`DIDW reg_pair`| Shift Right 4 bits 16-bit (`0→upper nibble`).|
|`BYUW`|Shift left 8 bits (lower byte ← 0).|
|`BYDW`|Shift right 8 bits (upper byte ← 0).|


- **Flags**: All Shift/Rotate instructions update `Z` (zero result), `C` (lost bit), `LZ` (low-nibble zero), `UZ` (high-nibble zero).  
- **16-bit Shift/Rotate**: Operand specifies register pair (e.g., `$2` = `$2`+`$3`). 
- `BYUW`/`BYDW`: Only have 16-bit.

### Control Flow:

|Instruction|Summary|
|---|---|
|`JP address`|Absolute Jump (16-bit immediate `address`).|
|`JP condition, address`|Conditional Absolute Jump. Conditions: `Z`, `NZ`, `C`, `NC`, `LZ`, `UZ`.|
|`JR offset`|Relative Jump (label or +/- immediate offset, -127 to +127). Generally preferred for position-independent code or short jumps.|
|`JR condition, offset`|Conditional Relative Jump. Same conditions as `JP`.|
|`CAL address`|Call Subroutine (pushes PC+instruction_length onto System Stack `SS`, then jumps to `address`).|
|`CAL condition, address`|Conditional Call. Same conditions as `JP`.|
|`RTN`|Return from Subroutine (pops address from `SS`, adds 1, loads into PC).  Can and should be used as the last instruction to end program execution.|
|`RTN condition`|Conditional Return. Same conditions as `JP`.|

### Block Operations:

|Instruction|Summary|
|---|---|
|`BUP`|Block copy upwards (IX=src_start, IY=src_end, IZ=dst_start).|
|`BDN`|Block copy downwards (IX=src_start, IY=src_end, IZ=dst_start).|
|`SUP { $reg or IM8 }`|Search upwards (IX=start, IY=end) for value. Sets Z to 1 if not found.|
|`SDN { $reg or IM8 }`|Search downwards (IX=start, IY=end-1) for value. Sets Z if not found.|

### Special Instructions:

|Instruction|Summary|
|---|---|
|NOP|No operation (could be useful for timing/padding).|
|CLT|Clear Timer.  Inputs a SET signal to all timer counters to set the value of the counters to 0.|
|FST|Uses the system clock without dividing (high-speed processing mode).  The system usually operates in the high-speed mode.|
|SLW|Uses the system clock with 1/16 dividing (low power mode).  Slow mode can cause inestabilities sin some ROM calls like LCD access.  A program could run in slow mode as long as it doesn't display anything in the screen.  (Compiler likely won't use SLW.).|
|OFF|Power off. (Compiler won't use).|
|TRP|Software trap. (Compiler won't use).|
|`CANI`| Cancel interrupt. (Compiler won't use directly).|
|`RTNI`| Return from interrupt. (Compiler won't use directly).|


---


## 5.2. Full Instruction Set Reference Guide

Expanded Instruction set nmemonic table for the Hitachi HD61700 CPU.

This section contains the definitive technical specification for all the *valid* instruction formats, addressing modes, bytes occupied per instruction format, machine cycles for fetch/execution, flags modified and other important details for the HD61700 CPU and built-in assembler.   It must be considered the absolute ground truth for PB-1000 valid instruction formats, etc.


** Notation Conventions Used in this Section**

*   `n`: 8-bit immediate data
*   `m`: 16-bit immediate data
*   `p`: 7-bit immediate data
*   `a`: Number of bytes processed (for BDN/BUP/SDN/SUP).
*   `$`: Register (e.g., `$2`, `$4`). Register pairs denoted by lower register number (e.g. `$2` implies `$2/$3` for 16-bit instructions).
*   `($)`: Indirect addressing using register pair (e.g., `($0)` uses `$0/$1`).
*   `(IX+/-$)`, `(IZ+/-$)`: Indexed addressing with positive or negative register offset (e.g., `(IX+$2)`).
*   `(IX+/-n)`, `(IZ+/-n)`: Indexed addressing with 8-bit immediate offset (e.g., `(IZ+10)`, `(IX-&H5)`).
*   `+/-P`: Relative addressing offset (7-bit label or immediate, e.g., `LABEL`, `+20`, `-10`).
*   `M`: Flag is Modified according to the result of the operation.
*   `(blank)`: Flag is not affected by the operation.
*   `0`/`1`: Flag is set to 0 or 1 respectively.

| Mnemonic | Valid Format for Command | Op Code (Binary) | Hex | Bytes Occupied| Machine Cycles Fetch | Machine Cycles Exec | State Fetch | State Exec | Flag Z | Flag C | Flag LZ | Flag UZ | Example | Example AI Inferred? |
| :------- | :---------------- | :--------------- | :-- | :---- | :------- | :------ | :---------- | :--------- | :----- | :----- | :------ | :------ | :---------------- | :---------------- |
| LD       | LD $,$            | `00000010`       | 02  | 3     | 2        | 2       | 6           | 6          |        |        |         |         | `LD $2, $0`       | No                |
| LD       | LD $,($)          | `00010001`       | 11  | 3     | 2        | 4       | 6           | 11         |        |        |         |         | `LD $2, ($0)`     | No                |
| LD       | LD $, (IX+/-$)      | `00101000`       | 28  | 3     | 2        | 4       | 6           | 11         |        |        |         |         | `LD $2, (IX+$0)`  | No                |
| LD       | LD $, (IZ+/-$)      | `00101001`       | 29  | 3     | 2        | 4       | 6           | 11         |        |        |         |         | `LD $2, (IZ+$0)`  | No                |
| LD       | LD $,n            | `01000010`       | 42  | 3     | 2        | 2       | 6           | 6          |        |        |         |         | `LD $2, 10`       | No                |
| LD       | LD $, (IX+/-n)      | `01101000`       | 68  | 3     | 2        | 4       | 6           | 11         |        |        |         |         | `LD $2, (IX-250)`  | No                |
| LD       | LD $, (IZ+/-n)      | `01101001`       | 69  | 3     | 2        | 4       | 6           | 11         |        |        |         |         | `LD $2, (IZ+254)`  | No                |
| LDI      | LDI $, (IX+/- $)    | `00101010`       | 2A  | 3     | 2        | 4       | 6           | 11         |        |        |         |         | `LDI $4, (IX+$2)` | No                |
| LDI      | LDI $, (IZ+/- $)    | `00101011`       | 2B  | 3     | 2        | 4       | 6           | 11         |        |        |         |         | `LDI $4, (IZ+$2)` | No                |
| LDI      | LDI $, (IX+/-n)     | `01101010`       | 6A  | 3     | 2        | 4       | 6           | 11         |        |        |         |         | `LDI $4, (IX+10)` | No                |
| LDI      | LDI $, (IZ+/-n)     | `01101011`       | 6B  | 3     | 2        | 4       | 6           | 11         |        |        |         |         | `LDI $4, (IZ+10)` | No                |
| ST       | ST $,($)          | `00010000`       | 10  | 3     | 2        | 4       | 6           | 11         |        |        |         |         | `ST $2, ($0)`     | No                |
| ST       | ST $, (IX+/-$)      | `00100000`       | 20  | 3     | 2        | 4       | 6           | 11         |        |        |         |         | `ST $2, (IX+$0)`  | No                |
| ST       | ST $, (IZ+/-$)      | `00100001`       | 21  | 3     | 2        | 4       | 6           | 11         |        |        |         |         | `ST $2, (IZ+$1)`  | No                |
| ST       | ST $, (IX+/-n)      | `01100000`       | 60  | 3     | 2        | 4       | 6           | 11         |        |        |         |         | `ST $2, (IX+&H04)`  | No                |
| ST       | ST $, (IZ+/-n)      | `01100001`       | 61  | 3     | 2        | 4       | 6           | 11         |        |        |         |         | `ST $2, (IZ+10)`  | No                |
| STI      | STI $, (IX+/-$)     | `00100010`       | 22  | 3     | 2        | 4       | 6           | 11         |        |        |         |         | `STI $4, (IX+$2)` | No                |
| STI      | STI $, (IZ+/-$)     | `00100011`       | 23  | 3     | 2        | 4       | 6           | 11         |        |        |         |         | `STI $4, (IZ+$2)` | No                |
| STI      | STI $, (IX+/-n)     | `01100010`       | 62  | 3     | 2        | 4       | 6           | 11         |        |        |         |         | `STI $4, (IX+10)` | No                |
| STI      | STI $, (IZ+/-n)     | `01100011`       | 63  | 3     | 2        | 4       | 6           | 11         |        |        |         |         | `STI $4, (IZ-&H20)` | No                |
| PHS      | PHS $             | `00100110`       | 26  | 2     | 1        | 3       | 3           | 9          |        |        |         |         | `PHS $10`          | No                |
| PHU      | PHU $             | `00100111`       | 27  | 2     | 1        | 3       | 3           | 9          |        |        |         |         | `PHU $12`          | No                |
| PPS      | PPS $             | `00101110`       | 2E  | 2     | 1        | 4       | 3           | 11         |        |        |         |         | `PPS $13`          | No                |
| PPU      | PPU $             | `00101111`       | 2F  | 2     | 1        | 4       | 3           | 11         |        |        |         |         | `PPU $14`          | No                |
| GFL      | GFL $             | `00011100`       | 1C  | 2     | 1        | 2       | 3           | 6          |        |        |         |         | `GFL $15`          | No                |
| GPO      | GPO $             | `00011100`       | 1C  | 2     | 1        | 2       | 3           | 6          |        |        |         |         | `GPO $16`          | No                |
| GST      | GST PE, $         | `00011110`       | 1E  | 2     | 1        | 2       | 3           | 6          |        |        |         |         | `GST PE, $0`      | Yes               |
| GST      | GST PD, $         | `00011110`       | 1E  | 2     | 1        | 2       | 3           | 6          |        |        |         |         | `GST PD, $1`      | Yes               |
| GST      | GST UA, $         | `00011110`       | 1E  | 2     | 1        | 2       | 3           | 6          |        |        |         |         | `GST UA, $2`      | Yes               |
| GST      | GST IA, $         | `00011111`       | 1F  | 2     | 1        | 2       | 3           | 6          |        |        |         |         | `GST IA, $3`      | Yes               |
| GST      | GST IE, $         | `00011111`       | 1F  | 2     | 1        | 2       | 3           | 6          |        |        |         |         | `GST IE, $4`      | Yes               |
| GST      | GST TM, $         | `00011111`       | 1F  | 2     | 1        | 2       | 3           | 6          |        |        |         |         | `GST TM, $5`      | Yes               |
| PFL      | PFL $             | `00010100`       | 14  | 2     | 1        | 2       | 3           | 6          | M      | M      | M       | M       | `PFL $2`          | No                |
| PST      | PST PE, $         | `00010110`       | 16  | 2     | 1        | 2       | 3           | 6          |        |        |         |         | `PST PE, $0`      | Yes               |
| PST      | PST PD, $         | `00010110`       | 16  | 2     | 1        | 2       | 3           | 6          |        |        |         |         | `PST PD, $1`      | Yes               |
| PST      | PST UA, $         | `00010110`       | 16  | 2     | 1        | 2       | 3           | 6          |        |        |         |         | `PST UA, $2`      | Yes               |
| PST      | PST IA, $         | `00010111`       | 17  | 2     | 1        | 2       | 3           | 6          |        |        |         |         | `PST IA, $3`      | Yes               |
| PST      | PST IE, $         | `00010111`       | 17  | 2     | 1        | 2       | 3           | 6          |        |        |         |         | `PST IE, $4`      | Yes               |
| PST      | PST PE,n          | `01010110`       | 56  | 3     | 2        | 2       | 6           | 6          |        |        |         |         | `PST PE, 1`       | Yes               |
| PST      | PST PD,n          | `01010110`       | 56  | 3     | 2        | 2       | 6           | 6          |        |        |         |         | `PST PD, 2`       | Yes               |
| PST      | PST UA,n          | `01010110`       | 56  | 3     | 2        | 2       | 6           | 6          |        |        |         |         | `PST UA, 0`       | Yes               |
| PST      | PST IA,n          | `01010111`       | 57  | 3     | 2        | 2       | 6           | 6          |        |        |         |         | `PST IA, &H0`       | Yes               |
| PST      | PST IE,n          | `01010111`       | 57  | 3     | 2        | 2       | 6           | 6          |        |        |         |         | `PST IE, &H01`       | Yes               |
| LDW      | LDW $,$           | `10000010`       | 82  | 3     | 2        | 4       | 6           | 11         |        |        |         |         | `LDW $2, $0`      | No                |
| LDW      | LDW $,($)         | `10010001`       | 91  | 3     | 2        | 5       | 6           | 14         |        |        |         |         | `LDW $2, ($0)`    | No                |
| LDW      | LDW $,(IX+/-$)      | `10101000`       | A8  | 3     | 2        | 5       | 6           | 14         |        |        |         |         | `LDW $2, (IX+$0)` | No                |
| LDW      | LDW $,(IZ+/-$)      | `10101001`       | A9  | 3     | 2        | 5       | 6           | 14         |        |        |         |         | `LDW $2, (IZ+$0)` | No                |
| LDW      | LDW $,m           | `11010001`       | D1  | 4     | 3        | 5       | 9           | 14         |        |        |         |         | `LDW $4, &H1234`  | No                |
| LDIW     | LDIW $,(IX+/- $)    | `10101010`       | AA  | 3     | 2        | 5       | 6           | 14         |        |        |         |         | `LDIW $2, (IX+$0)`| No                |
| LDIW     | LDIW $,(IZ+/- $)    | `10101011`       | AB  | 3     | 2        | 5       | 6           | 14         |        |        |         |         | `LDIW $4, (IZ+$2)`| No                |
| STW      | STW $,(IX+/-$)      | `10100000`       | A0  | 3     | 2        | 5       | 6           | 14         |        |        |         |         | `STW $4, (IX+$2)` | No                |
| STW      | STW $,(IZ+/-$)      | `10100001`       | A1  | 3     | 2        | 5       | 6           | 14         |        |        |         |         | `STW $4, (IZ+$2)` | No                |
| STW      | STW $,($)         | `10010000`       | 90  | 3     | 2        | 5       | 6           | 14         |        |        |         |         | `STW $2, ($0)`    | No                |
| STIW     | STIW $,(IX+/- $)    | `10100010`       | A2  | 3     | 2        | 4       | 6           | 14         |        |        |         |         | `STIW $4, (IX+$2)`| No                |
| STIW     | STIW $,(IZ+/- $)    | `10100011`       | A3  | 3     | 2        | 4       | 6           | 14         |        |        |         |         | `STIW $4, (IZ+$2)`| No                |
| PHSW     | PHSW $            | `10100110`       | A6  | 2     | 1        | 4       | 3           | 12         |        |        |         |         | `PHSW $2`         | No                |
| PHUW     | PHUW $            | `10100111`       | A7  | 2     | 1        | 4       | 3           | 12         |        |        |         |         | `PHUW $3`         | No                |
| PPSW     | PPSW $            | `10101110`       | AE  | 2     | 1        | 5       | 3           | 14         |        |        |         |         | `PPSW $5`         | No                |
| PPUW     | PPUW $            | `10101111`       | AF  | 2     | 1        | 5       | 3           | 14         |        |        |         |         | `PPUW $6`         | No                |
| GRE      | GRE IX, $         | `10011110`       | 9E  | 2     | 1        | 4       | 3           | 11         |        |        |         |         | `GRE IX, $0`      | Yes               |
| GRE      | GRE IY, $         | `10011110`       | 9E  | 2     | 1        | 4       | 3           | 11         |        |        |         |         | `GRE IY, $2`      | Yes               |
| GRE      | GRE IZ, $         | `10011110`       | 9E  | 2     | 1        | 4       | 3           | 11         |        |        |         |         | `GRE IZ, $3`      | Yes               |
| GRE      | GRE US, $         | `10011110`       | 9E  | 2     | 1        | 4       | 3           | 11         |        |        |         |         | `GRE US, $4`      | Yes               |
| GRE      | GRE SS, $         | `10011111`       | 9F  | 2     | 1        | 4       | 3           | 11         |        |        |         |         | `GRE SS, $5`      | Yes               |
| GRE      | GRE KY, $         | `10011111`       | 9F  | 2     | 1        | 4       | 3           | 11         |        |        |         |         | `GRE KY, $6`      | Yes               |
| PRE      | PRE IX, $         | `10010110`       | 96  | 2     | 1        | 4       | 3           | 11         |        |        |         |         | `PRE IX, $7`      | Yes               |
| PRE      | PRE IY, $         | `10010110`       | 96  | 2     | 1        | 4       | 3           | 11         |        |        |         |         | `PRE IY, $8`      | Yes               |
| PRE      | PRE IZ, $         | `10010110`       | 96  | 2     | 1        | 4       | 3           | 11         |        |        |         |         | `PRE IZ, $9`      | Yes               |
| PRE      | PRE US, $         | `10010110`       | 96  | 2     | 1        | 4       | 3           | 11         |        |        |         |         | `PRE US, $30`      | Yes               |
| PRE      | PRE SS, $         | `10010111`       | 97  | 2     | 1        | 4       | 3           | 11         |        |        |         |         | `PRE SS, $31`      | Yes               |
| PRE      | PRE IX,m          | `11010110`       | D6  | 4     | 3        | 4       | 9           | 11         |        |        |         |         | `PRE IX, &H70FF`       | Yes               |
| PRE      | PRE IY,m          | `11010110`       | D6  | 4     | 3        | 4       | 9           | 11         |        |        |         |         | `PRE IY, &H001F`       | Yes               |
| PRE      | PRE IZ,m          | `11010110`       | D6  | 4     | 3        | 4       | 9           | 11         |        |        |         |         | `PRE IZ, &H104F`       | Yes               |
| PRE      | PRE US,m          | `11010110`       | D6  | 4     | 3        | 4       | 9           | 11         |        |        |         |         | `PRE US, &H70AF`       | Yes               |
| PRE      | PRE SS,m          | `11010111`       | D7  | 4     | 3        | 4       | 9           | 11         |        |        |         |         | `PRE SS, &H2011`       | Yes               |
| AD       | AD $,$            | `00001000`       | 08  | 3     | 2        | 2       | 6           | 6          | M      | M      | M       | M       | `AD $4, $2`       | No                |
| AD       | AD (IX+/-$),$       | `00111100`       | 3C  | 3     | 2        | 4       | 6           | 12         | M      | M      | M       | M       | `AD (IX+$4), $2`  | No                |
| AD       | AD (IZ+/-$), $      | `00111101`       | 3D  | 3     | 2        | 4       | 6           | 12         | M      | M      | M       | M       | `AD (IZ+$4), $2`  | No                |
| AD       | AD $,n            | `01001000`       | 48  | 3     | 2        | 2       | 6           | 6          | M      | M      | M       | M       | `AD $4, 123`      | No                |
| AD       | AD (IX+/-n), $      | `01111100`       | 7C  | 3     | 2        | 4       | 6           | 12         | M      | M      | M       | M       | `AD (IX+10), $2`  | No                |
| AD       | AD (IZ+/-n), $      | `01111101`       | 7D  | 3     | 2        | 4       | 6           | 12         | M      | M      | M       | M       | `AD (IZ+10), $2`  | No                |
| ADB      | ADB $,$           | `00001010`       | 0A  | 3     | 2        | 2       | 6           | 6          | M      | M      | M       | M       | `ADB $4, $2`      | No                |
| ADB      | ADB $,n           | `01001010`       | 4A  | 3     | 2        | 2       | 6           | 6          | M      | M      | M       | M       | `ADB $4, 123`     | No                |
| ADC      | ADC $,$           | `00000000`       | 00  | 3     | 2        | 2       | 6           | 6          | M      | M      | M       | M       | `ADC $4, $2`      | No                |
| ADC      | ADC (IX+/-$),$      | `00111000`       | 38  | 3     | 2        | 4       | 6           | 12         | M      | M      | M       | M       | `ADC (IX+$4), $2` | No                |
| ADC      | ADC (IZ+/- $),$     | `00111001`       | 39  | 3     | 2        | 4       | 6           | 12         | M      | M      | M       | M       | `ADC (IZ+$4), $2` | No                |
| ADC      | ADC $,n           | `01000000`       | 40  | 3     | 2        | 2       | 6           | 6          | M      | M      | M       | M       | `ADC $4, 123`     | No                |
| ADC      | ADC (IX+/-n), $     | `01111000`       | 78  | 3     | 2        | 4       | 6           | 12         | M      | M      | M       | M       | `ADC (IX+10), $2` | No                |
| ADC      | ADC (IZ+/-n), $     | `01111001`       | 79  | 3     | 2        | 4       | 6           | 12         | M      | M      | M       | M       | `ADC (IZ+10), $2` | No                |
| AN       | AN $,$            | `00001100`       | 0C  | 3     | 2        | 2       | 6           | 6          | M      | 0      | M       | M       | `AN $4, $2`       | No                |
| AN       | AN $,n            | `01001100`       | 4C  | 3     | 2        | 2       | 6           | 6          | M      | 0      | M       | M       | `AN $4, 123`      | No                |
| ANC      | ANC $,$           | `00000100`       | 04  | 3     | 2        | 2       | 6           | 6          | M      | 0      | M       | M       | `ANC $4, $2`      | No                |
| ANC      | ANC $,n           | `01000100`       | 44  | 3     | 2        | 2       | 6           | 6          | M      | 0      | M       | M       | `ANC $4, 123`     | No                |
| NA       | NA $,$            | `00001101`       | 0D  | 3     | 2        | 2       | 6           | 6          | M      | 1      | M       | M       | `NA $2, $3`       | Yes               |
| NA       | NA $,n            | `01001101`       | 4D  | 3     | 2        | 2       | 6           | 6          | M      | 1      | M       | M       | `NA $4, &H55`     | Yes               |
| NAC      | NAC $,$           | `00000101`       | 05  | 3     | 2        | 2       | 6           | 6          | M      | 1      | M       | M       | `NAC $2, $3`      | Yes               |
| NAC      | NAC $,n           | `01000101`       | 45  | 3     | 2        | 2       | 6           | 6          | M      | 1      | M       | M       | `NAC $4, &H0F`    | Yes               |
| OR       | OR $,$            | `00001110`       | 0E  | 3     | 2        | 2       | 6           | 6          | M      | 1      | M       | M       | `OR $4, $2`       | No                |
| OR       | OR $,n            | `01001110`       | 4E  | 3     | 2        | 2       | 6           | 6          | M      | 1      | M       | M       | `OR $4, 123`      | No                |
| ORC      | ORC $,$           | `00000110`       | 06  | 3     | 2        | 2       | 6           | 6          | M      | 1      | M       | M       | `ORC $4, $2`      | No                |
| ORC      | ORC $,n           | `01000110`       | 46  | 3     | 2        | 2       | 6           | 6          | M      | 1      | M       | M       | `ORC $4, 123`     | No                |
| SB       | SB $,$            | `00001001`       | 09  | 3     | 2        | 2       | 6           | 6          | M      | M      | M       | M       | `SB $4, $2`       | No                |
| SB       | SB (IX+/-$), $      | `00111110`       | 3E  | 3     | 2        | 4       | 6           | 12         | M      | M      | M       | M       | `SB (IX+$4), $2`  | No                |
| SB       | SB (IZ+/-$), $      | `00111111`       | 3F  | 3     | 2        | 4       | 6           | 12         | M      | M      | M       | M       | `SB (IZ+$4), $2`  | No                |
| SB       | SB $,n            | `01001001`       | 49  | 3     | 2        | 2       | 6           | 6          | M      | M      | M       | M       | `SB $4, 123`      | No                |
| SB       | SB (IX+/-n), $      | `01111110`       | 7E  | 3     | 2        | 4       | 6           | 12         | M      | M      | M       | M       | `SB (IX+10), $2`  | No                |
| SB       | SB (IZ+/-n), $      | `01111111`       | 7F  | 3     | 2        | 4       | 6           | 12         | M      | M      | M       | M       | `SB (IZ+10), $2`  | No                |
| SBB      | SBB $,$           | `00001011`       | 0B  | 3     | 2        | 2       | 6           | 6          | M      | M      | M       | M       | `SBB $4, $2`      | No                |
| SBB      | SBB $,n           | `01001011`       | 4B  | 3     | 2        | 2       | 6           | 6          | M      | M      | M       | M       | `SBB $4, 123`     | No                |
| SBC      | SBC $,$           | `00000001`       | 01  | 3     | 2        | 2       | 6           | 6          | M      | M      | M       | M       | `SBC $4, $2`      | No                |
| SBC      | SBC (IX+/-$),$      | `00111010`       | 3A  | 3     | 2        | 4       | 6           | 12         | M      | M      | M       | M       | `SBC (IX+$4), $2` | No                |
| SBC      | SBC (IZ+/-$),$      | `00111011`       | 3B  | 3     | 2        | 4       | 6           | 12         | M      | M      | M       | M       | `SBC (IZ+$4), $2` | No                |
| SBC      | SBC $,n           | `01000001`       | 41  | 3     | 2        | 2       | 6           | 6          | M      | M      | M       | M       | `SBC $4, 123`     | No                |
| SBC      | SBC (IX+/-n), $     | `01111010`       | 7A  | 3     | 2        | 4       | 6           | 12         | M      | M      | M       | M       | `SBC (IX+10), $2` | No                |
| SBC      | SBC (IZ+/-n), $     | `01111011`       | 7B  | 3     | 2        | 4       | 6           | 12         | M      | M      | M       | M       | `SBC (IZ+10), $2` | No                |
| XR       | XR $,$            | `00001111`       | 0F  | 3     | 2        | 2       | 6           | 6          | M      | 0      | M       | M       | `XR $4, $2`       | No                |
| XR       | XR $,n            | `01001111`       | 4F  | 3     | 2        | 2       | 6           | 6          | M      | 0      | M       | M       | `XR $4, 123`      | No                |
| XRC      | XRC $,$           | `00000111`       | 07  | 3     | 2        | 2       | 6           | 6          | M      | 0      | M       | M       | `XRC $4, $2`      | No                |
| XRC      | XRC $,n           | `01000111`       | 47  | 3     | 2        | 2       | 6           | 6          | M      | 0      | M       | M       | `XRC $4, 123`     | No                |
| ADW      | ADW $,$           | `10001000`       | 88  | 3     | 2        | 4       | 6           | 11         | M      | M      | M       | M       | `ADW $4, $2`      | No                |
| ADW      | ADW (IX+/-$),$      | `10111100`       | BC  | 3     | 2        | 6       | 6           | 18         | M      | M      | M       | M       | `ADW (IX+$4), $2` | No                |
| ADW      | ADW (IZ+/-$),$      | `10111101`       | BD  | 3     | 2        | 6       | 6           | 18         | M      | M      | M       | M       | `ADW (IZ+$4), $2` | No                |
| ADBW     | ADBW $,$          | `10001010`       | 8A  | 3     | 2        | 4       | 6           | 11         | M      | M      | M       | M       | `ADBW $4, $2`     | No                |
| ADCW     | ADCW $,$          | `10000000`       | 80  | 3     | 2        | 4       | 6           | 11         | M      | M      | M       | M       | `ADCW $4, $2`     | No                |
| ADCW     | ADCW (IX+/-$),$     | `10111000`       | B8  | 3     | 2        | 4       | 6           | 11         | M      | M      | M       | M       | `ADCW (IX+$4), $2`| No                |
| ADCW     | ADCW (IZ+/-$),$     | `10111001`       | B9  | 3     | 2        | 4       | 6           | 11         | M      | M      | M       | M       | `ADCW (IZ+$4), $2`| No                |
| ANW      | ANW $,$           | `10001100`       | 8C  | 3     | 2        | 4       | 6           | 11         | M      | 0      | M       | M       | `ANW $4, $2`      | No                |
| ANCW     | ANCW $,$          | `10000100`       | 84  | 3     | 2        | 4       | 6           | 11         | M      | 0      | M       | M       | `ANCW $4, $2`     | No                |
| NAW      | NAW $,$           | `10001101`       | 8D  | 3     | 2        | 4       | 6           | 11         | M      | 1      | M       | M       | `NAW $2, $4`      | Yes               |
| NACW     | NACW $,$          | `10000101`       | 85  | 3     | 2        | 4       | 6           | 11         | M      | 1      | M       | M       | `NACW $2, $4`     | Yes               |
| ORW      | ORW $,$           | `10001110`       | 8E  | 3     | 2        | 4       | 6           | 11         | M      | 1      | M       | M       | `ORW $4, $2`      | No                |
| ORCW     | ORCW $,$          | `10000110`       | 86  | 3     | 2        | 4       | 6           | 11         | M      | 1      | M       | M       | `ORCW $4, $2`     | No                |
| SBW      | SBW $,$           | `10001001`       | 89  | 3     | 2        | 4       | 6           | 11         | M      | M      | M       | M       | `SBW $4, $2`      | No                |
| SBW      | SBW (IX+/-$), $     | `10111110`       | BE  | 3     | 2        | 6       | 6           | 18         | M      | M      | M       | M       | `SBW (IX+$4), $2` | No                |
| SBW      | SBW (IZ+/-$),$      | `10111111`       | BF  | 3     | 2        | 6       | 6           | 18         | M      | M      | M       | M       | `SBW (IZ+$4), $2` | No                |
| SBBW     | SBBW $,$          | `10001011`       | 8B  | 3     | 2        | 6       | 6           | 18         | M      | M      | M       | M       | `SBBW $4, $2`     | No                |
| SBCW     | SBCW $,$          | `10000001`       | 81  | 3     | 2        | 4       | 6           | 11         | M      | M      | M       | M       | `SBCW $4, $2`     | No                |
| SBCW     | SBCW (IX+/-$),$     | `10111010`       | BA  | 3     | 2        | 6       | 6           | 18         | M      | M      | M       | M       | `SBCW (IX+$4), $2`| No                |
| SBCW     | SBCW (IZ+/-$),$     | `10111011`       | BB  | 3     | 2        | 6       | 6           | 18         | M      | M      | M       | M       | `SBCW (IZ+$4), $2`| No                |
| XRW      | XRW $,$           | `10001111`       | 8F  | 3     | 2        | 4       | 6           | 11         | M      | 0      | M       | M       | `XRW $4, $2`      | No                |
| XRCW     | XRCW $,$          | `10000111`       | 87  | 3     | 2        | 4       | 6           | 11         | M      | 0      | M       | M       | `XRCW $4, $2`     | No                |
| BID      | BID $             | `00011000`       | 18  | 2     | 1        | 2       | 3           | 6          | M      | M      | M       | M       | `BID $2`          | No                |
| BIU      | BIU $             | `00011000`       | 18  | 2     | 1        | 2       | 3           | 6          | M      | M      | M       | M       | `BIU $2`          | No                |
| ROD      | ROD $             | `00011000`       | 18  | 2     | 1        | 2       | 3           | 6          | M      | M      | M       | M       | `ROD $2`          | No                |
| ROU      | ROU $             | `00011000`       | 18  | 2     | 1        | 2       | 3           | 6          | M      | M      | M       | M       | `ROU $2`          | No                |
| DID      | DID $             | `00011010`       | 1A  | 2     | 1        | 2       | 3           | 6          | M      | 0      | M       | 0       | `DID $2`          | No                |
| DIU      | DIU $             | `00011010`       | 1A  | 2     | 1        | 2       | 3           | 6          | M      | 0      | M       | 0       | `DIU $2`          | No                |
| CMP      | CMP $             | `00011011`       | 1B  | 2     | 1        | 2       | 3           | 6          | M      | M      | M       | M       | `CMP $2`          | No                |
| INV      | INV $             | `00011011`       | 1B  | 2     | 1        | 2       | 3           | 6          | M      | 1      | M       | M       | `INV $2`          | No                |
| BIDW     | BIDW $            | `10011000`       | 98  | 2     | 1        | 4       | 3           | 11         | M      | M      | M       | M       | `BIDW $2`         | No                |
| BIUW     | BIUW $            | `10011000`       | 98  | 2     | 1        | 4       | 3           | 11         | M      | M      | M       | M       | `BIUW $2`         | No                |
| RODW     | RODW $            | `10011000`       | 98  | 2     | 1        | 4       | 3           | 11         | M      | M      | M       | M       | `RODW $2`         | No                |
| ROUW     | ROUW $            | `10011000`       | 98  | 2     | 1        | 4       | 3           | 11         | M      | M      | M       | M       | `ROUW $2`         | No                |
| DIDW     | DIDW $            | `10011010`       | 9A  | 2     | 1        | 4       | 3           | 11         | M      | 0      | M       | M       | `DIDW $2`         | No                |
| DIUW     | DIUW $            | `10011010`       | 9A  | 2     | 1        | 4       | 3           | 11         | M      | 0      | M       | M       | `DIUW $2`         | No                |
| BYDW     | BYDW $            | `10011010`       | 9A  | 2     | 1        | 4       | 3           | 11         | M      | 0      | M       | M       | `BYDW $2`         | No                |
| BYUW     | BYUW $            | `10011010`       | 9A  | 2     | 1        | 4       | 3           | 11         | M      | 0      | M       | M       | `BYUW $2`         | No                |
| CMPW     | CMPW $            | `10011011`       | 9B  | 2     | 1        | 4       | 3           | 11         | M      | M      | M       | M       | `CMPW $2`         | No                |
| INVW     | INVW $            | `10011011`       | 9B  | 2     | 1        | 4       | 3           | 11         | M      | 1      | M       | M       | `INVW $2`         | No                |
| JP       | JP m              | `00110111`       | 37  | 3     | 2        | 2       | 6           | 6          |        |        |         |         | `JP LBL0`         | No                |
| JP Z     | JP Z,m            | `00110000`       | 30  | 3     | 2        | 2       | 6           | 6          |        |        |         |         | `JP Z, LBL1`      | No                |
| JP NC    | JP NC,m           | `00110001`       | 31  | 3     | 2        | 2       | 6           | 6          |        |        |         |         | `JP NC, LBL2`     | No                |
| JP LZ    | JP LZ,m           | `00110010`       | 32  | 3     | 2        | 2       | 6           | 6          |        |        |         |         | `JP LZ, &H7100`   | No                |
| JP UZ    | JP UZ,m           | `00110011`       | 33  | 3     | 2        | 2       | 6           | 6          |        |        |         |         | `JP UZ, LBL4`     | No                |
| JP NZ    | JP NZ,m           | `00110100`       | 34  | 3     | 2        | 2       | 6           | 6          |        |        |         |         | `JP NZ, LBL5`     | No                |
| JP C     | JP C,m            | `00110101`       | 35  | 3     | 2        | 2       | 6           | 6          |        |        |         |         | `JP C, LBL6`      | No                |
| JR       | JR +/-P             | `10110111`       | B7  | 2     | 1        | 2       | 3           | 6          |        |        |         |         | `JR ENDL`       | No                |
| JR Z     | JR Z,+/-P           | `10110000`       | B0  | 2     | 1        | 2       | 3           | 6          |        |        |         |         | `JR Z, LOOP1`   | No                |
| JR NC    | JR NC,+/-P          | `10110001`       | B1  | 2     | 1        | 2       | 3           | 6          |        |        |         |         | `JR NC, +1`     | No                |
| JR LZ    | JR LZ,+/-P          | `10110010`       | B2  | 2     | 1        | 2       | 3           | 6          |        |        |         |         | `JR LZ, -10`    | No                |
| JR UZ    | JR UZ,+/-P          | `10110011`       | B3  | 2     | 1        | 2       | 3           | 6          |        |        |         |         | `JR UZ, +126`   | No                |
| JR NZ    | JR NZ,+/-P          | `10110100`       | B4  | 2     | 1        | 2       | 3           | 6          |        |        |         |         | `JR NZ, -&HF`   | No                |
| JR C     | JR C,+/-P           | `10110101`       | B5  | 2     | 1        | 2       | 3           | 6          |        |        |         |         | `JR C, CARRY`   | No                |
| CAL      | CAL m             | `01110111`       | 77  | 3     | 2        | 4       | 6           | 12         |        |        |         |         | `CAL &H9664`      | No                |
| CAL Z    | CAL Z,m           | `01110000`       | 70  | 3     | 2        | 4       | 6           | 12         |        |        |         |         | `CAL Z, SUB1`     | No                |
| CAL NC   | CAL NC,m          | `01110001`       | 71  | 3     | 2        | 4       | 6           | 12         |        |        |         |         | `CAL NC, SUB2`    | No                |
| CAL LZ   | CAL LZ,m          | `01110010`       | 72  | 3     | 2        | 4       | 6           | 12         |        |        |         |         | `CAL LZ, SUB3`    | No                |
| CAL UZ   | CAL UZ,m          | `01110011`       | 73  | 3     | 2        | 4       | 6           | 12         |        |        |         |         | `CAL UZ, &H95CE`    | No                |
| CAL NZ   | CAL NZ,m          | `01110100`       | 74  | 3     | 2        | 4       | 6           | 12         |        |        |         |         | `CAL NZ, SUB5`    | No                |
| CAL C    | CAL C,m           | `01110101`       | 75  | 3     | 2        | 4       | 6           | 12         |        |        |         |         | `CAL C, SUB6`     | No                |
| RTN      | RTN               | `11110111`       | F7  | 1     | 0        | 5       | 0           | 14         |        |        |         |         | `RTN`             | No                |
| RTN      | RTN Z             | `11110000`       | F0  | 1     | 0        | 5       | 0           | 14         |        |        |         |         | `RTN Z`           | No                |
| RTN      | RTN NC            | `11110001`       | F1  | 1     | 0        | 5       | 0           | 14         |        |        |         |         | `RTN NC`          | No                |
| RTN      | RTN LZ            | `11110010`       | F2  | 1     | 0        | 5       | 0           | 14         |        |        |         |         | `RTN LZ`          | No                |
| RTN      | RTN UZ            | `11110011`       | F3  | 1     | 0        | 5       | 0           | 14         |        |        |         |         | `RTN UZ`          | No                |
| RTN      | RTN NZ            | `11110100`       | F4  | 1     | 0        | 5       | 0           | 14         |        |        |         |         | `RTN NZ`          | No                |
| RTN      | RTN C             | `11110101`       | F5  | 1     | 0        | 5       | 0           | 14         |        |        |         |         | `RTN C`           | No                |
| BDN      | BDN               | `11011001`       | D9  | 1     | 0        | 2a+2    | 0           | 6a+6       |        |        |         |         | `BDN`             | No                |
| BUP      | BUP               | `11011000`       | D8  | 1     | 0        | 2a+2    | 0           | 6a+6       |        |        |         |         | `BUP`             | No                |
| SDN      | SDN $             | `11011101`       | DD  | 2     | 1        | 2a+2    | 3           | 6a+6       | M      | M      | M       | M       | `SDN $2`          | No                |
| SDN      | SDN n             | `01011101`       | 5D  | 2     | 1        | 2a+2    | 3           | 6a+6       | M      | M      | M       | M       | `SDN 123`         | No                |
| SUP      | SUP $             | `11011100`       | DC  | 2     | 1        | 2a+2    | 3           | 6a+6       | M      | M      | M       | M       | `SUP $2`          | No                |
| SUP      | SUP n             | `01011100`       | 5C  | 2     | 1        | 2a+2    | 3           | 6a+6       | M      | M      | M       | M       | `SUP 123`         | No                |
| NOP      | NOP               | `11111000`       | F8  | 1     | 0        | 2       | 0           | 6          |        |        |         |         | `NOP`             | No                |
| CLT      | CLT               | `11111001`       | F9  | 1     | 0        | 2       | 0           | 6          |        |        |         |         | `CLT`             | No                |
| FST      | FST               | `11111010`       | FA  | 1     | 0        | 2       | 0           | 6          |        |        |         |         | `FST`             | No                |
| SLW      | SLW               | `11111011`       | FB  | 1     | 0        | 2       | 0           | 6          |        |        |         |         | `SLW`             | No                |
| CANI     | CANI              | `11111100`       | FC  | 1     | 0        | 2       | 0           | 6          |        |        |         |         | `CANI`            | No                |
| RTNI     | RTNI              | `11111101`       | FD  | 1     | 0        | 5       | 0           | 14         |        |        |         |         | `RTNI`            | No                |
| OFF      | OFF               | `11111110`       | FE  | 1     | 0        | 2       | 0           | 6          |        |        |         |         | `OFF`             | No                |
| TRP      | TRP               | `11111111`       | FF  | 1     | 0        | 4       | 0           | 12         |        |        |         |         | `TRP`             | No                |


---

## 5.3. Implementing Comparisons on the HD61700 using `SBC`/`SBCW`

### 5.3.1. Purpose

This section provides the definitive, optimized guide for implementing standard C comparison operators (`==`, `!=`, `<`, `<=`, `>`, `>=`) for both signed and unsigned 8-bit and 16-bit integers on the Hitachi HD61700 processor, specifically targeting the Casio PB-1000's built-in assembler constraints. Due to the absence of a dedicated Compare instruction, comparisons are achieved using the **`SBC`** (Subtract Check 8-bit) and **`SBCW`** (Subtract Check 16-bit) instructions, followed by direct conditional branching based on the processor flags.

Critically, `SBC` and `SBCW` perform a subtraction solely to **set the processor flags** based on the result, **without modifying the destination operand register**. This makes them the correct and necessary instructions for implementing non-destructive comparisons.

### 5.3.2. Core Comparison Mechanism

To compare operand A against operand B (evaluating `A comparison B`), perform the following steps in assembly:

1.  **Load Operands:** Ensure operand A is in a register (e.g., `$RegA`) or register pair (e.g., `$RegA_L/$RegA_H`). Ensure operand B is in another register (`$RegB`) or pair (`$RegB_L/$RegB_H`).
2.  **Perform Subtract Check:** Execute the appropriate Subtract Check instruction:
    *   For 8-bit: `SBC $RegA, $RegB`
    *   For 16-bit: `SBCW $RegA_L, $RegB_L` (Operands are `$RegA_L/$RegA_H` and `$RegB_L/$RegB_H`)
3.  **Conditional Branch:** Immediately after the `SBC` or `SBCW` instruction, use the appropriate conditional jump instruction (`JR C`, `JR NC`, `JR Z`, `JR NZ`). **Prefer `JR` (Relative Jump)** over `JP` (Absolute Jump) for efficiency if the target label is within the relative range **-127 to +127 bytes**. Use `JP` only if the jump distance exceeds the `JR` range.

### 5.3.3. Flag Interpretation Rules & Conditional Branching

The relevant flags set by `SBC`/`SBCW` are directly tested by conditional jump instructions:

*   **Z (Zero) Flag:** Tested by `JR Z` (Jump if Zero, A == B) and `JR NZ` (Jump if Not Zero, A != B).
*   **C (Carry/Borrow) Flag:** Tested by `JR C` (Jump if Carry/Borrow Set, Unsigned A < Unsigned B) and `JR NC` (Jump if No Carry/Borrow, Unsigned A >= Unsigned B).

#### 5.3.3.1. Unsigned Comparisons (8-bit `SBC`, 16-bit `SBCW`)

Branch directly based on Z and C flags:

*   **`if (A == B)`:** Perform `SBC`/`SBCW`, then `JR Z, label_if_true`
*   **`if (A != B)`:** Perform `SBC`/`SBCW`, then `JR NZ, label_if_true`
*   **`if (A < B)`:** Perform `SBC`/`SBCW`, then `JR C, label_if_true`
*   **`if (A <= B)`:** Perform `SBC`/`SBCW`. Jump if C=1 (`JR C, label_if_true`). If no jump, jump if Z=0 (`JR Z, label_if_true`). *(Requires two potential jumps)*
*   **`if (A > B)`:** Perform `SBC`/`SBCW`. Branch if C=0 AND Z=1. (Implementation: `JR C, label_if_false`; `JR Z, label_if_false`; code_if_true...) *(Branching on complement `<= ` might be simpler)*
*   **`if (A >= B)`:** Perform `SBC`/`SBCW`, then `JR NC, label_if_true`

#### 5.3.3.2. Signed Comparisons (8-bit `SBC`, 16-bit `SBCW`)

These require checking operand signs *before* the `SBC`/`SBCW` in some cases, combined with conditional jumps based on flags.

Define sign helper conditions (check original operand registers):

*   For 8-bit: `Sign Flag = (Operand & &H80)` (Non-zero if negative)
*   For 16-bit: `Sign Flag = (HighByteOfOperand & &H80)` (Non-zero if negative)

The implementation strategy involves checking signs and branching directly when possible, or performing `SBC`/`SBCW` and then branching based on flags if signs are the same.

*   **`if (A == B)` (Signed):** Perform `SBC`/`SBCW`, then `JR Z, label_if_true`
*   **`if (A != B)` (Signed):** Perform `SBC`/`SBCW`, then `JR NZ, label_if_true`
*   **`if (A < B)` (Signed):** *(Logic: `(SignA=1 & SignB=0) || ((SignA==SignB) & C=1)`)*
    1.  Check signs: If `SignA=1` and `SignB=0`, jump to `label_if_true`.
    2.  Check signs: If `SignA=0` and `SignB=1`, jump to `label_if_false`.
    3.  If signs are same: Perform `SBC`/`SBCW`, then `JR C, label_if_true`.
*   **`if (A <= B)` (Signed):** *(Logic: `(A < B) || (A == B)`)*
    1.  Implement logic for `A < B` as above. If true, jump to `label_if_true`.
    2.  If `A < B` was false, perform `SBC`/`SBCW` (or use flags if already done), then `JR Z, label_if_true`.
*   **`if (A > B)` (Signed):** *(Logic: `(SignA=0 & SignB=1) || ((SignA==SignB) & C=0 & Z=1)`)*
    1.  Check signs: If `SignA=0` and `SignB=1`, jump to `label_if_true`.
    2.  Check signs: If `SignA=1` and `SignB=0`, jump to `label_if_false`.
    3.  If signs are same: Perform `SBC`/`SBCW`. Branch to `label_if_false` if C=1 (`JR C, label_if_false`). Branch to `label_if_false` if Z=0 (`JR Z, label_if_false`). Otherwise (`C=0 AND Z=1`), A > B is true. *(Branching on complement `<= false` is often simpler)*
*   **`if (A >= B)` (Signed):** *(Logic: `(A > B) || (A == B)`)*
    1.  Check signs: If `SignA=0` and `SignB=1`, jump to `label_if_true`.
    2.  Check signs: If `SignA=1` and `SignB=0`, jump to `label_if_false`.
    3.  If signs are same: Perform `SBC`/`SBCW`, then `JR NC, label_if_true`. *(Simpler check: If No Borrow, it's >=)*

### 5.3.4. Implementation Examples (Optimized Assembly Snippets)

These examples use direct conditional branching and prefer `JR`. Assume `$2` holds A, `$3` holds B (8-bit). Ensure labels are <= 5 characters.

A compiler would have to save register state when required.  This is not shown in the examples below because the purpose is to focus on optimized signed and unsigned comparisons.

#### Example 5.3.4.1: `if (unsigned_a >= unsigned_b)` (Optimized)

```assembly
      ; Assume $2=A, $3=B (unsigned 8-bit)
      SBC   $2, $3      ; Perform A - B comparison
      JR    NC, ISGE    ; If C=0 (No Carry/Borrow), then A >= B. Jump.
      ; Code for FALSE case (A < B) executes if no jump
      ; ...
      JR    ENDIF       ; Jump over TRUE case (use JP if target out of range -127 to +127)
ISGE:
      ; Code for TRUE case (A >= B) executes here
      ; ...
ENDIF:
      ; ... rest of program
```

#### Example 5.3.4.2: `if (signed_a < signed_b)` (Optimized & Corrected)

```assembly
      ; Assume $2=A, $3=B (signed 8-bit)
      ; Step 1: Check Signs directly from operands
      LD    $6, $2      ; Copy A
      LD    $7, $3      ; Copy B
      AN    $6, &H80    ; Isolate SignA bit in $6 (Non-zero if neg)
      AN    $7, &H80    ; Isolate SignB bit in $7 (Non-zero if neg)

      LD    $8, $6      ; Copy isolated SignA
      XR    $8, $7      ; $8 = SignA XOR SignB. $8 is 0 if signs are same.
      JR    Z, SGNSE    ; Jump if signs are the SAME

      ; Signs are DIFFERENT. A < B is TRUE only if A is negative.
      JR    Z, ISFAL    ; If $6 (SignA bit) is zero, A is positive -> LT is FALSE
      ; SignA=1, SignB=0 -> LT is TRUE
      JR    ISTRU     ; Jump to TRUE block (A < B) (use JP if target out of range -127 to +127)

SGNSE:
      ; Signs are the SAME. Perform comparison and check Carry.
      SBC   $2, $3
      JR    C, ISTRU    ; If C=1 (borrow), LT is TRUE for same signs. Jump.
      ; No carry -> LT is FALSE for same signs
ISFAL:
      ; Code for FALSE case (A >= B) executes here
      ; ...
      JR    ENDIF       ; Jump over TRUE block (use JP if target out of range -127 to +127)
ISTRU:
      ; Code for TRUE case (A < B) executes here
      ; ...
ENDIF:
      ; ... rest of program
```

### 5.3.5. Summary Table (Branching Conditions)

| C Condition       | Type     | Branch to `label_true` if... (after `SBC`/`SBCW`)                                | Notes                                        |
| :---------------- | :------- | :--------------------------------------------------------------------------------- | :------------------------------------------- |
| `A == B`          | Unsigned | `JR Z, label_true`                                                                 |                                              |
| `A != B`          | Unsigned | `JR NZ, label_true`                                                                |                                              |
| `A < B`           | Unsigned | `JR C, label_true`                                                                 |                                              |
| `A <= B`          | Unsigned | `JR C, label_true` then (if no jump) `JR Z, label_true`                            | Requires two potential jumps                 |
| `A > B`           | Unsigned | (Branch on complement: `JR C, false; JR Z, false; else true`)                      | Simpler to branch on `<= false`              |
| `A >= B`          | Unsigned | `JR NC, label_true`                                                                |                                              |
| `A == B`          | Signed   | `JR Z, label_true`                                                                 | Same as unsigned                             |
| `A != B`          | Signed   | `JR NZ, label_true`                                                                | Same as unsigned                             |
| `A < B`           | Signed   | Check Signs first, if same: `SBC`/`SBCW`, then `JR C, label_true`                  | See Example 4.2                              |
| `A <= B`          | Signed   | Check Signs first, if same: `SBC`/`SBCW`, then `JR C` or `JR Z`                    | Combines `<` logic with `==` logic         |
| `A > B`           | Signed   | Check Signs first, if same: `SBC`/`SBCW`, branch if !C and !Z                    | Branch on complement `<= false` is simpler    |
| `A >= B`          | Signed   | Check Signs first, if same: `SBC`/`SBCW`, then `JR NC, label_true`                 | Simpler than implementing `>` directly       |

*(Note: "Check Signs first" refers to the optimized logic shown in Example 4.2 where branching occurs based on sign differences before performing `SBC`/`SBCW` when necessary.)*



### 5.3.6. Final Remarks

By using the `SBC` (8-bit) or `SBCW` (16-bit) instructions followed immediately by the appropriate direct conditional jump (`JR Z/NZ/C/NC`), all standard C relational operators can be implemented efficiently and correctly on the HD61700. Signed comparisons require initial checks on operand signs when they differ. This optimized approach avoids unnecessary flag manipulation and directly utilizes the processor's branching capabilities, preserving operand registers as required for comparisons and adhering to PB-1000 assembler constraints. **Prefer `JR` over `JP`** for jumps unless the target distance exceeds the relative jump range (**-127 to +127 bytes**).



# 6. Compiler Design Considerations

*   **Register Allocation:** The 32 registers ($0-$31) are the primary resource. Develop a strategy for mapping HLL variables and temporary values. Consider reserving some registers for specific purposes (e.g., stack frame pointer if needed, function arguments/return value). `IX` and `IZ` are natural choices for pointers or array base addresses.
*   **Data Representation:**
    *   8-bit types (char, bool): Use single registers or memory bytes.
    *   16-bit types (int, short, pointers): Use register pairs (`$n`, `$n+1`) or adjacent memory bytes (remember little-endian).
    *   Larger types: Use multiple registers or memory locations, handled via loops with 8/16-bit instructions.
*   **Function Calls:**
    *   Use `CAL` for calls and `RTN` for returns.
    *   Manage activation records (stack frames) on the User Stack (`US`). A frame might include return address (pushed by `CAL` onto `SS`, but consider moving it to US frame or managing it manually), saved registers, local variables, and parameters.
    *   Parameter Passing: Decide on a convention (e.g., first few parameters in designated registers like `$0-$3`, rest on the stack; or all on the stack).
    *   Return Values: Use a designated register (e.g., `$0` for 8-bit, `$0/$1` for 16-bit).
*   **Comparisons:** Fully covered in the previous section.
*   **Control Structures:**
    *   `if/else`: Use comparison (`SBC`/`SBCW`) followed by conditional jumps (`JR Z`, `JR NZ`, `JR C`, `JR NC`, etc.) to branch around code blocks.
    *   `while/for` loops: Use comparisons and conditional jumps back to the loop start or forward out of the loop.
*   **Memory Management:** Generate `ORG &H7000` and `START entry_label` at the beginning. Use `DS` to reserve space for static/global variables (remembering it's uninitialized) and `DB` for initialized data. Manage the stack pointer (`US`) carefully for function calls and local variable allocation.

# 7. Conclusion

Generating valid assembly code for the Casio PB-1000's built-in assembler requires careful adherence to its limitations.  Strictly adhere to the following rules.

## 7.1. Quick HD61700 assembler rules and Cheat Sheet for AI models.

1. Never use a instruction format that is not listed in the section 5.2: Full Instruction Set Reeference Guide.
1. The only valid Assembler Directives or Pseudo-Instructions are `ORG`, `START`, `EQU`, `DB`, `DS`.
1. Never use Labels or Equates with other than JP, JR or CAL.
1. Never use labels with more than 5 characters (remember restricted character set).
1. Always use the $30 for 1 and $31 for 0 (constants).
1. Best Practice: Use `SBC`/`SBCW` to compare and then `JP`/`JR` to jump on `Z` (Zero), `NZ` (Non Zero), `C` (Carry) or `NC` (Non Carry).  Depending on the logic of the program you could also use RTN (flag condition) or CAL (flag condition) directly.
1. Best Practice: Always add a comment if a STORE or a LOAD is using one of the Equates addresses, for example: `LDW $IZ, &H68F4 ; Load WORK1 address in IZ
1. Best Practice: All pseudo-instructions and instructions in CAPS.
1. Use only the documented subset of HD61700 instructions and addressing modes compatible with the PB-1000.
1. Strictly avoid undocumented features, especially Specific Index Registers (SX/SY/SZ) and multi-byte instructions (`LDM`, `STM`, etc.).
1. Never use label arithmetic.  This is not accepted by the built-in assembler.
1. Best practice: Use only the supported pseudo-instructions (`ORG`, `START`, `EQU`, `DB`, `DS`), noting `DS` does not zero memory.
1. Recomendation: Always Manage data types considering the 8-bit registers and little-endian 16-bit pairing for 16 instructions.
1. Always implement control flow using subtraction check (`SBC` or `SBCW`) for comparison followed by conditional jumps (`JR`/`JP`) like `Z`, `NZ`, `C`, `NC`.  This way the inverted flag logic does not affect, for example, if you want to jump on zero after `SBC`/`SBCW` result then simply use JR Z, dest_label.
1. *NEVER* use more than one instruction per line separated by `;` or `:` because they are invalid.
1. The TAB character is invalid for indentation and white-space.
1. Never use ASCII characters sets above ASCII code 127.
1. Given the limited LCD screen (32 columns per 4 rows), avoid abundant white-space in code.  


By following these guidelines based on the provided documentation, the target AI model should be able to construct valid assembler instructions for the built-in assembler in the PB-1000.

## 7.2. Quick PB-1000 BASIC C61 Dialect rules and Cheat Sheet for AI models.

1. All Lines need a number, and the instructions will be interpreted in that line order.
1. Do not use DEFFN, is not supported.
1. Variables cannot start with a reserved word: Basic tokenizes each line, if the token matches a BASIC C61 reserved word or function, that line will fail.   The instruction `ASCIIZ =10` will fail because that variable starts with the reserved word ASC which returns the character code of the fist character in a string.   Suggestion: add a 'bv' lower prefix to all your variables and you will avoid this problem.
1. CRITICAL: ALL IF STATEMENTS MUST BE SINGLE-LINE. IF condition THEN statement or IF condition THEN line_number or IF condition GOTO line_number. Multi-line IF...THEN...ELSE...END IF blocks are STRICTLY FORBIDDEN and will cause C61 BASIC to fail. Complex conditional logic MUST be implemented using sequences of single-line IFs and/or GOTO statements.
1. In C61 BASIC, you cannot use DIM on a variable to re-dimensionate it.  If you need to do that, you need to ERASE it first with the ERASE C61 BASIC instruction.  That array will be deleted and then you can call DIM again using the same variable name to create the ARRAY again.
1. Never use ASCII characters sets above ASCII code 127.
1. Given the limited LCD screen (32 columns per 4 rows), avoid abundant white-space in code.  
1. One space for tokens is good practice in BASIC, if possible avoid more than that.
1. Best practice: Lines numbers in multiples of 10 is preferred.

---

# Addendum: HD61700 Assembly Language for the Casio PB-1000 Built-in Assembler

**1. Purpose**

This section serves as a crucial supplement to the general HD61700 assembly language documentation. It highlights specific limitations, restrictions, and required programming practices when using the **built-in assembler** provided within the Casio PB-1000 firmware. Adherence to these points is essential for successful assembly and execution of machine code programs directly on the device. Features available in external PC-based HD61700 cross-assemblers (like the "HD61" tool) or capabilities of the HD61700 CPU itself may **not** be supported by the PB-1000's limited assembler.

**2. Key Restrictions and Required Practices**

**2.1. Instruction Set**

*   **Unsupported Instructions:** The PB-1000 built-in assembler does **not** support the multi-byte block instructions (e.g., `LDM`, `STM`, `ADBM`, `SBBM`, `ANM`, `ORM`, `XRM`, `DIUM`, `DIDM`, `BYUM`, `BYDM`). Operations on multi-byte data blocks must be implemented using loops with standard 8-bit or 16-bit instructions.
*   **Undocumented Instructions:** Avoid using any undocumented HD61700 instructions or instruction variants. They are highly unlikely to be recognized by the built-in assembler and may cause errors or unpredictable behavior. Stick strictly to the documented instruction set known to be compatible with the PB-1000 ROM/assembler.

**2.2. Addressing Modes**

*   **Supported Modes:** The following addressing modes are generally supported:
    *   Register Direct (`$n`)
    *   Immediate 8-bit (`IM8`, e.g., `10`, `&HA`)
    *   Immediate 16-bit (`IM16`, e.g., `&H1234`, `5000`)
    *   Register Indirect (`($n)`, using `$n/$n+1` pair)
    *   Indexed (`(IX+offset)`, `(IX-offset)`, `(IZ+offset)`, `(IZ-offset)`) where `offset` is an 8-bit immediate value or an 8-bit register (`$m`).
*   **Unsupported Modes:**
    *   Stack Pointer Relative Addressing (`(US+/-offset)`, `(SS+/-offset)`) is **not** supported for load/store instructions like `LD`, `ST`, `LDW`, `STW`. To access data relative to `US` or `SS`, you must first copy the stack pointer's value into `IX` or `IZ` using `GRE`/`PRE` instructions and then use standard indexed addressing `(IX/IZ+/-offset)`.
    *   Usage of Specific Index Registers (`SX`, `SY`, `SZ`) as registers or offsets is **not** supported and must be avoided.

**2.3. Label Usage and Expression Evaluation**

*   Labels can be freely declared, but only JP, JR and CAL commands may be used as operands with label names.

*   **Label Length/Format:** Labels are limited to 5 characters (`@`, `_`, A-Z, a-z, 0-9; cannot start with a digit).
*   **Labels as Operands:** Labels **cannot** be used directly as immediate numeric operands in instructions expecting a value (e.g., `LDW`, `PRE`, `AD`, `SB`, etc.).
    *   **Incorrect:** `LDW $0, MY_ADDRESS` (where `MY_ADDRESS` is a label)
    *   **Correct:** `LDW $0, &H701A` (using the literal numeric address)
*   **Label Definition:** Labels are defined by appending a colon (`:`) either on their own line or before an instruction/directive (e.g., `LOOP1:`, `MYDATA: DS 10`).
*   **Expression Evaluation:** The assembler has **very limited or no** expression evaluation capabilities.
    *   **Label Arithmetic:** Calculations involving labels (e.g., `LABEL2 - LABEL1`) are **not** supported, even within `EQU` directives. Offsets must be calculated manually and inserted as literal values.
    *   **Complex Expressions:** Avoid complex mathematical expressions in operands. Pre-calculate values.
*   **EQU Directive:** The `EQU` directive can only be used to assign a **literal numeric value** to a label.
    *   **Correct:** `BUFFER_SIZE: EQU 256`, `START_ADDR: EQU &H7000`
    *   **Incorrect:** `END_ADDR: EQU START_ADDR + BUFFER_SIZE`

**2.4. Assembler Directives**

*   **Supported Directives:** Only the following directives are supported:
    *   `ORG address`: Set origin (must be a literal numeric address).
    *   `START address`: Define entry point (must be a literal numeric address).
    *   `LABEL: EQU value`: Assign a literal numeric value to a label.
    *   `[LABEL:] DB byte1, byte2, "string", ...`: Define byte data.  
    *   `[LABEL:] DS size`: Define storage (reserves `size` bytes).
*   **Unsupported Directives:** The `DW` (Define Word) directive is **not** supported. Define 16-bit values using two consecutive `DB` definitions (remembering little-endian order). Other directives common in advanced assemblers (`#IF`, `#INCLUDE`, macros, etc.) are **not** available.
*   **`DS` Initialization:** Crucially, `DS` **reserves** space but does **not** initialize it. Memory allocated with `DS` will contain undefined values. Zero-initialization must be done explicitly with code if required.

**2.5. Data Definition and Addressing (Critical Point)**

Given the restrictions on label usage in instructions and expression evaluation, accessing global variables or data requires careful handling:

1.  **Define Data After `ORG` and `START`:** Place all global data definitions (`DS`, `DB`) immediately after the `ORG` and `START` directive at the beginning of your program space (typically `ORG &H7000`).
2.  **Calculate Absolute Addresses:** Manually calculate the exact 16-bit absolute address of each data variable based on the `ORG` address and the sizes of preceding data definitions.
3.  **Document Addresses (Optional but Recommended):** Use `EQU` to assign the calculated *numeric* address to a label for readability in comments or documentation *only*. Do **not** use this `EQU` label directly in `LDW`/`PRE`.
    *   `ORG &H7000`
	*    START MAIN
    *   `MY_VAR1_ADDR: EQU &H7000`
    *   `MY_VAR1: DS 2`
    *   `MY_VAR2_ADDR: EQU &H7002` ; (&H7000 + 2 bytes for MY_VAR1)
    *   `MY_VAR2: DS 4`
4.  **Load Pointers Using Literal Addresses:** To get the address of data into a pointer register (`IX`, `IZ`, or a register pair), load the calculated **literal numeric address** into a register pair and then transfer it to the index register:
    *   `LDW $0, &H7002` ; Load the known address of MY_VAR2
    *   `PRE IX, $0` ; IX now points to MY_VAR2
5.  **Access Data:** Use standard register indirect or indexed addressing modes with the loaded pointer:
    *   `LD $2, (IX+0)` ; Load first byte of MY_VAR2
    *   `LDW $4, (IX+2)` ; Load third and fourth bytes of MY_VAR2

**Do NOT attempt:**
*   `LDW $0, MY_VAR2` (Label not allowed as immediate value).  Never forget this!
*   To use complex `CAL`/`PPSW` tricks involving label subtraction, as label subtraction is not supported.

**2.6. Register Usage and ROM Calls**

*   **Specific Index Registers (`SX`/`SY`/`SZ`):** Do not use. These are shortcuts for `$31`/`$30`/`$0` likely only supported by external cross-assemblers.
*   **ROM Calls:** When using `CAL address` to call ROM routines:
    *   Consult documentation for the specific routine regarding input registers, output registers, and potentially modified ("clobbered") registers.
    *   Be particularly aware that registers **`$30` and `$31`** are frequently used internally by ROM routines and should generally be saved (`PHU`/`PHS`) by the caller before `CAL` and restored (`PPU`/`PPS`) afterwards if their values are needed, unless the routine's documentation explicitly states otherwise.
    *   Remember `CAL` pushes the return address onto the **System Stack (`SS`)**. Ensure `RTN` is used to return correctly.

**3. Conclusion**

The Casio PB-1000's built-in assembler is functional but significantly more limited than modern assemblers or even documented PC-based HD61700 cross-assemblers. Successful development requires strict adherence to the supported subset of instructions, addressing modes, and directives, particularly regarding label usage and data addressing. Always favor explicit, literal numeric values where addresses or offsets are required in instructions. Calculate data addresses manually based on `ORG` and `DS`/`DB` placement. Avoid assumptions based on general HD61700 documentation or external tools.

# Addendum: Errors codes generated by the built-in assembler in the Casio PB-1000

The PB-1000's built-in assembler reports errors by creating a text file containing a list of errors encountered (up to 32). It also displays the total error count (up to 99). This error file is automatically deleted before assembly begins.

**Error File Format:**

```
L llll: aaaa ERR e
```

*   `llll`: Line number (decimal) in the source `.ASM` file where the error occurred.
*   `aaaa`: Hexadecimal address (`&Haaaa`) corresponding to the instruction location at the time of the error.
*   `e`: Error code number.

**Assembler Error Codes:**

| Code | Name       | Description                                             |
| :--- | :--------- | :------------------------------------------------------ |
| 1    | (Double Def) | Double definition of a label.                           |
| 2    | (Sys Error)  | Assembler internal system error (rare).                 |
| 3    | (Op Syntax)  | Operand syntax error (incorrect format, wrong type).    |
| 4    | (Mnem Syntax)| Mnemonic syntax error (unrecognized instruction name).  |
| 5    | (Label Err)  | Labeling attempt on a command/directive that cannot be labeled. |
| OR   | (ORG Error)  | Erroneous `ORG` specification (e.g., invalid address).  |
| OM   | (Mem Error)  | Insufficient memory available to complete the assembly. |

**Object Program Execution Error:**

| Name      | Description                                                          |
| :-------- | :------------------------------------------------------------------- |
| OM error  | Address set outside of the valid machine language area (e.g., during `BLOAD, R` or via invalid pointer usage). |

# Addendum: Defining Global Data with Fixed Addresses on the PB-1000

Below is a revised addendum that reflects the PB-1000’s requirement that the **START** directive appear immediately after the **ORG** directive. This section explains how to define global data at fixed addresses so that—for example—the first global variable (your string) occupies address &H7000, while still satisfying the assembler’s syntax rules for code entry.

Because the PB-1000 built-in assembler mandates that the **START** instruction come immediately after the **ORG** directive, many experienced programmers use a layout that “pins” global data at fixed addresses by placing them right after the **START** directive. This technique permits you to know exactly where each global variable or string is located (e.g., knowing that a string is at &H7000) and avoids the need to calculate extra padding manually.

A compiler can easily keep track of all the global variables and their respective memory addresses or offsets from an initial starting address point which can easily be standardaized in the PB-1000 as &H7000.  The table in `5.3 Full Instruction Formats an Byte Sizes` will be required to keep track of the size of each particular instruction format used in the assembly code.

## 1. Layout Requirements

- **Critical Ordering:**  
  The very first byte of the program image (at the address specified by `ORG`) must contain executable code. Therefore, the **START** directive must appear immediately after `ORG`. You cannot insert any data definitions before **START**. For example:
```assembly
  ORG     &H7000         ; Program image begins at &H7000
  START   MAIN           ; Entry point must immediately follow ORG
```
  This ensures that when the program is loaded, execution begins at the intended entry point.

- **Global Data in the Fixed Region:**  
  After the **START** line, you can define all your global (static) data contiguously. Since the start of the image is now known to the programmer, you can “reserve” fixed addresses for your globals. For example:
```assembly
  ORG     &H7000
  START   MAIN
  
  __d09:  DB "0123456789",0   ; occupies address &H7000 (if the code for MAIN is linked to follow later)
  @S1:    DB "true",0         ; will occupy the next fixed address, e.g., &H700B
  HELO:   DB "Hello World",0  ; will then be placed at a known offset, e.g., &H7010
  MAIN:
      ; your code here
```
  *Note:* Although the data definitions appear after **START** in the file, the assembler will place them in the image sequentially. With careful planning, you can leave your globals at fixed, documented addresses for use as literal numeric values in your instructions.

## 2. Using Fixed Global Addresses

- **Literal Operands Only:**  
  The PB-1000 assembler does not allow you to use labels (like `HELO`) as immediate operands in instructions. This means that even though you document that `HELO` is at, for example, &H7010, you must use that literal value when loading its address:
```assembly
  LDW   $15, &H7010    ; Load pointer to "Hello World" string at known address
```
- **Document Everything:**  
  Because the assembler won’t calculate addresses for you, it is crucial to annotate your source file. List the starting address of each global variable (as you expect it to reside) in comments. This reduces errors when your code refers to these locations.

## 3. Maintenance Guidelines

- **Stability in a Fixed Layout:**  
  This technique is widely used and stable for PB-1000 assembly programming. Every time you modify a global’s size or insert new globals, you must update your documented addresses and any hard-coded references accordingly.
- **Preventing Accidental Execution:**  
  By placing the **START** directive immediately after `ORG`, you ensure that the processor begins execution at your intended entry point rather than in the data area. Do not insert code or data in between these directives if you want to guarantee proper startup.

## 4. Example Layout

Below is a `Hello-world` example that follows this convention. Global variables (or constant data) are defined after the **START** directive. Their placement is fixed in the program image, and all instruction operands referring to them must use literal numeric addresses.

```assembly
; Define ROM routine addresses for clarity
OUTCR:  EQU &H95CE
PRLB1:  EQU &H9664
KBM16:  EQU &H0BB1
BXWY:   EQU &H9C99

ORG     &H7000         ; Program image begins at &H7000
START   MAIN           ; Entry point must come immediately after ORG

; Global Data Area
__d09:  DB "0123456789",0   ; Fixed at &H7000 (11 bytes)
@S1:    DB "true",0         ; Immediately follows (__d09 ends at &H700A, so starts at &H700B)
HELO:   DB "Hello World",0  ; Follows @S1; documented to start at &H7010

; Program Code
MAIN:
    ; Example: Print "Hello World"
    LDW   $15, &H7010       ; Load pointer with the literal address of HELO
	LDW   $17, 11			; Size of the HELO
    CAL   PRLB1            ; Call routine to print a string
    CAL   OUTCR            ; Output carriage return/linefeed
    RTN                    ; End program
```
## 5. Global Variables optimization

An effective optimization, when dealing with Global Variables, is to use IZ as an offset from &H7000 so any LOAD or STORE can use the +/-IZ indirect addressing mode and avoid loading the address first in a register and then loading the global variable at that address.

This avoids having to load the address of a global variable in a register before actually loading or storing a an global variable.  In this case you only need to set IZ to &H7000 with PRE at the beginning of the program perhaps as a set of global initialization routines.  However the compiler might need to save the status of the IZ register (with GRE and then PHUW) if it calls a command or system call that alters IZ.

One complication is that LDW/STW using offsets with +/- IZ has a maximum offset of +/- 255 bytes and you have to user a general register for the offset, not an immediate, while LD/ST can use +/- 255 immediate byte offsets.  For 16-bit values, one solution could be to use two LD or ST instructions with immediates to save registers.  

Another idea for compiler designers or assembly programmers is to make IZ point to the middle of the global variable total space, (the compiler can easily determine this total area by calculating the size of all global variables).  In fact, the compiler should know when, not even this technique is enough, issue a warning and revert to loading the address in a general register and then loading or storing a value at said address.  Two operations instead of only one when IZ can be used.

Another complication is that arrays could use IZ or IX.  Unfortunately, the IY register is reserver for transfer and search commands only.  If the compiler uses IZ for global variables and IX for the frame pointer, then the compiler will have to save the state of IZ or IX, the one least used in that function, before being able to use the register as an efficient index for arrays.  Or simply use one of the main general registers for the offset for LDW and STW which is an accepted format.

All these complications are normal and have tradeofss and should easily be managed by a professional compiler in order to generate the most efficient code.

## 6. Conclusion

This addendum demonstrates the proven, stable method used by most PB-1000 assembly programmers:

- **The critical rule:** The **START** directive is placed immediately after `ORG &H7000` so that executable code occupies the starting address and is not confused with data.
- **Global Data:** Define your global variables immediately after **START**. Their fixed, sequential placement lets you easily determine their literal addresses.
- **Literal Address Use:** Since the assembler does not allow label arithmetic or use of labels as immediates, record and use the pre-calculated numeric addresses in your instructions.

By following these guidelines, you earn the benefits of a stable, maintainable layout for both code and global data on the PB-1000, fully compliant with its built-in assembler constraints.

This revised section should make clear the correct ordering—and why the **START** instruction must always follow **ORG** immediately—while still enabling the fixed global data method that experienced programmers rely on.

# Addendum: Ordering of EQU Directives

Below is an additional addendum to emphasize the correct placement of `EQU` directives. This addendum clarifies that all constant definitions must precede any code or directives that might reference them, such as `ORG` or `START`.

**Issue:**  
A common mistake is placing `EQU` directives after the `ORG` (or even later in the code) so that these constant definitions are not available when the assembler encounters instructions that reference them. Because the PB‑1000's built-in assembler processes the source file sequentially (typically in a single pass), it will generate an error if an instruction references a label that has not yet been defined.

**Resolution:**  
To avoid such errors, **all** `EQU` directives **must be defined at the very top** of your assembly source file—before any other directives (including the `ORG` directive) or any code.

**Recommendation for Assembly File Layout:**  

1. **Define ROM Routine Equates First:**  
   Place all `EQU` definitions that assign values to labels at the very beginning. This means even before the `ORG` directive.  
   
```assembly
   ; ROM Routine Equates
   PRLB1   EQU     &H9664         ; ROM routine for printing strings
   OUTCR   EQU     &H95CE         ; ROM routine for outputting carriage return/linefeed
```

2. **Then Specify the Memory Location:**  
   Follow the constant definitions with the `ORG` directive and the `START` directive immediately after it.  
   
```assembly
   ORG     &H7000         ; Program image begins at address &H7000
   START   MAIN           ; Entry point for execution
```

3. **Proceed with Global Data and Code:**  
   Finally, define your data and code that depend on these equates.  
   
```assembly
   HELO:   DB      "Hello World",0

   MAIN:
           LDW     $15, &H7000    ; Load the address of HELO
           LDW     $17, &HB       ; Load the string length
           CAL     PRLB1          ; Call the print string routine
           CAL     OUTCR          ; Call the carriage return routine
           RTN                  ; End program
```

**Rationale:**  
- **Sequential Processing:** Since the assembler reads and processes the file from top to bottom, it is essential that every label or constant used is defined first.  
- **Error Prevention:** Defining `EQU` directives first guarantees that when instructions reference these constants (e.g., via `CAL PRLB1`), the assembler already knows their values.  
- **Best Practice:** Placing all constant definitions at the very start provides clarity and helps maintain consistency across source files. It means if you later update an address (for instance, due to changes in ROM routine addresses), you only have to update it in one, clearly marked location.

By following this guideline, developers can avoid many common errors related to undefined labels or constants, streamlining the development of assembly programs for the Casio PB‑1000.

This addendum has been incorporated into the main documentation to remind developers always to define constants (using `EQU`) before any executable code.

---

# Addendum: Casio PB-1000 ROM Routines for Cross-Compiler Development

**1. Introduction**

This additional section details selected ROM routines available on the Casio PB-1000 and it complements another report on the HD61700 CPU and assembly language.

The goal is to provide the necessary information for an AI model, proficient in compiler design but unfamiliar with the PB-1000, to generate HD61700 assembly code that utilizes these ROM routines effectively. The generated assembly must be compatible with the PB-1000's built-in assembler.

**Exclusions:** Floating-point routines, BASIC interpreter internals, complex file system calls (beyond simple character I/O if available), direct hardware manipulation calls, and routines relying on undocumented CPU features are excluded. Only routines deemed relevant for a basic C cross-compiler (integer math, string output, character I/O, standard utilities) are included.

**2. Calling ROM Routines**

*   **Mechanism:** ROM routines are invoked using the `CAL address` instruction, where `address` is the 16-bit entry point of the routine (e.g., `CAL &H95D7`).
*   **System Stack (SS):** When `CAL` is executed, the CPU pushes the address of the instruction *following* the `CAL` onto the System Stack (`SS`). A standard `RTN` instruction at the end of the ROM routine will pop this address and return execution flow correctly.
*   **BASIC Interpreter Context:** The German book notes that when called from BASIC (`CALL` or `BLOAD,R`), the stack also contains a "text pointer" used by the interpreter. For standalone machine code generated by the cross-compiler (presumably started via `BLOAD,R` or a custom loader), directly manipulating this text pointer is likely unnecessary and risky. The primary concern is ensuring the `SS` is balanced upon return.
*   **Argument Passing & Return Values:** Arguments and return values are typically passed via main registers ($0-$31). The specific registers used are documented below for each selected routine. It is *crucial* that the caller sets up input registers correctly before the `CAL` and retrieves results from the specified output registers after the `RTN`.
*   **Register Preservation:** ROM routines may modify (destroy) certain registers besides those used for output. These are noted as "Destroyed". The compiler's code generator must save and restore any needed registers around `CAL` instructions if they are listed as destroyed by the routine or if their state preservation is uncertain. The German book specifically warns that registers **$30 and $31** are often used internally by ROM routines and should generally be preserved by user code calling these routines unless explicitly documented otherwise.

**3. Important System Variables (Context)**

While direct manipulation is usually avoided, awareness of these helps prevent conflicts:

*   **`OUTDV` (&H690C):** Output Device register.
    *   `0`: Display.  This is the default in the system.
    *   `2`: Printer
    *   `4`: File (via FCB pointed to by `NOWFC`)
*   **Stack Pointers:** Locations defined in memory map (`SSTOP`, `SBOT`, `FORSK`, `GOSSK`) demarcate stack areas used by BASIC/OS. The compiler should manage its stack (likely using `US`) carefully to avoid collisions. The system stack pointer register `SS` is primarily managed by `CAL`/`RTN` and interrupts.
*   **Text Pointer (`IX` Register):** Heavily used by the BASIC interpreter ROM routines for parsing commands. Standalone compiled code should generally manage its own data pointers and avoid assuming `IX` holds a text pointer unless specifically interfacing with a BASIC parsing routine (which is generally discouraged for compiled code).

**4. Selected ROM Routines (Non-Floating Point)**

*(Note: Register notation `$n` refers to main registers. `$n/m` refers to a 16-bit value held in registers `$n` (low byte) and `$m` (high byte), following little-endian convention where applicable for memory access, though register pair usage for arguments might vary.)*


*Note: Many of the ROM routines include a suggested label to use instead of the direct call by address.  Using the EQU command to declare labels for CALL destinations which are ROM rou¬tines makes programs easier to read.  Labels can be freely assigned, but only JP, JR, and CAL commands may be used as operands with label names. Therefore, usage is rather limited when compared to general use label names because LOAD or STORE instructions cannot use those labels.  This EQU commands could be placed at the top of the assembler program, for example: OUTAC: EQU &H95D7*

---

**4.1. Character and String Output**

*   **Label:** `OUTAC` (Suggested)
*   **Address:** `&H95D7`
*   **Description:** Outputs a single character based on the current `OUTDV` setting. Handles basic control codes for the display.
*   **Input:** `$16` = ASCII Character code to output.`OUTDV` (&H690C) value determines output device: 0=Display, 2=Printer, 4=File.
*   **Output:** Character sent to the specified device.
*   **Clobbers:** Recently verified: $0 to $13, `IZ` and FLAG.
*   **Compiler Relevance:** Fundamental routine for printing single characters. Forms the basis for string and number printing. Slow, too much system overhead.  
*   **Observation:** The original address for OUTAC is &HFF9E, but the instruction at &HFF9E, simply jumps to: &H95D7.

*   **Label:** `PUTCH` (Suggested)
*   **Address:** `&H030D`
*   **Description:** Outputs the character $5 on the LCD Display.
*   **Output:** Character sent to the LCD.
*   **Clobbers:**  Recently verified: $0 to $15, `IX`, `IZ` and FLAG.
*   **Compiler Relevance:** Faster than OUTAC, prefer this one for print()

*   **Label:** `OUTCR` (Suggested)
*   **Address:** `&H95CE`
*   **Description:** Outputs a Carriage Return / Line Feed sequence to the device specified by `OUTDV`.
*   **Input:** `OUTDV` (&H690C) must be set.
*   **Output:** CRLF sent to the device.
*   **Clobbers:**  Recently verified: $0 to $13, $16, `IZ`.
*   **Compiler Relevance:** Standard function for ending lines of text output (newline).

*   **Label:** `PRNUI` (Suggested)
*   **Address:** `&HDBC9`
*   **Description:** Outputs to the display the unsigned word specified in $2/3 in decimal format.
*   **Clobbers:**  Recently verified $0 to $3, $6 to $19, $23, $24, `IZ` and FLAG.
*   **Compiler Relevance:** Printing unsigned numbers.  If the data type in c is signed: Add assembler check for sign and if negative, perform CMPW, print the '-' sign and call PRNUI

*   **Label:** `PRLB1` (From the Technical Reference)
*   **Address:** `&H9664`
*   **Description:** Outputs a string to the device specified by `OUTDV`.
*   **Input:** `$15/16` = Start address of the string data. `$17/18` = Length of the string (16-bit). `OUTDV` (&H690C) must be set.
*   **Output:** String sent to the device.
*   **Clobbers:** Recently verified: $0 to $19, `IX`, `IZ`, FLAG.
*   **Compiler Relevance:** ROM routine for printing strings only if you know the length. A compiler would need to load the string's address and length into the specified registers before calling.

*   **Label:** `CLS` (Suggested, Direct CLS execution routine)
*   **Address:** `&H0482` 
*   **Description:** Clears the display screen and homes the cursor. Assumes output is directed to the display (`OUTDV=0`).
*   **Input:** None explicit (Assumes `OUTDV=0`).
*   **Output:** Screen cleared.
*   **Clobbers:** Recently verified $0 to $15, `IX`, `IZ` and FLAG.
*   **Compiler Relevance:** Standard screen clearing function.
*   **Observation:** The original address for CLS is &H95CA, but that is the entry point for BASIC CLS with more overhead.

---

**4.2. Integer / BCD **

*   **Label:** `BNBCD` (Suggested)
*   **Address:** `&HD03A`
*   **Description:** Converts an 8-bit binary integer to BCD format.
*   **Input:** `$0` = 8-bit binary value (0-255).
*   **Output:** `$0` = 8-bit BCD value (e.g., binary 25 -> BCD &H25).
*   **Clobbers:** Recently verified.  `$0`: input byte is replaced by its packed-BCD result. `$1`: used as a scratch/overflow register.  FLAG register is also altered.
*   **Compiler Relevance:** Step 1 for printing 8-bit unsigned integers. The resulting BCD byte can then be split into two ASCII digits ('0'-'9') for printing via `OUTAC`.

*   **Label:** `WRBCD` (Suggested)
*   **Address:** `&H01F1`
*   **Description:** Converts a 16-bit unsigned binary integer to a 3-byte BCD representation.
*   **Input:** `$3/4` = 16-bit binary value (little-endian pair: $3=low, $4=high). Registers `$14`, `$15`, `$16` *must be cleared to 0* before calling.
*   **Output:** `$14/15/16` = 3-byte BCD result (packed, e.g., binary 12345 -> $14=&H45, $15=&H23, $16=&H01).
*   **Clobbers:** `$0`, $3, $4, $14, $15, $16, FLAG.
*   **Compiler Relevance:** Step 1 for printing 16-bit unsigned integers (0-65535). The resulting 3 BCD bytes can be split into 5 or 6 ASCII digits for printing.

*   **Label:** `NISIN` (From German Book)
*   **Address:** `&H9A3F`
*   **Description:** Converts an 8-bit packed BCD number to an 8-bit binary integer.
*   **Input:** `$17` = 8-bit packed BCD value (e.g., &H25).
*   **Output:** `$17` = 8-bit binary value (e.g., 25).
*   **Altered:** `$19`.
*   **Compiler Relevance:** Useful for converting single-byte BCD input (e.g., from hardware or specific data files) into a usable binary format for calculations.

---

**4.3. Integer Arithmetic**

*Note: The HD61700 lacks dedicated hardware division instructions. You can use the following routines for integer multiplication and division.*

*   **Label:** `KBM16` (From German Book)
*   **Address:** `&H0BB1`
*   **Description:** Performs 16-bit x 16-bit unsigned multiplication.
*   **Input:** `$5/6` = Factor 1 (16-bit,  little-endian pair). `$15/16` = Factor 2 (16-bit, little-endian pair).
*   **Output:** `$0/1` = Result (16-bit, low part). `C=1` indicates overflow (result > 65535).
*   **Altered:** `$5/6`, `$15/16` (Input registers are altered).
*   **Compiler Relevance:** Essential for implementing 16-bit multiplication. The compiler needs to optionally handle potential overflow/underflow indicated by the Carry flag.

*   **Label:** `BXWY` (From German Book)
*   **Address:** `&H9C99`
*   **Description:** Returns quotient for x/y, cutting off remainder. x and y are integers in range of —32728 to 32767
*   **Input:** `$5/6` = Integer value y (divisor). `$15/16` = Integer value x (dividend) 
*   **Output:** `$15/16` = Quotient. Carry Flag register set (to 1) and quotient returned as 32768 when calculation results in overflow.
*   **Altered:** Contents of following registers altered: $0 to $2, $4 to $6
*   **Compiler Relevance:** Essential for implementing 16-bit division. The compiler needs to optionally handle potential overflow/underflow indicated by the Carry flag.

---

**4.4. Memory Utilities**

* **Label:** `CLRME` (From German Book)
* **Address:** `&h016E`
* **Description:** Clears a block of memory by filling it with zeros (`&h00`). Analysis of the ROM code confirms this behavior. It starts by zeroing registers `$6` through `$13` and then uses `stim` and `sti` instructions to write these zeros into the specified memory range.
* **Input:**
    * `$2/3`: Start address of the memory block to clear.
    * `$4/5`: Length (16-bit) of the memory block to clear.
* **Output:**
    * Memory block from Start Address to Start Address + Length - 1 is filled with zeros.
    * `IX`: Contains End Address + 1 upon completion.
    * `$6-$13`: Contain `&h00`.
* **Altered Registers:**
    * `$4`, `$5` (Input length is consumed).
    * `$6` through `$13` (Used internally, end containing zero).
    * `IX` (Used as the memory pointer).
    * Flags Register (`F`).
* **Notes:** Input registers `$2/$3` (Start Address) are *not* altered by the routine after the initial setup (`pre ix, $2`). This routine is very useful for zeroing memory, such as initializing data segments or clearing buffers.


*   **Label:** `BLKCPY` (Suggested)
*   **Address:** `&H0180`
*   **Description:** Copies a block of memory. Direction (up/down) is not specified, assume simple forward copy. Compare with `BUP`/`BDN` CPU instructions if overlap handling is needed.
*   **Input:** `$0/1` = Length (16-bit). `$15/16` = Source address. `$2/3` = Destination address.
*   **Output:** Memory block copied.
*   **Altered:** `$0/1` = &HFFFF. `$5-$12`, `IZ`.
*   **Compiler Relevance:** Standard library function `memcpy()`.

---

**4.5. Input**

*   **Label:** `CRTKY` (From German Book)
*   **Address:** `&H9158`
*   **Description:** Waits for a key press and returns its code. Handles keyboard debounce, repeat (?), and special keys (like `BRK`, contrast).
*   **Input:** None explicit.
*   **Output:** `$0` = Key code (standard ASCII or special code listed in German book). Flags potentially modified.
*   **Altered:** Unknown (Assume standard scratch registers $0-$7, $10-$1F might be affected).
*   **Compiler Relevance:** Basic blocking character input function `getchar()` or `getch()`.

---

**4.6 Useful Inline Assembler Routines**

```assembly
;*************************************
;* GET SECONDS FROM TIMER DATA REGISTER
;* ALTERS $0,$1
;* RETURN VALUE IN $0
;*************************************
GETSC:
gst tm,$0
gst tm,$1
sbc $0,$1  ;repeat until two consecutive reads yield ...
jr nz,GETSC
an $0,&H3F ;$0 = seconds
LD $1,0    ;zero high order
RTN
;*************************************
;* GET MINUTES FROM TIME$ AREA
;* ALTERS $0,$1,$5,$6,$7,$15,$16
;* RETURN VALUE IN $0
;*************************************
GETMN:
LDW $0, &H6BB0    ; system address for minute in TIME$
_STP:
LD $7, ($0)       ; read minutes part.  its encoded in bcd
LD $5, $7         ; make copy
AN $7, &H0F       ; bcd to decimal 1st digit right to left
AN $5, &HF0       ; bcd to decimal 2nd digit
DID $5            ; prepare to multiply by 10
LD $6, 0          ; kbm expects word
LD $15, 10
LD $16, 0
CAL KBM16         ; multiply 2nd decimal digit by 10
AD $0, $7         ; add 1st decimal digit
LD $1, 0          ; zero high byte word
RTN
;*************************************
;* GET HOUR FROM TIME$ AREA
;* ALTERS $0,$1,$5,$6,$7,$15,$16
;* RETURN VALUE IN $0
;*************************************
GETHR:
LDW $0, &H6BB1    ; system address for hour in TIME$
LD $7, ($0)       ; read minutes part.  its encoded in bcd
LD $5, $7         ; make copy
AN $7, &H0F       ; bcd to decimal 1st digit right to left
AN $5, &HF0       ; bcd to decimal 2nd digit
DID $5            ; prepare to multiply by 10
LD $6, 0          ; KBM16 expects word
LD $15, 10
LD $16, 0
CAL KBM16         ; multiply 2nd decimal digit by 10
AD $0, $7         ; add 1st decimal digit
_SALI:
LD $1, 0          ; zero high byte word
RTN
```


---
** 4.7 ROM Routines for Handling of System Errors**

These ROM routines are used by BASIC to display error messages and halt execution. A compiler can `CAL` these addresses to terminate the program upon detecting a runtime error (e.g., division by zero, array bounds violation).
The ROM Error routine expects $18 with the error code, then simply CAL the address of the specific error message required.

|Error | BASIC Code | Address | Entry Instruction Snippet | Error Meaning|
|---|---|---|---|---|
|OM | 1 | &HABBD | ld $18,$H0, jr &HAC3B | Insufficient memory|
|SN | 2 | &HABC0 | ld $18,&H02, jr &HAC3B | Syntax error|
|ST | 3 | &HABC4 | ld $18,&H03, jr &HAC3B | String too long|
|TC | 4 | &HABC8 | ld $18,&H04, jr &HAC3B | Formula too complex|
|BV | 5 | &HABCC | ld $18,&H05, jr &HAC3B | Buffer overflow|
|NR | 6 | &HABD0 | ld $18,&H06, jr &HAC3B | I/O device not ready|
|RW | 7 | &HABD4 | ld $18,&H07, jr &HAC3B | I/O device operation error|
|BF | 8 | &HABD8 | ld $18,&H08, jr &HAC3B | Improper filename|
|BN | 9 | &HABDC | ld $18,&H09, jr &HAC3B | Improper file handle|
|NF | 10 | &HABE0 | ld $18,&H0A, jr &HAC3B | File not found|
|LB | 11 | &HABE4 | ld $18,&H0B, jr &HAC3B | Low batteries in the FDD|
|FL | 12 | &HABE8 | ld $18,&H0C, jr &HAC3B | Disk write error|
|OV | 13 | &HABEC | ld $18,&H0D, jr &HAC3B | Overflow|
|MA | 14 | &HABF0 | ld $18,&H0E, jr &HAC3B | Mathematical error|
|DD | 15 | &HABF4 | ld $18,&H0F, jr &HAC3B | Double array declaration|
|BS | 16 | &HABF8 | ld $18,&H10, jr &HAC3B | Subscript out of range|
|FC | 17 | &HABFC | ld $18,&H11, jr &HAC3B | Unrecognized command|
|UL | 18 | &HAC00 | ld $18,&H12, jr &HAC3B | BASIC line number error|
|TM | 19 | &HAC04 | ld $18,&H13, jr &HAC3B | Mismatch of variable type|
|RE | 20 | &HAC08 | ld $18,&H14, jr &HAC3B | Misplaced RESUME|
|PR | 21 | &HAC0C | ld $18,&H15, jr &HAC3B | Password error|
|DA | 22 | &HAC10 | ld $18,&H16, jr &HAC3B | No data for READ|
|FO | 23 | &HAC14 | ld $18,&H17, jr &HAC3B | NEXT without FOR|
|NX | 24 | &HAC18 | ld $18,&H18, jr &HAC3B | FOR without NEXT|
|GS | 25 | &HAC1C | ld $18,&H19, jr &HAC3B | Mismatched GOSUB/RETURN|
|FM | 26 | &HAC20 | ld $18,&H1A, jr &HAC3B | Bad disk|
|FD | 27 | &HAC24 | ld $18,&H1B, jr &HAC3B | FIELD too long|
|OP | 28 | &HAC28 | ld $18,&H1C, jr &HAC3B | OPEN error|
|AM | 29 | &HAC2C | ld $18,&H1D, jr &HAC3B | File access error|
|FR | 30 | &HAC30 | ld $18,&H1E, jr &HAC3B | Framing error|
|PO | 31 | &HAC34 | ld $18,&H1F, jr &HAC3B | Parity or overrun error|


**Common Handler**: `&HAC3B` don't call this directly without specifying the error code in $18.



**5. Additional Recommendations**

1.  **Routine Selection:** Prioritize the most fundamental routines: `OUTAC`, `OUTCR`, `PRLB1`, `CLS` for output; `CRTKY` for input; `KBM16` for multiplication; `BNBCD`, `WRDBCD`, `NISIN` for number conversions. `CLRME` and `BLKCPY` are useful utilities. Approach string functions like `STR$`, `CHR$`, `HEX$` with caution regarding their calling conventions from machine code.
2.  **Register Management:** Be meticulous about saving/restoring live registers around ROM system `CAL` calls according to the altered registers.
3.  **Output Device:** You can assume that `OUTDV` (&H690C) is already set to 0 (Display) before calling display output routines.
4.  **Error Handling:** The ROM routines may generate BASIC error codes on invalid input. The compiler needs a strategy to handle this, potentially by returning an error code to the C program or terminating execution via a dedicated error handler. The German book mentions errors like SN (Syntax), TM (Type Mismatch), OV (Overflow), BS (Bad Subscript), FC (False Command), OM (Out of Memory), etc.
5.  **Implementation vs. Call:** For simple routines like `BNBCD` where the code is provided, the compiler *could* inline the assembly directly. However, calling the ROM routine (`CAL &HD03A`) is generally safer and saves code space. For complex routines like multiplication (`KBM16`), calling is strongly preferred over reimplementation unless a new routine is needed for signed or unsigned values.

# Addendum: Sample Implementation of the Stack Frame for Function Calls on the PB-1000

Below is an addendum to the original documentation that explains one sample implementation of a function call stack frame on the PB-1000.  The example implementation provided in this example is meant for illustration only. 

In a modern production compiler, both Caller-Saved and Callee-Saved Registers strategies are mixed for efficiency, which means some registers are caller-saved, while others are callee-saved, depending on the convention (e.g., System V AMD64, Microsoft x64) where certain registers are caller-saved (volatile) and others are callee-saved (non-volatile).

For a modern and professional example, in the Microsoft x64 ABI, function parameters are primarily passed in registers, but when there are more than four integer parameters, the additional ones are passed on the stack.  Additionally, the caller must allocate space on the stack for the first four register parameters (even though they are passed in registers). This is called shadow space, ensuring the callee can spill register values if needed.

In a production C compiler for the HD61700, it has been verified that the same approach would be optimal: both Caller-Saved and Callee-Saved Registers strategies should be mixed for efficiency, which means some registers are caller-saved, while others are callee-saved.  Function parameters are primarily passed in registers, but when there are more than four integer parameters, the additional ones are passed on the stack.  Additionally, the caller must allocate space on the stack for the first four register parameters (even though they are passed in registers). This is called shadow space, ensuring the callee can spill register values if needed.  Return values can be in $0/1.

This addendum shows how one might set up a complete frame stack protocol using the PB-1000’s nuances, including the fact that index registers (IX/IZ) are 16-bit and cannot be directly pushed or popped with the 16-bit PHUW/PPUW instructions.

However, this addendum demonstrates only a sample scheme for establishing a stack frame for function calls in HD61700 assembly on the Casio PB-1000. Although the example is somewhat “for demonstrations purposes only” (in a fully optimized compiler the first four parameters would be passed via registers and additional parameters via stack), it serves to illustrate how to overcome PB-1000’s limitations and use its dedicated addressing modes.

## Key Nuances on the PB-1000 and HD61700

1. **Index Register Limitations:**  
   - Registers such as IX and IZ are 16-bit and support indexed addressing.  
   - Instructions like `LDW` only allow offsets from IX or IZ, not from an arbitrary “frame pointer” held in a general-purpose register.
   - Consequently, it is beneficial to designate one of these index registers (IX, for example) as the frame pointer (FP).

2. **Pushing and Popping 16-Bit Values:**  
   - The 8-bit push/pop instructions (PHU/PPU) cannot operate on IX or IZ.  
   - You must use the 16-bit versions (PHUW/PPUW) otherwise you would need to issue two 8-bit push or pop.
   - Furthermore, the PHUW/PPUW instructions operate on register pairs in a specific fashion: if you push with, for example, `PHUW $5`, that instruction uses the pair ($4, $5) with $5 holding the high byte and $4 the low byte, and to restore you must pop using `PPUW $4`.

3. **Function Call Convention (Fictional):**  
   - **Prologue:** The caller’s frame pointer (held in IX) is saved by first transferring its 16-bit content via a `GRE` instruction into a general-purpose register pair, then pushed with PHUW.
   - **Establish Frame Pointer:** US (the user stack pointer) is loaded into IX using `PRE IX, US` so that IX becomes the base for local variable references.
   - **Locals Allocation:** Here we simulate the allocation of three 16-bit (integer) local variables by pushing three dummy 16-bit zeros. Because each push decrements US, the locals can be addressed as negative offsets from IX.
   - **Epilogue:** Local variable space is deallocated by popping these words off the US, and the caller’s FP is restored by using the correct PHUW/PPUW pairing.
   
4. **Parameter Passing (Real-World Note):**  
   - In a production compiler the first four function parameters are typically passed in registers. Additional parameters, if any, are placed on the stack.  
   - The example below is simplified to focus solely on the stack frame and local variable management.


## Stack Frame Generation for function calling

Below is an annotated code snippet that demonstrates these points. In this example, we “allocate” three 16-bit local integers, assign them values, perform a simple computation, and then restore the caller's context.

```assembly
;-------------------------------------------------------------
; FUNCTION: FUNC
; Purpose: Demonstrate allocation of three 16-bit local integers.
;
; Conventions:
;   - Use IX as the frame pointer.
;   - Caller’s FP (IX) is saved and later restored.
;   - Three locals are allocated by pushing dummy 16-bit zeros.
;     With US growing downward, the locals are at:
;         Local1: (IX - 2)
;         Local2: (IX - 4)
;         Local3: (IX - 6)
;   - The sample then assigns values:
;         local1 = 5, local2 = 10, local3 = 20,
;     and computes: result = local1 + local2 - local3.
;     The result is stored back into local1.
;
; Note: This example is fictional. A complete compiler would pass
;       the first 4 parameters in registers rather than on the stack.
;-------------------------------------------------------------

FUNC:
    ; --- Prologue: Save caller’s FP and establish new FP ---
    ; Step 1: Save the caller's IX.
    ; Copy the 16-bit IX into a register pair.
    GRE   IX, $2        ; Now $2 holds the low byte and $3 holds the high byte of IX.
    PHUW  $3           ; Correctly pushes the 16-bit value from ($2,$3). 
                       ; (Remember: PHUW operates on $5 if you had used GRE IX, $4, for example.)
    
    ; Step 2: Establish new frame pointer:
    PRE   IX, US      ; Now IX = US and acts as our frame pointer.

    ; --- Allocate Three 16-bit Local Variables ---
    ; Reserve three words (each 2 bytes), allocated by pushing dummy zeros.
    ; After allocation, locals are at the following offsets from IX:
    ;    Local1 at (IX - 2)
    ;    Local2 at (IX - 4)
    ;    Local3 at (IX - 6)

    ; Allocate Local1:
    LD    $4, 0
    LD    $5, 0
    PHUW  $5          ; Pushes 16-bit 0 from registers ($4,$5); local1 now at (IX - 2)

    ; Allocate Local2:
    LD    $4, 0
    LD    $5, 0
    PHUW  $5          ; Push another word; local2 now at (IX - 4)

    ; Allocate Local3:
    LD    $4, 0
    LD    $5, 0
    PHUW  $5          ; Push a third word; local3 now at (IX - 6)

    ; --- Reference and Manipulate the Local Variables ---
    ; Here we store initial values into each of the locals.
    ; Use LDW/STW with indexed addressing (supported for IX)

    LDW   $1, 5
    STW   $1, (IX - 2)    ; local1 = 5

    LDW   $1, 10
    STW   $1, (IX - 4)    ; local2 = 10

    LDW   $1, 20
    STW   $1, (IX - 6)    ; local3 = 20

    ; Perform computation: result = local1 + local2 - local3
    LDW   $2, (IX - 2)    ; Load local1 (5) into $2
    LDW   $3, (IX - 4)    ; Load local2 (10) into $3
    ADW   $2, $3         ; Now $2 = 15 (5 + 10)
    LDW   $3, (IX - 6)    ; Load local3 (20) into $3
    SBW   $2, $3         ; Now $2 = 15 - 20 = -5  (assuming two's complement)

    STW   $2, (IX - 2)    ; Store the result back into local1

    ; --- Epilogue: Deallocate Locals and Restore Caller’s FP ---
    ; Reverse the allocation order.
    PPUW  $4         ; Pop local3 (using PPUW with $4 since we pushed with PHUW $5)
    PPUW  $4         ; Pop local2
    PPUW  $4         ; Pop local1

    ; Restore the caller's frame pointer.
    PPUW  $2         ; Pop the saved 16-bit FP into registers ($2,$3)
    PRE   IX, $2     ; Restore IX from $2; IX now contains the caller's original FP

    ; Return from the function (RTN pops the return address from SS)
    RTN
```

## Explanation

- **Saving and Restoring IX (Frame Pointer):**  
  Because IX is 16-bit and cannot be directly pushed with PHU, we use the 16-bit push instruction. In our example we first use `GRE IX, $2` (which stores the low byte in $2 and the high byte in $3). Then we push with `PHUW $3` (because PHUW requires its operand to be the register holding the high byte) and later restore with `PPUW $2` (using the proper pairing as documented).

- **Allocation of Locals:**  
  Three local 16-bit variables are simulated by pushing three zeros. Since the US grows downward, their positions relative to the frame pointer IX after setting up the new frame are at offsets –2, –4, and –6.

- **Accessing Locals via Indexed Addressing:**  
  With IX used as our frame pointer, instructions such as `LDW` and `STW` can reference these locals using the `(IX - offset)` addressing mode. This bypasses the need for additional GRE/PRE operations for each reference.

- **Fictional Nature of the Example:**  
  While this sample illustrates the principles of building a stack frame under the PB-1000’s constraints, real-world compilers for HD61700 code would typically reserve registers for the first several parameters (often the first four) and use the stack only for additional function arguments.
  
  
## Calling Convention Recommendations for Variable Arguments (like printf)

The cdecl (C declaration) calling convention is traditionally used for functions with a variable number of arguments, such as printf(). This is because:

- The caller is responsible for cleaning up the stack after the function call.
- The number of arguments is not known to the callee, so it cannot adjust the stack accordingly.
- This allows functions like printf() to accept a flexible number of parameters.

This section served as a conceptual model showing how you might implement a user-level stack frame on the PB-1000 using IX as the frame pointer. It incorporates all the intricacies of the PB-1000’s assembler—such as the need for 16-bit push/pop operations (PHUW/PPUW) and the use of GRE/PRE for register transfers—that were discussed.

---

# Hello World sample assembly program

Below is a heavily annotated “Hello World” program written in HD61700 assembly (for the Casio PB-1000) that follows the documented restrictions. In this example…

- All ROM call addresses are defined via EQU directives at the top.  
- The ORG and START directives are placed per the PB-1000’s rules.  
- Global data (our “Hello World” string) is “pinned” at a fixed address by inserting padding so that we know its literal address.  
- All labels conform to the 5-character limit.

Review the comments for detailed explanations.

```assembly
; ============================================================
; "Hello World" Program for the HD61700 (Casio PB-1000)
; ------------------------------------------------------------
; This program outputs "Hello World" to the display using the
; built-in ROM routines. It follows the PB-1000 assembler’s
; requirements:
;
;  • All EQU directives are defined at the top.
;  • The ORG directive sets the program origin at &H7000.
;  • The START directive appears immediately after ORG.
;  • Global data is defined after START so that its absolute 
;    address is fixed and known.
;  • Only supported instructions and addressing modes are used.
;
; ROM routines used:
;    PRLB1  - Prints a string (expects the pointer in $15 and length in $17 (word))
;             Defined at &H9664.
;    OUTCR  - Outputs a Carriage Return/Line Feed sequence
;             Defined at &H95CE.
;
; Labels are limited to 5 characters, and all numeric literals
; are pre-calculated.
; ============================================================

; --- ROM Routine Equates (must come first) ---
PRLB1  EQU   &H9664     ; ROM routine: Print string
OUTCR  EQU   &H95CE     ; ROM routine: Output CR/LF

; --- Memory Layout ---
ORG    &H7000         ; Our program image begins at address &H7000
START  MAIN           ; Execution starts at the label MAIN

; --- Global Data Area ---
; To demonstrate the tracking of global areas variables, 
; and ensure our string appears at a fixed, known address, first we define some variable and keep track of it's size.
; In this sample, 16 bytes for (decimal 16 = &H10) immediately after START at &H7000.
; This puts the “HELLO” string at address &H7010.
GlVar:   DS    16

; Define the null-terminated "Hello World" string.
HELLO: DB    "Hello World", 0

; --- Program Code ---
MAIN:
        ; Load the address of the HELLO string into register pair $15/$16.
        ; According to our layout, HELLO is at &H7010.
        LDW   $15, &H7010   ; $15 gets the low byte, $16 gets the high byte of the address
		LDW   $17, 11 ; $17/16 gets the size of the string as requested by PRLB1
        
        ; Call the ROM routine to print the string.
        CAL   PRLB1        ; The routine prints the string starting at the address in $15
        
        ; Call the routine to output a newline (CR and LF).
        CAL   OUTCR
        
        ; End the program.
        RTN              ; Return—this ends execution.
; ============================================================
; End of "Hello World" program.
```

## Additional Thoughts

If later you decide to expand your program, remember that:

- Global data must remain in fixed locations. Any change (like adding more variables) requires recalculating literal addresses used in instructions.  
- Use conditional jumps (JR with conditions) as needed to control program flow but keep in mind the offset limitations of JR.

# Addendum: Optimized Pointer Dereferencing Strategy

**1. Context and Requirement**

Given the architectural constraints of the HD61700 and the specific roles assigned to index registers `IX` (Frame Pointer) and `IZ` (Optimized Global Variable Base Pointer) within a plausible compiler's strategy, it is **imperative** to implement pointer dereferencing efficiently without unnecessarily utilizing `IX` or `IZ`. Defaulting to a pattern where `IX` or `IZ` are loaded with a pointer's address merely to perform a subsequent `(IX+0)` or `(IZ+0)` access not recommended for simple dereferencing operations, as this would conflict with their primary roles and incur significant overhead from saving/restoring these vital index registers.

**2. Mandated Approach for Simple Dereferencing**

For standard C pointer dereferencing operations (e.g., `*ptr`, where `ptr` holds a direct memory address), the compiler **must** prioritize and generate code using the HD61700's **register indirect addressing modes**.

*   **Loading (`value = *ptr`):**
    *   **8-bit:** Load the pointer address into a general-purpose register pair (e.g., `$n/$n+1`). Use the `LD $reg, ($n)` instruction.
    *   **16-bit:** Load the pointer address into a general-purpose register pair (e.g., `$n/$n+1`). Use the `LDW $reg_pair, ($n)` instruction.

*   **Storing (`*ptr = value`):**
    *   **8-bit:** Load the pointer address into a general-purpose register pair (e.g., `$n/$n+1`). Load the value into another register (`$m`). Use the `ST $m, ($n)` instruction.
    *   **16-bit:** Load the pointer address into a general-purpose register pair (e.g., `$n/$n+1`). Load the 16-bit value into another register pair (`$m/$m+1`). Use the `STW $m, ($n)` instruction.

**3. Rationale**

*   **Efficiency:** Using register indirect modes (`($n)`) directly avoids the extra `PRE IX/IZ, $n` instruction and the cycles associated with it compared to the `(IX/IZ+0)` approach.
*   **Register Preservation:** Prevents unnecessary clobbering of `IX` (Frame Pointer) and `IZ` (Global Base), which would otherwise require frequent saving (`GRE`, `PHUW`) and restoring (`PPUW`, `PRE`) around simple pointer accesses, significantly increasing code size and execution time.
*   **Instruction Availability:** The HD61700 provides these efficient register indirect instructions, and they must be leveraged.

**4. Scope**

This mandate applies specifically to **simple pointer dereferencing** where the address held by the pointer is used directly without an offset calculation *at the point of access*.

Operations involving calculated offsets, such as:

*   Array indexing (`array[index]`)
*   Struct member access via pointer (`ptr->member`)

will likely still require loading the base address into `IX` or `IZ` to utilize the efficient **indexed addressing modes** (`(IX/IZ + offset)`), where `offset` is calculated or loaded separately. The compiler's logic must differentiate between these cases and choose the most appropriate addressing mode and which index register IX or IZ incurrs in less cost (probably IZ).

**5. Conclusion**

The compiler's code generator must be sophisticated enough to recognize simple pointer dereferences and utilize the direct register indirect addressing modes (`($n)`) available on the HD61700. Relying on `IX` or `IZ` as intermediate holders for addresses in these simple cases is considered a sub-optimal and unacceptable strategy due to the dedicated roles of these registers and the associated performance penalties on the target PB-1000 platform. Optimization passes should ensure that the most direct load/store instructions are used whenever feasible.

---

This addition makes it explicit that the compiler *must* avoid the inefficient pattern and use the direct register indirect instructions for basic `*ptr` operations, reserving `IX`/`IZ` for situations where their indexing capabilities are actually required (FP relative access, global base relative access, array/struct offsets).



---
# Selected Valid Assembler Programs

The following sections include several selected assembler programs that are 100% valid and functional.

They do not come from C compilation, they are 100% carefully handcrafted by humans in assembler.

## N-QUEENS SAMPLE PROGRAM

The following is a sample N-QUEENS implementation for N=8 in assembly language for the HD61700.

The program calculates the same solution 1000 times only to allow the user to be able to measure how much time it takes to run this program.

```assembly
      ORG   &H7000
      START L07
      DS    2
      DS    9
 L07: LDW   $7,1000
 L06: LD    $2,8
      PRE   IZ,&H7002
      LDW   $3,0
      LD    $0,$31
 L00: SBC   $2,$0
      JR    Z,L05
      AD    $0,$30
      ST    $2,(IZ+$0)
 L01: ADW   $3,$30
      LD    $1,$0
 L02: SB    $1,$30
      JR    Z,L00
      LD    $5,(IZ+$0)
      LD    $6,(IZ+$1)
      SB    $5,$6
      JR    Z,L04
      JR    NC,L03
      CMP   $5
 L03: AD    $5,$1
      SB    $5,$0
      JR    NZ,L02
 L04: SB    (IZ+$0),$30
      JR    NZ,L01
      SB    $0,$30
      JR    NZ,L04
 L05: SBW   $7,$30
      JR    NZ,L06
      PRE   IZ,&H7000
      STW   $3,(IZ+$31)
      RTN
```

---

## RENUMBER SAMPLE PROGRAM

The following assembler program rearranges the line numbers of a Casio PB-1000 BASIC program so the program is numbered at 10 line increments starting from line 10. 
Jump destinations (in such statements as GOTO) are also adjusted to match the renumbered program. 
This program helps to keep BASIC programs easy to follow for quicker debugging procedures. 
Since this is a machine language pro¬gram, very little time is required for the- renumbering procedure.
Analyze this program to better understand how the HD61700 assembly language works.
This program has been verified to work and was created by the Casio PB-1000 engineers.

```assembly
ORG &H7000
START &H7000
;
FCERR: EQU &H8BE9
;
RENUM: PRE IZ,&H6F51
LD $0,(IZ+0)
SBC $0,1
JP NZ,FCERR
PRE IZ,&H6F54
LDW $1,(IZ+$31)
PRE IX,$1
LDW $25,(IX+$30)
SBW $15,$15
CAL LNSCH
LDW $0,6554
SBCW $2,$0
JP NC,FCERR
PRE IZ,$25
RENM1: SBC (IZ+0),$31
JR Z,RENM3
LDI $0,(IZ+2)
RENM2: LDI $0,(IZ+0)
SBC $0,0
JR Z,RENM1
SBC $0,3
JR NZ,RENM2
LDIW $15,(IZ+$31)
CAL LNSCH
JR C,RENM2
JP FCERR
RENM3: PRE IZ,$25
RENM4: SBC (IZ+0),$31
JR Z,LNNEW
LDI $0,(IZ+2)
RENM5: LDI $0,(IZ+0)
SBC $0,0
JR Z,RENM4
SBC $0,3
JR NZ,RENM5
LDW $15,(IZ+$31)
CAL LNSCH
BIUW $2
LDW $0,$2
BIUW $2
BIUW $2
ADW $2,$0
STIW $2,(IZ+$31)
JR RENM5
;
LNSCH: PRE IX,$25
LDW $2,1
LNSC1: SBC (IX+0),$31
RTN Z
SBCW (IX+$30),$15
JR Z,SEC
LD $0,(IX+0)
LDI $0,(IX+$0)
ADW $2,$30
JR LNSC1
SEC: SBC $31,$30
RTN
;
LNNEW: PRE IX,$25
LDW $0,10
LDW $2,$0
LNNE1: SBC (IX+0),$31
RTN Z
STW $2,(IX+$30)
LD $4,(IX+0)
LDI $4,(IX+$4)
ADW $2,$0
JR LNNE1
```

---

## BASIC and ASSEMBLER MIX

The following sample shows a very common approach in the PB-1000, to create high-level user interface in BASIC integrated with low-level/critical sections in Assembler.

### FAST SORT SAMPLE PROGRAM

The following programs demonstrate how a BASIC program can interact with machine language program for the Casio PB-1000, with the BASIC portion handling input and display and user interface with easy to use interfaces and features, while the machine language portion doing the actual high speed processing or low level operations.

Sorting random data into ascending or descending order usually takes quite a bit of time, except when a high speed sort program is used. 

Up to 200 data items can be input with data input being terminated by input of a negative number.  In this particular case, the basic program writes data at a position that is beyond the end of the .EXE program.  Simply ensure a big enough are for machine language programs with the BASIC command: CLEAR ,1024 in CAL (CALCULATOR) mode which would work just fine for the small assembler program in this section.  The other two strategies in the next section are more robust and avoid having to worry about the size of the machine language program.

*This is the BASIC program for the Casio PB-1000:*

```basic
10 'INITIALIZATION
20 N=0
30 'DATA READ
40 PRINT "DATA";N+1;"=";:INPUT D
50 IF D>200 THEN PRINT "OVER FLOW";:GOTO 40
60 IF D<0 THEN 110
70 N=N+1:POKE &H7100,N
80 POKE &H7100+N,D
90 IF N>200 THEN 110
100 GOTO 40
110 'SORT SPECIFICATION
120 CLS:PRINT"SELECT ORDER"
130 LOCATE 7,2:PRINT REV;"[DESCEND]"
140 LOCATE 23,2:PRINT "[ASCEND]";NORM;
150 IN=ASC(INKEY$)
160 IF IN=249 THEN H=0 ELSE IF IN=251 THEN H=1 ELSE 150
170 POKE &H70FF,H
180 'SORT ROUTINE
200 CALL "SORT.EXE"
210 ' DATA DISPLAY
220 FOR I=1 TO N
230 PRINT PEEK(&H7100+I);
240 NEXT I
250 END
```


*And this is the assembly program for the HD61700:*
```assembly
       ORG   &H7000
       START &H7000
       LDW   $1,&H70FF
       LD    $0,($1)
       SB    $0,00
       JR    Z,PRO
       LDW   $1,&H7100
       LD    $0,($1)
       PRE   IX,&H7100
       LD    $1,1
LOOP:  LD    $2,$1
       AD    $2,1
LOOP2: LD    $3,(IX+$1)
       LD    $4,(IX+$2)
       SBC   $3,$4
       JR    C,NON
       LD    $5,$3
       LD    $3,$4
       LD    $4,$5
       ST    $3,(IX+$1)
       ST    $4,(IX+$2)
NON:   AD    $2,1
       SBC   $0,$2
       JR    NC,LOOP2
       AD    $1,1
       SBC   $0,$1
       JR    NZ,LOOP
       RTN
PRO:   LDW   $1,&H7100
       LD    $0,($1)
       PRE   IX,&H7100
       LD    $1,1
LOOP3: LD    $2,$1
       AD    $2,1
LOOP4: LD    $3,(IX+$1)
       LD    $4,(IX+$2)
       SBC   $3,$4
       JR    NC,NON1
       LD    $5,$3
       LD    $3,$4
       LD    $4,$5
       ST    $3,(IX+$1)
       ST    $4,(IX+$2)
NON1:  AD    $2,1
       SBC   $0,$2
       JR    NC,LOOP4
       AD    $1,1
       SBC   $0,$1
       JR    NZ,LOOP3
       RTN
```

### Problem with CALL from BASIC programs to exchange data with Assembler Programs

When a BASIC program uses POKE to place data at low memory addresses (e.g., &H7000, &H7001) intended for an assembly routine, and then invokes that routine using CALL "filename.EXE", a conflict arises. The CALL command often loads the assembly program's binary image starting at the address specified by the ORG directive within the .ASM source file. If ORG is set to the same low address (e.g., ORG &H7000) used by POKE, the loading process overwrites the POKEd data before the assembly code executes, leading to incorrect results or system instability.

**Solutions**:

- 1.  **Adjust `ORG` (Simple)**:
    *   **BASIC**: `POKE`/`PEEK` data at low addresses (e.g., `&H7000`-`&H7003`). It could be much more than 3 bytes.
    *   **ASM**:
        *   `ORG &H7004` (Code image starts after data area).
        *   `START MAIN` (Execution entry point label).
        *   Access data via fixed literal addresses (e.g., `LDW $0, &H7000`).

    **Note on Available Space:** The PB-1000 allows up to 4KB (4096 bytes) for machine language programs, typically starting at `&H7000`. When using the `ORG` adjustment method, the space reserved at the beginning (e.g., `&H7000` up to the adjusted `ORG` address like `&H7004`) is not limited to only a few bytes. Sufficient space can be reserved here for all necessary BASIC `POKE`/`PEEK` data, utilizing the available portion of the 4KB program area that precedes the actual code entry point defined by `START`.


- 2.  **`WORK1`: Use the general purpose calculation work buffer at &H68F4 with 24 bytes

**Guidelines**:
*   **Solution 1**: Best for minimal data tightly coupled with the ASM routine.
*   **Solution 2**: Safer for larger data or when avoiding `ORG` changes; ensure no CALC mode interaction. Always verify buffer addresses do not conflict with other operations.

Okay, here's a concise addition to the "Addendum: BASIC/Assembly Interface" to clarify the flexibility of the `ORG`-adjustment method regarding available space. This text should be inserted within or added after the description of Solution 1.

- 3.  **Use `CALCBUF` (Larger Buffer)**:
    *   **Buffer**: `CALCBUF` at `&H6D1A`. Size: `258` bytes (Range: `&H6D1A`-`&H6E1B`). Unused outside PB-1000's CALC mode.
    *   **BASIC**: `POKE`/`PEEK` within `CALCBUF` range (e.g., `POKE &H6D1A, A`).
    *   **ASM**:
        *   `ORG` can be set freely (e.g., `&H7000`).
        *   Access `CALCBUF` via fixed literal addresses (e.g., `ST $4, (&H6D1C)`).

