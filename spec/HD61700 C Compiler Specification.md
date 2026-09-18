# HD61700 / PB-1000 Target Specification for SDCC

# Target-Specific Requirements for an SDCC Port to the Hitachi HD61700 CPU and Casio PB-1000

## Purpose and Implementation Strategy

This document defines the target-specific requirements for adding the Hitachi HD61700 CPU and Casio PB-1000 as a new target for the Small Device C Compiler (SDCC).

The implementation shall be developed as a normal SDCC target port and shall reuse SDCC's existing infrastructure, functionality, and normal toolchain behavior as extensively as possible.

Do not reimplement functionality already provided by SDCC unless a specific HD61700 or PB-1000 requirement makes doing so necessary.

In particular, the implementation should leverage SDCC's existing:

- C frontend and preprocessor
- language support and semantic analysis
- intermediate representations
- generic optimization infrastructure
- target-port interfaces
- register-allocation infrastructure
- calling-convention infrastructure
- assembler infrastructure
- relocatable object format
- linker and relocation infrastructure
- section and memory placement
- libraries and multi-module compilation
- symbol and memory-map generation
- listing and debugging infrastructure
- optimization options
- regression-test infrastructure
- normal executable, binary, or load-image outputs

The HD61700 target shall behave like a normal SDCC target internally.

The compiler, its intermediate representations, generated desktop assembly, relocatable object files, and linker shall not be artificially restricted by limitations of the PB-1000 built-in assembler.

Internally, normal symbolic names, arbitrary-length function and variable names, relocations, symbolic memory references, sections, expressions, and other useful SDCC assembler/linker features may be used.

PB-1000 assembler restrictions shall be applied only when producing the final PB-1000 textual linked-output format described below.

The PB-1000 / HD61700 documents in this repository provide the authoritative target-specific hardware and assembler behavior. Empirically validated behavior takes precedence over assumptions based on other CPUs, assemblers, or compiler targets.

## Authoritative Target References

This specification states the integration requirements. Detailed hardware facts, PB-1000 assembler rules, ROM entry points, and experimental results remain in the focused references:

- `HD61700 PB-1000 Assembly Guide.md`
- `HD61700 PB-1000 Compiler Developer Lessons Learned (Phase 0).md`
- `Addendum Results from Phase 0.md`
- `Optimized Printing Routines for Casio PB-1000.md`
- `Phase 0 Validation Prompts.md`
- `pb1000.h.txt`

Do not copy large hardware tables or instruction descriptions into this specification. Correct discrepancies at their authoritative source instead.

## C Language Support

The HD61700 target should expose the C language functionality normally provided by the version of SDCC on which the port is based.

Do not create or maintain a separate PB-1000-specific C grammar, preprocessor, parser, type system, or semantic analyzer unless a target-specific extension provides sufficient benefit to justify it.

Where a generic SDCC language feature requires target-specific runtime support, implement that support using normal SDCC target mechanisms where practical.

Target limitations should be introduced only when required by the HD61700 architecture, available PB-1000 memory, missing runtime support, or another demonstrated technical constraint.

The initial implementation may bring features online incrementally, but the architecture should not unnecessarily prevent SDCC-supported features from being implemented later.

C source shall use normal C hexadecimal syntax such as `0x6000`. PB-1000 hexadecimal syntax such as `&H6000` belongs only in PB-1000 assembly and final textual output.

## Target Data Model and Memory

The initial target model should reflect the native HD61700 and PB-1000 environment:

- 8-bit bytes and `char`
- efficient 8-bit and 16-bit integer operations
- a 16-bit flat address space for normal PB-1000 pointers
- no artificial alignment requirement where the hardware and SDCC internals permit byte alignment
- direct access to PB-1000 memory-mapped system data through standard C pointer operations

Exact C type sizes, signedness defaults, alignment rules, address spaces, and memory models shall be defined through normal SDCC port mechanisms after inspecting current SDCC requirements. They shall be recorded as stable ABI decisions and covered by regression tests.

Floating-point, wider integers, variadic functions, structures, unions, bit-fields, recursion, and other generic language features shall not be forbidden merely because the first implementation does not yet support their target runtime or code generation. Unsupported capabilities should be tracked as implementation status, not permanent language restrictions, unless a demonstrated architectural constraint requires otherwise.

Access to memory-mapped state must respect standard C `volatile` semantics. The port must not remove, combine, cache, or incorrectly reorder volatile accesses. Convenience APIs such as `peek8`, `poke8`, `peek16`, and `poke16` may be supplied in a target library or header, but standard volatile pointer access remains the primary language mechanism.

## HD61700 Calling Convention and ABI

The original PBCC project proposed a detailed HD61700 calling convention before the decision was made to base the compiler on SDCC.

That historical design may be consulted as architectural research, but it is not a mandatory ABI for the SDCC port.

The HD61700 port shall use SDCC's normal target-port and calling-convention infrastructure.

Before defining the final HD61700 calling convention, the implementation should inspect representative SDCC targets and determine the convention that best fits both SDCC's architecture and the HD61700 CPU.

Calling-convention decisions should prioritize:

- correctness
- reuse of SDCC infrastructure
- implementation simplicity
- efficient use of HD61700 registers and stacks
- code size
- execution performance

The initial convention may deliberately favor simplicity and correctness. It may later be optimized based on generated-code measurements and regression testing.

Any ABI decision that affects interoperability with hand-written HD61700 assembly shall be documented once the convention becomes stable.

In particular, register arguments, return registers, caller- and callee-saved registers, use of the user and system stacks, frame-pointer policy, stack direction, stack cleanup, variadic calls, and ROM-call wrappers must be derived and tested rather than inherited as mandatory rules from the former PBCC proposal.

## Compiler Intermediate Representations and Generic Optimization

Use SDCC's existing intermediate representations, data-flow infrastructure, and generic optimization passes.

The PB-1000 project shall not define a parallel TAC, SSA, or other general-purpose compiler IR merely for this target.

Target-specific transformations should be implemented through the normal SDCC target mechanisms at the most appropriate existing stage.

Before introducing any new compiler-wide representation or pass, first verify that the required functionality cannot reasonably be implemented using existing SDCC infrastructure.

## Optimization

Use SDCC's existing optimization framework and user-visible optimization options wherever possible.

HD61700-specific optimization should focus on target-dependent decisions such as:

- instruction selection
- register and register-pair allocation
- stack/frame handling
- efficient 8-bit versus 16-bit operations
- HD61700 comparison sequences
- branch generation
- target-specific peephole optimization
- use of PB-1000 ROM routines where beneficial
- code-size versus execution-speed tradeoffs

Do not duplicate generic optimizations already performed by SDCC.

Correctness and regression coverage take priority over optimization. Instruction timing and code-size assumptions should be measured against generated programs and verified target behavior.

## Assembly and Runtime Integration

The port shall use normal SDCC mechanisms for inline assembly and separately assembled modules. It shall not add a PB-1000-specific `asm` grammar if SDCC's existing inline-assembly facilities are sufficient.

The desktop assembler must accept the HD61700 instructions and addressing modes required for compiler output and hand-written target modules. It should support symbolic expressions, relocations, sections, arbitrary-length internal symbols, and the object format required by the SDCC linker, even where the PB-1000 built-in assembler cannot.

Target runtime support may use verified PB-1000 ROM routines when they improve correctness, code size, or performance. Candidate facilities include character and string output, screen clearing, integer conversion, clock access, memory copying, and memory filling. Each wrapper must document and test its inputs, outputs, clobbered registers, memory effects, and dependence on PB-1000 state.

Printing support should be exposed through ordinary C headers and target-library functions wherever practical. It shall not require a special parser-only `print` statement. Optimized routines and verified ROM calls described in `Optimized Printing Routines for Casio PB-1000.md` remain implementation references.

Target headers should evolve from `pb1000.h.txt` and use standard C literals and declarations. Fixed addresses and other empirically established facts should have one canonical definition where practical.

## Sections, Placement, and Startup

Code, initialized data, read-only data, uninitialized data, stack space, and any PB-1000-specific regions shall be represented through SDCC's normal section and linker mechanisms wherever possible.

Program origins and memory placement shall be linker concerns rather than a one-shot parser directive that disables relocation. If a target-specific placement option or pragma is needed, it should compose with normal multi-module linking.

Startup code shall establish the machine state required by the selected memory model and stable ABI. Any required interaction with the PB-1000 BASIC environment, built-in assembler, system stack, user stack, display state, or ROM shall be explicitly documented and tested.

## PB-1000 Textual Linked-Output Format

The compiler and desktop toolchain shall carry normal SDCC symbolic information and relocation capabilities through the normal assembler and linker pipeline.

Do not impose PB-1000 built-in assembler limitations on earlier compiler stages.

For example, intermediate/generated desktop assembly may use ordinary symbolic expressions such as:

```assembly
LDW $0,my_global_variable
CAL my_long_function_name
```

even when those forms cannot be assembled directly by the PB-1000.

After normal linking and relocation have assigned final addresses to code, functions, variables, constants, and other objects, the toolchain shall provide a PB-1000 textual linked-output format.

This output shall create a file whose filename extension is `.ASM` in uppercase and whose contents can be transferred directly to a Casio PB-1000 and assembled by its built-in assembler without manual editing.

Only at this final linked-output stage shall PB-1000 assembler limitations be applied.

The output stage shall:

- use only instructions and addressing modes supported by the PB-1000 built-in assembler
- use only supported PB-1000 directives and required source ordering
- convert symbolic data references into final absolute numeric addresses where PB-1000 instructions cannot use labels
- retain symbolic labels only where supported by the PB-1000 assembler
- translate retained labels into unique valid labels no longer than five characters
- translate internal function and variable symbols to short labels or final numeric addresses according to PB-1000 instruction syntax
- emit final numeric hexadecimal values in PB-1000 assembler syntax
- obey all `ORG`, `START`, `EQU`, `DB`, and `DS` requirements documented in the PB-1000 Assembly Guide
- contain only 7-bit ASCII text
- preserve correct program semantics after symbol and address translation

The label mapper may use any deterministic scheme that produces unique valid PB-1000 labels. The former prefix-plus-base-36 proposal is historical guidance, not a required naming ABI.

The PB-1000 textual output is logically produced after final address resolution. The implementation is not required to reside literally inside the linker if another SDCC-integrated representation or output stage provides a cleaner implementation.

The AI implementing the port shall inspect SDCC's existing assembler, linker, linked-listing, map, and output-format mechanisms before deciding where this functionality should be implemented.

## Normal SDCC Linked Output

The PB-1000 `.ASM` output is an additional target output and does not need to replace normal SDCC linked output.

If the normal HD61700 toolchain can also produce binary, Intel HEX, Motorola S-record, or another useful executable/load image, retain that functionality.

Such outputs may later be useful for emulators, automated testing, direct loading, disassembly comparison, and debugging.

The primary initial physical-PB-1000 workflow remains:

```text
C source
    ->
SDCC HD61700 target
    ->
normal assembler and relocation
    ->
SDCC linker
    ->
fully resolved linked program
    ->
PB-1000 textual output
    ->
PROGRAM.ASM
    ->
transfer to PB-1000
    ->
PB-1000 built-in assembler
    ->
native executable
```

## Diagnostics and Testing

Use SDCC's normal diagnostics and regression-test infrastructure. Target-specific diagnostics should be added only for real HD61700 or PB-1000 constraints and should explain the offending construct and practical remedy.

Regression coverage should include, as the implementation matures:

- target data-model and ABI invariants
- instruction selection for 8-bit and 16-bit operations
- signed and unsigned comparisons and condition-code behavior
- register-pair and stack behavior
- volatile memory access
- calls, returns, recursion, and variadic behavior when supported
- relocations, sections, and multi-module linking
- ROM-call wrappers and their clobber sets
- PB-1000 label and absolute-address conversion
- exact syntax, ordering, character set, and repeatability of final `.ASM` output
- execution in an accurate emulator when available
- representative execution on physical PB-1000 hardware

Physical PB-1000 behavior or a demonstrably accurate emulator is the final authority for target behavior. New empirical findings should update the focused ground-truth documents and gain regression tests where automation is possible.

## Licensing Scope

This independent specification and research repository is licensed under the repository's Apache License 2.0.

A separate repository derived from SDCC must preserve SDCC's upstream copyright and license notices and follow the license applicable to each component and file. New backend code placed within that derivative should use a license compatible with the surrounding SDCC component and the project's contribution requirements.

Citation metadata and attribution requests do not replace, restrict, or alter the applicable software licenses.
