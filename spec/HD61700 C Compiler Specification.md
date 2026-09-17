# HD61700 C Compiler Specification

# Formal Specification for a Modern and Professional C Cross Compiler Targeting the Hitachi HD61700 CPU

## Introduction

This document defines the requirements for a modern C cross compiler designed for the Hitachi HD61700 CPU in the Casio PB-1000 handheld computer. The compiler aims to provide a joyful programming experience, enabling developers to create small benchmarking programs and detailed RAM analysis tools. It excludes floating-point support, focusing on integer operations, and omits a standard library while providing built-in functions for essential tasks. The specification ensures compatibility with the HD61700’s architecture and the PB-1000’s environment, supporting inline assembly for direct system call interaction. A comparison with the C23 standard highlights differences to aid decision-making on additional features.


## Conformance and Implementation Constraints

- This compiler implements a **deliberate subset** of C suitable for the PB-1000/HD61700. It is **not** intended to be a fully conforming C23 implementation.
- The C23 standard is referenced only for terminology and comparison.
- Any syntax or feature not explicitly listed as supported in this specification should produce a clear compile-time error.
- Implementation constraint: the compiler must be runnable and testable using **only Python** (no Java / no ANTLR / no external parser generator).
- Implement a Pratt or precedence-climbing expression parser for operator precedence/associativity.  Do not implement a LALR(1)/LR table generation, in other words, no parser generators needed.

## Compiler Objectives

The compiler must:

- Maximize programmer joy with intuitive C-like syntax, and advanced code generation surpassing the built-in PB-1000 BASIC interpreter in convenience, expressiveness, structured programming, advanced programming features and code execution performance.
- Generate HD61700 assembly code in one text file, compatible with the PB-1000’s internal assembler.
- The compiler will run on a powerful PC and will generate assembly code in an ANSI text file with `.ASM extension`.  The file will be sent to the PB-1000 where it's internal assembler will produce the executable program.
- There is no requirement to make this compiler fast and it can use all the memory it needs because it will run under a powerful PC.  The compiler should use all the resources it needs in the host PC.
- The compiler must be written in Python 3.12.x.
- The lexer and parser must be **hand-written in pure Python** (no parser generators / no ANTLR / no external code-generation step), so the whole project can be executed and tested end-to-end with just Python.
- The compiler will use the latest techniques to produce the most efficient and small code or fastest code depending on the -O command line switches.
- Enable C programs to access and analyze the PB-1000’s RAM, including system variables from `&H6000` to `&H6FFF`.
- Provide a programmer-friendly experience with intuitive syntax, built-in functions, and inline assembly.
- Handle the HD61700’s 8-bit and 16-bit operations efficiently, respecting memory and register constraints.

## C Supported Features

### 1. Data Types

- **Signed and Unsigned Integers**:
  - `char` (8-bit, signed and unsigned).
  - `int` (16-bit, signed and unsigned).
  - `short` (16-bit, equivalent to `int` for consistency).
- **Pointers**:
  - Character pointers.
  - Full pointer support for 16-bit addresses according the HD61700.  No need for banking support since this is not really needed to assembly programmers in HD61700.
  - Full Pointer arithmetic for accessing memory regions, including system variables.
- **Constants**:
  - Support `const` qualifier for read-only variables.
- **No Floating-Point**:
  - Exclude `float`, `double`, and related types due to hardware limitations.
- **Literals**:
  - Decimal literals: Standard base-10 numbers (e.g., `123`, `4567`).
  - Hexadecimal literals: Prefixed with `0x` or `0X`, representing base-16 (e.g., `0x1A3F`, `0XFF`). Case insensitive.
  - PB-1000 style of Hexadecimal literals: Prefixed with `&H` or `&h`, representing base-16 (e.g., `&H6000`, `&h6FFF`).
  - Octal literals: Prefixed with `0`, representing base-8 (e.g., `0754`, `0123`).
  - Binary literals (supported as a modern C extension): Prefixed with `0b` or `0B`, representing base-2 (e.g., `0b1011`, `0B1100`).
  - Character literals: Single characters enclosed in single quotes (e.g., `'A'`, `'z'`).
  - String literals: Sequences of characters enclosed in double quotes (e.g., `"Hello, world!"`).
- **`volatile` Type Qualifier**:
  - Purpose: Enforce direct memory access for variables modified externally (hardware, ISRs, DMA).
  - Syntax:
    - Declarations: `volatile T var;`, `T volatile *ptr;`, etc.  
    - Casts: `*(volatile T*)ADDRESS` for memory-mapped I/O.  
  - Compiler Rules:  
    1. **No optimization**: All explicit reads/writes emit ASM instructions.  
    2. **No register caching**: Always re-read from memory.  
    3. **No reordering**: Sequence volatile ops as written.  
  - PB-1000 Use Cases:  
    - System variables (e.g., `OUTDV` at `&h690C`).
    - Hardware registers (if accessed directly).
    - Shared ISR variables (hypothetical).
- **Other**:
  - Full support for C standard enumeration types (enum), allowing users to define named integer constants for improved readability and maintainability  (e.g., `enum Terminator { SPACE, TAB, CR, LF, CRLF };`).

### 2. Control Structures

- **Conditional Statements**:
  - `if`, `else if`, `else` for branching.
  - `switch` with `case` and `default` for multi-way branching.
- **Loops**:
  - `for`, `while`, `do-while` for iteration.
  - `break` and `continue` for loop control.
- **Logical and Bitwise Operations**:
  - All Logical operators including : `&&`, `||`, `!`.
  - All Bitwise operators including: `&`, `|`, `^`, `~`, `<<`, `>>`.
- **Comparison Operators**:
  - `==`, `!=`, `<`, `>`, `<=`, `>=`.
- **Control Flow**:
  - `goto` and `labels`
- **Recursion**:
  - The compiler will fully support recursion even when the limited RAM resources available in the PB-1000 might restrict the levels actually supported.

### 3. Data Structures

- **Structs**:
  - Support `struct` definitions for grouping related data.
  - Allow nested structs and pointer members.
  - Ensure alignment to 8-bit or 16-bit boundaries as needed.
- **Typedefs**:
  - Support `typedef` for creating type aliases, enhancing code readability.
- **Arrays**:
  - Support fixed-size arrays of any supported type.
  - Allow multi-dimensional arrays with pointer-based access.
- **Other**:
  - Unions and Bit-fields.  The HD61700 CPU doesn't need or require any memory alignment.

### 4. Preprocessor Directives

- **#define**:
  - Support macro definitions for constants and simple functions.
  - Example: `#define MAX_SIZE 100`.
- **#include**:
  - Support inclusion of header files for modular code.
  - Handle relative paths for user-defined headers.
- **#ifdef**, **#ifndef**, **#endif**:
  - Support conditional compilation for configuration.
- **`#pragma org 0xNNNN`:**
  - Once per translation unit.
  - Sets `ORG`.
  - The compiler does *not* support later relocation.


### 5. Functions

- **Function Definitions**:
  - Support function declarations and definitions with return types (`void`, `char`, `int`).
  - Allow parameters of supported types, passed via registers or stack.
  - Allow variable number of parameters like `printf`.
- **Calling Convention**:
  - Refer to the HD61700 manual for initial optimal calling convention ideas, cdecl and adapt stack frame examples as required to generate a professional cross compiler according to the following detailed specs.
*   **Detailed Calling Convention Specification:** The compiler shall implement the following Application Binary Interface (ABI) for function calls to ensure efficiency and predictable register usage:
    *   **Parameter Passing:**
        *   The first four 8-bit or 16-bit integer/pointer parameters are passed in designated registers. Recommended mapping:
            *   Parameter 1: `$0` (8-bit) or `$0/$1` (16-bit)
            *   Parameter 2: `$2` (8-bit) or `$2/$3` (16-bit)
            *   Parameter 3: `$4` (8-bit) or `$4/$5` (16-bit)
            *   Parameter 4: `$6` (8-bit) or `$6/$7` (16-bit)
            *(Note: Parameters larger than 16 bits are not supported directly).*
        *   Parameters beyond the fourth are pushed onto the **User Stack (`US`)** by the caller in reverse order (right-to-left, C standard).
        *   **Variable Arguments (`...`):** Functions with variable arguments (like the built-in `print`) receive their fixed arguments in registers $0-$7 as applicable. The caller is responsible for pushing all variable arguments onto the User Stack (`US`) in reverse order. The caller is also responsible for stack cleanup after the call returns (`cdecl` convention).
    *   **Return Values:**
        *   8-bit values (`char`, `unsigned char`) are returned in register `$0`.
        *   16-bit values (`int`, `short`, `unsigned int`, pointers) are returned in register pair `$0/$1` (where `$0` holds the low byte, `$1` holds the high byte - little-endian).
        *   `void` functions do not return a value; the contents of `$0/$1` are undefined upon return.
    *   **Stack Management:**
        *   The **User Stack (`US`)** is used for passing parameters (beyond the first four), storing local variables, and saving registers. The stack grows downwards (towards lower addresses).
        *   The **Index Register `IX`** shall be used as the **Frame Pointer (FP)**, pointing to a fixed location within the current function's activation record.
        *   The **System Stack (`SS`)** is used implicitly by `CAL`/`RTN` for the return address and should generally not be manipulated directly for parameters or local variables by compiled C code.
    *   **Shadow Space (Home Space):**
        *   The **caller** is responsible for allocating **8 bytes** of space on the User Stack (`US`) immediately *before* pushing any stack-based arguments (i.e., just below where the first stack parameter would be placed if parameter 5 existed). This "shadow space" corresponds to the four register parameters ($0-$7).
        *   This space *must* be allocated by the caller even if the function takes fewer than four parameters or no parameters at all.
        *   The **callee** may use this shadow space to save the values of the incoming register parameters ($0-$7) if it needs to reuse those registers. The callee is *not required* to save them there if it doesn't need to.
    *   **Register Saving Conventions:**
        *   **Caller-Saved Registers (Volatile):** Registers `$0` - `$7` (parameter/return registers) and `$10` - `$17` (general scratch registers) are considered volatile. A calling function must assume that these registers *may be modified* by the called function (callee). If the caller needs the values in these registers after the call returns, the caller must save them (e.g., push onto the User Stack) before the `CAL` instruction and restore them afterwards.
        *   **Callee-Saved Registers (Non-Volatile):** Registers `$18` - `$31` are considered non-volatile. A called function (callee) *must preserve* the values of these registers. If the callee needs to use any of these registers, it must save their original values upon entry (e.g., push onto the User Stack) and restore them before returning (`RTN`).
        *   **Index/Stack Registers:**
            *   `IX` (Frame Pointer): Must be saved by the callee upon entry and restored before return.
            *   `IZ` (Global Pointer): Considered non-volatile (callee-saved). If used by the callee, it must be saved and restored.
            *   `US` (User Stack Pointer): Modified implicitly by pushes/pops and must be correctly managed by both caller (argument pushing/cleanup) and callee (prologue/epilogue, local allocation). Its final value upon callee return must match its value before the caller pushed arguments/allocated shadow space (unless return values are passed on the stack, which isn't the case here).
            *   `SS` (System Stack Pointer): Managed by `CAL`/`RTN`; generally not modified directly.
            *   `IY`: Considered volatile (caller-saved), primarily used by block/search instructions which are unlikely to be generated directly from simple C. If used by specific built-ins or inline assembly, its state might be affected.
    *   **Function Prologue/Epilogue:**
        *   **Callee Prologue:** Save old `IX` (FP), set new `IX` to current `US`, save any callee-saved registers (`$18-$31`, `IZ` if used), allocate space for local variables by adjusting `US`.
        *   **Callee Epilogue:** Deallocate local variables, restore callee-saved registers, restore old `IX` (FP), execute `RTN`.
        *   **Caller Cleanup (for `cdecl` / varargs):** After the `CAL` returns, the caller must adjust `US` to deallocate the shadow space and any arguments pushed onto the stack.
- **Inline Assembly**:
  - Support `asm` keyword for embedding HD61700 assembly code.
  - Example: `asm("LD $2, 10; CAL &h9664");` for direct system call invocation.
  - Don't worry about preserving register states around `asm` blocks.

### 6. Built-in Functions

The compiler will provide built-in functions to replace a standard library, leveraging PB-1000 ROM system calls for efficiency and code size reduction.


- **Print Statement**:
  - Generalized `print()` statement.  See the next sections for a complete specification.
- **Character Output**:
  - `put_char(char c)`: Outputs a single character using `&h95D7`.
  - `clrscr()`: Clear the LCD Display in the PB-1000'.
- **Integer Operations**:
  - `int abs(int x)`: Returns the absolute value of a 16-bit integer.
  - `int min(int a, int b)`: Returns the smaller of two integers.
  - `int max(int a, int b)`: Returns the larger of two integers.
- **Memory Operations**:
  - `int memcpy(void *dest, void *src, int len)`: Copies memory using `&h0180`.
  - `int memset(void *dest, int value, int len)`: Fills memory using `&h016E`.
- **Clock Functions**:
  - `int get_second()`: Returns the current second from the system clock.
  - `int get_minute()`: Returns the current minute from the system clock.
  - `int get_hour()`: Returns the current minute from the system clock.
  - You can use assembler similar to following to implement these clock functions.

```assembly
;*************************************
;* GET SECONDS FROM TIMER DATA REGISTER
;* ALTERS $0,$1
;* RETURN VALUE IN $0,$1
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
;* RETURN VALUE IN $0, $1
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
;* RETURN VALUE IN $0, $1
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


#### Generalized and built-in `print()` Statement Specification

The generalized `print()` statement is a built-in function added to the HD61700 C Cross Compiler Specification for the Casio PB-1000’s Hitachi HD61700 CPU. It enhances programmer joy by allowing flexible, formatted output of mixed data types in a single statement, streamlining code for benchmarking programs and RAM analysis tools. It will support variadic arguments, including string literals, string variables (`char*`), integer variables (`int`), and terminator identifiers (`SPACE`, `TAB`, `CR`, `LF`, `CRLF`) defined via an `enum Terminator`. It is designed as a statement, not an expression, to reflect its `void` nature and prevent misuse in expression contexts (e.g., `x = print("Hello");`). The implementation leverages existing system calls (`&h9664` for strings, `&h95D7` for characters, `&hD03A` for integer-to-BCD conversion) and generates efficient HD61700 assembly at compile time, avoiding runtime parsing to respect the PB-1000’s limited RAM and 16-bit CPU constraints.

##### Key Features
- **Syntax**: `print(expr1, expr2, ...);`
  - Accepts zero or more arguments, each an expression (string literal, `char*`, `int`, or `enum Terminator` identifier).
  - Example: `print("Name=", str_var, TAB, "Age=", int_var, CRLF);` outputs `"Name=John    Age=42\n"`.
- **Argument Types**:
  - **String Literals**: e.g., `"Name="`, output via `&h9664`.
  - **String Variables**: `char*` pointers, output via `&h9664` after loading the address.
  - **Integer Variables**: `int` values, converted to decimal via `&hD03A` and output via `&h95D7`.
  - **Terminators**: Identifiers from `enum Terminator { SPACE, TAB, CR, LF, CRLF };`, outputting control characters (ASCII 32, 9, 13, 10) via `&h95D7`.  If the Compiler sees the identifier CRLF in a print() statement, the compiler will generate a call via `&h95CE`
- **Statement-Based**:
  - Defined as a statement, requiring a semicolon (e.g., `print("Hello");`), preventing use in expressions for semantic correctness.
- **Programmer Joy**:
  - Mimics `printf` but tailored to PB-1000 constraints, simplifying formatted output.
  - Uses intuitive `enum` terminators (e.g., `CRLF` instead of `"CRLF"`) for readability and type safety.
- **Efficiency**:
  - Compile-time code generation ensures no runtime parsing, minimizing resource usage.
  - Code size: ~6–8 bytes per string, 10–20 bytes per integer (due to BCD conversion), 4–8 bytes per terminator.
- **Integration**:
  - Uses documented system calls (`&h9664`, `&h95D7`, `&hD03A`, `&h95CE`), aligning with the specification’s reliance on PB-1000 ROM routines.
- **Constraints**:
  - Validates argument types (string literal, `char*`, `int`, `enum Terminator`) to prevent errors, with clear diagnostics.
  - PB-1000 display supports ASCII 9 (TAB), 13 (CR), 10 (LF);
  - Empty calls (`print();`) are valid but do nothing; can be restricted via warnings if desired.

##### Parsing Notes (Hand-written Parser)

`print()` is parsed as a **statement** (not an expression) with the following grammar:

```ebnf
print_statement := "print" "(" [ expression { "," expression } ] ")" ";" ;
```

**Parsing rule (recursive-descent):**
- When the current token is the keyword `print`, parse a `PrintStatement` node.
- Parse the parenthesized, comma-separated expression list (optional; `print();` is valid).  Depending on the type of expresion, generate/emit the correct calls to the specific type-dependant ROM or optimized print routine in ASM.  
- Require a trailing semicolon.

**Notes:**
- Treat `print` as a reserved keyword (not an identifier).
- Reuse the normal `expression` parser for each argument, so literals/identifiers/casts/etc. work naturally.
- Emit a clear error if `print` appears where an expression is required (e.g., `x = print(...);`).

##### Implementation Notes
- **Code Generation**:
  - String literals/variables: The optimized `PRSTZ` asm routine can be used (for null terminated strings).  `PRLB1` at &H9664 can only be used if you already know the length of the string.
  - Unsigned 16-bit Integers: Optimized/custom `PRUI`.  See `Optimized Printing Routines for Casio PB-1000.md`
  - Terminators: Use ROM based `PUTCH` or `OUTCR`.
  - Registers (`$0`, `$5`, `IZ`) are managed carefully, with optional `PUSH`/`POP` for safety.
- **Semantic Validation**:
  - The semantic analysis pass checks argument types using a symbol table, raising errors for invalid types (e.g., `print(struct_var);`).
- **Programmer Diagnostics**:
  - Syntax errors (e.g., missing semicolon, `print("Hello") + 1;`) are caught by the parser.
  - Semantic errors (e.g., wrong argument types) produce clear messages, enhancing usability.

##### Need for End-User Programs to Define the Terminator Enum

To use terminator identifiers (`SPACE` `TAB`, `CR`, `LF`, `CRLF`) in the `print()` and `print_string()` built-in functions of the HD61700 C Compiler for the Casio PB-1000, end-user C programs must define `enum Terminator { SPACE, TAB, CR, LF, CRLF };` either at the top of their program or by including a standard header file (e.g., `pb1000.h`) that provides this definition. This enum ensures type-safe, readable identifiers for formatting output with control characters (e.g., `print("Hello", CRLF);` for a line break). Defining the enum directly in the program or via a header allows the compiler to recognize these identifiers, enabling intuitive, error-free code that aligns with C conventions and the specification’s focus on programmer joy. Without this definition, compilation will fail (e.g., “undefined identifier: CRLF”).

### Built-in Function Implementation Strategy

*   **Modular ASM Templates:** Implement simple, fixed-logic built-ins (e.g., abs, min, max, memcpy, memset, peek*, poke*, get_second, get_minute, get_hour, put_char) each in a standalone `.ASM` file with professional-grade documentation.
*   **Selective Inclusion:** During final assembly generation, track all called template-based built-ins. Append **only** the `.ASM` files corresponding to the *used* functions to the output. Unused templates are thus automatically excluded (Dead Code Elimination) without the need for optimization command line switches.
    *(Example: No `ABS.ASM` if `abs()` is unused.)*
*   **Direct Code Generation:** Implement complex, type-dependent, or variadic built-ins (e.g., `print()`) via direct assembly emission during the compiler's code generation phase (Python AST/codegen pass), analyzing arguments at compile time.
*   **ABI Compliance:** **All** built-in functions, whether template-based or directly generated, **must** strictly adhere to the specified calling convention (parameter/return registers, caller/callee saves, stack usage).

### 7. Memory Management

- **Code Placement**:
  - Generate code starting at `&H7000`, avoiding system areas (`&H6000`–`&H6FFF`).
- **RAM Access**:
  - Allow direct access to system variables (e.g., `LCDST` at `&H68C7`) for RAM analysis.
  - Support pointer-based access to any valid memory address.

### 8. Assembly Output

- **Format**:
  - Generate a single ANSI text file containing HD61700 assembly compatible with the **PB-1000 built-in assembler**.
  - Use only documented instruction formats (see Table 5.2 in the Assembly Guide).

- **Directives (PB-1000 built-in assembler)**:
  - Supported: `EQU`, `ORG`, `START`, `DB`, `DS`.
  - Not supported: `DW` (emit words as two `DB` bytes, little-endian), macros, expressions, `#include`, conditional assembly, etc.
  - `EQU` definitions must appear at the top of the file, before `ORG`.

- **Labels (PB-1000 built-in assembler)**:
  - Max 5 characters.
  - Labels may be used **only** as operands for `JP`, `JR`, and `CAL`.
  - Do **not** emit labels as immediates or address operands for `LD`, `ST`, `LDW`, `STW`, `PRE`, etc. Use literal numeric addresses (e.g., `&H7A20`) where an address is required.

- **Compatibility**:
  - Use only documented addressing modes (direct literals, register indirect `($n)`, indexed `(IX/IZ ± offset)` as supported).
  - Avoid undocumented features like `SX`, `SY`, `SZ`.

### 9. Optimization

- **Register Allocation**:
  - Prioritize main registers (`$0` to `$31`) for variables and temporaries.
  - Spill C locals (named variables) to frame slots via ST/LD using IX-relative addressing.
  - Spill ephemeral temporaries under register pressure using PUSH/POP on the user stack (US).
- **Instruction Selection**:
  - Favor 16-bit instructions (`LDW`, `ADW`) for performance when handling `int`.
  - Use system calls for complex operations (e.g., multiplication via `&h9CE4`).
- **Code Size**:
  - Minimize code size to fit within the PB-1000’s limited RAM, optimal selection of instruction to issue.
- **Register Allocation Policy for $30/$31**:
  - **Immutable by default:** `$30` and `$31` hold the PIC base and must never be used for ordinary temporaries.
  - **Last resort fallback:** If *all* other caller saved registers (`$24 to $29`, etc.) are exhausted, the allocator may temporarily assign a value to `$30` or `$31`.
  - **Auto restore before ROM calls:** Any time the compiler lowers a ROM `CAL` instruction, it must check if `$30/$31` were dirtied and, only if so, emit instructions to reload their original PIC base values immediately before the call.  Also before returning from MAIN.
- **Suggested Preferred scratch register block is `$24 to $29`**
  - Allocator may spill/expand beyond it.


### 10. Error Handling

- **Normal Output**:
  - While a silent output would be the normal-composed-professional behavior for a compiler that did not detected errors or warnings, in the PB-1000, we need to know how many bytes are required for the program to run, and issue a CLEAR ,XXXX instruction to reserve the required space.  So if all goes well, the compiler needs to display a concise, brief, succint and professional message indicating program size in bytes and total space required from &H7000 which is normally the most common starting point.
- **Diagnostics**:
  - Provide clear error messages for syntax errors, type mismatches, and memory violations.
  - Warn about potential register clobbering in inline assembly.
- **System Call Errors**:
  - Include optional compiler directive to Check flags (e.g., carry for overflow in `&h9CE4`) and propagate errors to C code.  For example '-underflow' or '-overflow'.

## Features Not Supported

- **Floating-Point Types**:
  - Exclude `float`, `double`, and related operations due to hardware limitations.
- **Standard Library**:
  - Omit `<stdio.h>`, `<stdlib.h>`, etc., relying on built-in functions.
- **Recent C Features Not Supported**:
  - Exclude `constexpr`, `typeof`, and digit separators to simplify implementation.
  - Exclude 64-bit integers (`int64_t`) due to performance overhead on 16-bit hardware.
- **Advanced Preprocessor**:
  - Exclude `#error`, and complex macro expansions.
- **Function Pointers**:
  - Exclude function pointers to avoid complexity in code generation.  HD61700 does not support dynamic indirect JUMP instructions.
- **Dynamic Memory Allocation**:
  - Exclude `malloc` and `free`, as the PB-1000 lacks a heap.

## Additional Features for Programmer Joy

- **Header Files**:
  - Use pre-existing header (e.g., `pb1000.h`) with PB-1000 system variable addresses and system call wrappers.
  - Example: `#define LCDST (*(volatile char *)0x68C7)`.
  - Professional documentaion in the the header file on how to use it.
- **Compiler Command Line Options**:
	- **`-symboltable`**: Print a nicely formatted representation of the **symbol table**. (Function: `print_symbol_table` in Python)
	- **`-ir`**: Print a well-structured **Intermediate Representation (IR)**.
	- **`-ast`**: Output a clear and detailed **Abstract Syntax Tree (AST)**.
	- **`-tokens`**: Display all **tokens** generated during lexical analysis.
	- **`-cfg`**: Generate a visual representation of the **Control Flow Graph**.
	- **`-dfg`**: Print the **Data Flow Graph**, showing dependencies between variables.
	- **`-debug`**: Provide detailed **debugging output**, including error diagnostics.
	- **`-verbose`**: Show additional details for each **compilation phase**.
	- **`-errors`**: Print only the **syntax and semantic errors** for quick debugging.
	- **`-labels`**: The compiler will print a user friendly mapping of all used labels as applicable.
	- **`-dump`** to print all the relevant compiler data structures using the functions created for `-ir`, `-ast`, etc. as applicable.  This option will ask the user for confirmation before printing anything.
	- **`-comments`**: Enable **verbose comment generation** in the **assembler output**, providing informative annotations alongside instructions. For example,  
```assembly
LDW $0, &h7002  ; Load global variable 'result' into register pair $0 and $1
```	
- **Debugging Support**:
  - Generate source-level debug information (e.g., line numbers) for PB-1000 debugging tools.  Define Optional command line to support this.
  - There is no need for the compiler to try to recover from errors.  When the compiler finds an error it can report it and exit.  If `-dump` was used, print all applicable compiler data structures before exit.
- **Syntax Highlighting**:
  - Ensure generated assembly is formatted for readability (e.g., consistent indentation).
- **Documentation**:
  - Include a user manual with examples for benchmarking and RAM analysis.

## Toolchain Integration Workflow 

1. **Cross-Compile**:  
   - Open-source HD61 cross-compiler processes `.ASM` → emits errors (if any).  HD61 is a command line PC/Windows program.  This can only be used when the compiler is finally able to produce a full complete assembler program.  
   - Additionally, if the AI can execute python programs, it can invoke pb1klinter.py which has been developed exclusively for this project and to help AI to fix bugs in the generated assembler HD61700 code.
   - **Debug Cycle**:  
     - Human relays errors → AI regenerates `.ASM`.  (HD61)
	 - AI analyzes errors output from pb1klinter.py
     - Repeat until HD61 assembles successfully.  

2. **Emulator Testing**:  
   - `RAMTRANS` (CLI) loads binary into Delphi-based PB-1000 emulator.  
   - **Debug Cycle**:  
     - Run in emulator → detect errors → AI adjusts code.  
     - Repeat until success.  

3. **Hardware Deployment**:  
   - Transfer binary to real PB-1000 via serial cable.  
   - **Debug Cycle**:  
     - Test on hardware → detect errors → AI adjusts code.  
     - Repeat until success.  

**Key Tools**:  

All the tools are already available for this project and have been confirmed to work correctly:

- HD61 cross-compiler (error reporting).  
- Delphi emulator + `RAMTRANS` (testing).  
- Serial utilities (real-device transfer).  

## Compiler optimizations

The following table specifies the list of optimization techniques to implement in the compiler.

| **Rank** | **Optimization Technique**                                  | **Proposed `-O` Switch**                                       | **Primary Gains**                         |
|----------|--------------------------------------------------------------|----------------------------------------------------------------|-------------------------------------------|
| **0**    | **No Optimization**                                          | `-O0`                                                        | Primarily for debugging and development, with a simple, stack-based code generator: Straight parser → AST → naive codegen.	Simple, local, stack-based register allocations strategy.  Direct mapping of AST to assembler.  No constant propagation, instruction scheduling, or redundant code removal.  Preserves source structure for easy debugging. |
| **1**    | **Local Dead Code Elimination**                              | `-Olocal_dead_code_elimination`                              | **Smallest Code** — Removes unreachable and redundant code. |
| **2**    | **Constant Folding & Propagation**                           | `-Oconstant_folding_and_propagation`                         | **Smallest Code / Efficiency** — Precomputes constant expressions at compile time, reducing unnecessary operations. |
| **3**    | **Peephole Optimization (Simple)**                           | `-Opeephole_optimization_simple`                             | **Smallest Code** — Rewrites small instruction patterns to be more compact. |
| **4**    | **Advanced Peephole Optimization**                           | `-Opeephole_optimization_advanced`                           | **Smallest Code** — Expands the peephole window to remove more redundant sequences. |
| **5**    | **Exhaustive Peephole/Rewriting**                            | `-Opeephole_optimization_exhaustive`                         | **Smallest Code** — A final cleanup to ensure the shortest possible instruction sequences. |
| **6**    | **Global Register Allocation (Graph Coloring)**              | `-Oglobal_register_allocation_graph_coloring`                | **Efficiency** — Minimizes spills and improves register usage across the entire function. |
| **7**    | **Loop Invariant Code Motion**                               | `-Oloop_invariant_code_motion`                               | **Efficiency** — Moves constant computations out of loops to reduce redundant execution. |
| **8**    | **Basic Interprocedural Refactoring**                        | `-Ointerprocedural_refactoring_basic`                        | **Efficiency** — Cleans up redundant operations across function boundaries. |
| **9**    | **Interprocedural Optimization**                             | `-Ointerprocedural_optimization`                             | **Efficiency** — Further streamlines cross-function execution paths and removes redundant calculations globally. |
| **10**   | **Simple Instruction Scheduling**                            | `-Oinstruction_scheduling_simple`                            | **Fastest Execution Speed** — Performs local reordering within blocks to reduce stalls. |
| **11**   | **Multi-Pass Instruction Scheduling**                        | `-Oinstruction_scheduling_multi_pass`                        | **Fastest Execution Speed** — Optimizes instruction order across basic blocks to reduce stalls. |
| **12**   | **Advanced Branch Prediction/Conditional Optimization**      | `-Obranch_prediction_and_conditional_optimization_advanced`  | **Fastest Execution Speed** — Reorders branches for optimal CPU prediction performance. |
| **13**   | **Aggressive Function Inlining**                             | `-Oaggressive_function_inlining`                             | **Fastest Execution Speed** — Eliminates function call overhead by merging functions into their callers. |

- **Post-Expansion Replay:**  After any pass that inlines or reorganizes code (e.g. interprocedural or inlining), if the user’s `-O…` switches enabled them, rerun in this order:  
  1. Constant folding & propagation (rank 2)  
  2. Local dead-code elimination (rank 1)  
  3. All peephole passes (ranks 3, 4 and 5)  
  4. Simple instruction scheduling (rank 10)  

  This final cleanup ensures that newly generated patterns are optimized to fix-point before emission.

- The following table outlines which optimization options should be invoked for each high-level flag (`-O1`, `-O2`, `-O3`, `-Osize`, and `-Ospeed`) in an order that ensures feasibility and efficiency. The justification column explains why these passes are sequenced in this way.

| **Optimization Level** | **Ordered Optimization Passes Invoked**                                                                                          | **Justification**                                                                                              |
|------------------------|-------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| **`-O1`**             | `-Olocal_dead_code_elimination`, `-Oconstant_folding_and_propagation`, `-Opeephole_optimization_simple`, `-Oinstruction_scheduling_simple` | Basic optimizations focused on cleaning up redundant code, simple instruction scheduling, and minor improvements. |
| **`-O2`**             | `-Olocal_dead_code_elimination`, `-Oconstant_folding_and_propagation`, `-Opeephole_optimization_advanced`, `-Oloop_invariant_code_motion`, `-Oglobal_register_allocation_graph_coloring`, `-Oinstruction_scheduling_multi_pass` | Incorporates loop optimizations and register allocation while refining instruction scheduling for moderate performance gains. |
| **`-O3`**             | `-Olocal_dead_code_elimination`, `-Oconstant_folding_and_propagation`, `-Opeephole_optimization_exhaustive`, `-Oloop_invariant_code_motion`, `-Oglobal_register_allocation_graph_coloring`, `-Ointerprocedural_refactoring_basic`, `-Ointerprocedural_optimization`, `-Oinstruction_scheduling_multi_pass`, `-Obranch_prediction_and_conditional_optimization_advanced`, `-Oaggressive_function_inlining` | The most aggressive optimizations, including whole-function analysis, branch prediction, deep instruction scheduling, and function inlining to maximize efficiency and speed. |
| **`-Osize`**          | `-Olocal_dead_code_elimination`, `-Oconstant_folding_and_propagation`, `-Opeephole_optimization_simple`, `-Opeephole_optimization_advanced`, `-Opeephole_optimization_exhaustive`, `-Oglobal_register_allocation_graph_coloring`, `-Ointerprocedural_refactoring_basic` | Focused solely on reducing code size using dead code elimination, constant folding, multiple peephole passes, and minimal interprocedural refactoring. |
| **`-Ospeed`**         | `-Oaggressive_function_inlining`, `-Oinstruction_scheduling_multi_pass`, `-Oloop_invariant_code_motion`, `-Oglobal_register_allocation_graph_coloring`, `-Oinstruction_scheduling_simple`, `-Obranch_prediction_and_conditional_optimization_advanced`, `-Ointerprocedural_optimization` | Prioritizes execution speed via advanced scheduling, efficient register allocation, branch prediction, and cross-function optimization. |

If necesary, you can adjust the definitions to adapt them for the PB-1000, but clearly document in the code any deviations.

## Intermediate Representation (IR) Strategy

To clarify the mandated strategy for the Intermediate Representation (IR) to effectively support the specified range of optimizations, particularly those involving complex data flow analysis.

**1. Specified IR Approach to use in this compiler**

1.  **Base IR:** The compiler shall utilize a **Three-Address Code (TAC)** based representation as its primary high-level IR generated from the Abstract Syntax Tree (AST).
2.  **Optimization Phase IR:** For performing data flow analysis and the core optimization passes (including but not limited to constant propagation, dead code elimination, loop optimizations, global register allocation), the TAC IR **shall be converted into Static Single Assignment (SSA) Form**.
3.  **Post-Optimization:** Following the optimization phases, the IR must be converted **out of SSA Form** (e.g., eliminating Phi functions) before final assembly code generation.
4.  **Target Mapping:** The code generation phase must efficiently map the optimized, post-SSA TAC representation onto the HD61700's predominantly two-operand instruction set architecture. This requires intelligent instruction selection and leveraging peephole optimizations to mitigate potential inefficiencies arising from the TAC-to-2AC mapping.

**2. Rationale**

While the HD61700 features many two-operand instructions, the benefits of SSA for simplifying and enabling robust implementation of the required advanced global and interprocedural optimizations outweigh the challenges of mapping from a TAC/SSA representation during final code generation. A direct two-address code IR is considered less suitable for facilitating the specified optimization suite. This strategy prioritizes optimization power and correctness over initial IR-to-target mapping simplicity.

## Conclusion

This specification defines a modern and professional C cross compiler for the HD61700 CPU, balancing programmer joy with hardware constraints. It supports advanced features for benchmarking and RAM analysis, leveraging inline assembly and system calls for efficiency. The compiler ensures compatibility with the PB-1000’s assembler, enabling small, performant programs maximizing programming joy for retro computing enthusiasts.


# Addendum: Label Naming Convention for Generated Assembly (Revised)

**1. Purpose**

To enhance the readability and debugging potential of the HD61700 assembly code generated by the C cross-compiler, while adhering to the Casio PB-1000 built-in assembler's constraints (5-character maximum label length), the following naming convention shall be used for all generated labels. This convention also facilitates the generation of informative comments when the `-comments` compiler option is enabled.

**2. Format**

All compiler-generated labels will follow the format:

`Pxxxx`

Where:

*   **`P`**: A single uppercase letter prefix indicating the *type* of code construct or data the label refers to (see Section 3 below).
*   **`xxxx`**: A sequence of exactly four characters used to ensure uniqueness. These characters must be chosen from the set:
    *   Digits: `0` - `9` (10 characters)
    *   Uppercase Letters: `A` - `Z` (26 characters)
    *   **Total allowed characters per position:** 10 + 26 = **36 characters**

This format provides 36<sup>4</sup> = **1,679,616** unique identifiers *per prefix*, which is more than sufficient for the PB-1000's 4KB code size limit.

**3. Prefix Definitions**

The following table defines the single-letter prefixes (`P`), their intended meaning, and examples of how the `-comments` comment generation option should utilize them. The compiler maintains a mapping between generated labels and the original C source constructs (function names, variable names, statement types/locations).

| Prefix | Meaning / Usage                                    | Example Label(s) | Comment Example (`-comments` enabled)                                                                       | Notes                                                                                                                                                           |
| :----- | :------------------------------------------------- | :--------------- | :--------------------------------------------------------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **F**  | **Function** Entry Point                           | `FMAIN`, `F1AB2` | `FMAIN: EQU &H70XX ; Entry point for function 'main'`                                                  | Marks the starting address of a compiled C function. The EQU is conceptual; the label might directly precede the first instruction.                       |
| **I**  | **If** Statement Condition/Block Start             | `I0001`, `I0A3F` | `JR Z, I0001 ; Jump if Zero to 'if' block start @ line Y` <br> `I0001: ; Start of 'if' block @ line Y` | Target for jumping *into* the 'then' part or marks the start of the block.                                                                                      |
| **E**  | **Else** Statement Block Start                     | `E0001`, `E0A3F` | `JR E0001 ; Jump to 'else' block @ line Z` <br> `E0001: ; Start of 'else' block @ line Z`            | Target for jumping *into* the 'else' part or marks the start of the block.                                                                                      |
| **N**  | **End If** (Merge Point after If/Else)             | `N0001`, `N0A3F` | `JR N0001 ; Jump to end of 'if/else' @ line W` <br> `N0001: ; End of 'if/else' @ line W`             | The common execution point after 'then' or 'else'.                                                                                                              |
| **W**  | **While** Loop Condition Check                     | `W0001`, `WFFFF` | `JR W0001 ; Jump to 'while' condition check @ line X` <br> `W0001: ; 'while' condition check @ line X` | Top of a `while` loop where the condition is evaluated.                                                                                                         |
| **B**  | Loop **Body** Start (While/For/Do)                 | `B0001`, `B1234` | `B0001: ; Start of loop body @ line X+1`                                                               | Optional: Marks the first instruction *inside* the loop body.                                                                                                   |
| **X**  | Loop E**x**it / End Loop                           | `X0001`, `XAABB` | `JR X0001 ; Jump to loop exit (break) @ line M` <br> `X0001: ; End of loop @ line M`                  | Target for `break` statements or the point after the loop finishes normally.                                                                                    |
| **R**  | **For** Loop Condition Check                       | `R0001`, `R001A` | `JR R0001 ; Jump to 'for' condition check @ line P` <br> `R0001: ; 'for' condition check @ line P`    | Location where the `for` loop condition is checked.                                                                                                             |
| **P**  | **For** Loop u**P**date/Increment                  | `P0001`, `P55CC` | `JR P0001 ; Jump to 'for' update @ line Q` <br> `P0001: ; 'for' update expression @ line Q`           | Location of the `for` loop's update expression. `continue` in a `for` loop often jumps here.                                                                    |
| **O**  | **Do**-While Loop Start                            | `O0001`, `O0CDE` | `O0001: ; Start of 'do-while' body @ line S`                                                           | The entry point of the `do-while` loop body.                                                                                                                    |
| **H**  | **Switch** Dispatch / Jump Table                   | `H0001`, `H000A` | `H0001: ; Start of 'switch' dispatch @ line T`                                                         | Start of the code evaluating the switch expression.                                                                                                             |
| **C**  | **Case** Label (Switch)                            | `C0001`, `C001A` | `JR C0001 ; Jump to 'case 10' @ line U` <br> `C0001: ; 'case 10' block @ line U`                     | Entry point for a specific `case` block.                                                                                                                        |
| **D**  | **Default** Case Label (Switch)                    | `D0001`, `D000A` | `JR D0001 ; Jump to 'default' case @ line V` <br> `D0001: ; 'default' case block @ line V`           | Entry point for the `default` block.                                                                                                                            |
| **G**  | **Goto** Target Label                              | `G0001`, `GABCD` | `JR G0001 ; Jump to label 'user_label' @ line L` <br> `G0001: ; Target for 'goto user_label' @ line L` | A label explicitly defined in C (`user_label:`) and targeted by `goto user_label;`.                                                                           |
| **V**  | **Variable** Definition (Global/Static)            | `V0001`, `VCNTR` | `VCNTR: DS 2 ; Definition for global variable 'counter'`                                             | Marks the actual storage location defined using `DS` or `DB`. This label itself isn't typically a jump target.                                                  |
| **A**  | **Address** of Variable (`EQU` only)             | `A0001`, `AVARZ` | `A0001: EQU &H701A ; Address of global variable 'config_flags'`                                    | *Compile-time only.* Assigns the calculated *absolute address* via `EQU`. **Not** a jump target. **Not** the variable definition itself.                        |
| **S**  | **String** Literal Address (`EQU` only)            | `S0001`, `SMSG1` | `SMSG1: EQU &H7050 ; Address of string literal "Error\n"`                                          | *Compile-time only.* Assigns the *absolute address* of a string literal via `EQU`. **Not** a jump target.                                                      |
| **L**  | **Local** / Generic / Internal Label               | `L0001`, `LTEMP` | `JR L0001 ; Internal jump` <br> `L0001: ; Temporary target`                                        | General-purpose label for compiler-generated jumps not fitting other categories (e.g., short-circuit evaluation, temporary calculations).                     |

**4. Uniqueness Generation (`xxxx`)**

The compiler must ensure that the `xxxx` portion makes the entire 5-character label unique *within the scope of the entire assembly file*. A simple and effective strategy is to maintain a single global counter:

1.  Initialize a counter (e.g., `label_counter`) to 0.
2.  Whenever a new label is needed, regardless of its prefix:
    *   Increment `label_counter`.
    *   Convert `label_counter` into a 4-character base-36 string (`0-9`, `A-Z`) padding with leading zeros if necessary (e.g., 1 -> "0001", 37 -> "0011", 1300 -> "0104").
    *   Combine the appropriate prefix (`P`) with the generated 4-character string (`xxxx`).
3.  This guarantees uniqueness across all label types throughout the program.

**5. Comment Generation (`-comments` option)**

When the previously defined `-comments` command line option is enabled:

*   The compiler will access its internal mapping between the generated `Pxxxx` label and the original C source construct.
*   For label **definitions** (e.g., `FMAIN:`, `W0001:`, `VTEMP: DS 2`), the comment should describe what the label marks (e.g., function entry, loop condition check, variable definition) and ideally include the original C name or location (e.g., function name, variable name, approximate line number).
*   For instructions that **use** a label as an operand (e.g., `JR Z, I0001`, `CAL FFUNC`), the comment should describe the *target* of the jump or call based on the prefix and the associated C construct.

**6. Conclusion**

This structured label naming convention provides significant benefits for understanding the generated assembly code for the Casio PB-1000. By using meaningful prefixes, ensuring uniqueness, and integrating with the `-comments` comment option, the compiler can produce output that is easier to relate back to the original C source code, aiding in debugging and verification, while fully respecting the target assembler's limitations. The distinction between runtime jump targets (most prefixes) and compile-time definition/address labels (`V`, `A`, `S`) is crucial.

# Addendum: The `volatile` Type Qualifier

**1. Purpose**

The `volatile` type qualifier needs to be included in this C specification to provide the standard C mechanism for informing the compiler that a variable or memory location has special properties. It indicates that the value of the object might change at any time without any action being taken by the code the compiler finds nearby (e.g., modified by hardware, an interrupt service routine, or accessed via direct memory access - DMA). It also signifies that accesses (reads or writes) to the object may have side effects beyond simply transferring data (e.g., interacting with memory-mapped hardware registers).

**2. Syntax**

The `volatile` keyword precedes the data type in a declaration or cast:

*   `volatile unsigned char hardware_register;`
*   `volatile int *status_ptr;` (A non-volatile pointer to a volatile integer)
*   `int * volatile data_buffer_ptr;` (A volatile pointer to a non-volatile integer)
*   `volatile int * volatile command_ptr;` (A volatile pointer to a volatile integer)
*   `*((volatile unsigned char *)ADDRESS)` (Casting an address to a volatile pointer and dereferencing)

**3. Effects on Compilation**

When an object is declared or accessed as `volatile`, the compiler **must** adhere to the following rules:

1.  **No Optimization of Accesses:** The compiler must not optimize away reads from or writes to the volatile object. Every explicit read or write in the C source code must correspond to a memory access instruction in the generated assembly. For example, reading a volatile variable twice in succession must generate two separate load instructions. Writing to a volatile variable, even if its value isn't subsequently read by the surrounding code, must still generate a store instruction.
2.  **No Caching in Registers:** The compiler cannot assume the value of a volatile object remains unchanged between accesses. It must not cache the value in a CPU register and reuse it; each access must interact directly with the object's memory location.
3.  **Strict Ordering:** Accesses to volatile objects must not be reordered by the compiler relative to other volatile accesses or sequence points in the program. This ensures that interactions with hardware or shared memory occur in the intended sequence.

**4. Use Cases on PB-1000 / HD61700**

The `volatile` qualifier is essential for:

*   **Accessing Memory-Mapped System Variables:** Reading from or writing to known system RAM locations (like `OUTDV` at `&h690C`, stack pointers, display buffer addresses) whose values might be read or modified by ROM routines or hardware.
*   **Interacting with Hardware Registers:** Although less common in high-level PB-1000 programming (most hardware is accessed via ROM calls), if direct hardware register access were performed, `volatile` would be mandatory.
*   **(Hypothetical) Interrupt Service Routines:** If ISRs were used, variables shared between the ISR and the main code would need to be `volatile`.

**5. Relation to `peek()` and `poke()`**

The introduction of `volatile` provides the standard C mechanism for the operations previously handled by the non-standard `peek()` and `poke()` built-ins. Direct pointer access using `volatile` is now the **recommended and standard C approach**.

```c
#define OUTDV_ADDR 0x690C
#define OUTDV_REG (*(volatile unsigned char *)OUTDV_ADDR) // Preferred way

// Writing:
OUTDV_REG = OUTDV_PRINTER;

// Reading:
unsigned char current_device = OUTDV_REG;
```

While the `peek()` and `poke()` functions might be retained for familiarity or specific use cases, their necessity is greatly diminished. Developers should prefer using standard C pointer syntax with the `volatile` qualifier for clarity, portability (within C), and adherence to standard practices. The compiler must still ensure that the *internal implementation* of `peek`/`poke` behaves as if the accesses were volatile, even if the keyword isn't visible in the `peek`/`poke` function call itself.

**6. Conclusion**

The `volatile` qualifier is a fundamental part of C for systems programming. Its inclusion allows for correct, reliable, and standard interaction with memory locations that have special properties, crucial for developing robust tools and benchmarks on the PB-1000 platform.

---

## Example: Accessing `OUTDV` with `volatile`

Here's how accessing the `OUTDV` register should look in C using `volatile` and its corresponding *optimized* assembly translation.

**C Code:**

```c
// Define address and values for clarity
#define OUTDV_ADDR       0x690C  // Output Device selector address
#define OUTDV_PRINTER    0x02    // Value for Printer output

// Define a macro for easier access (recommended practice)
#define OUTDV_REG        (*(volatile unsigned char *)OUTDV_ADDR)

void set_printer_output() {
    // Write to OUTDV using the macro (volatile access)
    OUTDV_REG = OUTDV_PRINTER;
}

unsigned char get_output_device() {
    // Read from OUTDV using the macro (volatile access)
    unsigned char current_device;
    current_device = OUTDV_REG;
    return current_device;
}

int main() {
    set_printer_output();
    unsigned char device = get_output_device();
    // ... use 'device' ...
    return 0;
}
```

**Conceptual Optimized HD61700 Assembly Translation:**

*(Assuming standard function call prologue/epilogue are handled elsewhere. Focus is on the core volatile access.)*

```assembly
; --- Translation for: OUTDV_REG = OUTDV_PRINTER; ---
; Inside set_printer_output function

    LDW   $0, &h690C     ; Load ADDRESS (&h690C) into register pair $0/$1
    LD    $2, &h02        ; Load VALUE (&h02, OUTDV_PRINTER) into register $2
    ST    $2, ($0)        ; Store VALUE from $2 into memory using register indirect.
                          ; 'volatile' ensures this ST instruction is generated and executed.

; --- Translation for: current_device = OUTDV_REG; ---
; Inside get_output_device function

    LDW   $4, &h690C     ; Load ADDRESS (&h690C) into register pair $4/$5
    LD    $6, ($4)        ; Load VALUE from memory using register indirect into $6.
                          ; 'volatile' ensures this LD instruction is generated.
    ; Assuming 'current_device' is assigned to register pair $0/$1 for return
    LD    $0, $6          ; Move result to return register $0 (low byte)
    LD    $1, 0           ; Zero high byte for 16-bit return value (unsigned char fits in low)

    ; (Function epilogue and RTN would follow)
```

**Key Points of the Assembly Translation:**

1.  **Optimized Dereferencing:** Both the read (`LD $6, ($4)`) and write (`ST $2, ($0)`) operations use the efficient **register indirect addressing mode**. They do *not* involve loading the address into `IX` or `IZ` and then using `(IX+0)` or `(IZ+0)`.
2.  **`volatile` Enforcement:** The presence of `volatile` in the C code guarantees that the compiler generates these `LD` and `ST` instructions exactly where they appear in the sequence and does not optimize them away or cache the value `&h690C` in a CPU register across accesses.

---

# Addendum: Built-in `peek()` and `poke()` Functions (Revised)

**1. Purpose**

Alongside the standard C `volatile` qualifier (which is the recommended method for accessing memory-mapped hardware and system variables), this compiler provides `peek()` and `poke()` built-in functions. These functions offer an alternative syntax that may feel more familiar or convenient to programmers accustomed to BASIC environments, simplifying direct memory reads and writes without requiring explicit pointer casting. They serve as a bridge for easily interacting with specific memory locations on the PB-1000.

**2. Functions**

The following functions are implemented by the compiler as built-ins. The compiler generates efficient, non-optimized (volatile-like) memory operations, leveraging the HD61700's register indirect addressing modes where possible. They adhere to the compiler's standard calling convention for parameters and return values.

*   **`unsigned char peek8(unsigned int address);`**
    *   **Description:** Reads a single 8-bit byte from the specified memory `address`.
    *   **Parameters:** `address` (passed in `$0/$1`).
    *   **Returns:** 8-bit value (in `$0`).
    *   **Implementation Note:** Ensures a direct memory read.
    *   **Example Assembly (Optimized, Conceptual):**
        ```assembly
        ; Built-in: unsigned char peek8(unsigned int address)
        ; Input: address in $0/$1
        ; Output: value in $0 ($1 zeroed for consistency)
        LD    $2, ($0)     ; Load byte from address specified by $0/$1 into temp $2
        LD    $0, $2       ; Move result byte to the return register $0
        LD    $1, 0        ; Zero the high byte ($1) of the return register pair
        ; (Implicit RTN if treated as a function call/inlined)
        ```

*   **`void poke8(unsigned int address, unsigned char value);`**
    *   **Description:** Writes a single 8-bit byte (`value`) to the specified memory `address`.
    *   **Parameters:** `address` (in `$0/$1`), `value` (in `$2`).
    *   **Returns:** `void`.
    *   **Implementation Note:** Ensures a direct memory write.
    *   **Example Assembly (Optimized, Conceptual):**
        ```assembly
        ; Built-in: void poke8(unsigned int address, unsigned char value)
        ; Input: address in $0/$1, value in $2
        ; Output: none
        ST    $2, ($0)     ; Store value from $2 to address specified by $0/$1
        ; (Implicit RTN if treated as a function call/inlined)
        ```

*   **`unsigned int peek16(unsigned int address);`**
    *   **Description:** Reads a 16-bit word from the specified memory `address` (little-endian).
    *   **Parameters:** `address` (in `$0/$1`).
    *   **Returns:** 16-bit value (in `$0/$1`).
    *   **Implementation Note:** Ensures direct memory reads.
    *   **Example Assembly (Optimized, Conceptual):**
        ```assembly
        ; Built-in: unsigned int peek16(unsigned int address)
        ; Input: address in $0/$1
        ; Output: value in $0/$1 ($0=low, $1=high)
        LDW   $0, ($0)     ; Load 16-bit word from address in $0/$1 directly into $0/$1
        ; (Implicit RTN if treated as a function call/inlined)
        ```

*   **`void poke16(unsigned int address, unsigned int value);`**
    *   **Description:** Writes a 16-bit word (`value`) to the specified memory `address` (little-endian).
    *   **Parameters:** `address` (in `$0/$1`), `value` (in `$2/$3`).
    *   **Returns:** `void`.
    *   **Implementation Note:** Ensures direct memory writes.
    *   **Example Assembly (Optimized, Conceptual):**
        ```assembly
        ; Built-in: void poke16(unsigned int address, unsigned int value)
        ; Input: address in $0/$1, value in $2/$3 ($2=low, $3=high)
        ; Output: none
        STW   $2, ($0)     ; Store 16-bit value from $2/$3 to address specified by $0/$1
        ; (Implicit RTN if treated as a function call/inlined)
        ```

**3. Usage Example**

```c
#define OUTDV_ADDR       0x690C
#define OUTDV_PRINTER    0x02

// Set output device to printer using poke8
poke8(OUTDV_ADDR, OUTDV_PRINTER);

// Read current setting using peek8
unsigned char current_device = peek8(OUTDV_ADDR);
```

**4. Programmer Convenience**

While direct pointer access with the `volatile` keyword (`*(volatile unsigned char *)addr = val;`) is the standard C method and preferred for its explicitness and type safety, the `peek()` and `poke()` functions provide a layer of convenience. They abstract the pointer casting and dereferencing syntax, which can be helpful for quick memory interactions or for programmers transitioning from languages like BASIC where such functions are common. Both approaches achieve the necessary non-optimized memory access.

**5. Conclusion**

The `peek`/`poke` family of built-in functions offers a convenient alternative syntax for direct memory access on the PB-1000, complementing the standard `volatile` qualifier. The compiler ensures these functions generate efficient, non-optimized memory operations internally, leveraging the HD61700's register indirect addressing modes for optimal performance.

# Addendum: Licensing

- All new source files carry a BSD-3-Clause license header.

# Addendum: Python guidelines

## Style and Formatting

1. Enforce PEP 8 (can use Black and Ruff if available but don't rely on it).
1. Semicolons or multi-statement lines are **FORBIDDEN**.
1. `snake_case` for functions/vars; `CamelCase` for classes.
1. Whenever the user asks for the ready-to-run program, you will produce completely error-free, ready to run programs according to the required specifications.  Empty files, partial implementations, stubs or starting points are **FORBIDDEN**.  If you can't comply, stop and let the user know of the problem.

## Project Layout

1. Root: `pbcc.py` + `README.md`.
1. Front-end, semantic core, lowering, IR, optimisation passes, and back-end each live in their own top-level folder under the project root (e.g. front_end/, semantic/, lowering/, ir/, passes/, back_end/).
1. Tests:  All test files under `tests/`.

## Inline Documentation

1. Generous and clear inline comments for any non-trivial logic (flag parsing, IR transforms, instruction semantics, tree traversal, decorators, etc).

## Modularity & Extensibility

1. One responsibility per module/class/function.
1. Generate beautiful, modular software design with intuitive abstractions, clean separation of concerns, DRY/KISS/SOLID principles—so that any AI agent or human developer can immediately understand, maintain, and extend the code.
