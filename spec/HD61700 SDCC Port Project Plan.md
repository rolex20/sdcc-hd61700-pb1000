# HD61700 / PB-1000 SDCC Port Project Plan

## Objective

Add the Hitachi HD61700 CPU and Casio PB-1000 as a high-quality target for SDCC while reusing the existing SDCC compiler, assembler, linker, library, optimization, and regression-test infrastructure as extensively as possible.

The project shall generate normal SDCC linked outputs where practical and additionally generate a final PB-1000-compatible `.ASM` text file that can be transferred directly to the PB-1000 and assembled by its built-in assembler.

## Development Principle

Do not design or implement a second compiler architecture beside SDCC.

Before adding target-specific infrastructure, inspect the current SDCC implementation and determine whether the required capability already exists or can be extended cleanly.

The HD61700 should behave like a normal SDCC target throughout the compiler, assembler, and linker pipeline.

The unusual limitations of the PB-1000 built-in assembler should be isolated to the final PB-1000 textual linked-output stage.

## Authoritative Target References

The following project documents provide the authoritative HD61700/PB-1000 target information:

- `HD61700 PB-1000 Assembly Guide.md`
- `HD61700 PB-1000 Compiler Developer Lessons Learned (Phase 0).md`
- `Addendum Results from Phase 0.md`
- `Optimized Printing Routines for Casio PB-1000.md`
- `Phase 0 Validation Prompts.md`
- `pb1000.h.txt`

Empirically validated PB-1000 behavior takes precedence over assumptions derived from other CPUs or other compiler targets.

## Initial AI Planning Task

Before making substantial implementation changes, the AI development model shall inspect the actual current SDCC source tree.

It shall study representative existing SDCC targets and identify which ports provide the closest reusable patterns for:

- register architecture
- 8-bit and 16-bit operations
- unusual or irregular register sets
- stack handling
- calling conventions
- assembler integration
- relocatable object generation
- linker integration
- memory and section placement
- linked listing generation
- target libraries
- regression testing
- target-specific optimization

The AI shall then produce an implementation plan based on the actual SDCC architecture it finds.

The plan shall identify:

1. which existing SDCC components can be reused unchanged
2. which existing target ports should be used as implementation references
3. which new HD61700-specific files are required
4. the minimum target-independent SDCC changes required
5. how HD61700 assembly and relocations should integrate with SDCC's assembler/linker infrastructure
6. how normal linked executable output will be produced
7. the cleanest implementation point for the PB-1000 textual linked output
8. the regression strategy for validating generated HD61700 code
9. the incremental milestones required to reach a working compiler
10. optimization work to defer until correctness has been established

Do not predetermine implementation stages merely because they appeared in the original from-scratch PBCC project plan.

Let the actual SDCC source architecture determine the detailed implementation plan.

## Required End State

The desired logical toolchain is:

```text
C source
    |
    v
SDCC frontend / optimizer
    |
    v
HD61700 target backend
    |
    v
symbolic HD61700 desktop assembly
    |
    v
SDCC-compatible HD61700 assembler
    |
    v
relocatable object modules
    |
    v
SDCC linker
    |
    +--> normal linked/binary output where practical
    |
    v
PB-1000 textual linked-output format
    |
    v
PROGRAM.ASM
    |
    v
Casio PB-1000 built-in assembler
    |
    v
native HD61700 executable
```

Within the desktop SDCC toolchain, normal symbolic names, relocations, sections, and linker functionality should be preserved.

Only the final PB-1000 textual output needs to satisfy the limitations of the PB-1000 built-in assembler.

## Development Priorities

The priority order is:

1. Correctness.
2. Maximum reuse of SDCC infrastructure.
3. Automated regression coverage.
4. Clean isolation of PB-1000 assembler limitations.
5. Maintainability and potential future upstreaming.
6. Code-size and performance optimization.

Optimization must not precede basic correctness.

Every target-specific behavior that can be tested automatically should eventually have a regression test.

Physical PB-1000 execution or a sufficiently accurate emulator remains the final authority when validating target behavior.
