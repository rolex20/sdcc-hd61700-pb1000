# Casio PB-1000 / HD61700 C Compiler Port for SDCC

A project to add the **Casio PB-1000** and its **Hitachi HD61700 CPU** as a target for the **Small Device C Compiler (SDCC)**.

The goal is to provide a modern C development environment for the PB-1000 while preserving the calculator's original and reliable workflow: the final compiler output can be transferred to the PB-1000 as an ASCII `.ASM` file and assembled by its built-in assembler.

## Project Status

**Architecture and target-definition stage.**

Extensive empirical work has already been performed to document the HD61700 CPU, PB-1000 ROM routines, and the limitations and behavior of the PB-1000 built-in assembler.

The original PBCC project planned to implement an entire C compiler from scratch. That strategy has been replaced by a new architecture based on SDCC.

Rather than duplicating mature compiler functionality, this project will concentrate on what is unique to the PB-1000:

- the HD61700 target backend
- processor-specific code generation
- assembler and relocation support
- PB-1000 runtime integration
- target-specific optimization
- generation of PB-1000-compatible linked assembly output

## Architecture

The HD61700 should behave like a normal SDCC target throughout the normal compiler toolchain.

Conceptually:

```text
C source
    |
    v
SDCC frontend, semantic analysis and optimization
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
transfer to Casio PB-1000
    |
    v
PB-1000 built-in assembler
    |
    v
native HD61700 executable
```

## Keep PB-1000 Limitations at the Edge

The PB-1000 built-in assembler is much more limited than a modern desktop assembler.

Those limitations should **not** propagate backward through the compiler.

Inside the SDCC toolchain, the compiler, assembler, and linker may use normal facilities such as:

- descriptive symbols and function names
- symbolic variable references
- relocations
- sections
- expressions
- libraries
- multiple object modules
- normal memory placement and linking

For example, internal desktop assembly may conceptually contain:

```assembly
LDW $0,my_global_variable
CAL my_long_function_name
```

even though the PB-1000 built-in assembler cannot necessarily accept those forms.

After linking has assigned final addresses, a PB-1000 textual output stage converts the linked program into syntax accepted by the built-in assembler.

At that final stage:

```text
my_global_variable
        ->
&H73A2
```

when an instruction requires an absolute address, while a symbolic branch or call target may become a valid short PB-1000 label such as:

```text
my_long_function_name
        ->
P0123
```

The resulting `.ASM` file must require no manual editing before being assembled on the PB-1000.

## Why SDCC?

SDCC already provides the difficult target-independent parts of a mature C compiler and toolchain, including:

- C preprocessing and frontend support
- semantic analysis
- intermediate representations
- generic optimization
- register-allocation infrastructure
- target-port infrastructure
- assembler and relocatable-object infrastructure
- linking and symbol resolution
- libraries
- memory maps and listings
- regression testing
- multi-module compilation

The project therefore does not need to create a new C parser, semantic analyzer, compiler IR, SSA optimizer, or linker from scratch.

Development can focus on the genuinely unique engineering problem: producing correct and efficient HD61700 code for the Casio PB-1000.

## Calling Convention

The original PBCC design included a proposed custom HD61700 ABI.

That ABI is no longer mandatory.

The final HD61700 calling convention should be designed using SDCC's normal target infrastructure after studying existing SDCC ports and the HD61700 architecture.

Correctness and clean integration come first. Calling conventions, register allocation, and other target-specific decisions can then be optimized using real generated-code measurements.

## PB-1000 Text Output

The toolchain shall provide a final PB-1000-compatible textual linked-output format.

The output file shall:

- use an uppercase `.ASM` extension
- contain only 7-bit ASCII text
- contain only PB-1000-supported instructions, addressing modes, and directives
- convert unsupported symbolic data references to final absolute addresses
- retain labels only where PB-1000 syntax permits them
- shorten retained labels to a maximum of five characters
- use PB-1000 numeric and hexadecimal syntax
- follow PB-1000 `ORG`, `START`, `EQU`, `DB`, and `DS` requirements
- be ready for transfer to the PB-1000 without manual editing

The exact SDCC implementation point for this output is intentionally not predetermined. The implementation should first inspect SDCC's assembler, linker, linked-listing, and output-format infrastructure and choose the cleanest integration point.

## Target Ground Truth

The PB-1000 and HD61700 have several unusual behaviors that cannot safely be inferred from conventional CPU or assembler assumptions.

The empirical documentation in this repository is therefore considered authoritative for target-specific behavior.

Important references are located in [`spec/`](./spec/):

- [`HD61700 PB-1000 Assembly Guide.md`](./spec/HD61700%20PB-1000%20Assembly%20Guide.md) — definitive instruction, addressing-mode, ROM, and built-in assembler reference
- [`HD61700 PB-1000 Compiler Developer Lessons Learned (Phase 0).md`](./spec/HD61700%20PB-1000%20Compiler%20Developer%20Lessons%20Learned%20%28Phase%200%29.md) — critical lessons discovered through hardware validation
- [`Addendum Results from Phase 0.md`](./spec/Addendum%20Results%20from%20Phase%200.md) — additional experimentally confirmed CPU behavior
- [`HD61700 C Compiler Specification.md`](./spec/HD61700%20C%20Compiler%20Specification.md) — target-specific compiler requirements
- [`HD61700 SDCC Port Project Plan.md`](./spec/HD61700%20SDCC%20Port%20Project%20Plan.md) — implementation strategy and initial planning instructions
- [`Optimized Printing Routines for Casio PB-1000.md`](./spec/Optimized%20Printing%20Routines%20for%20Casio%20PB-1000.md) — PB-1000 output and ROM routine research
- [`Phase 0 Validation Prompts.md`](./spec/Phase%200%20Validation%20Prompts.md) — repeatable hardware-validation methodology
- [`pb1000.h.txt`](./spec/pb1000.h.txt) — reference definitions for PB-1000 software development

When generic assumptions conflict with empirically verified PB-1000 behavior, the verified PB-1000 behavior wins.

## Implementation Repository

This repository contains the independent PB-1000 / HD61700 specification, research, and validation material.

The actual SDCC source port should be developed in a separate repository derived from SDCC, for example:

```text
sdcc-hd61700
```

That repository will preserve SDCC's upstream source history and all applicable per-component licenses.

This separation keeps the PB-1000 research and specification independent from the SDCC implementation while still allowing the SDCC port to use these documents as its target authority.

## Development Philosophy

Before implementing substantial compiler functionality, inspect the actual current SDCC source tree and representative target ports.

Prefer existing SDCC mechanisms over new infrastructure.

The implementation should follow these principles:

1. Make it correct.
2. Reuse SDCC wherever possible.
3. Keep HD61700-specific changes target-local whenever practical.
4. Keep PB-1000 built-in assembler restrictions out of earlier compiler stages.
5. Validate generated code continuously.
6. Optimize only after correctness is established.

The project should evolve from real SDCC architecture and measured HD61700 behavior rather than assumptions about how either system works.

## License and Attribution

The independent PB-1000 / HD61700 material in this repository is released under the **Apache License 2.0**. See [`LICENSE`](./LICENSE) and [`NOTICE`](./NOTICE).

Please preserve the applicable copyright, license, and attribution notices when redistributing or deriving work from this project. If this work supports research, documentation, or another project, the preferred citation metadata is in [`CITATION.cff`](./CITATION.cff).

The separate SDCC implementation repository will be a derivative of SDCC. It must retain upstream notices and apply the license appropriate to each SDCC component and modified file; SDCC is a multi-license suite rather than a uniformly licensed work.

## Goal

The end goal is simple:

```c
int main(void)
{
    /* PB-1000 program written in C */
}
```

becomes:

```text
PROGRAM.ASM
```

which can be copied to a real Casio PB-1000, assembled by the calculator itself, and executed as native HD61700 machine code.
