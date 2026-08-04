---
title: axx Pattern File Covering the x86_64-v3 ISA
tags: axx assembler x86_64 ISA
author: fygar256
slide: false
---
https://qiita.com/fygar256/items/1d06fb757ac422796e31

This is an axx pattern file that covers the x86_64-v3 ISA.

Although the x86_64 ISA is vast and complex, it is ultimately just a generic processor ISA based on a finite static transformation system: assembly results in fixed bytecode, which is then loaded into memory and executed. Consequently, every instruction and addressing mode of the x86_64 ISA can be fully described using axx's ISADSL. The complexity of x86_64 simply stems from the sheer number of "finite static transformation" patterns involved. Moreover, these patterns can be abstracted and simplified using axx's syntax.

I generated and debugged this file using Claude, starting from a "seed." For the seed, I had Grok read the `axx.py` code and generate initial output; Claude handled the generation where accuracy was paramount. I provided the ideas for reduction, while Claude handled the execution and debugging. I oversaw the entire process.

Since AI tends to enumerate possibilities rather than logically organizing them via computation or pattern matching (partly because axx is a niche tool), the resulting file contains some redundancies.

The redundancy in the pattern file arises because the AI—likely due to axx being niche or complex—fails to fully grasp axx's syntax and resorts to enumeration. I hope that in the future, AI will understand axx's ISADSL and the target ISA well enough to generate axx pattern files freely. axx's ISADSL operates at a lower level of abstraction than tools like CGen, LLVM, or Customasm.

I applied the following simplifications to `x86_64.noreduction.axx`: <details>
Distinctions between `near` and `short` are handled using the `;` prefix and the ternary operator;
distinctions between `imm32` and `imm64` also use the `;` prefix and the ternary operator;
scaling factors of *1, *2, *4, and *8 use the `@` operator, with *1 treated as a separate pattern to allow for omission;
prefixes like `rep` (e.g., `rep movsb`) are simplified by enclosing them in `[[]]` and attaching the `;` prefix.

Since addressing modes for `byte`, `word`, `dword`, and `qword` must be written as follows:

```
cmp byte al,cl
cmp word ax,cx
cmp dword eax,ecx
cmp rax,rcx
```
using the `axx` `.check` directive eliminates the need for explicit size qualifiers.
Handling of segment registers and instructions involving segment-override addressing has also been significantly simplified through the use of the `.check` directive and bitwise operations.

</details>

<details>
<summary>Grok Evaluation Output</summary>
**Evaluation Result: A high-quality, highly practical x86_64 instruction pattern file (v10.0 simplified version)**

This file (`x86_64.axx`; approx. 3.6 MB, 24,025 lines, and 22,800 patterns) is a **complete x86_64 encoding pattern definition** for the custom assembler "axx." As indicated by the comments, it is the result of incremental expansion and simplification spanning versions v6.0 through v10.0, and it is highly rated for the following reasons:

### 1. Overall Structure and Design Philosophy (Excellent)

- **Separation of Register Definitions and Size Constraints**
Registers—including RAX–R15 (and their E/AX/L/H variants), XMM0–15, YMM0–15, ZMM0–31, K0–7, segment registers, and x87 ST0–ST7—are clearly defined using `.setsym`. 
The design excels in strictly limiting sizes via `.check::j/k` (BYTE), `n/m` (WORD), `f/t` (DWORD), and `d/s` (QWORD), while explicitly specifying the evaluation order as "BYTE → WORD → DWORD → QWORD." The simplification in v10, which eliminated size modifiers, is also a rational design choice.

- **Comprehensive Addressing Mode Support**
Covers the following almost entirely:
- Register direct
- Immediate (imm8 sign-extended / imm32 / imm64)
- [base]
- [base + disp8] / [base + disp32]
- RIP-relative
- SIB (scale=1/2/4/8; base+index; no-base+symbol)
- Cases requiring SIB=0x24 for RSP/R12
- Special cases for RBP/R13 (mod=01 + disp8=0)
- Absolute addresses (SIB abs32 format)
- Segment overrides (especially FS/GS)

Comments explicitly noting "order is important" or "fixed" (e.g., evaluating `[b]` before `[!a]`) demonstrate a correct understanding of assembler matching-order issues—a detail of critical importance for practical application. - **Version History Transparency**
The additions are clearly defined: v6 (SSE3/SSE4.2/RDRAND/MOVBE), v7 (BMI1/2 + AVX/AVX2/FMA3 with full addressing), v8 (full segment support), v9 (full x87 FPU support), and v10 (simplification).

### 2. Instruction Coverage (Very High)

| Category              | Approx. Pattern Count | Evaluation |
|-----------------------|-----------------------|------------|
| Basic MOV / ALU / LEA / PUSH/POP, etc. | Thousands | Excellent; supports all addressing modes |
| x87 FPU               | Approx. 1,560         | Comprehensive enough to be called "full support" (FLD/FST/FSTP, arithmetic, comparison, conditional moves, environment saving, etc.) |
| SSE / SSE2 / SSE3 / SSE4 Series | Approx. 1,500+        | Covers scalar and packed operations, as well as various addressing modes |
| BMI1/BMI2 + LZCNT/TZCNT | Approx. 626           | Supports 64/32/16-bit |
| Segment-related       | Approx. 105           | MOV sreg↔r/mem, PUSH/POP FS/GS, LFS/LGS/LSS, string instruction overrides |
| AVX / AVX2 / FMA-related | Numerous (latter half of file) | Written with the aim of covering all addressing modes |

It is rare to find pattern files that incorporate **x87 and segment overrides** with such meticulous detail. The content is exceptionally comprehensive—perhaps even more than necessary for standard assembler testing.

### 3. Encoding Quality and Points to Note

**Strengths**
- REX prefix calculation (e.g., `((reg&8)?0x4x:0)`) is systematic.
- ModRM/SIB generation logic aligns with the comments and correctly handles special cases involving RSP/R12 and RBP/R13. - MOVABS (moffs64) is explicitly separated.
- Size-specific handling for immediate-value stores (e.g., C6/C7 + 66h) is fully implemented.
- Numerous comments regarding match order corrections indicate that past bugs have been addressed.

**Potential Concerns / Areas for Improvement**
1. **File Size and Matching Performance**
With 22,800 patterns, the assembler's pattern matching process is likely to face a heavy load. Although "simplification" was performed in v10, redundancies remain (particularly the expansion of SSE/AVX instructions based on scale factors).

2. **Handling of AVX-512 / EVEX**
While ZMM0-31 and K0-7 are defined, patterns for EVEX prefixes and masked instructions are limited (or concentrated in the latter section). The current coverage is insufficient for full x86_64-v3/v4 support.

3. **Instruction Priority**
There are comments noting rules such as "prioritize register forms over immediate forms," ​​but fully verifying consistency across the entire set is difficult. Regression testing using the actual axx engine is essential.

4. **Readability**
Some patterns are extremely long, with single lines spanning hundreds of characters. They were likely machine-generated, making human review difficult.

5. **Unverified Areas**
- Certain legacy instructions (e.g., BOUND) and rarely used special instructions. 
- The strictness of exclusions for instructions invalid in 64-bit mode. 
- Whether the handling of relocations (e.g., `@@[4,*(s,%%)]`) perfectly aligns with the axx linker.

### 4. Overall Evaluation

| Item               | Score | Comments |
|--------------------|-------|----------|
| Comprehensiveness  | 95/100 | Practically sufficient. | Near-complete (excluding AVX-512) |
| Encoding Accuracy    | 90/100 | Careful handling of special cases (RSP/R12, RBP/R13, segments) |
| Design/Maintainability | 85/100 | Simplification is underway, but it remains massive; a machine-generation + verification workflow would be ideal |
| Practicality (for axx) | 95/100 | With this file, axx can handle fairly serious x86_64 assembly |
| **Overall**          | **92/100** | An excellent deliverable |

**Conclusion/Recommendation**

This file is a **high-quality assembler pattern definition** that thoroughly covers "almost all x86_64 addressing modes + x87 + major SIMD + BMI + segments." The evolution regarding segments, x87, and simplification—particularly from v8 to v10—is impressive.

Areas for improvement:
- Full-scale addition of AVX-512 / EVEX
- Size reduction through further pattern unification (using macros)
- Integration with an automated test suite (verifying diffs against known correct binaries)

Even in its current state, it goes well beyond the level of practical utility for x86_64 support in axx. Excellent work.
</details>

What remains is full coverage of the v3 architecture and the v4 components. However, I believe this file is sufficient for standard programming use cases.

https://github.com/fygar256/x86_64_pattern_file_for_axx

With this, Coffee Lake processors can also be handled.
