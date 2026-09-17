**Title: HD61700 C Cross Compiler Project Plan

**Phase 0: Foundational Validation & "HD61700 Behavior Reference Guide"**
*   **Objective:** Eliminate ambiguities and establish definitive ground truth for the HD61700 CPU, PB-1000 ROM routines, and assembler behavior. Create a reference document for the AI.
*   **Tasks:**
    1.  **Assembly Tests:** Write small, targeted `.ASM` programs adhering strictly to PB-1000 assembler limitations. Test:
        *   **Signed/Unsigned Comparisons:** Generate definitive truth tables for `Z` and `C` flags after `SBC`/`SBCW` (8/16-bit) covering all relations (`==`, `!=`, `<`, `<=`, `>`, `>=`) and edge cases (zero, max/min positive/negative values).
        *   **Arithmetic/Logical Flags:** Verify flag states after `AD(W)`, `SB(W)`, `AN(W)`, `OR(W)`, `XR(W)`, shifts, rotates.
        *   **Addressing Modes:** Confirm behavior and limitations of register indirect `($n)` and indexed `(IX/IZ+/-offset)`.
        *   **ROM Call Behavior:** Call selected ROM routines (especially those used for built-ins like `OUTAC`, `PRLB1`, `KBM16`, `BXWY`) and meticulously document register inputs, outputs, *and specifically which registers are altered ("clobbered")*.
    2.  **Sample Program Validation:** Assemble and run the provided N-Queens, Renumber, and Sort `.ASM` files on emulator/hardware to confirm toolchain and understanding.
    3.  **Built-in Function Templates:** Write and *manually test* the `.ASM` templates for simple built-ins (`abs`, `min`, `max`, `memcpy`, `memset`, clock functions, `put_char`) adhering to the specified ABI. Validate inputs, outputs, and register usage.
    4.  **Signed Modulo Strategy:** Decide, implement (if manual), and test the approach for signed `%`.
    5.  **Documentation:** Compile all findings into a concise **"HD61700 Behavior Reference Guide"**. *This guide is critical input for the AI in subsequent phases.*
*   **Output:** Validated `.ASM` test snippets, validated `.ASM` built-in templates, Signed Modulo solution, and the crucial "HD61700 Behavior Reference Guide".

**Phase 1: Front-End Foundation (Lexer, Parser, AST, Semantics)**
*   **Objective:** Develop and validate the front-end components: hand-written Lexer, hand-written Parser, AST, Symbol Table, and basic Semantic Analysis.
*   **Tasks:**
    1.  **C Test Suite:** Create comprehensive C test programs covering:
        *   **Positive Cases:** All specified C features (data types, control structures, structs, enums, pointers, `volatile`, built-in calls, recursion, `print()`, literals, `asm()`, `typedef`).
        *   **Negative Cases:** Syntax errors, type mismatches, use of unsupported features (float, function pointers, etc.).
    2.  **Hand-written Lexer (Tokenizer):**
        *   Implement a deterministic tokenizer in Python (no parser generators).
        *   Produce tokens with filename/line/column spans for high-quality diagnostics.
        *   Support PB-1000 literals and extensions (e.g., `&H...` hex, binary literals if specified, character and string literals, escape handling).
    3.  **Hand-written Parser & AST Generation:**
        *   Implement a recursive-descent parser for declarations, statements, and translation units.
        *   Use a Pratt / precedence-climbing expression parser to handle C operator precedence and associativity.
        *   Build a clean, testable AST with explicit node types and source locations.
    4.  **Symbol Table & Semantic Analysis:** Implement symbol table management (scopes, types, variables, functions) and core semantic checks (type checking, declaration errors, `const` enforcement, `volatile` usage checks).
    5.  **Table-driven `UnsupportedFeatureChecker` pass emits an *immediate error* for floats, long long, function pointers, etc.  Deleting a table entry re-enables the feature.**
    6.  **Testing & Refinement:** Use the C test suite to thoroughly test the lexer, parser, and semantic analyzer. Utilize compiler flags (`-tokens`, `-ast`, `-symboltable`, `-errors`) for debugging. Ensure all valid programs parse correctly and semantic errors are caught. Unit tests use *pytest*; test files live under `tests/`.
*   **Output:** Python code for lexing, parsing, AST generation, symbol table, and semantic analysis. Tested C programs.
**Phase 2: Intermediate Representation (IR) Design & Generation**
*   **Objective:** Define and implement the TAC/SSA-based IR pipeline as specified in the IR Addendum.
*   **Tasks:**
    1.  **IR Design:** Formalize the structure of the Three-Address Code (TAC) representation.
    2.  **AST -> TAC Conversion:** Implement the logic to translate the validated AST (with semantic information) into the TAC IR.
    3.  **IR Validation:** Enhance the C test suite or add new tests specifically to verify the correctness of the generated TAC. Implement the `-ir` flag for dumping the TAC.
*   **Output:** Python code for TAC generation, refined C test suite.

**Phase 3: Basic Code Generation (`-O0`) & Built-in Integration**
*   **Objective:** Create a functional compiler that generates correct, but unoptimized HD61700 assembly with with a simple, stack-based code generator (naive codegen). Integrate validated built-ins.
*   **Tasks:**
    1.  **TAC -> Assembly Emitter:** Implement the core back-end to translate TAC instructions into HD61700 assembly text.
    2.  **Code Generation Logic:**
        *   Emit code for assignments, arithmetic, logical operations, and control flow, using the validated flag logic from the "Reference Guide".
        *   Implement the full function calling convention (prologue/epilogue, parameter passing via regs/stack, shadow space, return values, using `IX` as FP).
        *   Handle memory addressing (globals via `IZ` or literals, locals via `IX`, pointers via `($n)` or `IX/IZ`, `volatile`).
        *   Generate correct `DS`/`DB` definitions respecting `ORG`/`START` order and implement *explicit zero-initialization* for static/global variables.
        *   Implement `asm("...")` pass-through.
        *   Implement the `Pxxxx` label generation scheme and `-comments` flag.
    3.  **Built-in Integration:**
        *   Implement logic to selectively *append* the validated `.ASM` templates (from Phase 0) only when their corresponding C functions are used.
        *   Implement direct code generation for complex built-ins like `print()` (using ROM calls like `PRLB1`, `WRDBCD`, `OUTAC`, `OUTCR` as per spec).
    4.  **Testing:** Compile the C test suite to `.ASM`. First, assemble using a fast cross-assembler (if available) for quick syntax checks, then use the PB-1000 built-in assembler. Test the resulting executables thoroughly on emulator/hardware. Debug generated assembly using `-labels`, `-dump`. Focus 100% on correctness.
*   **Output:** A working `-O0` compiler generating correct assembly.

**Phase 4: SSA & Optimization Framework**
*   **Objective:** Implement the SSA infrastructure and the framework for testing optimizations.
*   **Tasks:**
    1.  **SSA Conversion:** Implement TAC -> SSA conversion (including Phi function insertion).
    2.  **SSA Deconstruction:** Implement SSA -> TAC conversion (Phi function elimination).
    3.  **Optimization Test Framework:** Develop scripts/tools to:
        *   Compile the C test suite with different optimization levels (`-O0` vs `-Ox`).
        *   Assemble the outputs.
        *   Run executables (emulator).
        *   Compare outputs/behavior for functional correctness.
        *   Measure code size and optionally estimate cycle counts.
*   **Output:** SSA conversion logic, optimization testing framework.

**Phase 5: Optimization Pass Implementation (Iterative)**
*   **Objective:** Implement and validate the specified optimization passes.
*   **Tasks:**
    1.  **Implement Passes Individually:** Implement optimizations one by one, following the detailed ranking/grouping provided in the C compiler specification. Process the IR *in SSA form* for relevant passes.
    2.  **Test Each Pass:** After implementing *each* optimization, use the test framework (Phase 4) to:
        *   Verify functional correctness (output must match `-O0`).
        *   Measure impact on size/speed.
        *   Debug aggressively using `-cfg`, `-dfg`, and other dumps.
    3.  **Integrate `-O` Flags:** Once individual passes are verified, implement the logic for the high-level flags (`-O1`, `-O2`, `-O3`, `-Osize`, `-Ospeed`) to invoke the correct sequences of passes.
*   **Output:** Compiler with verified optimization passes and `-O` flag logic.

**Phase 6: Final Validation, Refinement & Documentation**
*   **Objective:** Ensure overall robustness, usability, and produce final deliverables.
*   **Tasks:**
    1.  **Edge Case Testing:** Test scenarios identified (recursion depth, `volatile` interaction, memory boundary conditions).
    2.  **Error Reporting:** Review and enhance compiler error messages for clarity.
    3.  **Documentation:** Write the comprehensive user manual.
    4.  **Final Comprehensive Testing:** Run a wide variety of C programs through the entire toolchain.
    5.  **Packaging:** Prepare the compiler distribution.
*   **Output:** Final compiler, user manual, test suite.

