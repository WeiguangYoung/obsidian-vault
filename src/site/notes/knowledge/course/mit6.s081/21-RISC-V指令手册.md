---
{"dg-publish":true,"permalink":"/knowledge/course/mit6.s081/21-RISC-V指令手册/","title":"6.S081 · RISC-V 指令手册","tags":["MIT6.1810","xv6","操作系统","RISC-V","指令集","参考手册"],"dg-note-properties":{"title":"6.S081 · RISC-V 指令手册","tags":["MIT6.1810","xv6","操作系统","RISC-V","指令集","参考手册"],"date":"2026-11-18"}}
---


# 6.S081 · RISC-V 指令手册

> The RISC-V Instruction Set Manual（合订本）
> 卷 I 用户级 ISA · Version 2.2 ／ 卷 II 特权架构 · Version 1.12-draft

返回 [[knowledge/course/mit6.s081/index\|📚 6.S081 目录]] · [[knowledge/course/index\|📚 课程资源]] · [[首页\|🏠 首页]]

---

> **说明**：本文由官方 PDF 逐页自动转换，**未改写任何内容**。指令编码、CSR 地址、位域定义与原文一致（十六进制常量 265 处逐一核对无差异）。
> `<!-- p.N -->` 为原 PDF 页码锚点，便于对照原文。
> 表格与位域图以等宽代码块呈现，以保住对齐。

---

<!-- p.1 -->

        The RISC-V Instruction Set Manual
                  Volume I: User-Level ISA
                     Document Version 2.2

          Editors: Andrew Waterman1 , Krste Asanović1,2

                          1
                            SiFive Inc.,
2
  CS Division, EECS Department, University of California, Berkeley
            andrew@sifive.com, krste@berkeley.edu
                          May 7, 2017 Contributors to all versions of the spec in alphabetical order (please contact editors to suggest
corrections): Krste Asanović, Rimas Avižienis, Jacob Bachmeyer, Christopher F. Batten, Allen
J. Baum, Alex Bradbury, Scott Beamer, Preston Briggs, Christopher Celio, David Chisnall, Paul
Clayton, Palmer Dabbelt, Stefan Freudenberger, Jan Gray, Michael Hamburg, John Hauser, David
Horner, Olof Johansson, Ben Keller, Yunsup Lee, Joseph Myers, Rishiyur Nikhil, Stefan O’Rear,
Albert Ou, John Ousterhout, David Patterson, Colin Schmidt, Michael Taylor, Wesley Terpstra,
Matt Thomas, Tommy Thorn, Ray VanDeWalker, Megan Wachs, Andrew Waterman, Robert Watson, and Reinoud Zandijk.

<!-- p.2 -->

This document is released under a Creative Commons Attribution 4.0 International License.

This document is a derivative of “The RISC-V Instruction Set Manual, Volume I: User-Level ISA
Version 2.1” released under the following license: c 2010–2017 Andrew Waterman, Yunsup Lee,
David Patterson, Krste Asanović. Creative Commons Attribution 4.0 International License.

Please cite as: “The RISC-V Instruction Set Manual, Volume I: User-Level ISA, Document Version
2.2”, Editors Andrew Waterman and Krste Asanović, RISC-V Foundation, May 2017.

<!-- p.3 -->

## Preface

This is version 2.2 of the document describing the RISC-V user-level architecture. The document
contains the following versions of the RISC-V ISA modules:

```text
                                    Base       Version   Frozen?
                                   RV32I       2.0          Y
                                   RV32E       1.9          N
                                   RV64I       2.0          Y
                                   RV128I      1.7          N
                                  Extension    Version   Frozen?
                                     M         2.0          Y
                                     A         2.0          Y
                                      F        2.0          Y
                                     D         2.0          Y
                                     Q         2.0          Y
                                      L        0.0          N
                                     C         2.0          Y
                                     B         0.0          N
                                      J        0.0          N
                                     T         0.0          N
                                      P        0.1          N
                                     V         0.2          N
                                     N         1.1          N
```

To date, no parts of the standard have been officially ratified by the RISC-V Foundation, but
the components labeled “frozen” above are not expected to change during the ratification process
beyond resolving ambiguities and holes in the specification.

The major changes in this version of the document include:

- The previous version of this document was released under a Creative Commons Attribution 4.0 International Licence by the original authors, and this and future versions of this document will be released under the same licence.
- Rearranged chapters to put all extensions first in canonical order.
- Improvements to the description and commentary.
- Modified implicit hinting suggestion on JALR to support more efficient macro-op fusion of LUI/JALR and AUIPC/JALR pairs.

                                                 i •   Clarification of constraints on load-reserved/store-conditional sequences.
- A new table of control and status register (CSR) mappings.
- Clarified purpose and behavior of high-order bits of fcsr.
- Corrected the description of the FNMADD.fmt and FNMSUB.fmt instructions, which had suggested the incorrect sign of a zero result.
- Instructions FMV.S.X and FMV.X.S were renamed to FMV.W.X and FMV.X.W respectively to be more consistent with their semantics, which did not change. The old names will continue to be supported in the tools.
- Specified behavior of narrower (<FLEN) floating-point values held in wider f registers using NaN-boxing model.
- Defined the exception behavior of FMA(∞, 0, qNaN).
- Added note indicating that the P extension might be reworked into an integer packed-SIMD proposal for fixed-point operations using the integer registers.
- A draft proposal of the V vector instruction set extension.
- An early draft proposal of the N user-level traps extension.
- An expanded pseudoinstruction listing.
- Removal of the calling convention chapter, which has been superseded by the RISC-V ELF psABI Specification [1].
- The C extension has been frozen and renumbered version 2.0.

<!-- p.4 -->

Preface to Document Version 2.1

This is version 2.1 of the document describing the RISC-V user-level architecture. Note the frozen
user-level ISA base and extensions IMAFDQ version 2.0 have not changed from the previous version
of this document [36], but some specification holes have been fixed and the documentation has been
improved. Some changes have been made to the software conventions.

- Numerous additions and improvements to the commentary sections.
- Separate version numbers for each chapter.
- Modification to long instruction encodings >64 bits to avoid moving the rd specifier in very long instruction formats.
- CSR instructions are now described in the base integer format where the counter registers are introduced, as opposed to only being introduced later in the floating-point section (and the companion privileged architecture manual).
- The SCALL and SBREAK instructions have been renamed to ECALL and EBREAK, respectively. Their encoding and functionality are unchanged.
- Clarification of floating-point NaN handling, and a new canonical NaN value.
- Clarification of values returned by floating-point to integer conversions that overflow.
- Clarification of LR/SC allowed successes and required failures, including use of compressed instructions in the sequence.
- A new RV32E base ISA proposal for reduced integer register counts, supports MAC extensions.
- A revised calling convention.
- Relaxed stack alignment for soft-float calling convention, and description of the RV32E calling convention.
- A revised proposal for the C compressed extension, version 1.9.

<!-- p.5 -->

Preface to Version 2.0

This is the second release of the user ISA specification, and we intend the specification of the
base user ISA plus general extensions (i.e., IMAFD) to remain fixed for future development. The
following changes have been made since Version 1.0 [35] of this ISA specification.

- The ISA has been divided into an integer base with several standard extensions.
- The instruction formats have been rearranged to make immediate encoding more efficient.
- The base ISA has been defined to have a little-endian memory system, with big-endian or bi-endian as non-standard variants.
- Load-Reserved/Store-Conditional (LR/SC) instructions have been added in the atomic instruction extension.
- AMOs and LR/SC can support the release consistency model.
- The FENCE instruction provides finer-grain memory and I/O orderings.
- An AMO for fetch-and-XOR (AMOXOR) has been added, and the encoding for AMOSWAP has been changed to make room.
- The AUIPC instruction, which adds a 20-bit upper immediate to the PC, replaces the RDNPC instruction, which only read the current PC value. This results in significant savings for position-independent code.
- The JAL instruction has now moved to the U-Type format with an explicit destination register, and the J instruction has been dropped being replaced by JAL with rd=x0. This removes the only instruction with an implicit destination register and removes the J-Type instruction format from the base ISA. There is an accompanying reduction in JAL reach, but a significant reduction in base ISA complexity.
- The static hints on the JALR instruction have been dropped. The hints are redundant with the rd and rs1 register specifiers for code compliant with the standard calling convention.
- The JALR instruction now clears the lowest bit of the calculated target address, to simplify hardware and to allow auxiliary information to be stored in function pointers.
- The MFTX.S and MFTX.D instructions have been renamed to FMV.X.S and FMV.X.D, respectively. Similarly, MXTF.S and MXTF.D instructions have been renamed to FMV.S.X and FMV.D.X, respectively.
- The MFFSR and MTFSR instructions have been renamed to FRCSR and FSCSR, respectively. FRRM, FSRM, FRFLAGS, and FSFLAGS instructions have been added to individually access the rounding mode and exception flags subfields of the fcsr.
- The FMV.X.S and FMV.X.D instructions now source their operands from rs1, instead of rs2. This change simplifies datapath design.
- FCLASS.S and FCLASS.D floating-point classify instructions have been added.
- A simpler NaN generation and propagation scheme has been adopted.
- For RV32I, the system performance counters have been extended to 64-bits wide, with separate read access to the upper and lower 32 bits.
- Canonical NOP and MV encodings have been defined.

<!-- p.6 -->

- Standard instruction-length encodings have been defined for 48-bit, 64-bit, and >64-bit instructions.
- Description of a 128-bit address space variant, RV128, has been added.
- Major opcodes in the 32-bit base instruction format have been allocated for user-defined custom extensions.
- A typographical error that suggested that stores source their data from rd has been corrected to refer to rs2.

<!-- p.13 -->

## 第一卷 · 用户级 ISA

> Volume I: RISC-V User-Level ISA V2.2

### 第 1 章 Introduction

RISC-V (pronounced “risk-five”) is a new instruction set architecture (ISA) that was originally
designed to support computer architecture research and education, but which we now hope will
also become a standard free and open architecture for industry implementations. Our goals in
defining RISC-V include:

- A completely open ISA that is freely available to academia and industry.
- A real ISA suitable for direct native hardware implementation, not just simulation or binary translation.
- An ISA that avoids “over-architecting” for a particular microarchitecture style (e.g., microcoded, in-order, decoupled, out-of-order) or implementation technology (e.g., full-custom, ASIC, FPGA), but which allows efficient implementation in any of these.
- An ISA separated into a small base integer ISA, usable by itself as a base for customized accelerators or for educational purposes, and optional standard extensions, to support generalpurpose software development.
- Support for the revised 2008 IEEE-754 floating-point standard [14].
- An ISA supporting extensive user-level ISA extensions and specialized variants.
- Both 32-bit and 64-bit address space variants for applications, operating system kernels, and hardware implementations.
- An ISA with support for highly-parallel multicore or manycore implementations, including heterogeneous multiprocessors.
- Optional variable-length instructions to both expand available instruction encoding space and to support an optional dense instruction encoding for improved performance, static code size, and energy efficiency.
- A fully virtualizable ISA to ease hypervisor development.
- An ISA that simplifies experiments with new supervisor-level and hypervisor-level ISA designs.

     Commentary on our design decisions is formatted as in this paragraph, and can be skipped if the
     reader is only interested in the specification itself.

     The name RISC-V was chosen to represent the fifth major RISC ISA design from UC Berkeley
     (RISC-I [23], RISC-II [15], SOAR [32], and SPUR [18] were the first four). We also pun on the use of the Roman numeral “V” to signify “variations” and “vectors”, as support for a range of
    architecture research, including various data-parallel accelerators, is an explicit goal of the ISA
    design.

<!-- p.14 -->

    We developed RISC-V to support our own needs in research and education, where our group is
    particularly interested in actual hardware implementations of research ideas (we have completed
    eleven different silicon fabrications of RISC-V since the first edition of this specification), and
    in providing real implementations for students to explore in classes (RISC-V processor RTL de-
    signs have been used in multiple undergraduate and graduate classes at Berkeley). In our current
    research, we are especially interested in the move towards specialized and heterogeneous accel-
    erators, driven by the power constraints imposed by the end of conventional transistor scaling.
    We wanted a highly flexible and extensible base ISA around which to build our research effort.
        A question we have been repeatedly asked is “Why develop a new ISA?” The biggest obvious
    benefit of using an existing commercial ISA is the large and widely supported software ecosystem,
    both development tools and ported applications, which can be leveraged in research and teaching.
    Other benefits include the existence of large amounts of documentation and tutorial examples.
    However, our experience of using commercial instruction sets for research and teaching is that
    these benefits are smaller in practice, and do not outweigh the disadvantages:
- Commercial ISAs are proprietary. Except for SPARC V8, which is an open IEEE standard [2], most owners of commercial ISAs carefully guard their intellectual property and do not welcome freely available competitive implementations. This is much less of an issue for academic research and teaching using only software simulators, but has been a major concern for groups wishing to share actual RTL implementations. It is also a major concern for entities who do not want to trust the few sources of commercial ISA implementations, but who are prohibited from creating their own clean room implementations. We cannot guarantee that all RISC-V implementations will be free of third-party patent infringements, but we can guarantee we will not attempt to sue a RISC-V implementor.
- Commercial ISAs are only popular in certain market domains. The most obvious examples at time of writing are that the ARM architecture is not well supported in the server space, and the Intel x86 architecture (or for that matter, almost every other architecture) is not well supported in the mobile space, though both Intel and ARM are attempting to enter each other’s market segments. Another example is ARC and Tensilica, which provide extensible cores but are focused on the embedded space. This market segmentation dilutes the benefit of supporting a particular commercial ISA as in practice the software ecosystem only exists for certain domains, and has to be built for others.
- Commercial ISAs come and go. Previous research infrastructures have been built around commercial ISAs that are no longer popular (SPARC, MIPS) or even no longer in production (Alpha). These lose the benefit of an active software ecosystem, and the lingering intellectual property issues around the ISA and supporting tools interfere with the ability of interested third parties to continue supporting the ISA. An open ISA might also lose popularity, but any interested party can continue using and developing the ecosystem.
- Popular commercial ISAs are complex. The dominant commercial ISAs (x86 and ARM) are both very complex to implement in hardware to the level of supporting common software stacks and operating systems. Worse, nearly all the complexity is due to bad, or at least outdated, ISA design decisions rather than features that truly improve efficiency.
- Commercial ISAs alone are not enough to bring up applications. Even if we expend the effort to implement a commercial ISA, this is not enough to run existing applications for that ISA. Most applications need a complete ABI (application binary interface) to run, not just the user-level ISA. Most ABIs rely on libraries, which in turn rely on operating system support. To run an existing operating system requires implementing the supervisor-level ISA and device interfaces expected by the OS. These are usually much less well-specified and considerably more complex to implement than the user-level ISA.

<!-- p.15 -->

- Popular commercial ISAs were not designed for extensibility. The dominant commercial ISAs were not particularly designed for extensibility, and as a consequence have added considerable instruction encoding complexity as their instruction sets have grown. Companies such as Tensilica (acquired by Cadence) and ARC (acquired by Synopsys) have built ISAs and toolchains around extensibility, but have focused on embedded applications rather than general-purpose computing systems.
- A modified commercial ISA is a new ISA. One of our main goals is to support architecture research, including major ISA extensions. Even small extensions diminish the benefit of using a standard ISA, as compilers have to be modified and applications rebuilt from source code to use the extension. Larger extensions that introduce new architectural state also require modifications to the operating system. Ultimately, the modified commercial ISA becomes a new ISA, but carries along all the legacy baggage of the base ISA. Our position is that the ISA is perhaps the most important interface in a computing system, and there is no reason that such an important interface should be proprietary. The dominant commercial ISAs are based on instruction set concepts that were already well known over 30 years ago. Software developers should be able to target an open standard hardware target, and commercial processor designers should compete on implementation quality. We are far from the first to contemplate an open ISA design suitable for hardware implementation. We also considered other existing open ISA designs, of which the closest to our goals was the OpenRISC architecture [22]. We decided against adopting the OpenRISC ISA for several technical reasons:
- OpenRISC has condition codes and branch delay slots, which complicate higher performance implementations.
- OpenRISC uses a fixed 32-bit encoding and 16-bit immediates, which precludes a denser instruction encoding and limits space for later expansion of the ISA.
- OpenRISC does not support the 2008 revision to the IEEE 754 floating-point standard.
- The OpenRISC 64-bit design had not been completed when we began. By starting from a clean slate, we could design an ISA that met all of our goals, though of course, this took far more effort than we had planned at the outset. We have now invested considerable effort in building up the RISC-V ISA infrastructure, including documentation, compiler tool chains, operating system ports, reference ISA simulators, FPGA implementations, efficient ASIC implementations, architecture test suites, and teaching materials. Since the last edition of this manual, there has been considerable uptake of the RISC-V ISA in both academia and industry, and we have created the non-profit RISC-V Foundation to protect and promote the standard. The RISC-V Foundation website at http: // riscv. org contains the latest information on the Foundation membership and various open-source projects using RISC-V.

The RISC-V manual is structured in two volumes. This volume covers the user-level ISA design,
including optional ISA extensions. The second volume provides the privileged architecture.

      In this user-level manual, we aim to remove any dependence on particular microarchitectural
      features or on privileged architecture details. This is both for clarity and to allow maximum
      flexibility for alternative implementations.

#### 1.1 RISC-V ISA Overview

The RISC-V ISA is defined as a base integer ISA, which must be present in any implementation,
plus optional extensions to the base ISA. The base integer ISA is very similar to that of the early RISC processors except with no branch delay slots and with support for optional variable-length
instruction encodings. The base is carefully restricted to a minimal set of instructions sufficient
to provide a reasonable target for compilers, assemblers, linkers, and operating systems (with
additional supervisor-level operations), and so provides a convenient ISA and software toolchain
“skeleton” around which more customized processor ISAs can be built.

<!-- p.16 -->

Each base integer instruction set is characterized by the width of the integer registers and the
corresponding size of the user address space. There are two primary base integer variants, RV32I
and RV64I, described in Chapters 2 and 4, which provide 32-bit or 64-bit user-level address spaces
respectively. Hardware implementations and operating systems might provide only one or both of
RV32I and RV64I for user programs. Chapter 3 describes the RV32E subset variant of the RV32I
base instruction set, which has been added to support small microcontrollers. Chapter 5 describes
a future RV128I variant of the base integer instruction set supporting a flat 128-bit user address
space. The base integer instruction sets use a two’s-complement representation for signed integer
values.

     Although 64-bit address spaces are a requirement for larger systems, we believe 32-bit address
     spaces will remain adequate for many embedded and client devices for decades to come and will
     be desirable to lower memory traffic and energy consumption. In addition, 32-bit address spaces
     are sufficient for educational purposes. A larger flat 128-bit address space might eventually be
     required, so we ensured this could be accommodated within the RISC-V ISA framework.
The base integer ISA may be subset by a hardware implementation, but opcode traps and software
emulation by a more privileged layer must then be used to implement functionality not provided
by hardware.

     Subsets of the base integer ISA might be useful for pedagogical purposes, but the base has been
     defined such that there should be little incentive to subset a real hardware implementation beyond
     omitting support for misaligned memory accesses and treating all SYSTEM instructions as a
     single trap.
RISC-V has been designed to support extensive customization and specialization. The base integer
ISA can be extended with one or more optional instruction-set extensions, but the base integer
instructions cannot be redefined. We divide RISC-V instruction-set extensions into standard and
non-standard extensions. Standard extensions should be generally useful and should not conflict
with other standard extensions. Non-standard extensions may be highly specialized, or may conflict
with other standard or non-standard extensions. Instruction-set extensions may provide slightly
different functionality depending on the width of the base integer instruction set. Chapter 21
describes various ways of extending the RISC-V ISA. We have also developed a naming convention
for RISC-V base instructions and instruction-set extensions, described in detail in Chapter 22.

To support more general software development, a set of standard extensions are defined to provide
integer multiply/divide, atomic operations, and single and double-precision floating-point arithmetic. The base integer ISA is named “I” (prefixed by RV32 or RV64 depending on integer register width), and contains integer computational instructions, integer loads, integer stores, and
control-flow instructions, and is mandatory for all RISC-V implementations. The standard integer
multiplication and division extension is named “M”, and adds instructions to multiply and divide
values held in the integer registers. The standard atomic instruction extension, denoted by “A”,
adds instructions that atomically read, modify, and write memory for inter-processor synchronization. The standard single-precision floating-point extension, denoted by “F”, adds floating-point registers, single-precision computational instructions, and single-precision loads and stores. The
standard double-precision floating-point extension, denoted by “D”, expands the floating-point
registers, and adds double-precision computational instructions, loads, and stores. An integer base
plus these four standard extensions (“IMAFD”) is given the abbreviation “G” and provides a
general-purpose scalar instruction set. RV32G and RV64G are currently the default target of our
compiler toolchains. Later chapters describe these and other planned standard RISC-V extensions.

<!-- p.17 -->

Beyond the base integer ISA and the standard extensions, it is rare that a new instruction will
provide a significant benefit for all applications, although it may be very beneficial for a certain
domain. As energy efficiency concerns are forcing greater specialization, we believe it is important to
simplify the required portion of an ISA specification. Whereas other architectures usually treat their
ISA as a single entity, which changes to a new version as instructions are added over time, RISC-V
will endeavor to keep the base and each standard extension constant over time, and instead layer
new instructions as further optional extensions. For example, the base integer ISAs will continue
as fully supported standalone ISAs, regardless of any subsequent extensions.

      With the 2.0 release of the user ISA specification, we intend the “RV32IMAFD” and
      “RV64IMAFD”base and standard extensions (aka. “RV32G” and “RV64G”) to remain con-
      stant for future development.

#### 1.2 Instruction Length Encoding

The base RISC-V ISA has fixed-length 32-bit instructions that must be naturally aligned on 32-bit
boundaries. However, the standard RISC-V encoding scheme is designed to support ISA extensions
with variable-length instructions, where each instruction can be any number of 16-bit instruction
parcels in length and parcels are naturally aligned on 16-bit boundaries. The standard compressed
ISA extension described in Chapter 12 reduces code size by providing compressed 16-bit instructions
and relaxes the alignment constraints to allow all instructions (16 bit and 32 bit) to be aligned on
any 16-bit boundary to improve code density.

Figure 1.1 illustrates the standard RISC-V instruction-length encoding convention. All the 32-bit
instructions in the base ISA have their lowest two bits set to 11. The optional compressed 16-bit
instruction-set extensions have their lowest two bits equal to 00, 01, or 10. Standard instructionset extensions encoded with more than 32 bits have additional low-order bits set to 1, with the
conventions for 48-bit and 64-bit lengths shown in Figure 1.1. Instruction lengths between 80 bits
and 176 bits are encoded using a 3-bit field in bits [14:12] giving the number of 16-bit words in
addition to the first 5×16-bit words. The encoding with bits [14:12] set to 111 is reserved for future
longer instruction encodings.

      Given the code size and energy savings of a compressed format, we wanted to build in support
      for a compressed format to the ISA encoding scheme rather than adding this as an afterthought,
      but to allow simpler implementations we didn’t want to make the compressed format mandatory.
      We also wanted to optionally allow longer instructions to support experimentation and larger
      instruction-set extensions. Although our encoding convention required a tighter encoding of the
      core RISC-V ISA, this has several beneficial effects.
          An implementation of the standard G ISA need only hold the most-significant 30 bits in
      instruction caches (a 6.25% saving). On instruction cache refills, any instructions encountered xxxxxxxxxxxxxxaa        16-bit (aa 6= 11)

<!-- p.18 -->

```text
                                 xxxxxxxxxxxxxxxx         xxxxxxxxxxxbbb11        32-bit (bbb 6= 111)
```

- · ·xxxx   xxxxxxxxxxxxxxxx         xxxxxxxxxx011111        48-bit

- · ·xxxx   xxxxxxxxxxxxxxxx         xxxxxxxxx0111111        64-bit

- · ·xxxx   xxxxxxxxxxxxxxxx         xnnnxxxxx1111111        (80+16*nnn)-bit, nnn6=111

- · ·xxxx   xxxxxxxxxxxxxxxx         x111xxxxx1111111        Reserved for ≥192-bits

```text
    Byte Address:     base+4                   base+2                      base
```

**Figure 1.1:** RISC-V instruction length encoding.

        with either low bit clear should be recoded into illegal 30-bit instructions before storing in the
        cache to preserve illegal instruction exception behavior.
            Perhaps more importantly, by condensing our base ISA into a subset of the 32-bit instruc-
        tion word, we leave more space available for custom extensions. In particular, the base RV32I
        ISA uses less than 1/8 of the encoding space in the 32-bit instruction word. As described in
        Chapter 21, an implementation that does not require support for the standard compressed in-
        struction extension can map 3 additional 30-bit instruction spaces into the 32-bit fixed-width
        format, while preserving support for standard >=32-bit instruction-set extensions. Further, if
        the implementation also does not need instructions >32-bits in length, it can recover a further
        four major opcodes.

        We consider it a feature that any length of instruction containing all zero bits is not legal, as
        this quickly traps erroneous jumps into zeroed memory regions. Similarly, we also reserve the
        instruction encoding containing all ones to be an illegal instruction, to catch the other common
        pattern observed with unprogrammed non-volatile memory devices, disconnected memory buses,
        or broken memory devices.
The base RISC-V ISA has a little-endian memory system, but non-standard variants can provide a
big-endian or bi-endian memory system. Instructions are stored in memory with each 16-bit parcel
stored in a memory halfword according to the implementation’s natural endianness. Parcels forming
one instruction are stored at increasing halfword addresses, with the lowest addressed parcel holding
the lowest numbered bits in the instruction specification, i.e., instructions are always stored in a
little-endian sequence of parcels regardless of the memory system endianness. The code sequence
in Figure 1.2 will store a 32-bit instruction to memory correctly regardless of memory system
endianness.

        We chose little-endian byte ordering for the RISC-V memory system because little-endian sys-
        tems are currently dominant commercially (all x86 systems; iOS, Android, and Windows for
        ARM). A minor point is that we have also found little-endian memory systems to be more nat-
        ural for hardware designers. However, certain application areas, such as IP networking, operate
        on big-endian data structures, and so we leave open the possibility of non-standard big-endian
        or bi-endian systems.
            We have to fix the order in which instruction parcels are stored in memory, independent
        of memory system endianness, to ensure that the length-encoding bits always appear first in

<!-- p.19 -->

```text
      // Store   32-bit instruction in x2 register to location pointed to by x3.
      sh   x2,   0(x3)    // Store low bits of instruction in first parcel.
      srli x2,   x2, 16   // Move high bits down to low bits, overwriting x2.
      sh   x2,   2(x3)    // Store high bits in second parcel.
```

**Figure 1.2:** Recommended code sequence to store 32-bit instruction from register to memory. Operates correctly on both big- and little-endian memory systems and avoids misaligned accesses when used with variable-length instruction-set extensions.

      halfword address order. This allows the length of a variable-length instruction to be quickly
      determined by an instruction fetch unit by examining only the first few bits of the first 16-bit
      instruction parcel. Once we had decided to fix on a little-endian memory system and instruction
      parcel ordering, this naturally led to placing the length-encoding bits in the LSB positions of the
      instruction format to avoid breaking up opcode fields.

#### 1.3 Exceptions, Traps, and Interrupts

We use the term exception to refer to an unusual condition occurring at run time associated with
an instruction in the current RISC-V thread. We use the term trap to refer to the synchronous
transfer of control to a trap handler caused by an exceptional condition occurring within a RISC-V
thread. Trap handlers usually execute in a more privileged environment.

We use the term interrupt to refer to an external event that occurs asynchronously to the current
RISC-V thread. When an interrupt that must be serviced occurs, some instruction is selected to
receive an interrupt exception and subsequently experiences a trap.

The instruction descriptions in following chapters describe conditions that raise an exception during execution. Whether and how these are converted into traps is dependent on the execution
environment, though the expectation is that most environments will take a precise trap when an
exception is signaled (except for floating-point exceptions, which, in the standard floating-point
extensions, do not cause traps).

      Our use of “exception” and “trap” matches that in the IEEE-754 floating-point standard.

<!-- p.21 -->

### 第 2 章 RV32I Base Integer Instruction Set, Version 2.0

This chapter describes version 2.0 of the RV32I base integer instruction set. Much of the commentary also applies to the RV64I variant.

      RV32I was designed to be sufficient to form a compiler target and to support modern operating
      system environments. The ISA was also designed to reduce the hardware required in a mini-
      mal implementation. RV32I contains 47 unique instructions, though a simple implementation
      might cover the eight SCALL/SBREAK/CSRR* instructions with a single SYSTEM hardware
      instruction that always traps and might be able to implement the FENCE and FENCE.I in-
      structions as NOPs, reducing hardware instruction count to 38 total. RV32I can emulate almost
      any other ISA extension (except the A extension, which requires additional hardware support for
      atomicity).

#### 2.1 Programmers’ Model for Base Integer Subset

Figure 2.1 shows the user-visible state for the base integer subset. There are 31 general-purpose
registers x1–x31, which hold integer values. Register x0 is hardwired to the constant 0. There is
no hardwired subroutine return address link register, but the standard software calling convention
uses register x1 to hold the return address on a call. For RV32, the x registers are 32 bits wide,
and for RV64, they are 64 bits wide. This document uses the term XLEN to refer to the current
width of an x register in bits (either 32 or 64).

There is one additional user-visible register: the program counter pc holds the address of the current
instruction.

      The number of available architectural registers can have large impacts on code size, performance,
      and energy consumption. Although 16 registers would arguably be sufficient for an integer ISA
      running compiled code, it is impossible to encode a complete ISA with 16 registers in 16-bit
      instructions using a 3-address format. Although a 2-address format would be possible, it would
      increase instruction count and lower efficiency. We wanted to avoid intermediate instruction
      sizes (such as Xtensa’s 24-bit instructions) to simplify base hardware implementations, and once a 32-bit instruction size was adopted, it was straightforward to support 32 integer registers. A
     larger number of integer registers also helps performance on high-performance code, where there
     can be extensive use of loop unrolling, software pipelining, and cache tiling.
         For these reasons, we chose a conventional size of 32 integer registers for the base ISA. Dy-
     namic register usage tends to be dominated by a few frequently accessed registers, and regfile im-
     plementations can be optimized to reduce access energy for the frequently accessed registers [31].
     The optional compressed 16-bit instruction format mostly only accesses 8 registers and hence can
     provide a dense instruction encoding, while additional instruction-set extensions could support
     a much larger register space (either flat or hierarchical) if desired.
         For resource-constrained embedded applications, we have defined the RV32E subset, which
     only has 16 registers (Chapter 3).

<!-- p.22 -->

                                     XLEN-1                           0
                                                   x0 / zero
                                                   x1
                                                   x2
                                                   x3
                                                   x4
                                                   x5
                                                   x6
                                                   x7
                                                   x8
                                                   x9
                                                  x10
                                                  x11
                                                  x12
                                                  x13
                                                  x14
                                                  x15
                                                  x16
                                                  x17
                                                  x18
                                                  x19
                                                  x20
                                                  x21
                                                  x22
                                                  x23
                                                  x24
                                                  x25
                                                  x26
                                                  x27
                                                  x28
                                                  x29
                                                  x30
                                                  x31
                                                  XLEN
                                     XLEN-1                           0
                                                   pc
                                                  XLEN

**Figure 2.1:** RISC-V user-level base integer register state.

<!-- p.23 -->

#### 2.2 Base Instruction Formats

In the base ISA, there are four core instruction formats (R/I/S/U), as shown in Figure 2.2. All are
a fixed 32 bits in length and must be aligned on a four-byte boundary in memory. An instruction
address misaligned exception is generated on a taken branch or unconditional jump if the target
address is not four-byte aligned. No instruction fetch misaligned exception is generated for a
conditional branch that is not taken.

      The alignment constraint for base ISA instructions is relaxed to a two-byte boundary when
      instruction extensions with 16-bit lengths or other odd multiples of 16-bit lengths are added.

```text
      31                25 24         20 19         15 14    12 11          7 6              0
            funct7              rs2           rs1      funct3        rd           opcode         R-type
                imm[11:0]                     rs1      funct3        rd           opcode         I-type
           imm[11:5]            rs2           rs1      funct3    imm[4:0]         opcode         S-type
                            imm[31:12]                               rd           opcode         U-type
```

**Figure 2.2:** RISC-V base instruction formats. Each immediate subfield is labeled with the bit position (imm[x ]) in the immediate value being produced, rather than the bit position within the instruction’s immediate field as is usually done.

The RISC-V ISA keeps the source (rs1 and rs2) and destination (rd) registers at the same position
in all formats to simplify decoding. Except for the 5-bit immediates used in CSR instructions
(Section 2.8), immediates are always sign-extended, and are generally packed towards the leftmost
available bits in the instruction and have been allocated to reduce hardware complexity. In particular, the sign bit for all immediates is always in bit 31 of the instruction to speed sign-extension
circuitry.

      Decoding register specifiers is usually on the critical paths in implementations, and so the in-
      struction format was chosen to keep all register specifiers at the same position in all formats at
      the expense of having to move immediate bits across formats (a property shared with RISC-IV
      aka. SPUR [18]).
          In practice, most immediates are either small or require all XLEN bits. We chose an asym-
      metric immediate split (12 bits in regular instructions plus a special load upper immediate in-
      struction with 20 bits) to increase the opcode space available for regular instructions.
          Immediates are sign-extended because we did not observe a benefit to using zero-extension
      for some immediates as in the MIPS ISA and wanted to keep the ISA as simple as possible.

#### 2.3 Immediate Encoding Variants

There are a further two variants of the instruction formats (B/J) based on the handling of immediates, as shown in Figure 2.3.

<!-- p.24 -->

The only difference between the S and B formats is that the 12-bit immediate field is used to encode
branch offsets in multiples of 2 in the B format. Instead of shifting all bits in the instruction-encoded
immediate left by one in hardware as is conventionally done, the middle bits (imm[10:1]) and sign
bit stay in fixed positions, while the lowest bit in S format (inst[7]) encodes a high-order bit in B
format.

Similarly, the only difference between the U and J formats is that the 20-bit immediate is shifted
left by 12 bits to form U immediates and by 1 bit to form J immediates. The location of instruction
bits in the U and J format immediates is chosen to maximize overlap with the other formats and
with each other.

```text
          31         30         25 24       21     20       19            15 14    12 11         8        7          6    0
                funct7                      rs2                   rs1        funct3              rd                  opcode R-type
                            imm[11:0]                             rs1        funct3              rd                  opcode I-type
               imm[11:5]                    rs2                   rs1        funct3            imm[4:0]              opcode S-type
     imm[12]         imm[10:5]              rs2                   rs1        funct3   imm[4:1] imm[11] opcode B-type
                                        imm[31:12]                                               rd                  opcode U-type
     imm[20]                imm[10:1]            imm[11]           imm[19:12]                    rd                  opcode J-type
```

**Figure 2.3:** RISC-V base instruction formats showing immediate variants.

Figure 2.4 shows the immediates produced by each of the base instruction formats, and is labeled
to show which instruction bit (inst[y ]) produces each bit of the immediate value.

```text
     31         30                  20 19                  12     11        10        5    4          1          0
                              — inst[31] —                                  inst[30:25] inst[24:21] inst[20] I-immediate
                              — inst[31] —                                  inst[30:25]    inst[11:8]         inst[7]    S-immediate
                          — inst[31] —                          inst[7]     inst[30:25]    inst[11:8]           0        B-immediate
inst[31]              inst[30:20]          inst[19:12]                                —0—                                U-immediate
           — inst[31] —                    inst[19:12]          inst[20] inst[30:25] inst[24:21]                0        J-immediate
```

**Figure 2.4:** Types of immediate produced by RISC-V instructions. The fields are labeled with the instruction bits used to construct their value. Sign extension always uses inst[31].

          Sign-extension is one of the most critical operations on immediates (particularly in RV64I), and
          in RISC-V the sign bit for all immediates is always held in bit 31 of the instruction to allow
          sign-extension to proceed in parallel with instruction decoding.
              Although more complex implementations might have separate adders for branch and jump
          calculations and so would not benefit from keeping the location of immediate bits constant across types of instruction, we wanted to reduce the hardware cost of the simplest implementations. By
       rotating bits in the instruction encoding of B and J immediates instead of using dynamic hard-
       ware muxes to multiply the immediate by 2, we reduce instruction signal fanout and immediate
       mux costs by around a factor of 2. The scrambled immediate encoding will add negligible time
       to static or ahead-of-time compilation. For dynamic generation of instructions, there is some
       small additional overhead, but the most common short forward branches have straightforward
       immediate encodings.

<!-- p.25 -->

#### 2.4 Integer Computational Instructions

Most integer computational instructions operate on XLEN bits of values held in the integer register
file. Integer computational instructions are either encoded as register-immediate operations using
the I-type format or as register-register operations using the R-type format. The destination is
register rd for both register-immediate and register-register instructions. No integer computational
instructions cause arithmetic exceptions.

       We did not include special instruction set support for overflow checks on integer arithmetic
       operations in the base instruction set, as many overflow checks can be cheaply implemented using
       RISC-V branches. Overflow checking for unsigned addition requires only a single additional
       branch instruction after the addition: add t0, t1, t2; bltu t0, t1, overflow.
           For signed addition, if one operand’s sign is known, overflow checking requires only a single
       branch after the addition: addi t0, t1, +imm; blt t0, t1, overflow. This covers the
       common case of addition with an immediate operand.
           For general signed addition, three additional instructions after the addition are required,
       leveraging the observation that the sum should be less than one of the operands if and only if the
       other operand is negative.

```asm
                  add t0, t1, t2
                  slti t3, t2, 0
                  slt t4, t0, t1
                  bne t3, t4, overflow
```

       In RV64, checks of 32-bit signed additions can be optimized further by comparing the results of
       ADD and ADDW on the operands.

Integer Register-Immediate Instructions

```text
  31                                 20 19          15 14              12 11           7 6                  0
             imm[11:0]                       rs1         funct3                 rd             opcode
                12                            5             3                    5               7
          I-immediate[11:0]                  src      ADDI/SLTI[U]             dest           OP-IMM
          I-immediate[11:0]                  src      ANDI/ORI/XORI            dest           OP-IMM
```

ADDI adds the sign-extended 12-bit immediate to register rs1. Arithmetic overflow is ignored and
the result is simply the low XLEN bits of the result. ADDI rd, rs1, 0 is used to implement the MV
rd, rs1 assembler pseudo-instruction.

SLTI (set less than immediate) places the value 1 in register rd if register rs1 is less than the signextended immediate when both are treated as signed numbers, else 0 is written to rd. SLTIU is similar but compares the values as unsigned numbers (i.e., the immediate is first sign-extended to
XLEN bits then treated as an unsigned number). Note, SLTIU rd, rs1, 1 sets rd to 1 if rs1 equals
zero, otherwise sets rd to 0 (assembler pseudo-op SEQZ rd, rs).

<!-- p.26 -->

ANDI, ORI, XORI are logical operations that perform bitwise AND, OR, and XOR on register rs1
and the sign-extended 12-bit immediate and place the result in rd. Note, XORI rd, rs1, -1 performs
a bitwise logical inversion of register rs1 (assembler pseudo-instruction NOT rd, rs).

```text
     31               25 24          20 19          15 14               12 11                7 6                  0
          imm[11:5]        imm[4:0]          rs1            funct3               rd                  opcode
              7               5               5                3                  5                    7
           0000000        shamt[4:0]         src             SLLI               dest                OP-IMM
           0000000        shamt[4:0]         src             SRLI               dest                OP-IMM
           0100000        shamt[4:0]         src             SRAI               dest                OP-IMM
```

Shifts by a constant are encoded as a specialization of the I-type format. The operand to be shifted
is in rs1, and the shift amount is encoded in the lower 5 bits of the I-immediate field. The right
shift type is encoded in a high bit of the I-immediate. SLLI is a logical left shift (zeros are shifted
into the lower bits); SRLI is a logical right shift (zeros are shifted into the upper bits); and SRAI
is an arithmetic right shift (the original sign bit is copied into the vacated upper bits).

```text
          31                                                         12 11             7 6                    0
                                 imm[31:12]                                   rd                   opcode
                                     20                                        5                     7
                              U-immediate[31:12]                             dest                   LUI
                              U-immediate[31:12]                             dest                  AUIPC
```

LUI (load upper immediate) is used to build 32-bit constants and uses the U-type format. LUI
places the U-immediate value in the top 20 bits of the destination register rd, filling in the lowest
12 bits with zeros.

AUIPC (add upper immediate to pc) is used to build pc-relative addresses and uses the U-type
format. AUIPC forms a 32-bit offset from the 20-bit U-immediate, filling in the lowest 12 bits with
zeros, adds this offset to the pc, then places the result in register rd.

          The AUIPC instruction supports two-instruction sequences to access arbitrary offsets from the
          PC for both control-flow transfers and data accesses. The combination of an AUIPC and the
          12-bit immediate in a JALR can transfer control to any 32-bit PC-relative address, while an
          AUIPC plus the 12-bit immediate offset in regular load or store instructions can access any
          32-bit PC-relative data address.
              The current PC can be obtained by setting the U-immediate to 0. Although a JAL +4
          instruction could also be used to obtain the PC, it might cause pipeline breaks in simpler mi-
          croarchitectures or pollute the BTB structures in more complex microarchitectures.

Integer Register-Register Operations

RV32I defines several arithmetic R-type operations. All operations read the rs1 and rs2 registers
as source operands and write the result into register rd. The funct7 and funct3 fields select the type of operation.

<!-- p.27 -->

```text
   31               25 24          20 19             15 14             12 11           7 6                 0
         funct7              rs2              rs1         funct3                 rd            opcode
            7                 5                5             3                    5              7
         0000000            src2             src1      ADD/SLT/SLTU             dest            OP
         0000000            src2             src1      AND/OR/XOR               dest            OP
         0000000            src2             src1        SLL/SRL                dest            OP
         0100000            src2             src1        SUB/SRA                dest            OP
```

ADD and SUB perform addition and subtraction respectively. Overflows are ignored and the low
XLEN bits of results are written to the destination. SLT and SLTU perform signed and unsigned
compares respectively, writing 1 to rd if rs1 < rs2, 0 otherwise. Note, SLTU rd, x0, rs2 sets rd to 1
if rs2 is not equal to zero, otherwise sets rd to zero (assembler pseudo-op SNEZ rd, rs). AND, OR,
and XOR perform bitwise logical operations.

SLL, SRL, and SRA perform logical left, logical right, and arithmetic right shifts on the value in
register rs1 by the shift amount held in the lower 5 bits of register rs2.

NOP Instruction

```text
  31                                 20 19            15 14             12 11            7 6                   0
               imm[11:0]                       rs1            funct3              rd            opcode
                  12                            5                3                 5              7
                   0                            0             ADDI                 0           OP-IMM
```

The NOP instruction does not change any user-visible state, except for advancing the pc. NOP is
encoded as ADDI x0, x0, 0.

        NOPs can be used to align code segments to microarchitecturally significant address boundaries,
        or to leave space for inline code modifications. Although there are many possible ways to encode
        a NOP, we define a canonical NOP encoding to allow microarchitectural optimizations as well
        as for more readable disassembly output.

#### 2.5 Control Transfer Instructions

RV32I provides two types of control transfer instructions: unconditional jumps and conditional
branches. Control transfer instructions in RV32I do not have architecturally visible delay slots.

Unconditional Jumps

The jump and link (JAL) instruction uses the J-type format, where the J-immediate encodes a
signed offset in multiples of 2 bytes. The offset is sign-extended and added to the pc to form the
jump target address. Jumps can therefore target a ±1 MiB range. JAL stores the address of the
instruction following the jump (pc+4) into register rd. The standard software calling convention
uses x1 as the return address register and x5 as an alternate link register.

<!-- p.28 -->

     The alternate link register supports calling millicode routines (e.g., those to save and restore
     registers in compressed code) while preserving the regular return address register. The register
     x5 was chosen as the alternate link register as it maps to a temporary in the standard calling
     convention, and has an encoding that is only one bit different than the regular link register.

Plain unconditional jumps (assembler pseudo-op J) are encoded as a JAL with rd=x0.

```text
          31   30                    21       20          19           12 11          7 6             0
     imm[20]         imm[10:1]          imm[11]           imm[19:12]            rd          opcode
       1                10                1                   8                  5            7
                           offset[20:1]                                        dest          JAL
```

The indirect jump instruction JALR (jump and link register) uses the I-type encoding. The target
address is obtained by adding the 12-bit signed I-immediate to the register rs1, then setting the
least-significant bit of the result to zero. The address of the instruction following the jump (pc+4)
is written to register rd. Register x0 can be used as the destination if the result is not required.
     31                                   20 19                15 14   12 11          7 6             0
                imm[11:0]                           rs1           funct3        rd          opcode
                     12                              5              3            5            7
                offset[11:0]                       base             0          dest         JALR

     The unconditional jump instructions all use PC-relative addressing to help support position-
     independent code. The JALR instruction was defined to enable a two-instruction sequence to
     jump anywhere in a 32-bit absolute address range. A LUI instruction can first load rs1 with the
     upper 20 bits of a target address, then JALR can add in the lower bits. Similarly, AUIPC then
     JALR can jump anywhere in a 32-bit pc-relative address range.
         Note that the JALR instruction does not treat the 12-bit immediate as multiples of 2 bytes,
     unlike the conditional branch instructions. This avoids one more immediate format in hardware.
     In practice, most uses of JALR will have either a zero immediate or be paired with a LUI or
     AUIPC, so the slight reduction in range is not significant.
         The JALR instruction ignores the lowest bit of the calculated target address. This both
     simplifies the hardware slightly and allows the low bit of function pointers to be used to store
     auxiliary information. Although there is potentially a slight loss of error checking in this case,
     in practice jumps to an incorrect instruction address will usually quickly raise an exception.
         When used with a base rs1=x0, JALR can be used to implement a single instruction sub-
     routine call to the lowest 2 KiB or highest 2 KiB address region from anywhere in the address
     space, which could be used to implement fast calls to a small runtime library.

The JAL and JALR instructions will generate a misaligned instruction fetch exception if the target
address is not aligned to a four-byte boundary.

     Instruction fetch misaligned exceptions are not possible on machines that support extensions
     with 16-bit aligned instructions, such as the compressed instruction set extension, C.

Return-address prediction stacks are a common feature of high-performance instruction-fetch units,
but require accurate detection of instructions used for procedure calls and returns to be effective.
For RISC-V, hints as to the instructions’ usage are encoded implicitly via the register numbers
used. A JAL instruction should push the return address onto a return-address stack (RAS) only
when rd=x1/x5. JALR instructions should push/pop a RAS as shown in the Table 2.1.

<!-- p.29 -->

```text
                                  rd        rs1    rs1=rd   RAS action
                                !link      !link      -     none
                                !link      link       -     pop
                                link       !link      -     push
                                link       link       0     push and pop
                                link       link       1     push
```

**Table 2.1:** Return-address stack prediction hints encoded in register specifiers used in the instruc- tion. In the above, link is true when the register is either x1 or x5.

     Some other ISAs added explicit hint bits to their indirect-jump instructions to guide return-
     address stack manipulation. We use implicit hinting tied to register numbers and the calling
     convention to reduce the encoding space used for these hints.
         When two different link registers (x1 and x5) are given as rs1 and rd, then the RAS
     is both pushed and popped to support coroutines. If rs1 and rd are the same link regis-
     ter (either x1 or x5), the RAS is only pushed to enable macro-op fusion of the sequences:
     lui ra, imm20; jalr ra, ra, imm12 and auipc ra, imm20; jalr ra, ra, imm12

Conditional Branches

All branch instructions use the B-type instruction format. The 12-bit B-immediate encodes signed
offsets in multiples of 2, and is added to the current pc to give the target address. The conditional
branch range is ±4 KiB.

```text
     31     30          25 24      20 19       15 14        12 11          8   7      6              0
  imm[12] imm[10:5]          rs2         rs1        funct3      imm[4:1]    imm[11]        opcode
    1              6          5           5            3           4           1             7
      offset[12,10:5]       src2        src1       BEQ/BNE        offset[11,4:1]          BRANCH
      offset[12,10:5]       src2        src1        BLT[U]        offset[11,4:1]          BRANCH
      offset[12,10:5]       src2        src1        BGE[U]        offset[11,4:1]          BRANCH
```

Branch instructions compare two registers. BEQ and BNE take the branch if registers rs1 and rs2
are equal or unequal respectively. BLT and BLTU take the branch if rs1 is less than rs2, using
signed and unsigned comparison respectively. BGE and BGEU take the branch if rs1 is greater
than or equal to rs2, using signed and unsigned comparison respectively. Note, BGT, BGTU,
BLE, and BLEU can be synthesized by reversing the operands to BLT, BLTU, BGE, and BGEU,
respectively.

     Signed array bounds may be checked with a single BLTU instruction, since any negative index
     will compare greater than any nonnegative bound.

Software should be optimized such that the sequential code path is the most common path, with
less-frequently taken code paths placed out of line. Software should also assume that backward
branches will be predicted taken and forward branches as not taken, at least the first time they are
encountered. Dynamic predictors should quickly learn any predictable branch behavior.

<!-- p.30 -->

Unlike some other architectures, the RISC-V jump (JAL with rd=x0) instruction should always
be used for unconditional branches instead of a conditional branch instruction with an alwaystrue condition. RISC-V jumps are also PC-relative and support a much wider offset range than
branches, and will not pressure conditional branch prediction tables.

      The conditional branches were designed to include arithmetic comparison operations between
      two registers (as also done in PA-RISC and Xtensa ISA), rather than use condition codes (x86,
      ARM, SPARC, PowerPC), or to only compare one register against zero (Alpha, MIPS), or
      two registers only for equality (MIPS). This design was motivated by the observation that a
      combined compare-and-branch instruction fits into a regular pipeline, avoids additional condition
      code state or use of a temporary register, and reduces static code size and dynamic instruction
      fetch traffic. Another point is that comparisons against zero require non-trivial circuit delay
      (especially after the move to static logic in advanced processes) and so are almost as expensive as
      arithmetic magnitude compares. Another advantage of a fused compare-and-branch instruction
      is that branches are observed earlier in the front-end instruction stream, and so can be predicted
      earlier. There is perhaps an advantage to a design with condition codes in the case where multiple
      branches can be taken based on the same condition codes, but we believe this case to be relatively
      rare.
           We considered but did not include static branch hints in the instruction encoding. These
      can reduce the pressure on dynamic predictors, but require more instruction encoding space and
      software profiling for best results, and can result in poor performance if production runs do not
      match profiling runs.
           We considered but did not include conditional moves or predicated instructions, which can
      effectively replace unpredictable short forward branches. Conditional moves are the simpler of
      the two, but are difficult to use with conditional code that might cause exceptions (memory
      accesses and floating-point operations). Predication adds additional flag state to a system, addi-
      tional instructions to set and clear flags, and additional encoding overhead on every instruction.
      Both conditional move and predicated instructions add complexity to out-of-order microarchitec-
      tures, adding an implicit third source operand due to the need to copy the original value of the
      destination architectural register into the renamed destination physical register if the predicate
      is false. Also, static compile-time decisions to use predication instead of branches can result
      in lower performance on inputs not included in the compiler training set, especially given that
      unpredictable branches are rare, and becoming rarer as branch prediction techniques improve.
           We note that various microarchitectural techniques exist to dynamically convert unpredictable
      short forward branches into internally predicated code to avoid the cost of flushing pipelines on
      a branch mispredict [13, 17, 16] and have been implemented in commercial processors [27].
      The simplest techniques just reduce the penalty of recovering from a mispredicted short forward
      branch by only flushing instructions in the branch shadow instead of the entire fetch pipeline,
      or by fetching instructions from both sides using wide instruction fetch or idle instruction fetch
      slots. More complex techniques for out-of-order cores add internal predicates on instructions in
      the branch shadow, with the internal predicate value written by the branch instruction, allowing
      the branch and following instructions to be executed speculatively and out-of-order with respect
      to other code [27].

#### 2.6 Load and Store Instructions

RV32I is a load-store architecture, where only load and store instructions access memory and
arithmetic instructions only operate on CPU registers. RV32I provides a 32-bit user address space
that is byte-addressed and little-endian. The execution environment will define what portions of
the address space are legal to access. Loads with a destination of x0 must still raise any exceptions
and action any other side effects even though the load value is discarded.

<!-- p.31 -->

```text
     31                                 20 19          15 14    12 11              7 6                0
                  imm[11:0]                      rs1      funct3         rd              opcode
                       12                         5         3             5                7
                  offset[11:0]                  base      width         dest             LOAD
      31                  25 24         20 19          15 14   12 11               7 6                0
           imm[11:5]              rs2            rs1      funct3     imm[4:0]             opcode
                7                  5              5         3             5                 7
           offset[11:5]           src           base      width      offset[4:0]         STORE
```

Load and store instructions transfer a value between the registers and memory. Loads are encoded
in the I-type format and stores are S-type. The effective byte address is obtained by adding register
rs1 to the sign-extended 12-bit offset. Loads copy a value from memory to register rd. Stores copy
the value in register rs2 to memory.

The LW instruction loads a 32-bit value from memory into rd. LH loads a 16-bit value from memory,
then sign-extends to 32-bits before storing in rd. LHU loads a 16-bit value from memory but then
zero extends to 32-bits before storing in rd. LB and LBU are defined analogously for 8-bit values.
The SW, SH, and SB instructions store 32-bit, 16-bit, and 8-bit values from the low bits of register
rs2 to memory.

For best performance, the effective address for all loads and stores should be naturally aligned
for each data type (i.e., on a four-byte boundary for 32-bit accesses, and a two-byte boundary for
16-bit accesses). The base ISA supports misaligned accesses, but these might run extremely slowly
depending on the implementation. Furthermore, naturally aligned loads and stores are guaranteed
to execute atomically, whereas misaligned loads and stores might not, and hence require additional
synchronization to ensure atomicity.

     Misaligned accesses are occasionally required when porting legacy code, and are essential for good
     performance on many applications when using any form of packed-SIMD extension. Our ratio-
     nale for supporting misaligned accesses via the regular load and store instructions is to simplify
     the addition of misaligned hardware support. One option would have been to disallow misaligned
     accesses in the base ISA and then provide some separate ISA support for misaligned accesses,
     either special instructions to help software handle misaligned accesses or a new hardware address-
     ing mode for misaligned accesses. Special instructions are difficult to use, complicate the ISA,
     and often add new processor state (e.g., SPARC VIS align address offset register) or complicate
     access to existing processor state (e.g., MIPS LWL/LWR partial register writes). In addition,
     for loop-oriented packed-SIMD code, the extra overhead when operands are misaligned motivates
     software to provide multiple forms of loop depending on operand alignment, which complicates
     code generation and adds to loop startup overhead. New misaligned hardware addressing modes
     take considerable space in the instruction encoding or require very simplified addressing modes
     (e.g., register indirect only).
         We do not mandate atomicity for misaligned accesses so simple implementations can just
     use a machine trap and software handler to handle some or all misaligned accesses. If hardware
     misaligned support is provided, software can exploit this by simply using regular load and store
     instructions. Hardware can then automatically optimize accesses depending on whether runtime
     addresses are aligned.

<!-- p.32 -->

#### 2.7 Memory Model

        This section is out of date as the RISC-V memory model is currently under revision to ensure it
        can efficiently support current programming language memory models. The revised base mem-
        ory model will contain further ordering constraints, including at least that loads to the same
        address from the same hart cannot be reordered, and that syntactic data dependencies between
        instructions are respected.

The base RISC-V ISA supports multiple concurrent threads of execution within a single user address
space. Each RISC-V hardware thread, or hart, has its own user register state and program counter,
and executes an independent sequential instruction stream. The execution environment will define
how RISC-V harts are created and managed. RISC-V harts can communicate and synchronize with
other harts either via calls to the execution environment, which are documented separately in the
specification for each execution environment, or directly via the shared memory system. RISC-V
harts can also interact with I/O devices, and indirectly with each other, via loads and stores to
portions of the address space assigned to I/O.

        We use the term hart to unambiguously and concisely describe a hardware thread as opposed to
        software-managed thread contexts.

In the base RISC-V ISA, each RISC-V hart observes its own memory operations as if they executed
sequentially in program order. RISC-V has a relaxed memory model between harts, requiring an
explicit FENCE instruction to guarantee ordering between memory operations from different RISCV harts. Chapter 7 describes the optional atomic memory instruction extensions “A”, which provide
additional synchronization operations.

```text
  31        28 27   26    25    24     23    22    21    20     19     15 14   12 11        7 6              0
       0      PI    PO PR PW           SI   SO SR SW             rs1      funct3       rd           opcode
        4     1      1     1    1      1     1    1   1            5        3           5             7
        0           predecessor             successor              0     FENCE          0         MISC-MEM
```

The FENCE instruction is used to order device I/O and memory accesses as viewed by other RISCV harts and external devices or coprocessors. Any combination of device input (I), device output
(O), memory reads (R), and memory writes (W) may be ordered with respect to any combination
of the same. Informally, no other RISC-V hart or external device can observe any operation
in the successor set following a FENCE before any operation in the predecessor set preceding the
FENCE. The execution environment will define what I/O operations are possible, and in particular,
which load and store instructions might be treated and ordered as device input and device output
operations respectively rather than memory reads and writes. For example, memory-mapped I/O
devices will typically be accessed with uncached loads and stores that are ordered using the I and O
bits rather than the R and W bits. Instruction-set extensions might also describe new coprocessor
I/O instructions that will also be ordered using the I and O bits in a FENCE.

The unused fields in the FENCE instruction, imm[11:8], rs1, and rd, are reserved for finer-grain
fences in future extensions. For forward compatibility, base implementations shall ignore these
fields, and standard software shall zero these fields.

<!-- p.33 -->

       We chose a relaxed memory model to allow high performance from simple machine implementa-
       tions, however a completely relaxed memory model is too weak to support programming language
       memory models and so the memory model is being tightened.
           A relaxed memory model is also most compatible with likely future coprocessor or accelerator
       extensions. We separate out I/O ordering from memory R/W ordering to avoid unnecessary
       serialization within a device-driver hart and also to support alternative non-memory paths to
       control added coprocessors or I/O devices. Simple implementations may additionally ignore the
       predecessor and successor fields and always execute a conservative fence on all operations.

```text
  31                                 20 19          15 14              12 11           7 6                  0
              imm[11:0]                      rs1             funct3            rd              opcode
                 12                           5                 3               5                7
                  0                           0             FENCE.I             0            MISC-MEM
```

The FENCE.I instruction is used to synchronize the instruction and data streams. RISC-V does not
guarantee that stores to instruction memory will be made visible to instruction fetches on the same
RISC-V hart until a FENCE.I instruction is executed. A FENCE.I instruction only ensures that a
subsequent instruction fetch on a RISC-V hart will see any previous data stores already visible to
the same RISC-V hart. FENCE.I does not ensure that other RISC-V harts’ instruction fetches will
observe the local hart’s stores in a multiprocessor system. To make a store to instruction memory
visible to all RISC-V harts, the writing hart has to execute a data FENCE before requesting that
all remote RISC-V harts execute a FENCE.I.

The unused fields in the FENCE.I instruction, imm[11:0], rs1, and rd, are reserved for finer-grain
fences in future extensions. For forward compatibility, base implementations shall ignore these
fields, and standard software shall zero these fields.

       The FENCE.I instruction was designed to support a wide variety of implementations. A sim-
       ple implementation can flush the local instruction cache and the instruction pipeline when the
       FENCE.I is executed. A more complex implementation might snoop the instruction (data) cache
       on every data (instruction) cache miss, or use an inclusive unified private L2 cache to invalidate
       lines from the primary instruction cache when they are being written by a local store instruction.
       If instruction and data caches are kept coherent in this way, then only the pipeline needs to be
       flushed at a FENCE.I.
           We considered but did not include a “store instruction word” instruction (as in MAJC [30]).
       JIT compilers may generate a large trace of instructions before a single FENCE.I, and amor-
       tize any instruction cache snooping/invalidation overhead by writing translated instructions to
       memory regions that are known not to reside in the I-cache.

#### 2.8 Control and Status Register Instructions

SYSTEM instructions are used to access system functionality that might require privileged access
and are encoded using the I-type instruction format. These can be divided into two main classes:
those that atomically read-modify-write control and status registers (CSRs), and all other potentially privileged instructions. CSR instructions are described in this section, with the two other
user-level SYSTEM instructions described in the following section.

<!-- p.34 -->

     The SYSTEM instructions are defined to allow simpler implementations to always trap to a
     single software trap handler. More sophisticated implementations might execute more of each
     system instruction in hardware.

CSR Instructions

We define the full set of CSR instructions here, although in the standard user-level base ISA, only
a handful of read-only counter CSRs are accessible.

```text
       31                             20 19           15 14   12 11          7 6              0
                    csr                       rs1        funct3        rd           opcode
                     12                        5           3            5             7
                 source/dest                source      CSRRW         dest         SYSTEM
                 source/dest                source      CSRRS         dest         SYSTEM
                 source/dest                source      CSRRC         dest         SYSTEM
                 source/dest              uimm[4:0]     CSRRWI        dest         SYSTEM
                 source/dest              uimm[4:0]     CSRRSI        dest         SYSTEM
                 source/dest              uimm[4:0]     CSRRCI        dest         SYSTEM
```

The CSRRW (Atomic Read/Write CSR) instruction atomically swaps values in the CSRs and
integer registers. CSRRW reads the old value of the CSR, zero-extends the value to XLEN bits,
then writes it to integer register rd. The initial value in rs1 is written to the CSR. If rd=x0, then
the instruction shall not read the CSR and shall not cause any of the side-effects that might occur
on a CSR read.

The CSRRS (Atomic Read and Set Bits in CSR) instruction reads the value of the CSR, zeroextends the value to XLEN bits, and writes it to integer register rd. The initial value in integer
register rs1 is treated as a bit mask that specifies bit positions to be set in the CSR. Any bit that
is high in rs1 will cause the corresponding bit to be set in the CSR, if that CSR bit is writable.
Other bits in the CSR are unaffected (though CSRs might have side effects when written).

The CSRRC (Atomic Read and Clear Bits in CSR) instruction reads the value of the CSR, zeroextends the value to XLEN bits, and writes it to integer register rd. The initial value in integer
register rs1 is treated as a bit mask that specifies bit positions to be cleared in the CSR. Any bit
that is high in rs1 will cause the corresponding bit to be cleared in the CSR, if that CSR bit is
writable. Other bits in the CSR are unaffected.

For both CSRRS and CSRRC, if rs1=x0, then the instruction will not write to the CSR at all, and
so shall not cause any of the side effects that might otherwise occur on a CSR write, such as raising
illegal instruction exceptions on accesses to read-only CSRs. Note that if rs1 specifies a register
holding a zero value other than x0, the instruction will still attempt to write the unmodified value
back to the CSR and will cause any attendant side effects.

The CSRRWI, CSRRSI, and CSRRCI variants are similar to CSRRW, CSRRS, and CSRRC respectively, except they update the CSR using an XLEN-bit value obtained by zero-extending a 5-bit
unsigned immediate (uimm[4:0]) field encoded in the rs1 field instead of a value from an integer register. For CSRRSI and CSRRCI, if the uimm[4:0] field is zero, then these instructions will not
write to the CSR, and shall not cause any of the side effects that might otherwise occur on a CSR
write. For CSRRWI, if rd=x0, then the instruction shall not read the CSR and shall not cause any
of the side-effects that might occur on a CSR read.

<!-- p.35 -->

Some CSRs, such as the instructions retired counter, instret, may be modified as side effects
of instruction execution. In these cases, if a CSR access instruction reads a CSR, it reads the
value prior to the execution of the instruction. If a CSR access instruction writes a CSR, the
update occurs after the execution of the instruction. In particular, a value written to instret by
one instruction will be the value read by the following instruction (i.e., the increment of instret
caused by the first instruction retiring happens before the write of the new value).

The assembler pseudo-instruction to read a CSR, CSRR rd, csr, is encoded as CSRRS rd, csr, x0.
The assembler pseudo-instruction to write a CSR, CSRW csr, rs1, is encoded as CSRRW x0, csr,
rs1, while CSRWI csr, uimm, is encoded as CSRRWI x0, csr, uimm.

Further assembler pseudo-instructions are defined to set and clear bits in the CSR when the old
value is not required: CSRS/CSRC csr, rs1; CSRSI/CSRCI csr, uimm.

Timers and Counters

```text
       31                             20 19         15 14   12 11           7 6             0
                  csr                         rs1      funct3        rd            opcode
                   12                          5         3            5              7
               RDCYCLE[H]                      0      CSRRS         dest          SYSTEM
                RDTIME[H]                      0      CSRRS         dest          SYSTEM
              RDINSTRET[H]                     0      CSRRS         dest          SYSTEM
```

RV32I provides a number of 64-bit read-only user-level counters, which are mapped into the 12-bit
CSR address space and accessed in 32-bit pieces using CSRRS instructions.

The RDCYCLE pseudo-instruction reads the low XLEN bits of the cycle CSR which holds a count
of the number of clock cycles executed by the processor core on which the hart is running from an
arbitrary start time in the past. RDCYCLEH is an RV32I-only instruction that reads bits 63–32 of
the same cycle counter. The underlying 64-bit counter should never overflow in practice. The rate
at which the cycle counter advances will depend on the implementation and operating environment.
The execution environment should provide a means to determine the current rate (cycles/second)
at which the cycle counter is incrementing.

The RDTIME pseudo-instruction reads the low XLEN bits of the time CSR, which counts wall-clock
real time that has passed from an arbitrary start time in the past. RDTIMEH is an RV32I-only instruction that reads bits 63–32 of the same real-time counter. The underlying 64-bit counter should
never overflow in practice. The execution environment should provide a means of determining the
period of the real-time counter (seconds/tick). The period must be constant. The real-time clocks
of all harts in a single user application should be synchronized to within one tick of the real-time
clock. The environment should provide a means to determine the accuracy of the clock.

The RDINSTRET pseudo-instruction reads the low XLEN bits of the instret CSR, which counts the number of instructions retired by this hart from some arbitrary start point in the past.
RDINSTRETH is an RV32I-only instruction that reads bits 63–32 of the same instruction counter.
The underlying 64-bit counter that should never overflow in practice.

<!-- p.36 -->

The following code sequence will read a valid 64-bit cycle counter value into x3:x2, even if the
counter overflows between reading its upper and lower halves.
      again:
          rdcycleh          x3
          rdcycle           x2
          rdcycleh          x4

          bne               x3, x4, again

**Figure 2.5:** Sample code for reading the 64-bit cycle counter in RV32.

      We mandate these basic counters be provided in all implementations as they are essential for
      basic performance analysis, adaptive and dynamic optimization, and to allow an application to
      work with real-time streams. Additional counters should be provided to help diagnose performance
      problems and these should be made accessible from user-level application code with low overhead.
          We required the counters be 64 bits wide, even on RV32, as otherwise it is very difficult for
      software to determine if values have overflowed. For a low-end implementation, the upper 32
      bits of each counter can be implemented using software counters incremented by a trap handler
      triggered by overflow of the lower 32 bits. The sample code described above shows how the full
      64-bit width value can be safely read using the individual 32-bit instructions.
          In some applications, it is important to be able to read multiple counters at the same instant
      in time. When run under a multitasking environment, a user thread can suffer a context switch
      while attempting to read the counters. One solution is for the user thread to read the real-time
      counter before and after reading the other counters to determine if a context switch occurred in
      the middle of the sequence, in which case the reads can be retried. We considered adding output
      latches to allow a user thread to snapshot the counter values atomically, but this would increase
      the size of the user context, especially for implementations with a richer set of counters.

#### 2.9 Environment Call and Breakpoints

```text
       31                                 20 19          15 14    12 11            7 6                0
                   funct12                        rs1       funct3         rd             opcode
                      12                           5          3             5               7
                    ECALL                          0        PRIV            0            SYSTEM
                   EBREAK                          0        PRIV            0            SYSTEM
```

The ECALL instruction is used to make a request to the supporting execution environment, which is
usually an operating system. The ABI for the system will define how parameters for the environment
request are passed, but usually these will be in defined locations in the integer register file.

The EBREAK instruction is used by debuggers to cause control to be transferred back to a debugging environment.

      ECALL and EBREAK were previously named SCALL and SBREAK. The instructions have the same functionality and encoding, but were renamed to reflect that they can be used more
     generally than to call a supervisor-level operating system or debugger.

<!-- p.39 -->

### 第 3 章 RV32E Base Integer Instruction Set, Version 1.9

This chapter describes the RV32E base integer instruction set, which is a reduced version of RV32I
designed for embedded systems. The main change is to reduce the number of integer registers to 16,
and to remove the counters that are mandatory in RV32I. This chapter only outlines the differences
between RV32E and RV32I, and so should be read after Chapter 2.

      RV32E was designed to provide an even smaller base core for embedded microcontrollers. Al-
      though we had mentioned this possibility in version 2.0 of this document, we initially resisted
      defining this subset. However, given the demand for the smallest possible 32-bit microcontroller,
      and in the interests of preempting fragmentation in this space, we have now defined RV32E as
      a fourth standard base ISA in addition to RV32I, RV64I, and RV128I. The E variant is only
      standardized for the 32-bit address space width.

#### 3.1 RV32E Programmers’ Model

RV32E reduces the integer register count to 16 general-purpose registers, (x0–x15), where x0 is a
dedicated zero register.

      We have found that in the small RV32I core designs, the upper 16 registers consume around one
      quarter of the total area of the core excluding memories, thus their removal saves around 25%
      core area with a corresponding core power reduction.

      This change requires a different calling convention and ABI. In particular, RV32E is only used
      with a soft-float calling convention. Systems with hardware floating-point must use an I base.

#### 3.2 RV32E Instruction Set

RV32E uses the same instruction set encoding as RV32I, except that use of register specifiers
x16–x31 in an instruction will result in an illegal instruction exception being raised.

<!-- p.40 -->

      Any future standard extensions will not make use of the instruction bits freed up by the reduced
      register-specifier fields and so these are available for non-standard extensions.

A further simplification is that the counter instructions (rdcycle[h],rdtime[h], rdinstret[h])
are no longer mandatory.

      The mandatory counters require additional registers and logic, and can be replaced with more
      application-specific facilities.

#### 3.3 RV32E Extensions

RV32E can be extended with the M, A, and C user-level standard extensions.

      We do not intend to support hardware floating-point with the RV32E subset. The savings from
      reduced register count become negligible in the context of a hardware floating-point unit, and we
      wish to reduce the proliferation of ABIs.

The privileged architecture of an RV32E system can include user mode as well as machine mode,
and the physical memory protection scheme described in Volume II.

      We do not intend to support full Unix-style operating systems with the RV32E subset. The
      savings from reduced register count become negligible in the context of an OS-capable core, and
      we wish to avoid OS fragmentation.

<!-- p.41 -->

### 第 4 章 RV64I Base Integer Instruction Set, Version 2.0

This chapter describes the RV64I base integer instruction set, which builds upon the RV32I variant
described in Chapter 2. This chapter presents only the differences with RV32I, so should be read
in conjunction with the earlier chapter.

#### 4.1 Register State

RV64I widens the integer registers and supported user address space to 64 bits (XLEN=64 in
Figure 2.1).

#### 4.2 Integer Computational Instructions

Additional instruction variants are provided to manipulate 32-bit values in RV64I, indicated by a
‘W’ suffix to the opcode. These “*W” instructions ignore the upper 32 bits of their inputs and
always produce 32-bit signed values, i.e. bits XLEN-1 through 31 are equal. They cause an illegal
instruction exception in RV32I.

      The compiler and calling convention maintain an invariant that all 32-bit values are held in a
      sign-extended format in 64-bit registers. Even 32-bit unsigned integers extend bit 31 into bits 63
      through 32. Consequently, conversion between unsigned and signed 32-bit integers is a no-op,
      as is conversion from a signed 32-bit integer to a signed 64-bit integer. Existing 64-bit wide
      SLTU and unsigned branch compares still operate correctly on unsigned 32-bit integers under
      this invariant. Similarly, existing 64-bit wide logical operations on 32-bit sign-extended integers
      preserve the sign-extension property. A few new instructions (ADD[I]W/SUBW/SxxW) are
      required for addition and shifts to ensure reasonable performance for 32-bit values.

<!-- p.42 -->

Integer Register-Immediate Instructions

```text
  31                                20 19               15 14                  12 11             7 6               0
                imm[11:0]                   rs1                  funct3                  rd              opcode
                   12                        5                      3                     5                7
             I-immediate[11:0]              src                 ADDIW                   dest           OP-IMM-32
```

ADDIW is an RV64I-only instruction that adds the sign-extended 12-bit immediate to register rs1
and produces the proper sign-extension of a 32-bit result in rd. Overflows are ignored and the
result is the low 32 bits of the result sign-extended to 64 bits. Note, ADDIW rd, rs1, 0 writes the
sign-extension of the lower 32 bits of register rs1 into register rd (assembler pseudo-op SEXT.W).

```text
  31             26   25     24         20 19               15 14              12 11             7 6               0
     imm[11:6]     imm[5]      imm[4:0]           rs1                funct3              rd              opcode
         6            1           5                5                    3                 5                7
       000000     shamt[5]    shamt[4:0]          src                 SLLI              dest            OP-IMM
       000000     shamt[5]    shamt[4:0]          src                 SRLI              dest            OP-IMM
       010000     shamt[5]    shamt[4:0]          src                 SRAI              dest            OP-IMM
       000000       0         shamt[4:0]          src               SLLIW               dest           OP-IMM-32
       000000       0         shamt[4:0]          src               SRLIW               dest           OP-IMM-32
       010000       0         shamt[4:0]          src               SRAIW               dest           OP-IMM-32
```

Shifts by a constant are encoded as a specialization of the I-type format using the same instruction
opcode as RV32I. The operand to be shifted is in rs1, and the shift amount is encoded in the lower
6 bits of the I-immediate field for RV64I. The right shift type is encoded in bit 30. SLLI is a
logical left shift (zeros are shifted into the lower bits); SRLI is a logical right shift (zeros are shifted
into the upper bits); and SRAI is an arithmetic right shift (the original sign bit is copied into the
vacated upper bits). For RV32I, SLLI, SRLI, and SRAI generate an illegal instruction exception if
imm[5] 6= 0.

SLLIW, SRLIW, and SRAIW are RV64I-only instructions that are analogously defined but operate
on 32-bit values and produce signed 32-bit results. SLLIW, SRLIW, and SRAIW generate an illegal
instruction exception if imm[5] 6= 0.

```text
        31                                                             12 11               7 6                0
                              imm[31:12]                                          rd               opcode
                                  20                                               5                 7
                           U-immediate[31:12]                                    dest               LUI
                           U-immediate[31:12]                                    dest              AUIPC
```

LUI (load upper immediate) uses the same opcode as RV32I. LUI places the 20-bit U-immediate
into bits 31–12 of register rd and places zero in the lowest 12 bits. The 32-bit result is sign-extended
to 64 bits.

AUIPC (add upper immediate to pc) uses the same opcode as RV32I. AUIPC (add upper immediate
to pc) is used to build pc-relative addresses and uses the U-type format. AUIPC appends 12 loworder zero bits to the 20-bit U-immediate, sign-extends the result to 64 bits, then adds it to the pc
and places the result in register rd.

<!-- p.43 -->

Integer Register-Register Operations

```text
   31                  25 24           20 19            15 14             12 11             7 6                 0
         funct7                  rs2             rs1         funct3                 rd                opcode
            7                     5               5             3                    5                  7
         0000000                src2            src1        SLL/SRL                dest                OP
         0100000                src2            src1          SRA                  dest                OP
         0000000                src2            src1         ADDW                  dest               OP-32
         0000000                src2            src1      SLLW/SRLW                dest               OP-32
         0100000                src2            src1      SUBW/SRAW                dest               OP-32
```

ADDW and SUBW are RV64I-only instructions that are defined analogously to ADD and SUB
but operate on 32-bit values and produce signed 32-bit results. Overflows are ignored, and the low
32-bits of the result is sign-extended to 64-bits and written to the destination register.

SLL, SRL, and SRA perform logical left, logical right, and arithmetic right shifts on the value
in register rs1 by the shift amount held in register rs2. In RV64I, only the low 6 bits of rs2 are
considered for the shift amount.

SLLW, SRLW, and SRAW are RV64I-only instructions that are analogously defined but operate
on 32-bit values and produce signed 32-bit results. The shift amount is given by rs2[4:0].

#### 4.3 Load and Store Instructions

RV64I extends the address space to 64 bits. The execution environment will define what portions
of the address space are legal to access.

```text
  31                                    20 19            15 14             12 11                7 6                 0
               imm[11:0]                          rs1            funct3               rd               opcode
                    12                             5                3                  5                 7
               offset[11:0]                      base            width               dest              LOAD
  31                    25 24           20 19            15 14             12 11                7 6                 0
        imm[11:5]                rs2              rs1            funct3           imm[4:0]             opcode
             7                    5                5                3                  5                 7
        offset[11:5]             src             base            width            offset[4:0]         STORE
```

The LD instruction loads a 64-bit value from memory into register rd for RV64I.

The LW instruction loads a 32-bit value from memory and sign-extends this to 64 bits before storing
it in register rd for RV64I. The LWU instruction, on the other hand, zero-extends the 32-bit value from memory for RV64I. LH and LHU are defined analogously for 16-bit values, as are LB and
LBU for 8-bit values. The SD, SW, SH, and SB instructions store 64-bit, 32-bit, 16-bit, and 8-bit
values from the low bits of register rs2 to memory respectively.

<!-- p.44 -->

#### 4.4 System Instructions

In RV64I, the CSR instructions can manipulate 64-bit CSRs. In particular, the RDCYCLE, RDTIME, and RDINSTRET pseudo-instructions read the full 64 bits of the cycle, time, and instret
counters. Hence, the RDCYCLEH, RDTIMEH, and RDINSTRETH instructions are not necessary
and are illegal in RV64I.

<!-- p.45 -->

### 第 5 章 RV128I Base Integer Instruction Set, Version 1.7

     “There is only one mistake that can be made in computer design that is difficult to re-
     cover from—not having enough address bits for memory addressing and memory man-
     agement.” Bell and Strecker, ISCA-3, 1976.

This chapter describes RV128I, a variant of the RISC-V ISA supporting a flat 128-bit address space.
The variant is a straightforward extrapolation of the existing RV32I and RV64I designs.

     The primary reason to extend integer register width is to support larger address spaces. It is
     not clear when a flat address space larger than 64 bits will be required. At the time of writing,
     the fastest supercomputer in the world as measured by the Top500 benchmark had over 1 PB
     of DRAM, and would require over 50 bits of address space if all the DRAM resided in a single
     address space. Some warehouse-scale computers already contain even larger quantities of DRAM,
     and new dense solid-state non-volatile memories and fast interconnect technologies might drive a
     demand for even larger memory spaces. Exascale systems research is targeting 100 PB memory
     systems, which occupy 57 bits of address space. At historic rates of growth, it is possible that
     greater than 64 bits of address space might be required before 2030.
         History suggests that whenever it becomes clear that more than 64 bits of address space is
     needed, architects will repeat intensive debates about alternatives to extending the address space,
     including segmentation, 96-bit address spaces, and software workarounds, until, finally, flat 128-
     bit address spaces will be adopted as the simplest and best solution.
         We have not frozen the RV128 spec at this time, as there might be need to evolve the design
     based on actual usage of 128-bit address spaces.
RV128I builds upon RV64I in the same way RV64I builds upon RV32I, with integer registers
extended to 128 bits (i.e., XLEN=128). Most integer computational instructions are unchanged as
they are defined to operate on XLEN bits. The RV64I “*W” integer instructions that operate on
32-bit values in the low bits of a register are retained, and a new set of “*D” integer instructions
that operate on 64-bit values held in the low bits of the 128-bit integer registers are added. The
“*D” instructions consume two major opcodes (OP-IMM-64 and OP-64) in the standard 32-bit
encoding.

Shifts by an immediate (SLLI/SRLI/SRAI) are now encoded using the low 7 bits of the I-immediate,
and variable shifts (SLL/SRL/SRA) use the low 7 bits of the shift amount source register.

<!-- p.46 -->

A LDU (load double unsigned) instruction is added using the existing LOAD major opcode, along
with new LQ and SQ instructions to load and store quadword values. SQ is added to the STORE
major opcode, while LQ is added to the MISC-MEM major opcode.

The floating-point instruction set is unchanged, although the 128-bit Q floating-point extension can
now support FMV.X.Q and FMV.Q.X instructions, together with additional FCVT instructions to
and from the T (128-bit) integer format.

<!-- p.47 -->

### 第 6 章 “M” Standard Extension for Integer Multiplication and Division, Version 2.0

This chapter describes the standard integer multiplication and division instruction extension, which
is named “M” and contains instructions that multiply or divide values held in two integer registers.

        We separate integer multiply and divide out from the base to simplify low-end implementations,
        or for applications where integer multiply and divide operations are either infrequent or better
        handled in attached accelerators.

#### 6.1 Multiplication Operations

```text
   31               25 24             20 19        15 14             12 11           7 6                   0
        funct7             rs2              rs1         funct3       rd                      opcode
           7                 5               5             3          5                        7
        MULDIV           multiplier     multiplicand MUL/MULH[[S]U] dest                      OP
        MULDIV           multiplier     multiplicand   MULW         dest                     OP-32
```

MUL performs an XLEN-bit×XLEN-bit multiplication and places the lower XLEN bits in the
destination register. MULH, MULHU, and MULHSU perform the same multiplication but return
the upper XLEN bits of the full 2×XLEN-bit product, for signed×signed, unsigned×unsigned, and
signed×unsigned multiplication respectively. If both the high and low bits of the same product
are required, then the recommended code sequence is: `MULH[[S]U] rdh, rs1, rs2` ; `MUL rdl, rs1, rs2`
(source register specifiers must be in same order and rdh cannot be the same as rs1 or rs2).
Microarchitectures can then fuse these into a single multiply operation instead of performing two
separate multiplies.

MULW is only valid for RV64, and multiplies the lower 32 bits of the source registers, placing the
sign-extension of the lower 32 bits of the result into the destination register. MUL can be used to obtain the upper 32 bits of the 64-bit product, but signed arguments must be proper 32-bit signed
values, whereas unsigned arguments must have their upper 32 bits clear.

<!-- p.48 -->

#### 6.2 Division Operations

```text
  31              25 24              20 19          15 14              12 11            7 6                  0
       funct7               rs2             rs1            funct3               rd              opcode
          7                  5               5                3                  5                7
       MULDIV             divisor        dividend      DIV[U]/REM[U]           dest              OP
       MULDIV             divisor        dividend     DIV[U]W/REM[U]W          dest             OP-32
```

DIV and DIVU perform signed and unsigned integer division of XLEN bits by XLEN bits. REM
and REMU provide the remainder of the corresponding division operation. If both the quotient
and remainder are required from the same division, the recommended code sequence is: DIV[U]
rdq, rs1, rs2; REM[U] rdr, rs1, rs2 (rdq cannot be the same as rs1 or rs2). Microarchitectures can
then fuse these into a single divide operation instead of performing two separate divides.

The semantics for division by zero and division overflow are summarized in Table 6.1. The quotient
of division by zero has all bits set, i.e. 2XLEN − 1 for unsigned division or −1 for signed division.
The remainder of division by zero equals the dividend. Signed division overflow occurs only when
the most-negative integer, −2XLEN −1 , is divided by −1. The quotient of signed division overflow
is equal to the dividend, and the remainder is zero. Unsigned division overflow cannot occur.

```text
     Condition                       Dividend   Divisor       DIVU         REMU           DIV         REM
     Division by zero                   x          0        2XLEN − 1        x            −1           x
     Overflow (signed only)         −2XLEN −1     −1            –            –        −2XLEN −1        0
```

**Table 6.1:** Semantics for division by zero and division overflow.

       We considered raising exceptions on integer divide by zero, with these exceptions causing a trap in
       most execution environments. However, this would be the only arithmetic trap in the standard
       ISA (floating-point exceptions set flags and write default values, but do not cause traps) and
       would require language implementors to interact with the execution environment’s trap handlers
       for this case. Further, where language standards mandate that a divide-by-zero exception must
       cause an immediate control flow change, only a single branch instruction needs to be added to
       each divide operation, and this branch instruction can be inserted after the divide and should
       normally be very predictably not taken, adding little runtime overhead.
           The value of all bits set is returned for both unsigned and signed divide by zero to simplify
       the divider circuitry. The value of all 1s is both the natural value to return for unsigned divide,
       representing the largest unsigned number, and also the natural result for simple unsigned divider
       implementations. Signed division is often implemented using an unsigned division circuit and
       specifying the same overflow result simplifies the hardware.

DIVW and DIVUW instructions are only valid for RV64, and divide the lower 32 bits of rs1 by
the lower 32 bits of rs2, treating them as signed and unsigned integers respectively, placing the
32-bit quotient in rd, sign-extended to 64 bits. REMW and REMUW instructions are only valid
for RV64, and provide the corresponding signed and unsigned remainder operations respectively.

<!-- p.49 -->

Both REMW and REMUW always sign-extend the 32-bit result to 64 bits, including on a divide
by zero.

<!-- p.51 -->

### 第 7 章 “A” Standard Extension for Atomic Instructions, Version 2.0

      This section is somewhat out of date as the RISC-V memory model is currently under revision
      to ensure it can efficiently support current programming language memory models. The revised
      base memory model will contain further ordering constraints, including at least that loads to
      the same address from the same hart cannot be reordered, and that syntactic data dependencies
      between instructions are respected.

The standard atomic instruction extension is denoted by instruction subset name “A”, and contains instructions that atomically read-modify-write memory to support synchronization between
multiple RISC-V harts running in the same memory space. The two forms of atomic instruction
provided are load-reserved/store-conditional instructions and atomic fetch-and-op memory instructions. Both types of atomic instruction support various memory consistency orderings including
unordered, acquire, release, and sequentially consistent semantics. These instructions allow RISC-V
to support the RCsc memory consistency model [10].

      After much debate, the language community and architecture community appear to have finally
      settled on release consistency as the standard memory consistency model and so the RISC-V
      atomic support is built around this model.

#### 7.1 Specifying Ordering of Atomic Instructions

The base RISC-V ISA has a relaxed memory model, with the FENCE instruction used to impose
additional ordering constraints. The address space is divided by the execution environment into
memory and I/O domains, and the FENCE instruction provides options to order accesses to one
or both of these two address domains.

To provide more efficient support for release consistency [10], each atomic instruction has two bits,
aq and rl, used to specify additional memory ordering constraints as viewed by other RISC-V harts.
The bits order accesses to one of the two address domains, memory or I/O, depending on which
address domain the atomic instruction is accessing. No ordering constraint is implied to accesses
to the other domain, and a FENCE instruction should be used to order across both domains.

<!-- p.52 -->

If both bits are clear, no additional ordering constraints are imposed on the atomic memory operation. If only the aq bit is set, the atomic memory operation is treated as an acquire access,
i.e., no following memory operations on this RISC-V hart can be observed to take place before
the acquire memory operation. If only the rl bit is set, the atomic memory operation is treated
as a release access, i.e., the release memory operation can not be observed to take place before
any earlier memory operations on this RISC-V hart. If both the aq and rl bits are set, the atomic
memory operation is sequentially consistent and cannot be observed to happen before any earlier
memory operations or after any later memory operations in the same RISC-V hart, and can only be
observed by any other hart in the same global order of all sequentially consistent atomic memory
operations to the same address domain.

          Theoretically, the definition of the aq and rl bits allows for implementations without global store
          atomicity. When both aq and rl bits are set, however, we require full sequential consistency for
          the atomic operation which implies global store atomicity in addition to both acquire and release
          semantics. In practice, hardware systems are usually implemented with global store atomicity,
          embodied in local processor ordering rules together with single-writer cache coherence protocols.

#### 7.2 Load-Reserved/Store-Conditional Instructions

```text
     31            27   26    25   24          20 19           15 14    12 11            7 6                    0
          funct5        aq    rl        rs2             rs1       funct3         rd               opcode
             5           1    1          5               5          3             5                 7
            LR          ordering         0             addr       width         dest              AMO
            SC          ordering        src            addr       width         dest              AMO
```

Complex atomic memory operations on a single memory word are performed with the load-reserved
(LR) and store-conditional (SC) instructions. LR loads a word from the address in rs1, places the
sign-extended value in rd, and registers a reservation on the memory address. SC writes a word in
rs2 to the address in rs1, provided a valid reservation still exists on that address. SC writes zero
to rd on success or a nonzero code on failure.

          Both compare-and-swap (CAS) and LR/SC can be used to build lock-free data structures. After
          extensive discussion, we opted for LR/SC for several reasons: 1) CAS suffers from the ABA
          problem, which LR/SC avoids because it monitors all accesses to the address rather than only
          checking for changes in the data value; 2) CAS would also require a new integer instruction for-
          mat to support three source operands (address, compare value, swap value) as well as a different
          memory system message format, which would complicate microarchitectures; 3) Furthermore,
          to avoid the ABA problem, other systems provide a double-wide CAS (DW-CAS) to allow a
          counter to be tested and incremented along with a data word. This requires reading five regis-
          ters and writing two in one instruction, and also a new larger memory system message type,
          further complicating implementations; 4) LR/SC provides a more efficient implementation of
          many primitives as it only requires one load as opposed to two with CAS (one load before the
          CAS instruction to obtain a value for speculative computation, then a second load as part of the
          CAS instruction to check if value is unchanged before updating).
              The main disadvantage of LR/SC over CAS is livelock, which we avoid with an architected
          guarantee of eventual forward progress as described below. Another concern is whether the influ-
          ence of the current x86 architecture, with its DW-CAS, will complicate porting of synchronization libraries and other software that assumes DW-CAS is the basic machine primitive. A possible
      mitigating factor is the recent addition of transactional memory instructions to x86, which might
      cause a move away from DW-CAS.
The failure code with value 1 is reserved to encode an unspecified failure. Other failure codes are
reserved at this time, and portable software should only assume the failure code will be non-zero.
LR and SC operate on naturally-aligned 64-bit (RV64 only) or 32-bit words in memory. Misaligned
addresses will generate misaligned address exceptions.

<!-- p.53 -->

      We reserve a failure code of 1 to mean “unspecified” so that simple implementations may return
      this value using the existing mux required for the SLT/SLTU instructions. More specific failure
      codes might be defined in future versions or extensions to the ISA.
In the standard A extension, certain constrained LR/SC sequences are guaranteed to succeed
eventually. The static code for the LR/SC sequence plus the code to retry the sequence in case
of failure must comprise at most 16 integer instructions placed sequentially in memory. For the
sequence to be guaranteed to eventually succeed, the dynamic code executed between the LR and
SC instructions can only contain other instructions from the base “I” subset, excluding loads, stores,
backward jumps or taken backward branches, FENCE, FENCE.I, and SYSTEM instructions. The
code to retry a failing LR/SC sequence can contain backward jumps and/or branches to repeat the
LR/SC sequence, but otherwise has the same constraints. The SC must be to the same address as
the latest LR executed. LR/SC sequences that do not meet these constraints might complete on
some attempts on some implementations, but there is no guarantee of eventual success.

      One advantage of CAS is that it guarantees that some hart eventually makes progress, whereas
      an LR/SC atomic sequence could livelock indefinitely on some systems. To avoid this concern,
      we added an architectural guarantee of forward progress to LR/SC atomic sequences. The re-
      strictions on LR/SC sequence contents allows an implementation to capture a cache line on the
      LR and complete the LR/SC sequence by holding off remote cache interventions for a bounded
      short time. Interrupts and TLB misses might cause the reservation to be lost, but eventually
      the atomic sequence can complete. We restricted the length of LR/SC sequences to fit within
      64 contiguous instruction bytes in the base ISA to avoid undue restrictions on instruction cache
      and TLB size and associativity. Similarly, we disallowed other loads and stores within the se-
      quences to avoid restrictions on data cache associativity. The restrictions on branches and jumps
      limits the time that can be spent in the sequence. Floating-point operations and integer multi-
      ply/divide were disallowed to simplify the operating system’s emulation of these instructions on
      implementations lacking appropriate hardware support.
An implementation can reserve an arbitrary subset of the memory space on each LR and multiple
LR reservations might be active simultaneously for a single hart. An SC can succeed if no accesses
from other harts to the address can be observed to have occurred between the SC and the last LR
in this hart to reserve the address. Note this LR might have had a different address argument, but
reserved the SC’s address as part of the memory subset. Following this model, in systems with
memory translation, an SC is allowed to succeed if the earlier LR reserved the same location using
an alias with a different virtual address, but is also allowed to fail if the virtual address is different.
The SC must fail if there is an observable memory access from another hart to the address, or
if there is an intervening context switch on this hart, or if in the meantime the hart executed a
privileged exception-return instruction.

      The specification explicitly allows implementations to support more powerful implementations
      with wider guarantees, provided they do not void the atomicity guarantees for the constrained
      sequences.

<!-- p.54 -->

LR/SC can be used to construct lock-free data structures. An example using LR/SC to implement
a compare-and-swap function is shown in Figure 7.1. If inlined, compare-and-swap functionality
need only take three instructions.

```asm
         # a0 holds address of memory location
         # a1 holds expected value
         # a2 holds desired value
         # a0 holds return value, 0 if successful, !0 otherwise
```

     cas:
         lr.w t0, (a0)        # Load original value.
         bne t0, a1, fail     # Doesn’t match, so fail.
         sc.w a0, a2, (a0)    # Try to update.
         jr ra                # Return.
     fail:
         li a0, 1             # Set return to failure.
         jr ra                # Return.

**Figure 7.1:** Sample code for compare-and-swap function using LR/SC.

An SC instruction can never be observed by another RISC-V hart before the immediately preceding
LR. Due to the atomic nature of the LR/SC sequence, no memory operations from any hart can be
observed to have occurred between the LR and a successful SC. The LR/SC sequence can be given
acquire semantics by setting the aq bit on the SC instruction. The LR/SC sequence can be given
release semantics by setting the rl bit on the LR instruction. Setting both aq and rl bits on the LR
instruction, and setting the aq bit on the SC instruction makes the LR/SC sequence sequentially
consistent with respect to other sequentially consistent atomic operations.

If neither bit is set on both LR and SC, the LR/SC sequence can be observed to occur before or
after surrounding memory operations from the same RISC-V hart. This can be appropriate when
the LR/SC sequence is used to implement a parallel reduction operation.

     In general, a multi-word atomic primitive is desirable but there is still considerable debate about
     what form this should take, and guaranteeing forward progress adds complexity to a system. Our
     current thoughts are to include a small limited-capacity transactional memory buffer along the
     lines of the original transactional memory proposals as an optional standard extension “T”.

<!-- p.55 -->

#### 7.3 Atomic Memory Operations

```text
   31                  27   26   25   24          20 19          15 14    12 11            7 6             0
     funct5     aq    rl                   rs2             rs1      funct3         rd            opcode
        5        1    1                     5               5         3             5              7
  AMOSWAP.W/D ordering                     src            addr      width         dest           AMO
   AMOADD.W/D ordering                     src            addr      width         dest           AMO
   AMOAND.W/D ordering                     src            addr      width         dest           AMO
   AMOOR.W/D ordering                      src            addr      width         dest           AMO
   AMOXOR.W/D ordering                     src            addr      width         dest           AMO
  AMOMAX[U].W/D ordering                   src            addr      width         dest           AMO
  AMOMIN[U].W/D ordering                   src            addr      width         dest           AMO
```

The atomic memory operation (AMO) instructions perform read-modify-write operations for multiprocessor synchronization and are encoded with an R-type instruction format. These AMO instructions atomically load a data value from the address in rs1, place the value into register rd,
apply a binary operator to the loaded value and the original value in rs2, then store the result back
to the address in rs1. AMOs can either operate on 64-bit (RV64 only) or 32-bit words in memory.
For RV64, 32-bit AMOs always sign-extend the value placed in rd. The address held in rs1 must be
naturally aligned to the size of the operand (i.e., eight-byte aligned for 64-bit words and four-byte
aligned for 32-bit words). If the address is not naturally aligned, a misaligned address exception
will be generated.

The operations supported are swap, integer add, logical AND, logical OR, logical XOR, and signed
and unsigned integer maximum and minimum. Without ordering constraints, these AMOs can
be used to implement parallel reduction operations, where typically the return value would be
discarded by writing to x0.

        We provided fetch-and-op style atomic primitives as they scale to highly parallel systems better
        than LR/SC or CAS. A simple microarchitecture can implement AMOs using the LR/SC prim-
        itives. More complex implementations might also implement AMOs at memory controllers, and
        can optimize away fetching the original value when the destination is x0.
             The set of AMOs was chosen to support the C11/C++11 atomic memory operations effi-
        ciently, and also to support parallel reductions in memory. Another use of AMOs is to provide
        atomic updates to memory-mapped device registers (e..g, setting, clearing, or toggling bits) in
        the I/O space.

To help implement multiprocessor synchronization, the AMOs optionally provide release consistency semantics. If the aq bit is set, then no later memory operations in this RISC-V hart can be
observed to take place before the AMO. Conversely, if the rl bit is set, then other RISC-V harts
will not observe the AMO before memory accesses preceding the AMO in this RISC-V hart.

        The AMOs were designed to implement the C11 and C++11 memory models efficiently. Al-
        though the FENCE R, RW instruction suffices to implement the acquire operation and FENCE
        RW, W suffices to implement release, both imply additional unnecessary ordering as compared
        to AMOs with the corresponding aq or rl bit set.

An example code sequence for a critical section guarded by a test-and-set spinlock is shown in
Figure 7.2. Note the first AMO is marked aq to order the lock acquisition before the critical section,
and the second AMO is marked rl to order the critical section before the lock relinquishment.

<!-- p.56 -->

         li           t0, 1                  # Initialize swap value.
     again:
         amoswap.w.aq t0, t0, (a0)           # Attempt to acquire lock.
         bnez         t0, again              # Retry if held.

```asm
         # ...
         # Critical section.
         # ...
```

         amoswap.w.rl x0, x0, (a0)           # Release lock by storing 0.

**Figure 7.2:** Sample code for mutual exclusion. a0 contains the address of the lock.

     We recommend the use of the AMO Swap idiom shown above for both lock acquire and release
     to simplify the implementation of speculative lock elision [25].
         At the risk of complicating the implementation of atomic operations, microarchitectures can
     elide the store within the acquire swap if the lock value matches the swap value, to avoid dirtying
     a cache line held in a shared or exclusive clean state. The effect is similar to a test-and-test-
     and-set lock but with shorter code paths.

The instructions in the “A” extension can also be used to provide sequentially consistent loads and
stores. A sequentially consistent load can be implemented as an LR with both aq and rl set. A
sequentially consistent store can be implemented as an AMOSWAP that writes the old value to x0
and has both aq and rl set.

<!-- p.57 -->

### 第 8 章 “F” Standard Extension for Single-Precision Floating-Point, Version 2.0

This chapter describes the standard instruction-set extension for single-precision floating-point,
which is named “F” and adds single-precision floating-point computational instructions compliant
with the IEEE 754-2008 arithmetic standard [14].

#### 8.1 F Register State

The F extension adds 32 floating-point registers, f0–f31, each 32 bits wide, and a floating-point
control and status register fcsr, which contains the operating mode and exception status of the
floating-point unit. This additional state is shown in Figure 8.1. We use the term FLEN to
describe the width of the floating-point registers in the RISC-V ISA, and FLEN=32 for the F
single-precision floating-point extension. Most floating-point instructions operate on values in the
floating-point register file. Floating-point load and store instructions transfer floating-point values
between registers and memory. Instructions to transfer values to and from the integer register file
are also provided.

      We considered a unified register file for both integer and floating-point values as this simplifies
      software register allocation and calling conventions, and reduces total user state. However,
      a split organization increases the total number of registers accessible with a given instruction
      width, simplifies provision of enough regfile ports for wide superscalar issue, supports decoupled
      floating-point-unit architectures, and simplifies use of internal floating-point encoding techniques.
      Compiler support and calling conventions for split register file architectures are well understood,
      and using dirty bits on floating-point register file state can reduce context-switch overhead.

<!-- p.58 -->

                             FLEN-1                       0
                                          f0
                                          f1
                                          f2
                                          f3
                                          f4
                                          f5
                                          f6
                                          f7
                                          f8
                                          f9
                                         f10
                                         f11
                                         f12
                                         f13
                                         f14
                                         f15
                                         f16
                                         f17
                                         f18
                                         f19
                                         f20
                                         f21
                                         f22
                                         f23
                                         f24
                                         f25
                                         f26
                                         f27
                                         f28
                                         f29
                                         f30
                                         f31
                                         FLEN
                             31                           0
                                          fcsr
                                           32

**Figure 8.1:** RISC-V standard F extension single-precision floating-point state.

<!-- p.59 -->

```text
    31                                         87                      5   4     3     2     1     0
                     Reserved                    Rounding Mode (frm)       Accrued Exceptions (fflags)
                                                                           NV DZ OF UF             NX
                        24                                3                1     1     1     1      1
```

**Figure 8.2:** Floating-point control and status register.

#### 8.2 Floating-Point Control and Status Register

The floating-point control and status register, fcsr, is a RISC-V control and status register (CSR).
It is a 32-bit read/write register that selects the dynamic rounding mode for floating-point arithmetic operations and holds the accrued exception flags, as shown in Figure 8.2.

The fcsr register can be read and written with the FRCSR and FSCSR instructions, which are
assembler pseudo-ops built on the underlying CSR access instructions. FRCSR reads fcsr by
copying it into integer register rd. FSCSR swaps the value in fcsr by copying the original value
into integer register rd, and then writing a new value obtained from integer register rs1 into fcsr.

The fields within the fcsr can also be accessed individually through different CSR addresses, and
separate assembler pseudo-ops are defined for these accesses. The FRRM instruction reads the
Rounding Mode field frm and copies it into the least-significant three bits of integer register rd,
with zero in all other bits. FSRM swaps the value in frm by copying the original value into integer
register rd, and then writing a new value obtained from the three least-significant bits of integer
register rs1 into frm. FRFLAGS and FSFLAGS are defined analogously for the Accrued Exception
Flags field fflags. Additional pseudo-instructions FSRMI and FSFLAGSI swap values using an
immediate value instead of register rs1.

Bits 31–8 of the fcsr are reserved for other standard extensions, including the “L” standard
extension for decimal floating-point. If these extensions are not present, implementations shall
ignore writes to these bits and supply a zero value when read. Standard software should preserve
the contents of these bits.

Floating-point operations use either a static rounding mode encoded in the instruction, or a dynamic
rounding mode held in frm. Rounding modes are encoded as shown in Table 8.1. A value of 111 in
the instruction’s rm field selects the dynamic rounding mode held in frm. If frm is set to an invalid
value (101–111), any subsequent attempt to execute a floating-point operation with a dynamic
rounding mode will cause an illegal instruction trap. Some instructions that have the rm field are
nevertheless unaffected by the rounding mode; they should have their rm field set to RNE (000).

      The C99 language standard effectively mandates the provision of a dynamic rounding mode
      register.

<!-- p.60 -->

```text
        Rounding Mode       Mnemonic                               Meaning
             000            RNE          Round to Nearest, ties to Even
             001            RTZ          Round towards Zero
             010            RDN          Round Down (towards −∞)
             011            RUP          Round Up (towards +∞)
             100            RMM          Round to Nearest, ties to Max Magnitude
             101                         Invalid. Reserved for future use.
             110                         Invalid. Reserved for future use.
             111                         In instruction’s rm field, selects dynamic rounding mode;
                                         In Rounding Mode register, Invalid.
```

**Table 8.1:** Rounding mode encoding.

The accrued exception flags indicate the exception conditions that have arisen on any floating-point
arithmetic instruction since the field was last reset by software, as shown in Table 8.2.

```text
                                    Flag Mnemonic       Flag Meaning
                                         NV           Invalid Operation
                                         DZ            Divide by Zero
                                         OF               Overflow
                                         UF               Underflow
                                         NX                Inexact
```

**Table 8.2:** Accrued exception flag encoding.

      As allowed by the standard, we do not support traps on floating-point exceptions in the base
      ISA, but instead require explicit checks of the flags in software. We considered adding branches
      controlled directly by the contents of the floating-point accrued exception flags, but ultimately
      chose to omit these instructions to keep the ISA simple.

#### 8.3 NaN Generation and Propagation

Except when otherwise stated, if the result of a floating-point operation is NaN, it is the canonical
NaN. The canonical NaN has a positive sign and all significand bits clear except the MSB, a.k.a.
the quiet bit. For single-precision floating-point, this corresponds to the pattern 0x7fc00000.

For FMIN and FMAX, if at least one input is a signaling NaN, or if both inputs are quiet NaNs,
the result is the canonical NaN. If one operand is a quiet NaN and the other is not a NaN, the
result is the non-NaN operand.

The sign-injection instructions (FSGNJ, FSGNJN, FSGNJX) do not canonicalize NaNs; they manipulate the underlying bit patterns directly.

      We considered propagating NaN payloads, as is recommended by the standard, but this decision
      would have increased hardware cost. Moreover, since this feature is optional in the standard, it
      cannot be used in portable code.
          Implementors are free to provide a NaN payload propagation scheme as a nonstandard exten-
      sion enabled by a nonstandard operating mode. However, the canonical NaN scheme described
      above must always be supported and should be the default mode.

<!-- p.61 -->

      We require implementations to return the standard-mandated default values in the case of ex-
      ceptional conditions, without any further intervention on the part of user-level software (unlike
      the Alpha ISA floating-point trap barriers). We believe full hardware handling of exceptional
      cases will become more common, and so wish to avoid complicating the user-level ISA to opti-
      mize other approaches. Implementations can always trap to machine-mode software handlers to
      provide exceptional default values.

#### 8.4 Subnormal Arithmetic

Operations on subnormal numbers are handled in accordance with the IEEE 754-2008 standard.

In the parlance of the IEEE standard, tininess is detected after rounding.

      Detecting tininess after rounding results in fewer spurious underflow signals.

#### 8.5 Single-Precision Load and Store Instructions

Floating-point loads and stores use the same base+offset addressing mode as the integer base ISA,
with a base address in register rs1 and a 12-bit signed byte offset. The FLW instruction loads
a single-precision floating-point value from memory into floating-point register rd. FSW stores a
single-precision value from floating-point register rs2 to memory.

```text
      31                                20 19          15 14    12 11              7 6                0
                  imm[11:0]                      rs1       width         rd                opcode
                       12                         5          3            5                  7
                  offset[11:0]                  base        W           dest             LOAD-FP
      31                  25 24         20 19          15 14    12 11              7 6                0
           imm[11:5]              rs2            rs1      width      imm[4:0]              opcode
                7                  5              5         3             5                  7
           offset[11:5]           src           base       W         offset[4:0]         STORE-FP
```

FLW and FSW are only guaranteed to execute atomically if the effective address is naturally
aligned.

#### 8.6 Single-Precision Floating-Point Computational Instructions

Floating-point arithmetic instructions with one or two source operands use the R-type format
with the OP-FP major opcode. FADD.S, FSUB.S, FMUL.S, and FDIV.S perform single-precision
floating-point addition, subtraction, multiplication, and division, respectively, between rs1 and rs2, writing the result to rd. FMIN.S and FMAX.S write, respectively, the smaller or larger of rs1 and
rs2 to rd. FSQRT.S computes the square root of rs1 and writes the result to rd.

<!-- p.62 -->

The 2-bit floating-point format field fmt is encoded as shown in Table 8.3. It is set to S (00) for all
instructions in the F extension.

```text
                              fmt field   Mnemonic       Meaning
                                 00          S           32-bit single-precision
                                 01          D           64-bit double-precision
                                 10          -           reserved
                                 11          Q           128-bit quad-precision
```

**Table 8.3:** Format field encoding.

All floating-point operations that perform rounding can select the rounding mode using the rm
field with the encoding shown in Table 8.1.

```text
     31          27 26     25 24          20 19           15 14     12 11          7 6                  0
       funct5  fmt                  rs2            rs1        rm             rd          opcode
          5     2                    5              5          3              5            7
     FADD/FSUB S                   src2           src1       RM             dest         OP-FP
     FMUL/FDIV S                   src2           src1       RM             dest         OP-FP
     FMIN-MAX   S                  src2           src1      MIN/MAX         dest         OP-FP
       FSQRT    S                    0             src       RM             dest         OP-FP
```

Floating-point fused multiply-add instructions require a new standard instruction format. R4type instructions specify three source registers (rs1, rs2, and rs3) and a destination register (rd).
This format is only used by the floating-point fused multiply-add instructions. Fused multiply-add
instructions multiply the values in rs1 and rs2, optionally negate the product, then add or subtract
the value in rs3, writing the final result to rd. FMADD.S computes rs1×rs2+rs3; FMSUB.S
computes rs1×rs2-rs3; FNMSUB.S computes -rs1×rs2+rs3; and FNMADD.S computes -rs1×rs2rs3.

The fused multiply-add instructions must raise the invalid operation exception when the multiplicands are ∞ and zero, even when the addend is a quiet NaN.

      The IEEE 754-2008 standard permits, but does not require, raising the invalid exception for the
      operation ∞ × 0 + qNaN.

```text
     31          27 26     25 24          20 19           15 14     12 11          7 6                  0
          rs3        fmt            rs2            rs1         rm            rd          opcode
            5         2              5              5           3             5            7
          src3        S            src2           src1         RM           dest    F[N]MADD/F[N]MSUB
```

<!-- p.63 -->

8.7        Single-Precision                  Floating-Point               Conversion                and      Move
           Instructions

Floating-point-to-integer and integer-to-floating-point conversion instructions are encoded in the
OP-FP major opcode space. FCVT.W.S or FCVT.L.S converts a floating-point number in floatingpoint register rs1 to a signed 32-bit or 64-bit integer, respectively, in integer register rd. FCVT.S.W
or FCVT.S.L converts a 32-bit or 64-bit signed integer, respectively, in integer register rs1 into a
floating-point number in floating-point register rd. FCVT.WU.S, FCVT.LU.S, FCVT.S.WU, and
FCVT.S.LU variants convert to or from unsigned integer values. FCVT.L[U].S and FCVT.S.L[U]
are illegal in RV32. If the rounded result is not representable in the destination format, it is
clipped to the nearest value and the invalid flag is set. Table 8.4 gives the range of valid inputs for
FCVT.int.S and the behavior for invalid inputs.

```text
                                                      FCVT.W.S        FCVT.WU.S         FCVT.L.S     FCVT.LU.S
      Minimum valid input (after rounding)                 −231                 0           −263              0
                                                          31
      Maximum valid input (after rounding)               2 −1             232 − 1          63
                                                                                          2 −1          264 − 1
      Output for out-of-range negative input               −231                 0           −263              0
      Output for −∞                                        −231                 0           −263              0
                                                          31
      Output for out-of-range positive input             2 −1             232 − 1          63
                                                                                          2 −1          264 − 1
      Output for +∞ or NaN                               231 − 1          232 − 1         263 − 1       264 − 1
```

**Table 8.4:** Domains of float-to-integer conversions and behavior for invalid inputs.

All floating-point to integer and integer to floating-point conversion instructions round according
to the rm field. A floating-point register can be initialized to floating-point positive zero using
FCVT.S.W rd, x0, which will never raise any exceptions.

```text
      31            27 26     25 24           20 19           15 14      12 11             7 6                0
      funct5     fmt                rs2                rs1         rm             rd                opcode
         5        2                  5                  5           3              5                  7
    FCVT.int.fmt S               W[U]/L[U]             src         RM            dest               OP-FP
    FCVT.fmt.int S               W[U]/L[U]             src         RM            dest               OP-FP
```

Floating-point to floating-point sign-injection instructions, FSGNJ.S, FSGNJN.S, and FSGNJX.S,
produce a result that takes all bits except the sign bit from rs1. For FSGNJ, the result’s sign
bit is rs2’s sign bit; for FSGNJN, the result’s sign bit is the opposite of rs2’s sign bit; and for
FSGNJX, the sign bit is the XOR of the sign bits of rs1 and rs2. Sign-injection instructions do not
set floating-point exception flags. Note, FSGNJ.S rx, ry, ry moves ry to rx (assembler pseudo-op
FMV.S rx, ry); FSGNJN.S rx, ry, ry moves the negation of ry to rx (assembler pseudo-op FNEG.S
rx, ry); and FSGNJX.S rx, ry, ry moves the absolute value of ry to rx (assembler pseudo-op FABS.S
rx, ry).

```text
      31            27 26     25 24           20 19           15 14      12 11             7 6                0
           funct5       fmt            rs2              rs1       rm              rd                opcode
              5          2              5                5         3               5                  7
           FSGNJ         S            src2             src1     J[N]/JX          dest               OP-FP
```

<!-- p.64 -->

      The sign-injection instructions provide floating-point MV, ABS, and NEG, as well as supporting
      a few other operations, including the IEEE copySign operation and sign manipulation in tran-
      scendental math function libraries. Although MV, ABS, and NEG only need a single register
      operand, whereas FSGNJ instructions need two, it is unlikely most microarchitectures would add
      optimizations to benefit from the reduced number of register reads for these relatively infrequent
      instructions. Even in this case, a microarchitecture can simply detect when both source registers
      are the same for FSGNJ instructions and only read a single copy.

Instructions are provided to move bit patterns between the floating-point and integer registers.
FMV.X.W moves the single-precision value in floating-point register rs1 represented in IEEE 7542008 encoding to the lower 32 bits of integer register rd. For RV64, the higher 32 bits of the
destination register are filled with copies of the floating-point number’s sign bit. FMV.W.X moves
the single-precision value encoded in IEEE 754-2008 standard encoding from the lower 32 bits of
integer register rs1 to the floating-point register rd. The bits are not modified in the transfer, and
in particular, the payloads of non-canonical NaNs are preserved.

      The FMV.W.X and FMV.X.W instructions were previously called FMV.S.X and FMV.X.S. The
      use of W is more consistent with their semantics as an instruction that moves 32 bits without
      interpreting them. This became clearer after defining NaN-boxing. To avoid disturbing existing
      code, both the W and S versions will be supported by tools.

```text
      31         27 26      25 24         20 19           15 14      12 11          7 6                      0
       funct5         fmt           rs2            rs1         rm             rd            opcode
          5            2             5              5           3              5              7
      FMV.X.W          S             0             src         000           dest           OP-FP
      FMV.W.X          S             0             src         000           dest           OP-FP
```

      The base floating-point ISA was defined so as to allow implementations to employ an internal
      recoding of the floating-point format in registers to simplify handling of subnormal values and
      possibly to reduce functional unit latency. To this end, the base ISA avoids representing integer
      values in the floating-point registers by defining conversion and comparison operations that read
      and write the integer register file directly. This also removes many of the common cases where
      explicit moves between integer and floating-point registers are required, reducing instruction count
      and critical paths for common mixed-format code sequences.

#### 8.8 Single-Precision Floating-Point Compare Instructions

Floating-point compare instructions perform the specified comparison (equal, less than, or less
than or equal) between floating-point registers rs1 and rs2 and record the Boolean result in integer
register rd.

FLT.S and FLE.S perform what the IEEE 754-2008 standard refers to as signaling comparisons:
that is, an Invalid Operation exception is raised if either input is NaN. FEQ.S performs a quiet comparison: only signaling NaN inputs cause an Invalid Operation exception. For all three instructions,
the result is 0 if either operand is NaN.

<!-- p.65 -->

```text
   31            27 26     25 24             20 19          15 14     12 11          7 6            0
        funct5       fmt            rs2               rs1       rm      rd                 opcode
           5          2              5                 5         3       5                   7
         FCMP         S            src2              src1     EQ/LT/LE dest                OP-FP
```

#### 8.9 Single-Precision Floating-Point Classify Instruction

The FCLASS.S instruction examines the value in floating-point register rs1 and writes to integer
register rd a 10-bit mask that indicates the class of the floating-point number. The format of the
mask is described in Table 8.5. The corresponding bit in rd will be set if the property is true and
clear otherwise. All other bits in rd are cleared. Note that exactly one bit in rd will be set.

```text
   31            27 26     25 24             20 19          15 14     12 11          7 6            0
        funct5       fmt           rs2               rs1        rm             rd          opcode
           5          2             5                 5          3              5            7
        FCLASS        S             0                src        001           dest         OP-FP
                              rd bit      Meaning
                                0         rs1 is −∞.
                                1         rs1 is a negative normal number.
                                2         rs1 is a negative subnormal number.
                                3         rs1 is −0.
                                4         rs1 is +0.
                                5         rs1 is a positive subnormal number.
                                6         rs1 is a positive normal number.
                                7         rs1 is +∞.
                                8         rs1 is a signaling NaN.
                                9         rs1 is a quiet NaN.
```

**Table 8.5:** Format of result of FCLASS instruction.

<!-- p.67 -->

### 第 9 章 “D” Standard Extension for Double-Precision Floating-Point, Version 2.0

This chapter describes the standard double-precision floating-point instruction-set extension, which
is named “D” and adds double-precision floating-point computational instructions compliant with
the IEEE 754-2008 arithmetic standard. The D extension depends on the base single-precision
instruction subset F.

#### 9.1 D Register State

The D extension widens the 32 floating-point registers, f0–f31, to 64 bits (FLEN=64 in Figure 8.1).
The f registers can now hold either 32-bit or 64-bit floating-point values as described below in
Section 9.2.

      FLEN can be 32, 64, or 128 depending on which of the F, D, and Q extensions are supported.
      There can be up to four different floating-point precisions supported, including H, F, D, and Q.
      Half-precision H scalar values are only supported if the V vector extension is supported.

#### 9.2 NaN Boxing of Narrower Values

When multiple floating-point precisions are supported, then valid values of narrower n-bit types,
n < FLEN, are represented in the lower n bits of an FLEN-bit NaN value, in a process termed
NaN-boxing. The upper bits of a valid NaN-boxed value must be all 1s. Valid NaN-boxed n-bit
values therefore appear as negative quiet NaNs (qNaNs) when viewed as any wider m-bit value,
n < m ≤ FLEN.

      Software might not know the current type of data stored in a floating-point register but has to be able to save and restore the register values, hence the result of using wider operations to transfer
      narrower values has to be defined. A common case is for callee-saved registers, but a standard
      convention is also desirable for features including varargs, user-level threading libraries, virtual
      machine migration, and debugging.
Floating-point n-bit transfer operations move external values held in IEEE standard formats into
and out of the f registers, and comprise floating-point loads and stores (FLn/FSn) and floatingpoint move instructions (FMV.n.X/FMV.X.n). A narrower n-bit transfer, n < FLEN, into the f
registers will create a valid NaN-boxed value by setting all upper FLEN−n bits of the destination
f register to 1. A narrower n-bit transfer out of the floating-point registers will transfer the lower
n bits of the register ignoring the upper FLEN−n bits.

<!-- p.68 -->

Floating-point compute and sign-injection operations calculate results based on the FLEN-bit values
held in the f registers. A narrow n-bit operation, where n < FLEN, checks that input operands
are correctly NaN-boxed, i.e., all upper FLEN−n bits are 1. If so, the n least-significant bits of the
input are used as the input value, otherwise the input value is treated as an n-bit canonical NaN.
An n-bit floating-point result is written to the n least-significant bits of the destination f register,
with all 1s written to the uppermost FLEN−n bits to yield a legal NaN-boxed value.

Conversions from integer to floating-point (e.g., FCVT.S.X), will NaN-box any results narrower
than FLEN to fill the FLEN-bit destination register. Conversions from narrower n-bit floatingpoint values to integer (e.g., FCVT.X.S) will check for legal NaN-boxing and treat the input as the
n-bit canonical NaN if not a legal n-bit value.

      Earlier versions of this document did not define the behavior of feeding the results of narrower or
      wider operands into an operation, except to require that wider saves and restores would preserve
      the value of a narrower operand. The new definition removes this implementation-specific behav-
      ior, while still accommodating both non-recoded and recoded implementations of the floating-point
      unit. The new definition also helps catch software errors by propagating NaNs if values are used
      incorrectly.
          Non-recoded implementations unpack and pack the operands to IEEE standard format on
      the input and output of every floating-point operation. The NaN-boxing cost to a non-recoded
      implementation is primarily in checking if the upper bits of a narrower operation represent a
      legal NaN-boxed value, and in writing all 1s to the upper bits of a result.
          Recoded implementations use a more convenient internal format to represent floating-point
      values, with an added exponent bit to allow all values to be held normalized. The cost to the
      recoded implementation is primarily the extra tagging needed to track the internal types and
      sign bits, but this can be done without adding new state bits by recoding NaNs internally in the
      exponent field. Small modifications are needed to the pipelines used to transfer values in and
      out of the recoded format, but the datapath and latency costs are minimal. The recoding process
      has to handle shifting of input subnormal values for wide operands in any case, and extracting
      the NaN-boxed value is a similar process to normalization except for skipping over leading-1 bits
      instead of skipping over leading-0 bits, allowing the datapath muxing to be shared.

#### 9.3 Double-Precision Load and Store Instructions

The FLD instruction loads a double-precision floating-point value from memory into floating-point
register rd. FSD stores a double-precision value from the floating-point registers to memory.

      The double-precision value may be a NaN-boxed single-precision value.

<!-- p.69 -->

```text
       31                                   20 19            15 14    12 11               7 6               0
                   imm[11:0]                          rs1       width              rd              opcode
                        12                             5          3                 5                7
                   offset[11:0]                      base        D                dest           LOAD-FP
       31                  25 24            20 19            15 14    12 11               7 6               0
            imm[11:5]                rs2              rs1       width       imm[4:0]               opcode
                 7                    5                5          3              5                   7
            offset[11:5]             src             base        D          offset[4:0]          STORE-FP
```

FLD and FSD are only guaranteed to execute atomically if the effective address is naturally aligned
and XLEN≥64.

#### 9.4 Double-Precision Floating-Point Computational Instructions

The double-precision floating-point computational instructions are defined analogously to their
single-precision counterparts, but operate on double-precision operands and produce doubleprecision results.

```text
      31           27 26     25 24           20 19            15 14       12 11            7 6                  0
      funct5  fmt                     rs2              rs1        rm               rd              opcode
         5     2                       5                5          3                5                7
    FADD/FSUB D                      src2             src1       RM               dest             OP-FP
    FMUL/FDIV D                      src2             src1       RM               dest             OP-FP
    FMIN-MAX   D                     src2             src1      MIN/MAX           dest             OP-FP
      FSQRT    D                       0               src       RM               dest             OP-FP
      31           27 26     25 24           20 19            15 14       12 11            7 6                  0
            rs3        fmt            rs2              rs1           rm            rd            opcode
              5         2              5                5             3             5              7
            src3        D            src2             src1           RM           dest      F[N]MADD/F[N]MSUB
```

#### 9.5 Double-Precision Floating-Point Conversion and Move In-

           structions

Floating-point-to-integer and integer-to-floating-point conversion instructions are encoded in the
OP-FP major opcode space. FCVT.W.D or FCVT.L.D converts a double-precision floating-point
number in floating-point register rs1 to a signed 32-bit or 64-bit integer, respectively, in integer
register rd. FCVT.D.W or FCVT.D.L converts a 32-bit or 64-bit signed integer, respectively,
in integer register rs1 into a double-precision floating-point number in floating-point register rd.

<!-- p.70 -->

FCVT.WU.D, FCVT.LU.D, FCVT.D.WU, and FCVT.D.LU variants convert to or from unsigned
integer values. FCVT.L[U].D and FCVT.D.L[U] are illegal in RV32. The range of valid inputs for
FCVT.int.D and the behavior for invalid inputs are the same as for FCVT.int.S.

All floating-point to integer and integer to floating-point conversion instructions round according
to the rm field. Note FCVT.D.W[U] always produces an exact result and is unaffected by rounding
mode.

```text
     31            27 26     25 24          20 19          15 14     12 11          7 6            0
       funct5          fmt         rs2              rs1        rm             rd          opcode
          5             2           5                5          3              5            7
     FCVT.int.D         D       W[U]/L[U]           src        RM            dest         OP-FP
     FCVT.D.int         D       W[U]/L[U]           src        RM            dest         OP-FP
```

The double-precision to single-precision and single-precision to double-precision conversion instructions, FCVT.S.D and FCVT.D.S, are encoded in the OP-FP major opcode space and both the
source and destination are floating-point registers. The rs2 field encodes the datatype of the
source, and the fmt field encodes the datatype of the destination. FCVT.S.D rounds according to
the RM field; FCVT.D.S will never round.

```text
     31            27 26     25 24          20 19          15 14     12 11          7 6            0
      funct5           fmt           rs2            rs1        rm             rd          opcode
         5              2             5              5          3              5            7
     FCVT.S.D           S             D             src        RM            dest         OP-FP
     FCVT.D.S           D             S             src        RM            dest         OP-FP
```

Floating-point to floating-point sign-injection instructions, FSGNJ.D, FSGNJN.D, and FSGNJX.D
are defined analogously to the single-precision sign-injection instruction.

```text
     31            27 26     25 24          20 19          15 14     12 11          7 6            0
          funct5       fmt            rs2            rs1       rm             rd          opcode
             5          2              5              5         3              5            7
          FSGNJ         D            src2           src1     J[N]/JX         dest         OP-FP
```

For RV64 only, instructions are provided to move bit patterns between the floating-point and
integer registers. FMV.X.D moves the double-precision value in floating-point register rs1 to a
representation in IEEE 754-2008 standard encoding in integer register rd. FMV.D.X moves the
double-precision value encoded in IEEE 754-2008 standard encoding from the integer register rs1
to the floating-point register rd.

```text
     31            27 26     25 24          20 19          15 14     12 11          7 6            0
       funct5          fmt           rs2            rs1        rm             rd          opcode
          5             2             5              5          3              5            7
      FMV.X.D           D             0             src        000           dest         OP-FP
      FMV.D.X           D             0             src        000           dest         OP-FP
```

<!-- p.71 -->

#### 9.6 Double-Precision Floating-Point Compare Instructions

The double-precision floating-point compare instructions are defined analogously to their singleprecision counterparts, but operate on double-precision operands.

```text
   31            27 26     25 24          20 19          15 14     12 11          7 6            0
        funct5       fmt            rs2            rs1       rm      rd                 opcode
           5          2              5              5         3       5                   7
         FCMP         D            src2           src1     EQ/LT/LE dest                OP-FP
```

#### 9.7 Double-Precision Floating-Point Classify Instruction

The double-precision floating-point classify instruction, FCLASS.D, is defined analogously to its
single-precision counterpart, but operates on double-precision operands.

```text
   31            27 26     25 24          20 19          15 14     12 11          7 6            0
        funct5       fmt           rs2            rs1        rm             rd          opcode
           5          2             5              5          3              5            7
        FCLASS        D             0             src        001           dest         OP-FP
```

<!-- p.73 -->

### 第 10 章 “Q” Standard Extension for Quad-Precision Floating-Point, Version 2.0

This chapter describes the Q standard extension for 128-bit binary floating-point instructions compliant with the IEEE 754-2008 arithmetic standard. The 128-bit or quad-precision binary floatingpoint instruction subset is named “Q”, and requires RV64IFD. The floating-point registers are
now extended to hold either a single, double, or quad-precision floating-point value (FLEN=128).
The NaN-boxing scheme described in Section 9.2 is now extended recursively to allow a singleprecision value to be NaN-boxed inside a double-precision value which is itself NaN-boxed inside a
quad-precision value.

#### 10.1 Quad-Precision Load and Store Instructions

New 128-bit variants of LOAD-FP and STORE-FP instructions are added, encoded with a new
value for the funct3 width field.

```text
      31                                 20 19               15 14   12 11             7 6              0
                   imm[11:0]                      rs1           width         rd               opcode
                        12                         5              3            5                 7
                   offset[11:0]                  base            Q           dest            LOAD-FP
       31                  25 24         20 19           15 14       12 11             7 6              0
            imm[11:5]              rs2            rs1           width    imm[4:0]              opcode
                 7                  5              5              3           5                  7
            offset[11:5]           src           base            Q       offset[4:0]         STORE-FP
```

FLQ and FSQ are only guaranteed to execute atomically if the effective address is naturally aligned
and XLEN=128.

<!-- p.74 -->

#### 10.2 Quad-Precision Computational Instructions

A new supported format is added to the format field of most instructions, as shown in Table 10.1.

```text
                            fmt field     Mnemonic       Meaning
                               00            S           32-bit single-precision
                               01            D           64-bit double-precision
                               10            -           reserved
                               11            Q           128-bit quad-precision
```

**Table 10.1:** Format field encoding.

The quad-precision floating-point computational instructions are defined analogously to their
double-precision counterparts, but operate on quad-precision operands and produce quad-precision
results.

```text
     31          27 26     25 24          20 19           15 14    12 11           7 6            0
       funct5  fmt                  rs2            rs1        rm            rd           opcode
          5     2                    5              5          3             5             7
     FADD/FSUB Q                   src2           src1       RM            dest          OP-FP
     FMUL/FDIV Q                   src2           src1       RM            dest          OP-FP
     FMIN-MAX   Q                  src2           src1      MIN/MAX        dest          OP-FP
       FSQRT    Q                    0             src       RM            dest          OP-FP
     31          27 26     25 24          20 19           15 14    12 11           7 6            0
          rs3        fmt            rs2            rs1        rm            rd           opcode
            5         2              5              5          3             5             7
          src3        Q            src2           src1        RM           dest     F[N]MADD/F[N]MSUB
```

#### 10.3 Quad-Precision Convert and Move Instructions

```text
     31          27 26     25 24          20 19           15 14    12 11           7 6            0
       funct5        fmt         rs2              rs1         rm            rd           opcode
          5           2           5                5           3             5             7
     FCVT.int.Q       Q       W[U]/L[U]           src         RM           dest          OP-FP
     FCVT.Q.int       Q       W[U]/L[U]           src         RM           dest          OP-FP
```

New floating-point to floating-point conversion instructions FCVT.S.Q, FCVT.Q.S, FCVT.D.Q,
FCVT.Q.D are added.

<!-- p.75 -->

```text
     31            27 26     25 24               20 19            15 14     12 11               7 6              0
      funct5           fmt               rs2              rs1         rm                 rd            opcode
         5              2                 5                5           3                  5              7
     FCVT.S.Q           S                 Q               src         RM                dest           OP-FP
     FCVT.Q.S           Q                 S               src         RM                dest           OP-FP
     FCVT.D.Q           D                 Q               src         RM                dest           OP-FP
     FCVT.Q.D           Q                 D               src         RM                dest           OP-FP
```

Floating-point to floating-point sign-injection instructions, FSGNJ.Q, FSGNJN.Q, and FSGNJX.Q
are defined analogously to the double-precision sign-injection instruction.

```text
     31            27 26     25 24               20 19            15 14     12 11               7 6              0
          funct5       fmt                rs2              rs1        rm                 rd            opcode
             5          2                  5                5          3                  5              7
          FSGNJ         Q                src2             src1      J[N]/JX             dest           OP-FP
```

FMV.X.Q and FMV.Q.X instructions are not provided, so quad-precision bit patterns must be
moved to the integer registers via memory.

        RV128 supports FMV.X.Q and FMV.Q.X in the Q extension.

#### 10.4 Quad-Precision Floating-Point Compare Instructions

Floating-point compare instructions perform the specified comparison (equal, less than, or less
than or equal) between floating-point registers rs1 and rs2 and record the Boolean result in integer
register rd.

```text
   31               27 26        25 24            20 19            15 14        12 11            7 6                 0
          funct5           fmt             rs2              rs1        rm      rd                       opcode
             5              2               5                5          3       5                         7
           FCMP             Q             src2             src1      EQ/LT/LE dest                      OP-FP
```

#### 10.5 Quad-Precision Floating-Point Classify Instruction

The quad-precision floating-point classify instruction, FCLASS.Q, is defined analogously to its
double-precision counterpart, but operates on quad-precision operands.

```text
   31               27 26        25 24            20 19            15 14        12 11            7 6                 0
        funct5             fmt            rs2              rs1            rm              rd            opcode
           5                2              5                5              3               5              7
        FCLASS              Q              0               src            001            dest           OP-FP
```

<!-- p.77 -->

### 第 11 章 “L” Standard Extension for Decimal Floating-Point, Version 0.0

This chapter is a placeholder for the specification of a standard extension named “L” designed to
support decimal floating-point arithmetic as defined in the IEEE 754-2008 standard.

#### 11.1 Decimal Floating-Point Registers

Existing floating-point registers are used to hold 64-bit and 128-bit decimal floating-point values,
and the existing floating-point load and store instructions are used to move values to and from
memory.

     Due to the large opcode space required by the fused multiply-add instructions, the decimal floating-
     point instruction extension will require five 25-bit major opcodes in a 30-bit encoding space.

<!-- p.79 -->

### 第 12 章 “C” Standard Extension for Compressed Instructions, Version 2.0

This chapter describes the current draft proposal for the RISC-V standard compressed instruction
set extension, named “C”, which reduces static and dynamic code size by adding short 16-bit
instruction encodings for common operations. The C extension can be added to any of the base
ISAs (RV32, RV64, RV128), and we use the generic term “RVC” to cover any of these. Typically,
50%–60% of the RISC-V instructions in a program can be replaced with RVC instructions, resulting
in a 25%–30% code-size reduction.

#### 12.1 Overview

RVC uses a simple compression scheme that offers shorter 16-bit versions of common 32-bit RISC-V
instructions when:

- the immediate or address offset is small, or

- one of the registers is the zero register (x0), the ABI link register (x1), or the ABI stack pointer (x2), or

- the destination register and the first source register are identical, or

- the registers used are the 8 most popular ones.

The C extension is compatible with all other standard instruction extensions. The C extension
allows 16-bit instructions to be freely intermixed with 32-bit instructions, with the latter now able
to start on any 16-bit boundary. With the addition of the C extension, JAL and JALR instructions
will no longer raise an instruction misaligned exception.

<!-- p.80 -->

       Removing the 32-bit alignment constraint on the original 32-bit instructions allows significantly
       greater code density.

The compressed instruction encodings are mostly common across RV32C, RV64C, and RV128C,
but as shown in Table 12.3, a few opcodes are used for different purposes depending on base
ISA width. For example, the wider address-space RV64C and RV128C variants require additional
opcodes to compress loads and stores of 64-bit integer values, while RV32C uses the same opcodes
to compress loads and stores of single-precision floating-point values. Similarly, RV128C requires
additional opcodes to capture loads and stores of 128-bit integer values, while these same opcodes
are used for loads and stores of double-precision floating-point values in RV32C and RV64C. If the
C extension is implemented, the appropriate compressed floating-point load and store instructions
must be provided whenever the relevant standard floating-point extension (F and/or D) is also
implemented. In addition, RV32C includes a compressed jump and link instruction to compress
short-range subroutine calls, where the same opcode is used to compress ADDIW for RV64C and
RV128C.

       Double-precision loads and stores are a significant fraction of static and dynamic instructions,
       hence the motivation to include them in the RV32C and RV64C encoding.
           Although single-precision loads and stores are not a significant source of static or dynamic
       compression for benchmarks compiled for the currently supported ABIs, for microcontrollers
       that only provide hardware single-precision floating-point units and have an ABI that only sup-
       ports single-precision floating-point numbers, the single-precision loads and stores will be used
       at least as frequently as double-precision loads and stores in the measured benchmarks. Hence,
       the motivation to provide compressed support for these in RV32C.
           Short-range subroutine calls are more likely in small binaries for microcontrollers, hence the
       motivation to include these in RV32C.
           Although reusing opcodes for different purposes for different base register widths adds some
       complexity to documentation, the impact on implementation complexity is small even for designs
       that support multiple base ISA register widths. The compressed floating-point load and store
       variants use the same instruction format with the same register specifiers as the wider integer
       loads and stores.

RVC was designed under the constraint that each RVC instruction expands into a single 32-bit
instruction in either the base ISA (RV32I/E, RV64I, or RV128I) or the F and D standard extensions
where present. Adopting this constraint has two main benefits:

- Hardware designs can simply expand RVC instructions during decode, simplifying verification and minimizing modifications to existing microarchitectures.
- Compilers can be unaware of the RVC extension and leave code compression to the assembler and linker, although a compression-aware compiler will generally be able to produce better results.

       We felt the multiple complexity reductions of a simple one-one mapping between C and base
       IFD instructions far outweighed the potential gains of a slightly denser encoding that added
       additional instructions only supported in the C extension, or that allowed encoding of multiple
       IFD instructions in one C instruction.

It is important to note that the C extension is not designed to be a stand-alone ISA, and is meant
to be used alongside a base ISA.

<!-- p.81 -->

     Variable-length instruction sets have long been used to improve code density. For example, the
     IBM Stretch [6], developed in the late 1950s, had an ISA with 32-bit and 64-bit instructions,
     where some of the 32-bit instructions were compressed versions of the full 64-bit instructions.
     Stretch also employed the concept of limiting the set of registers that were addressable in some
     of the shorter instruction formats, with short branch instructions that could only refer to one
     of the index registers. The later IBM 360 architecture [3] supported a simple variable-length
     instruction encoding with 16-bit, 32-bit, or 48-bit instruction formats.
         In 1963, CDC introduced the Cray-designed CDC 6600 [28], a precursor to RISC archi-
     tectures, that introduced a register-rich load-store architecture with instructions of two lengths,
     15-bits and 30-bits. The later Cray-1 design used a very similar instruction format, with 16-bit
     and 32-bit instruction lengths.
         The initial RISC ISAs from the 1980s all picked performance over code size, which was
     reasonable for a workstation environment, but not for embedded systems. Hence, both ARM
     and MIPS subsequently made versions of the ISAs that offered smaller code size by offering an
     alternative 16-bit wide instruction set instead of the standard 32-bit wide instructions. The com-
     pressed RISC ISAs reduced code size relative to their starting points by about 25–30%, yielding
     code that was significantly smaller than 80x86. This result surprised some, as their intuition
     was that the variable-length CISC ISA should be smaller than RISC ISAs that offered only 16-bit
     and 32-bit formats.
         Since the original RISC ISAs did not leave sufficient opcode space free to include these
     unplanned compressed instructions, they were instead developed as complete new ISAs. This
     meant compilers needed different code generators for the separate compressed ISAs. The first
     compressed RISC ISA extensions (e.g., ARM Thumb and MIPS16) used only a fixed 16-bit in-
     struction size, which gave good reductions in static code size but caused an increase in dynamic
     instruction count, which led to lower performance compared to the original fixed-width 32-bit
     instruction size. This led to the development of a second generation of compressed RISC ISA
     designs with mixed 16-bit and 32-bit instruction lengths (e.g., ARM Thumb2, microMIPS, Pow-
     erPC VLE), so that performance was similar to pure 32-bit instructions but with significant
     code size savings. Unfortunately, these different generations of compressed ISAs are incompati-
     ble with each other and with the original uncompressed ISA, leading to significant complexity in
     documentation, implementations, and software tools support.
         Of the commonly used 64-bit ISAs, only PowerPC and microMIPS currently supports a
     compressed instruction format. It is surprising that the most popular 64-bit ISA for mobile
     platforms (ARM v8) does not include a compressed instruction format given that static code
     size and dynamic instruction fetch bandwidth are important metrics. Although static code size
     is not a major concern in larger systems, instruction fetch bandwidth can be a major bottleneck
     in servers running commercial workloads, which often have a large instruction working set.
         Benefiting from 25 years of hindsight, RISC-V was designed to support compressed instruc-
     tions from the outset, leaving enough opcode space for RVC to be added as a simple extension
     on top of the base ISA (along with many other extensions). The philosophy of RVC is to reduce
     code size for embedded applications and to improve performance and energy-efficiency for all
     applications due to fewer misses in the instruction cache. Waterman shows that RVC fetches
     25%-30% fewer instruction bits, which reduces instruction cache misses by 20%-25%, or roughly
     the same performance impact as doubling the instruction cache size [33].

#### 12.2 Compressed Instruction Formats

Table 12.1 shows the eight compressed instruction formats. CR, CI, and CSS can use any of the
32 RVI registers, but CIW, CL, CS, and CB are limited to just 8 of them. Table 12.2 lists these
popular registers, which correspond to registers x8 to x15. Note that there is a separate version of load and store instructions that use the stack pointer as the base address register, since saving to
and restoring from the stack are so prevalent, and that they use the CI and CSS formats to allow
access to all 32 data registers. CIW supplies an 8-bit immediate for the ADDI4SPN instruction.

<!-- p.82 -->

     The RISC-V ABI was changed to make the frequently used registers map to registers x8–x15.
     This simplifies the decompression decoder by having a contiguous naturally aligned set of register
     numbers, and is also compatible with the RV32E subset base specification, which only has 16
     integer registers.

Compressed register-based floating-point loads and stores also use the CL and CS formats respectively, with the eight registers mapping to f8 to f15.

     The standard RISC-V calling convention maps the most frequently used floating-point registers
     to registers f8 to f15, which allows the same register decompression decoding as for integer
     register numbers.

The formats were designed to keep bits for the two register source specifiers in the same place in all
instructions, while the destination register field can move. When the full 5-bit destination register
specifier is present, it is in the same place as in the 32-bit RISC-V encoding. Where immediates
are sign-extended, the sign-extension is always from bit 12. Immediate fields have been scrambled,
as in the base specification, to reduce the number of immediate muxes required.

     The immediate fields are scrambled in the instruction formats instead of in sequential order so
     that as many bits as possible are in the same position in every instruction, thereby simplify-
     ing implementations. For example, immediate bits 17—10 are always sourced from the same
     instruction bit positions. Five other immediate bits (5, 4, 3, 1, and 0) have just two source
     instruction bits, while four (9, 7, 6, and 2) have three sources and one (8) has four sources.

For many RVC instructions, zero-valued immediates are disallowed and x0 is not a valid 5-bit
register specifier. These restrictions free up encoding space for other instructions requiring fewer
operand bits.

```text
        Format         Meaning           15 14 13   12    11 10 9    8   7   6   5   4   3   2   1    0
          CR           Register             funct4          rd/rs1            rs2                op
          CI          Immediate          funct3 imm         rd/rs1           imm                 op
         CSS      Stack-relative Store   funct3           imm                 rs2                op
         CIW       Wide Immediate        funct3              imm                   rd0           op
          CL             Load            funct3    imm           rs10    imm       rd0           op
          CS             Store           funct3    imm           rs10    imm      rs20           op
          CB            Branch           funct3    offset        rs10        offset              op
          CJ             Jump            funct3                jump target                       op
```

**Table 12.1:** Compressed 16-bit RVC instruction formats.

<!-- p.83 -->

```text
       RVC Register Number                    000   001     010   011    100    101   110   111
       Integer Register Number                x8    x9      x10   x11    x12    x13   x14   x15
       Integer Register ABI Name              s0    s1       a0    a1    a2      a3    a4   a5
       Floating-Point Register Number         f8    f9      f10   f11    f12    f13   f14   f15
       Floating-Point Register ABI Name       fs0   fs1     fa0   fa1    fa2    fa3   fa4   fa5
```

**Table 12.2:** Registers specified by the three-bit rs1’, rs2’, and rd’ fields of the CIW, CL, CS, and CB formats.

#### 12.3 Load and Store Instructions

To increase the reach of 16-bit instructions, data-transfer instructions use zero-extended immediates
that are scaled by the size of the data in bytes: ×4 for words, ×8 for double words, and ×16 for
quad words.

RVC provides two variants of loads and stores. One uses the ABI stack pointer, x2, as the base
address and can target any data register. The other can reference one of 8 base address registers
and one of 8 data registers.

Stack-Pointer-Based Loads and Stores

```text
     15             13   12      11                       7 6                         2 1         0
           funct3     imm                 rd                           imm                  op
              3         1                  5                             5                   2
          C.LWSP     offset[5]          dest6=0                   offset[4:2|7:6]           C2
           C.LDSP    offset[5]          dest6=0                   offset[4:3|8:6]           C2
           C.LQSP    offset[5]          dest6=0                    offset[4|9:6]            C2
          C.FLWSP    offset[5]           dest                     offset[4:2|7:6]           C2
          C.FLDSP    offset[5]           dest                     offset[4:3|8:6]           C2
```

These instructions use the CI format.

C.LWSP loads a 32-bit value from memory into register rd. It computes an effective address
by adding the zero-extended offset, scaled by 4, to the stack pointer, x2. It expands to lw rd,
offset[7:2](x2).

C.LDSP is an RV64C/RV128C-only instruction that loads a 64-bit value from memory into register
rd. It computes its effective address by adding the zero-extended offset, scaled by 8, to the stack
pointer, x2. It expands to ld rd, offset[8:3](x2).

C.LQSP is an RV128C-only instruction that loads a 128-bit value from memory into register rd. It
computes its effective address by adding the zero-extended offset, scaled by 16, to the stack pointer,
x2. It expands to lq rd, offset[9:4](x2).

<!-- p.84 -->

C.FLWSP is an RV32FC-only instruction that loads a single-precision floating-point value from
memory into floating-point register rd. It computes its effective address by adding the zero-extended
offset, scaled by 4, to the stack pointer, x2. It expands to flw rd, offset[7:2](x2).

C.FLDSP is an RV32DC/RV64DC-only instruction that loads a double-precision floating-point
value from memory into floating-point register rd. It computes its effective address by adding
the zero-extended offset, scaled by 8, to the stack pointer, x2. It expands to fld rd,
offset[8:3](x2).

```text
      15              13 12                                7 6                            2 1         0
            funct3                      imm                               rs2                   op
               3                          6                                5                     2
           C.SWSP                  offset[5:2|7:6]                        src                   C2
            C.SDSP                 offset[5:3|8:6]                        src                   C2
            C.SQSP                 offset[5:4|9:6]                        src                   C2
           C.FSWSP                 offset[5:2|7:6]                        src                   C2
           C.FSDSP                 offset[5:3|8:6]                        src                   C2
```

These instructions use the CSS format.

C.SWSP stores a 32-bit value in register rs2 to memory. It computes an effective address by
adding the zero-extended offset, scaled by 4, to the stack pointer, x2. It expands to sw rs2,
offset[7:2](x2).

C.SDSP is an RV64C/RV128C-only instruction that stores a 64-bit value in register rs2 to memory.
It computes an effective address by adding the zero-extended offset, scaled by 8, to the stack pointer,
x2. It expands to sd rs2, offset[8:3](x2).

C.SQSP is an RV128C-only instruction that stores a 128-bit value in register rs2 to memory. It
computes an effective address by adding the zero-extended offset, scaled by 16, to the stack pointer,
x2. It expands to sq rs2, offset[9:4](x2).

C.FSWSP is an RV32FC-only instruction that stores a single-precision floating-point value in
floating-point register rs2 to memory. It computes an effective address by adding the zero-extended
offset, scaled by 4, to the stack pointer, x2. It expands to fsw rs2, offset[7:2](x2).

C.FSDSP is an RV32DC/RV64DC-only instruction that stores a double-precision floating-point
value in floating-point register rs2 to memory. It computes an effective address by adding the zeroextended offset, scaled by 8, to the stack pointer, x2. It expands to fsd rs2, offset[8:3](x2).

      Register save/restore code at function entry/exit represents a significant portion of static code
      size. The stack-pointer-based compressed loads and stores in RVC are effective at reducing the
      save/restore static code size by a factor of 2 while improving performance by reducing dynamic
      instruction bandwidth.
          A common mechanism used in other ISAs to further reduce save/restore code size is load-
      multiple and store-multiple instructions. We considered adopting these for RISC-V but noted
      the following drawbacks to these instructions:
- These instructions complicate processor implementations.

<!-- p.85 -->

- For virtual memory systems, some data accesses could be resident in physical memory and some could not, which requires a new restart mechanism for partially executed instructions.
- Unlike the rest of the RVC instructions, there is no IFD equivalent to Load Multiple and Store Multiple.
- Unlike the rest of the RVC instructions, the compiler would have to be aware of these instructions to both generate the instructions and to allocate registers in an order to maximize the chances of the them being saved and stored, since they would be saved and restored in sequential order.
- Simple microarchitectural implementations will constrain how other instructions can be scheduled around the load and store multiple instructions, leading to a potential performance loss.
- The desire for sequential register allocation might conflict with the featured registers selected for the CIW, CL, CS, and CB formats. Furthermore, much of the gains can be realized in software by replacing prologue and epilogue code with subroutine calls to common prologue and epilogue code, a technique described in Section 5.6 of [34]. While reasonable architects might come to different conclusions, we decided to omit load and store multiple and instead use the software-only approach of calling save/restore millicode routines to attain the greatest code size reduction.

Register-Based Loads and Stores

```text
      15                13 12                10 9               7 6          5 4                2 1          0
            funct3               imm                 rs10            imm              rd0             op
               3                   3                  3                2              3                2
             C.LW             offset[5:3]            base         offset[2|6]        dest             C0
             C.LD             offset[5:3]            base         offset[7:6]        dest             C0
             C.LQ            offset[5|4|8]           base         offset[7:6]        dest             C0
            C.FLW             offset[5:3]            base         offset[2|6]        dest             C0
            C.FLD             offset[5:3]            base         offset[7:6]        dest             C0
```

These instructions use the CL format.

C.LW loads a 32-bit value from memory into register rd0 . It computes an effective address by adding
the zero-extended offset, scaled by 4, to the base address in register rs10 . It expands to lw rd0 ,
offset[6:2](rs10 ).

C.LD is an RV64C/RV128C-only instruction that loads a 64-bit value from memory into register
rd0 . It computes an effective address by adding the zero-extended offset, scaled by 8, to the base
address in register rs10 . It expands to ld rd0 , offset[7:3](rs10 ).

C.LQ is an RV128C-only instruction that loads a 128-bit value from memory into register rd0 . It
computes an effective address by adding the zero-extended offset, scaled by 16, to the base address
in register rs10 . It expands to lq rd0 , offset[8:4](rs10 ).

C.FLW is an RV32FC-only instruction that loads a single-precision floating-point value from memory into floating-point register rd0 . It computes an effective address by adding the zero-extended
offset, scaled by 4, to the base address in register rs10 . It expands to flw rd0 , offset[6:2](rs10 ).

<!-- p.86 -->

C.FLD is an RV32DC/RV64DC-only instruction that loads a double-precision floating-point value
from memory into floating-point register rd0 . It computes an effective address by adding the
zero-extended offset, scaled by 8, to the base address in register rs10 . It expands to fld rd0 ,
offset[7:3](rs10 ).

```text
       15            13 12               10 9          7 6        5 4              2 1        0
            funct3           imm                rs10       imm           rs20            op
               3               3                 3           2             3              2
             C.SW         offset[5:3]           base    offset[2|6]       src            C0
             C.SD         offset[5:3]           base    offset[7:6]       src            C0
             C.SQ        offset[5|4|8]          base    offset[7:6]       src            C0
            C.FSW         offset[5:3]           base    offset[2|6]       src            C0
             C.FSD        offset[5:3]           base    offset[7:6]       src            C0
```

These instructions use the CS format.

C.SW stores a 32-bit value in register rs20 to memory. It computes an effective address by adding
the zero-extended offset, scaled by 4, to the base address in register rs10 . It expands to sw rs20 ,
offset[6:2](rs10 ).

C.SD is an RV64C/RV128C-only instruction that stores a 64-bit value in register rs20 to memory.
It computes an effective address by adding the zero-extended offset, scaled by 8, to the base address
in register rs10 . It expands to sd rs20 , offset[7:3](rs10 ).

C.SQ is an RV128C-only instruction that stores a 128-bit value in register rs20 to memory. It
computes an effective address by adding the zero-extended offset, scaled by 16, to the base address
in register rs10 . It expands to sq rs20 , offset[8:4](rs10 ).

C.FSW is an RV32FC-only instruction that stores a single-precision floating-point value in floatingpoint register rs20 to memory. It computes an effective address by adding the zero-extended offset,
scaled by 4, to the base address in register rs10 . It expands to fsw rs20 , offset[6:2](rs10 ).

C.FSD is an RV32DC/RV64DC-only instruction that stores a double-precision floating-point value
in floating-point register rs20 to memory. It computes an effective address by adding the zeroextended offset, scaled by 8, to the base address in register rs10 . It expands to fsd rs20 ,
offset[7:3](rs10 ).

#### 12.4 Control Transfer Instructions

RVC provides unconditional jump instructions and conditional branch instructions. As with base
RVI instructions, the offsets of all RVC control transfer instruction are in multiples of 2 bytes.

<!-- p.87 -->

```text
           15               13 12                                                          2 1              0
                funct3                                     imm                                     op
                   3                                        11                                     2
                  C.J                          offset[11|4|9:8|10|6|7|3:1|5]                       C1
                 C.JAL                         offset[11|4|9:8|10|6|7|3:1|5]                       C1
```

These instructions use the CJ format.

C.J performs an unconditional control transfer. The offset is sign-extended and added to the pc to
form the jump target address. C.J can therefore target a ±2 KiB range. C.J expands to jal x0,
offset[11:1].

C.JAL is an RV32C-only instruction that performs the same operation as C.J, but additionally
writes the address of the instruction following the jump (pc+2) to the link register, x1. C.JAL
expands to jal x1, offset[11:1].

```text
      15                         12 11                         7 6                               2 1            0
                funct4                          rs1                            rs2                     op
                   4                             5                              5                      2
                 C.JR                         src6=0                            0                      C2
                C.JALR                        src6=0                            0                      C2
```

These instructions use the CR format.

C.JR (jump register) performs an unconditional control transfer to the address in register rs1. C.JR
expands to jalr x0, rs1, 0.

C.JALR (jump and link register) performs the same operation as C.JR, but additionally writes the
address of the instruction following the jump (pc+2) to the link register, x1. C.JALR expands to
jalr x1, rs1, 0.

     Strictly speaking, C.JALR does not expand exactly to a base RVI instruction as the value added
     to the PC to form the link address is 2 rather than 4 as in the base ISA, but supporting both
     offsets of 2 and 4 bytes is only a very minor change to the base microarchitecture.

```text
      15                 13 12               10 9              7 6                               2 1            0
           funct3                imm                   rs10                  imm                       op
              3                    3                    3                      5                       2
           C.BEQZ            offset[8|4:3]             src             offset[7:6|2:1|5]               C1
           C.BNEZ            offset[8|4:3]             src             offset[7:6|2:1|5]               C1
```

These instructions use the CB format.

<!-- p.88 -->

C.BEQZ performs conditional control transfers. The offset is sign-extended and added to the pc to
form the branch target address. It can therefore target a ±256 B range. C.BEQZ takes the branch
if the value in register rs10 is zero. It expands to beq rs10 , x0, offset[8:1].

C.BNEZ is defined analogously, but it takes the branch if rs10 contains a nonzero value. It expands
to bne rs10 , x0, offset[8:1].

#### 12.5 Integer Computational Instructions

RVC provides several instructions for integer arithmetic and constant generation.

Integer Constant-Generation Instructions

```text
The two constant-generation instructions both use the CI instruction format and can target any
integer register.
     15            13      12   11                      7 6                         2 1        0
          funct3         imm[5]           rd                       imm[4:0]               op
             3            1                5                           5                  2
            C.LI        imm[5]         dest6=0                      imm[4:0]              C1
           C.LUI        nzuimm[17]   dest6={0, 2}                nzuimm[16:12]            C1
```

C.LI loads the sign-extended 6-bit immediate, imm, into register rd. C.LI is only valid when rd6=x0.
C.LI expands into addi rd, x0, imm[5:0].

C.LUI loads the non-zero 6-bit immediate field into bits 17–12 of the destination register, clears
the bottom 12 bits, and sign-extends bit 17 into all higher bits of the destination. C.LUI is only
valid when rd6={x0, x2}, and when the immediate is not equal to zero. C.LUI expands into lui
rd, nzuimm[17:12].

Integer Register-Immediate Operations

```text
These integer register-immediate operations are encoded in the CI format and perform operations
on any non-x0 integer register and a 6-bit immediate. The immediate cannot be zero.
     15            13      12   11                      7 6                         2 1        0
       funct3    imm[5]                 rd/rs1                       imm[4:0]             op
          3       1                        5                            5                 2
       C.ADDI   nzimm[5]                 dest                       nzimm[4:0]            C1
      C.ADDIW imm[5]                    dest6=0                      imm[4:0]             C1
     C.ADDI16SP nzimm[9]                   2                     nzimm[4|6|8:7|5]         C1
```

C.ADDI adds the non-zero sign-extended 6-bit immediate to the value in register rd then writes
the result to rd. C.ADDI expands into addi rd, rd, nzimm[5:0].

<!-- p.89 -->

C.ADDIW is an RV64C/RV128C-only instruction that performs the same computation but produces a 32-bit result, then sign-extends result to 64 bits. C.ADDIW expands into addiw rd, rd,
imm[5:0]. The immediate can be zero for C.ADDIW, where this corresponds to sext.w rd.

C.ADDI16SP shares the opcode with C.LUI, but has a destination field of x2. C.ADDI16SP adds
the non-zero sign-extended 6-bit immediate to the value in the stack pointer (sp=x2), where the
immediate is scaled to represent multiples of 16 in the range (-512,496). C.ADDI16SP is used
to adjust the stack pointer in procedure prologues and epilogues. It expands into addi x2, x2,
nzimm[9:4].

        In the standard RISC-V calling convention, the stack pointer sp is always 16-byte aligned.

```text
        15               13 12                                          5 4              2 1          0
           funct3                               imm                            rd0              op
             3                                    8                             3                2
        C.ADDI4SPN                        nzuimm[5:4|9:6|2|3]                  dest             C0
```

C.ADDI4SPN is a CIW-format RV32C/RV64C-only instruction that adds a zero-extended non-zero
immediate, scaled by 4, to the stack pointer, x2, and writes the result to rd0 . This instruction is used
to generate pointers to stack-allocated variables, and expands to addi rd0 , x2, nzuimm[9:2].
   15               13       12      11                           7 6                          2 1        0
         funct3           shamt[5]              rd/rs1                    shamt[4:0]                 op
            3              1                       5                          5                      2
         C.SLLI          shamt[5]               dest6=0                   shamt[4:0]                 C2

C.SLLI is a CI-format instruction that performs a logical left shift of the value in register rd then
writes the result to rd. The shift amount is encoded in the shamt field, where shamt[5] must be
zero for RV32C. For RV32C and RV64C, the shift amount must be non-zero. For RV128C, a shift
amount of zero is used to encode a shift of 64. C.SLLI expands into slli rd, rd, shamt[5:0],
except for RV128C with shamt=0, which expands to slli rd, rd, 64.
   15               13       12      11        10 9               7 6                          2 1        0
         funct3        shamt[5]      funct2           rd0 /rs10           shamt[4:0]                 op
            3           1               2                 3                   5                      2
         C.SRLI       shamt[5]       C.SRLI             dest              shamt[4:0]                 C1
         C.SRAI       shamt[5]       C.SRAI             dest              shamt[4:0]                 C1

C.SRLI is a CB-format instruction that performs a logical right shift of the value in register rd0
then writes the result to rd0 . The shift amount is encoded in the shamt field, where shamt[5] must
be zero for RV32C. For RV32C and RV64C, the shift amount must be non-zero. For RV128C, a
shift amount of zero is used to encode a shift of 64. Furthermore, the shift amount is sign-extended
for RV128C, and so the legal shift amounts are 1–31, 64, and 96–127. C.SRLI expands into srli
rd0 , rd0 , shamt[5:0], except for RV128C with shamt=0, which expands to srli rd0 , rd0 , 64.

C.SRAI is defined analogously to C.SRLI, but instead performs an arithmetic right shift. C.SRAI
expands to srai rd0 , rd0 , shamt[5:0].

<!-- p.90 -->

      Left shifts are usually more frequent than right shifts, as left shifts are frequently used to scale
      address values. Right shifts have therefore been granted less encoding space and are placed in
      an encoding quadrant where all other immediates are sign-extended. For RV128, the decision
      was made to have the 6-bit shift-amount immediate also be sign-extended. Apart from reducing
      the decode complexity, we believe right-shift amounts of 96–127 will be more useful than 64–95,
      to allow extraction of tags located in the high portions of 128-bit address pointers. We note
      that RV128C will not be frozen at the same point as RV32C and RV64C, to allow evaluation of
      typical usage of 128-bit address-space codes.

```text
     15             13      12           11    10 9                   7 6                           2 1             0
           funct3         imm[5]          funct2        rd0 /rs10                 imm[4:0]                 op
              3            1                2               3                        5                     2
           C.ANDI        imm[5]          C.ANDI           dest                    imm[4:0]                 C1
```

C.ANDI is a CB-format instruction that computes the bitwise AND of of the value in register rd0
and the sign-extended 6-bit immediate, then writes the result to rd0 . C.ANDI expands to andi
rd0 , rd0 , imm[5:0].

```text
Integer Register-Register Operations
          15                     12 11                              7 6                           2 1           0
               funct4                         rd/rs1                                rs2                   op
                  4                              5                                   5                    2
                C.MV                          dest6=0                             src6=0                  C2
               C.ADD                          dest6=0                             src6=0                  C2
```

These instructions use the CR format.

C.MV copies the value in register rs2 into register rd. C.MV expands into add rd, x0, rs2.

```text
C.ADD adds the values in registers rd and rs2 and writes the result to register rd. C.ADD expands
into add rd, rd, rs2.
      15                                      10 9                  7 6          5 4              2 1           0
                     funct6                           rd0 /rs10         funct              rs20           op
                        6                                 3               2                 3              2
                     C.AND                              dest           C.AND               src            C1
                      C.OR                              dest            C.OR               src            C1
                     C.XOR                              dest           C.XOR               src            C1
                     C.SUB                              dest           C.SUB               src            C1
                    C.ADDW                              dest          C.ADDW               src            C1
                    C.SUBW                              dest          C.SUBW               src            C1
```

These instructions use the CS format.

C.AND computes the bitwise AND of the values in registers rd0 and rs20 , then writes the result to
register rd0 . C.AND expands into and rd0 , rd0 , rs20 .

<!-- p.91 -->

C.OR computes the bitwise OR of the values in registers rd0 and rs20 , then writes the result to
register rd0 . C.OR expands into or rd0 , rd0 , rs20 .

C.XOR computes the bitwise XOR of the values in registers rd0 and rs20 , then writes the result to
register rd0 . C.XOR expands into xor rd0 , rd0 , rs20 .

C.SUB subtracts the value in register rs20 from the value in register rd0 , then writes the result to
register rd0 . C.SUB expands into sub rd0 , rd0 , rs20 .

C.ADDW is an RV64C/RV128C-only instruction that adds the values in registers rd0 and rs20 ,
then sign-extends the lower 32 bits of the sum before writing the result to register rd0 . C.ADDW
expands into addw rd0 , rd0 , rs20 .

C.SUBW is an RV64C/RV128C-only instruction that subtracts the value in register rs20 from the
value in register rd0 , then sign-extends the lower 32 bits of the difference before writing the result
to register rd0 . C.SUBW expands into subw rd0 , rd0 , rs20 .

        This group of six instructions do not provide large savings individually, but do not occupy much
        encoding space and are straightforward to implement, and as a group provide a worthwhile im-
        provement in static and dynamic compression.

```text
Defined Illegal Instruction
     15                13     12     11                         7 6                             2 1            0
               0              0                  0                              0                       0
               3               1                 5                              5                      2
               0               0                 0                              0                      0
```

A 16-bit instruction with all bits zero is permanently reserved as an illegal instruction.

        We reserve all-zero instructions to be illegal instructions to help trap attempts to execute zero-ed
        or non-existent portions of the memory space. The all-zero value should not be redefined in any
        non-standard extension. Similarly, we reserve instructions with all bits set to 1 (corresponding
        to very long instructions in the RISC-V variable-length encoding scheme) as illegal to capture
        another common value seen in non-existent memory regions.

```text
NOP Instruction
   15                13       12      11                          7 6                            2 1               0
          funct3            imm[5]             rd/rs1                        imm[4:0]                  op
            3                 1                  5                              5                      2
          C.NOP               0                  0                              0                      C1
```

C.NOP is a CI-format instruction that does not change any user-visible state, except for advancing
the pc. C.NOP is encoded as c.addi x0, 0 and so expands to addi x0, x0, 0.

<!-- p.92 -->

```text
Breakpoint Instruction
       15                 12 11                                                      2 1         0
              funct4                                    0                                  op
                 4                                      10                                  2
            C.EBREAK                                    0                                  C2
```

Debuggers can use the C.EBREAK instruction, which expands to ebreak, to cause control to be
transferred back to the debugging environment. C.EBREAK shares the opcode with the C.ADD
instruction, but with rd and rs2 both zero, thus can also use the CR format.

#### 12.6 Usage of C Instructions in LR/SC Sequences

On implementations that support the C extension, compressed forms of the I instructions permitted
inside LR/SC sequences can be used while retaining the guarantee of eventual success, as described
in Section 7.2.

     The implication is that any implementation that claims to support both the A and C extensions
     must ensure that LR/SC sequences containing valid C instructions will eventually complete.

<!-- p.93 -->

#### 12.7 RVC Instruction Set Listings

Table 12.3 shows a map of the major opcodes for RVC. Opcodes with the lower two bits set
correspond to instructions wider than 16 bits, including those in the base ISAs. Several instructions
are only valid for certain operands; when invalid, they are marked either RES to indicate that the
opcode is reserved for future standard extensions; NSE to indicate that the opcode is reserved
for non-standard extensions; or HINT to indicate that the opcode is reserved for future standard
microarchitectural hints. Instructions marked HINT must execute as no-ops on implementations
for which the hint has no effect.

      The HINT instructions are designed to support future addition of microarchitectural hints that
      might affect performance but cannot affect architectural state. The HINT encodings have been
      chosen so that simple implementations can ignore the HINT encoding and execute the HINT as
      a regular operation that does not change architectural state. For example, C.ADD is a HINT if
      the destination register is x0, where the five-bit rs2 field encodes details of the HINT. However,
      a simple implementation can simply execute the HINT as an add to register x0, which will have
      no effect.

```text
inst[15:13]
                000        001      010          011                100           101      110      111
  inst[1:0]
                          FLD           FLW                     FSD        FSW  RV32
        00 ADDI4SPN       FLD   LW       LD          Reserved   FSD   SW    SD  RV64
                          LQ             LD                      SQ         SD  RV128
                          JAL                                                   RV32
        01    ADDI       ADDIW   LI LUI/ADDI16SP    MISC-ALU      J  BEQZ BNEZ RV64
                         ADDIW                                                  RV128
                         FLDSP         FLWSP                   FSDSP      FSWSP RV32
        10     SLLI      FLDSP LWSP     LDSP     J[AL]R/MV/ADD FSDSP SWSP SDSP RV64
                          LQ            LDSP                     SQ        SDSP RV128
        11                                    >16b
```

**Table 12.3:** RVC opcode map

Tables 12.4–12.6 list the RVC instructions.

<!-- p.94 -->

```text
      15 14 13 12     11 10     9   8   7   6       5       4    3     2     1        0
        000                    0                                 0               00       Illegal instruction
        000             nzuimm[5:4|9:6|2|3]                     rd0              00       C.ADDI4SPN (RES, nzuimm=0)
        001       uimm[5:3]    rs10      uimm[7:6]              rd0              00       C.FLD (RV32/64)
        001      uimm[5:4|8]   rs10      uimm[7:6]              rd0              00       C.LQ (RV128)
        010       uimm[5:3]    rs10      uimm[2|6]              rd0              00       C.LW
        011       uimm[5:3]    rs10      uimm[2|6]              rd0              00       C.FLW (RV32)
        011       uimm[5:3]    rs10      uimm[7:6]              rd0              00       C.LD (RV64/128)
        100                          —                                           00       Reserved
        101       uimm[5:3]    rs10      uimm[7:6]              rs20             00       C.FSD (RV32/64)
        101      uimm[5:4|8]   rs10      uimm[7:6]              rs20             00       C.SQ (RV128)
        110       uimm[5:3]    rs10      uimm[2|6]              rs20             00       C.SW
                                   0
        111       uimm[5:3]    rs1       uimm[2|6]              rs20             00       C.FSW (RV32)
        111       uimm[5:3]    rs10      uimm[7:6]              rs20             00       C.SD (RV64/128)
```

**Table 12.4:** Instruction listing for RVC, Quadrant 0.

```text
     15 14 13       12        11 10 9   8   7   6       5   4   3      2     1        0
      000           0                 0                 0                        01       C.NOP
      000       nzimm[5]         rs1/rd6=0          nzimm[4:0]                   01       C.ADDI (HINT, nzimm=0)
      001                  imm[11|4|9:8|10|6|7|3:1|5]                            01       C.JAL (RV32)
      001         imm[5]         rs1/rd6=0           imm[4:0]                    01       C.ADDIW (RV64/128; RES, rd=0)
      010         imm[5]            rd6=0            imm[4:0]                    01       C.LI (HINT, rd=0)
      011        nzimm[9]             2          nzimm[4|6|8:7|5]                01       C.ADDI16SP (RES, nzimm=0)
      011       nzimm[17]        rd6={0, 2}        nzimm[16:12]                  01       C.LUI (RES, nzimm=0; HINT, rd=0)
                                          0    0
      100       nzuimm[5]      00      rs1 /rd     nzuimm[4:0]                   01       C.SRLI (RV32 NSE, nzuimm[5]=1)
      100            0         00      rs10 /rd0        0                        01       C.SRLI64 (RV128; RV32/64 HINT)
      100       nzuimm[5]      01      rs10 /rd0   nzuimm[4:0]                   01       C.SRAI (RV32 NSE, nzuimm[5]=1)
      100            0         01      rs10 /rd0        0                        01       C.SRAI64 (RV128; RV32/64 HINT)
      100         imm[5]       10      rs10 /rd0     imm[4:0]                    01       C.ANDI
      100            0         11      rs10 /rd0  00      rs20                   01       C.SUB
                                          0    0
      100            0         11      rs1 /rd    01      rs20                   01       C.XOR
                                          0    0
      100            0         11      rs1 /rd    10      rs20                   01       C.OR
                                          0    0
      100            0         11      rs1 /rd    11      rs20                   01       C.AND
                                          0    0
      100            1         11      rs1 /rd    00      rs20                   01       C.SUBW (RV64/128; RV32 RES)
                                          0    0
      100            1         11      rs1 /rd    01      rs20                   01       C.ADDW (RV64/128; RV32 RES)
      100            1         11         —       10       —                     01       Reserved
      100            1         11         —       11       —                     01       Reserved
      101                  imm[11|4|9:8|10|6|7|3:1|5]                            01       C.J
      110           imm[8|4:3]           rs10     imm[7:6|2:1|5]                 01       C.BEQZ
                                             0
      111           imm[8|4:3]           rs1      imm[7:6|2:1|5]                 01       C.BNEZ
```

**Table 12.5:** Instruction listing for RVC, Quadrant 1.

<!-- p.95 -->

```text
   15 14 13      12       11 10 9   8   7   6   5   4   3   2   1        0
     000      nzuimm[5]     rs1/rd6=0       nzuimm[4:0]             10       C.SLLI (HINT, rd=0; RV32 NSE, nzuimm[5]=1)
     000          0         rs1/rd6=0              0                10       C.SLLI64 (RV128; RV32/64 HINT; HINT, rd=0)
     001       uimm[5]          rd          uimm[4:3|8:6]           10       C.FLDSP (RV32/64)
     001       uimm[5]        rd6=0          uimm[4|9:6]            10       C.LQSP (RV128; RES, rd=0)
     010       uimm[5]        rd6=0         uimm[4:2|7:6]           10       C.LWSP (RES, rd=0)
     011       uimm[5]          rd          uimm[4:2|7:6]           10       C.FLWSP (RV32)
     011       uimm[5]        rd6=0         uimm[4:3|8:6]           10       C.LDSP (RV64/128; RES, rd=0)
     100          0           rs16=0               0                10       C.JR (RES, rs1=0)
     100          0           rd6=0             rs26=0              10       C.MV (HINT, rd=0)
     100          1              0                 0                10       C.EBREAK
     100          1           rs16=0               0                10       C.JALR
     100          1         rs1/rd6=0           rs26=0              10       C.ADD (HINT, rd=0)
     101            uimm[5:3|8:6]                 rs2               10       C.FSDSP (RV32/64)
     101            uimm[5:4|9:6]                 rs2               10       C.SQSP (RV128)
     110            uimm[5:2|7:6]                 rs2               10       C.SWSP
     111            uimm[5:2|7:6]                 rs2               10       C.FSWSP (RV32)
     111            uimm[5:3|8:6]                 rs2               10       C.SDSP (RV64/128)
```

**Table 12.6:** Instruction listing for RVC, Quadrant 2.

<!-- p.97 -->

### 第 13 章 “B” Standard Extension for Bit Manipulation, Version 0.0

This chapter is a placeholder for a future standard extension to provide bit manipulation instructions, including instructions to insert, extract, and test bit fields, and for rotations, funnel shifts,
and bit and byte permutations.

      Although bit manipulation instructions are very effective in some application domains, particu-
      larly when dealing with externally packed data structures, we excluded them from the base ISA
      as they are not useful in all domains and can add additional complexity or instruction formats
      to supply all needed operands.
          We anticipate the B extension will be a brownfield encoding within the base 30-bit instruction
      space.

<!-- p.99 -->

### 第 14 章 “J” Standard Extension for Dynamically Translated Languages, Version 0.0

This chapter is a placeholder for a future standard extension to support dynamically translated
languages.

     Many popular languages are usually implemented via dynamic translation, including Java and
     Javascript. These languages can benefit from additional ISA support for dynamic checks and
     garbage collection.

<!-- p.101 -->

### 第 15 章 “T” Standard Extension for Transactional Memory, Version 0.0

This chapter is a placeholder for a future standard extension to provide transactional memory
operations.

     Despite much research over the last twenty years, and initial commercial implementations, there
     is still much debate on the best way to support atomic operations involving multiple addresses.
         Our current thoughts are to include a small limited-capacity transactional memory buffer
     along the lines of the original transactional memory proposals.

<!-- p.103 -->

### 第 16 章 “P” Standard Extension for Packed-SIMD Instructions, Version 0.1

     Discussions at the 5th RISC-V workshop indicated a desire to drop this packed-SIMD proposal
     for floating-point registers in favor of standardizing on the V extension for large floating-point
     SIMD operations. However, there was interest in packed-SIMD fixed-point operations for use in
     the integer registers of small RISC-V implementations.

In this chapter, we outline a standard packed-SIMD extension for RISC-V. We’ve reserved the
instruction subset name “P” for a future standard set of packed-SIMD extensions. Many other
extensions can build upon a packed-SIMD extension, taking advantage of the wide data registers
and datapaths separate from the integer unit.

     Packed-SIMD extensions, first introduced with the Lincoln Labs TX-2 [9], have become a pop-
     ular way to provide higher throughput on data-parallel codes. Earlier commercial microproces-
     sor implementations include the Intel i860, HP PA-RISC MAX [19], SPARC VIS [29], MIPS
     MDMX [12], PowerPC AltiVec [8], Intel x86 MMX/SSE [24, 26], while recent designs include
     Intel x86 AVX [20] and ARM Neon [11]. We describe a standard framework for adding packed-
     SIMD in this chapter, but are not actively working on such a design. In our opinion, packed-
     SIMD designs represent a reasonable design point when reusing existing wide datapath resources,
     but if significant additional resources are to be devoted to data-parallel execution then designs
     based on traditional vector architectures are a better choice and should use the V extension.

A RISC-V packed-SIMD extension reuses the floating-point registers (f0-f31). These registers can
be defined to have widths of FLEN=32 to FLEN=1024. The standard floating-point instruction
subsets require registers of width 32 bits (“F”), 64 bits (“D”), or 128 bits (“Q”).

     It is natural to use the floating-point registers for packed-SIMD values rather than the integer
     registers (PA-RISC and Alpha packed-SIMD extensions) as this frees the integer registers for
     control and address values, simplifies reuse of scalar floating-point units for SIMD floating-
     point execution, and leads naturally to a decoupled integer/floating-point hardware design. The
     floating-point load and store instruction encodings also have space to handle wider packed-SIMD
     registers. However, reusing the floating-point registers for packed-SIMD values does make it
     more difficult to use a recoded internal format for floating-point values.

<!-- p.104 -->

The existing floating-point load and store instructions are used to load and store various-sized words
from memory to the f registers. The base ISA supports 32-bit and 64-bit loads and stores, but the
LOAD-FP and STORE-FP instruction encodings allows 8 different widths to be encoded as shown
in Table 16.1. When used with packed-SIMD operations, it is desirable to support non-naturally
aligned loads and stores in hardware.

```text
                                  width field   Code    Size in bits
                                     000        B                  8
                                     001        H                16
                                     010        W                32
                                     011        D                64
                                     100        Q               128
                                     101        Q2              256
                                     110        Q4              512
                                     111        Q8             1024
```

**Table 16.1:** LOAD-FP and STORE-FP width encoding.

Packed-SIMD computational instructions operate on packed values in f registers. Each value can
be 8-bit, 16-bit, 32-bit, 64-bit, or 128-bit, and both integer and floating-point representations can
be supported. For example, a 64-bit packed-SIMD extension can treat each register as 1×64-bit,
2×32-bit, 4×16-bit, or 8×8-bit packed values.

     Simple packed-SIMD extensions might fit in unused 32-bit instruction opcodes, but more exten-
     sive packed-SIMD extensions will likely require a dedicated 30-bit instruction space.

<!-- p.105 -->

### 第 17 章 “V” Standard Extension for Vector Operations, Version 0.2

This chapter presents a proposal for the RISC-V vector instruction set extension. The vector extension supports a configurable vector unit, to tradeoff the number of architectural vector registers
and supported element widths against available maximum vector length. The vector extension is
designed to allow the same binary code to work efficiently across a variety of hardware implementations varying in physical vector storage capacity and datapath parallelism.

     The vector extension is based on the style of vector register architecture introduced by Seymour
     Cray in the 1970s, as opposed to the earlier packed SIMD approach, introduced with the Lincoln
     Labs TX-2 in 1957 and now adopted by most other commercial instruction sets.
         The vector instruction set contains many features developed in earlier research projects,
     including the Berkeley T0 and VIRAM vector microprocessors, the MIT Scale vector-thread
     processor, and the Berkeley Maven and Hwacha projects.

#### 17.1 Vector Unit State

The additional vector unit architectural state consists of 32 vector data registers (v0–v31), 8 vector
predicate registers (vp0-vp7), and an XLEN-bit WARL vector length CSR, vl. In addition, the
current configuration of the vector unit is held in a set vector configuration CSRs (vcmaxw, vctype,
vcnpred), as described below. The implementation determines an available maximum vector length
(MVL) for the current configuration held in the vcmaxw and vcnpred registers. There is also a 3-bit
fixed-point rounding mode CSR vxrm, and a single-bit fixed-point saturation status CSR vxsat.

#### 17.2 Element Datatypes and Width

The datatypes and operations supported by the V extension depend upon the base scalar ISA and
supported extensions, and may include 8-bit, 16-bit, 32-bit, 64-bit, and 128-bit integer and fixedpoint data types (X8, X16, X32, X64, and X128 respectively), and 16-bit, 32-bit, 64-bit, and 128-bit

<!-- p.106 -->

```text
                           CSR name       Number      Base ISA
                           vl              0x020      RV32, RV64,    RV128
                           vxrm            0x020      RV32, RV64,    RV128
                           vxsat           0x020      RV32, RV64,    RV128
                           vcsr            0x020      RV32, RV64,    RV128
                           vcnpred         0x020      RV32, RV64,    RV128
                           vcmaxw          0x020      RV32, RV64,    RV128
                           vcmaxw1         0x020      RV32
                           vcmaxw2         0x020      RV32, RV64
                           vcmaxw3         0x020      RV32
                           vctype          0x020      RV32, RV64,    RV128
                           vctype1         0x020      RV32
                           vctype2         0x020      RV32, RV64
                           vctype3         0x020      RV32
                           vctypev0        0x020      RV32, RV64,    RV128
                           vctypev1        0x020      RV32, RV64,    RV128
                           ...
                           vctypev31       0x020      RV32, RV64, RV128
```

**Table 17.1:** Vector extension CSRs. Supported Fixed-Point Widths RV32I    X8, X16, X32 RV64I    X8, X16, X32, X64 RV128I X8, X16, X32, X64, X128 Supported Floating-Point Widths F        F16, F32 FD       F16, F32, F64 FDQ      F16, F32, F64, F128

**Table 17.2:** Supported data element widths depending on base integer ISA and supported floating- point extensions. Note that supporting a given floating-point width mandates support for all narrower floating-point widths.

floating-point types (F16, F32, F64, and F128 respectively). When the V extension is added, it
must support the vector data element types implied by the supported scalar types as defined by
Table 17.2. The largest element width supported:
                                    ELEN = max(XLEN, FLEN)

     Compiler support for vectorization is greatly simplified when any hardware-supported data types
     are supported by both scalar and vector instructions.
Adding the vector extension to any machine with floating-point support adds support for the IEEE
standard half-precision 16-bit floating-point data type. This includes a set of scalar half-precision
instructions described in Section ??. The scalar half-precision instructions follow the template for
other floating-point precisions, but using the hitherto unused fmt field encoding of 10.

     We only support scalar half-precision floating-point types as part of the vector extension, as the main benefits of half-precision are obtained when using vector instructions that amortize
     per-operation control overhead. Not supporting a separate scalar half-precision floating-point
     extension also reduces the number of standard instruction-set variants.

<!-- p.107 -->

#### 17.3 Vector Configuration Registers (vcmaxw, vctype, vcp)

The vector unit must be configured before use. Each architectural vector data register (v0–v31) is
configured with the maximum number of bits allowed in each element of that vector data register,
or can be disabled to free physical vector storage for other architectural vector data registers. The
number of available vector predicate registers can also be set independently.

The available MVL depends on the configuration setting, but MVL must always have the same
value for the same configuration parameters on a given implementation. Implementations must
provide an MVL of at least four elements for all supported configuration settings.

Each vector data register’s current maximum-width is held in a separate four-bit field in the vcmaxw
CSRs, encoded as shown in Table 17.3.

```text
                                           Width      Encoding
                                         Disabled       0000
                                                8       1000
                                               16       1001
                                               32       1010
                                               64       1011
                                              128       1100
```

**Table 17.3:** Encoding of vcmaxw fields. All other values are reserved.

     Several earlier vector machines had the ability to configure physical vector register storage into
     a larger number of short vectors or a shorter number of long vectors, in particular the Fujitsu
     VP series [21].

In addition, each vector data register has an associated dynamic type field that is held in a four-bit
field in the vctype CSRs, encoded as shown in Table 17.4. The dynamic type field of a vector
data register is constrained to only hold types that have equal or lesser width than the value in the
corresponding vcmaxw field for that vector data register. Changes to vctype do not alter MVL.

     Vector data registers have both a maximum element width and a current element data type to
     support vector function calls, where the caller does not know the types needed by the callee, as
     described below.

To reduce configuration time, writes to a vcmaxw field also write the corresponding vctype field.
The vcmaxw field can be written any value taken from the type encoding in Table 17.4, but only
the width information as shown in Table 17.3 will be recorded in the vcmaxw fields whereas the full
type information will be recorded in the corresponding vctype field.

Attempting to write any vcmaxw field with a width larger than that supported by the implementation will raise an illegal instruction exception. Implementations are allowed to record a vcmaxw

<!-- p.108 -->

```text
                         Type        vctype encoding      vcmaxw equivalent
                         Disabled         0000                  0000
                         F16              0001                  1001
                         F32              0010                  1010
                         F64              0011                  1011
                         F128             0100                  1100
                         X8               1000                  1000
                         X16              1001                  1001
                         X32              1010                  1010
                         X64              1011                  1011
                         X128             1100                  1100
```

**Table 17.4:** Encoding of vctype fields. The third column shows the value that will be saved when writing to vcmaxw fields. All other values are reserved.

value larger than the value requested. In particular, an implementation may choose to hardwire
vcmaxw fields to the largest supported width.

Attempting to write an unsupported type or a type that requires more than the current vcmaxw
width to a vctype field will raise an exception.

Any write to a field in the vcmaxw register configures the vector unit and causes all vector data
registers to be zeroed and all vector predicate registers to be set, and the vector length register vl
to be set to the maximum supported vector length.

Any write to a vctype field zeros only the associated vector data register, leaving the other vector
unit state undisturbed. Attempting to write a type needing more bits than the corresponding
vcmaxw value to a vctype field will raise an illegal instruction exception.

     Vector registers are zeroed on reconfiguration to prevent security holes and to avoid exposing
     differences between how different implementations manage physical vector register storage.
         In-order implementations will probaby use a flag bit per register to mux in 0 instead of
     garbage values on each source until it is overwritten. For in-order machines, partial writes
     due to predication or vector lengths less than MVL complicate this zeroing, but these cases can
     be handled by adopting a hardware read-modify-write, adding a zero bit per element, or a trap
     to machine-mode trap handler if first write access after configuration is partial. Out-of-order
     machines can just point initial rename table at physical zero register.

In RV128, vcmaxw is a single CSR holding 32 4-bit width fields. Bits (4N + 3)–(4N ) hold the
maximum width of vector data register N . In RV64, the vcmaxw2 CSR provides access to the
upper 64 bits of vcmaxw. In RV32, the vcmaxw1 CSR provides access to bits 63–32 of vcmaxw, while
vcmax3 CSR provides access to bits 127–96.

The vcnpred CSR contains a single 4-bit WLRL field giving the number of enabled architectural
predicate registers, between 0 and 8. Any write to vcnpred zeros all vector data registers, sets all
bits in visible vector predicate registers, and sets the vector length register vl to the maximum
supported vector length. Attempting to write a value larger than 8 to vcnpred raises an illegal
instruction exception.

<!-- p.109 -->

```text
                                       AVL Value                vl setting
                                     AVL ≥ 2 MVL                  MVL
                                  2 MVL > AVL > MVL             bAVL/2c
                                      MVL ≥ AVL                   AVL
```

**Table 17.5:** Operation of setvl instruction to set vector length register vl based on requested application vector length (AVL) and current maximum vector length (MVL).

#### 17.4 Vector Length

The active vector length is held in the XLEN-bit WARL vector length CSR vl, which can only hold
values between 0 and MVL inclusive. Any writes to the maximum configuration registers (vcmaxw
or vcnpred) cause vl to be initialized with MVL. Writes to vctype do not affect vl.

The active vector length is usually written with the setvl instruction, which is encoded as a csrrw
instruction to the vl CSR number. The source argument to the csrrw is the requested application
vector length (AVL) as an unsigned XLEN-bit integer. The setvl instruction calculates the value
to assign to vl according to Table 17.5.

     The rules for setting the vl register help keep vector pipelines full over the last two iterations of
     a stripmined loop. Similar rules were previously used in Cray-designed machines [7].

The result of this calculation is also returned as the result of the setvl instruction. Note that
unlike a regular csrrw instruction, the value written to integer register rd is not the original CSR
value but the modified value.

     The idea of having implementation-defined vector length dates back to at least the IBM 3090
     Vector Facility [5], which used a special “Load Vector Count and Update” (VLVCU) instruction
     to control stripmine loops. The setvl instruction included here is based on the simpler setvlr
     instruction introduced by Asanović [4].

The setvl instruction is typically used at the start of every iteration of a stripmined loop to set
the number of vector elements to operate on in the following loop iteration. The current MVL can
be obtained by performing a setvl with a source argument that has all bits set (largest unsigned
integer).

No element operations are performed for any vector instruction when vl=0.

#### 17.5 Rapid Configuration Instructions

It can take several instructions to set vcmaxw, vctype and vcnpred to a given configuration. To
accelerate configuring the vector unit, specialized vcfg instructions are added that are encoded as
writes to CSRs with encoded immediate values that set multiple fields in the vcmaxw, vctype, and
vncpred configuration registers.

The vcfgd instruction is encoded as a CSRRW that takes a register value encoded as shown in
Figure 17.2, and which returns the corresponding MVL in the destination register. A correspond-

<!-- p.110 -->

```text
                 #   Vector-vector 32-bit add loop.
                 #   Assume vector unit configured with correct types.
                 #   a0 holds N
                 #   a1 holds pointer to result vector
                 #   a2 holds pointer to first source vector
                 #   a3 holds pointer to second source vector.
            loop:    setvl t0, a0
                     vld v0, a2      # Load first vector
                     sll t1, t0, 2   # multiply by bytes
                     add a2, t1      # Bump pointer
                     vld v1, a3      # Load second vector
                     add a3, t1      # Bump pointer
                     vadd v0, v1     # Add elements
                     sub a0, t0      # Decrement elements completed
                     vst v0, a1      # Store result vector
                     add a1, t1      # Bump pointer
                     bnez a0, loop   # Any more?
```

**Figure 17.1:** Example vector-vector add loop.

ing vcfgdi instruction is encoded as a CSRRWI that takes a 5-bit immediate value to set the
configuration, and returns MVL in the destination register.

     One of the primary uses of vcfgdi is to configure the vector unit with single-byte element vectors
     for use in memcpy and memset routines. A single instruction can configure the vector unit for
     these operation.

The vcfgd instruction also clears the vcnpred register, so no predicate registers are allocated.

```text
                                                     0   F64    F32    F16     X32    X16     X8    RV32
                                                     2    5      5      5       5      5       5
                          0              F128     X64    F64    F32    F16     X32    X16     X8    RV64
                          24              5        5      5      5      5       5      5       5
              0                 X128     F128     X64    F64    F32    F16     X32    X16     X8    RV128
             83                  5        5        5      5      5      5       5      5       5
```

**Figure 17.2:** Format of the vcfgd value for different base ISAs, holding 5-bit vector register numbers for each supported type. Fields must either contain 0 indicating no vector registers are allocated for that type, or a vector register number greater than all to the right. All vector register numbers inbetween two non-zero fields are allocated to the type with the higher vector register number.

The vcfgd value specifies how many vector registers of each datatype are allocated, and is divided
into 5-bit fields, one per supported datatype. A value of 0 in a field indicates that no registers of
that type are allocated. A non-zero value indicates the highest vector

Each 5-bit field in the vcfgd value must contain either zero, indicating that no vector registers are
allocated for that type, or a vector register number greater than all fields in lower bit positions,
indicating the highest vector register containing the associated type. This encoding can compactly

<!-- p.111 -->

```text
                               0   F64   F32     F16    X32   X16     X8
                               0    18    12      0      1     0      0
                          Vector registers     vcmaxw   vctype       Type
                             v31–v19            0000     0000       Disabled
                             v18–v13            1011     0011         F64
                              v12–v2            1010     0010         F32
                               v1–v0            1010     1010         X32
```

**Figure 17.3:** Example use of vcfgd value to set configuration.

represent any arbitrary allocation of vector registers to data types, except that there must be
at least two vector registers (v0 and v1) allocated to the narrowest required type. An example
allocation is shown in Figure 17.3.

Separate vcfgp and vcfgpi instructions are provided, using the CSRRW and CSRRWI encodings
respectively, that write the source value to the vcnpred register and return the new MVL. These
writes also clear the vector data registers, set all bits in the allocated predicate registers, and set
vl=MVL. A vcfgp or vcfgpi instruction can be used after a vcfgd to complete a reconfiguration
of the vector unit.

If a zero argument is given to vcgfd the vector unit will be unconfigured with no enabled registers,
and the value 0 will be returned for MVL. Only the configuration registers vcmaxw and vcnpred
can be accessed in this state, either directly or via vcfgd, vcfgdi, vcfgp, or vcfgpi instructions.
Other vector instructions will raise an illegal instruction exception.

To quickly change the individual types of a vector register, each vector data register n has a dedicated CSR address to access its vctype field, named vctypevn. The vcfgt and vcfgti instructions
are assembler pseudo-instructions for regular CSRRW and CSRRWI instructions that update the
type fields and return the original value. The vcfgti instruction is typically used to change to a
desired type while recording the previous type in one instruction, and the vcfgt instruction is used
to revert back to the saved type.

<!-- p.113 -->

### 第 18 章 “N” Standard Extension for User-Level Interrupts, Version 1.1

     This is a placeholder for a more complete writeup of the N extension, and to form a basis for
     discussion.

This chapter presents a proposal for adding RISC-V user-level interrupt and exception handling.
When the N extension is present, and the outer execution environment has delegated designated
interrupts and exceptions to user-level, then hardware can transfer control directly to a user-level
trap handler without invoking the outer execution environment.

     User-level interrupts are primarily intended to support secure embedded systems with only M-
     mode and U-mode present, but can also be supported in systems running Unix-like operating
     systems to support user-level trap handling.
         When used in an Unix environment, the user-level interrupts would likely not replace conven-
     tional signal handling, but could be used as a building block for further extensions that generate
     user-level events such as garbage collection barriers, integer overflow, floating-point traps.

#### 18.1 Additional CSRs

The user-visible CSRs added to support the N extension are listed in Table 18.1.

```text
                  Number      Name          Description
                  0x000       ustatus       User status register.
                  0x004       uie           User interrupt-enable register.
                  0x005       utvec         User trap handler base address.
                  0x040       uscratch      Scratch register for user trap handlers.
                  0x041       uepc          User exception program counter.
                  0x042       ucause        User trap cause.
                  0x043       utval         User bad address or instruction.
                  0x044       uip           User interrupt pending.
```

**Table 18.1:** CSRs for N extension.

<!-- p.114 -->

#### 18.2 User Status Register (ustatus)

The ustatus register is an XLEN-bit read/write register formatted as shown in Figure 18.1. The
ustatus register keeps track of and controls the hart’s current operating state.

```text
               XLEN                                          5     4     3   1        0
                                  WPRI                           UPIE WPRI          UIE
                                  XLEN-5                          1    3             1
```

**Figure 18.1:** User-mode status register (ustatus).

The user interrupt-enable bit UIE disables user-level interrupts when clear. The value of UIE is
copied into UPIE when a user-level trap is taken, and the value of UIE is set to zero to provide
atomicity for the user-level trap handler.

      There is no UPP bit to hold the previous privilege mode as it can only be user mode.
The URET instructions is used to return from traps in U-mode, and URET copies UPIE into UIE,
then sets UPIE.

      UPIE is set after the UPIE/UIE stack is popped to enable interrupts and help catch coding
      errors.

#### 18.3 Other CSRs

The remaining CSRs function in an analogous way to the trap handling registers defined for M-mode
and S-mode.

      A more complete writeup to follow.

#### 18.4 N Extension Instructions

The URET instruction is added to perform the analogous function to MRET and SRET.

#### 18.5 Reducing Context-Swap Overhead

The user-level interrupt-handling registers add considerable state to the user-level context, yet will
usually rarely be active in normal use. In particular, uepc, ucause, and utval are only valid during
execution of a trap handler.

An NS field can be added to mstatus and sstatus following the format of the FS and XS fields
to reduce context-switch overhead when the values are not live. Execution of URET will place the
uepc, ucause, and utval back into initial state.

<!-- p.115 -->

### 第 19 章 RV32/64G Instruction Set Listings

One goal of the RISC-V project is that it be used as a stable software development target. For this
purpose, we define a combination of a base ISA (RV32I or RV64I) plus selected standard extensions
(IMAFD) as a “general-purpose” ISA, and we use the abbreviation G for the IMAFD combination
of instruction-set extensions. This chapter presents opcode maps and instruction-set listings for
RV32G and RV64G.

```text
   inst[4:2]  000    001      010      011     100    101          110        111
   inst[6:5]                                                                (> 32b)
         00  LOAD  LOAD-FP custom-0 MISC-MEM OP-IMM AUIPC      OP-IMM-32      48b
         01 STORE STORE-FP custom-1   AMO      OP     LUI         OP-32       64b
         10  MADD   MSUB   NMSUB     NMADD    OP-FP reserved custom-2/rv128   48b
         11 BRANCH  JALR   reserved    JAL   SYSTEM reserved custom-3/rv128  ≥ 80b
```

**Table 19.1:** RISC-V base opcode map, inst[1:0]=11

Table 19.1 shows a map of the major opcodes for RVG. Major opcodes with 3 or more lower bits
set are reserved for instruction lengths greater than 32 bits. Opcodes marked as reserved should be
avoided for custom instruction set extensions as they might be used by future standard extensions.
Major opcodes marked as custom-0 and custom-1 will be avoided by future standard extensions and
are recommended for use by custom instruction-set extensions within the base 32-bit instruction
format. The opcodes marked custom-2/rv128 and custom-3/rv128 are reserved for future use by
RV128, but will otherwise be avoided for standard extensions and so can also be used for custom
instruction-set extensions in RV32 and RV64.

We believe RV32G and RV64G provide simple but complete instruction sets for a broad range of
general-purpose computing. The optional compressed instruction set described in Chapter 12 can
be added (forming RV32GC and RV64GC) to improve performance, code size, and energy efficiency,
though with some additional hardware complexity.

As we move beyond IMAFDC into further instruction set extensions, the added instructions tend
to be more domain-specific and only provide benefits to a restricted class of applications, e.g., for
multimedia or security. Unlike most commercial ISAs, the RISC-V ISA design clearly separates
the base ISA and broadly applicable standard extensions from these more specialized additions.
Chapter 21 has a more extensive discussion of ways to add extensions to the RISC-V ISA.

<!-- p.116 -->

```text
      31         27   26 25 24         20   19         15   14   12   11      7     6             0
              funct7             rs2             rs1        funct3       rd             opcode        R-type
                   imm[11:0]                     rs1        funct3       rd             opcode        I-type
            imm[11:5]           rs2              rs1        funct3     imm[4:0]         opcode        S-type
           imm[12|10:5]         rs2              rs1        funct3    imm[4:1|11]       opcode        B-type
                            imm[31:12]                                   rd             opcode        U-type
                        imm[20|10:1|11|19:12]                            rd             opcode        J-type
                                  RV32I Base Instruction Set
                              imm[31:12]                        rd                      0110111       LUI
                              imm[31:12]                        rd                      0010111       AUIPC
                         imm[20|10:1|11|19:12]                  rd                      1101111       JAL
                    imm[11:0]                  rs1   000        rd                      1100111       JALR
           imm[12|10:5]           rs2          rs1   000    imm[4:1|11]                 1100011       BEQ
           imm[12|10:5]           rs2          rs1   001    imm[4:1|11]                 1100011       BNE
           imm[12|10:5]           rs2          rs1   100    imm[4:1|11]                 1100011       BLT
           imm[12|10:5]           rs2          rs1   101    imm[4:1|11]                 1100011       BGE
           imm[12|10:5]           rs2          rs1   110    imm[4:1|11]                 1100011       BLTU
           imm[12|10:5]           rs2          rs1   111    imm[4:1|11]                 1100011       BGEU
                    imm[11:0]                  rs1   000        rd                      0000011       LB
                    imm[11:0]                  rs1   001        rd                      0000011       LH
                    imm[11:0]                  rs1   010        rd                      0000011       LW
                    imm[11:0]                  rs1   100        rd                      0000011       LBU
                    imm[11:0]                  rs1   101        rd                      0000011       LHU
             imm[11:5]            rs2          rs1   000     imm[4:0]                   0100011       SB
             imm[11:5]            rs2          rs1   001     imm[4:0]                   0100011       SH
             imm[11:5]            rs2          rs1   010     imm[4:0]                   0100011       SW
                    imm[11:0]                  rs1   000        rd                      0010011       ADDI
                    imm[11:0]                  rs1   010        rd                      0010011       SLTI
                    imm[11:0]                  rs1   011        rd                      0010011       SLTIU
                    imm[11:0]                  rs1   100        rd                      0010011       XORI
                    imm[11:0]                  rs1   110        rd                      0010011       ORI
                    imm[11:0]                  rs1   111        rd                      0010011       ANDI
              0000000           shamt          rs1   001        rd                      0010011       SLLI
              0000000           shamt          rs1   101        rd                      0010011       SRLI
              0100000           shamt          rs1   101        rd                      0010011       SRAI
              0000000             rs2          rs1   000        rd                      0110011       ADD
              0100000             rs2          rs1   000        rd                      0110011       SUB
              0000000             rs2          rs1   001        rd                      0110011       SLL
              0000000             rs2          rs1   010        rd                      0110011       SLT
              0000000             rs2          rs1   011        rd                      0110011       SLTU
              0000000             rs2          rs1   100        rd                      0110011       XOR
              0000000             rs2          rs1   101        rd                      0110011       SRL
              0100000             rs2          rs1   101        rd                      0110011       SRA
              0000000             rs2          rs1   110        rd                      0110011       OR
              0000000             rs2          rs1   111        rd                      0110011       AND
            0000         pred       succ     00000   000      00000                     0001111       FENCE
            0000        0000        0000     00000   001      00000                     0001111       FENCE.I
                  000000000000               00000   000      00000                     1110011       ECALL
                  000000000001               00000   000      00000                     1110011       EBREAK
                       csr                     rs1   001        rd                      1110011       CSRRW
                       csr                     rs1   010        rd                      1110011       CSRRS
                       csr                     rs1   011        rd                      1110011       CSRRC
                       csr                   zimm    101        rd                      1110011       CSRRWI
                       csr                   zimm    110        rd                      1110011       CSRRSI
                       csr                   zimm    111        rd                      1110011       CSRRCI
   31           27   26   25   24          20   19         15   14   12   11        7   6             0
            funct7                   rs2             rs1        funct3      rd              opcode        R-type
                 imm[11:0]                           rs1        funct3      rd              opcode        I-type
          imm[11:5]                  rs2             rs1        funct3    imm[4:0]          opcode        S-type
                RV64I Base Instruction               Set (in addition to RV32I)
                imm[11:0]                            rs1       110       rd                 0000011       LWU
                imm[11:0]                            rs1       011       rd                 0000011       LD
          imm[11:5]           rs2                    rs1       011     imm[4:0]             0100011       SD
         000000           shamt                      rs1       001       rd                 0010011       SLLI
         000000           shamt                      rs1       101       rd                 0010011       SRLI
         010000           shamt                      rs1       101       rd                 0010011       SRAI
                imm[11:0]                            rs1       000       rd                 0011011       ADDIW
           0000000          shamt                    rs1       001       rd                 0011011       SLLIW
           0000000          shamt                    rs1       101       rd                 0011011       SRLIW
           0100000          shamt                    rs1       101       rd                 0011011       SRAIW
           0000000            rs2                    rs1       000       rd                 0111011       ADDW
           0100000            rs2                    rs1       000       rd                 0111011       SUBW
           0000000            rs2                    rs1       001       rd                 0111011       SLLW
           0000000            rs2                    rs1       101       rd                 0111011       SRLW
           0100000            rs2                    rs1       101       rd                 0111011       SRAW
                                    RV32M Standard Extension
           0000001                   rs2      rs1      000                     rd           0110011       MUL
           0000001                   rs2      rs1      001                     rd           0110011       MULH
           0000001                   rs2      rs1      010                     rd           0110011       MULHSU
           0000001                   rs2      rs1      011                     rd           0110011       MULHU
           0000001                   rs2      rs1      100                     rd           0110011       DIV
           0000001                   rs2      rs1      101                     rd           0110011       DIVU
           0000001                   rs2      rs1      110                     rd           0110011       REM
           0000001                   rs2      rs1      111                     rd           0110011       REMU
                RV64M Standard Extension (in addition to RV32M)
           0000001         rs2        rs1      000        rd                                0111011       MULW
           0000001         rs2        rs1      100        rd                                0111011       DIVW
           0000001         rs2        rs1      101        rd                                0111011       DIVUW
           0000001         rs2        rs1      110        rd                                0111011       REMW
           0000001         rs2        rs1      111        rd                                0111011       REMUW
                                    RV32A Standard Extension
        00010        aq   rl        00000     rs1      010                     rd           0101111       LR.W
        00011        aq   rl         rs2      rs1      010                     rd           0101111       SC.W
        00001        aq   rl         rs2      rs1      010                     rd           0101111       AMOSWAP.W
        00000        aq   rl         rs2      rs1      010                     rd           0101111       AMOADD.W
        00100        aq   rl         rs2      rs1      010                     rd           0101111       AMOXOR.W
        01100        aq   rl         rs2      rs1      010                     rd           0101111       AMOAND.W
        01000        aq   rl         rs2      rs1      010                     rd           0101111       AMOOR.W
        10000        aq   rl         rs2      rs1      010                     rd           0101111       AMOMIN.W
        10100        aq   rl         rs2      rs1      010                     rd           0101111       AMOMAX.W
        11000        aq   rl         rs2      rs1      010                     rd           0101111       AMOMINU.W
        11100        aq   rl         rs2      rs1      010                     rd           0101111       AMOMAXU.W
      31           27      26   25   24          20   19         15   14   12   11     7   6             0
                  funct7                   rs2             rs1        funct3      rd           opcode        R-type
            rs3      funct2                rs2             rs1        funct3      rd           opcode        R4-type
                   imm[11:0]                               rs1        funct3      rd           opcode        I-type
             imm[11:5]                     rs2             rs1        funct3    imm[4:0]       opcode        S-type
                        RV64A Standard Extension (in addition to RV32A)
           00010         aq rl    00000       rs1      011        rd                           0101111       LR.D
           00011         aq rl     rs2        rs1      011        rd                           0101111       SC.D
           00001         aq rl     rs2        rs1      011        rd                           0101111       AMOSWAP.D
           00000         aq rl     rs2        rs1      011        rd                           0101111       AMOADD.D
           00100         aq rl     rs2        rs1      011        rd                           0101111       AMOXOR.D
           01100         aq rl     rs2        rs1      011        rd                           0101111       AMOAND.D
           01000         aq rl     rs2        rs1      011        rd                           0101111       AMOOR.D
           10000         aq rl     rs2        rs1      011        rd                           0101111       AMOMIN.D
           10100         aq rl     rs2        rs1      011        rd                           0101111       AMOMAX.D
           11000         aq rl     rs2        rs1      011        rd                           0101111       AMOMINU.D
           11100         aq rl     rs2        rs1      011        rd                           0101111       AMOMAXU.D
                                          RV32F Standard Extension
                    imm[11:0]                        rs1     010                  rd           0000111       FLW
              imm[11:5]                    rs2       rs1     010                imm[4:0]       0100111       FSW
            rs3         00                 rs2       rs1     rm                   rd           1000011       FMADD.S
            rs3         00                 rs2       rs1     rm                   rd           1000111       FMSUB.S
            rs3         00                 rs2       rs1     rm                   rd           1001011       FNMSUB.S
            rs3         00                 rs2       rs1     rm                   rd           1001111       FNMADD.S
               0000000                     rs2       rs1     rm                   rd           1010011       FADD.S
               0000100                     rs2       rs1     rm                   rd           1010011       FSUB.S
               0001000                     rs2       rs1     rm                   rd           1010011       FMUL.S
               0001100                     rs2       rs1     rm                   rd           1010011       FDIV.S
               0101100                    00000      rs1     rm                   rd           1010011       FSQRT.S
               0010000                     rs2       rs1     000                  rd           1010011       FSGNJ.S
               0010000                     rs2       rs1     001                  rd           1010011       FSGNJN.S
               0010000                     rs2       rs1     010                  rd           1010011       FSGNJX.S
               0010100                     rs2       rs1     000                  rd           1010011       FMIN.S
               0010100                     rs2       rs1     001                  rd           1010011       FMAX.S
               1100000                    00000      rs1     rm                   rd           1010011       FCVT.W.S
               1100000                    00001      rs1     rm                   rd           1010011       FCVT.WU.S
               1110000                    00000      rs1     000                  rd           1010011       FMV.X.W
               1010000                     rs2       rs1     010                  rd           1010011       FEQ.S
               1010000                     rs2       rs1     001                  rd           1010011       FLT.S
               1010000                     rs2       rs1     000                  rd           1010011       FLE.S
               1110000                    00000      rs1     001                  rd           1010011       FCLASS.S
               1101000                    00000      rs1     rm                   rd           1010011       FCVT.S.W
               1101000                    00001      rs1     rm                   rd           1010011       FCVT.S.WU
               1111000                    00000      rs1     000                  rd           1010011       FMV.W.X
    31       27   26 25      24          20   19         15   14   12   11     7   6             0
            funct7                 rs2             rs1        funct3      rd           opcode        R-type
         rs3       funct2          rs2             rs1        funct3      rd           opcode        R4-type
                 imm[11:0]                         rs1        funct3      rd           opcode        I-type
          imm[11:5]                rs2             rs1        funct3    imm[4:0]       opcode        S-type
               RV64F Standard Extension (in addition to RV32F)
          1100000        00010       rs1      rm        rd                             1010011       FCVT.L.S
          1100000        00011       rs1      rm        rd                             1010011       FCVT.LU.S
          1101000        00010       rs1      rm        rd                             1010011       FCVT.S.L
          1101000        00011       rs1      rm        rd                             1010011       FCVT.S.LU
                                  RV32D Standard Extension
                imm[11:0]                   rs1      011     rd                        0000111       FLD
          imm[11:5]                rs2      rs1      011   imm[4:0]                    0100111       FSD
         rs3        01             rs2      rs1      rm      rd                        1000011       FMADD.D
         rs3        01             rs2      rs1      rm      rd                        1000111       FMSUB.D
         rs3        01             rs2      rs1      rm      rd                        1001011       FNMSUB.D
         rs3        01             rs2      rs1      rm      rd                        1001111       FNMADD.D
           0000001                 rs2      rs1      rm      rd                        1010011       FADD.D
           0000101                 rs2      rs1      rm      rd                        1010011       FSUB.D
           0001001                 rs2      rs1      rm      rd                        1010011       FMUL.D
           0001101                 rs2      rs1      rm      rd                        1010011       FDIV.D
           0101101                00000     rs1      rm      rd                        1010011       FSQRT.D
           0010001                 rs2      rs1      000     rd                        1010011       FSGNJ.D
           0010001                 rs2      rs1      001     rd                        1010011       FSGNJN.D
           0010001                 rs2      rs1      010     rd                        1010011       FSGNJX.D
           0010101                 rs2      rs1      000     rd                        1010011       FMIN.D
           0010101                 rs2      rs1      001     rd                        1010011       FMAX.D
           0100000                00001     rs1      rm      rd                        1010011       FCVT.S.D
           0100001                00000     rs1      rm      rd                        1010011       FCVT.D.S
           1010001                 rs2      rs1      010     rd                        1010011       FEQ.D
           1010001                 rs2      rs1      001     rd                        1010011       FLT.D
           1010001                 rs2      rs1      000     rd                        1010011       FLE.D
           1110001                00000     rs1      001     rd                        1010011       FCLASS.D
           1100001                00000     rs1      rm      rd                        1010011       FCVT.W.D
           1100001                00001     rs1      rm      rd                        1010011       FCVT.WU.D
           1101001                00000     rs1      rm      rd                        1010011       FCVT.D.W
           1101001                00001     rs1      rm      rd                        1010011       FCVT.D.WU
               RV64D Standard Extension (in addition to RV32D)
          1100001        00010       rs1      rm        rd                             1010011       FCVT.L.D
          1100001        00011       rs1      rm        rd                             1010011       FCVT.LU.D
          1110001        00000       rs1      000       rd                             1010011       FMV.X.D
          1101001        00010       rs1      rm        rd                             1010011       FCVT.D.L
          1101001        00011       rs1      rm        rd                             1010011       FCVT.D.LU
          1111001        00000       rs1      000       rd                             1010011       FMV.D.X
```

<!-- p.119 -->

**Table 19.2:** Instruction listing for RISC-V Table 19.3 lists the CSRs that have currently been allocated CSR addresses. The timers, counters, and floating-point CSRs are the only CSRs defined in this specification.

<!-- p.120 -->

```text
 Number    Privilege     Name        Description
                           Floating-Point Control and Status Registers
 0x001     Read/write    fflags      Floating-Point Accrued Exceptions.
 0x002     Read/write    frm         Floating-Point Dynamic Rounding Mode.
 0x003     Read/write    fcsr        Floating-Point Control and Status Register (frm + fflags).
                                      Counters and Timers
 0xC00     Read-only     cycle       Cycle counter for RDCYCLE instruction.
 0xC01     Read-only     time        Timer for RDTIME instruction.
 0xC02     Read-only     instret     Instructions-retired counter for RDINSTRET instruction.
 0xC80     Read-only     cycleh      Upper 32 bits of cycle, RV32I only.
 0xC81     Read-only     timeh       Upper 32 bits of time, RV32I only.
 0xC82     Read-only     instreth Upper 32 bits of instret, RV32I only.
```

**Table 19.3:** RISC-V control and status register (CSR) address map.

<!-- p.121 -->

### 第 20 章 RISC-V Assembly Programmer’s Handbook

This chapter is a placeholder for an assembly programmer’s manual.

Table 20.1 lists the assembler mnemonics for the x and f registers and their role in the standard
calling convention.

```text
               Register   ABI Name     Description                         Saver
               x0         zero         Hard-wired zero                     —
               x1         ra           Return address                      Caller
               x2         sp           Stack pointer                       Callee
               x3         gp           Global pointer                      —
               x4         tp           Thread pointer                      —
               x5         t0           Temporary/alternate link register   Caller
               x6–7       t1–2         Temporaries                         Caller
               x8         s0/fp        Saved register/frame pointer        Callee
               x9         s1           Saved register                      Callee
               x10–11     a0–1         Function arguments/return values    Caller
               x12–17     a2–7         Function arguments                  Caller
               x18–27     s2–11        Saved registers                     Callee
               x28–31     t3–6         Temporaries                         Caller
               f0–7       ft0–7        FP temporaries                      Caller
               f8–9       fs0–1        FP saved registers                  Callee
               f10–11     fa0–1        FP arguments/return values          Caller
               f12–17     fa2–7        FP arguments                        Caller
               f18–27     fs2–11       FP saved registers                  Callee
               f28–31     ft8–11       FP temporaries                      Caller
```

**Table 20.1:** Assembler mnemonics for RISC-V integer and floating-point registers.

Tables 20.2 and 20.3 contain a listing of standard RISC-V pseudoinstructions.

<!-- p.122 -->

```text
 Pseudoinstruction             Base Instruction(s)               Meaning
                               auipc rd, symbol[31:12]
 la rd, symbol                                                   Load address
                               addi rd, rd, symbol[11:0]
                               auipc rd, symbol[31:12]
 l{b|h|w|d} rd, symbol                                           Load global
                               l{b|h|w|d} rd, symbol[11:0](rd)
                               auipc rt, symbol[31:12]
 s{b|h|w|d} rd, symbol, rt                                       Store global
                               s{b|h|w|d} rd, symbol[11:0](rt)
                               auipc rt, symbol[31:12]
 fl{w|d} rd, symbol, rt                                          Floating-point load global
                               fl{w|d} rd, symbol[11:0](rt)
                               auipc rt, symbol[31:12]
 fs{w|d} rd, symbol, rt                                          Floating-point store global
                               fs{w|d} rd, symbol[11:0](rt)
 nop                           addi x0, x0, 0                    No operation
 li rd, immediate              Myriad sequences                  Load immediate
 mv rd, rs                     addi rd, rs, 0                    Copy register
 not rd, rs                    xori rd, rs, -1                   One’s complement
 neg rd, rs                    sub rd, x0, rs                    Two’s complement
 negw rd, rs                   subw rd, x0, rs                   Two’s complement word
 sext.w rd, rs                 addiw rd, rs, 0                   Sign extend word
 seqz rd, rs                   sltiu rd, rs, 1                   Set if = zero
 snez rd, rs                   sltu rd, x0, rs                   Set if 6= zero
 sltz rd, rs                   slt rd, rs, x0                    Set if < zero
 sgtz rd, rs                   slt rd, x0, rs                    Set if > zero
 fmv.s rd, rs                  fsgnj.s rd, rs, rs                Copy single-precision register
 fabs.s rd, rs                 fsgnjx.s rd, rs, rs               Single-precision absolute value
 fneg.s rd, rs                 fsgnjn.s rd, rs, rs               Single-precision negate
 fmv.d rd, rs                  fsgnj.d rd, rs, rs                Copy double-precision register
 fabs.d rd, rs                 fsgnjx.d rd, rs, rs               Double-precision absolute value
 fneg.d rd, rs                 fsgnjn.d rd, rs, rs               Double-precision negate
 beqz rs, offset               beq rs, x0, offset                Branch if = zero
 bnez rs, offset               bne rs, x0, offset                Branch if 6= zero
 blez rs, offset               bge x0, rs, offset                Branch if ≤ zero
 bgez rs, offset               bge rs, x0, offset                Branch if ≥ zero
 bltz rs, offset               blt rs, x0, offset                Branch if < zero
 bgtz rs, offset               blt x0, rs, offset                Branch if > zero
 bgt rs, rt, offset            blt rt, rs, offset                Branch if >
 ble rs, rt, offset            bge rt, rs, offset                Branch if ≤
 bgtu rs, rt, offset           bltu rt, rs, offset               Branch if >, unsigned
 bleu rs, rt, offset           bgeu rt, rs, offset               Branch if ≤, unsigned
 j offset                      jal x0, offset                    Jump
 jal offset                    jal x1, offset                    Jump and link
 jr rs                         jalr x0, rs, 0                    Jump register
 jalr rs                       jalr x1, rs, 0                    Jump and link register
 ret                           jalr x0, x1, 0                    Return from subroutine
                               auipc x6, offset[31:12]
 call offset                                                     Call far-away subroutine
                               jalr x1, x6, offset[11:0]
                               auipc x6, offset[31:12]
 tail offset                                                     Tail call far-away subroutine
                               jalr x0, x6, offset[11:0]
 fence                         fence iorw, iorw                  Fence on all memory and I/O
```

**Table 20.2:** RISC-V pseudoinstructions.

<!-- p.123 -->

```text
      Pseudoinstruction   Base Instruction             Meaning
      rdinstret[h] rd     csrrs rd, instret[h], x0     Read instructions-retired counter
      rdcycle[h] rd       csrrs rd, cycle[h], x0       Read cycle counter
      rdtime[h] rd        csrrs rd, time[h], x0        Read real-time clock
      csrr rd, csr        csrrs rd, csr, x0            Read CSR
      csrw csr, rs        csrrw x0, csr, rs            Write CSR
      csrs csr, rs        csrrs x0, csr, rs            Set bits in CSR
      csrc csr, rs        csrrc x0, csr, rs            Clear bits in CSR
      csrwi csr, imm      csrrwi x0, csr, imm          Write CSR, immediate
      csrsi csr, imm      csrrsi x0, csr, imm          Set bits in CSR, immediate
      csrci csr, imm      csrrci x0, csr, imm          Clear bits in CSR, immediate
      frcsr rd            csrrs rd, fcsr, x0           Read FP control/status register
      fscsr rd, rs        csrrw rd, fcsr, rs           Swap FP control/status register
      fscsr rs            csrrw x0, fcsr, rs           Write FP control/status register
      frrm rd             csrrs rd, frm, x0            Read FP rounding mode
      fsrm rd, rs         csrrw rd, frm, rs            Swap FP rounding mode
      fsrm rs             csrrw x0, frm, rs            Write FP rounding mode
      fsrmi rd, imm       csrrwi rd, frm, imm          Swap FP rounding mode, immediate
      fsrmi imm           csrrwi x0, frm, imm          Write FP rounding mode, immediate
      frflags rd          csrrs rd, fflags, x0         Read FP exception flags
      fsflags rd, rs      csrrw rd, fflags, rs         Swap FP exception flags
      fsflags rs          csrrw x0, fflags, rs         Write FP exception flags
      fsflagsi rd, imm    csrrwi rd, fflags, imm       Swap FP exception flags, immediate
      fsflagsi imm        csrrwi x0, fflags, imm       Write FP exception flags, immediate
```

**Table 20.3:** Pseudoinstructions for accessing control and status registers.

<!-- p.125 -->

### 第 21 章 Extending RISC-V

In addition to supporting standard general-purpose software development, another goal of RISC-V
is to provide a basis for more specialized instruction-set extensions or more customized accelerators.
The instruction encoding spaces and optional variable-length instruction encoding are designed to
make it easier to leverage software development effort for the standard ISA toolchain when building
more customized processors. For example, the intent is to continue to provide full software support
for implementations that only use the standard I base, perhaps together with many non-standard
instruction-set extensions.

This chapter describes various ways in which the base RISC-V ISA can be extended, together
with the scheme for managing instruction-set extensions developed by independent groups. This
volume only deals with the user-level ISA, although the same approach and terminology is used for
supervisor-level extensions described in the second volume.

#### 21.1 Extension Terminology

This section defines some standard terminology for describing RISC-V extensions.

Standard versus Non-Standard Extension

Any RISC-V processor implementation must support a base integer ISA (RV32I or RV64I). In
addition, an implementation may support one or more extensions. We divide extensions into two
broad categories: standard versus non-standard.

- A standard extension is one that is generally useful and that is designed to not conflict with any other standard extension. Currently, “MAFDQLCBTPV”, described in other chapters of this manual, are either complete or planned standard extensions.

- A non-standard extension may be highly specialized and may conflict with other standard or non-standard extensions. We anticipate a wide variety of non-standard extensions will be developed over time, with some eventually being promoted to standard extensions.

<!-- p.126 -->

Instruction Encoding Spaces and Prefixes

An instruction encoding space is some number of instruction bits within which a base ISA or
ISA extension is encoded. RISC-V supports varying instruction lengths, but even within a single
instruction length, there are various sizes of encoding space available. For example, the base ISA
is defined within a 30-bit encoding space (bits 31–2 of the 32-bit instruction), while the atomic
extension “A” fits within a 25-bit encoding space (bits 31–7).

We use the term prefix to refer to the bits to the right of an instruction encoding space (since
RISC-V is little-endian, the bits to the right are stored at earlier memory addresses, hence form a
prefix in instruction-fetch order). The prefix for the standard base ISA encoding is the two-bit “11”
field held in bits 1–0 of the 32-bit word, while the prefix for the standard atomic extension “A”
is the seven-bit “0101111” field held in bits 6–0 of the 32-bit word representing the AMO major
opcode. A quirk of the encoding format is that the 3-bit funct3 field used to encode a minor opcode
is not contiguous with the major opcode bits in the 32-bit instruction format, but is considered
part of the prefix for 22-bit instruction spaces.

Although an instruction encoding space could be of any size, adopting a smaller set of common
sizes simplifies packing independently developed extensions into a single global encoding. Table 21.1
gives the suggested sizes for RISC-V.

```text
  Size                      Usage                      # Available in standard instruction length
                                                       16-bit 32-bit 48-bit          64-bit
 14-bit   Quadrant of compressed 16-bit encoding            3
 22-bit   Minor opcode in base 32-bit encoding                    28      220                 235
 25-bit   Major opcode in base 32-bit encoding                    32      217                 232
 30-bit   Quadrant of base 32-bit encoding                          1     2 12                227
 32-bit   Minor opcode in 48-bit encoding                                 210                 225
 37-bit   Major opcode in 48-bit encoding                                  32                 220
 40-bit   Quadrant of 48-bit encoding                                        4                217
 45-bit   Sub-minor opcode in 64-bit encoding                                                 212
 48-bit   Minor opcode in 64-bit encoding                                                      29
 52-bit   Major opcode in 64-bit encoding                                                      32
```

**Table 21.1:** Suggested standard RISC-V instruction encoding space sizes.

Greenfield versus Brownfield Extensions

We use the term greenfield extension to describe an extension that begins populating a new instruction encoding space, and hence can only cause encoding conflicts at the prefix level. We
use the term brownfield extension to describe an extension that fits around existing encodings in
a previously defined instruction space. A brownfield extension is necessarily tied to a particular
greenfield parent encoding, and there may be multiple brownfield extensions to the same greenfield
parent encoding. For example, the base ISAs are greenfield encodings of a 30-bit instruction space,
while the FDQ floating-point extensions are all brownfield extensions adding to the parent base
ISA 30-bit encoding space.

<!-- p.127 -->

Note that we consider the standard A extension to have a greenfield encoding as it defines a new
previously empty 25-bit encoding space in the leftmost bits of the full 32-bit base instruction
encoding, even though its standard prefix locates it within the 30-bit encoding space of the base
ISA. Changing only its single 7-bit prefix could move the A extension to a different 30-bit encoding
space while only worrying about conflicts at the prefix level, not within the encoding space itself.

```text
                                          Adds state           No new state
                        Greenfield   RV32I(30), RV64I(30)         A(25)
                        Brownfield     F(I), D(F), Q(D)           M(I)
```

**Table 21.2:** Two-dimensional characterization of standard instruction-set extensions.

Table 21.2 shows the bases and standard extensions placed in a simple two-dimensional taxonomy.
One axis is whether the extension is greenfield or brownfield, while the other axis is whether the
extension adds architectural state. For greenfield extensions, the size of the instruction encoding
space is given in parentheses. For brownfield extensions, the name of the extension (greenfield or
brownfield) it builds upon is given in parentheses. Additional user-level architectural state usually
implies changes to the supervisor-level system or possibly to the standard calling convention.

Note that RV64I is not considered an extension of RV32I, but a different complete base encoding.

Standard-Compatible Global Encodings

A complete or global encoding of an ISA for an actual RISC-V implementation must allocate a
unique non-conflicting prefix for every included instruction encoding space. The bases and every
standard extension have each had a standard prefix allocated to ensure they can all coexist in a
global encoding.

A standard-compatible global encoding is one where the base and every included standard extension
have their standard prefixes. A standard-compatible global encoding can include non-standard
extensions that do not conflict with the included standard extensions. A standard-compatible
global encoding can also use standard prefixes for non-standard extensions if the associated standard
extensions are not included in the global encoding. In other words, a standard extension must use
its standard prefix if included in a standard-compatible global encoding, but otherwise its prefix is
free to be reallocated. These constraints allow a common toolchain to target the standard subset
of any RISC-V standard-compatible global encoding.

Guaranteed Non-Standard Encoding Space

To support development of proprietary custom extensions, portions of the encoding space are
guaranteed to never be used by standard extensions.

<!-- p.128 -->

#### 21.2 RISC-V Extension Design Philosophy

We intend to support a large number of independently developed extensions by encouraging extension developers to operate within instruction encoding spaces, and by providing tools to pack
these into a standard-compatible global encoding by allocating unique prefixes. Some extensions
are more naturally implemented as brownfield augmentations of existing extensions, and will share
whatever prefix is allocated to their parent greenfield extension. The standard extension prefixes
avoid spurious incompatibilities in the encoding of core functionality, while allowing custom packing
of more esoteric extensions.

This capability of repacking RISC-V extensions into different standard-compatible global encodings
can be used in a number of ways.

One use-case is developing highly specialized custom accelerators, designed to run kernels from
important application domains. These might want to drop all but the base integer ISA and add
in only the extensions that are required for the task in hand. The base ISA has been designed to
place minimal requirements on a hardware implementation, and has been encoded to use only a
small fraction of a 32-bit instruction encoding space.

Another use-case is to build a research prototype for a new type of instruction-set extension. The
researchers might not want to expend the effort to implement a variable-length instruction-fetch
unit, and so would like to prototype their extension using a simple 32-bit fixed-width instruction
encoding. However, this new extension might be too large to coexist with standard extensions in
the 32-bit space. If the research experiments do not need all of the standard extensions, a standardcompatible global encoding might drop the unused standard extensions and reuse their prefixes to
place the proposed extension in a non-standard location to simplify engineering of the research
prototype. Standard tools will still be able to target the base and any standard extensions that are
present to reduce development time. Once the instruction-set extension has been evaluated and
refined, it could then be made available for packing into a larger variable-length encoding space to
avoid conflicts with all standard extensions.

The following sections describe increasingly sophisticated strategies for developing implementations
with new instruction-set extensions. These are mostly intended for use in highly customized, educational, or experimental architectures rather than for the main line of RISC-V ISA development.

#### 21.3 Extensions within fixed-width 32-bit instruction format

In this section, we discuss adding extensions to implementations that only support the base fixedwidth 32-bit instruction format.

      We anticipate the simplest fixed-width 32-bit encoding will be popular for many restricted accel-
      erators and research prototypes.

<!-- p.129 -->

Available 30-bit instruction encoding spaces

In the standard encoding, three of the available 30-bit instruction encoding spaces (those with 2-bit
prefixes 00, 01, and 10) are used to enable the optional compressed instruction extension. However,
if the compressed instruction-set extension is not required, then these three further 30-bit encoding
spaces become available. This quadruples the available encoding space within the 32-bit format.

Available 25-bit instruction encoding spaces

A 25-bit instruction encoding space corresponds to a major opcode in the base and standard
extension encodings.

There are four major opcodes expressly reserved for custom extensions (Table 19.1), each of which
represents a 25-bit encoding space. Two of these are reserved for eventual use in the RV128 base
encoding (will be OP-IMM-64 and OP-64), but can be used for standard or non-standard extensions
for RV32 and RV64.

The two opcodes reserved for RV64 (OP-IMM-32 and OP-32) can also be used for standard and
non-standard extensions to RV32 only.

If an implementation does not require floating-point, then the seven major opcodes reserved for
standard floating-point extensions (LOAD-FP, STORE-FP, MADD, MSUB, NMSUB, NMADD,
OP-FP) can be reused for non-standard extensions. Similarly, the AMO major opcode can be
reused if the standard atomic extensions are not required.

If an implementation does not require instructions longer than 32-bits, then an additional four
major opcodes are available (those marked in gray in Table 19.1).

The base RV32I encoding uses only 11 major opcodes plus 3 reserved opcodes, leaving up to 18
available for extensions. The base RV64I encoding uses only 13 major opcodes plus 3 reserved
opcodes, leaving up to 16 available for extensions.

Available 22-bit instruction encoding spaces

A 22-bit encoding space corresponds to a funct3 minor opcode space in the base and standard
extension encodings. Several major opcodes have a funct3 field minor opcode that is not completely
occupied, leaving available several 22-bit encoding spaces.

Usually a major opcode selects the format used to encode operands in the remaining bits of the
instruction, and ideally, an extension should follow the operand format of the major opcode to
simplify hardware decoding.

Other spaces

Smaller spaces are available under certain major opcodes, and not all minor opcodes are entirely
filled.

<!-- p.130 -->

#### 21.4 Adding aligned 64-bit instruction extensions

The simplest approach to provide space for extensions that are too large for the base 32-bit fixedwidth instruction format is to add naturally aligned 64-bit instructions. The implementation must
still support the 32-bit base instruction format, but can require that 64-bit instructions are aligned
on 64-bit boundaries to simplify instruction fetch, with a 32-bit NOP instruction used as alignment
padding where necessary.

To simplify use of standard tools, the 64-bit instructions should be encoded as described in Figure 1.1. However, an implementation might choose a non-standard instruction-length encoding for
64-bit instructions, while retaining the standard encoding for 32-bit instructions. For example, if
compressed instructions are not required, then a 64-bit instruction could be encoded using one or
more zero bits in the first two bits of an instruction.

      We anticipate processor generators that produce instruction-fetch units capable of automatically
      handling any combination of supported variable-length instruction encodings.

#### 21.5 Supporting VLIW encodings

Although RISC-V was not designed as a base for a pure VLIW machine, VLIW encodings can be
added as extensions using several alternative approaches. In all cases, the base 32-bit encoding has
to be supported to allow use of any standard software tools.

Fixed-size instruction group

The simplest approach is to define a single large naturally aligned instruction format (e.g., 128 bits)
within which VLIW operations are encoded. In a conventional VLIW, this approach would tend
to waste instruction memory to hold NOPs, but a RISC-V-compatible implementation would have
to also support the base 32-bit instructions, confining the VLIW code size expansion to VLIWaccelerated functions.

Encoded-Length Groups

Another approach is to use the standard length encoding from Figure 1.1 to encode parallel instruction groups, allowing NOPs to be compressed out of the VLIW instruction. For example,
a 64-bit instruction could hold two 28-bit operations, while a 96-bit instruction could hold three
28-bit operations, and so on. Alternatively, a 48-bit instruction could hold one 42-bit operation,
while a 96-bit instruction could hold two 42-bit operations, and so on.

This approach has the advantage of retaining the base ISA encoding for instructions holding a
single operation, but has the disadvantage of requiring a new 28-bit or 42-bit encoding for operations
within the VLIW instructions, and misaligned instruction fetch for larger groups. One simplification
is to not allow VLIW instructions to straddle certain microarchitecturally significant boundaries
(e.g., cache lines or virtual memory pages).

<!-- p.131 -->

Fixed-Size Instruction Bundles

Another approach, similar to Itanium, is to use a larger naturally aligned fixed instruction bundle
size (e.g., 128 bits) across which parallel operation groups are encoded. This simplifies instruction
fetch, but shifts the complexity to the group execution engine. To remain RISC-V compatible, the
base 32-bit instruction would still have to be supported.

End-of-Group bits in Prefix

None of the above approaches retains the RISC-V encoding for the individual operations within
a VLIW instruction. Yet another approach is to repurpose the two prefix bits in the fixed-width
32-bit encoding. One prefix bit can be used to signal “end-of-group” if set, while the second bit
could indicate execution under a predicate if clear. Standard RISC-V 32-bit instructions generated
by tools unaware of the VLIW extension would have both prefix bits set (11) and thus have the
correct semantics, with each instruction at the end of a group and not predicated.

The main disadvantage of this approach is that the base ISA lacks the complex predication support
usually required in an aggressive VLIW system, and it is difficult to add space to specify more
predicate registers in the standard 30-bit encoding space.

<!-- p.133 -->

### 第 22 章 ISA Subset Naming Conventions

This chapter describes the RISC-V ISA subset naming scheme that is used to concisely describe
the set of instructions present in a hardware implementation, or the set of instructions used by an
application binary interface (ABI).

     The RISC-V ISA is designed to support a wide variety of implementations with various exper-
     imental instruction-set extensions. We have found that an organized naming scheme simplifies
     software tools and documentation.

#### 22.1 Case Sensitivity

The ISA naming strings are case insensitive.

#### 22.2 Base Integer ISA

RISC-V ISA strings begin with either RV32I, RV32E, RV64I, or RV128I indicating the supported
address space size in bits for the base integer ISA.

#### 22.3 Instruction Extensions Names

Standard ISA extensions are given a name consisting of a single letter. For example, the first
four standard extensions to the integer bases are: “M” for integer multiplication and division, “A”
for atomic memory instructions, “F” for single-precision floating-point instructions, and “D” for
double-precision floating-point instructions. Any RISC-V instruction set variant can be succinctly
described by concatenating the base integer prefix with the names of the included extensions. For
example, “RV64IMAFD”.

We have also defined an abbreviation “G” to represent the “IMAFD” base and extensions, as this
is intended to represent our standard general-purpose ISA.

<!-- p.134 -->

Standard extensions to the RISC-V ISA are given other reserved letters, e.g., “Q” for quad-precision
floating-point, or “C” for the 16-bit compressed instruction format.

#### 22.4 Version Numbers

Recognizing that instruction sets may expand or alter over time, we encode subset version numbers
following the subset name. Version numbers are divided into major and minor version numbers,
separated by a “p”. If the minor version is “0”, then “p0” can be omitted from the version string.
Changes in major version numbers imply a loss of backwards compatibility, whereas changes in only
the minor version number must be backwards-compatible. For example, the original 64-bit standard
ISA defined in release 1.0 of this manual can be written in full as “RV64I1p0M1p0A1p0F1p0D1p0”,
more concisely as “RV64I1M1A1F1D1”, or even more concisely as “RV64G1”. The G ISA subset
can be written as “RV64I2p0M2p0A2p0F2p0D2p0”, or more concisely “RV64G2”.

We introduced the version numbering scheme with the second release, which we also intend to
become a permanent standard. Hence, we define the default version of a standard subset to be
that present at the time of this document, e.g., “RV32G” is equivalent to “RV32I2M2A2F2D2”.

#### 22.5 Non-Standard Extension Names

Non-standard subsets are named using a single “X” followed by a name beginning with a letter
and an optional version number. For example, “Xhwacha” names the Hwacha vector-fetch ISA
extension; “Xhwacha2” and “Xhwacha2p0” name version 2.0 of same.

Non-standard extensions must be separated from other multi-letter extensions by a single underscore. For example, an ISA with non-standard extensions Argle and Bargle may be named
“RV64GXargle Xbargle”.

#### 22.6 Supervisor-level Instruction Subsets

Standard supervisor instruction subsets are defined in Volume II, but are named using “S” as a
prefix, followed by a supervisor subset name beginning with a letter and an optional version number.

Supervisor extensions must be separated from other multi-letter extensions by a single underscore.

#### 22.7 Supervisor-level Extensions

Non-standard extensions to the supervisor-level ISA are defined using the “SX” prefix.

<!-- p.135 -->

#### 22.8 Subset Naming Convention

Table 22.1 summarizes the standardized subset names.

```text
                      Subset                                   Name
                                Standard General-Purpose ISA
                      Integer                                   I
                      Integer Multiplication and Division       M
                      Atomics                                   A
                      Single-Precision Floating-Point           F
                      Double-Precision Floating-Point           D
                      General                              G = IMAFD
                               Standard User-Level Extensions
                      Quad-Precision Floating-Point             Q
                      Decimal Floating-Point                    L
                      16-bit Compressed Instructions            C
                      Bit Manipulation                          B
                      Dynamic Languages                         J
                      Transactional Memory                      T
                      Packed-SIMD Extensions                    P
                      Vector Extensions                         V
                      User-Level Interrupts                     N
                             Non-Standard User-Level Extensions
                      Non-standard extension “abc”            Xabc
                                Standard Supervisor-Level ISA
                      Supervisor extension “def”               Sdef
                          Non-Standard Supervisor-Level Extensions
                      Supervisor extension “ghi”              SXghi
```

**Table 22.1:** Standard ISA subset names. The table also defines the canonical order in which subset names must appear in the name string, with top-to-bottom in table indicating first-to-last in the name string, e.g., RV32IMAFDQC is legal, whereas RV32IMAFDCQ is not.

<!-- p.137 -->

### 第 23 章 History and Acknowledgments

#### 23.1 History from Revision 1.0 of ISA manual

The RISC-V ISA and instruction set manual builds up several earlier projects. Several aspects of
the supervisor-level machine and the overall format of the manual date back to the T0 (Torrent-0)
vector microprocessor project at UC Berkeley and ICSI, begun in 1992. T0 was a vector processor
based on the MIPS-II ISA, with Krste Asanović as main architect and RTL designer, and Brian
Kingsbury and Bertrand Irrisou as principal VLSI implementors. David Johnson at ICSI was a
major contributor to the T0 ISA design, particularly supervisor mode, and to the manual text.
John Hauser also provided considerable feedback on the T0 ISA design.

The Scale (Software-Controlled Architecture for Low Energy) project at MIT, begun in 2000, built
upon the T0 project infrastructure, refined the supervisor-level interface, and moved away from the
MIPS scalar ISA by dropping the branch delay slot. Ronny Krashinsky and Christopher Batten
were the principal architects of the Scale Vector-Thread processor at MIT, while Mark Hampton
ported the GCC-based compiler infrastructure and tools for Scale.

A lightly edited version of the T0 MIPS scalar processor specification (MIPS-6371) was used in
teaching a new version of the MIT 6.371 Introduction to VLSI Systems class in the Fall 2002
semester, with Chris Terman and Krste Asanović as lecturers. Chris Terman contributed most
of the lab material for the class (there was no TA!). The 6.371 class evolved into the trial 6.884
Complex Digital Design class at MIT, taught by Arvind and Krste Asanović in Spring 2005, which
became a regular Spring class 6.375. A reduced version of the Scale MIPS-based scalar ISA, named
SMIPS, was used in 6.884/6.375. Christopher Batten was the TA for the early offerings of these
classes and developed a considerable amount of documentation and lab material based around the
SMIPS ISA. This same SMIPS lab material was adapted and enhanced by TA Yunsup Lee for
the UC Berkeley Fall 2009 CS250 VLSI Systems Design class taught by John Wawrzynek, Krste
Asanović, and John Lazzaro.

The Maven (Malleable Array of Vector-thread ENgines) project was a second-generation vectorthread architecture. Its design was led by Christopher Batten when he was an Exchange Scholar
at UC Berkeley starting in summer 2007. Hidetaka Aoki, a visiting industrial fellow from Hitachi,
gave considerable feedback on the early Maven ISA and microarchitecture design. The Maven infrastructure was based on the Scale infrastructure but the Maven ISA moved further away from
the MIPS ISA variant defined in Scale, with a unified floating-point and integer register file. Maven
was designed to support experimentation with alternative data-parallel accelerators. Yunsup Lee
was the main implementor of the various Maven vector units, while Rimas Avižienis was the main
implementor of the various Maven scalar units. Yunsup Lee and Christopher Batten ported GCC
to work with the new Maven ISA. Christopher Celio provided the initial definition of a traditional
vector instruction set (“Flood”) variant of Maven.

<!-- p.138 -->

Based on experience with all these previous projects, the RISC-V ISA definition was begun in
Summer 2010. An initial version of the RISC-V 32-bit instruction subset was used in the UC
Berkeley Fall 2010 CS250 VLSI Systems Design class, with Yunsup Lee as TA. RISC-V is a clean
break from the earlier MIPS-inspired designs. John Hauser contributed to the floating-point ISA
definition, including the sign-injection instructions and a register encoding scheme that permits
internal recoding of floating-point values.

#### 23.2 History from Revision 2.0 of ISA manual

Multiple implementations of RISC-V processors have been completed, including several silicon
fabrications, as shown in Figure 23.1.

```text
           Name        Tapeout Date             Process                 ISA
          Raven-1         May 29, 2011      ST 28nm FDSOI       RV64G1 Xhwacha1
          EOS14            April 1, 2012    IBM 45nm SOI        RV64G1p1 Xhwacha2
          EOS16         August 17, 2012     IBM 45nm SOI        RV64G1p1 Xhwacha2
          Raven-2       August 22, 2012     ST 28nm FDSOI       RV64G1p1 Xhwacha2
          EOS18        February 6, 2013     IBM 45nm SOI        RV64G1p1 Xhwacha2
          EOS20             July 3, 2013    IBM 45nm SOI        RV64G1p99 Xhwacha2
          Raven-3    September 26, 2013     ST 28nm SOI         RV64G1p99 Xhwacha2
          EOS22           March 7, 2014     IBM 45nm SOI        RV64G1p9999 Xhwacha3
```

**Table 23.1:** Fabricated RISC-V testchips.

The first RISC-V processors to be fabricated were written in Verilog and manufactured in a preproduction 28 nm FDSOI technology from ST as the Raven-1 testchip in 2011. Two cores were
developed by Yunsup Lee and Andrew Waterman, advised by Krste Asanović, and fabricated
together: 1) an RV64 scalar core with error-detecting flip-flops, and 2) an RV64 core with an
attached 64-bit floating-point vector unit. The first microarchitecture was informally known as
“TrainWreck”, due to the short time available to complete the design with immature design libraries.

Subsequently, a clean microarchitecture for an in-order decoupled RV64 core was developed by
Andrew Waterman, Rimas Avižienis, and Yunsup Lee, advised by Krste Asanović, and, continuing
the railway theme, was codenamed “Rocket” after George Stephenson’s successful steam locomotive
design. Rocket was written in Chisel, a new hardware design language developed at UC Berkeley.
The IEEE floating-point units used in Rocket were developed by John Hauser, Andrew Waterman,
and Brian Richards. Rocket has since been refined and developed further, and has been fabricated
two more times in 28 nm FDSOI (Raven-2, Raven-3), and five times in IBM 45 nm SOI technology
(EOS14, EOS16, EOS18, EOS20, EOS22) for a photonics project. Work is ongoing to make the Rocket design available as a parameterized RISC-V processor generator.

<!-- p.139 -->

EOS14–EOS22 chips include early versions of Hwacha, a 64-bit IEEE floating-point vector unit,
developed by Yunsup Lee, Andrew Waterman, Huy Vo, Albert Ou, Quan Nguyen, and Stephen
Twigg, advised by Krste Asanović. EOS16–EOS22 chips include dual cores with a cache-coherence
protocol developed by Henry Cook and Andrew Waterman, advised by Krste Asanović. EOS14
silicon has successfully run at 1.25 GHz. EOS16 silicon suffered from a bug in the IBM pad libraries.
EOS18 and EOS20 have successfully run at 1.35 GHz.

Contributors to the Raven testchips include Yunsup Lee, Andrew Waterman, Rimas Avižienis,
Brian Zimmer, Jaehwa Kwak, Ruzica Jevtić, Milovan Blagojević, Alberto Puggelli, Steven Bailey,
Ben Keller, Pi-Feng Chiu, Brian Richards, Borivoje Nikolić, and Krste Asanović.

Contributors to the EOS testchips include Yunsup Lee, Rimas Avižienis, Andrew Waterman, Henry
Cook, Huy Vo, Daiwei Li, Chen Sun, Albert Ou, Quan Nguyen, Stephen Twigg, Vladimir Stojanović, and Krste Asanović.

Andrew Waterman and Yunsup Lee developed the C++ ISA simulator “Spike”, used as a golden
model in development and named after the golden spike used to celebrate completion of the US
transcontinental railway. Spike has been made available as a BSD open-source project.

Andrew Waterman completed a Master’s thesis with a preliminary design of the RISC-V compressed
instruction set [33].

Various FPGA implementations of the RISC-V have been completed, primarily as part of integrated
demos for the Par Lab project research retreats. The largest FPGA design has 3 cache-coherent
RV64IMA processors running a research operating system. Contributors to the FPGA implementations include Andrew Waterman, Yunsup Lee, Rimas Avižienis, and Krste Asanović.

RISC-V processors have been used in several classes at UC Berkeley. Rocket was used in the Fall
2011 offering of CS250 as a basis for class projects, with Brian Zimmer as TA. For the undergraduate
CS152 class in Spring 2012, Christopher Celio used Chisel to write a suite of educational RV32
processors, named “Sodor” after the island on which “Thomas the Tank Engine” and friends live.
The suite includes a microcoded core, an unpipelined core, and 2, 3, and 5-stage pipelined cores,
and is publicly available under a BSD license. The suite was subsequently updated and used again
in CS152 in Spring 2013, with Yunsup Lee as TA, and in Spring 2014, with Eric Love as TA.
Christopher Celio also developed an out-of-order RV64 design known as BOOM (Berkeley Out-ofOrder Machine), with accompanying pipeline visualizations, that was used in the CS152 classes.
The CS152 classes also used cache-coherent versions of the Rocket core developed by Andrew
Waterman and Henry Cook.

Over the summer of 2013, the RoCC (Rocket Custom Coprocessor) interface was defined to simplify adding custom accelerators to the Rocket core. Rocket and the RoCC interface were used
extensively in the Fall 2013 CS250 VLSI class taught by Jonathan Bachrach, with several student
accelerator projects built to the RoCC interface. The Hwacha vector unit has been rewritten as a
RoCC coprocessor.

Two Berkeley undergraduates, Quan Nguyen and Albert Ou, have successfully ported Linux to run
on RISC-V in Spring 2013.

<!-- p.140 -->

Colin Schmidt successfully completed an LLVM backend for RISC-V 2.0 in January 2014.

Darius Rad at Bluespec contributed soft-float ABI support to the GCC port in March 2014.

John Hauser contributed the definition of the floating-point classification instructions.

We are aware of several other RISC-V core implementations, including one in Verilog by Tommy
Thorn, and one in Bluespec by Rishiyur Nikhil.

Acknowledgments

Thanks to Christopher F. Batten, Preston Briggs, Christopher Celio, David Chisnall, Stefan
Freudenberger, John Hauser, Ben Keller, Rishiyur Nikhil, Michael Taylor, Tommy Thorn, and
Robert Watson for comments on the draft ISA version 2.0 specification.

#### 23.3 History for Revision 2.1

Uptake of the RISC-V ISA has been very rapid since the introduction of the frozen version 2.0 in
May 2014, with too much activity to record in a short history section such as this. Perhaps the
most important single event was the formation of the non-profit RISC-V Foundation in August
2015. The Foundation will now take over stewardship of the official RISC-V ISA standard, and the
official website riscv.org is the best place to obtain news and updates on the RISC-V standard.

Acknowledgments

Thanks to Scott Beamer, Allen J. Baum, Christopher Celio, David Chisnall, Paul Clayton, Palmer
Dabbelt, Jan Gray, Michael Hamburg, and John Hauser for comments on the version 2.0 specification.

#### 23.4 History for Revision 2.2

Acknowledgments

Thanks to Jacob Bachmeyer, Alex Bradbury, David Horner, Stefan O’Rear, and Joseph Myers for
comments on the version 2.1 specification.

<!-- p.141 -->

#### 23.5 Funding

Development of the RISC-V architecture and implementations has been partially funded by the
following sponsors.

- Par Lab: Research supported by Microsoft (Award #024263) and Intel (Award #024894) funding and by matching funding by U.C. Discovery (Award #DIG07-10227). Additional support came from Par Lab affiliates Nokia, NVIDIA, Oracle, and Samsung.

- Project Isis: DoE Award DE-SC0003624.

- ASPIRE Lab: DARPA PERFECT program, Award HR0011-12-2-0016. DARPA POEM program Award HR0011-11-C-0100. The Center for Future Architectures Research (C-FAR), a STARnet center funded by the Semiconductor Research Corporation. Additional support from ASPIRE industrial sponsor, Intel, and ASPIRE affiliates, Google, Hewlett Packard Enterprise, Huawei, Nokia, NVIDIA, Oracle, and Samsung.

The content of this paper does not necessarily reflect the position or the policy of the US government
and no official endorsement should be inferred.

<!-- p.143 -->

Bibliography

 [1] RISC-V ELF psABI Specification. https://github.com/riscv/riscv-elf-psabi-doc/.

 [2] IEEE standard for a 32-bit microprocessor. IEEE Std. 1754-1994, 1994.

 [3] G. M. Amdahl, G. A. Blaauw, and Jr. F. P. Brooks. Architecture of the IBM System/360.
     IBM Journal of R. & D., 8(2), 1964.

 [4] Krste Asanović. Vector Microprocessors. PhD thesis, University of California at Berkeley, May
     1998. Available as techreport UCB/CSD-98-1014.

 [5] W. Buchholz. The IBM System/370 vector architecture. IBM Systems Journal, 25(1):51–62,
     1986.

 [6] Werner Buchholz, editor. Planning a computer system: Project Stretch. McGraw-Hill Book
     Company, 1962.

 [7] Cray Inc. Cray Assembly Language (CAL) for Cray X1 Systems Reference Manual, 1.1 edition,
     June 2003.

 [8] K. Diefendorff, P.K. Dubey, R. Hochsprung, and H. Scale. AltiVec extension to PowerPC
     accelerates media processing. IEEE Micro, 20(2):85 –95, 2000.

 [9] John M. Frankovich and H. Philip Peterson. A functional description of the Lincoln TX-2
     computer. In Western Joint Computer Conference, Los Angeles, CA, February 1957.

[10] Kourosh Gharachorloo, Daniel Lenoski, James Laudon, Phillip Gibbons, Anoop Gupta, and
     John Hennessy. Memory consistency and event ordering in scalable shared-memory multipro-
     cessors. In In Proceedings of the 17th Annual International Symposium on Computer Archi-
     tecture, pages 15–26, 1990.

[11] J. Goodacre and A.N. Sloss. Parallelism and the ARM instruction set architecture. Computer,
     38(7):42 – 50, 2005.

[12] Linley Gwennap. Digital, MIPS add multimedia extensions. Microprocessor Report, 1996.

[13] Timothy H. Heil and James E. Smith. Selective dual path execution. Technical report, Uni-
     versity of Wisconsin - Madison, November 1996.

[14] ANSI/IEEE Std 754-2008, IEEE standard for floating-point arithmetic, 2008.

[15] Manolis G.H. Katevenis, Robert W. Sherburne, Jr., David A. Patterson, and Carlo H. Séquin.
     The RISC II micro-architecture. In Proceedings VLSI 83 Conference, August 1983.

<!-- p.144 -->

[16] Hyesoon Kim, Onur Mutlu, Jared Stark, and Yale N. Patt. Wish branches: Combining condi-
     tional branching and predication for adaptive predicated execution. In Proceedings of the 38th
     annual IEEE/ACM International Symposium on Microarchitecture, MICRO 38, pages 43–54,
     2005.
[17] A. Klauser, T. Austin, D. Grunwald, and B. Calder. Dynamic hammock predication for non-
     predicated instruction set architectures. In Proceedings of the 1998 International Conference
     on Parallel Architectures and Compilation Techniques, PACT ’98, Washington, DC, USA,
     1998.
[18] David D. Lee, Shing I. Kong, Mark D. Hill, George S. Taylor, David A. Hodges, Randy H.
     Katz, and David A. Patterson. A VLSI chip set for a multiprocessor workstation–Part I: An
     RISC microprocessor with coprocessor interface and support for symbolic processing. IEEE
     JSSC, 24(6):1688–1698, December 1989.
[19] R.B. Lee. Subword parallelism with MAX-2. IEEE Micro, 16(4):51 –59, August 1996.
[20] Chris Lomont. Introduction to Intel Advanced Vector Extensions. Intel White Paper, 2011.
[21] Kenichi Miura and Keiichiro Uchida. FACOM Vector Processor System: VP-100/VP-200. In
     Kawalik, editor, Proceedings of NATO Advanced Research Workshop on High Speed Comput-
     ing, volume F7. Springer-Verlag, 1984. Also in: IEEE Tutorial Supercomputers: Design and
     Applications. Kai Hwang(editor), pp59-73.
[22] OpenCores. OpenRISC 1000 architecture manual, architecture version 1.0, December 2012.
[23] David A. Patterson and Carlo H. Séquin. RISC I: A reduced instruction set VLSI computer.
     In ISCA, pages 443–458, 1981.
[24] A. Peleg and U. Weiser. MMX technology extension to the Intel architecture. IEEE Micro,
     16(4):42 –50, August 1996.
[25] Ravi Rajwar and James R. Goodman. Speculative lock elision: enabling highly concurrent mul-
     tithreaded execution. In Proceedings of the 34th annual ACM/IEEE International Symposium
     on Microarchitecture, MICRO 34, pages 294–305. IEEE Computer Society, 2001.
[26] S.K. Raman, V. Pentkovski, and J. Keshava. Implementing streaming SIMD extensions on
     the Pentium-III processor. IEEE Micro, 20(4):47 –57, 2000.
[27] Balaram Sinharoy, R. Kalla, W. J. Starke, H. Q. Le, R. Cargnoni, J. A. Van Norstrand,
     B. J. Ronchetti, J. Stuecheli, J. Leenstra, G. L. Guthrie, D. Q. Nguyen, B. Blaner, C. F.
     Marino, E. Retter, and P. Williams. IBM POWER7 multicore server processor. IBM Journal
     of Research and Development, 55(3):1–1, 2011.
[28] James E. Thornton. Parallel operation in the Control Data 6600. In Proceedings of the October
     27-29, 1964, Fall Joint Computer Conference, Part II: Very High Speed Computer Systems,
     AFIPS ’64 (Fall, part II), pages 33–40, 1965.
[29] M. Tremblay, J.M. O’Connor, V. Narayanan, and Liang He. VIS speeds new media processing.
     IEEE Micro, 16(4):10 –20, August 1996.
[30] Marc Tremblay, Jeffrey Chan, Shailender Chaudhry, Andrew W. Conigliaro, and Shing Sheung
     Tse. The MAJC architecture: A synthesis of parallelism and scalability. IEEE Micro, 20(6):12–
     25, 2000.

<!-- p.145 -->

[31] J. Tseng and K. Asanović. Energy-efficient register access. In Proc. of the 13th Symposium on
     Integrated Circuits and Systems Design, pages 377–384, Manaus, Brazil, September 2000.

[32] David Ungar, Ricki Blau, Peter Foley, Dain Samples, and David Patterson. Architecture of
     SOAR: Smalltalk on a RISC. In ISCA, pages 188–197, Ann Arbor, MI, 1984.

[33] Andrew Waterman. Improving Energy Efficiency and Reducing Code Size with RISC-V Com-
     pressed. Master’s thesis, University of California, Berkeley, 2011.

[34] Andrew Waterman. Design of the RISC-V Instruction Set Architecture. PhD thesis, University
     of California, Berkeley, 2016.

[35] Andrew Waterman, Yunsup Lee, David A. Patterson, and Krste Asanović. The RISC-V
     instruction set manual, Volume I: Base user-level ISA. Technical Report UCB/EECS-2011-62,
     EECS Department, University of California, Berkeley, May 2011.

[36] Andrew Waterman, Yunsup Lee, David A. Patterson, and Krste Asanović. The RISC-V in-
     struction set manual, Volume I: Base user-level ISA version 2.0. Technical Report UCB/EECS-
     2014-54, EECS Department, University of California, Berkeley, May 2014.

<!-- p.146 -->

         The RISC-V Instruction Set Manual
              Volume II: Privileged Architecture
                  Document Version 1.12-draft

    Editors: Andrew Waterman1 , Krste Asanović1,2 , John Hauser
                           1
                             SiFive Inc.,
 2
   CS Division, EECS Department, University of California, Berkeley
andrew@sifive.com, krste@berkeley.edu, jh.riscv@jhauser.us
                       September 15, 2021 Contributors to all versions of the spec in alphabetical order (please contact editors to suggest
corrections): Krste Asanović, Peter Ashenden, Rimas Avižienis, Jacob Bachmeyer, Allen J. Baum,
Jonathan Behrens, Paolo Bonzini, Ruslan Bukin, Christopher Celio, Chuanhua Chang, David Chisnall, Anthony Coulter, Palmer Dabbelt, Monte Dalrymple, Greg Favor, Dennis Ferguson, Marc
Gauthier, Andy Glew, Gary Guo, Mike Frysinger, John Hauser, David Horner, Olof Johansson,
David Kruckemyer, Yunsup Lee, Daniel Lustig, Andrew Lutomirski, Prashanth Mundkur, Jonathan
Neuschäfer, Rishiyur Nikhil, Stefan O’Rear, Albert Ou, John Ousterhout, David Patterson, Dmitri
Pavlov, Kade Phillips, Josh Scheid, Colin Schmidt, Michael Taylor, Wesley Terpstra, Matt Thomas,
Tommy Thorn, Ray VanDeWalker, Megan Wachs, Steve Wallach, Andrew Waterman, Clifford Wolf,
and Reinoud Zandijk.

<!-- p.147 -->

This document is released under a Creative Commons Attribution 4.0 International License.

This document is a derivative of the RISC-V privileged specification version 1.9.1 released under
following license: ⃝
                   c 2010–2017 Andrew Waterman, Yunsup Lee, Rimas Avižienis, David Patterson,
Krste Asanović. Creative Commons Attribution 4.0 International License.

Please cite as: “The RISC-V Instruction Set Manual, Volume II: Privileged Architecture, Document Version 1.12-draft”, Editors Andrew Waterman and Krste Asanović, RISC-V Foundation,
November 2020.

<!-- p.148 -->

## Preface

This is a draft of version 1.12 of the RISC-V privileged architecture proposal. The document
contains the following versions of the RISC-V ISA modules:

```text
                                  Module           Version   Status
                               Machine ISA         1.12      Draft
                              Supervisor ISA       1.12      Draft
                             Svnapot Extension     0.1       Draft
                             Svpbmt Extension      0.1       Draft
                              Hypervisor ISA       0.6       Draft
                                N Extension        1.1       Draft
```

The Machine and Supervisor ISAs, version 1.11, have been ratified by the RISC-V Foundation.
Version 1.12 of these modules, described in this document, is a minor revision to version 1.11.

The following changes have been made since version 1.11, which, while not strictly backwards
compatible, are not anticipated to cause software portability problems in practice:
- Changed MRET and SRET to clear mstatus.MPRV when leaving M-mode.
- Reserved additional satp patterns for future use.
- Stated that the scause Exception Code field must implement bits 4–0 at minimum.
- Relaxed I/O regions have been specified to follow RVWMO. The previous specification implied that PPO rules other than fences and acquire/release annotations did not apply.
- Constrained the LR/SC reservation set size and shape when using page-based virtual memory.
- PMP changes require an SFENCE.VMA on any hart that implements page-based virtual memory, even if VM is not currently enabled.
- Allowed for speculative updates of page table entry A bits.
- Clarify that PTEs with reserved bits set and non-leaf PTEs with D, A, or U set must trigger page-fault exceptions when accessed by the address-translation algorithm.

Additionally, the following compatible changes have been made since version 1.11:
- Moved N extension into its own chapter.
- Defined the RV32-only CSR mstatush, which contains most of the same fields as the upper 32 bits of RV64’s mstatus.
- Permitted the unconditional delegation of less-privileged interrupts.
- Added optional big-endian and bi-endian support.
- Made priority of load/store/AMO address-misaligned exceptions implementation-defined relative to load/store/AMO page-fault and access-fault exceptions.

                                               i • PMP reset values are now platform-defined.
- An additional 48 optional PMP registers have been defined.
- Added the Svnapot Standard Extension draft, along with the N bit in Sv39, Sv48, and Sv57 PTEs
- Added the Svpbmt Standard Extension draft, along with the PBMT bits in Sv39, Sv48, and Sv57 PTEs.
- Described the behavior of address-translation caches a little more explicitly.
- Slightly relaxed the atomicity requirement for A and D bit updates performed by the implementation.
- Added Sv57 and Sv57x4 address translation modes.
- Software breakpoint exceptions are permitted to write either 0 or the PC to x tval.

<!-- p.149 -->

Finally, the hypervisor architecture proposal has been extensively revised.

<!-- p.150 -->

Preface to Version 1.11

This is version 1.11 of the RISC-V privileged architecture. The document contains the following
versions of the RISC-V ISA modules:

```text
                                  Module          Version    Status
                               Machine ISA        1.11      Ratified
                              Supervisor ISA      1.11      Ratified
                               Hypervisor ISA     0.3        Draft
Changes from version 1.10 include:
   •   Moved Machine and Supervisor spec to Ratified status.
   •   Improvements to the description and commentary.
   •   Added a draft proposal for a hypervisor extension.
   •   Specified which interrupt sources are reserved for standard use.
   •   Allocated some synchronous exception causes for custom use.
   •   Specified the priority ordering of synchronous exceptions.
   •   Added specification that xRET instructions may, but are not required to, clear LR reserva-
       tions if A extension present.
   •   The virtual-memory system no longer permits supervisor mode to execute instructions from
       user pages, regardless of the SUM setting.
   •   Clarified that ASIDs are private to a hart, and added commentary about the possibility of a
       future global-ASID extension.
   •   SFENCE.VMA semantics have been clarified.
   •   Made the mstatus.MPP field WARL, rather than WLRL.
   •   Made the unused xip fields WPRI, rather than WIRI.
   •   Made the unused misa fields WARL, rather than WIRI.
   •   Made the unused pmpaddr and pmpcfg fields WARL, rather than WIRI.
   •   Required all harts in a system to employ the same PTE-update scheme as each other.
   •   Rectified an editing error that misdescribed the mechanism by which mstatus.xIE is written
       upon an exception.
   •   Described scheme for emulating misaligned AMOs.
   •   Specified the behavior of the misa and xepc registers in systems with variable IALIGN.
   •   Specified the behavior of writing self-contradictory values to the misa register.
   •   Defined the mcountinhibit CSR, which stops performance counters from incrementing to
       reduce energy consumption.
   •   Specified semantics for PMP regions coarser than four bytes.
   •   Specified contents of CSRs across XLEN modification.
   •   Moved PLIC chapter into its own document.
```

<!-- p.151 -->

Preface to Version 1.10

This is version 1.10 of the RISC-V privileged architecture proposal. Changes from version 1.9.1
include:

- The previous version of this document was released under a Creative Commons Attribution 4.0 International License by the original authors, and this and future versions of this document will be released under the same license.
- The explicit convention on shadow CSR addresses has been removed to reclaim CSR space. Shadow CSRs can still be added as needed.
- The mvendorid register now contains the JEDEC code of the core provider as opposed to a code supplied by the Foundation. This avoids redundancy and offloads work from the Foundation.
- The interrupt-enable stack discipline has been simplified.
- An optional mechanism to change the base ISA used by supervisor and user modes has been added to the mstatus CSR, and the field previously called Base in misa has been renamed to MXL for consistency.
- Clarified expected use of XS to summarize additional extension state status fields in mstatus.
- Optional vectored interrupt support has been added to the mtvec and stvec CSRs.
- The SEIP and UEIP bits in the mip CSR have been redefined to support software injection of external interrupts.
- The mbadaddr register has been subsumed by a more general mtval register that can now capture bad instruction bits on an illegal instruction fault to speed instruction emulation.
- The machine-mode base-and-bounds translation and protection schemes have been removed from the specification as part of moving the virtual memory configuration to sptbr (now satp). Some of the motivation for the base and bound schemes are now covered by the PMP registers, but space remains available in mstatus to add these back at a later date if deemed useful.
- In systems with only M-mode, or with both M-mode and U-mode but without U-mode trap support, the medeleg and mideleg registers now do not exist, whereas previously they returned zero.
- Virtual-memory page faults now have mcause values distinct from physical-memory access faults. Page-fault exceptions can now be delegated to S-mode without delegating exceptions generated by PMA and PMP checks.
- An optional physical-memory protection (PMP) scheme has been proposed.
- The supervisor virtual memory configuration has been moved from the mstatus register to the sptbr register. Accordingly, the sptbr register has been renamed to satp (Supervisor Address Translation and Protection) to reflect its broadened role.
- The SFENCE.VM instruction has been removed in favor of the improved SFENCE.VMA instruction.
- The mstatus bit MXR has been exposed to S-mode via sstatus.
- The polarity of the PUM bit in sstatus has been inverted to shorten code sequences involving MXR. The bit has been renamed to SUM.
- Hardware management of page-table entry Accessed and Dirty bits has been made optional; simpler implementations may trap to software to set them.

<!-- p.152 -->

- The counter-enable scheme has changed, so that S-mode can control availability of counters to U-mode.
- H-mode has been removed, as we are focusing on recursive virtualization support in S-mode. The encoding space has been reserved and may be repurposed at a later date.
- A mechanism to improve virtualization performance by trapping S-mode virtual-memory management operations has been added.
- The Supervisor Binary Interface (SBI) chapter has been removed, so that it can be maintained as a separate specification.

<!-- p.153 -->

Preface to Version 1.9.1

This is version 1.9.1 of the RISC-V privileged architecture proposal. Changes from version 1.9
include:

- Numerous additions and improvements to the commentary sections.
- Change configuration string proposal to be use a search process that supports various formats including Device Tree String and flattened Device Tree.
- Made misa optionally writable to support modifying base and supported ISA extensions. CSR address of misa changed.
- Added description of debug mode and debug CSRs.
- Added a hardware performance monitoring scheme. Simplified the handling of existing hardware counters, removing privileged versions of the counters and the corresponding delta registers.
- Fixed description of SPIE in presence of user-level interrupts.

<!-- p.160 -->

### 第 1 章 Introduction

This document describes the RISC-V privileged architecture, which covers all aspects of RISCV systems beyond the unprivileged ISA, including privileged instructions as well as additional
functionality required for running operating systems and attaching external devices.

      Commentary on our design decisions is formatted as in this paragraph, and can be skipped if the
      reader is only interested in the specification itself.

      We briefly note that the entire privileged-level design described in this document could be replaced
      with an entirely different privileged-level design without changing the unprivileged ISA, and pos-
      sibly without even changing the ABI. In particular, this privileged specification was designed to
      run existing popular operating systems, and so embodies the conventional level-based protection
      model. Alternate privileged specifications could embody other more flexible protection-domain
      models. For simplicity of expression, the text is written as if this was the only possible privileged
      architecture.

#### 1.1 RISC-V Privileged Software Stack Terminology

This section describes the terminology we use to describe components of the wide range of possible
privileged software stacks for RISC-V.

Figure 1.1 shows some of the possible software stacks that can be supported by the RISC-V architecture. The left-hand side shows a simple system that supports only a single application running
on an application execution environment (AEE). The application is coded to run with a particular
application binary interface (ABI). The ABI includes the supported user-level ISA plus a set of
ABI calls to interact with the AEE. The ABI hides details of the AEE from the application to allow greater flexibility in implementing the AEE. The same ABI could be implemented natively on
multiple different host OSs, or could be supported by a user-mode emulation environment running
on a machine with a different native ISA.

      Our graphical convention represents abstract interfaces using black boxes with white text, to
      separate them from concrete instances of components implementing the interfaces.

<!-- p.161 -->

```text
Application         Application Application         Application Application Application Application
   ABI                 ABI         ABI                 ABI         ABI         ABI         ABI
  AEE                         OS                              OS                      OS
                              SBI                             SBI                     SBI
                              SEE                                     Hypervisor
                                                                          HBI
                                                                          HEE
```

**Figure 1.1:** Different implementation stacks supporting various forms of privileged execution.

The middle configuration shows a conventional operating system (OS) that can support multiprogrammed execution of multiple applications. Each application communicates over an ABI with
the OS, which provides the AEE. Just as applications interface with an AEE via an ABI, RISC-V
operating systems interface with a supervisor execution environment (SEE) via a supervisor binary
interface (SBI). An SBI comprises the user-level and supervisor-level ISA together with a set of
SBI function calls. Using a single SBI across all SEE implementations allows a single OS binary
image to run on any SEE. The SEE can be a simple boot loader and BIOS-style IO system in a
low-end hardware platform, or a hypervisor-provided virtual machine in a high-end server, or a
thin translation layer over a host operating system in an architecture simulation environment.

       Most supervisor-level ISA definitions do not separate the SBI from the execution environment
       and/or the hardware platform, complicating virtualization and bring-up of new hardware plat-
       forms.

The rightmost configuration shows a virtual machine monitor configuration where multiple multiprogrammed OSs are supported by a single hypervisor. Each OS communicates via an SBI with
the hypervisor, which provides the SEE. The hypervisor communicates with the hypervisor execution environment (HEE) using a hypervisor binary interface (HBI), to isolate the hypervisor from
details of the hardware platform.

       The ABI, SBI, and HBI are still a work-in-progress, but we are now prioritizing support for
       Type-2 hypervisors where the SBI is provided recursively by an S-mode OS.

Hardware implementations of the RISC-V ISA will generally require additional features beyond the
privileged ISA to support the various execution environments (AEE, SEE, or HEE).

#### 1.2 Privilege Levels

At any time, a RISC-V hardware thread (hart) is running at some privilege level encoded as a mode
in one or more CSRs (control and status registers). Three RISC-V privilege levels are currently
defined as shown in Table 1.1.

Privilege levels are used to provide protection between different components of the software stack,
and attempts to perform operations not permitted by the current privilege mode will cause an

<!-- p.162 -->

```text
                        Level    Encoding          Name             Abbreviation
                          0         00        User/Application           U
                          1         01           Supervisor              S
                          2         10            Reserved
                          3         11            Machine                 M
```

**Table 1.1:** RISC-V privilege levels.

exception to be raised. These exceptions will normally cause traps into an underlying execution
environment.

      In the description, we try to separate the privilege level for which code is written, from the
      privilege mode in which it runs, although the two are often tied. For example, a supervisor-
      level operating system can run in supervisor-mode on a system with three privilege modes, but
      can also run in user-mode under a classic virtual machine monitor on systems with two or
      more privilege modes. In both cases, the same supervisor-level operating system binary code can
      be used, coded to a supervisor-level SBI and hence expecting to be able to use supervisor-level
      privileged instructions and CSRs. When running a guest OS in user mode, all supervisor-level
      actions will be trapped and emulated by the SEE running in the higher-privilege level.

The machine level has the highest privileges and is the only mandatory privilege level for a RISC-V
hardware platform. Code run in machine-mode (M-mode) is usually inherently trusted, as it has
low-level access to the machine implementation. M-mode can be used to manage secure execution
environments on RISC-V. User-mode (U-mode) and supervisor-mode (S-mode) are intended for
conventional application and operating system usage respectively.

Each privilege level has a core set of privileged ISA extensions with optional extensions and variants.
For example, machine-mode supports an optional standard extension for memory protection. Also,
supervisor mode can be extended to support Type-2 hypervisor execution as described in Chapter 7.

Implementations might provide anywhere from 1 to 3 privilege modes trading off reduced isolation
for lower implementation cost, as shown in Table 1.2.

```text
       Number of levels     Supported Modes       Intended Usage
             1              M                     Simple embedded systems
             2              M, U                  Secure embedded systems
             3              M, S, U               Systems running Unix-like operating systems
```

**Table 1.2:** Supported combinations of privilege modes.

All hardware implementations must provide M-mode, as this is the only mode that has unfettered
access to the whole machine. The simplest RISC-V implementations may provide only M-mode,
though this will provide no protection against incorrect or malicious application code.

      The lock feature of the optional PMP facility can provide some limited protection even with only
      M-mode implemented.

<!-- p.163 -->

Many RISC-V implementations will also support at least user mode (U-mode) to protect the rest
of the system from application code. Supervisor mode (S-mode) can be added to provide isolation
between a supervisor-level operating system and the SEE.

A hart normally runs application code in U-mode until some trap (e.g., a supervisor call or a timer
interrupt) forces a switch to a trap handler, which usually runs in a more privileged mode. The hart
will then execute the trap handler, which will eventually resume execution at or after the original
trapped instruction in U-mode. Traps that increase privilege level are termed vertical traps, while
traps that remain at the same privilege level are termed horizontal traps. The RISC-V privileged
architecture provides flexible routing of traps to different privilege layers.

      Horizontal traps can be implemented as vertical traps that return control to a horizontal trap
      handler in the less-privileged mode.

#### 1.3 Debug Mode

Implementations may also include a debug mode to support off-chip debugging and/or manufacturing test. Debug mode (D-mode) can be considered an additional privilege mode, with even more
access than M-mode. The separate debug specification proposal describes operation of a RISC-V
hart in debug mode. Debug mode reserves a few CSR addresses that are only accessible in D-mode,
and may also reserve some portions of the physical address space on a platform.

<!-- p.164 -->

### 第 2 章 Control and Status Registers (CSRs)

The SYSTEM major opcode is used to encode all privileged instructions in the RISC-V ISA. These
can be divided into two main classes: those that atomically read-modify-write control and status
registers (CSRs), which are defined in the Zicsr extension, and all other privileged instructions.
The privileged architecture requires the Zicsr extension; which other privileged instructions are
required depends on the privileged-architecture feature set.

In addition to the user-level state described in Volume I of this manual, an implementation may
contain additional CSRs, accessible by some subset of the privilege levels using the CSR instructions
described in the user-level manual. In this chapter, we map out the CSR address space. The
following chapters describe the function of each of the CSRs according to privilege level, as well as
the other privileged instructions which are generally closely associated with a particular privilege
level. Note that although CSRs and instructions are associated with one privilege level, they are
also accessible at all higher privilege levels.

#### 2.1 CSR Address Mapping Conventions

The standard RISC-V ISA sets aside a 12-bit encoding space (csr[11:0]) for up to 4,096 CSRs.
By convention, the upper 4 bits of the CSR address (csr[11:8]) are used to encode the read and
write accessibility of the CSRs according to privilege level as shown in Table 2.1. The top two bits
(csr[11:10]) indicate whether the register is read/write (00, 01, or 10) or read-only (11). The next
two bits (csr[9:8]) encode the lowest privilege level that can access the CSR.

      The CSR address convention uses the upper bits of the CSR address to encode default access
      privileges. This simplifies error checking in the hardware and provides a larger CSR space, but
      does constrain the mapping of CSRs into the address space.
          Implementations might allow a more-privileged level to trap otherwise permitted CSR ac-
      cesses by a less-privileged level to allow these accesses to be intercepted. This change should be
      transparent to the less-privileged software.

Attempts to access a non-existent CSR raise an illegal instruction exception. Attempts to access a
CSR without appropriate privilege level or to write a read-only register also raise illegal instruction exceptions. A read/write register might also contain some bits that are read-only, in which case
writes to the read-only bits are ignored.

<!-- p.165 -->

Table 2.1 also indicates the convention to allocate CSR addresses between standard and custom
uses. The CSR addresses designated for custom uses will not be redefined by future standard
extensions.

Machine-mode standard read-write CSRs 0x7A0–0x7BF are reserved for use by the debug system.
Of these CSRs, 0x7A0–0x7AF are accessible to machine mode, whereas 0x7B0–0x7BF are only visible
to debug mode. Implementations should raise illegal instruction exceptions on machine-mode access
to the latter set of registers.

      Effective virtualization requires that as many instructions run natively as possible inside a virtu-
      alized environment, while any privileged accesses trap to the virtual machine monitor [1]. CSRs
      that are read-only at some lower privilege level are shadowed into separate CSR addresses if they
      are made read-write at a higher privilege level. This avoids trapping permitted lower-privilege ac-
      cesses while still causing traps on illegal accesses. Currently, the counters are the only shadowed
      CSRs.

#### 2.2 CSR Listing

Tables 2.2–2.6 list the CSRs that have currently been allocated CSR addresses. The timers, counters, and floating-point CSRs are standard user-level CSRs, as well as the additional user trap
registers added by the N extension. The other registers are used by privileged code, as described
in the following chapters. Note that not all registers are required on all implementations.

<!-- p.166 -->

```text
                CSR Address           Hex                Use and Accessibility
            [11:10] [9:8] [7:4]
                                        User CSRs
              00     00    XXXX   0x000-0x0FF Standard read/write
              01     00    XXXX   0x400-0x4FF Standard read/write
              10     00    XXXX   0x800-0x8FF Custom read/write
              11     00    0XXX   0xC00-0xC7F Standard read-only
              11     00    10XX   0xC80-0xCBF Standard read-only
              11     00    11XX   0xCC0-0xCFF Custom read-only
                                      Supervisor CSRs
              00     01    XXXX   0x100-0x1FF Standard read/write
              01     01    0XXX   0x500-0x57F Standard read/write
              01     01    10XX   0x580-0x5BF Standard read/write
              01     01    11XX   0x5C0-0x5FF Custom read/write
              10     01    0XXX   0x900-0x97F Standard read/write
              10     01    10XX   0x980-0x9BF Standard read/write
              10     01    11XX   0x9C0-0x9FF Custom read/write
              11     01    0XXX   0xD00-0xD7F Standard read-only
              11     01    10XX   0xD80-0xDBF Standard read-only
              11     01    11XX   0xDC0-0xDFF Custom read-only
                                      Hypervisor CSRs
              00     10    XXXX   0x200-0x2FF Standard read/write
              01     10    0XXX   0x600-0x67F Standard read/write
              01     10    10XX   0x680-0x6BF Standard read/write
              01     10    11XX   0x6C0-0x6FF Custom read/write
              10     10    0XXX   0xA00-0xA7F Standard read/write
              10     10    10XX   0xA80-0xABF Standard read/write
              10     10    11XX   0xAC0-0xAFF Custom read/write
              11     10    0XXX   0xE00-0xE7F Standard read-only
              11     10    10XX   0xE80-0xEBF Standard read-only
              11     10    11XX   0xEC0-0xEFF Custom read-only
                                       Machine CSRs
              00     11    XXXX   0x300-0x3FF Standard read/write
              01     11    0XXX   0x700-0x77F Standard read/write
              01     11    100X   0x780-0x79F Standard read/write
              01     11    1010   0x7A0-0x7AF Standard read/write debug CSRs
              01     11    1011   0x7B0-0x7BF Debug-mode-only CSRs
              01     11    11XX   0x7C0-0x7FF Custom read/write
              10     11    0XXX   0xB00-0xB7F Standard read/write
              10     11    10XX   0xB80-0xBBF Standard read/write
              10     11    11XX   0xBC0-0xBFF Custom read/write
              11     11    0XXX   0xF00-0xF7F Standard read-only
              11     11    10XX   0xF80-0xFBF Standard read-only
              11     11    11XX   0xFC0-0xFFF Custom read-only
```

**Table 2.1:** Allocation of RISC-V CSR address ranges.

<!-- p.167 -->

```text
    Number   Privilege   Name          Description
                                         User Trap Setup
    0x000    URW         ustatus       User status register.
    0x004    URW         uie           User interrupt-enable register.
    0x005    URW         utvec         User trap handler base address.
                                       User Trap Handling
    0x040    URW         uscratch      Scratch register for user trap handlers.
    0x041    URW         uepc          User exception program counter.
    0x042    URW         ucause        User trap cause.
    0x043    URW         utval         User bad address or instruction.
    0x044    URW         uip           User interrupt pending.
                                     User Floating-Point CSRs
    0x001    URW         fflags        Floating-Point Accrued Exceptions.
    0x002    URW         frm           Floating-Point Dynamic Rounding Mode.
    0x003    URW         fcsr          Floating-Point Control and Status Register (frm + fflags).
                                       User Counter/Timers
    0xC00    URO         cycle         Cycle counter for RDCYCLE instruction.
    0xC01    URO         time          Timer for RDTIME instruction.
    0xC02    URO         instret       Instructions-retired counter for RDINSTRET instruction.
    0xC03    URO         hpmcounter3   Performance-monitoring counter.
    0xC04    URO         hpmcounter4   Performance-monitoring counter.
                                ..
                                 .
    0xC1F    URO         hpmcounter31    Performance-monitoring counter.
    0xC80    URO         cycleh          Upper 32 bits of cycle, RV32 only.
    0xC81    URO         timeh           Upper 32 bits of time, RV32 only.
    0xC82    URO         instreth        Upper 32 bits of instret, RV32 only.
    0xC83    URO         hpmcounter3h    Upper 32 bits of hpmcounter3, RV32 only.
    0xC84    URO         hpmcounter4h    Upper 32 bits of hpmcounter4, RV32 only.
                                ..
                                 .
    0xC9F    URO         hpmcounter31h   Upper 32 bits of hpmcounter31, RV32 only.
```

**Table 2.2:** Currently allocated RISC-V user-level CSR addresses.

<!-- p.168 -->

```text
       Number    Privilege   Name          Description
                                    Supervisor Trap Setup
       0x100     SRW         sstatus       Supervisor status register.
       0x102     SRW         sedeleg       Supervisor exception delegation register.
       0x103     SRW         sideleg       Supervisor interrupt delegation register.
       0x104     SRW         sie           Supervisor interrupt-enable register.
       0x105     SRW         stvec         Supervisor trap handler base address.
       0x106     SRW         scounteren Supervisor counter enable.
                                   Supervisor Trap Handling
       0x140     SRW         sscratch      Scratch register for supervisor trap handlers.
       0x141     SRW         sepc          Supervisor exception program counter.
       0x142     SRW         scause        Supervisor trap cause.
       0x143     SRW         stval         Supervisor bad address or instruction.
       0x144     SRW         sip           Supervisor interrupt pending.
                             Supervisor Protection and Translation
       0x180     SRW         satp          Supervisor address translation and protection.
                                    Debug/Trace Registers
       0x5A8     SRW         scontext      Supervisor-mode context register.
```

**Table 2.3:** Currently allocated RISC-V supervisor-level CSR addresses.

<!-- p.169 -->

```text
     Number   Privilege    Name            Description
                                       Hypervisor Trap Setup
     0x600    HRW          hstatus         Hypervisor status register.
     0x602    HRW          hedeleg         Hypervisor exception delegation register.
     0x603    HRW          hideleg         Hypervisor interrupt delegation register.
     0x604    HRW          hie             Hypervisor interrupt-enable register.
     0x606    HRW          hcounteren      Hypervisor counter enable.
     0x607    HRW          hgeie           Hypervisor guest external interrupt-enable register.
                                      Hypervisor Trap Handling
     0x643    HRW          htval           Hypervisor bad guest physical address.
     0x644    HRW          hip             Hypervisor interrupt pending.
     0x645    HRW          hvip            Hypervisor virtual interrupt pending.
     0x64A    HRW          htinst          Hypervisor trap instruction (transformed).
     0xE12    HRO          hgeip           Hypervisor guest external interrupt pending.
                               Hypervisor Protection and Translation
     0x680    HRW          hgatp           Hypervisor guest address translation and protection.
                                       Debug/Trace Registers
     0x6A8    HRW          hcontext        Hypervisor-mode context register.
                          Hypervisor Counter/Timer Virtualization Registers
     0x605    HRW          htimedelta      Delta for VS/VU-mode timer.
     0x615    HRW          htimedeltah Upper 32 bits of htimedelta, RV32 only.
                                     Virtual Supervisor Registers
     0x200    HRW          vsstatus        Virtual supervisor status register.
     0x204    HRW          vsie            Virtual supervisor interrupt-enable register.
     0x205    HRW          vstvec          Virtual supervisor trap handler base address.
     0x240    HRW          vsscratch       Virtual supervisor scratch register.
     0x241    HRW          vsepc           Virtual supervisor exception program counter.
     0x242    HRW          vscause         Virtual supervisor trap cause.
     0x243    HRW          vstval          Virtual supervisor bad address or instruction.
     0x244    HRW          vsip            Virtual supervisor interrupt pending.
     0x280    HRW          vsatp           Virtual supervisor address translation and protection.
```

**Table 2.4:** Currently allocated RISC-V hypervisor-level CSR addresses.

<!-- p.170 -->

```text
   Number    Privilege   Name         Description
                                 Machine Information Registers
   0xF11     MRO         mvendorid    Vendor ID.
   0xF12     MRO         marchid      Architecture ID.
   0xF13     MRO         mimpid       Implementation ID.
   0xF14     MRO         mhartid      Hardware thread ID.
                                     Machine Trap Setup
   0x300     MRW         mstatus      Machine status register.
   0x301     MRW         misa         ISA and extensions
   0x302     MRW         medeleg      Machine exception delegation register.
   0x303     MRW         mideleg      Machine interrupt delegation register.
   0x304     MRW         mie          Machine interrupt-enable register.
   0x305     MRW         mtvec        Machine trap-handler base address.
   0x306     MRW         mcounteren Machine counter enable.
   0x310     MRW         mstatush     Additional machine status register, RV32 only.
                                   Machine Trap Handling
   0x340     MRW         mscratch     Scratch register for machine trap handlers.
   0x341     MRW         mepc         Machine exception program counter.
   0x342     MRW         mcause       Machine trap cause.
   0x343     MRW         mtval        Machine bad address or instruction.
   0x344     MRW         mip          Machine interrupt pending.
   0x34A     MRW         mtinst       Machine trap instruction (transformed).
   0x34B     MRW         mtval2       Machine bad guest physical address.
                                  Machine Memory Protection
   0x3A0     MRW         pmpcfg0      Physical memory protection configuration.
   0x3A1     MRW         pmpcfg1      Physical memory protection configuration, RV32 only.
   0x3A2     MRW         pmpcfg2      Physical memory protection configuration.
   0x3A3     MRW         pmpcfg3      Physical memory protection configuration, RV32 only.
                              ..
                               .
   0x3AE     MRW         pmpcfg14     Physical   memory   protection   configuration.
   0x3AF     MRW         pmpcfg15     Physical   memory   protection   configuration, RV32 only.
   0x3B0     MRW         pmpaddr0     Physical   memory   protection   address register.
   0x3B1     MRW         pmpaddr1     Physical   memory   protection   address register.
                              ..
                               .
   0x3EF     MRW         pmpaddr63    Physical memory protection address register.
```

**Table 2.5:** Currently allocated RISC-V machine-level CSR addresses.

<!-- p.171 -->

```text
     Number   Privilege   Name              Description
                                    Machine Counter/Timers
     0xB00    MRW         mcycle            Machine cycle counter.
     0xB02    MRW         minstret          Machine instructions-retired counter.
     0xB03    MRW         mhpmcounter3      Machine performance-monitoring counter.
     0xB04    MRW         mhpmcounter4      Machine performance-monitoring counter.
                                 ..
                                  .
     0xB1F    MRW         mhpmcounter31     Machine performance-monitoring counter.
     0xB80    MRW         mcycleh           Upper 32 bits of mcycle, RV32 only.
     0xB82    MRW         minstreth         Upper 32 bits of minstret, RV32 only.
     0xB83    MRW         mhpmcounter3h     Upper 32 bits of mhpmcounter3, RV32 only.
     0xB84    MRW         mhpmcounter4h     Upper 32 bits of mhpmcounter4, RV32 only.
                                 ..
                                  .
     0xB9F    MRW         mhpmcounter31h Upper 32 bits of mhpmcounter31, RV32 only.
                                    Machine Counter Setup
     0x320    MRW         mcountinhibit    Machine counter-inhibit register.
     0x323    MRW         mhpmevent3       Machine performance-monitoring event selector.
     0x324    MRW         mhpmevent4       Machine performance-monitoring event selector.
                                 ..
                                  .
     0x33F    MRW       mhpmevent31       Machine performance-monitoring event selector.
                      Debug/Trace Registers (shared with Debug Mode)
     0x7A0    MRW       tselect           Debug/Trace trigger register select.
     0x7A1    MRW       tdata1            First Debug/Trace trigger data register.
     0x7A2    MRW       tdata2            Second Debug/Trace trigger data register.
     0x7A3    MRW       tdata3            Third Debug/Trace trigger data register.
     0x7A8    MRW       mcontext          Machine-mode context register.
                                  Debug Mode Registers
     0x7B0    DRW       dcsr              Debug control and status register.
     0x7B1    DRW       dpc               Debug PC.
     0x7B2    DRW       dscratch0         Debug scratch register 0.
     0x7B3    DRW       dscratch1         Debug scratch register 1.
```

**Table 2.6:** Currently allocated RISC-V machine-level CSR addresses.

<!-- p.172 -->

#### 2.3 CSR Field Specifications

The following definitions and abbreviations are used in specifying the behavior of fields within the
CSRs.

Reserved Writes Preserve Values, Reads Ignore Values (WPRI)

Some whole read/write fields are reserved for future use. Software should ignore the values read
from these fields, and should preserve the values held in these fields when writing values to other
fields of the same register. For forward compatibility, implementations that do not furnish these
fields must hardwire them to zero. These fields are labeled WPRI in the register descriptions.

      To simplify the software model, any backward-compatible future definition of previously reserved
      fields within a CSR must cope with the possibility that a non-atomic read/modify/write sequence
      is used to update other fields in the CSR. Alternatively, the original CSR definition must specify
      that subfields can only be updated atomically, which may require a two-instruction clear bit/set
      bit sequence in general that can be problematic if intermediate values are not legal.

Write/Read Only Legal Values (WLRL)

Some read/write CSR fields specify behavior for only a subset of possible bit encodings, with other
bit encodings reserved. Software should not write anything other than legal values to such a field,
and should not assume a read will return a legal value unless the last write was of a legal value,
or the register has not been written since another operation (e.g., reset) set the register to a legal
value. These fields are labeled WLRL in the register descriptions.

      Hardware implementations need only implement enough state bits to differentiate between the
      supported values, but must always return the complete specified bit-encoding of any supported
      value when read.

Implementations are permitted but not required to raise an illegal instruction exception if an
instruction attempts to write a non-supported value to a WLRL field. Implementations can
return arbitrary bit patterns on the read of a WLRL field when the last write was of an illegal
value, but the value returned should deterministically depend on the illegal written value and the
value of the field prior to the write.

Write Any Values, Reads Legal Values (WARL)

Some read/write CSR fields are only defined for a subset of bit encodings, but allow any value to be
written while guaranteeing to return a legal value whenever read. Assuming that writing the CSR
has no other side effects, the range of supported values can be determined by attempting to write
a desired setting then reading to see if the value was retained. These fields are labeled WARL in
the register descriptions.

<!-- p.173 -->

Implementations will not raise an exception on writes of unsupported values to a WARL field.
Implementations can return any legal value on the read of a WARL field when the last write was of
an illegal value, but the legal value returned should deterministically depend on the illegal written
value and the architectural state of the hart.

#### 2.4 CSR Width Modulation

If the width of a CSR is changed (for example, by changing MXLEN or UXLEN, as described in
Section 3.1.6.2), the values of the writable fields and bits of the new-width CSR are, unless specified
otherwise, determined from the previous-width CSR as though by this algorithm:

     1. The value of the previous-width CSR is copied to a temporary register of the same width.

     2. For the read-only bits of the previous-width CSR, the bits at the same positions in the
        temporary register are set to zeros.

     3. The width of the temporary register is changed to the new width. If the new width W is
        narrower than the previous width, the least-significant W bits of the temporary register are
        retained and the more-significant bits are discarded. If the new width is wider than the
        previous width, the temporary register is zero-extended to the wider width.

     4. Each writable field of the new-width CSR takes the value of the bits at the same positions in
        the temporary register.

Changing the width of a CSR is not a read or write of the CSR and thus does not trigger any side
effects.

<!-- p.174 -->

### 第 3 章 Machine-Level ISA, Version 1.12

This chapter describes the machine-level operations available in machine-mode (M-mode), which is
the highest privilege mode in a RISC-V system. M-mode is used for low-level access to a hardware
platform and is the first mode entered at reset. M-mode can also be used to implement features that
are too difficult or expensive to implement in hardware directly. The RISC-V machine-level ISA
contains a common core that is extended depending on which other privilege levels are supported
and other details of the hardware implementation.

#### 3.1 Machine-Level CSRs

In addition to the machine-level CSRs described in this section, M-mode code can access all CSRs
at lower privilege levels.

##### 3.1.1 Machine ISA Register misa

The misa CSR is a WARL read-write register reporting the ISA supported by the hart. This
register must be readable in any implementation, but a value of zero can be returned to indicate
the misa register has not been implemented, requiring that CPU capabilities be determined through
a separate non-standard mechanism.
        MXLEN-1 MXLEN-2 MXLEN-3 26 25                                                      0
        MXL[1:0] (WARL)   0 (WARL)                    Extensions[25:0] (WARL)
                 2        MXLEN-28                               26

**Figure 3.1:** Machine ISA register (misa).

The MXL (Machine XLEN) field encodes the native base integer ISA width as shown in Table 3.1.
The MXL field may be writable in implementations that support multiple base ISAs. The effective
XLEN in M-mode, MXLEN, is given by the setting of MXL, or has a fixed value if misa is zero.
The MXL field is always set to the widest supported ISA variant at reset.

<!-- p.175 -->

```text
                                               MXL      XLEN
                                                 1         32
                                                 2         64
                                                 3        128
```

**Table 3.1:** Encoding of MXL field in misa

The misa CSR is MXLEN bits wide. If the value read from misa is nonzero, field MXL of that value
always denotes the current MXLEN. If a write to misa causes MXLEN to change, the position of
MXL moves to the most-significant two bits of misa at the new width.

     The base width can be quickly ascertained using branches on the sign of the returned misa value,
     and possibly a shift left by one and a second branch on the sign. These checks can be written
     in assembly code without knowing the register width (XLEN) of the machine. The base width is
     given by XLEN = 2MXL+4 .
         The base width can also be found if misa is zero, by placing the immediate 4 in a register
     then shifting the register left by 31 bits at a time. If zero after one shift, then the machine is
     RV32. If zero after two shifts, then the machine is RV64, else RV128.

The Extensions field encodes the presence of the standard extensions, with a single bit per letter
of the alphabet (bit 0 encodes presence of extension “A” , bit 1 encodes presence of extension “B”,
through to bit 25 which encodes “Z”). The “I” bit will be set for RV32I, RV64I, RV128I base ISAs,
and the “E” bit will be set for RV32E. The Extensions field is a WARL field that can contain
writable bits where the implementation allows the supported ISA to be modified. At reset, the
Extensions field shall contain the maximal set of supported extensions, and I shall be selected over
E if both are available.

When a standard extension is disabled by clearing its bit in misa, the instructions and CSRs defined
or modified by the extension revert to their defined or reserved behaviors as if the extension is not
implemented.

The RV128I base ISA is not yet frozen, and while much of the remainder of this specification is
expected to apply to RV128, this version of the document focuses only on RV32 and RV64.

The “U” and “S” bits will be set if there is support for user and supervisor modes respectively.

The “X” bit will be set if there are any non-standard extensions.

     The misa CSR exposes a rudimentary catalog of CPU features to machine-mode code. More
     extensive information can be obtained in machine mode by probing other machine registers, and
     examining other ROM storage in the system as part of the boot process.
         We require that lower privilege levels execute environment calls instead of reading CPU
     registers to determine features available at each privilege level. This enables virtualization layers
     to alter the ISA observed at any level, and supports a much richer command interface without
     burdening hardware designs.

The “E” bit is read-only. Unless misa is hardwired to zero, the “E” bit always reads as the
complement of the “I” bit. An implementation that supports both RV32E and RV32I can select
RV32E by clearing the “I” bit.

<!-- p.176 -->

```text
     Bit   Character   Description
       0          A    Atomic extension
       1          B    Tentatively reserved for Bit-Manipulation extension
       2          C    Compressed extension
       3          D    Double-precision floating-point extension
       4          E    RV32E base ISA
       5          F    Single-precision floating-point extension
       6          G    Reserved
       7          H    Hypervisor extension
       8           I   RV32I/64I/128I base ISA
       9           J   Tentatively reserved for Dynamically Translated Languages extension
      10          K    Reserved
      11          L    Tentatively reserved for Decimal Floating-Point extension
      12          M    Integer Multiply/Divide extension
      13          N    User-level interrupts supported
      14          O    Reserved
      15          P    Tentatively reserved for Packed-SIMD extension
      16          Q    Quad-precision floating-point extension
      17          R    Reserved
      18           S   Supervisor mode implemented
      19          T    Tentatively reserved for Transactional Memory extension
      20          U    User mode implemented
      21          V    Tentatively reserved for Vector extension
      22          W    Reserved
      23          X    Non-standard extensions present
      24          Y    Reserved
      25           Z   Reserved
```

**Table 3.2:** Encoding of Extensions field in misa. All bits that are reserved for future use must return zero when read.

<!-- p.177 -->

If an ISA feature x depends on an ISA feature y, then attempting to enable feature x but disable
feature y results in both features being disabled. For example, setting “F”=0 and “D”=1 results
in both “F” and “D” being cleared.

An implementation may impose additional constraints on the collective setting of two or more misa
fields, in which case they function collectively as a single WARL field. An attempt to write an
unsupported combination causes those bits to be set to some supported combination.

Writing misa may increase IALIGN, e.g., by disabling the “C” extension. If an instruction that
would write misa increases IALIGN, and the subsequent instruction’s address is not IALIGN-bit
aligned, the write to misa is suppressed, leaving misa unchanged.

When software enables an extension that was previously disabled, then all state uniquely associated
with that extension is unspecified, unless otherwise specified by that extension.

##### 3.1.2 Machine Vendor ID Register mvendorid

The mvendorid CSR is a 32-bit read-only register providing the JEDEC manufacturer ID of the
provider of the core. This register must be readable in any implementation, but a value of 0 can be
returned to indicate the field is not implemented or that this is a non-commercial implementation.
        31                                                                  7   6               0
                                      Bank                                           Offset
                                       25                                              7

**Figure 3.2:** Vendor ID register (mvendorid).

JEDEC manufacturer IDs are ordinarily encoded as a sequence of one-byte continuation codes 0x7f,
terminated by a one-byte ID not equal to 0x7f, with an odd parity bit in the most-significant bit
of each byte. mvendorid encodes the number of one-byte continuation codes in the Bank field, and
encodes the final byte in the Offset field, discarding the parity bit. For example, the JEDEC manufacturer ID 0x7f 0x7f 0x7f 0x7f 0x7f 0x7f 0x7f 0x7f 0x7f 0x7f 0x7f 0x7f 0x8a (twelve
continuation codes followed by 0x8a) would be encoded in the mvendorid field as 0x60a.

     In JEDEC’s parlance, the bank number is one greater than the number of continuation codes;
     hence, the mvendorid Bank field encodes a value that is one less than the JEDEC bank number.

     Previously the vendor ID was to be a number allocated by the RISC-V Foundation, but this
     duplicates the work of JEDEC in maintaining a manufacturer ID standard. At time of writing,
     registering a manufacturer ID with JEDEC has a one-time cost of $500.

##### 3.1.3 Machine Architecture ID Register marchid

The marchid CSR is an MXLEN-bit read-only register encoding the base microarchitecture of the
hart. This register must be readable in any implementation, but a value of 0 can be returned to
indicate the field is not implemented. The combination of mvendorid and marchid should uniquely
identify the type of hart microarchitecture that is implemented.

<!-- p.178 -->

                MXLEN-1                                                                     0
                                              Architecture ID
                                                 MXLEN

**Figure 3.3:** Machine Architecture ID register (marchid).

Open-source project architecture IDs are allocated globally by the RISC-V Foundation, and have
non-zero architecture IDs with a zero most-significant-bit (MSB). Commercial architecture IDs are
allocated by each commercial vendor independently, but must have the MSB set and cannot contain
zero in the remaining MXLEN-1 bits.

     The intent is for the architecture ID to represent the microarchitecture associated with the repo
     around which development occurs rather than a particular organization. Commercial fabrica-
     tions of open-source designs should (and might be required by the license to) retain the original
     architecture ID. This will aid in reducing fragmentation and tool support costs, as well as provide
     attribution. Open-source architecture IDs should be administered by the Foundation and should
     only be allocated to released, functioning open-source projects. Commercial architecture IDs can
     be managed independently by any registered vendor but are required to have IDs disjoint from
     the open-source architecture IDs (MSB set) to prevent collisions if a vendor wishes to use both
     closed-source and open-source microarchitectures.
         The convention adopted within the following Implementation field can be used to segregate
     branches of the same architecture design, including by organization. The misa register also helps
     distinguish different variants of a design.

##### 3.1.4 Machine Implementation ID Register mimpid

The mimpid CSR provides a unique encoding of the version of the processor implementation. This
register must be readable in any implementation, but a value of 0 can be returned to indicate that
the field is not implemented. The Implementation value should reflect the design of the RISC-V
processor itself and not any surrounding system.
                MXLEN-1                                                                     0
                                              Implementation
                                                 MXLEN

**Figure 3.4:** Machine Implementation ID register (mimpid).

     The format of this field is left to the provider of the architecture source code, but will often be
     printed by standard tools as a hexadecimal string without any leading or trailing zeros, so the
     Implementation value can be left-justified (i.e., filled in from most-significant nibble down) with
     subfields aligned on nibble boundaries to ease human readability.

##### 3.1.5 Hart ID Register mhartid

The mhartid CSR is an MXLEN-bit read-only register containing the integer ID of the hardware
thread running the code. This register must be readable in any implementation. Hart IDs might
not necessarily be numbered contiguously in a multiprocessor system, but at least one hart must
have a hart ID of zero. Hart IDs must be unique within the execution environment.

<!-- p.179 -->

                     MXLEN-1                                                                                                            0
                                                                  Hart ID
                                                                  MXLEN

**Figure 3.5:** Hart ID register (mhartid).

      In certain cases, we must ensure exactly one hart runs some code (e.g., at reset), and so require
      one hart to have a known hart ID of zero.
          For efficiency, system implementers should aim to reduce the magnitude of the largest hart
      ID used in a system.

##### 3.1.6 Machine Status Registers (mstatus and mstatush)

The mstatus register is an MXLEN-bit read/write register formatted as shown in Figure 3.6 for
RV32 and Figure 3.7 for RV64. The mstatus register keeps track of and controls the hart’s current
operating state. A restricted view of mstatus appears as the sstatus register in the S-level ISA.

```text
        31    30                                                            23        22      21        20           19            18            17
        SD                                WPRI                                    TSR TW TVM MXR SUM MPRV
         1                                 8                                       1   1  1   1   1    1
      16 15        14 13     12    11      10   9        8        7         6             5        4            3              2             1            0
     XS[1:0] FS[1:0] MPP[1:0] WPRI SPP MPIE UBE SPIE WPRI MIE WPRI SIE WPRI
       2       2       2       2    1   1    1    1   1    1   1    1   1
```

**Figure 3.6:** Machine-mode status register (mstatus) for RV32.

```text
        63    62             38     37     36        35 34        33   32        31           23       22       21        20            19           18
      SD           WPRI            MBE SBE SXL[1:0] UXL[1:0]                      WPRI             TSR TW TVM MXR SUM
       1            25              1   1     2       2                            9                1   1  1   1   1
      17       16 15       14 13     12   11        10   9    8         7             6       5             4        3              2            1            0
     MPRV XS[1:0] FS[1:0] MPP[1:0] WPRI SPP MPIE UBE SPIE WPRI MIE WPRI SIE WPRI
      1     2       2       2       2    1   1    1    1   1    1   1    1   1
```

**Figure 3.7:** Machine-mode status register (mstatus) for RV64.

For RV32 only, mstatush is a 32-bit read/write register formatted as shown in Figure 3.8. Bits
30:4 of mstatush generally contain the same fields found in bits 62:36 of mstatus for RV64. Fields
SD, SXL, and UXL do not exist in mstatush.

The mstatush register is not required to be implemented if every field would be hardwired to zero.

<!-- p.180 -->

```text
     31                                                                       6    5     4     3         0
                                     WPRI                                         MBE SBE      WPRI
                                      26                                           1   1        4
```

**Figure 3.8:** Additional machine-mode status register (mstatush) for RV32.

###### 3.1.6.1 Privilege and Global Interrupt-Enable Stack in mstatus register

Global interrupt-enable bits, MIE and SIE, are provided for M-mode and S-mode respectively.
These bits are primarily used to guarantee atomicity with respect to interrupt handlers in the
current privilege mode.

     The global xIE bits are located in the low-order bits of mstatus, allowing them to be atomically
     set or cleared with a single CSR instruction.

When a hart is executing in privilege mode x, interrupts are globally enabled when x IE=1 and
globally disabled when x IE=0. Interrupts for lower-privilege modes, w<x, are always globally
disabled regardless of the setting of any global w IE bit for the lower-privilege mode. Interrupts for
higher-privilege modes, y>x, are always globally enabled regardless of the setting of the global yIE
bit for the higher-privilege mode. Higher-privilege-level code can use separate per-interrupt enable
bits to disable selected higher-privilege-mode interrupts before ceding control to a lower-privilege
mode.

     A higher-privilege mode y could disable all of its interrupts before ceding control to a lower-
     privilege mode but this would be unusual as it would leave only a synchronous trap, non-maskable
     interrupt, or reset as means to regain control of the hart.

To support nested traps, each privilege mode x that can respond to interrupts has a two-level
stack of interrupt-enable bits and privilege modes. x PIE holds the value of the interrupt-enable bit
active prior to the trap, and x PP holds the previous privilege mode. The x PP fields can only hold
privilege modes up to x, so MPP is two bits wide and SPP is one bit wide. When a trap is taken
from privilege mode y into privilege mode x, x PIE is set to the value of x IE; x IE is set to 0; and
x PP is set to y.

     For lower privilege modes, any trap (synchronous or asynchronous) is usually taken at a higher
     privilege mode with interrupts disabled upon entry. The higher-level trap handler will either
     service the trap and return using the stacked information, or, if not returning immediately to
     the interrupted context, will save the privilege stack before re-enabling interrupts, so only one
     entry per stack is required.

An MRET or SRET instruction is used to return from a trap in M-mode or S-mode respectively.
When executing an x RET instruction, supposing x PP holds the value y, x IE is set to x PIE; the
privilege mode is changed to y; x PIE is set to 1; and x PP is set to the least-privileged supported
mode (U if U-mode is implemented, else M). If x PP̸=M, x RET also sets MPRV=0.

     Setting xPP to the least-privileged supported mode on an xRET helps identify software bugs in
     the management of the two-level privilege-mode stack.

<!-- p.181 -->

x PP fields are WARL fields that can hold only privilege mode x and any implemented privilege
mode lower than x. If privilege mode x is not implemented, then x PP must be hardwired to 0.

     M-mode software can determine whether a privilege mode is implemented by writing that mode
     to MPP then reading it back.
         If the machine provides only U and M modes, then only a single hardware storage bit is
     required to represent either 00 or 11 in MPP.

###### 3.1.6.2 Base ISA Control in mstatus Register

For RV64 systems, the SXL and UXL fields are WARL fields that control the value of XLEN for
S-mode and U-mode, respectively. The encoding of these fields is the same as the MXL field of
misa, shown in Table 3.1. The effective XLEN in S-mode and U-mode are termed SXLEN and
UXLEN, respectively.

For RV32 systems, the SXL and UXL fields do not exist, and SXLEN=32 and UXLEN=32.

For RV64 systems, if S-mode is not supported, then SXL is hardwired to zero. Otherwise, it is
a WARL field that encodes the current value of SXLEN. In particular, an implementation may
make SXL be a read-only field whose value always ensures that SXLEN=MXLEN.

For RV64 systems, if U-mode is not supported, then UXL is hardwired to zero. Otherwise, it is a
WARL field that encodes the current value of UXLEN. In particular, an implementation may make
UXL be a read-only field whose value always ensures that UXLEN=MXLEN or UXLEN=SXLEN.

Whenever XLEN in any mode is set to a value less than the widest supported XLEN, all operations
must ignore source operand register bits above the configured XLEN, and must sign-extend results
to fill the entire widest supported XLEN in the destination register. Similarly, pc bits above XLEN
are ignored, and when the pc is written, it is sign-extended to fill the widest supported XLEN.

     We require that operations always fill the entire underlying hardware registers with defined values
     to avoid implementation-defined behavior.
         To reduce hardware complexity, the architecture imposes no checks that lower-privilege modes
     have XLEN settings less than or equal to the next-higher privilege mode. In practice, such
     settings would almost always be a software bug, but machine operation is well-defined even in
     this case.

If MXLEN is changed from 32 to a wider width, each of mstatus fields SXL and UXL, if not
restricted to a single value, gets the value corresponding to the widest supported width not wider
than the new MXLEN.

###### 3.1.6.3 Memory Privilege in mstatus Register

The MPRV (Modify PRiVilege) bit modifies the effective privilege mode, i.e., the privilege level
at which loads and stores execute. When MPRV=0, loads and stores behave as normal, using
the translation and protection mechanisms of the current privilege mode. When MPRV=1, load
and store memory addresses are translated and protected, and endianness is applied, as though the current privilege mode were set to MPP. Instruction address-translation and protection are
unaffected by the setting of MPRV. MPRV is hardwired to 0 if U-mode is not supported.

<!-- p.182 -->

An MRET or SRET instruction that changes the privilege mode to a mode less privileged than M
also sets MPRV=0.

The MXR (Make eXecutable Readable) bit modifies the privilege with which loads access virtual
memory. When MXR=0, only loads from pages marked readable (R=1 in Figure 4.18) will succeed.
When MXR=1, loads from pages marked either readable or executable (R=1 or X=1) will succeed.
MXR has no effect when page-based virtual memory is not in effect. MXR is hardwired to 0 if
S-mode is not supported.

     The MPRV and MXR mechanisms were conceived to improve the efficiency of M-mode routines
     that emulate missing hardware features, e.g., misaligned loads and stores. MPRV obviates the
     need to perform address translation in software. MXR allows instruction words to be loaded
     from pages marked execute-only.
         The current privilege mode and the privilege mode specified by MPP might have different
     XLEN settings. When MPRV=1, load and store memory addresses are treated as though the
     current XLEN were set to MPP’s XLEN, following the rules in Section 3.1.6.2.

The SUM (permit Supervisor User Memory access) bit modifies the privilege with which S-mode
loads and stores access virtual memory. When SUM=0, S-mode memory accesses to pages that are
accessible by U-mode (U=1 in Figure 4.18) will fault. When SUM=1, these accesses are permitted.
SUM has no effect when page-based virtual memory is not in effect. Note that, while SUM is
ordinarily ignored when not executing in S-mode, it is in effect when MPRV=1 and MPP=S. SUM
is hardwired to 0 if S-mode is not supported or if satp.MODE is hardwired to 0.

The MXR and SUM mechanisms only affect the interpretation of permissions encoded in page-table
entries. In particular, they have no impact on whether access-fault exceptions are raised due to
PMAs or PMP.

###### 3.1.6.4 Endianness Control in mstatus and mstatush Registers

The MBE, SBE, and UBE bits in mstatus and mstatush are WARL fields that control the
endianness of memory accesses other than instruction fetches. Instruction fetches are always littleendian.

MBE controls whether non-instruction-fetch memory accesses made from M-mode (assuming
mstatus.MPRV=0) are little-endian (MBE=0) or big-endian (MBE=1).

If S-mode is not supported, SBE is hardwired to 0. Otherwise, SBE controls whether explicit load
and store memory accesses made from S-mode are little-endian (SBE=0) or big-endian (SBE=1).

If U-mode is not supported, UBE is hardwired to 0. Otherwise, UBE controls whether explicit load
and store memory accesses made from U-mode are little-endian (UBE=0) or big-endian (UBE=1).

For implicit accesses to supervisor-level memory management data structures, such as page tables,
endianness is always controlled by SBE. Since changing SBE alters the implementation’s interpretation of these data structures, if any such data structures remain in use across a change to SBE, M-mode software must follow such a change to SBE by executing an SFENCE.VMA instruction
with rs1=x0 and rs2=x0.

<!-- p.183 -->

     Only in contrived scenarios will a given memory-management data structure be interpreted as
     both little-endian and big-endian. In practice, SBE will only be changed at runtime on world
     switches, in which case neither the old nor new memory-management data structure will be
     reinterpreted in a different endianness. In this case, no additional SFENCE.VMA is necessary,
     beyond what would ordinarily be required for a world switch.

If S-mode is supported, an implementation may make SBE be a read-only copy of MBE. If U-mode
is supported, an implementation may make UBE be a read-only copy of either MBE or SBE.

     An implementation supports only little-endian memory accesses if fields MBE, SBE, and UBE
     are all hardwired to 0. An implementation supports only big-endian memory accesses (aside
     from instruction fetches) if MBE is hardwired to 1 and SBE and UBE are each hardwired to 1
     when S-mode and U-mode are supported.

     Volume I defines a hart’s address space as a circular sequence of 2XLEN bytes at consecutive
     addresses. The correspondence between addresses and byte locations is fixed and not affected by
     any endianness mode. Rather, the applicable endianness mode determines the order of mapping
     between memory bytes and a multibyte quantity (halfword, word, etc.).

     Standard RISC-V ABIs are expected to be purely little-endian-only or big-endian-only, with no
     accommodation for mixing endianness. Nevertheless, endianness control has been defined so as
     to permit, for instance, an OS of one endianness to execute user-mode programs of the opposite
     endianness. Consideration has been given also to the possibility of nonstandard usages whereby
     software flips the endianness of memory accesses as needed.

     RISC-V instructions are uniformly little-endian to decouple instruction encoding from the cur-
     rent endianness settings, for the benefit of both hardware and software. Otherwise, for instance,
     a RISC-V assembler or disassembler would always need to know the intended active endianness,
     despite that the endianness mode might change dynamically during execution. In contrast, by
     giving instructions a fixed endianness, it is sometimes possible for carefully written software to
     be endianness-agnostic even in binary form, much like position-independent code.
         The choice to have instructions be only little-endian does have consequences, however, for
     RISC-V software that encodes or decodes machine instructions. In big-endian mode, such soft-
     ware must account for the fact that explicit loads and stores have endianness opposite that of
     instructions, for example by swapping byte order after loads and before stores.

###### 3.1.6.5 Virtualization Support in mstatus Register

The TVM (Trap Virtual Memory) bit is a WARL field that supports intercepting supervisor
virtual-memory management operations. When TVM=1, attempts to read or write the satp CSR
or execute the SFENCE.VMA instruction while executing in S-mode will raise an illegal instruction
exception. When TVM=0, these operations are permitted in S-mode. TVM is hard-wired to 0 when
S-mode is not supported.

<!-- p.184 -->

      The TVM mechanism improves virtualization efficiency by permitting guest operating systems to
      execute in S-mode, rather than classically virtualizing them in U-mode. This approach obviates
      the need to trap accesses to most S-mode CSRs.
           Trapping satp accesses and the SFENCE.VMA instruction provides the hooks necessary to
      lazily populate shadow page tables.

The TW (Timeout Wait) bit is a WARL field that supports intercepting the WFI instruction (see
Section 3.3.3). When TW=0, the WFI instruction may execute in lower privilege modes when
not prevented for some other reason. When TW=1, then if WFI is executed in any less-privileged
mode, and it does not complete within an implementation-specific, bounded time limit, the WFI
instruction causes an illegal instruction exception. The time limit may always be 0, in which case
WFI always causes an illegal instruction exception in less-privileged modes when TW=1. TW is
hard-wired to 0 when there are no modes less privileged than M.

      Trapping the WFI instruction can trigger a world switch to another guest OS, rather than
      wastefully idling in the current guest.

When S-mode is implemented, then executing WFI in U-mode causes an illegal instruction exception, unless it completes within an implementation-specific, bounded time limit. A future revision
of this specification might add a feature that allows S-mode to selectively permit WFI in U-mode.
Such a feature would only be active when TW=0.

The TSR (Trap SRET) bit is a WARL field that supports intercepting the supervisor exception
return instruction, SRET. When TSR=1, attempts to execute SRET while executing in S-mode
will raise an illegal instruction exception. When TSR=0, this operation is permitted in S-mode.
TSR is hard-wired to 0 when S-mode is not supported.

      Trapping SRET is necessary to emulate the hypervisor extension (see Chapter 7) on implemen-
      tations that do not provide it.

###### 3.1.6.6 Extension Context Status in mstatus Register

Supporting substantial extensions is one of the primary goals of RISC-V, and hence we define a
standard interface to allow unchanged privileged-mode code, particularly a supervisor-level OS, to
support arbitrary user-mode state extensions.

      To date, the V extension is the only standard extension that defines additional state beyond the
      floating-point CSR and data registers.

The FS[1:0] WARL field and the XS[1:0] read-only field are used to reduce the cost of context save
and restore by setting and tracking the current state of the floating-point unit and any other usermode extensions respectively. The FS field encodes the status of the floating-point unit, including
the CSR fcsr and floating-point data registers f0–f31, while the XS field encodes the status of
additional user-mode extensions and associated state. These fields can be checked by a context
switch routine to quickly determine whether a state save or restore is required. If a save or restore is
required, additional instructions and CSRs are typically required to effect and optimize the process.

<!-- p.185 -->

     The design anticipates that most context switches will not need to save/restore state in either
     or both of the floating-point unit or other extensions, so provides a fast check via the SD bit.

The FS and XS fields use the same status encoding as shown in Table 3.3, with the four possible
status values being Off, Initial, Clean, and Dirty.

```text
                        Status    FS Meaning       XS Meaning
                             0    Off              All off
                             1    Initial          None dirty or clean, some on
                             2    Clean            None dirty, some clean
                             3    Dirty            Some dirty
```

**Table 3.3:** Encoding of FS[1:0] and XS[1:0] status fields.

In systems that do not implement S-mode and do not have a floating-point unit, the FS field is
hardwired to zero.

In systems without additional user extensions requiring new state, the XS field is hardwired to
zero. Every additional extension with state provides a CSR field that encodes the equivalent of the
XS states. The XS field represents a summary of all extensions’ status as shown in Table 3.3.

     The XS field effectively reports the maximum status value across all user-extension status fields,
     though individual extensions can use a different encoding than XS.

The SD bit is a read-only bit that summarizes whether either the FS field or XS field signals the
presence of some dirty state that will require saving extended user context to memory. If both XS
and FS are hardwired to zero, then SD is also always zero.

When an extension’s status is set to Off, any instruction that attempts to read or write the corresponding state will cause an illegal instruction exception. When the status is Initial, the corresponding state should have an initial constant value. When the status is Clean, the corresponding
state is potentially different from the initial value, but matches the last value stored on a context
swap. When the status is Dirty, the corresponding state has potentially been modified since the
last context save.

During a context save, the responsible privileged code need only write out the corresponding state
if its status is Dirty, and can then reset the extension’s status to Clean. During a context restore,
the context need only be loaded from memory if the status is Clean (it should never be Dirty at
restore). If the status is Initial, the context must be set to an initial constant value on context
restore to avoid a security hole, but this can be done without accessing memory. For example, the
floating-point registers can all be initialized to the immediate value 0.

The FS and XS fields are read by the privileged code before saving the context. The FS field is
set directly by privileged code when resuming a user context, while the XS field is set indirectly by
writing to the status register of the individual extensions. The status fields will also be updated
during execution of instructions, regardless of privilege mode.

<!-- p.186 -->

Extensions to the user-mode ISA often include additional user-mode state, and this state can be
considerably larger than the base integer registers. The extensions might only be used for some
applications, or might only be needed for short phases within a single application. To improve
performance, the user-mode extension can define additional instructions to allow user-mode software
to return the unit to an initial state or even to turn off the unit.

For example, a coprocessor might require to be configured before use and can be “unconfigured”
after use. The unconfigured state would be represented as the Initial state for context save. If the
same application remains running between the unconfigure and the next configure (which would
set status to Dirty), there is no need to actually reinitialize the state at the unconfigure instruction,
as all state is local to the user process, i.e., the Initial state may only cause the coprocessor state
to be initialized to a constant value at context restore, not at every unconfigure.

Executing a user-mode instruction to disable a unit and place it into the Off state will cause an
illegal instruction exception to be raised if any subsequent instruction tries to use the unit before
it is turned back on. A user-mode instruction to turn a unit on must also ensure the unit’s state is
properly initialized, as the unit might have been used by another context meantime.

Changing the setting of FS has no effect on the contents of the floating-point register state. In
particular, setting FS=Off does not destroy the state, nor does setting FS=Initial clear the contents.
Other extensions might not preserve state when set to Off.

Implementations may choose to track the dirtiness of the floating-point register state imprecisely
by reporting the state to be dirty even when it has not been modified. On some implementations,
some instructions that do not mutate the floating-point state may cause the state to transition
from Initial or Clean to Dirty. On other implementations, dirtiness might not be tracked at all,
in which case the valid FS states are Off and Dirty, and an attempt to set FS to Initial or Clean
causes it to be set to Dirty.

      This definition of FS does not disallow setting FS to Dirty as a result of errant speculation. Some
      platforms may choose to disallow speculatively writing FS to close a potential side channel.

If an instruction explicitly or implicitly writes a floating-point register or the fcsr but does not
alter its contents, and FS=Initial or FS=Clean, it is implementation-defined whether FS transitions
to Dirty.

Table 3.4 shows all the possible state transitions for the FS or XS status bits. Note that the standard
floating-point extensions do not support user-mode unconfigure or disable/enable instructions.

Standard privileged instructions to initialize, save, and restore extension state are provided to
insulate privileged code from details of the added extension state by treating the state as an
opaque object.

      Many coprocessor extensions are only used in limited contexts that allows software to safely
      unconfigure or even disable units when done. This reduces the context-switch overhead of large
      stateful coprocessors.
          We separate out floating-point state from other extension state, as when a floating-point
      unit is present the floating-point registers are part of the standard calling convention, and so
      user-mode software cannot know when it is safe to disable the floating-point unit.

<!-- p.187 -->

```text
            Current State     Off           Initial           Clean                   Dirty
            Action
                                At context save in privileged code
            Save state?     No           No                No                  Yes
            Next state      Off          Initial           Clean               Clean
                               At context restore in privileged code
            Restore state? No            Yes, to initial Yes, from memory N/A
            Next state      Off          Initial           Clean               N/A
                                 Execute instruction to read state
            Action?         Exception Execute              Execute             Execute
            Next state      Off          Initial           Clean               Dirty
             Execute instruction that possibly modifies state, including configuration
            Action?         Exception Execute              Execute             Execute
            Next state      Off          Dirty             Dirty               Dirty
                             Execute instruction to unconfigure unit
            Action?         Exception Execute              Execute             Execute
            Next state      Off          Initial           Initial             Initial
                                Execute instruction to disable unit
            Action?         Execute      Execute           Execute             Execute
            Next state      Off          Off               Off                 Off
                                Execute instruction to enable unit
            Action?         Execute      Execute           Execute             Execute
            Next state      Initial      Initial           Initial             Initial
```

**Table 3.4:** FS and XS state transitions.

The XS field provides a summary of all added extension state, but additional microarchitectural
bits might be maintained in the extension to further reduce context save and restore overhead.

The SD bit is read-only and is set when either the FS or XS bits encode a Dirty state (i.e.,
SD=((FS==11) OR (XS==11))). This allows privileged code to quickly determine when no additional context save is required beyond the integer register set and PC.

The floating-point unit state is always initialized, saved, and restored using standard instructions
(F, D, and/or Q), and privileged code must be aware of FLEN to determine the appropriate space
to reserve for each f register.

All privileged modes share a single copy of the FS and XS bits. In a system with more than one
privileged mode, supervisor mode would normally use the FS and XS bits directly to record the
status with respect to the supervisor-level saved context. Other more-privileged active modes must
be more conservative in saving and restoring the extension state in their corresponding version of
the context.

     In any reasonable use case, the number of context switches between user and supervisor level
     should far outweigh the number of context switches to other privilege levels. Note that coprocessors should not require their context to be saved and restored to service asynchronous interrupts,
     unless the interrupt results in a user-level context swap.

<!-- p.188 -->

##### 3.1.7 Machine Trap-Vector Base-Address Register (mtvec)

The mtvec register is an MXLEN-bit WARL read/write register that holds trap vector configuration, consisting of a vector base address (BASE) and a vector mode (MODE).

```text
        MXLEN-1                                                                   21                0
                             BASE[MXLEN-1:2] (WARL)                                 MODE (WARL)
                                    MXLEN-2                                              2
```

**Figure 3.9:** Machine trap-vector base-address register (mtvec).

The mtvec register must always be implemented, but can contain a hardwired read-only value. If
mtvec is writable, the set of values the register may hold can vary by implementation. The value in
the BASE field must always be aligned on a 4-byte boundary, and the MODE setting may impose
additional alignment constraints on the value in the BASE field.

     We allow for considerable flexibility in implementation of the trap vector base address. On the
     one hand, we do not wish to burden low-end implementations with a large number of state bits,
     but on the other hand, we wish to allow flexibility for larger systems.

```text
             Value     Name        Description
                 0     Direct      All exceptions set pc to BASE.
                 1    Vectored     Asynchronous interrupts set pc to BASE+4×cause.
               ≥2        —         Reserved
```

**Table 3.5:** Encoding of mtvec MODE field.

The encoding of the MODE field is shown in Table 3.5. When MODE=Direct, all traps into
machine mode cause the pc to be set to the address in the BASE field. When MODE=Vectored,
all synchronous exceptions into machine mode cause the pc to be set to the address in the BASE
field, whereas interrupts cause the pc to be set to the address in the BASE field plus four times the
interrupt cause number. For example, a machine-mode timer interrupt (see Table 3.6 on page 39)
causes the pc to be set to BASE+0x1c.

     When vectored interrupts are enabled, interrupt cause 0, which corresponds to user-mode soft-
     ware interrupts, are vectored to the same location as synchronous exceptions. This ambiguity
     does not arise in practice, since user-mode software interrupts are either disabled or delegated
     to user mode.

An implementation may have different alignment constraints for different modes. In particular,
MODE=Vectored may have stricter alignment constraints than MODE=Direct.

     Allowing coarser alignments in Vectored mode enables vectoring to be implemented without a
     hardware adder circuit.

<!-- p.189 -->

     Reset and NMI vector locations are given in a platform specification.

##### 3.1.8 Machine Trap Delegation Registers (medeleg and mideleg)

By default, all traps at any privilege level are handled in machine mode, though a machine-mode
handler can redirect traps back to the appropriate level with the MRET instruction (Section 3.3.2).
To increase performance, implementations can provide individual read/write bits within medeleg
and mideleg to indicate that certain exceptions and interrupts should be processed directly by a
lower privilege level. The machine exception delegation register (medeleg) and machine interrupt
delegation register (mideleg) are MXLEN-bit read/write registers.

In systems with S-mode, the medeleg and mideleg registers must exist, and setting a bit in medeleg
or mideleg will delegate the corresponding trap, when occurring in S-mode or U-mode, to the Smode trap handler. In systems without S-mode, the medeleg and mideleg registers should not
exist (unless the N extension for user-mode interrupts is implemented).

     In versions 1.9.1 and earlier , these registers existed but were hardwired to zero in M-mode only,
     or M/U without N systems. There is no reason to require they return zero in those cases, as the
     misa register indicates whether they exist.

When a trap is delegated to S-mode, the scause register is written with the trap cause; the sepc
register is written with the virtual address of the instruction that took the trap; the stval register
is written with an exception-specific datum; the SPP field of mstatus is written with the active
privilege mode at the time of the trap; the SPIE field of mstatus is written with the value of the
SIE field at the time of the trap; and the SIE field of mstatus is cleared. The mcause, mepc, and
mtval registers and the MPP and MPIE fields of mstatus are not written.

An implementation can choose to subset the delegatable traps, with the supported delegatable bits
found by writing one to every bit location, then reading back the value in medeleg or mideleg to
see which bit positions hold a one.

An implementation shall not hardwire any bits of medeleg to one, i.e., any synchronous trap that
can be delegated must support not being delegated. Similarly, an implementation shall not hardwire
to one any bits of mideleg corresponding to machine-level interrupts (but may do so for lower-level
interrupts).

     Version 1.11 and earlier prohibited hardwiring any bits of mideleg to one. Platform standards
     may always add such restrictions.

Traps never transition from a more-privileged mode to a less-privileged mode. For example, if Mmode has delegated illegal instruction exceptions to S-mode, and M-mode software later executes
an illegal instruction, the trap is taken in M-mode, rather than being delegated to S-mode. By
contrast, traps may be taken horizontally. Using the same example, if M-mode has delegated illegal
instruction exceptions to S-mode, and S-mode software later executes an illegal instruction, the trap
is taken in S-mode.

<!-- p.190 -->

Delegated interrupts result in the interrupt being masked at the delegator privilege level. For
example, if the supervisor timer interrupt (STI) is delegated to S-mode by setting mideleg[5], STIs
will not be taken when executing in M-mode. By contrast, if mideleg[5] is clear, STIs can be taken
in any mode and regardless of current mode will transfer control to M-mode.
                       MXLEN-1                                                 0
                                    Synchronous Exceptions (WARL)
                                               MXLEN

**Figure 3.10:** Machine Exception Delegation Register medeleg.

medeleg has a bit position allocated for every synchronous exception shown in Table 3.6 on page 39,
with the index of the bit position equal to the value returned in the mcause register (i.e., setting
bit 8 allows user-mode environment calls to be delegated to a lower-privilege trap handler).

                       MXLEN-1                                                 0
                                           Interrupts (WARL)
                                                 MXLEN

**Figure 3.11:** Machine Interrupt Delegation Register mideleg.

mideleg holds trap delegation bits for individual interrupts, with the layout of bits matching those
in the mip register (i.e., STIP interrupt delegation control is located in bit 5).

For exceptions that cannot occur in less privileged modes, the corresponding medeleg bits should
be hardwired to zero. In particular, medeleg[11] is hardwired to zero.

##### 3.1.9 Machine Interrupt Registers (mip and mie)

The mip register is an MXLEN-bit read/write register containing information on pending interrupts,
while mie is the corresponding MXLEN-bit read/write register containing interrupt enable bits.
Interrupt cause number i (as reported in CSR mcause, Section 3.1.15) corresponds with bit i in
both mip and mie. Bits 15:0 are allocated to standard interrupt causes only, while bits 16 and
above are designated for platform or custom use.
                       MXLEN-1                                                 0
                                           Interrupts (WARL)
                                                 MXLEN

**Figure 3.12:** Machine Interrupt-Pending Register (mip).

                       MXLEN-1                                                 0
                                           Interrupts (WARL)
                                                 MXLEN

**Figure 3.13:** Machine Interrupt-Enable Register (mie).

An interrupt i will be taken if bit i is set in both mip and mie, and if interrupts are globally enabled.
By default, M-mode interrupts are globally enabled if the hart’s current privilege mode is less than
M, or if the current privilege mode is M and the MIE bit in the mstatus register is set. If bit i in mideleg is set, however, interrupts are considered to be globally enabled if the hart’s current
privilege mode equals the delegated privilege mode and that mode’s interrupt enable bit (x IE in
mstatus for mode x ) is set, or if the current privilege mode is less than the delegated privilege
mode.

<!-- p.191 -->

Each individual bit in register mip may be writable or may be read-only. When bit i in mip is
writable, a pending interrupt i can be cleared by writing 0 to this bit. If interrupt i can become
pending but bit i in mip is read-only, the implementation must provide some other mechanism for
clearing the pending interrupt.

A bit in mie must be writable if the corresponding interrupt can ever become pending. Bits of mie
that are not writable must be hardwired to zero.

```text
The standard portions (bits 15:0) of registers mip and mie are formatted as shown in Figures 3.14
and 3.15 respectively.
               15           12    11    10    9   8     7    6    5    4     3    2    1    0
                     0           MEIP   0    SEIP 0 MTIP 0 STIP 0 MSIP 0 SSIP 0
                     4            1     1      1  1  1   1   1  1  1   1   1  1
```

**Figure 3.14:** Standard portion (bits 15:0) of mip.

```text
               15           12    11    10    9   8     7    6    5    4     3    2    1    0
                     0           MEIE   0    SEIE 0 MTIE 0 STIE 0 MSIE 0 SSIE 0
                     4            1     1      1  1  1   1   1  1  1   1   1  1
```

**Figure 3.15:** Standard portion (bits 15:0) of mie.

     The machine-level interrupt registers handle a few root interrupt sources which are assigned a
     fixed service priority for simplicity, while separate external interrupt controllers can implement
     a more complex prioritization scheme over a much larger set of interrupts that are then muxed
     into the machine-level interrupt sources.

     The non-maskable interrupt is not made visible via the mip register as its presence is implicitly
     known when executing the NMI trap handler.

Bits mip.MEIP and mie.MEIE are the interrupt-pending and interrupt-enable bits for machinelevel external interrupts. MEIP is read-only in mip, and is set and cleared by a platform-specific
interrupt controller.

Bits mip.MTIP and mie.MTIE are the interrupt-pending and interrupt-enable bits for machine
timer interrupts. MTIP is read-only in mip, and is cleared by writing to the memory-mapped
machine-mode timer compare register.

Bits mip.MSIP and mie.MSIE are the interrupt-pending and interrupt-enable bits for machinelevel software interrupts. MSIP is read-only in mip, and is written by accesses to memory-mapped
control registers, which are used by remote harts to provide machine-level interprocessor interrupts.
A hart can write its own MSIP bit using the same memory-mapped control register.

If supervisor mode is not implemented, bits SEIP, STIP, and SSIP of mip and SEIE, STIE, and
SSIE of mie are hardwired to zeros.

<!-- p.192 -->

If supervisor mode is implemented, bits mip.SEIP and mie.SEIE are the interrupt-pending and
interrupt-enable bits for supervisor-level external interrupts. SEIP is writable in mip, and may be
written by M-mode software to indicate to S-mode that an external interrupt is pending. Additionally, the platform-level interrupt controller may generate supervisor-level external interrupts.
Supervisor-level external interrupts are made pending based on the logical-OR of the softwarewritable SEIP bit and the signal from the external interrupt controller. When mip is read with a
CSR instruction, the value of the SEIP bit returned in the rd destination register is the logicalOR of the software-writable bit and the interrupt signal from the interrupt controller, but the
signal from the interrupt controller is not used to calculate the value written to SEIP. Only the
software-writable SEIP bit participates in the read-modify-write sequence of a CSRRS or CSRRC
instruction.

     The SEIP field behavior is designed to allow a higher privilege layer to mimic external interrupts
     cleanly, without losing any real external interrupts. The behavior of the CSR instructions is
     slightly modified from regular CSR accesses as a result.

If supervisor mode is implemented, bits mip.STIP and mie.STIE are the interrupt-pending and
interrupt-enable bits for supervisor-level timer interrupts. STIP is writable in mip, and may be
written by M-mode software to deliver timer interrupts to S-mode.

If supervisor mode is implemented, bits mip.SSIP and mie.SSIE are the interrupt-pending and
interrupt-enable bits for supervisor-level software interrupts. SSIP is writable in mip.

     Interprocessor interrupts at supervisor level are implemented through implementation-specific
     mechanisms, e.g., via calls to an SEE, which might ultimately result in a machine-mode write
     to the receiving hart’s MSIP bit.
         We allow a hart to directly write only its own SSIP bit, not those of other harts, as other harts
     might be virtualized and possibly descheduled by higher privilege levels. We rely on calls to the
     SEE to provide interprocessor interrupts for this reason. Machine-mode harts are not virtualized
     and can directly interrupt other harts by setting their MSIP bits, typically using uncached I/O
     writes to memory-mapped control registers depending on the platform specification.

Multiple simultaneous interrupts destined for different privilege modes are handled in decreasing
order of destined privilege mode. Multiple simultaneous interrupts destined for the same privilege
mode are handled in the following decreasing priority order: MEI, MSI, MTI, SEI, SSI, STI.
Synchronous exceptions are of lower priority than all interrupts.

     The machine-level interrupt fixed-priority ordering rules were developed with the following ra-
     tionale.
         Interrupts for higher privilege modes must be serviced before interrupts for lower privilege
     modes to support preemption.
         The platform-specific machine-level interrupt sources in bits 16 and above have platform-
     specific priority, but are typically chosen to have the highest service priority to support very fast
     local vectored interrupts.
         External interrupts are handled before internal (timer/software) interrupts as external in-
     terrupts are usually generated by devices that might require low interrupt service times.
         Software interrupts are handled before internal timer interrupts, because internal timer in-
     terrupts are usually intended for time slicing, where time precision is less important, whereas
     software interrupts are used for inter-processor messaging. Software interrupts can be avoided
     when high-precision timing is required, or high-precision timer interrupts can be routed via a different interrupt path. Software interrupts are located in the lowest four bits of mip as these
          are often written by software, and this position allows the use of a single CSR instruction with
          a five-bit immediate.
              Synchronous exceptions are given the lowest priority to minimize worst-case interrupt la-
          tency.

<!-- p.193 -->

Restricted views of the mip and mie registers appear as the sip and sie registers for supervisor
level. If an interrupt is delegated to S-mode by setting a bit in the mideleg register, it becomes
visible in the sip register and is maskable using the sie register. Otherwise, the corresponding
bits in sip and sie appear to be hardwired to zero.

##### 3.1.10 Hardware Performance Monitor

M-mode includes a basic hardware performance-monitoring facility. The mcycle CSR counts the
number of clock cycles executed by the processor core on which the hart is running. The minstret
CSR counts the number of instructions the hart has retired. The mcycle and minstret registers
have 64-bit precision on all RV32 and RV64 systems.

The counter registers have an arbitrary value after the hart is reset, and can be written with a
given value. Any CSR write takes effect after the writing instruction has otherwise completed. The
mcycle CSR may be shared between harts on the same core, in which case writes to mcycle will
be visible to those harts. The platform should provide a mechanism to indicate which harts share
an mcycle CSR.

The hardware performance monitor includes 29 additional 64-bit event counters, mhpmcounter3–
mhpmcounter31. The event selector CSRs, mhpmevent3–mhpmevent31, are MXLEN-bit WARL
registers that control which event causes the corresponding counter to increment. The meaning
of these events is defined by the platform, but event 0 is defined to mean “no event.” All counters should be implemented, but a legal implementation is to hard-wire both the counter and its
corresponding event selector to 0.
     63                                              0
                           mcycle
                          minstret
                                                             MXLEN-1                                          0
                       mhpmcounter3                                             mhpmevent3
                       mhpmcounter4                                             mhpmevent4
                              ..                                                     ..
                               .                                                      .
                       mhpmcounter30                                           mhpmevent30
                       mhpmcounter31                                           mhpmevent31
                             64                                                  MXLEN

**Figure 3.16:** Hardware performance monitor counters.

The mhpmcounters are WARL registers that support up to 64 bits of precision on RV32 and RV64.

          A future revision of this specification will define a mechanism to generate an interrupt when a
          hardware performance monitor counter overflows.

<!-- p.194 -->

On RV32 only, reads of the mcycle, minstret, and mhpmcountern CSRs return the low 32 bits, while
reads of the mcycleh, minstreth, and mhpmcounternh CSRs return bits 63–32 of the corresponding
counter.
                               31                                        0
                                                  mcycleh
                                                 minstreth
                                               mhpmcounter3h
                                               mhpmcounter4h
                                                     ..
                                                      .
                                              mhpmcounter30h
                                              mhpmcounter31h
                                                    32

**Figure 3.17:** Upper 32 bits of hardware performance monitor counters, RV32 only.

##### 3.1.11 Machine Counter-Enable Register (mcounteren)

```text
The counter-enable register mcounteren is a 32-bit register that controls the availability of the
hardware performance-monitoring counters to the next-lowest privileged mode.
     31        30      29       28                             6    5        4      3     2    1      0
  HPM31 HPM30 HPM29                            ...                 HPM5 HPM4 HPM3 IR TM CY
    1     1     1                              23                    1    1    1   1  1  1
```

**Figure 3.18:** Counter-enable register (mcounteren).

The settings in this register only control accessibility. The act of reading or writing this register
does not affect the underlying counters, which continue to increment even when not accessible.

When the CY, TM, IR, or HPMn bit in the mcounteren register is clear, attempts to read the
cycle, time, instret, or hpmcountern register while executing in S-mode or U-mode will cause
an illegal instruction exception. When one of these bits is set, access to the corresponding register
is permitted in the next implemented privilege mode (S-mode if implemented, otherwise U-mode).

     The counter-enable bits support two common use cases with minimal hardware. For systems that
     do not need high-performance timers and counters, machine-mode software can trap accesses and
     implement all features in software. For systems that need high-performance timers and counters
     but are not concerned with obfuscating the underlying hardware counters, the counters can be
     directly exposed to lower privilege modes.

The cycle, instret, and hpmcountern CSRs are read-only shadows of mcycle, minstret, and
mhpmcountern, respectively. The time CSR is a read-only shadow of the memory-mapped mtime
register. Analogously, on RV32I the cycleh, instreth and hpmcountern CSRs are read-only
shadows of mcycleh, minstreth and mhpmcounternh, respectively. On RV32I the timeh CSR is a
read-only shadow of the upper 32 bits of the memory-mapped mtime register, while time shadows
only the lower 32 bits of mtime.

     Implementations can convert reads of the time and timeh CSRs into loads to the memory-
     mapped mtime register, or emulate this functionality in M-mode software.

<!-- p.195 -->

In systems with U-mode, the mcounteren must be implemented, but all fields are WARL and may
be hardwired to zero, indicating reads to the corresponding counter will cause an illegal instruction
exception when executing in a less-privileged mode. In systems without U-mode, the mcounteren
register should not exist.

##### 3.1.12 Machine Counter-Inhibit CSR (mcountinhibit)

```text
      31       30       29     28                                6    5       4       3       2   1    0
     HPM31 HPM30 HPM29                         ...                   HPM5 HPM4 HPM3 IR 0 CY
       1     1     1                           23                      1    1    1   1 1 1
```

**Figure 3.19:** Counter-inhibit register mcountinhibit.

The counter-inhibit register mcountinhibit is a 32-bit WARL register that controls which of the
hardware performance-monitoring counters increment. The settings in this register only control
whether the counters increment; their accessibility is not affected by the setting of this register.

When the CY, IR, or HPMn bit in the mcountinhibit register is clear, the cycle, instret, or
hpmcountern register increments as usual. When the CY, IR, or HPMn bit is set, the corresponding
counter does not increment.

The mcycle CSR may be shared between harts on the same core, in which case the
mcountinhibit.CY field is also shared between those harts, and so writes to mcountinhibit.CY
will be visible to those harts.

If the mcountinhibit register is not implemented, the implementation behaves as though the
register were set to zero.

      When the cycle and instret counters are not needed, it is desirable to conditionally inhibit
      them to reduce energy consumption. Providing a single CSR to inhibit all counters also allows
      the counters to be atomically sampled.
          Because the time counter can be shared between multiple cores, it cannot be inhibited with
      the mcountinhibit mechanism.

##### 3.1.13 Machine Scratch Register (mscratch)

The mscratch register is an MXLEN-bit read/write register dedicated for use by machine mode.
Typically, it is used to hold a pointer to a machine-mode hart-local context space and swapped
with a user register upon entry to an M-mode trap handler.
                MXLEN-1                                                                   0
                                                mscratch
                                                MXLEN

**Figure 3.20:** Machine-mode scratch register.

      The MIPS ISA allocated two user registers (k0/k1) for use by the operating system. Although
      the MIPS scheme provides a fast and simple implementation, it also reduces available user registers, and does not scale to further privilege levels, or nested traps. It can also require both
     registers are cleared before returning to user level to avoid a potential security hole and to provide
     deterministic debugging behavior.
         The RISC-V user ISA was designed to support many possible privileged system environments
     and so we did not want to infect the user-level ISA with any OS-dependent features. The RISC-
     V CSR swap instructions can quickly save/restore values to the mscratch register. Unlike the
     MIPS design, the OS can rely on holding a value in the mscratch register while the user context
     is running.

<!-- p.196 -->

##### 3.1.14 Machine Exception Program Counter (mepc)

mepc is an MXLEN-bit read/write register formatted as shown in Figure 3.21. The low bit of mepc
(mepc[0]) is always zero. On implementations that support only IALIGN=32, the two low bits
(mepc[1:0]) are always zero.

If an implementation allows IALIGN to be either 16 or 32 (by changing CSR misa, for example),
then, whenever IALIGN=32, bit mepc[1] is masked on reads so that it appears to be 0. This
masking occurs also for the implicit read by the MRET instruction. Though masked, mepc[1]
remains writable when IALIGN=32.

mepc is a WARL register that must be able to hold all valid virtual addresses. It need not be
capable of holding all possible invalid addresses. Implementations may convert some invalid address
patterns into other invalid addresses prior to writing them to mepc.

     When address translation is not in effect, virtual addresses and physical addresses are equal.
     Hence, the set of addresses mepc must be able to represent includes the set of physical addresses
     that can be used as a valid pc or effective address.

When a trap is taken into M-mode, mepc is written with the virtual address of the instruction
that was interrupted or that encountered the exception. Otherwise, mepc is never written by the
implementation, though it may be explicitly written by software.
                MXLEN-1                                                                       0
                                                    mepc
                                                   MXLEN

**Figure 3.21:** Machine exception program counter register.

##### 3.1.15 Machine Cause Register (mcause)

The mcause register is an MXLEN-bit read-write register formatted as shown in Figure 3.22. When
a trap is taken into M-mode, mcause is written with a code indicating the event that caused the
trap. Otherwise, mcause is never written by the implementation, though it may be explicitly
written by software.

The Interrupt bit in the mcause register is set if the trap was caused by an interrupt. The Exception
Code field contains a code identifying the last exception or interrupt. Table 3.6 lists the possible

<!-- p.197 -->

```text
machine-level exception codes. The Exception Code is a WLRL field, so is only guaranteed to
hold supported exception codes.
                  MXLEN-1 MXLEN-2                                                         0
                  Interrupt                     Exception Code (WLRL)
                       1                              MXLEN-1
```

**Figure 3.22:** Machine Cause register mcause.

Note that load and load-reserved instructions generate load exceptions, whereas store, storeconditional, and AMO instructions generate store/AMO exceptions.

     Interrupts can be separated from other traps with a single branch on the sign of the mcause
     register value. A shift left can remove the interrupt bit and scale the exception codes to index
     into a trap vector table.

     We do not distinguish privileged instruction exceptions from illegal opcode exceptions. This sim-
     plifies the architecture and also hides details of which higher-privilege instructions are supported
     by an implementation. The privilege level servicing the trap can implement a policy on whether
     these need to be distinguished, and if so, whether a given opcode should be treated as illegal or
     privileged.

If an instruction raises multiple synchronous exceptions, the decreasing priority order of Table 3.7
indicates which exception is taken and reported in mcause. The priority of any custom synchronous
exceptions is implementation-defined.

Note that load/store/AMO address-misaligned exceptions may have either higher or lower priority
than load/store/AMO page-fault and access-fault exceptions.

     The relative priority of load/store/AMO address-misaligned and page-fault exceptions is
     implementation-defined to flexibly cater to two design points. Implementations that never sup-
     port misaligned accesses can unconditionally raise the misaligned-address exception without per-
     forming address translation or protection checks. Implementations that support misaligned ac-
     cesses only to some physical addresses must translate and check the address before determining
     whether the misaligned access may proceed, in which case raising the page-fault exception or
     access is more appropriate.

     Instruction address breakpoints have the same cause value as, but different priority than, data
     address breakpoints (a.k.a. watchpoints) and environment break exceptions (which are raised by
     the EBREAK instruction).

     Instruction address misaligned exceptions are raised by control-flow instructions with misaligned
     targets, rather than by the act of fetching an instruction. Therefore, these exceptions have lower
     priority than other instruction address exceptions.

<!-- p.198 -->

```text
                 Interrupt   Exception Code   Description
                         1                0   Reserved
                         1                1   Supervisor software interrupt
                         1                2   Reserved
                         1                3   Machine software interrupt
                         1                4   Reserved
                         1                5   Supervisor timer interrupt
                         1                6   Reserved
                         1                7   Machine timer interrupt
                         1                8   Reserved
                         1                9   Supervisor external interrupt
                         1               10   Reserved
                         1               11   Machine external interrupt
                         1            12–15   Reserved
                         1              ≥16   Designated for platform use
                         0                0   Instruction address misaligned
                         0                1   Instruction access fault
                         0                2   Illegal instruction
                         0                3   Breakpoint
                         0                4   Load address misaligned
                         0                5   Load access fault
                         0                6   Store/AMO address misaligned
                         0                7   Store/AMO access fault
                         0                8   Environment call from U-mode
                         0                9   Environment call from S-mode
                         0               10   Reserved
                         0               11   Environment call from M-mode
                         0               12   Instruction page fault
                         0               13   Load page fault
                         0               14   Reserved
                         0               15   Store/AMO page fault
                         0            16–23   Reserved
                         0            24–31   Designated for custom use
                         0            32–47   Reserved
                         0            48–63   Designated for custom use
                         0              ≥64   Reserved
```

**Table 3.6:** Machine cause register (mcause) values after trap.

<!-- p.199 -->

```text
      Priority                        Exception Code       Description
      Highest                                        3     Instruction address breakpoint
                                                    12     Instruction page fault
                                                     1     Instruction access fault
                                                     2     Illegal instruction
                                                     0     Instruction address misaligned
                                              8, 9, 11     Environment call
                                                     3     Environment break
                                                     3     Load/Store/AMO address breakpoint
      Optionally, these may have                     6     Store/AMO address misaligned
      lowest priority instead.                       4     Load address misaligned
                                                    15     Store/AMO page fault
                                                    13     Load page fault
                                                     7     Store/AMO access fault
                                                     5     Load access fault
```

**Table 3.7:** Synchronous exception priority in decreasing priority order.

##### 3.1.16 Machine Trap Value Register (mtval)

The mtval register is an MXLEN-bit read-write register formatted as shown in Figure 3.23. When a
trap is taken into M-mode, mtval is either set to zero or written with exception-specific information
to assist software in handling the trap. Otherwise, mtval is never written by the implementation,
though it may be explicitly written by software. The hardware platform will specify which exceptions must set mtval informatively and which may unconditionally set it to zero.

When a breakpoint, address-misaligned, access-fault, or page-fault exception occurs on an instruction fetch, load, or store, mtval is written with the faulting virtual address. On an illegal instruction
trap, mtval may be written with the first XLEN or ILEN bits of the faulting instruction as described below. For other traps, mtval is set to zero, but a future standard may redefine mtval’s
setting for other traps.

      The mtval register replaces the mbadaddr register in the previous specification. In addition
      to providing bad addresses, the register can now provide the bad instruction that triggered an
      illegal instruction trap (and may in future be used to return other information). Returning the
      instruction bits accelerates instruction emulation and also removes some races that might be
      present when trying to emulate illegal instructions.

      When page-based virtual memory is enabled, mtval is written with the faulting virtual address,
      even for physical-memory access-fault exceptions. This design reduces datapath cost for most
      implementations, particularly those with hardware page-table walkers.

                 MXLEN-1                                                                 0
                                                 mtval
                                                MXLEN

**Figure 3.23:** Machine Trap Value register.

<!-- p.200 -->

For misaligned loads and stores that cause access-fault or page-fault exceptions, mtval will contain
the virtual address of the portion of the access that caused the fault. For instruction access-fault or
page-fault exceptions on systems with variable-length instructions, mtval will contain the virtual
address of the portion of the instruction that caused the fault while mepc will point to the beginning
of the instruction.

The mtval register can optionally also be used to return the faulting instruction bits on an illegal
instruction exception (mepc points to the faulting instruction in memory).

If this feature is not provided, then mtval is set to zero on an illegal instruction fault.

If this feature is provided, after an illegal instruction trap, mtval will contain the shortest of:

- the actual faulting instruction
- the first ILEN bits of the faulting instruction
- the first XLEN bits of the faulting instruction

The value loaded into mtval is right-justified and all unused upper bits are cleared to zero.

      Capturing the faulting instruction in mtval reduces the overhead of instruction emulation, po-
      tentially avoiding several partial instruction loads if the instruction is misaligned, and likely data
      cache misses or slow uncached accesses when loads are used to fetch the instruction into a data
      register. There is also a problem of atomicity if another agent is manipulating the instruction
      memory, as might occur in a dynamic translation system.
          A requirement is that the entire instruction (or at least the first XLEN bits) are fetched into
      mtval before taking the trap. This should not constrain implementations, which would typically
      fetch the entire instruction before attempting to decode the instruction, and avoids complicating
      software handlers.
          A value of zero in mtval signifies either that the feature is not supported, or an illegal zero
      instruction was fetched. A load from the instruction memory pointed to by mepc can be used
      to distinguish these two cases (or alternatively, the system configuration information can be
      interrogated to install the appropriate trap handling before runtime).

If the hardware platform specifies that no exceptions set mtval to a nonzero value, then it may
be hardwired to zero. Otherwise, mtval is a WARL register that must be able to hold all valid
virtual addresses and the value 0. It need not be capable of holding all possible invalid addresses.
Implementations may convert some invalid address patterns into other invalid addresses prior to
writing them to mtval. If the feature to return the faulting instruction bits is implemented, mtval
must also be able to hold all values less than 2N , where N is the smaller of XLEN and ILEN.

#### 3.2 Machine-Level Memory-Mapped Registers

##### 3.2.1 Machine Timer Registers (mtime and mtimecmp)

Platforms provide a real-time counter, exposed as a memory-mapped machine-mode read-write
register, mtime. mtime must increment at constant frequency, and the platform must provide a
mechanism for determining the timebase of mtime. The mtime register will wrap around if the
count overflows.

<!-- p.201 -->

The mtime register has a 64-bit precision on all RV32 and RV64 systems. Platforms provide a 64bit memory-mapped machine-mode timer compare register (mtimecmp). A machine timer interrupt
becomes pending whenever mtime contains a value greater than or equal to mtimecmp, treating the
values as unsigned integers. The interrupt remains posted until mtimecmp becomes greater than
mtime (typically as a result of writing mtimecmp). The interrupt will only be taken if interrupts
are enabled and the MTIE bit is set in the mie register.
               63                                                                          0
                                                   mtime
                                                    64

**Figure 3.24:** Machine time register (memory-mapped control register).

               63                                                                          0
                                                 mtimecmp
                                                   64

**Figure 3.25:** Machine time compare register (memory-mapped control register).

     The timer facility is defined to use wall-clock time rather than a cycle counter to support modern
     processors that run with a highly variable clock frequency to save energy through dynamic voltage
     and frequency scaling.
         Accurate real-time clocks (RTCs) are relatively expensive to provide (requiring a crystal
     or MEMS oscillator) and have to run even when the rest of system is powered down, and so
     there is usually only one in a system located in a different frequency/voltage domain from the
     processors. Hence, the RTC must be shared by all the harts in a system and accesses to the RTC
     will potentially incur the penalty of a voltage-level-shifter and clock-domain crossing. It is thus
     more natural to expose mtime as a memory-mapped register than as a CSR.
         Lower privilege levels do not have their own timecmp registers. Instead, machine-mode
     software can implement any number of virtual timers on a hart by multiplexing the next timer
     interrupt into the mtimecmp register.
         Simple fixed-frequency systems can use a single clock for both cycle counting and wall-clock
     time.

Writes to mtime and mtimecmp are guaranteed to be reflected in MTIP eventually, but not necessarily immediately.

     A spurious timer interrupt might occur if an interrupt handler increments mtimecmp then im-
     mediately returns, because MTIP might not yet have fallen in the interim. All software should be
     written to assume this event is possible, but most software should assume this event is extremely
     unlikely. It is almost always more performant to incur an occasional spurious timer interrupt
     than to poll MTIP until it falls.

In RV32, memory-mapped writes to mtimecmp modify only one 32-bit part of the register. The
following code sequence sets a 64-bit mtimecmp value without spuriously generating a timer interrupt
due to the intermediate value of the comparand:

For RV64, naturally aligned 64-bit memory accesses to the mtime and mtimecmp registers are atomic.

<!-- p.202 -->

```asm
# New comparand is in a1:a0.
li t0, -1
la t1, mtimecmp
```

sw t0, 0(t1)     # No smaller than old value.
sw a1, 4(t1)     # No smaller than new value.
sw a0, 0(t1)     # New value.

**Figure 3.26:** Sample code for setting the 64-bit time comparand in RV32, assuming a little-endian memory system and that the registers live in a strongly ordered I/O region. Storing -1 to the low-order bits of mtimecmp prevents mtimecmp from temporarily becoming smaller than the lesser of the old and new values.

#### 3.3 Machine-Mode Privileged Instructions

##### 3.3.1 Environment Call and Breakpoint

```text
        31                                20 19           15 14    12 11            7 6                0
                   funct12                        rs1        funct3         rd             opcode
                      12                           5           3             5               7
                    ECALL                          0         PRIV            0            SYSTEM
                   EBREAK                          0         PRIV            0            SYSTEM
```

The ECALL instruction is used to make a request to the supporting execution environment.
When executed in U-mode, S-mode, or M-mode, it generates an environment-call-from-U-mode
exception, environment-call-from-S-mode exception, or environment-call-from-M-mode exception,
respectively, and performs no other operation.

      ECALL generates a different exception for each originating privilege mode so that environment
      call exceptions can be selectively delegated. A typical use case for Unix-like operating systems is
      to delegate to S-mode the environment-call-from-U-mode exception but not the others.

The EBREAK instruction is used by debuggers to cause control to be transferred back to a debugging environment. It generates a breakpoint exception and performs no other operation.

      As described in the “C” Standard Extension for Compressed Instructions in Volume I of this
      manual, the C.EBREAK instruction performs the same operation as the EBREAK instruction.

ECALL and EBREAK cause the receiving privilege mode’s epc register to be set to the address of
the ECALL or EBREAK instruction itself, not the address of the following instruction. As ECALL
and EBREAK cause synchronous exceptions, they are not considered to retire, and should not
increment the minstret CSR.

##### 3.3.2 Trap-Return Instructions

Instructions to return from trap are encoded under the PRIV minor opcode.

<!-- p.203 -->

```text
        31                               20 19          15 14    12 11            7 6                0
                 funct12                         rs1       funct3         rd             opcode
                    12                            5          3             5               7
                MRET/SRET                         0        PRIV            0            SYSTEM
```

To return after handling a trap, there are separate trap return instructions per privilege level,
MRET and SRET. MRET is always provided. SRET must be provided if supervisor mode is
supported, and should raise an illegal instruction exception otherwise. SRET should also raise an
illegal instruction exception when TSR=1 in mstatus, as described in Section 3.1.6.5. An x RET
instruction can be executed in privilege mode x or higher, where executing a lower-privilege x RET
instruction will pop the relevant lower-privilege interrupt enable and privilege mode stack. In
addition to manipulating the privilege stack as described in Section 3.1.6.1, x RET sets the pc to
the value stored in the x epc register.

     Previously, there was only a single ERET instruction (which was also earlier known as SRET).
     To support the addition of user-level interrupts, we needed to add a separate URET instruction
     to continue to allow classic virtualization of OS code using the ERET instruction. It then became
     more orthogonal to support a different xRET instruction per privilege level.

If the A extension is supported, the x RET instruction is allowed to clear any outstanding LR
address reservation but is not required to. Trap handlers should explicitly clear the reservation if
required (e.g., by using a dummy SC) before executing the x RET.

     If xRET instructions always cleared LR reservations, it would be impossible to single-step through
     LR/SC sequences using a debugger.

##### 3.3.3 Wait for Interrupt

The Wait for Interrupt instruction (WFI) provides a hint to the implementation that the current
hart can be stalled until an interrupt might need servicing. Execution of the WFI instruction
can also be used to inform the hardware platform that suitable interrupts should preferentially be
routed to this hart. WFI is available in all privileged modes, and optionally available to U-mode.
This instruction may raise an illegal instruction exception when TW=1 in mstatus, as described
in Section 3.1.6.5.

```text
        31                               20 19          15 14    12 11            7 6                0
                   funct12                       rs1       funct3         rd             opcode
                      12                          5          3             5               7
                     WFI                          0        PRIV            0            SYSTEM
```

If an enabled interrupt is present or later becomes present while the hart is stalled, the interrupt
exception will be taken on the following instruction, i.e., execution resumes in the trap handler and
mepc = pc + 4.

     The following instruction takes the interrupt exception and trap, so that a simple return from
     the trap handler will execute code after the WFI instruction.

<!-- p.204 -->

The purpose of the WFI instruction is to provide a hint to the implementation, and so a legal
implementation is to simply implement WFI as a NOP.

      If the implementation does not stall the hart on execution of the instruction, then the interrupt
      will be taken on some instruction in the idle loop containing the WFI, and on a simple return
      from the handler, the idle loop will resume execution.

The WFI instruction can also be executed when interrupts are disabled. The operation of WFI
must be unaffected by the global interrupt bits in mstatus (MIE and SIE) and the delegation
register mideleg (i.e., the hart must resume if a locally enabled interrupt becomes pending, even if
it has been delegated to a less-privileged mode), but should honor the individual interrupt enables
(e.g, MTIE) (i.e., implementations should avoid resuming the hart if the interrupt is pending but
not individually enabled). WFI is also required to resume execution for locally enabled interrupts
pending at any privilege level, regardless of the global interrupt enable at each privilege level.

If the event that causes the hart to resume execution does not cause an interrupt to be taken,
execution will resume at pc + 4, and software must determine what action to take, including
looping back to repeat the WFI if there was no actionable event.

      By allowing wakeup when interrupts are disabled, an alternate entry point to an interrupt handler
      can be called that does not require saving the current context, as the current context can be saved
      or discarded before the WFI is executed.
          As implementations are free to implement WFI as a NOP, software must explicitly check
      for any relevant pending but disabled interrupts in the code following an WFI, and should loop
      back to the WFI if no suitable interrupt was detected. The mip, sip, or uip registers can be
      interrogated to determine the presence of any interrupt in machine, supervisor, or user mode
      respectively.
          The operation of WFI is unaffected by the delegation register settings.
          WFI is defined so that an implementation can trap into a higher privilege mode, either im-
      mediately on encountering the WFI or after some interval to initiate a machine-mode transition
      to a lower power state, for example.

      The same “wait-for-event” template might be used for possible future extensions that wait on
      memory locations changing, or message arrival.

#### 3.4 Reset

Upon reset, a hart’s privilege mode is set to M. The mstatus fields MIE and MPRV are reset to
0. If little-endian memory accesses are supported, the mstatus/mstatush field MBE is reset to 0.
The misa register is reset to enable the maximal set of supported extensions and widest MXLEN,
as described in Section 3.1.1. The pc is set to an implementation-defined reset vector. The mcause
register is set to a value indicating the cause of the reset. Writable PMP registers’ A and L fields
are set to 0, unless the platform mandates a different reset value for some PMP registers’ A and L
fields. All other hart state is unspecified.

The mcause values after reset have implementation-specific interpretation, but the value 0 should
be returned on implementations that do not distinguish different reset conditions. Implementations that distinguish different reset conditions should only use 0 to indicate the most complete reset
(e.g., hard reset).

<!-- p.205 -->

      Some designs may have multiple causes of reset (e.g., power-on reset, external hard reset,
      brownout detected, watchdog timer elapse, sleep-mode wakeup), which machine-mode software
      and debuggers may wish to distinguish.
          mcause reset values may alias mcause values following synchronous exceptions. There should
      be no ambiguity in this overlap, since on reset the pc is typically set to a different value than
      on other traps.

#### 3.5 Non-Maskable Interrupts

Non-maskable interrupts (NMIs) are only used for hardware error conditions, and cause an immediate jump to an implementation-defined NMI vector running in M-mode regardless of the state of a
hart’s interrupt enable bits. The mepc register is written with the virtual address of the instruction
that was interrupted, and mcause is set to a value indicating the source of the NMI. The NMI can
thus overwrite state in an active machine-mode interrupt handler.

The values written to mcause on an NMI are implementation-defined. The high Interrupt bit of
mcause should be set to indicate that this was an interrupt. An Exception Code of 0 is reserved
to mean “unknown cause” and implementations that do not distinguish sources of NMIs via the
mcause register should return 0 in the Exception Code.

Unlike resets, NMIs do not reset processor state, enabling diagnosis, reporting, and possible containment of the hardware error.

#### 3.6 Physical Memory Attributes

The physical memory map for a complete system includes various address ranges, some corresponding to memory regions, some to memory-mapped control registers, and some to vacant holes in the
address space. Some memory regions might not support reads, writes, or execution; some might
not support subword or subblock accesses; some might not support atomic operations; and some
might not support cache coherence or might have different memory models. Similarly, memorymapped control registers vary in their supported access widths, support for atomic operations, and
whether read and write accesses have associated side effects. In RISC-V systems, these properties
and capabilities of each region of the machine’s physical address space are termed physical memory
attributes (PMAs). This section describes RISC-V PMA terminology and how RISC-V systems
implement and check PMAs.

PMAs are inherent properties of the underlying hardware and rarely change during system operation. Unlike physical memory protection values described in Section 3.7, PMAs do not vary by
execution context. The PMAs of some memory regions are fixed at chip design time—for example,
for an on-chip ROM. Others are fixed at board design time, depending, for example, on which
other chips are connected to off-chip buses. Off-chip buses might also support devices that could
be changed on every power cycle (cold pluggable) or dynamically while the system is running (hot pluggable). Some devices might be configurable at run time to support different uses that imply
different PMAs—for example, an on-chip scratchpad RAM might be cached privately by one core
in one end-application, or accessed as a shared non-cached memory in another end-application.

<!-- p.206 -->

Most systems will require that at least some PMAs are dynamically checked in hardware later in
the execution pipeline after the physical address is known, as some operations will not be supported
at all physical memory addresses, and some operations require knowing the current setting of a
configurable PMA attribute. While many other architectures specify some PMAs in the virtual
memory page tables and use the TLB to inform the pipeline of these properties, this approach
injects platform-specific information into a virtualized layer and can cause system errors unless
attributes are correctly initialized in each page-table entry for each physical memory region. In
addition, the available page sizes might not be optimal for specifying attributes in the physical
memory space, leading to address-space fragmentation and inefficient use of expensive TLB entries.

For RISC-V, we separate out specification and checking of PMAs into a separate hardware structure,
the PMA checker. In many cases, the attributes are known at system design time for each physical
address region, and can be hardwired into the PMA checker. Where the attributes are run-time
configurable, platform-specific memory-mapped control registers can be provided to specify these
attributes at a granularity appropriate to each region on the platform (e.g., for an on-chip SRAM
that can be flexibly divided between cacheable and uncacheable uses). PMAs are checked for any
access to physical memory, including accesses that have undergone virtual to physical memory
translation. To aid in system debugging, we strongly recommend that, where possible, RISCV processors precisely trap physical memory accesses that fail PMA checks. Precisely trapped
PMA violations manifest as instruction, load, or store access-fault exceptions, distinct from virtualmemory page-fault exceptions. Precise PMA traps might not always be possible, for example, when
probing a legacy bus architecture that uses access failures as part of the discovery mechanism. In
this case, error responses from slave devices will be reported as imprecise bus-error interrupts.

PMAs must also be readable by software to correctly access certain devices or to correctly configure
other hardware components that access memory, such as DMA engines. As PMAs are tightly tied
to a given physical platform’s organization, many details are inherently platform-specific, as is the
means by which software can learn the PMA values for a platform. Some devices, particularly
legacy buses, do not support discovery of PMAs and so will give error responses or time out if
an unsupported access is attempted. Typically, platform-specific machine-mode code will extract
PMAs and ultimately present this information to higher-level less-privileged software using some
standard representation.

Where platforms support dynamic reconfiguration of PMAs, an interface will be provided to set
the attributes by passing requests to a machine-mode driver that can correctly reconfigure the
platform. For example, switching cacheability attributes on some memory regions might involve
platform-specific operations, such as cache flushes, that are available only to machine-mode.

##### 3.6.1 Main Memory versus I/O versus Vacant Regions

The most important characterization of a given memory address range is whether it holds regular
main memory, or I/O devices, or is vacant. Regular main memory is required to have a number
of properties, specified below, whereas I/O devices can have a much broader range of attributes.

<!-- p.207 -->

Memory regions that do not fit into regular main memory, for example, device scratchpad RAMs,
are categorized as I/O regions. Vacant regions are also classified as I/O regions but with attributes
specifying that no accesses are supported.

##### 3.6.2 Supported Access Type PMAs

Access types specify which access widths, from 8-bit byte to long multi-word burst, are supported,
and also whether misaligned accesses are supported for each access width.

     Although software running on a RISC-V hart cannot directly generate bursts to memory, software
     might have to program DMA engines to access I/O devices and might therefore need to know
     which access sizes are supported.

Main memory regions always support read and write of all access widths required by the attached
devices, and can specify whether instruction fetch is supported.

     Some platforms might mandate that all of main memory support instruction fetch. Other plat-
     forms might prohibit instruction fetch from some main memory regions.

     In some cases, the design of a processor or device accessing main memory might support other
     widths, but must be able to function with the types supported by the main memory.

I/O regions can specify which combinations of read, write, or execute accesses to which data widths
are supported.

For systems with page-based virtual memory, I/O and memory regions can specify which combinations of hardware page-table reads and hardware page-table writes are supported.

     Unix-like operating systems generally require that all of cacheable main memory supports page-
     table walks.

##### 3.6.3 Atomicity PMAs

Atomicity PMAs describes which atomic instructions are supported in this address region. Support
for atomic instructions is divided into two categories: LR/SC and AMOs.

     Some platforms might mandate that all of cacheable main memory support all atomic operations
     required by the attached processors.

###### 3.6.3.1 AMO PMA

Within AMOs, there are four levels of support: AMONone, AMOSwap, AMOLogical, and
AMOArithmetic. AMONone indicates that no AMO operations are supported. AMOSwap indicates that only amoswap instructions are supported in this address range. AMOLogical indicates that swap instructions plus all the logical AMOs (amoand, amoor, amoxor) are supported.

<!-- p.208 -->

AMOArithmetic indicates that all RISC-V AMOs are supported. For each level of support, naturally aligned AMOs of a given width are supported if the underlying memory region supports reads
and writes of that width. Main memory and I/O regions may only support a subset or none of the
processor-supported atomic operations.

```text
             AMO Class           Supported Operations
             AMONone             None
             AMOSwap             amoswap
             AMOLogical          above + amoand, amoor, amoxor
             AMOArithmetic       above + amoadd, amomin, amomax, amominu, amomaxu
```

**Table 3.8:** Classes of AMOs supported by I/O regions.

     We recommend providing at least AMOLogical support for I/O regions where possible.

###### 3.6.3.2 Reservability PMA

For LR/SC, there are three levels of support indicating combinations of the reservability and eventuality properties: RsrvNone, RsrvNonEventual, and RsrvEventual. RsrvNone indicates that no
LR/SC operations are supported (the location is non-reservable). RsrvNonEventual indicates that
the operations are supported (the location is reservable), but without the eventual success guarantee described in the unprivileged ISA specification. RsrvEventual indicates that the operations are
supported and provide the eventual success guarantee.

     We recommend providing RsrvEventual support for main memory regions where possible. Most
     I/O regions will not support LR/SC accesses, as these are most conveniently built on top of a
     cache-coherence scheme, but some may support RsrvNonEventual or RsrvEventual.

     When LR/SC is used for memory locations marked RsrvNonEventual, software should provide
     alternative fall-back mechanisms used when lack of progress is detected.

###### 3.6.3.3 Alignment

Memory regions that support aligned LR/SC or aligned AMOs might also support misaligned
LR/SC or misaligned AMOs for some addresses and access widths. If, for a given address and
access width, a misaligned LR/SC or AMO generates an address-misaligned exception, then all
loads, stores, LRs/SCs, and AMOs using that address and access width must generate addressmisaligned exceptions.

     The standard “A” extension does not support misaligned AMOs or LR/SC pairs. Support for
     misaligned AMOs is provided by the standard “Zam” extension. Support for misaligned LR/SC
     sequences is not currently standardized, so LR and SC to misaligned addresses must raise an
     exception.
         Mandating that misaligned loads and stores raise address-misaligned exceptions wherever
     misaligned AMOs raise address-misaligned exceptions permits the emulation of misaligned AMOs in an M-mode trap handler. The handler guarantees atomicity by acquiring a global mutex and
     emulating the access within the critical section. Provided that the handler for misaligned loads
     and stores uses the same mutex, all accesses to a given address that use the same word size will
     be mutually atomic.

<!-- p.209 -->

Implementations may raise access-fault exceptions instead of address-misaligned exceptions for
some misaligned accesses, indicating the instruction should not be emulated by a trap handler.
If, for a given address and access width, all misaligned LRs/SCs and AMOs generate access-fault
exceptions, then regular misaligned loads and stores using the same address and access width are
not required to execute atomically.

##### 3.6.4 Memory-Ordering PMAs

Regions of the address space are classified as either main memory or I/O for the purposes of ordering
by the FENCE instruction and atomic-instruction ordering bits.

Accesses by one hart to main memory regions are observable not only by other harts but also
by other devices with the capability to initiate requests in the main memory system (e.g., DMA
engines). Coherent main memory regions always have either the RVWMO or RVTSO memory
model. Incoherent main memory regions have an implementation-defined memory model.

Accesses by one hart to an I/O region are observable not only by other harts and bus mastering
devices but also by targeted slave I/O devices, and I/O regions may be accessed with either relaxed
or strong ordering. Accesses to an I/O region with relaxed ordering are generally observed by other
harts and bus mastering devices in a manner similar to the ordering of accesses to an RVWMO
memory region, as discussed in Section A.4.2 in Volume I of this specification. By contrast, accesses
to an I/O region with strong ordering are generally observed by other harts and bus mastering
devices in program order.

Each strongly ordered I/O region specifies a numbered ordering channel, which is a mechanism by
which ordering guarantees can be provided between different I/O regions. Channel 0 is used to
indicate point-to-point strong ordering only, where only accesses by the hart to the single associated
I/O region are strongly ordered.

Channel 1 is used to provide global strong ordering across all I/O regions. Any accesses by a hart to
any I/O region associated with channel 1 can only be observed to have occurred in program order
by all other harts and I/O devices, including relative to accesses made by that hart to relaxed I/O
regions or strongly ordered I/O regions with different channel numbers. In other words, any access
to a region in channel 1 is equivalent to executing a fence io,io instruction before and after the
instruction.

Other larger channel numbers provide program ordering to accesses by that hart across any regions
with the same channel number.

Systems might support dynamic configuration of ordering properties on each memory region.

     Strong ordering can be used to improve compatibility with legacy device driver code, or to enable
     increased performance compared to insertion of explicit ordering instructions when the imple-
     mentation is known to not reorder accesses.

<!-- p.210 -->

         Local strong ordering (channel 0) is the default form of strong ordering as it is often straight-
     forward to provide if there is only a single in-order communication path between the hart and
     the I/O device.
         Generally, different strongly ordered I/O regions can share the same ordering channel without
     additional ordering hardware if they share the same interconnect path and the path does not
     reorder requests.

##### 3.6.5 Coherence and Cacheability PMAs

Coherence is a property defined for a single physical address, and indicates that writes to that
address by one agent will eventually be made visible to other agents in the system. Coherence
is not to be confused with the memory consistency model of a system, which defines what values
a memory read can return given the previous history of reads and writes to the entire memory
system. In RISC-V platforms, the use of hardware-incoherent regions is discouraged due to software
complexity, performance, and energy impacts.

The cacheability of a memory region should not affect the software view of the region except
for differences reflected in other PMAs, such as main memory versus I/O classification, memory
ordering, supported accesses and atomic operations, and coherence. For this reason, we treat
cacheability as a platform-level setting managed by machine-mode software only.

Where a platform supports configurable cacheability settings for a memory region, a platformspecific machine-mode routine will change the settings and flush caches if necessary, so the system
is only incoherent during the transition between cacheability settings. This transitory state should
not be visible to lower privilege levels.

     We categorize RISC-V caches into three types: master-private, shared, and slave-private.
     Master-private caches are attached to a single master agent, i.e., one that issues read/write
     requests to the memory system. Shared caches are located between masters and slaves and may
     be hierarchically organized. Slave-private caches do not impact coherence, as they are local to
     a single slave and do not affect other PMAs at a master, so are not considered further here.
     We use private cache to mean a master-private cache in the following section, unless explicitly
     stated otherwise.
         Coherence is straightforward to provide for a shared memory region that is not cached by any
     agent. The PMA for such a region would simply indicate it should not be cached in a private or
     shared cache.
         Coherence is also straightforward for read-only regions, which can be safely cached by multiple
     agents without requiring a cache-coherence scheme. The PMA for this region would indicate that
     it can be cached, but that writes are not supported.
         Some read-write regions might only be accessed by a single agent, in which case they can be
     cached privately by that agent without requiring a coherence scheme. The PMA for such regions
     would indicate they can be cached. The data can also be cached in a shared cache, as other
     agents should not access the region.
         If an agent can cache a read-write region that is accessible by other agents, whether caching
     or non-caching, a cache-coherence scheme is required to avoid use of stale values. In regions
     lacking hardware cache coherence (hardware-incoherent regions), cache coherence can be im-
     plemented entirely in software, but software coherence schemes are notoriously difficult to im-
     plement correctly and often have severe performance impacts due to the need for conservative
     software-directed cache-flushing. Hardware cache-coherence schemes require more complex hardware and can impact performance due to the cache-coherence probes, but are otherwise invisible
      to software.
           For each hardware cache-coherent region, the PMA would indicate that the region is coherent
      and which hardware coherence controller to use if the system has multiple coherence controllers.
      For some systems, the coherence controller might be an outer-level shared cache, which might
      itself access further outer-level cache-coherence controllers hierarchically.
           Most memory regions within a platform will be coherent to software, because they will be
      fixed as either uncached, read-only, hardware cache-coherent, or only accessed by one agent.

<!-- p.211 -->

##### 3.6.6 Idempotency PMAs

Idempotency PMAs describe whether reads and writes to an address region are idempotent. Main
memory regions are assumed to be idempotent. For I/O regions, idempotency on reads and writes
can be specified separately (e.g., reads are idempotent but writes are not). If accesses are nonidempotent, i.e., there is potentially a side effect on any read or write access, then speculative or
redundant accesses must be avoided.

For the purposes of defining the idempotency PMAs, changes in observed memory ordering created
by redundant accesses are not considered a side effect.

      While hardware should always be designed to avoid speculative or redundant accesses to memory
      regions marked as non-idempotent, it is also necessary to ensure software or compiler optimiza-
      tions do not generate spurious accesses to non-idempotent memory regions.

      Non-idempotent regions might not support misaligned accesses. Misaligned accesses to such
      regions should raise access-fault exceptions rather than address-misaligned exceptions, indicating
      that software should not emulate the misaligned access using multiple smaller accesses, which
      could cause unexpected side effects.

#### 3.7 Physical Memory Protection

To support secure processing and contain faults, it is desirable to limit the physical addresses
accessible by software running on a hart. An optional physical memory protection (PMP) unit
provides per-hart machine-mode control registers to allow physical memory access privileges (read,
write, execute) to be specified for each physical memory region. The PMP values are checked in
parallel with the PMA checks described in Section 3.6.

The granularity of PMP access control settings are platform-specific, but the standard PMP encoding supports regions as small as four bytes. Certain regions’ privileges can be hardwired—for
example, some regions might only ever be visible in machine mode but in no lower-privilege layers.

      Platforms vary widely in demands for physical memory protection, and some platforms may
      provide other PMP structures in addition to or instead of the scheme described in this section.

PMP checks are applied to all accesses whose effective privilege mode is S or U, including instruction
fetches in S and U mode, data accesses in S and U mode when the MPRV bit in the mstatus register is clear, and data accesses in any mode when the MPRV bit in mstatus is set and the MPP field
in mstatus contains S or U. PMP checks are also applied to page-table accesses for virtual-address
translation, for which the effective privilege mode is S. Optionally, PMP checks may additionally
apply to M-mode accesses, in which case the PMP registers themselves are locked, so that even
M-mode software cannot change them until the hart is reset. In effect, PMP can grant permissions
to S and U modes, which by default have none, and can revoke permissions from M-mode, which
by default has full permissions.

<!-- p.212 -->

PMP violations are always trapped precisely at the processor.

##### 3.7.1 Physical Memory Protection CSRs

PMP entries are described by an 8-bit configuration register and one MXLEN-bit address register.
Some PMP settings additionally use the address register associated with the preceding PMP entry.
Up to 64 PMP entries are supported. Implementations may implement zero, 16, or 64 PMP CSRs.
All PMP CSR fields are WARL and may be hardwired to zero. PMP CSRs are only accessible to
M-mode.

The PMP configuration registers are densely packed into CSRs to minimize context-switch time.
For RV32, sixteen CSRs, pmpcfg0–pmpcfg15, hold the configurations pmp0cfg–pmp63cfg for the 64
PMP entries, as shown in Figure 3.27. For RV64, eight even-numbered CSRs, pmpcfg0, pmpcfg2,
. . . , pmpcfg14, hold the configurations for the 64 PMP entries, as shown in Figure 3.28. For RV64,
the odd-numbered configuration registers, pmpcfg1, pmpcfg3, . . . , pmpcfg15, are illegal.

     RV64 systems use pmpcfg2, rather than pmpcfg1, to hold configurations for PMP entries 8–15.
     This design reduces the cost of supporting multiple MXLEN values, since the configurations for
     PMP entries 8–11 appear in pmpcfg2[31:0] for both RV32 and RV64.

```text
                      31          24 23         16 15             87             0
                        pmp3cfg       pmp2cfg          pmp1cfg         pmp0cfg       pmpcfg0
                          8             8                8               8
                      31          24 23         16 15             87             0
                        pmp7cfg       pmp6cfg          pmp5cfg         pmp4cfg       pmpcfg1
                          8             8                8               8
                                                  ..
                      31          24 23         16 15             87             0
                       pmp63cfg      pmp62cfg          pmp61cfg        pmp60cfg      pmpcfg15
                          8             8                 8              8
```

**Figure 3.27:** RV32 PMP configuration CSR layout.

The PMP address registers are CSRs named pmpaddr0–pmpaddr63. Each PMP address register
encodes bits 33–2 of a 34-bit physical address for RV32, as shown in Figure 3.29. For RV64, each
PMP address register encodes bits 55–2 of a 56-bit physical address, as shown in Figure 3.30. Not
all physical address bits may be implemented, and so the pmpaddr registers are WARL.

<!-- p.213 -->

```text
63         56 55            48 47           40 39            32 31            24 23         16 15          87             0
 pmp7cfg        pmp6cfg           pmp5cfg          pmp4cfg          pmp3cfg       pmp2cfg       pmp1cfg         pmp0cfg           pmpcfg0
   8              8                 8                8                8             8             8               8
63         56 55            48 47           40 39            32 31            24 23         16 15          87             0
 pmp15cfg       pmp14cfg       pmp13cfg        pmp12cfg             pmp11cfg     pmp10cfg       pmp9cfg         pmp8cfg           pmpcfg2
    8              8              8               8                    8            8             8               8
                                                               ..
63         56 55            48 47           40 39            32 31            24 23         16 15          87             0
 pmp63cfg       pmp62cfg       pmp61cfg        pmp60cfg             pmp59cfg     pmp58cfg      pmp57cfg         pmp56cfg          pmpcfg14
    8              8              8               8                    8            8             8               8
```

**Figure 3.28:** RV64 PMP configuration CSR layout.

     The Sv32 page-based virtual-memory scheme described in Section 4.3 supports 34-bit physical
     addresses for RV32, so the PMP scheme must support addresses wider than XLEN for RV32.
     The Sv39 and Sv48 page-based virtual-memory schemes described in Sections 4.4 and 4.5 support
     a 56-bit physical address space, so the RV64 PMP address registers impose the same limit.

                   31                                                                                              0
                                                          address[33:2] (WARL)
                                                                   32

```text
                                    Figure 3.29: PMP address register format, RV32.
         63        54        53                                                                                               0
            0 (WARL)                                                address[55:2] (WARL)
              10                                                              54
```

**Figure 3.30:** PMP address register format, RV64.

Figure 3.31 shows the layout of a PMP configuration register. The R, W, and X bits, when set,
indicate that the PMP entry permits read, write, and instruction execution, respectively. When one
of these bits is clear, the corresponding access type is denied. The combination R=0 and W=1 is
reserved for future use. The remaining two fields, A and L, are described in the following sections.
            7           6                      5      4                    3          2                1                      0
     L (WARL)                0 (WARL)                     A (WARL)                X (WARL)          W (WARL)           R (WARL)
         1                       2                            2                       1                 1                  1

**Figure 3.31:** PMP configuration register format.

Attempting to fetch an instruction from a PMP region that does not have execute permissions raises
an instruction access-fault exception. Attempting to execute a load or load-reserved instruction
which accesses a physical address within a PMP region without read permissions raises a load
access-fault exception. Attempting to execute a store, store-conditional, or AMO instruction which
accesses a physical address within a PMP region without write permissions raises a store access-fault
exception.

If MXLEN is changed, the contents of the pmpxcfg fields are preserved, but appear in the pmpcfgy
CSR prescribed by the new setting of MXLEN. For example, when MXLEN is changed from 64 to 32, pmp4cfg moves from pmpcfg0[39:32] to pmpcfg1[7:0]. The pmpaddr CSRs follow the usual CSR
width modulation rules described in Section 2.4.

<!-- p.214 -->

Address Matching

The A field in a PMP entry’s configuration register encodes the address-matching mode of the
associated PMP address register. The encoding of this field is shown in Table 3.9. When A=0,
this PMP entry is disabled and matches no addresses. Two other address-matching modes are
supported: naturally aligned power-of-2 regions (NAPOT), including the special case of naturally
aligned four-byte regions (NA4); and the top boundary of an arbitrary range (TOR). These modes
support four-byte granularity.

```text
                 A    Name      Description
                 0    OFF       Null region (disabled)
                 1    TOR       Top of range
                 2    NA4       Naturally aligned four-byte region
                 3   NAPOT      Naturally aligned power-of-two region, ≥8 bytes
```

**Table 3.9:** Encoding of A field in PMP configuration registers.

NAPOT ranges make use of the low-order bits of the associated address register to encode the size
of the range, as shown in Table 3.10.

```text
                       pmpaddr      pmpcfg.A    Match type and size
                     yyyy...yyyy      NA4       4-byte NAPOT range
                     yyyy...yyy0    NAPOT       8-byte NAPOT range
                     yyyy...yy01    NAPOT       16-byte NAPOT range
                     yyyy...y011    NAPOT       32-byte NAPOT range
                         ...           ...                   ...
                     yy01...1111    NAPOT       2XLEN -byte NAPOT range
                     y011...1111    NAPOT       2XLEN+1 -byte NAPOT range
                     0111...1111    NAPOT       2XLEN+2 -byte NAPOT range
                     1111...1111    NAPOT       2XLEN+3 -byte NAPOT range
```

**Table 3.10:** NAPOT range encoding in PMP address and configuration registers.

If TOR is selected, the associated address register forms the top of the address range, and the
preceding PMP address register forms the bottom of the address range. If PMP entry i’s A field is
set to TOR, the entry matches any address y such that pmpaddri−1 ≤ y < pmpaddri (irrespective of
the value of pmpcfgi−1 ). If PMP entry 0’s A field is set to TOR, zero is used for the lower bound,
and so it matches any address y < pmpaddr0 .

     If pmpaddri−1 ≥ pmpaddri and pmpcfgi .A=TOR, then PMP entry i matches no addresses.

Although the PMP mechanism supports regions as small as four bytes, platforms may specify
coarser PMP regions. In general, the PMP grain is 2G+2 bytes and must be the same across all PMP regions. When G ≥ 1, the NA4 mode is not selectable. When G ≥ 2 and pmpcfgi .A[1] is set,
i.e. the mode is NAPOT, then bits pmpaddri [G-2:0] read as all ones. When G ≥ 1 and pmpcfgi .A[1]
is clear, i.e. the mode is OFF or TOR, then bits pmpaddri [G-1:0] read as all zeros. Bits pmpaddri [G1:0] do not affect the TOR address-matching logic. Although changing pmpcfgi .A[1] affects the value
read from pmpaddri , it does not affect the underlying value stored in that register—in particular,
pmpaddri [G-1] retains its original value when pmpcfgi .A is changed from NAPOT to TOR/OFF
then back to NAPOT.

<!-- p.215 -->

      Software may determine the PMP granularity by writing zero to pmp0cfg, then writing all ones
      to pmpaddr0, then reading back pmpaddr0. If G is the index of the least-significant bit set, the
      PMP granularity is 2G+2 bytes.

If the current XLEN is greater than MXLEN, the PMP address registers are zero-extended from
MXLEN to XLEN bits for the purposes of address matching.

Locking and Privilege Mode

The L bit indicates that the PMP entry is locked, i.e., writes to the configuration register and
associated address registers are ignored. Locked PMP entries remain locked until the hart is reset.
If PMP entry i is locked, writes to pmpicfg and pmpaddri are ignored. Additionally, if PMP entry i
is locked and pmpicfg.A is set to TOR, writes to pmpaddri-1 are ignored.

      Setting the L bit locks the PMP entry even when the A field is set to OFF.

In addition to locking the PMP entry, the L bit indicates whether the R/W/X permissions are
enforced on M-mode accesses. When the L bit is set, these permissions are enforced for all privilege
modes. When the L bit is clear, any M-mode access matching the PMP entry will succeed; the
R/W/X permissions apply only to S and U modes.

Priority and Matching Logic

PMP entries are statically prioritized. The lowest-numbered PMP entry that matches any byte of
an access determines whether that access succeeds or fails. The matching PMP entry must match
all bytes of an access, or the access fails, irrespective of the L, R, W, and X bits. For example,
if a PMP entry is configured to match the four-byte range 0xC–0xF, then an 8-byte access to the
range 0x8–0xF will fail, assuming that PMP entry is the highest-priority entry that matches those
addresses.

If a PMP entry matches all bytes of an access, then the L, R, W, and X bits determine whether the
access succeeds or fails. If the L bit is clear and the privilege mode of the access is M, the access
succeeds. Otherwise, if the L bit is set or the privilege mode of the access is S or U, then the access
succeeds only if the R, W, or X bit corresponding to the access type is set.

If no PMP entry matches an M-mode access, the access succeeds. If no PMP entry matches an
S-mode or U-mode access, but at least one PMP entry is implemented, the access fails.

<!-- p.216 -->

     If at least one PMP entry is implemented, but all PMP entries’ A fields are set to OFF, then
     all S-mode and U-mode memory accesses will fail.

Failed accesses generate an instruction, load, or store access-fault exception. Note that a single
instruction may generate multiple accesses, which may not be mutually atomic. An access-fault
exception is generated if at least one access generated by an instruction fails, though other accesses
generated by that instruction may succeed with visible side effects. Notably, instructions that
reference virtual memory are decomposed into multiple accesses.

On some implementations, misaligned loads, stores, and instruction fetches may also be decomposed into multiple accesses, some of which may succeed before an access-fault exception occurs.
In particular, a portion of a misaligned store that passes the PMP check may become visible,
even if another portion fails the PMP check. The same behavior may manifest for floating-point
stores wider than XLEN bits (e.g., the FSD instruction in RV32D), even when the store address is
naturally aligned.

##### 3.7.2 Physical Memory Protection and Paging

The Physical Memory Protection mechanism is designed to compose with the page-based virtual
memory systems described in Chapter 4. When paging is enabled, instructions that access virtual
memory may result in multiple physical-memory accesses, including implicit references to the page
tables. The PMP checks apply to all of these accesses. The effective privilege mode for implicit
page-table accesses is S.

Implementations with virtual memory are permitted to perform address translations speculatively
and earlier than required by an explicit virtual-memory access. The PMP settings for the resulting
physical address may be checked at any point between the address translation and the explicit
virtual-memory access. Hence, when the PMP settings are modified in a manner that affects either
the physical memory that holds the page tables or the physical memory to which the page tables
point, M-mode software must synchronize the PMP settings with the virtual memory system. This
is accomplished by executing an SFENCE.VMA instruction with rs1=x0 and rs2=x0, after the
PMP CSRs are written.

If page-based virtual memory is not implemented, memory accesses check the PMP settings synchronously, so no fence is needed.

<!-- p.218 -->

### 第 4 章 Supervisor-Level ISA, Version 1.12

This chapter describes the RISC-V supervisor-level architecture, which contains a common core
that is used with various supervisor-level address translation and protection schemes.

      Supervisor mode is deliberately restricted in terms of interactions with underlying physical hard-
      ware, such as physical memory and device interrupts, to support clean virtualization. In this
      spirit, certain supervisor-level facilities, including requests for timer and interprocessor inter-
      rupts, are provided by implementation-specific mechanisms. In some systems, a supervisor execu-
      tion environment (SEE) provides these facilities in a manner specified by a supervisor binary in-
      terface (SBI). Other systems supply these facilities directly, through some other implementation-
      defined mechanism.

#### 4.1 Supervisor CSRs

A number of CSRs are provided for the supervisor.

      The supervisor should only view CSR state that should be visible to a supervisor-level operating
      system. In particular, there is no information about the existence (or non-existence) of higher
      privilege levels (machine level or other) visible in the CSRs accessible by the supervisor.
          Many supervisor CSRs are a subset of the equivalent machine-mode CSR, and the machine-
      mode chapter should be read first to help understand the supervisor-level CSR descriptions.

##### 4.1.1 Supervisor Status Register (sstatus)

The sstatus register is an SXLEN-bit read/write register formatted as shown in Figure 4.1 for
RV32 and Figure 4.2 for RV64. The sstatus register keeps track of the processor’s current operating
state.

The SPP bit indicates the privilege level at which a hart was executing before entering supervisor
mode. When a trap is taken, SPP is set to 0 if the trap originated from user mode, or 1 otherwise.
When an SRET instruction (see Section 3.3.2) is executed to return from the trap handler, the

<!-- p.219 -->

```text
 31   30 20     19      18       17        16 15   14 13        12   9        8        7       6     5   4       2    1   0
 SD WPRI MXR SUM WPRI XS[1:0] FS[1:0] WPRI SPP WPRI UBE SPIE WPRI SIE WPRI
  1  11   1   1   1     2       2      4    1   1    1    1   3    1   1
```

**Figure 4.1:** Supervisor-mode status register (sstatus) for RV32.

```text
      63   62                                      34      33        32   31                   20   19   18          17
      SD                     WPRI                       UXL[1:0]                  WPRI              MXR SUM WPRI
       1                      29                          2                        12                1   1   1
                16 15        14 13    12       9    8           7         6        5       4    2    1       0
                XS[1:0] FS[1:0]       WPRI         SPP WPRI UBE SPIE WPRI SIE WPRI
                  2       2            4            1   1    1    1   3    1   1
```

**Figure 4.2:** Supervisor-mode status register (sstatus) for RV64.

privilege level is set to user mode if the SPP bit is 0, or supervisor mode if the SPP bit is 1; SPP
is then set to 0.

The SIE bit enables or disables all interrupts in supervisor mode. When SIE is clear, interrupts
are not taken while in supervisor mode. When the hart is running in user-mode, the value in
SIE is ignored, and supervisor-level interrupts are enabled. The supervisor can disable individual
interrupt sources using the sie CSR.

The SPIE bit indicates whether supervisor interrupts were enabled prior to trapping into supervisor
mode. When a trap is taken into supervisor mode, SPIE is set to SIE, and SIE is set to 0. When
an SRET instruction is executed, SIE is set to SPIE, then SPIE is set to 1.

The sstatus register is a subset of the mstatus register.

      In a straightforward implementation, reading or writing any field in sstatus is equivalent to
      reading or writing the homonymous field in mstatus.

###### 4.1.1.1 Base ISA Control in sstatus Register

The UXL field controls the value of XLEN for U-mode, termed UXLEN, which may differ from the
value of XLEN for S-mode, termed SXLEN. The encoding of UXL is the same as that of the MXL
field of misa, shown in Table 3.1.

For RV32 systems, the UXL field does not exist, and UXLEN=32. For RV64 systems, it is a WARL
field that encodes the current value of UXLEN. In particular, an implementation may make UXL
be a read-only field whose value always ensures that UXLEN=SXLEN.

If UXLEN ̸= SXLEN, instructions executed in the narrower mode must ignore source register
operand bits above the configured XLEN, and must sign-extend results to fill the widest supported
XLEN in the destination register.

<!-- p.220 -->

If UXLEN < SXLEN, user-mode instruction-fetch addresses and load and store effective addresses
are taken modulo 2UXLEN . For example, when UXLEN=32 and SXLEN=64, user-mode memory
accesses reference the lowest 4 GiB of the address space.

###### 4.1.1.2 Memory Privilege in sstatus Register

The MXR (Make eXecutable Readable) bit modifies the privilege with which loads access virtual
memory. When MXR=0, only loads from pages marked readable (R=1 in Figure 4.18) will succeed.
When MXR=1, loads from pages marked either readable or executable (R=1 or X=1) will succeed.
MXR has no effect when page-based virtual memory is not in effect.

The SUM (permit Supervisor User Memory access) bit modifies the privilege with which S-mode
loads and stores access virtual memory. When SUM=0, S-mode memory accesses to pages that are
accessible by U-mode (U=1 in Figure 4.18) will fault. When SUM=1, these accesses are permitted.
SUM has no effect when page-based virtual memory is not in effect, nor when executing in U-mode.
Note that S-mode can never execute instructions from user pages, regardless of the state of SUM.

SUM is hardwired to 0 if satp.MODE is hardwired to 0.

     The SUM mechanism prevents supervisor software from inadvertently accessing user memory.
     Operating systems can execute the majority of code with SUM clear; the few code segments that
     should access user memory can temporarily set SUM.
         The SUM mechanism does not avail S-mode software of permission to execute instructions
     in user code pages. Legitimate uses cases for execution from user memory in supervisor context
     are rare in general and nonexistent in POSIX environments. However, bugs in supervisors that
     lead to arbitrary code execution are much easier to exploit if the supervisor exploit code can be
     stored in a user buffer at a virtual address chosen by an attacker.
         Some non-POSIX single address space operating systems do allow certain privileged software
     to partially execute in supervisor mode, while most programs run in user mode, all in a shared
     address space. This use case can be realized by mapping the physical code pages at multiple
     virtual addresses with different permissions, possibly with the assistance of the instruction page-
     fault handler to direct supervisor software to use the alternate mapping.

###### 4.1.1.3 Endianness Control in sstatus Register

The UBE bit is a WARL field that controls the endianness of explicit memory accesses made from
U-mode, which may differ from the endianness of memory accesses in S-mode. An implementation
may make UBE be a read-only field that always specifies the same endianness as for S-mode.

UBE controls whether explicit load and store memory accesses made from U-mode are little-endian
(UBE=0) or big-endian (UBE=1).

UBE has no effect on instruction fetches, which are implicit memory accesses that are always
little-endian.

For implicit accesses to supervisor-level memory management data structures, such as page tables,
S-mode endianness always applies and UBE is ignored.

<!-- p.221 -->

     Standard RISC-V ABIs are expected to be purely little-endian-only or big-endian-only, with no
     accommodation for mixing endianness. Nevertheless, endianness control has been defined so as
     to permit an OS of one endianness to execute user-mode programs of the opposite endianness.

##### 4.1.2 Supervisor Trap Vector Base Address Register (stvec)

The stvec register is an SXLEN-bit read/write register that holds trap vector configuration, consisting of a vector base address (BASE) and a vector mode (MODE).

```text
        SXLEN-1                                                               21            0
                             BASE[SXLEN-1:2] (WARL)                             MODE (WARL)
                                     SXLEN-2                                       2
```

**Figure 4.3:** Supervisor trap vector base address register (stvec).

The BASE field in stvec is a WARL field that can hold any valid virtual or physical address,
subject to the following alignment constraints: the address must be 4-byte aligned, and MODE
settings other than Direct might impose additional alignment constraints on the value in the BASE
field.

```text
             Value     Name       Description
                 0     Direct     All exceptions set pc to BASE.
                 1    Vectored    Asynchronous interrupts set pc to BASE+4×cause.
               ≥2        —        Reserved
```

**Table 4.1:** Encoding of stvec MODE field.

The encoding of the MODE field is shown in Table 4.1. When MODE=Direct, all traps into
supervisor mode cause the pc to be set to the address in the BASE field. When MODE=Vectored,
all synchronous exceptions into supervisor mode cause the pc to be set to the address in the BASE
field, whereas interrupts cause the pc to be set to the address in the BASE field plus four times the
interrupt cause number. For example, a supervisor-mode timer interrupt (see Table 4.2) causes the
pc to be set to BASE+0x14. Setting MODE=Vectored may impose a stricter alignment constraint
on BASE.

##### 4.1.3 Supervisor Interrupt Registers (sip and sie)

The sip register is an SXLEN-bit read/write register containing information on pending interrupts,
while sie is the corresponding SXLEN-bit read/write register containing interrupt enable bits.
Interrupt cause number i (as reported in CSR scause, Section 4.1.8) corresponds with bit i in
both sip and sie. Bits 15:0 are allocated to standard interrupt causes only, while bits 16 and
above are designated for platform or custom use.

An interrupt i will be taken if bit i is set in both sip and sie, and if supervisor-level interrupts
are globally enabled. Supervisor-level interrupts are globally enabled if the hart’s current privilege SXLEN-1                                                                  0
                                          Interrupts (WARL)
                                                SXLEN Figure 4.4: Supervisor interrupt-pending register (sip).

<!-- p.222 -->

               SXLEN-1                                                                  0
                                          Interrupts (WARL)
                                                SXLEN

**Figure 4.5:** Supervisor interrupt-enable register (sie).

mode is less than S, or if the current privilege mode is S and the SIE bit in the sstatus register is
set.

Each individual bit in register sip may be writable or may be read-only. When bit i in sip is
writable, a pending interrupt i can be cleared by writing 0 to this bit. If interrupt i can become
pending but bit i in sip is read-only, the implementation must provide some other mechanism for
clearing the pending interrupt (which may involve a call to the execution environment).

A bit in sie must be writable if the corresponding interrupt can ever become pending. Bits of sie
that are not writable must be hardwired to zero.

```text
The standard portions (bits 15:0) of registers sip and sie are formatted as shown in Figures 4.6
and 4.7 respectively.
                     15             10    9     8       6    5     4       2    1   0
                             0           SEIP       0       STIP       0       SSIP 0
                             6             1        3         1        3         1  1
```

**Figure 4.6:** Standard portion (bits 15:0) of sip.

```text
                     15             10    9     8       6    5     4       2    1   0
                             0           SEIE       0       STIE       0       SSIE 0
                             6             1        3         1        3         1  1
```

**Figure 4.7:** Standard portion (bits 15:0) of sie.

Bits sip.SEIP and sie.SEIE are the interrupt-pending and interrupt-enable bits for supervisorlevel external interrupts. If implemented, SEIP is read-only in sip, and is set and cleared by the
execution environment, typically through a platform-specific interrupt controller.

Bits sip.STIP and sie.STIE are the interrupt-pending and interrupt-enable bits for supervisorlevel timer interrupts. If implemented, STIP is read-only in sip, and is set and cleared by the
execution environment.

Bits sip.SSIP and sie.SSIE are the interrupt-pending and interrupt-enable bits for supervisor-level
software interrupts. If implemented, SSIP is writable in sip. A supervisor-level software interrupt
is triggered on the current hart by writing 1 to SSIP, while a pending supervisor-level software
interrupt can be cleared by writing 0 to SSIP.

<!-- p.223 -->

       Interprocessor interrupts are sent to other harts by implementation-specific means, which will
       ultimately cause the SSIP bit to be set in the recipient hart’s sip register.

Each standard interrupt type (SEI, STI, or SSI) may not be implemented, in which case the
corresponding interrupt-pending and interrupt-enable bits are hardwired to zeros. All bits in sip
and sie are WARL fields. The implemented interrupts may be found by writing one to every bit
location in sie, then reading back to see which bit positions hold a one.

       The sip and sie registers are subsets of the mip and mie registers. Reading any implemented
       field, or writing any writable field, of sip/sie effects a read or write of the homonymous field
       of mip/mie.
            Bits 3, 7, and 11 of sip and sie correspond to the machine-mode software, timer, and
       external interrupts, respectively. Since most platforms will choose not to make these interrupts
       delegatable from M-mode to S-mode, they are shown as hardwired to 0 in Figures 4.6 and 4.7.

Multiple simultaneous interrupts destined for supervisor mode are handled in the following decreasing priority order: SEI, SSI, STI. Synchronous exceptions are of lower priority than all interrupts.

##### 4.1.4 Supervisor Timers and Performance Counters

Supervisor software uses the same hardware performance monitoring facility as user-mode software,
including the time, cycle, and instret CSRs. The implementation should provide a mechanism
to modify the counter values.

The implementation must provide a facility for scheduling timer interrupts in terms of the real-time
counter, time.

##### 4.1.5 Counter-Enable Register (scounteren)

```text
      31       30       29      28                                6     5       4       3     2    1      0
     HPM31 HPM30 HPM29                          ...                   HPM5 HPM4 HPM3 IR TM CY
       1     1     1                            23                      1    1    1   1  1  1
```

**Figure 4.8:** Counter-enable register (scounteren).

The counter-enable register scounteren is a 32-bit register that controls the availability of the
hardware performance monitoring counters to U-mode.

When the CY, TM, IR, or HPMn bit in the scounteren register is clear, attempts to read the cycle,
time, instret, or hpmcountern register while executing in U-mode will cause an illegal instruction
exception. When one of these bits is set, access to the corresponding register is permitted.

scounteren must be implemented. However, any of the bits may contain a hardwired value of zero,
indicating reads to the corresponding counter will cause an exception when executing in U-mode.
Hence, they are effectively WARL fields.

<!-- p.224 -->

     The setting of a bit in mcounteren does not affect whether the corresponding bit in scounteren
     is writable. However, U-mode may only access a counter if the corresponding bits in scounteren
     and mcounteren are both set.

##### 4.1.6 Supervisor Scratch Register (sscratch)

The sscratch register is an SXLEN-bit read/write register, dedicated for use by the supervisor.
Typically, sscratch is used to hold a pointer to the hart-local supervisor context while the hart is
executing user code. At the beginning of a trap handler, sscratch is swapped with a user register
to provide an initial working register.
               SXLEN-1                                                                 0
                                               sscratch
                                               SXLEN

**Figure 4.9:** Supervisor Scratch Register.

##### 4.1.7 Supervisor Exception Program Counter (sepc)

sepc is an SXLEN-bit read/write register formatted as shown in Figure 4.10. The low bit of sepc
(sepc[0]) is always zero. On implementations that support only IALIGN=32, the two low bits
(sepc[1:0]) are always zero.

If an implementation allows IALIGN to be either 16 or 32 (by changing CSR misa, for example),
then, whenever IALIGN=32, bit sepc[1] is masked on reads so that it appears to be 0. This
masking occurs also for the implicit read by the SRET instruction. Though masked, sepc[1]
remains writable when IALIGN=32.

sepc is a WARL register that must be able to hold all valid virtual addresses. It need not be
capable of holding all possible invalid addresses. Implementations may convert some invalid address
patterns into other invalid addresses prior to writing them to sepc.

When a trap is taken into S-mode, sepc is written with the virtual address of the instruction
that was interrupted or that encountered the exception. Otherwise, sepc is never written by the
implementation, though it may be explicitly written by software.
               SXLEN-1                                                                 0
                                                 sepc
                                               SXLEN

**Figure 4.10:** Supervisor exception program counter register.

##### 4.1.8 Supervisor Cause Register (scause)

The scause register is an SXLEN-bit read-write register formatted as shown in Figure 4.11. When
a trap is taken into S-mode, scause is written with a code indicating the event that caused the trap.

<!-- p.225 -->

Otherwise, scause is never written by the implementation, though it may be explicitly written by
software.

The Interrupt bit in the scause register is set if the trap was caused by an interrupt. The Exception
Code field contains a code identifying the last exception or interrupt. Table 4.2 lists the possible
exception codes for the current supervisor ISAs. The Exception Code is a WLRL field. It is
required to hold the values 0–31 (i.e., bits 4–0 must be implemented), but otherwise it is only
guaranteed to hold supported exception codes.
                  SXLEN-1 SXLEN-2                                                   0
                 Interrupt                    Exception Code (WLRL)
                      1                              SXLEN-1

**Figure 4.11:** Supervisor Cause register scause.

```text
                  Interrupt      Exception Code    Description
                          1                   0    Reserved
                          1                   1    Supervisor software interrupt
                          1                 2–4    Reserved
                          1                   5    Supervisor timer interrupt
                          1                 6–8    Reserved
                          1                   9    Supervisor external interrupt
                          1               10–15    Reserved
                          1                 ≥16    Designated for platform use
                          0                   0    Instruction address misaligned
                          0                   1    Instruction access fault
                          0                   2    Illegal instruction
                          0                   3    Breakpoint
                          0                   4    Load address misaligned
                          0                   5    Load access fault
                          0                   6    Store/AMO address misaligned
                          0                   7    Store/AMO access fault
                          0                   8    Environment call from U-mode
                          0                   9    Environment call from S-mode
                          0               10–11    Reserved
                          0                  12    Instruction page fault
                          0                  13    Load page fault
                          0                  14    Reserved
                          0                  15    Store/AMO page fault
                          0               16–23    Reserved
                          0               24–31    Designated for custom use
                          0               32–47    Reserved
                          0               48–63    Designated for custom use
                          0                 ≥64    Reserved
```

**Table 4.2:** Supervisor cause register (scause) values after trap. Synchronous exception priorities are given by Table 3.7.

<!-- p.226 -->

##### 4.1.9 Supervisor Trap Value (stval) Register

The stval register is an SXLEN-bit read-write register formatted as shown in Figure 4.12. When
a trap is taken into S-mode, stval is written with exception-specific information to assist software
in handling the trap. Otherwise, stval is never written by the implementation, though it may
be explicitly written by software. The hardware platform will specify which exceptions must set
stval informatively and which may unconditionally set it to zero.

When a breakpoint, address-misaligned, access-fault, or page-fault exception occurs on an instruction fetch, load, or store, stval is written with the faulting virtual address. On an illegal instruction
trap, stval may be written with the first XLEN or ILEN bits of the faulting instruction as described below. For other exceptions, stval is set to zero, but a future standard may redefine
stval’s setting for other exceptions.
                SXLEN-1                                                                0
                                                 stval
                                                SXLEN

**Figure 4.12:** Supervisor Trap Value register.

For misaligned loads and stores that cause access-fault or page-fault exceptions, stval will contain
the virtual address of the portion of the access that caused the fault. For instruction access-fault or
page-fault exceptions on systems with variable-length instructions, stval will contain the virtual
address of the portion of the instruction that caused the fault while sepc will point to the beginning
of the instruction.

The stval register can optionally also be used to return the faulting instruction bits on an illegal
instruction exception (sepc points to the faulting instruction in memory).

If this feature is not provided, then stval is set to zero on an illegal instruction fault.

If this feature is provided, after an illegal instruction trap, stval will contain the shortest of:

- the actual faulting instruction
- the first ILEN bits of the faulting instruction
- the first XLEN bits of the faulting instruction

The value loaded into stval is right-justified and all unused upper bits are cleared to zero.

stval is a WARL register that must be able to hold all valid virtual addresses and the value 0. It
need not be capable of holding all possible invalid addresses. Implementations may convert some
invalid address patterns into other invalid addresses prior to writing them to stval. If the feature
to return the faulting instruction bits is implemented, stval must also be able to hold all values
less than 2N , where N is the smaller of XLEN and ILEN.

##### 4.1.10 Supervisor Address Translation and Protection (satp) Register

The satp register is an SXLEN-bit read/write register, formatted as shown in Figure 4.13 for
SXLEN=32 and Figure 4.14 for SXLEN=64, which controls supervisor-mode address translation and protection. This register holds the physical page number (PPN) of the root page table, i.e., its
supervisor physical address divided by 4 KiB; an address space identifier (ASID), which facilitates
address-translation fences on a per-address-space basis; and the MODE field, which selects the
current address-translation scheme. Further details on the access to this register are described in
Section 3.1.6.5.
               31        30                   22 21                                            0
         MODE (WARL)          ASID (WARL)                         PPN (WARL)
                1                   9                                  22 Figure 4.13: RV32 Supervisor address translation and protection register satp.

<!-- p.227 -->

     Storing a PPN in satp, rather than a physical address, supports a physical address space larger
     than 4 GiB for RV32.
         The satp.PPN field might not be capable of holding all physical page numbers. Some platform
     standards might place constraints on the values satp.PPN may assume, e.g., by requiring that
     all physical page numbers corresponding to main memory be representable.

```text
63            60 59                         44 43                                                      0
 MODE (WARL)            ASID (WARL)                                   PPN (WARL)
       4                      16                                           44
```

**Figure 4.14:** RV64 Supervisor address translation and protection register satp, for MODE values Bare, Sv39, Sv48, and Sv57.

     We store the ASID and the page table base address in the same CSR to allow the pair to be
     changed atomically on a context switch. Swapping them non-atomically could pollute the old
     virtual address space with new translations, or vice-versa. This approach also slightly reduces
     the cost of a context switch.

Table 4.3 shows the encodings of the MODE field for RV32 and RV64. When MODE=Bare,
supervisor virtual addresses are equal to supervisor physical addresses, and there is no additional
memory protection beyond the physical memory protection scheme described in Section 3.7. To
select MODE=Bare, software must write zero to the remaining fields of satp (bits 30–0 for RV32,
or bits 59–0 for RV64). Attempting to select MODE=Bare with a nonzero pattern in the remaining
fields has an unspecified effect on the value that the remaining fields assume and an unspecified
effect on address translation and protection behavior.

For RV32, the satp encodings corresponding to MODE=Bare and ASID[8:7]=3 are designated for
custom use, whereas the encodings corresponding to MODE=Bare and ASID[8:7]̸=3 are reserved
for future standard use. For RV64, all satp encodings corresponding to MODE=Bare are reserved
for future standard use.

     Version 1.11 of this standard stated that the remaining fields in satp had no effect when
     MODE=Bare. Making these fields reserved facilitates future definition of additional transla-
     tion and protection modes, particularly in RV32, for which all patterns of the existing MODE
     field have already been allocated.

For RV32, the only other valid setting for MODE is Sv32, a paged virtual-memory scheme described
in Section 4.3.

<!-- p.228 -->

For RV64, three paged virtual-memory schemes are defined: Sv39, Sv48, and Sv57, described in
Sections 4.4, 4.5, and 4.6, respectively. One additional scheme, Sv64, will be defined in a later
version of this specification. The remaining MODE settings are reserved for future use and may
define different interpretations of the other fields in satp.

Implementations are not required to support all MODE settings, and if satp is written with an
unsupported MODE, the entire write has no effect; no fields in satp are modified.

```text
                                                   RV32
              Value    Name      Description
                0      Bare      No translation or protection.
                1      Sv32      Page-based 32-bit virtual addressing (see Section       4.3).
                                                 RV64
              Value    Name      Description
                0      Bare      No translation or protection.
               1–7      —        Reserved for standard use
                8      Sv39      Page-based 39-bit virtual addressing (see Section       4.4).
                9      Sv48      Page-based 48-bit virtual addressing (see Section       4.5).
                10     Sv57      Page-based 57-bit virtual addressing (see Section       4.6).
                11     Sv64      Reserved for page-based 64-bit virtual addressing.
              12–13     —        Reserved for standard use
              14–15     —        Designated for custom use
```

**Table 4.3:** Encoding of satp MODE field.

The number of ASID bits is unspecified and may be zero. The number of implemented ASID
bits, termed ASIDLEN, may be determined by writing one to every bit position in the ASID field,
then reading back the value in satp to see which bit positions in the ASID field hold a one. The
least-significant bits of ASID are implemented first: that is, if ASIDLEN > 0, ASID[ASIDLEN-1:0]
is writable. The maximal value of ASIDLEN, termed ASIDMAX, is 9 for Sv32 or 16 for Sv39,
Sv48, and Sv57.

     For many applications, the choice of page size has a substantial performance impact. A large page
     size increases TLB reach and loosens the associativity constraints on virtually indexed, physically
     tagged caches. At the same time, large pages exacerbate internal fragmentation, wasting physical
     memory and possibly cache capacity.
         After much deliberation, we have settled on a conventional page size of 4 KiB for both RV32
     and RV64. We expect this decision to ease the porting of low-level runtime software and device
     drivers. The TLB reach problem is ameliorated by transparent superpage support in modern
     operating systems [2]. Additionally, multi-level TLB hierarchies are quite inexpensive relative to
     the multi-level cache hierarchies whose address space they map.

The satp register is considered active when the effective privilege mode is S-mode or U-mode, i.e.,
when in S-mode or U-mode, or when MPRV=1 and either MPP=S or MPP=U. Executions of the
address-translation algorithm may only begin using a given value of satp when satp is active.

     Translations that began while satp was active are not required to complete or terminate when
     satp is no longer active, unless an SFENCE.VMA instruction matching the address and ASID
     is executed. The SFENCE.VMA instruction must be used to ensure that updates to the addresstranslation data structures are observed by subsequent implicit reads to those structures by a
      hart.

<!-- p.229 -->

Note that writing satp does not imply any ordering constraints between page-table updates and
subsequent address translations, nor does it imply any invalidation of address-translation caches.
If the new address space’s page tables have been modified, or if an ASID is reused, it may be
necessary to execute an SFENCE.VMA instruction (see Section 4.2.1) after writing satp.

      Not imposing upon implementations to flush address-translation caches upon satp writes reduces
      the cost of context switches, provided a sufficiently large ASID space.

#### 4.2 Supervisor Instructions

In addition to the SRET instruction defined in Section 3.3.2, one new supervisor-level instruction
is provided.

##### 4.2.1 Supervisor Memory-Management Fence Instruction

```text
        31                25 24          20 19            15 14   12 11           7 6               0
            funct7                rs2              rs1       funct3       rd             opcode
               7                   5                5          3           5               7
         SFENCE.VMA               asid            vaddr      PRIV          0            SYSTEM
```

The supervisor memory-management fence instruction SFENCE.VMA is used to synchronize updates to in-memory memory-management data structures with current execution. Instruction execution causes implicit reads and writes to these data structures; however, these implicit references
are ordinarily not ordered with respect to explicit loads and stores. Executing an SFENCE.VMA
instruction guarantees that any previous stores already visible to the current RISC-V hart are
ordered before certain implicit references by subsequent instructions in that hart to the memorymanagement data structures. The specific set of operations ordered by SFENCE.VMA is determined by rs1 and rs2, as described below. SFENCE.VMA is also used to invalidate entries in the
address-translation cache associated with a hart (see Section 4.3.2). Further details on the behavior
of this instruction are described in Section 3.1.6.5 and Section 3.7.2.

      The SFENCE.VMA is used to flush any local hardware caches related to address translation.
      It is specified as a fence rather than a TLB flush to provide cleaner semantics with respect to
      which instructions are affected by the flush operation and to support a wider variety of dynamic
      caching structures and memory-management schemes. SFENCE.VMA is also used by higher
      privilege levels to synchronize page table writes and the address translation hardware.

SFENCE.VMA orders only the local hart’s implicit references to the memory-management data
structures.

      Consequently, other harts must be notified separately when the memory-management data struc-
      tures have been modified. One approach is to use 1) a local data fence to ensure local writes are visible globally, then 2) an interprocessor interrupt to the other thread, then 3) a local
     SFENCE.VMA in the interrupt handler of the remote thread, and finally 4) signal back to orig-
     inating thread that operation is complete. This is, of course, the RISC-V analog to a TLB
     shootdown.

<!-- p.230 -->

For the common case that the translation data structures have only been modified for a single
address mapping (i.e., one page or superpage), rs1 can specify a virtual address within that mapping
to effect a translation fence for that mapping only. Furthermore, for the common case that the
translation data structures have only been modified for a single address-space identifier, rs2 can
specify the address space. The behavior of SFENCE.VMA depends on rs1 and rs2 as follows:

- If rs1=x0 and rs2=x0, the fence orders all reads and writes made to any level of the page tables, for all address spaces. The fence also invalidates all address-translation cache entries, for all address spaces.

- If rs1=x0 and rs2̸=x0, the fence orders all reads and writes made to any level of the page tables, but only for the address space identified by integer register rs2. Accesses to global mappings (see Section 4.3.1) are not ordered. The fence also invalidates all address-translation cache entries matching the address space identified by integer register rs2, except for entries containing global mappings.

- If rs1̸=x0 and rs2=x0, the fence orders only reads and writes made to leaf page table entries corresponding to the virtual address in rs1, for all address spaces. The fence also invalidates all address-translation cache entries that contain leaf page table entries corresponding to the virtual address in rs1, for all address spaces.

- If rs1̸=x0 and rs2̸=x0, the fence orders only reads and writes made to leaf page table entries corresponding to the virtual address in rs1, for the address space identified by integer register rs2. Accesses to global mappings are not ordered. The fence also invalidates all addresstranslation cache entries that contain leaf page table entries corresponding to the virtual address in rs1 and that match the address space identified by integer register rs2, except for entries containing global mappings.

If the value held in rs1 is not a valid virtual address, then the SFENCE.VMA instruction has no
effect. No exception is raised in this case.

When rs2̸=x0, bits SXLEN-1:ASIDMAX of the value held in rs2 are reserved for future use and
should be zeroed by software and ignored by current implementations. Furthermore, if ASIDLEN < ASIDMAX, the implementation shall ignore bits ASIDMAX-1:ASIDLEN of the value
held in rs2.

     It is always legal to over-fence, e.g., by fencing only based on a subset of the bits in rs1 and/or
     rs2, and/or by simply treating all SFENCE.VMA instructions as having rs1=x0 and/or rs2=x0.
     For example, simpler implementations can ignore the virtual address in rs1 and the ASID value
     in rs2 and always perform a global fence. The choice not to raise an exception when an invalid
     virtual address is held in rs1 facilicates this type of simplification.

     When synchronizing the address-translation data structures for multiple individual pages page
     at once, e.g., when updating a range of page table entries, it is recommended to insert at most

<!-- p.231 -->

```text
  # a0 holds    the address of the first VA to be invalidated
  # a1 holds    PAGESIZE
  # a2 holds    the upper bound
loop:
  sfence.vma    a0
  add           a0,a0,a1
  bgtu          a2,a0,loop
```

**Figure 4.15:** Sample code to synchronize a range of page table entries.

     one compressed or uncompressed integer add instruction and one compressed or uncompressed
     branch instruction between consecutive SFENCE.VMA instructions. This idiom, one form of
     which is demonstrated in Figure 4.15, will make it easier for certain implementations to detect
     the pattern and amortize any synchronization overheads across the whole sequence. This idiom
     is recommended strictly as a performance optimization; the functionality of the SFENCE.VMA
     instructions is unaffected regardless of whether the idiom is used.

An implicit read of the memory-management data structures may return any translation for an
address that was valid at any time since the most recent SFENCE.VMA that subsumes that address.
The ordering implied by SFENCE.VMA does not place implicit reads and writes to the memorymanagement data structures into the global memory order in a way that interacts cleanly with
the standard RVWMO ordering rules. In particular, even though an SFENCE.VMA orders prior
explicit accesses before subsequent implicit accesses, and those implicit accesses are ordered before
their associated explicit accesses, SFENCE.VMA does not necessarily place prior explicit accesses
before subsequent explicit accesses in the global memory order. These implicit loads also need not
otherwise obey normal program order semantics with respect to prior loads or stores to the same
address.

     A consequence of this specification is that if a leaf PTE is modified but a subsuming
     SFENCE.VMA is not executed, either the old translation or the new translation will be used,
     but the choice is unpredictable. The behavior is otherwise well-defined.
          In a conventional TLB design, it is possible for multiple entries to match a single address if,
     for example, a page is upgraded to a superpage without first clearing the original non-leaf PTE’s
     valid bit and executing an SFENCE.VMA with rs1=x0. In this case, a similar remark applies:
     it is unpredictable whether the old non-leaf PTE or the new leaf PTE is used, but the behavior
     is otherwise well defined.
          Another consequence of this specification is that it is generally unsafe to update a PTE using
     a set of stores of a width less than the width of the PTE, as it is legal for the implementation
     to read the PTE at any time, including when only some of the partial stores have taken effect.

     This specification permits the caching of PTEs whose V (Valid) bit is clear. Operating systems
     must be written to cope with this possibility, but implementers are reminded that eagerly caching
     invalid PTEs will reduce performance by causing additional page faults.

Changes to the sstatus fields SUM and MXR take effect immediately, without the need to execute
an SFENCE.VMA instruction. Changing satp.MODE from Bare to other modes and vice versa
also takes effect immediately, without the need to execute an SFENCE.VMA instruction. Likewise,
changes to satp.ASID take effect immediately.

<!-- p.232 -->

      The following common situations typically require executing an SFENCE.VMA instruction:
- When software recycles an ASID (i.e., reassociates it with a different page table), it should first change satp to point to the new page table using the recycled ASID, then execute SFENCE.VMA with rs1=x0 and rs2 set to the recycled ASID. Alternatively, software can execute the same SFENCE.VMA instruction while a different ASID is loaded into satp, provided the next time satp is loaded with the recycled ASID, it is simultaneously loaded with the new page table.
- If the implementation does not provide ASIDs, or software chooses to always use ASID 0, then after every satp write, software should execute SFENCE.VMA with rs1=x0. In the common case that no global translations have been modified, rs2 should be set to a register other than x0 but which contains the value zero, so that global translations are not flushed.
- If software modifies a non-leaf PTE, it should execute SFENCE.VMA with rs1=x0. If any PTE along the traversal path had its G bit set, rs2 must be x0; otherwise, rs2 should be set to the ASID for which the translation is being modified.
- If software modifies a leaf PTE, it should execute SFENCE.VMA with rs1 set to a virtual address within the page. If any PTE along the traversal path had its G bit set, rs2 must be x0; otherwise, rs2 should be set to the ASID for which the translation is being modified.
- For the special cases of increasing the permissions on a leaf PTE and changing an invalid PTE to a valid leaf, software may choose to execute the SFENCE.VMA lazily. After modifying the PTE but before executing SFENCE.VMA, either the new or old permissions will be used. In the latter case, a page-fault exception might occur, at which point software should execute SFENCE.VMA in accordance with the previous bullet point.

If a hart employs an address-translation cache, that cache must appear to be private to that hart.
In particular, the meaning of an ASID is local to a hart; software may choose to use the same ASID
to refer to different address spaces on different harts.

      A future extension could redefine ASIDs to be global across the SEE, enabling such options as
      shared translation caches and hardware support for broadcast TLB shootdown. However, as OSes
      have evolved to significantly reduce the scope of TLB shootdowns using novel ASID-management
      techniques, we expect the local-ASID scheme to remain attractive for its simplicity and possibly
      better scalability.

#### 4.3 Sv32: Page-Based 32-bit Virtual-Memory Systems

When Sv32 is written to the MODE field in the satp register (see Section 4.1.10), the supervisor
operates in a 32-bit paged virtual-memory system. In this mode, supervisor and user virtual
addresses are translated into supervisor physical addresses by traversing a radix-tree page table.
Sv32 is supported on RV32 systems and is designed to include mechanisms sufficient for supporting
modern Unix-based operating systems.

      The initial RISC-V paged virtual-memory architectures have been designed as straightforward
      implementations to support existing operating systems. We have architected page table layouts
      to support a hardware page-table walker. Software TLB refills are a performance bottleneck on
      high-performance systems, and are especially troublesome with decoupled specialized coprocessors.
      An implementation can choose to implement software TLB refills using a machine-mode trap
      handler as an extension to M-mode.

<!-- p.233 -->

##### 4.3.1 Addressing and Memory Protection

Sv32 implementations support a 32-bit virtual address space, divided into 4 KiB pages. An Sv32
virtual address is partitioned into a virtual page number (VPN) and page offset, as shown in
Figure 4.16. When Sv32 virtual memory mode is selected in the MODE field of the satp register,
supervisor virtual addresses are translated into supervisor physical addresses via a two-level page
table. The 20-bit VPN is translated into a 22-bit physical page number (PPN), while the 12bit page offset is untranslated. The resulting supervisor-level physical addresses are then checked
using any physical memory protection structures (Sections 3.7), before being directly converted to
machine-level physical addresses. If necessary, supervisor-level physical addresses are zero-extended
to the number of physical address bits found in the implementation.

     For example, consider an RV32 system supporting 34 bits of physical address. When the value
     of satp.MODE is Sv32, a 34-bit physical address is produced directly, and therefore no zero-
     extension is needed. When the value of satp.MODE is Bare, the 32-bit virtual address is trans-
     lated (unmodified) into a 32-bit physical address, and then that physical address is zero-extended
     into a 34-bit machine-level physical address.

```text
                    31                  22 21                  12 11                             0
                          VPN[1]                    VPN[0]                 page offset
                            10                        10                      12
```

**Figure 4.16:** Sv32 virtual address.

```text
                  33                      22 21                    12 11                             0
                          PPN[1]                     PPN[0]                    page offset
                            12                         10                         12
```

**Figure 4.17:** Sv32 physical address.

```text
        31                   20 19                    10 9         8   7   6      5    4     3           2   1   0
               PPN[1]                   PPN[0]               RSW       D   A     G    U      X       W       R   V
                 12                       10                  2        1   1     1    1      1       1       1   1
```

**Figure 4.18:** Sv32 page table entry.

Sv32 page tables consist of 210 page-table entries (PTEs), each of four bytes. A page table is exactly
the size of a page and must always be aligned to a page boundary. The physical page number of
the root page table is stored in the satp register.

The PTE format for Sv32 is shown in Figures 4.18. The V bit indicates whether the PTE is valid; if
it is 0, all other bits in the PTE are don’t-cares and may be used freely by software. The permission
bits, R, W, and X, indicate whether the page is readable, writable, and executable, respectively.
When all three are zero, the PTE is a pointer to the next level of the page table; otherwise, it is
a leaf PTE. Writable pages must also be marked readable; the contrary combinations are reserved
for future use. Table 4.4 summarizes the encoding of the permission bits.

Attempting to fetch an instruction from a page that does not have execute permissions raises a
fetch page-fault exception. Attempting to execute a load or load-reserved instruction whose effective
address lies within a page without read permissions raises a load page-fault exception. Attempting

<!-- p.234 -->

```text
                         X    W    R    Meaning
                         0    0    0    Pointer to next level of page table.
                         0    0    1    Read-only page.
                         0    1    0    Reserved for future use.
                         0    1    1    Read-write page.
                         1    0    0    Execute-only page.
                         1    0    1    Read-execute page.
                         1    1    0    Reserved for future use.
                         1    1    1    Read-write-execute page.
```

**Table 4.4:** Encoding of PTE R/W/X fields.

to execute a store, store-conditional, or AMO instruction whose effective address lies within a page
without write permissions raises a store page-fault exception.

     AMOs never raise load page-fault exceptions. Since any unreadable page is also unwritable,
     attempting to perform an AMO on an unreadable page always raises a store page-fault exception.

The U bit indicates whether the page is accessible to user mode. U-mode software may only access
the page when U=1. If the SUM bit in the sstatus register is set, supervisor mode software may
also access pages with U=1. However, supervisor code normally operates with the SUM bit clear,
in which case, supervisor code will fault on accesses to user-mode pages. Irrespective of SUM, the
supervisor may not execute code on pages with U=1.

     An alternative PTE format would support different permissions for supervisor and user. We
     omitted this feature because it would be largely redundant with the SUM mechanism (see Sec-
     tion 4.1.1.2) and would require more encoding space in the PTE.

The G bit designates a global mapping. Global mappings are those that exist in all address spaces.
For non-leaf PTEs, the global setting implies that all mappings in the subsequent levels of the page
table are global. Note that failing to mark a global mapping as global merely reduces performance,
whereas marking a non-global mapping as global is a software bug that, after switching to an
address space with a different non-global mapping for that address range, can unpredictably result
in either mapping being used.

     Global mappings need not be stored redundantly in address-translation caches for multiple
     ASIDs. Additionally, they need not be flushed from local address-translation caches when an
     SFENCE.VMA instruction is executed with rs2̸=x0.

The RSW field is reserved for use by supervisor software; the implementation shall ignore this field.

Each leaf PTE contains an accessed (A) and dirty (D) bit. The A bit indicates the virtual page has
been read, written, or fetched from since the last time the A bit was cleared. The D bit indicates
the virtual page has been written since the last time the D bit was cleared.

Two schemes to manage the A and D bits are permitted:

<!-- p.235 -->

- When a virtual page is accessed and the A bit is clear, or is written and the D bit is clear, a page-fault exception is raised.

- When a virtual page is accessed and the A bit is clear, or is written and the D bit is clear, the implementation sets the corresponding bit(s) in the PTE. The PTE update must be atomic with respect to other accesses to the PTE, and must atomically check that the PTE is valid and grants sufficient permissions. Updates of the A bit may be performed as a result of speculation, but updates to the D bit must be exact (i.e., not speculative), and observed in program order by the local hart. Furthermore, the PTE update must appear in the global memory order no later than the explicit memory access, or any subsequent explicit memory access to that virtual page by the local hart. The ordering on loads and stores provided by FENCE instructions and the acquire/release bits on atomic instructions also orders the PTE updates associated with those loads and stores as observed by remote harts. The PTE update is not required to be atomic with respect to the explicit memory access that caused the update, and the sequence is interruptible. However, the hart must not perform the explicit memory access before the PTE update is globally visible.

All harts in a system must employ the same PTE-update scheme as each other.

       Prior versions of this specification required PTE A bit updates to be exact, but allowing the A
       bit to be updated as a result of speculation simplifies the implementation of address translation
       prefetchers. System software typically uses the A bit as a page replacement policy hint, but
       does not require exactness for functional correctness. On the other hand, D bit updates are
       still required to be exact and performed in program order, as the D bit affects the functional
       correctness of page eviction.
            Implementations are of course still permitted to perform both A and D bit updates only in
       an exact manner.
            In both cases, requiring atomicity ensures that the PTE update will not be interrupted by
       other intervening writes to the page table, as such interruptions could lead to A/D bits being set
       on PTEs that have been reused for other purposes, on memory that has been reclaimed for other
       purposes, and so on. Simple implementations may instead generate page-fault exceptions.
            The A and D bits are never cleared by the implementation. If the supervisor software does
       not rely on accessed and/or dirty bits, e.g. if it does not swap memory pages to secondary storage
       or if the pages are being used to map I/O space, it should always set them to 1 in the PTE to
       improve performance.

Any level of PTE may be a leaf PTE, so in addition to 4 KiB pages, Sv32 supports 4 MiB megapages.
A megapage must be virtually and physically aligned to a 4 MiB boundary; a page-fault exception
is raised if the physical address is insufficiently aligned.

For non-leaf PTEs, the D, A, and U bits are reserved for future standard use and must be cleared
by software for forward compatibility.

For implementations with both page-based virtual memory and the “A” standard extension, the
LR/SC reservation set must lie completely within a single base page (i.e., a naturally aligned 4 KiB
region).

<!-- p.236 -->

##### 4.3.2 Virtual Address Translation Process

A virtual address va is translated into a physical address pa as follows:

  1. Let a be satp.ppn × PAGESIZE, and let i = LEVELS − 1. (For Sv32, PAGESIZE=212 and
     LEVELS=2.) The satp register must be active, i.e., the effective privilege mode must be
     S-mode or U-mode.

  2. Let pte be the value of the PTE at address a+va.vpn[i]×PTESIZE. (For Sv32, PTESIZE=4.)
     If accessing pte violates a PMA or PMP check, raise an access-fault exception corresponding
     to the original access type.

  3. If pte.v = 0, or if pte.r = 0 and pte.w = 1, or if any bits or encodings that are reserved for
     future standard use are set within pte, stop and raise a page-fault exception corresponding
     to the original access type.

  4. Otherwise, the PTE is valid. If pte.r = 1 or pte.x = 1, go to step 5. Otherwise, this PTE is a
     pointer to the next level of the page table. Let i = i − 1. If i < 0, stop and raise a page-fault
     exception corresponding to the original access type. Otherwise, let a = pte.ppn × PAGESIZE
     and go to step 2.

  5. A leaf PTE has been found. Determine if the requested memory access is allowed by the
     pte.r, pte.w, pte.x, and pte.u bits, given the current privilege mode and the value of the
     SUM and MXR fields of the mstatus register. If not, stop and raise a page-fault exception
     corresponding to the original access type.

  6. If i > 0 and pte.ppn[i − 1 : 0] ̸= 0, this is a misaligned superpage; stop and raise a page-fault
     exception corresponding to the original access type.

  7. If pte.a = 0, or if the original memory access is a store and pte.d = 0, either raise a page-fault
     exception corresponding to the original access type, or:

- If a store to pte would violate a PMA or PMP check, raise an access-fault exception corresponding to the original access type.
- Perform the following steps atomically: – Compare pte to the value of the PTE at address a + va.vpn[i] × PTESIZE. – If the values match, set pte.a to 1 and, if the original memory access is a store, also set pte.d to 1. – If the comparison fails, return to step 2

  8. The translation is successful. The translated physical address is given as follows:

- pa.pgoff = va.pgoff.
- If i > 0, then this is a superpage translation and pa.ppn[i − 1 : 0] = va.vpn[i − 1 : 0].
- pa.ppn[LEVELS − 1 : i] = pte.ppn[LEVELS − 1 : i].

All implicit accesses to the address-translation data structures in this algorithm are performed
using width PTESIZE.

<!-- p.237 -->

     This implies, for example, that an Sv48 implementation may not use two separate 4B reads to
     non-atomically access a single 8B PTE, and that A/D bit updates performed by the implemen-
     tation are treated as atomically updating the entire PTE, rather than just the A and/or D bit
     alone (even though the PTE value does not otherwise change).

An implicit read in step 2 may return any translation for an address that was valid (i.e., could
be generated by executing the address-translation algorithm) at any time since the most recent
SFENCE.VMA that subsumes that address. The results of implicit address-translation reads in step
2 may be held in a read-only, incoherent address-translation cache but not shared with other harts.
The address-translation cache may hold an arbitrary number of entries, including an arbitrary
number of entries for the same address and ASID. Entries in the address-translation cache may
then satisfy subsequent step 2 reads if the ASID associated with the entry matches the ASID loaded
in step 0 or if the entry is associated with a global mapping. To ensure that implicit reads observe
writes to the same memory locations, an SFENCE.VMA instruction must be executed after the
writes to flush the relevant cached translations.

The address-translation cache cannot be used in step 7; accessed and dirty bits may only be updated
in memory directly.

     It is permitted for multiple address-translation cache entries to co-exist for the same address.
     This represents the fact that in a conventional TLB hierarchy, it is possible for multiple entries
     to match a single address if, for example, a page is upgraded to a superpage without first clearing
     the original non-leaf PTE’s valid bit and executing an SFENCE.VMA with rs1=x0, or if multiple
     TLBs exist in parallel at a given level of the hierarchy. In this case, just as if an SFENCE.VMA
     is not executed between a write to the memory-management tables and subsequent implicit read
     of the same address: it is unpredictable whether the old non-leaf PTE or the new leaf PTE is
     used, but the behavior is otherwise well defined.

Implementations may also execute the address-translation algorithm speculatively at any time, for
any virtual address, as long as satp.MODE indicates that virtual addressing is enabled. Such
speculative executions have the effect of pre-populating the address-translation cache. Implementations must only perform implicit reads of the translation data structures pointed to by the current
contents of the satp register or a subsequent valid (V=1) translation data structure entry.

Speculative executions of the address-translation algorithm behave as non-speculative executions
of the algorithm do, except that they must not set the dirty bit for a PTE, they must not trigger
an exception, and they must not create address-translation cache entries if those entries would
have been invalidated by any SFENCE.VMA instruction executed by the hart since the speculative
execution of the algorithm began.

     For instance, it is illegal for both non-speculative and speculative executions of the translation
     algorithm to begin, read the level 2 page table, pause while the hart executes an SFENCE.VMA
     with rs1=rs2=x0, then resume using the now-stale level 2 PTE, as subsequent implicit reads
     could populate the address-translation cache with stale PTEs.
         In many implementations, an SFENCE.VMA instruction with rs1=x0 will therefore either
     terminate all previously-launched speculative executions of the address-translation algorithm (for
     the specified ASID, if applicable), or simply wait for them to complete (in which case any
     address-translation cache entries created will be invalidated by the SFENCE.VMA as appro-
     priate). Likewise, an SFENCE.VMA instruction with rs1̸=x0 generally must either ensure that
     previously-launched speculative executions of the address-translation algorithm (for the specified ASID, if applicable) are prevented from creating new address-translation cache entries mapping
      leaf PTEs, or wait for them to complete.

<!-- p.238 -->

      A consequence of implementations being permitted to read the translation data structures arbi-
      trarily early and speculatively is that at any time, all page table entries reachable by executing
      the algorithm may be loaded into the address-translation cache.

      Although it would be uncommon to place page tables in non-idempotent memory, there is no
      explicit prohibition against doing so. Since the algorithm may only touch page tables reachable
      from the root page table indicated in satp, the range of addresses that an implementation’s page
      table walker will touch is fully under supervisor control.

#### 4.4 Sv39: Page-Based 39-bit Virtual-Memory System

This section describes a simple paged virtual-memory system designed for RV64 systems, which
supports 39-bit virtual address spaces. The design of Sv39 follows the overall scheme of Sv32, and
this section details only the differences between the schemes.

      We specified multiple virtual memory systems for RV64 to relieve the tension between providing
      a large address space and minimizing address-translation cost. For many systems, 512 GiB of
      virtual-address space is ample, and so Sv39 suffices. Sv48 increases the virtual address space
      to 256 TiB, but increases the physical memory capacity dedicated to page tables, the latency
      of page-table traversals, and the size of hardware structures that store virtual addresses. Sv57
      increases the virtual address space, page table capacity requirement, and translation latency even
      further.

##### 4.4.1 Addressing and Memory Protection

Sv39 implementations support a 39-bit virtual address space, divided into 4 KiB pages. An Sv39
address is partitioned as shown in Figure 4.19. Instruction fetch addresses and load and store
effective addresses, which are 64 bits, must have bits 63–39 all equal to bit 38, or else a page-fault
exception will occur. The 27-bit VPN is translated into a 44-bit PPN via a three-level page table,
while the 12-bit page offset is untranslated.

      When mapping between narrower and wider addresses, RISC-V zero-extends a narrower physical
      address to a wider size. The mapping between 64-bit virtual addresses and the 39-bit usable
      address space of Sv39 is not based on zero-extension but instead follows an entrenched convention
      that allows an OS to use one or a few of the most-significant bits of a full-size (64-bit) virtual
      address to quickly distinguish user and supervisor address regions.

```text
            38                30 29                 21 20                12 11                  0
                  VPN[2]                 VPN[1]              VPN[0]               page offset
                    9                      9                   9                     12
```

**Figure 4.19:** Sv39 virtual address.

<!-- p.239 -->

```text
       55                           30 29                     21 20                        12 11                     0
                     PPN[2]                      PPN[1]                      PPN[0]                page offset
                       26                          9                           9                      12
```

**Figure 4.20:** Sv39 physical address.

```text
 63   62     61 60          54 53        28 27        19 18           10 9         8   7     6     5   4   3     2       1   0
 N    PBMT       Reserved       PPN[2]       PPN[1]       PPN[0]             RSW       D     A     G   U   X     W       R   V
 1      2           7             26           9            9                 2        1     1     1   1   1     1       1   1
```

**Figure 4.21:** Sv39 page table entry.

Sv39 page tables contain 29 page table entries (PTEs), eight bytes each. A page table is exactly
the size of a page and must always be aligned to a page boundary. The physical page number of
the root page table is stored in the satp register’s PPN field.

The PTE format for Sv39 is shown in Figure 4.21. Bits 9–0 have the same meaning as for Sv32.

Bit 63 is reserved for use by the Svnapot extension in Chapter 5. Bits 62–61 are reserved for use by
the Svpbmt extension in Chapter 6. Bits 60–54 are reserved for future standard use. All of these
bits must be zeroed by software for forward compatibility. If any of these bits are set, a page-fault
exception is raised.

      We reserved several PTE bits for a possible extension that improves support for sparse address
      spaces by allowing page-table levels to be skipped, reducing memory usage and TLB refill latency.
      These reserved bits may also be used to facilitate research experimentation. The cost is reducing
      the physical address space, but 64 PiB is presently ample. When it no longer suffices, the reserved
      bits that remain unallocated could be used to expand the physical address space.

Any level of PTE may be a leaf PTE, so in addition to 4 KiB pages, Sv39 supports 2 MiB megapages
and 1 GiB gigapages, each of which must be virtually and physically aligned to a boundary equal
to its size. A page-fault exception is raised if the physical address is insufficiently aligned.

The algorithm for virtual-to-physical address translation is the same as in Section 4.3.2, except
LEVELS equals 3 and PTESIZE equals 8.

#### 4.5 Sv48: Page-Based 48-bit Virtual-Memory System

This section describes a simple paged virtual-memory system designed for RV64 systems, which
supports 48-bit virtual address spaces. Sv48 is intended for systems for which a 39-bit virtual
address space is insufficient. It closely follows the design of Sv39, simply adding an additional level
of page table, and so this chapter only details the differences between the two schemes.

Implementations that support Sv48 must also support Sv39.

      Systems that support Sv48 can also support Sv39 at essentially no cost, and so should do so to
      maintain compatibility with supervisor software that assumes Sv39.

<!-- p.240 -->

##### 4.5.1 Addressing and Memory Protection

Sv48 implementations support a 48-bit virtual address space, divided into 4 KiB pages. An Sv48
address is partitioned as shown in Figure 4.22. Instruction fetch addresses and load and store
effective addresses, which are 64 bits, must have bits 63–48 all equal to bit 47, or else a page-fault
exception will occur. The 36-bit VPN is translated into a 44-bit PPN via a four-level page table,
while the 12-bit page offset is untranslated.
 47                    39 38                      30 29                 21 20                      12 11                         0
         VPN[3]                       VPN[2]                VPN[1]                  VPN[0]                       page offset
           9                            9                     9                       9                             12

**Figure 4.22:** Sv48 virtual address.

```text
55                       39 38                      30 29                   21 20                    12 11                           0
         PPN[3]                        PPN[2]                 PPN[1]                  PPN[0]                       page offset
           17                            9                      9                       9                             12
```

**Figure 4.23:** Sv48 physical address.

```text
 63     62     61 60     54      53      37 36      28 27      19 18    10 9          8   7    6     5       4      3     2      1        0
 N      PBMT      Reserved       PPN[3]        PPN[2]     PPN[1]   PPN[0]       RSW       D    A     G       U      X    W       R       V
 1        2         7             17             9          9        9           2        1    1     1       1      1    1       1       1
```

**Figure 4.24:** Sv48 page table entry.

The PTE format for Sv48 is shown in Figure 4.24. Bits 63–54 and 9–0 have the same meaning as
for Sv39. Any level of PTE may be a leaf PTE, so in addition to 4 KiB pages, Sv48 supports 2 MiB
megapages, 1 GiB gigapages, and 512 GiB terapages, each of which must be virtually and physically
aligned to a boundary equal to its size. A page-fault exception is raised if the physical address is
insufficiently aligned.

The algorithm for virtual-to-physical address translation is the same as in Section 4.3.2, except
LEVELS equals 4 and PTESIZE equals 8.

#### 4.6 Sv57: Page-Based 57-bit Virtual-Memory System

This section describes a simple paged virtual-memory system designed for RV64 systems, which
supports 57-bit virtual address spaces. Sv57 is intended for systems for which a 48-bit virtual
address space is insufficient. It closely follows the design of Sv48, simply adding an additional level
of page table, and so this chapter only details the differences between the two schemes.

Implementations that support Sv57 must also support Sv48.

      Systems that support Sv57 can also support Sv48 at essentially no cost, and so should do so to
      maintain compatibility with supervisor software that assumes Sv48.

<!-- p.241 -->

##### 4.6.1 Addressing and Memory Protection

Sv57 implementations support a 57-bit virtual address space, divided into 4 KiB pages. An Sv57
address is partitioned as shown in Figure 4.25. Instruction fetch addresses and load and store
effective addresses, which are 64 bits, must have bits 63–57 all equal to bit 56, or else a page-fault
exception will occur. The 45-bit VPN is translated into a 44-bit PPN via a five-level page table,
while the 12-bit page offset is untranslated.

```text
56               48 47                    39 38                    30 29                  21 20                     12 11                       0
      VPN[4]              VPN[3]                   VPN[2]                   VPN[1]                       VPN[0]               page offset
        9                   9                        9                        9                            9                     12
```

**Figure 4.25:** Sv57 virtual address.

```text
 55            48 47                  39 38                   30 29                     21 20                     12 11                     0
      PPN[4]             PPN[3]                   PPN[2]                   PPN[1]                   PPN[0]                page offset
        8                  9                        9                        9                        9                      12
```

**Figure 4.26:** Sv57 physical address.

```text
      63 62     61 60             54 53                                    10 9     8    7      6        5   4     3      2     1    0
      N    PBMT        Reserved                       PPN                      RSW       D      A        G   U     X      W     R    V
       1     2            7                            44                       2        1      1        1   1     1      1     1    1
                                    53        46 45        37 36       28 27        19 18           10
                                     PPN[4]        PPN[3]     PPN[2]         PPN[1]       PPN[0]
                                       8             9          9              9            9
```

**Figure 4.27:** Sv57 page table entry.

The PTE format for Sv57 is shown in Figure 4.27. Bits 63–54 and 9–0 have the same meaning
as for Sv39. Any level of PTE may be a leaf PTE, so in addition to 4 KiB pages, Sv57 supports
2 MiB megapages, 1 GiB gigapages, 512 GiB terapages, and 256 TiB petapages, each of which must
be virtually and physically aligned to a boundary equal to its size. A page-fault exception is raised
if the physical address is insufficiently aligned.

The algorithm for virtual-to-physical address translation is the same as in Section 4.3.2, except
LEVELS equals 5 and PTESIZE equals 8.

<!-- p.242 -->

### 第 5 章 “Svnapot” Standard Extension for NAPOT Translation Contiguity, Version 0.1

In Sv39, Sv48, and Sv57, when a PTE has N=1, the PTE represents a translation that is part of
a range of contiguous virtual-to-physical translations with the same values for PTE bits 5–0. Such
ranges must be of a naturally aligned power-of-2 (NAPOT) granularity larger than the base page
size.

```text
                  i        pte.ppn[i]   Description                  pte.napot bits
                 0     x   xxxx xxx1    Reserved                           −
                 0     x   xxxx xx1x    Reserved                           −
                 0     x   xxxx x1xx    Reserved                           −
                 0     x   xxxx 1000    64 KiB contiguous region           4
                 0     x   xxxx 0xxx    Reserved                           −
                 ≥1    x   xxxx xxxx    Reserved                           −
```

**Table 5.1:** Page table entry encodings when pte.N=1

NAPOT PTEs behave identically to non-NAPOT PTEs within the address-translation algorithm
in Section 4.3.2, except that:

- If the encoding in pte is valid according to Table 5.1, then instead of returning the original value of pte, implicit reads of a NAPOT PTE return a copy of pte in which pte.ppn[pte.napot bits − 1 : 0] is replaced by vpn[i][pte.napot bits − 1 : 0]. If the encoding in pte is reserved according to Table 5.1, then a page-fault exception must be raised.

- Implicit reads of NAPOT page table entries may create address-translation cache entries mapping a + va.vpn[j] × PTESIZE to a copy of pte in which pte.ppn[pte.napot bits − 1 : 0] is replaced by vpn[0][pte.napot bits − 1 : 0], for any or all j such that j[8 : napot bits] = i[8 : napot bits], all for the address space identified in satp as loaded by step 0.

<!-- p.243 -->

     The motivation for a NAPOT PTE is that it can be cached in a TLB as one or more entries rep-
     resenting the contiguous region as if it were a single (large) page covered by a single translation.
     This compaction can help relieve TLB pressure in some scenarios. The encoding is designed
     to fit within the pre-existing Sv39, Sv48, and Sv57 PTE formats so as not to disrupt existing
     implementations or designs that choose not to implement the scheme. It is also designed so as
     not to complicate the definition of the address-translation algorithm.
         The address translation cache abstraction captures the behavior that would result from the
     creation of a single TLB entry covering the entire NAPOT region. It is also designed to be
     consistent with implementations that support NAPOT PTEs by splitting the NAPOT region into
     TLB entries covering any smaller power-of-two region sizes. For example, a 64KiB NAPOT
     PTE might trigger the creation of 16 standard 4KiB TLB entries, all with contents generated
     from the NAPOT PTE (even if the PTEs for the other 4KiB regions have different contents).
         In typical usage scenarios, NAPOT PTEs in the same region will have the same attributes,
     same PPNs, and same values for bits 5–0. RSW remains reserved for supervisor software
     control. It is the responsibility of the OS and/or hypervisor to configure the page tables in such
     a way that there are no inconsistencies between NAPOT PTEs and other NAPOT or non-
     NAPOT PTEs that overlap the same address range. If an update needs to be made, the OS
     generally should first mark all of the PTEs invalid, then issue SFENCE.VMA instruction(s)
     covering all 4KiB regions within the range (either via a single SFENCE.VMA with rs1=x0, or
     with multiple SFENCE.VMA instructions with rs1̸=x0), then update the PTE(s), as described in
     Section 4.2.1, unless any inconsistencies are known to be benign. If any inconsistencies do exist,
     then the effect is the same as when SFENCE.VMA is used incorrectly: one of the translations
     will be chosen, but the choice is unpredictable.
         When updating a region of NAPOT PTEs all at once, it is recommended that software
     continue to follow the idiom in Figure 4.15 in which no more than one add and one branch
     instruction is inserted between consecutive SFENCE.VMA instructions.
         If an implementation chooses to use a NAPOT PTE (or cached version thereof ), it might
     not consult the PTE directly specified by the algorithm in Section 4.3.2 at all. Therefore, the D
     and A bits may not be identical across all mappings of the same address range even in typical
     use cases The operating system must query all NAPOT aliases of a page to determine whether
     that page has been accessed and/or is dirty. If the OS manually sets the A and/or D bits for a
     page, it is recommended that the OS also set the A and/or D bits for other NAPOT aliases as
     appropriate in order to avoid unnecessary traps.
         Just as with normal PTEs, TLBs are permitted to cache NAPOT PTEs whose V (Valid) bit
     is clear.
         Depending on need, the NAPOT scheme may be extended to other intermediate page sizes
     and/or to other levels of the page table in the future. The encoding is designed to accommodate
     other NAPOT sizes should that need arise. For example:
                      i      pte.ppn[i]    Description                    pte.napot bits
                      0    x xxxx xxx1     8 KiB contiguous region               1
                      0    x xxxx xx10     16 KiB contiguous region              2
                      0    x xxxx x100     32 KiB contiguous region              3
                      0    x xxxx 1000     64 KiB contiguous region              4
                      0    x xxx1 0000     128 KiB contiguous region             5
                     ...         ...       ...                                  ...
                      1    x xxxx xxx1     4 MiB contiguous region               1
                      1    x xxxx xx10     8 MiB contiguous region               2
                     ...         ...       ...                                  ...
         In such a case, an implementation may or may not support all options. The discoverability
     mechanism for this extension would be extended to allow system software to determine which
     sizes are supported.

<!-- p.244 -->

         Other sizes may remain deliberately excluded, so that PPN bits not being used to indicate
     a valid NAPOT region size (e.g., the least-significant bit of pte.ppn[i]) may be repurposed for
     other uses in the future.
         However, in case finer-grained intermediate page size support proves not to be useful, we
     have chosen to standardize only 64KiB support as a first step.

<!-- p.246 -->

### 第 6 章 “Svpbmt” Standard Extension for Page-Based Memory Types, Version 0.1

In Sv39, Sv48, and Sv57, bits 62–61 of a leaf page table entry indicate the use of page-based memory
types that override the PMA(s) for the associated memory pages. The encoding for the PBMT
bits is captured in Table 6.1.

```text
 Mode    Value    Requested Memory Attributes
 PMA       0      None
  NC       1      Non-cacheable, idempotent, weakly-ordered (RVWMO or RVTSO), main memory
  IO       2      Non-cacheable, non-idempotent, strongly-ordered (I/O ordering), I/O
   −       3      Reserved for future standard use
```

**Table 6.1:** Encodings for the PBMT field in Sv39, Sv48, and Sv57 PTEs. Attributes not mentioned are inherited from the PMA associated with the physical address.

     Future extensions may provide more and/or finer-grained control over which PMAs can be over-
     ridden.

For non-leaf PTEs, bits 62–61 are reserved for future standard use and must be cleared by software
for forward compatibility.

If the underlying physical memory attribute for a page is main memory and the page has
PBMT=PMA or PBMT=NC, or if the underlying physical memory attribute for a page is I/O
and the page has PBMT=PMA or PBMT=IO, then accesses to that page obey the same memory
ordering rules normally applied to accesses to that physical page.

If the underlying physical memory attribute for a page is I/O, and the page has PBMT=NC, then
then accesses to that page obey RVWMO. Accesses to such pages are considered main memory
rather than I/O for the purposes of FENCE, .aq, and .rl.

<!-- p.247 -->

If the underlying physical memory attribute for a page is main memory, and the page has
PBMT=IO, then accesses to that page obey strong channel 0 I/O ordering rules with respect
to other accesses to physical main memory and to other accesses to pages with PBMT=IO. Furthermore, accesses to such pages are considered I/O rather than main memory for the purposes of
FENCE, .aq, and .rl.

When Svpbmt is used with non-zero PBMT encodings, it is possible for multiple virtual aliases
of the same physical page to exist simultaneously with different memory attributes. It is also
possible for a U-mode or S-mode mapping through a PTE with Svpbmt enabled to observe different
memory attributes for a given region of physical memory than a concurrent access to the same
page performed by M-mode or when MODE=Bare. In such cases, the behaviors dictated by the
attributes may be violated, and platform-specific mechanisms must be used to restore the expected
behaviors.

     The forthcoming Zicbom extension and the FENCE instruction will collectively form a standard
     mechanism for restoring coherence in scenarios with mismatched page attributes.

     A device driver written to rely on I/O strong ordering rules will not operate correctly if the
     address range is mapped as main memory by the page-based memory types. As such, this con-
     figuration is discouraged.
         It will often still be useful to map physical I/O regions using PBMT=NC so that write
     combining and speculative accesses can be performed. Such optimizations will likely improve
     performance when applied with adequate care.

When two-stage address translation is enabled within the H extension, the page-based memory
types are also applied in two stages. First, if hgatp.MODE is not equal to zero, the G-stage PTE
PBMT bits are applied to the attributes in the PMA to produce an intermediate set of attributes.
Otherwise, the PMAs serve as the intermediate attributes. Second, if vsatp.MODE is not equal to
zero, the VS-stage PTE PBMT bits are applied to the intermediate attributes to produce the final
set of attributes used by accesses to the page in question. Otherwise, the intermediate attributes
are used as the final set of attributes.

<!-- p.248 -->

### 第 7 章 Hypervisor Extension, Version 0.6.1

Warning! This draft specification may change before being accepted as standard by
the RISC-V Foundation.

This chapter describes the RISC-V hypervisor extension, which virtualizes the supervisor-level
architecture to support the efficient hosting of guest operating systems atop a type-1 or type-2
hypervisor. The hypervisor extension changes supervisor mode into hypervisor-extended supervisor
mode (HS-mode, or hypervisor mode for short), where a hypervisor or a hosting-capable operating
system runs. The hypervisor extension also adds another stage of address translation, from guest
physical addresses to supervisor physical addresses, to virtualize the memory and memory-mapped
I/O subsystems for a guest operating system. HS-mode acts the same as S-mode, but with additional instructions and CSRs that control the new stage of address translation and support hosting
a guest OS in virtual S-mode (VS-mode). Regular S-mode operating systems can execute without
modification either in HS-mode or as VS-mode guests.

In HS-mode, an OS or hypervisor interacts with the machine through the same SBI as an OS
normally does from S-mode. An HS-mode hypervisor is expected to implement the SBI for its
VS-mode guest.

The hypervisor extension is enabled by setting bit 7 in the misa CSR, which corresponds to the
letter H. RISC-V harts that implement the hypervisor extension are encouraged not to hardwire
misa[7], so that the extension may be disabled.

     The baseline privileged architecture is designed to simplify the use of classic virtualization tech-
     niques, where a guest OS is run at user-level, as the few privileged instructions can be easily
     detected and trapped. The hypervisor extension improves virtualization performance by reducing
     the frequency of these traps.
         The hypervisor extension has been designed to be efficiently emulable on platforms that do
     not implement the extension, by running the hypervisor in S-mode and trapping into M-mode
     for hypervisor CSR accesses and to maintain shadow page tables. The majority of CSR accesses
     for type-2 hypervisors are valid S-mode accesses so need not be trapped. Hypervisors can support
     nested virtualization analogously.

<!-- p.249 -->

#### 7.1 Privilege Modes

The current virtualization mode, denoted V, indicates whether the hart is currently executing in a
guest. When V=1, the hart is either in virtual S-mode (VS-mode), or in virtual U-mode (VU-mode)
atop a guest OS running in VS-mode. When V=0, the hart is either in M-mode, in HS-mode, or in
U-mode atop an OS running in HS-mode. The virtualization mode also indicates whether two-stage
address translation is active (V=1) or inactive (V=0). Table 7.1 lists the possible operating modes
of a RISC-V hart with the hypervisor extension.

```text
 Virtualization    Privilege                                                                 Two-Stage
                                Abbreviation     Name
   Mode (V)        Encoding                                                                  Translation
       0               0        U-mode           User mode                                   Off
       0               1        HS-mode          Hypervisor-extended supervisor mode         Off
       0               3        M-mode           Machine mode                                Off
       1               0        VU-mode          Virtual user mode                           On
       1               1        VS-mode          Virtual supervisor mode                     On
```

**Table 7.1:** Operating modes with the hypervisor extension.

For purposes of interrupt global enables, HS-mode is considered more privileged than VS-mode, and
VS-mode is considered more privileged than VU-mode. VS-mode interrupts are globally disabled
when executing in U-mode.

      This description does not consider the possibility of U-mode or VU-mode interrupts and will be
      revised if the N extension for user-level interrupts is ultimately adopted.

#### 7.2 Hypervisor and Virtual Supervisor CSRs

An OS or hypervisor running in HS-mode uses the supervisor CSRs to interact with the exception,
interrupt, and address-translation subsystems. Additional CSRs are provided to HS-mode, but not
to VS-mode, to manage two-stage address translation and to control the behavior of a VS-mode
guest: hstatus, hedeleg, hideleg, hvip, hip, hie, hgeip, hgeie, hcounteren, htimedelta,
htimedeltah, htval, htinst, and hgatp.

Furthermore, several virtual supervisor CSRs (VS CSRs) are replicas of the normal supervisor
CSRs. For example, vsstatus is the VS CSR that duplicates the usual sstatus CSR.

When V=1, the VS CSRs substitute for the corresponding supervisor CSRs, taking over all functions of the usual supervisor CSRs except as specified otherwise. Instructions that normally read or
modify a supervisor CSR shall instead access the corresponding VS CSR. When V=1, an attempt
to read or write a VS CSR directly by its own separate CSR address causes a virtual instruction
exception. (Attempts from U-mode cause an illegal instruction exception as usual.) The VS CSRs
can be accessed as themselves only from M-mode or HS-mode.

While V=1, the normal HS-level supervisor CSRs that are replaced by VS CSRs retain their values
but do not affect the behavior of the machine unless specifically documented to do so. Conversely, when V=0, the VS CSRs do not ordinarily affect the behavior of the machine other than being
readable and writable by CSR instructions.

<!-- p.250 -->

A few standard supervisor CSRs (scounteren and, if the N extension is implemented, sedeleg and
sideleg) have no matching VS CSR. These supervisor CSRs continue to have their usual function
and accessibility even when V=1, except with VS-mode and VU-mode substituting for HS-mode
and U-mode. Hypervisor software is expected to manually swap the contents of these registers as
needed.

       Matching VS CSRs exist only for the supervisor CSRs that must be duplicated, which are mainly
       those that get automatically written by traps or that impact instruction execution immediately
       after trap entry and/or right before SRET, when software alone is unable to swap a CSR at
       exactly the right moment. Currently, most supervisor CSRs fall into this category, but future
       ones might not.

In this chapter, we use the term HSXLEN to refer to the effective XLEN when executing in HSmode, and VSXLEN to refer to the effective XLEN when executing in VS-mode.

##### 7.2.1 Hypervisor Status Register (hstatus)

The hstatus register is an HSXLEN-bit read/write register formatted as shown in Figure 7.1
when HSXLEN=32 and Figure 7.2 when HSXLEN=64. The hstatus register provides facilities
analogous to the mstatus register for tracking and controlling the exception behavior of a VS-mode
guest.

```text
  31        23    22       21      20     19 18       17            12        11 10   9       8            7            6        5   4    0
    WPRI         VTSR VTW VTVM WPRI                   VGEIN[5:0]          WPRI HU SPVP SPV GVA VSBE WPRI
     9             1   1    1   2                        6                 2    1   1   1   1    1   5
```

**Figure 7.1:** Hypervisor status register (hstatus) for RV32.

```text
  HSXLEN-1                                            34       33        32      31                   23           22       21       20
                         WPRI                              VSXL[1:0]                  WPRI                 VTSR VTW VTVM
                       HSXLEN-34                              2                        9                     1   1    1
                 19 18     17            12   11 10        9         8           7        6       5            4             0
                 WPRI       VGEIN[5:0]        WPRI HU SPVP SPV GVA VSBE                                            WPRI
                  2            6               2    1   1   1   1    1                                              5
```

**Figure 7.2:** Hypervisor status register (hstatus) for RV64.

The VSXL field controls the effective XLEN for VS-mode (known as VSXLEN), which may differ
from the XLEN for HS-mode (HSXLEN). When HSXLEN=32, the VSXL field does not exist, and
VSXLEN=32. When HSXLEN=64, VSXL is a WARL field that is encoded the same as the MXL
field of misa, shown in Table 3.1 on page 16. In particular, an implementation may make VSXL
be a read-only field whose value always ensures that VSXLEN=HSXLEN.

If HSXLEN is changed from 32 to a wider width, and if field VSXL is not restricted to a single value,
it gets the value corresponding to the widest supported width not wider than the new HSXLEN.

<!-- p.251 -->

The hstatus fields VTSR, VTW, and VTVM are defined analogously to the mstatus fields TSR,
TW, and TVM, but affect execution only in VS-mode, and cause virtual instruction exceptions
instead of illegal instruction exceptions. When VTSR=1, an attempt in VS-mode to execute SRET
raises a virtual instruction exception. When VTW=1 (and assuming mstatus.TW=0), an attempt
in VS-mode to execute WFI raises a virtual instruction exception if the WFI does not complete
within an implementation-specific, bounded time limit. When VTVM=1, an attempt in VS-mode
to execute SFENCE.VMA or to access CSR satp raises a virtual instruction exception.

The VGEIN (Virtual Guest External Interrupt Number) field selects a guest external interrupt
source for VS-level external interrupts. VGEIN is a WLRL field that must be able to hold values
between zero and the maximum guest external interrupt number (known as GEILEN), inclusive.
When VGEIN=0, no guest external interrupt source is selected for VS-level external interrupts.
GEILEN may be zero, in which case VGEIN may be hardwired to zero. Guest external interrupts
are explained in Section 7.2.4, and the use of VGEIN is covered further in Section 7.2.3.

Field HU (Hypervisor User mode) controls whether the virtual-machine load/store instructions,
HLV, HLVX, and HSV, can be used also in U-mode. When HU=1, these instructions can be
executed in U-mode the same as in HS-mode. When HU=0, all hypervisor instructions cause an
illegal instruction trap in U-mode.

     The HU bit allows a portion of a hypervisor to be run in U-mode for greater protection against
     software bugs, while still retaining access to a virtual machine’s memory.

The SPV bit (Supervisor Previous Virtualization mode) is written by the implementation whenever
a trap is taken into HS-mode. Just as the SPP bit in sstatus is set to the privilege mode at the
time of the trap, the SPV bit in hstatus is set to the value of the virtualization mode V at the
time of the trap. When an SRET instruction is executed when V=0, V is set to SPV.

When V=1 and a trap is taken into HS-mode, bit SPVP (Supervisor Previous Virtual Privilege)
is set to the privilege mode at the time of the trap, the same as sstatus.SPP. But if V=0 before
a trap, SPVP is left unchanged on trap entry. SPVP controls the effective privilege of explicit
memory accesses made by the virtual-machine load/store instructions, HLV, HLVX, and HSV.

     Without SPVP, if instructions HLV, HLVX, and HSV looked instead to sstatus.SPP for the
     effective privilege of their memory accesses, then, even with HU=1, U-mode could not access
     virtual machine memory at VS-level, because to enter U-mode using SRET always leaves SPP=0.
     Unlike SPP, field SPVP is untouched by transitions back-and-forth between HS-mode and U-
     mode.

Field GVA (Guest Virtual Address) is written by the implementation whenever a trap is taken into
HS-mode. For any trap (access fault, page fault, or guest-page fault) that writes a guest virtual
address to stval, GVA is set to 1. For any other trap into HS-mode, GVA is set to 0.

     For memory faults, GVA is redundant with field SPV (the two bits are set the same) except when
     the explicit memory access of an HLV, HLVX, or HSV instruction causes a fault. In that case,
     SPV=0 but GVA=1.

The VSBE bit is a WARL field that controls the endianness of explicit memory accesses made
from VS-mode. If VSBE=0, explicit load and store memory accesses made from VS-mode are littleendian, and if VSBE=1, they are big-endian. VSBE also controls the endianness of all implicit
accesses to VS-level memory management data structures, such as page tables. An implementation
may make VSBE a read-only field that always specifies the same endianness as HS-mode.

<!-- p.252 -->

##### 7.2.2 Hypervisor Trap Delegation Registers (hedeleg and hideleg)

Registers hedeleg and hideleg are HSXLEN-bit read/write registers, formatted as shown in Figures 7.3 and 7.4 respectively. By default, all traps at any privilege level are handled in M-mode,
though M-mode usually uses the medeleg and mideleg CSRs to delegate some traps to HS-mode.
The hedeleg and hideleg CSRs allow these traps to be further delegated to a VS-mode guest;
their layout is the same as medeleg and mideleg.
               HSXLEN-1                                                            0
                                   Synchronous Exceptions (WARL)
                                             HSXLEN

**Figure 7.3:** Hypervisor exception delegation register (hedeleg).

               HSXLEN-1                                                            0
                                         Interrupts (WARL)
                                              HSXLEN

**Figure 7.4:** Hypervisor interrupt delegation register (hideleg).

```text
                Bit   Attribute        Corresponding Exception
                  0   (See text)       Instruction address misaligned
                  1   Writable         Instruction access fault
                  2   Writable         Illegal instruction
                  3   Writable         Breakpoint
                  4   Writable         Load address misaligned
                  5   Writable         Load access fault
                  6   Writable         Store/AMO address misaligned
                  7   Writable         Store/AMO access fault
                  8   Writable         Environment call from U-mode or VU-mode
                  9   Read-only    0   Environment call from HS-mode
                 11   Read-only    0   Environment call from M-mode
                 12   Writable         Instruction page fault
                 13   Writable         Load page fault
                 15   Writable         Store/AMO page fault
                 20   Read-only    0   Instruction guest-page fault
                 21   Read-only    0   Load guest-page fault
                 22   Read-only    0   Virtual instruction
                 23   Read-only    0   Store/AMO guest-page fault
```

**Table 7.2:** Bits of hedeleg that must be writable or must be hardwired to zero.

A synchronous trap that has been delegated to HS-mode (using medeleg) is further delegated to
VS-mode if V=1 before the trap and the corresponding hedeleg bit is set. Each bit of hedeleg shall be either writable or hardwired to zero. Many bits of hedeleg are required specifically to be
writable or zero, as enumerated in Table 7.2. Bit 0, corresponding to instruction address misaligned
exceptions, must be writable if IALIGN=32.

<!-- p.253 -->

     Requiring that certain bits of hedeleg be writable reduces some of the burden on a hypervisor
     to handle variations of implementation.

An interrupt that has been delegated to HS-mode (using mideleg) is further delegated to VSmode if the corresponding hideleg bit is set. Among bits 15:0 of hideleg, only bits 10, 6, and
2 (corresponding to the standard VS-level interrupts) shall be writable, and the others shall be
hardwired to zero.

When a virtual supervisor external interrupt (code 10) is delegated to VS-mode, it is automatically
translated by the machine into a supervisor external interrupt (code 9) for VS-mode, including the
value written to vscause on an interrupt trap. Likewise, a virtual supervisor timer interrupt (6)
is translated into a supervisor timer interrupt (5) for VS-mode, and a virtual supervisor software
interrupt (2) is translated into a supervisor software interrupt (1) for VS-mode. Similar translations
may or may not be done for platform or custom interrupt causes (codes 16 and above).

##### 7.2.3 Hypervisor Interrupt Registers (hvip, hip, and hie)

Register hvip is an HSXLEN-bit read/write register that a hypervisor can write to indicate virtual
interrupts intended for VS-mode. The bit positions writable in hideleg shall also be writable in
hvip, and the other bits of hvip shall be hardwired to zeros.

               HSXLEN-1                                                                          0
                                         Virtual Interrupts (WARL)
                                                  HSXLEN

**Figure 7.5:** Hypervisor virtual-interrupt-pending register (hvip).

The standard portion (bits 15:0) of hvip is formatted as shown in Figure 7.6. Setting VSEIP=1 in
hvip asserts a VS-level external interrupt; setting VSTIP asserts a VS-level timer interrupt; and
setting VSSIP asserts a VS-level software interrupt.
                  15          11    10       9       7     6     5       3     2     1       0
                        0          VSEIP         0       VSTIP       0       VSSIP       0
                        5            1           3         1         3         1         2

**Figure 7.6:** Standard portion (bits 15:0) of hvip.

Registers hip and hie are HSXLEN-bit read/write registers that supplement HS-level’s sip and
sie respectively. The hip register indicates pending VS-level and hypervisor-specific interrupts,
while hie contains enable bits for the same interrupts. As with sip and sie, an interrupt i will be
taken in HS-mode if bit i is set in both hip and hie, and if supervisor-level interrupts are globally
enabled.

<!-- p.254 -->

                HSXLEN-1                                                                            0
                                                 Interrupts (WARL)
                                                      HSXLEN

**Figure 7.7:** Hypervisor interrupt-pending register (hip). HSXLEN-1                                                                            0 Interrupts (WARL) HSXLEN

**Figure 7.8:** Hypervisor interrupt-enable register (hie).

For each writable bit in sie, the corresponding bit shall be hardwired to zero in both hip and hie.
Hence, the nonzero bits in sie and hie are always mutually exclusive, and likewise for sip and
hip.

      The active bits of hip and hie cannot be placed in HS-level’s sip and sie because doing so
      would make it impossible for software to emulate the hypervisor extension on platforms that do
      not implement it in hardware.

If bit i of sie is hardwired to zero, the same bit in register hip may be writable or may be read-only.
When bit i in hip is writable, a pending interrupt i can be cleared by writing 0 to this bit. If
interrupt i can become pending in hip but bit i in hip is read-only, then either the interrupt can
be cleared by clearing bit i of hvip, or the implementation must provide some other mechanism
for clearing the pending interrupt (which may involve a call to the execution environment).

A bit in hie shall be writable if the corresponding interrupt can ever become pending in hip. Bits
of hie that are not writable shall be hardwired to zero.

```text
The standard portions (bits 15:0) of registers hip and hie are formatted as shown in Figures 7.9
and 7.10 respectively.
                15       13     12    11    10      9       7     6     5       3     2     1           0
                     0        SGEIP   0    VSEIP        0       VSTIP       0       VSSIP       0
                     3          1     1      1          3         1         3         1         2
```

**Figure 7.9:** Standard portion (bits 15:0) of hip.

```text
                15       13     12    11    10      9       7     6     5       3     2     1           0
                     0        SGEIE   0    VSEIE        0       VSTIE       0       VSSIE       0
                     3          1     1      1          3         1         3         1         2
```

**Figure 7.10:** Standard portion (bits 15:0) of hie.

Bits hip.SGEIP and hie.SGEIE are the interrupt-pending and interrupt-enable bits for guest
external interrupts at supervisor level (HS-level). SGEIP is read-only in hip, and is 1 if and only
if the bitwise logical-AND of CSRs hgeip and hgeie is nonzero in any bit. (See Section 7.2.4.)

Bits hip.VSEIP and hie.VSEIE are the interrupt-pending and interrupt-enable bits for VS-level
external interrupts. VSEIP is read-only in hip, and is the logical-OR of these interrupt sources:

- bit VSEIP of hvip;

<!-- p.255 -->

- the bit of hgeip selected by hstatus.VGEIN; and
- any other platform-specific external interrupt signal directed to VS-level.

Bits hip.VSTIP and hie.VSTIE are the interrupt-pending and interrupt-enable bits for VS-level
timer interrupts. VSTIP is read-only in hip, and is the logical-OR of hvip.VSTIP and any other
platform-specific timer interrupt signal directed to VS-level.

Bits hip.VSSIP and hie.VSSIE are the interrupt-pending and interrupt-enable bits for VS-level
software interrupts. VSSIP in hip is an alias (writable) of the same bit in hvip.

Multiple simultaneous interrupts destined for HS-mode are handled in the following decreasing
priority order: SEI, SSI, STI, SGEI, VSEI, VSSI, VSTI.

##### 7.2.4 Hypervisor Guest External Interrupt Registers (hgeip and hgeie)

The hgeip register is an HSXLEN-bit read-only register, formatted as shown in Figure 7.11, that
indicates pending guest external interrupts for this hart. The hgeie register is an HSXLEN-bit
read/write register, formatted as shown in Figure 7.12, that contains enable bits for the guest
external interrupts at this hart. Guest external interrupt number i corresponds with bit i in both
hgeip and hgeie.

```text
               HSXLEN-1                                                                 1   0
                                        Guest External Interrupts                           0
                                              HSXLEN-1                                      1
```

**Figure 7.11:** Hypervisor guest external interrupt-pending register (hgeip).

```text
               HSXLEN-1                                                                 1   0
                                   Guest External Interrupts (WARL)                         0
                                              HSXLEN-1                                      1
```

**Figure 7.12:** Hypervisor guest external interrupt-enable register (hgeie).

Guest external interrupts represent interrupts directed to individual virtual machines at VS-level.
If a RISC-V platform supports placing a physical device under the direct control of a guest OS with
minimal hypervisor intervention (known as pass-through or direct assignment between a virtual machine and the physical device), then, in such circumstance, interrupts from the device are intended
for a specific virtual machine. Each bit of hgeip summarizes all pending interrupts directed to one
virtual hart, as collected and reported by an interrupt controller. To distinguish specific pending
interrupts from multiple devices, software must query the interrupt controller.

       Support for guest external interrupts requires an interrupt controller that can collect virtual-
       machine-directed interrupts separately from other interrupts.

The number of bits implemented in hgeip and hgeie for guest external interrupts is unspecified
and may be zero. This number is known as GEILEN. The least-significant bits are implemented
first, apart from bit 0. Hence, if GEILEN is nonzero, bits GEILEN:1 shall be writable in hgeie,
and all other bit positions shall be hardwired to zeros in both hgeip and hgeie.

<!-- p.256 -->

      The set of guest external interrupts received and handled at one physical hart may differ from
      those received at other harts. Guest external interrupt number i at one physical hart is typically
      expected not to be the same as guest external interrupt i at any other hart. For any one physical
      hart, the maximum number of virtual harts that may directly receive guest external interrupts is
      limited by GEILEN. The maximum this number can be for any implementation is 31 for RV32
      and 63 for RV64, per physical hart.
          A hypervisor is always free to emulate devices for any number of virtual harts without being
      limited by GEILEN. Only direct pass-through (direct assignment) of interrupts is affected by the
      GEILEN limit, and the limit is on the number of virtual harts receiving such interrupts, not the
      number of distinct interrupts received. The number of distinct interrupts a single virtual hart
      may receive is determined by the interrupt controller.

Register hgeie selects the subset of guest external interrupts that cause a supervisor-level (HS-level)
guest external interrupt. The enable bits in hgeie do not affect the VS-level external interrupt
signal selected from hgeip by hstatus.VGEIN.

##### 7.2.5 Hypervisor Counter-Enable Register (hcounteren)

```text
The counter-enable register hcounteren is a 32-bit register that controls the availability of the
hardware performance monitoring counters to the guest virtual machine.
     31       30       29      28                                 6     5       4       3      2    1      0
  HPM31 HPM30 HPM29                             ...                   HPM5 HPM4 HPM3 IR TM CY
    1     1     1                               23                      1    1    1   1  1  1
```

**Figure 7.13:** Hypervisor counter-enable register (hcounteren).

When the CY, TM, IR, or HPMn bit in the hcounteren register is clear, attempts to read the
cycle, time, instret, or hpmcountern register while V=1 will cause a virtual instruction exception
if the same bit in mcounteren is 1. When one of these bits is set, access to the corresponding register
is permitted when V=1, unless prevented for some other reason. In VU-mode, a counter is not
readable unless the applicable bits are set in both hcounteren and scounteren.

hcounteren must be implemented. However, any of the bits may contain a hardwired value of zero,
indicating reads to the corresponding counter will cause an exception when V=1. Hence, they are
effectively WARL fields.

##### 7.2.6 Hypervisor Time Delta Registers (htimedelta, htimedeltah)

The htimedelta CSR is a read/write register that contains the delta between the value of the time
CSR and the value returned in VS-mode or VU-mode. That is, reading the time CSR in VS or
VU mode returns the sum of the contents of htimedelta and the actual value of time.

      Because overflow is ignored when summing htimedelta and time, large values of htimedelta
      may be used to represent negative time offsets.

<!-- p.257 -->

               63                                                                       0
                                               htimedelta
                                                  64

**Figure 7.14:** Hypervisor time delta register, HSXLEN=64.

For HSXLEN=32 only, htimedelta holds the lower 32 bits of the delta, and htimedeltah holds
the upper 32 bits of the delta.
               31                                                                       0
                                               htimedelta
                                              htimedeltah
                                                  32

**Figure 7.15:** Hypervisor time delta registers, HSXLEN=32.

##### 7.2.7 Hypervisor Trap Value Register (htval)

The htval register is an HSXLEN-bit read/write register formatted as shown in Figure 7.16. When
a trap is taken into HS-mode, htval is written with additional exception-specific information,
alongside stval, to assist software in handling the trap.
               HSXLEN-1                                                                 0
                                                 htval
                                               HSXLEN

**Figure 7.16:** Hypervisor trap value register (htval).

When a guest-page-fault trap is taken into HS-mode, htval is written with either zero or the guest
physical address that faulted, shifted right by 2 bits. For other traps, htval is set to zero, but a
future standard or extension may redefine htval’s setting for other traps.

A guest-page fault may arise due to an implicit memory access during first-stage (VS-stage) address
translation, in which case a guest physical address written to htval is that of the implicit memory
access that faulted—for example, the address of a VS-level page table entry that could not be read.
(The guest physical address corresponding to the original virtual address is unknown when VS-stage
translation fails to complete.) Additional information is provided in CSR htinst to disambiguate
such situations.

Otherwise, for misaligned loads and stores that cause guest-page faults, a nonzero guest physical
address in htval corresponds to the faulting portion of the access as indicated by the virtual address
in stval. For instruction guest-page faults on systems with variable-length instructions, a nonzero
htval corresponds to the faulting portion of the instruction as indicated by the virtual address in
stval.

     A guest physical address written to htval is shifted right by 2 bits to accommodate addresses
     wider than the current XLEN. For RV32, the hypervisor extension permits guest physical ad-
     dresses as wide as 34 bits, and htval reports bits 33:2 of the address. This shift-by-2 encoding
     of guest physical addresses matches the encoding of physical addresses in PMP address registers
     (Section 3.7) and in page table entries (Sections 4.3, 4.4, 4.5, and 4.6).

<!-- p.258 -->

         If the least-significant two bits of a faulting guest physical address are needed, these bits
     are ordinarily the same as the least-significant two bits of the faulting virtual address in stval.
     For faults due to implicit memory accesses for VS-stage address translation, the least-significant
     two bits are instead zeros. These cases can be distinguished using the value provided in register
     htinst.

htval is a WARL register that must be able to hold zero and may be capable of holding only an
arbitrary subset of other 2-bit-shifted guest physical addresses, if any.

     Unless it has reason to assume otherwise (such as a platform standard), software that writes a
     value to htval should read back from htval to confirm the stored value.

##### 7.2.8 Hypervisor Trap Instruction Register (htinst)

The htinst register is an HSXLEN-bit read/write register formatted as shown in Figure 7.17.
When a trap is taken into HS-mode, htinst is written with a value that, if nonzero, provides
information about the instruction that trapped, to assist software in handling the trap. The values
that may be written to htinst on a trap are documented in Section 7.6.3.
               HSXLEN-1                                                                    0
                                                  htinst
                                                 HSXLEN

**Figure 7.17:** Hypervisor trap instruction register (htinst).

htinst is a WARL register that need only be able to hold the values that the implementation
may automatically write to it on a trap.

##### 7.2.9 Hypervisor Guest Address Translation and Protection Register (hgatp)

The hgatp register is an HSXLEN-bit read/write register, formatted as shown in Figure 7.18 for
HSXLEN=32 and Figure 7.19 for HSXLEN=64, which controls G-stage address translation and
protection, the second stage of two-stage translation for guest virtual addresses (see Section 7.5).
Similar to CSR satp, this register holds the physical page number (PPN) of the guest-physical root
page table; a virtual machine identifier (VMID), which facilitates address-translation fences on a
per-virtual-machine basis; and the MODE field, which selects the address-translation scheme for
guest physical addresses. When mstatus.TVM=1, attempts to read or write hgatp while executing
in HS-mode will raise an illegal instruction exception.
         31     30        29 28                    22 21                                              0
        MODE   0 (WARL)           VMID (WARL)                          PPN (WARL)
          1        2                    7                                  22

**Figure 7.18:** RV32 Hypervisor guest address translation and protection register hgatp.

Table 7.3 shows the encodings of the MODE field for RV32 and RV64. When MODE=Bare,
guest physical addresses are equal to supervisor physical addresses, and there is no further memory

<!-- p.259 -->

```text
 63             60   59    58   57                 44 43                                      0
  MODE (WARL)        0 (WARL)        VMID (WARL)                     PPN (WARL)
      4                  2                14                             44
```

**Figure 7.19:** RV64 Hypervisor guest address translation and protection register hgatp, for MODE values Bare, Sv39x4, Sv48x4, and Sv57x4.

protection for a guest virtual machine beyond the physical memory protection scheme described in
Section 3.7. In this case, the remaining fields in hgatp must be set to zeros.

For RV32, the only other valid setting for MODE is Sv32x4, which is a modification of the usual
Sv32 paged virtual-memory scheme, extended to support 34-bit guest physical addresses. For RV64,
modes Sv39x4, Sv48x4, and Sv57x4 are defined as modifications of the Sv39, Sv48, and Sv57 paged
virtual-memory schemes. All of these paged virtual-memory schemes are described in Section 7.5.1.

The remaining MODE settings for RV64 are reserved for future use and may define different interpretations of the other fields in hgatp.

```text
                                              RV32
        Value     Name    Description
          0       Bare    No translation or protection.
          1      Sv32x4   Page-based 34-bit virtual addressing   (2-bit extension of Sv32).
                                             RV64
        Value     Name    Description
          0       Bare    No translation or protection.
         1–7       —      Reserved
          8      Sv39x4   Page-based 41-bit virtual addressing   (2-bit extension of Sv39).
          9      Sv48x4   Page-based 50-bit virtual addressing   (2-bit extension of Sv48).
          10     Sv57x4   Page-based 59-bit virtual addressing   (2-bit extension of Sv57).
        11–15      —      Reserved
```

**Table 7.3:** Encoding of hgatp MODE field.

RV64 implementations are not required to support all defined RV64 MODE settings.

A write to hgatp with an unsupported MODE value is not ignored as it is for satp. Instead, the
fields of hgatp are WARL in the normal way, when so indicated.

As explained in Section 7.5.1, for the paged virtual-memory schemes (Sv32x4, Sv39x4, Sv48x4,
and Sv57x4), the root page table is 16 KiB and must be aligned to a 16-KiB boundary. In these
modes, the lowest two bits of the physical page number (PPN) in hgatp always read as zeros. An
implementation that supports only the defined paged virtual-memory schemes and/or Bare may
hardwire PPN[1:0] to zero.

The number of VMID bits is unspecified and may be zero. The number of implemented VMID
bits, termed VMIDLEN, may be determined by writing one to every bit position in the VMID field,
then reading back the value in hgatp to see which bit positions in the VMID field hold a one. The
least-significant bits of VMID are implemented first: that is, if VMIDLEN > 0, VMID[VMIDLEN1:0] is writable. The maximal value of VMIDLEN, termed VMIDMAX, is 7 for Sv32x4 or 14 for
Sv39x4, Sv48x4, and Sv57x4.

<!-- p.260 -->

The hgatp register is considered active for the purposes of the address-translation algorithm when
the effective privilege mode is VS-mode or VU-mode. The hgatp register is also considered briefly
active when executing a virtual-machine load/store instruction (HLV, HLVX, or HSV), and hence
the address-translation algorithm may be executed speculatively for any guest virtual address when
such an instruction is executed.

      Defining there to be a brief window during which hgatp is active while executing an HLV, HLVX,
      or HSV instruction allows the implementation to prefetch adjacent PTEs in the same cache line,
      or to prefetch translations predicted to be needed by future HLV, HLVX, or HSV instructions,
      for example.

Note that writing hgatp does not imply any ordering constraints between page-table updates and
subsequent G-stage address translations. If the new virtual machine’s guest physical page tables
have been modified, it may be necessary to execute an HFENCE.GVMA instruction (see Section 7.3.2) before or after writing hgatp.

##### 7.2.10 Virtual Supervisor Status Register (vsstatus)

The vsstatus register is a VSXLEN-bit read/write register that is VS-mode’s version of supervisor register sstatus, formatted as shown in Figure 7.20 when VSXLEN=32 and Figure 7.21
when VSXLEN=64. When V=1, vsstatus substitutes for the usual sstatus, so instructions that
normally read or modify sstatus actually access vsstatus instead.

```text
 31   30 20    19       18       17        16 15   14 13     12   9        8            7       6        5        4       2    1        0
 SD WPRI MXR SUM WPRI XS[1:0] FS[1:0] WPRI SPP WPRI UBE SPIE WPRI SIE WPRI
  1  11   1   1   1     2       2      4    1   1    1    1   3    1   1
```

**Figure 7.20:** Virtual supervisor status register (vsstatus) for RV32.

```text
  VSXLEN-1    VSXLEN-2                                  34   33       32       31                   20       19           18       17
      SD                       WPRI                          UXL[1:0]                   WPRI             MXR SUM WPRI
       1                     VSXLEN-35                         2                         12               1   1   1
                16 15        14 13    12       9    8        7         6            5       4   2        1            0
               XS[1:0] FS[1:0]        WPRI         SPP WPRI UBE SPIE WPRI SIE WPRI
                 2       2             4            1   1    1    1   3    1   1
```

**Figure 7.21:** Virtual supervisor status register (vsstatus) for RV64.

The UXL field controls the effective XLEN for VU-mode, which may differ from the XLEN for VSmode (VSXLEN). When VSXLEN=32, the UXL field does not exist, and VU-mode XLEN=32.
When VSXLEN=64, UXL is a WARL field that is encoded the same as the MXL field of misa,
shown in Table 3.1 on page 16. In particular, an implementation may make UXL be a read-only
copy of field VSXL of hstatus, forcing VU-mode XLEN=VSXLEN.

<!-- p.261 -->

If VSXLEN is changed from 32 to a wider width, and if field UXL is not restricted to a single value,
it gets the value corresponding to the widest supported width not wider than the new VSXLEN.

When V=1, both vsstatus.FS and the HS-level sstatus.FS are in effect. Attempts to execute a
floating-point instruction when either field is 0 (Off) raise an illegal-instruction exception. Modifying the floating-point state when V=1 causes both fields to be set to 3 (Dirty).

      For a hypervisor to benefit from the extension context status, it must have its own copy in
      the HS-level sstatus, maintained independently of a guest OS running in VS-mode. While
      a version of the extension context status obviously must exist in vsstatus for VS-mode, a
      hypervisor cannot rely on this version being maintained correctly, given that VS-level software
      can change vsstatus.FS arbitrarily. If the HS-level sstatus.FS were not independently active
      and maintained by the hardware in parallel with vsstatus.FS while V=1, hypervisors would
      always be forced to conservatively swap all floating-point state when context-switching between
      virtual machines.

Read-only fields SD and XS summarize the extension context status as it is visible to VS-mode
only. For example, the value of the HS-level sstatus.FS does not affect vsstatus.SD.

An implementation may make field UBE be a read-only copy of hstatus.VSBE.

When V=0, vsstatus does not directly affect the behavior of the machine, unless a virtual-machine
load/store (HLV, HLVX, or HSV) or the MPRV feature in the mstatus register is used to execute
a load or store as though V=1.

##### 7.2.11 Virtual Supervisor Interrupt Registers (vsip and vsie)

The vsip and vsie registers are VSXLEN-bit read/write registers that are VS-mode’s versions of
supervisor CSRs sip and sie, formatted as shown in Figures 7.22 and 7.23 respectively. When
V=1, vsip and vsie substitute for the usual sip and sie, so instructions that normally read
or modify sip/sie actually access vsip/vsie instead. However, interrupts directed to HS-level
continue to be indicated in the HS-level sip register, not in vsip, when V=1.

                VSXLEN-1                                                                 0
                                           Interrupts (WARL)
                                                VSXLEN

**Figure 7.22:** Virtual supervisor interrupt-pending register (vsip).

                VSXLEN-1                                                                 0
                                           Interrupts (WARL)
                                                VSXLEN

**Figure 7.23:** Virtual supervisor interrupt-enable register (vsie).

The standard portions (bits 15:0) of registers vsip and vsie are formatted as shown in Figures
7.24 and 7.25 respectively.

When bit 10 of hideleg is zero, vsip.SEIP and vsie.SEIE are read-only zeros. Else, vsip.SEIP
and vsie.SEIE are aliases of hip.VSEIP and hie.VSEIE.

<!-- p.262 -->

```text
                      15              10    9     8       6    5     4       2    1        0
                               0           SEIP       0       STIP       0       SSIP 0
                               6             1        3         1        3         1  1
```

**Figure 7.24:** Standard portion (bits 15:0) of vsip.

```text
                      15              10    9     8       6    5     4       2    1        0
                               0           SEIE       0       STIE       0       SSIE 0
                               6             1        3         1        3         1  1
```

**Figure 7.25:** Standard portion (bits 15:0) of vsie.

When bit 6 of hideleg is zero, vsip.STIP and vsie.STIE are read-only zeros. Else, vsip.STIP
and vsie.STIE are aliases of hip.VSTIP and hie.VSTIE.

When bit 2 of hideleg is zero, vsip.SSIP and vsie.SSIE are read-only zeros. Else, vsip.SSIP and
vsie.SSIE are aliases of hip.VSSIP and hie.VSSIE.

##### 7.2.12 Virtual Supervisor Trap Vector Base Address Register (vstvec)

The vstvec register is a VSXLEN-bit read/write register that is VS-mode’s version of supervisor
register stvec, formatted as shown in Figure 7.26. When V=1, vstvec substitutes for the usual
stvec, so instructions that normally read or modify stvec actually access vstvec instead. When
V=0, vstvec does not directly affect the behavior of the machine.
         VSXLEN-1                                                                     21           0
                              BASE[VSXLEN-1:2] (WARL)                                  MODE (WARL)
                                     VSXLEN-2                                             2

**Figure 7.26:** Virtual supervisor trap vector base address register (vstvec).

##### 7.2.13 Virtual Supervisor Scratch Register (vsscratch)

The vsscratch register is a VSXLEN-bit read/write register that is VS-mode’s version of supervisor
register sscratch, formatted as shown in Figure 7.27. When V=1, vsscratch substitutes for the
usual sscratch, so instructions that normally read or modify sscratch actually access vsscratch
instead. The contents of vsscratch never directly affect the behavior of the machine.
               VSXLEN-1                                                                        0
                                                  vsscratch
                                                  VSXLEN

**Figure 7.27:** Virtual supervisor scratch register (vsscratch).

<!-- p.263 -->

##### 7.2.14 Virtual Supervisor Exception Program Counter (vsepc)

The vsepc register is a VSXLEN-bit read/write register that is VS-mode’s version of supervisor
register sepc, formatted as shown in Figure 7.28. When V=1, vsepc substitutes for the usual
sepc, so instructions that normally read or modify sepc actually access vsepc instead. When
V=0, vsepc does not directly affect the behavior of the machine.

vsepc is a WARL register that must be able to hold the same set of values that sepc can hold.

              VSXLEN-1                                                                0
                                             vsepc
                                           VSXLEN

**Figure 7.28:** Virtual supervisor exception program counter (vsepc).

##### 7.2.15 Virtual Supervisor Cause Register (vscause)

The vscause register is a VSXLEN-bit read/write register that is VS-mode’s version of supervisor
register scause, formatted as shown in Figure 7.29. When V=1, vscause substitutes for the usual
scause, so instructions that normally read or modify scause actually access vscause instead.
When V=0, vscause does not directly affect the behavior of the machine.

```text
vscause is a WLRL register that must be able to hold the same set of values that scause can
hold.
                 VSXLEN-1 VSXLEN-2                                                0
                 Interrupt                Exception Code (WLRL)
                      1                         VSXLEN-1
```

**Figure 7.29:** Virtual supervisor cause register (vscause).

##### 7.2.16 Virtual Supervisor Trap Value Register (vstval)

The vstval register is a VSXLEN-bit read/write register that is VS-mode’s version of supervisor
register stval, formatted as shown in Figure 7.30. When V=1, vstval substitutes for the usual
stval, so instructions that normally read or modify stval actually access vstval instead. When
V=0, vstval does not directly affect the behavior of the machine.

vstval is a WARL register that must be able to hold the same set of values that stval can hold.
              VSXLEN-1                                                                0
                                            vstval
                                           VSXLEN

**Figure 7.30:** Virtual supervisor trap value register (vstval).

<!-- p.264 -->

##### 7.2.17 Virtual Supervisor Address Translation and Protection Register (vsatp)

The vsatp register is a VSXLEN-bit read/write register that is VS-mode’s version of supervisor
register satp, formatted as shown in Figure 7.31 for VSXLEN=32 and Figure 7.32 for VSXLEN=64.
When V=1, vsatp substitutes for the usual satp, so instructions that normally read or modify
satp actually access vsatp instead. vsatp controls VS-stage address translation, the first stage of
two-stage translation for guest virtual addresses (see Section 7.5).

```text
                31        30                  22 21                                         0
         MODE (WARL)           ASID (WARL)                      PPN (WARL)
               1                     9                              22
```

**Figure 7.31:** RV32 virtual supervisor address translation and protection register vsatp.

```text
63             60    59                      44 43                                                       0
 MODE (WARL)              ASID (WARL)                               PPN (WARL)
     4                          16                                      44
```

**Figure 7.32:** RV64 virtual supervisor address translation and protection register vsatp, for MODE values Bare, Sv39, Sv48, and Sv57.

The vsatp register is considered active for the purposes of the address-translation algorithm when
the effective privilege mode is VS-mode or VU-mode. The vsatp register is also considered briefly
active when executing a virtual-machine load/store instruction (HLV, HLVX, or HSV), and hence
the address-translation algorithm may be executed speculatively for any guest virtual address when
such an instruction is executed.

      Just as with hgatp, defining there to be a brief window during which vsatp is active while
      executing an HLV, HLVX, or HSV instruction allows the implementation to prefetch PTEs
      associated with the vsatp register.

When V=0, a write to vsatp with an unsupported MODE value is not ignored as it is for satp.
Instead, the fields of vsatp are WARL in the normal way.

When V=0, vsatp does not directly affect the behavior of the machine, unless a virtual-machine
load/store (HLV, HLVX, or HSV) or the MPRV feature in the mstatus register is used to execute
a load or store as though V=1.

#### 7.3 Hypervisor Instructions

The hypervisor extension adds virtual-machine load and store instructions and two privileged fence
instructions.

<!-- p.265 -->

##### 7.3.1 Hypervisor Virtual-Machine Load and Store Instructions

```text
        31              25 24          20 19           15 14   12 11          7 6             0
            funct7               rs2             rs1      funct3        rd           opcode
              7                   5               5         3            5             7
           HLV.width             [U]            addr     PRIVM         dest         SYSTEM
         HLVX.HU/WU             HLVX            addr     PRIVM         dest         SYSTEM
           HSV.width             src            addr     PRIVM           0          SYSTEM
```

The hypervisor virtual-machine load and store instructions are valid only in M-mode or HS-mode,
or in U-mode when hstatus.HU=1. Each instruction performs an explicit memory access as though
V=1; i.e., with the address translation and protection, and the endianness, that apply to memory
accesses in either VS-mode or VU-mode. Field SPVP of hstatus controls the privilege level of the
access. The explicit memory access is done as though in VU-mode when SPVP=0, and as though
in VS-mode when SPVP=1. As usual when V=1, two-stage address translation is applied, and the
HS-level sstatus.SUM is ignored. HS-level sstatus.MXR makes execute-only pages readable for
both stages of address translation (VS-stage and G-stage), whereas vsstatus.MXR affects only
the first translation stage (VS-stage).

For every RV32I or RV64I load instruction, LB, LBU, LH, LHU, LW, LWU, and LD, there is
a corresponding virtual-machine load instruction: HLV.B, HLV.BU, HLV.H, HLV.HU, HLV.W,
HLV.WU, and HLV.D. For every RV32I or RV64I store instruction, SB, SH, SW, and SD, there is a
corresponding virtual-machine store instruction: HSV.B, HSV.H, HSV.W, and HSV.D. Instructions
HLV.WU, HLV.D, and HSV.D are not valid for RV32, of course.

Instructions HLVX.HU and HLVX.WU are the same as HLV.HU and HLV.WU, except that execute
permission takes the place of read permission during address translation. That is, the memory being
read must be executable in both stages of address translation, but read permission is not required.
HLVX.WU is valid for RV32, even though LWU and HLV.WU are not. (For RV32, HLVX.WU can
be considered a variant of HLV.W, as sign extension is irrelevant for 32-bit values.)

The hgatp and vsatp registers are considered active for the purposes of the address-translation
algorithm when executing virtual-machine load/store instructions (HLV, HLVX, or HSV).

HLVX cannot override machine-level physical memory protection (PMP), so attempting to read
memory that PMP designates as execute-only still results in an access-fault exception.

Attempts to execute a virtual-machine load/store instruction (HLV, HLVX, or HSV) when V=1
cause a virtual instruction trap. Attempts to execute one of these same instructions from U-mode
when hstatus.HU=0 cause an illegal instruction trap.

##### 7.3.2 Hypervisor Memory-Management Fence Instructions

```text
        31              25 24          20 19           15 14   12 11          7 6             0
            funct7               rs2            rs1       funct3       rd            opcode
              7                   5              5          3           5              7
        HFENCE.VVMA             asid           vaddr      PRIV          0           SYSTEM
        HFENCE.GVMA             vmid           gaddr      PRIV          0           SYSTEM
```

<!-- p.266 -->

The hypervisor memory-management fence instructions, HFENCE.VVMA and HFENCE.GVMA,
perform a function similar to SFENCE.VMA (Section 4.2.1), except applying to the VS-level
memory-management data structures controlled by CSR vsatp (HFENCE.VVMA) or the guestphysical memory-management data structures controlled by CSR hgatp (HFENCE.GVMA). Instruction SFENCE.VMA applies only to the memory-management data structures controlled by
the current satp (either the HS-level satp when V=0 or vsatp when V=1).

HFENCE.VVMA is valid only in M-mode or HS-mode. Its effect is much the same as temporarily
entering VS-mode and executing SFENCE.VMA. Executing an HFENCE.VVMA guarantees that
any previous stores already visible to the current hart are ordered before all subsequent implicit
reads by that hart of the VS-level memory-management data structures, when those implicit reads
are for instructions that

- are subsequent to the HFENCE.VVMA, and
- execute when hgatp.VMID has the same setting as it did when HFENCE.VVMA executed.

Implicit reads need not be ordered when hgatp.VMID is different than at the time HFENCE.VVMA
executed. If operand rs1̸=x0, it specifies a single guest virtual address, and if operand rs2̸=x0, it
specifies a single guest address-space identifier (ASID).

     An HFENCE.VVMA instruction applies only to a single virtual machine, identified by the setting
     of hgatp.VMID when HFENCE.VVMA executes.

When rs2̸=x0, bits XLEN-1:ASIDMAX of the value held in rs2 are reserved for future use and
should be zeroed by software and ignored by current implementations. Furthermore, if ASIDLEN < ASIDMAX, the implementation shall ignore bits ASIDMAX-1:ASIDLEN of the value
held in rs2.

     Simpler implementations of HFENCE.VVMA can ignore the guest virtual address in rs1 and
     the guest ASID value in rs2, as well as hgatp.VMID, and always perform a global fence for the
     VS-level memory management of all virtual machines, or even a global fence for all memory-
     management data structures.

Neither mstatus.TVM nor hstatus.VTVM causes HFENCE.VVMA to trap.

HFENCE.GVMA is valid only in HS-mode when mstatus.TVM=0, or in M-mode (irrespective of mstatus.TVM). Executing an HFENCE.GVMA instruction guarantees that any previous
stores already visible to the current hart are ordered before all subsequent implicit reads by that
hart of guest-physical memory-management data structures done for instructions that follow the
HFENCE.GVMA. If operand rs1̸=x0, it specifies a single guest physical address, shifted right by
2 bits, and if operand rs2̸=x0, it specifies a single virtual machine identifier (VMID).

     Like for a guest physical address written to htval on a trap, a guest physical address specified
     in rs1 is shifted right by 2 bits to accommodate addresses wider than the current XLEN.

When rs2̸=x0, bits XLEN-1:VMIDMAX of the value held in rs2 are reserved for future use and
should be zeroed by software and ignored by current implementations. Furthermore, if VMIDLEN < VMIDMAX, the implementation shall ignore bits VMIDMAX-1:VMIDLEN of the value
held in rs2.

<!-- p.267 -->

      Simpler implementations of HFENCE.GVMA can ignore the guest physical address in rs1 and
      the VMID value in rs2 and always perform a global fence for the guest-physical memory manage-
      ment of all virtual machines, or even a global fence for all memory-management data structures.

If hgatp.MODE is changed for a given VMID, an HFENCE.GVMA with rs1=x0 (and rs2 set to
either x0 or the VMID) must be executed to order subsequent guest translations with the MODE
change—even if the old MODE or new MODE is Bare.

Attempts to execute HFENCE.VVMA or HFENCE.GVMA when V=1 cause a virtual instruction
trap, while attempts to do the same in U-mode cause an illegal instruction trap. Attempting to
execute HFENCE.GVMA in HS-mode when mstatus.TVM=1 also causes an illegal instruction
trap.

#### 7.4 Machine-Level CSRs

The hypervisor extension augments or modifies machine CSRs mstatus, mstatush, mideleg, mip,
and mie, and adds CSRs mtval2 and mtinst.

##### 7.4.1 Machine Status Registers (mstatus and mstatush)

The hypervisor extension adds two fields, MPV and GVA, to the machine-level mstatus or mstatush
CSR, and modifies the behavior of several existing mstatus fields. Figure 7.33 shows the modified mstatus register when the hypervisor extension is implemented and MXLEN=64. When
MXLEN=32, the hypervisor extension adds MPV and GVA not to mstatus but to mstatush,
which must exist. Figure 7.34 shows the mstatush register when the hypervisor extension is implemented and MXLEN=32.

```text
      MXLEN-1     MXLEN-2                              40    39        38        37        36        35       34   33    32
        SD                     WPRI                          MPV GVA MBE SBE                        SXL[1:0]       UXL[1:0]
         1                    MXLEN-41                        1   1   1   1                            2             2
             31                   23     22   21        20        19        18        17            16 15      14 13
                     WPRI              TSR TW TVM MXR SUM MPRV XS[1:0] FS[1:0]
                      9                 1   1  1   1   1    1    2       2
             12     11   10   9   8       7        6         5         4          3             2         1        0
             MPP[1:0] WPRI SPP MPIE UBE SPIE WPRI MIE WPRI SIE WPRI
               2       2    1   1    1    1   1    1   1    1   1
```

**Figure 7.33:** Machine status register (mstatus) for RV64 when the hypervisor extension is imple- mented.

The MPV bit (Machine Previous Virtualization Mode) is written by the implementation whenever
a trap is taken into M-mode. Just as the MPP bit is set to the privilege mode at the time of the
trap, the MPV bit is set to the value of the virtualization mode V at the time of the trap. When an

<!-- p.268 -->

```text
     31                                                     8    7     6      5     4    3      0
                              WPRI                              MPV GVA MBE SBE          WPRI
                               24                                1   1   1   1            4
```

**Figure 7.34:** Additional machine status register (mstatush) for RV32 when the hypervisor extension is implemented. The format of mstatus is unchanged for RV32.

MRET instruction is executed, the virtualization mode V is set to MPV, unless MPP=3, in which
case V remains 0.

Field GVA (Guest Virtual Address) is written by the implementation whenever a trap is taken into
M-mode. For any trap (access fault, page fault, or guest-page fault) that writes a guest virtual
address to mtval, GVA is set to 1. For any other trap into M-mode, GVA is set to 0.

The TSR and TVM fields of mstatus affect execution only in HS-mode, not in VS-mode. The TW
field affects execution in all modes except M-mode.

Setting TVM=1 prevents HS-mode from accessing hgatp or executing HFENCE.GVMA, but has
no effect on accesses to vsatp or instruction HFENCE.VVMA.

The hypervisor extension changes the behavior of the the Modify Privilege field, MPRV, of mstatus.
When MPRV=0, translation and protection behave as normal. When MPRV=1, explicit memory
accesses are translated and protected, and endianness is applied, as though the current virtualization
mode were set to MPV and the current privilege mode were set to MPP. Table 7.4 enumerates the
cases.

  MPRV      MPV     MPP      Effect
   0         –       –       Normal access; current privilege and virtualization modes apply.
   1         0       0       U-level access with HS-level translation and protection only.
   1         0       1       HS-level access with HS-level translation and protection only.
   1         –       3       M-level access with no translation.
   1         1       0       VU-level access with two-stage translation and protection. The HS-
                             level MXR bit makes any executable page readable. vsstatus.MXR
                             makes readable those pages marked executable at the VS translation
                             stage, but only if readable at the guest-physical translation stage.
     1        1       1      VS-level access with two-stage translation and protection. The HS-
                             level MXR bit makes any executable page readable. vsstatus.MXR
                             makes readable those pages marked executable at the VS translation
                             stage, but only if readable at the guest-physical translation stage.
                             vsstatus.SUM applies instead of the HS-level SUM bit.

**Table 7.4:** Effect of MPRV on the translation and protection of explicit memory accesses.

MPRV does not affect the virtual-machine load/store instructions, HLV, HLVX, and HSV. The
explicit loads and stores of these instructions always act as though V=1 and the privilege mode
were hstatus.SPVP, overriding MPRV.

The mstatus register is a superset of the HS-level sstatus register but is not a superset of vsstatus.

<!-- p.269 -->

##### 7.4.2 Machine Interrupt Delegation Register (mideleg)

When the hypervisor extension is implemented, bits 10, 6, and 2 of mideleg (corresponding to
the standard VS-level interrupts) are each hardwired to one. Furthermore, if any guest external
interrupts are implemented (GEILEN is nonzero), bit 12 of mideleg (corresponding to supervisorlevel guest external interrupts) is also hardwired to one. VS-level interrupts and guest external
interrupts are always delegated past M-mode to HS-mode.

##### 7.4.3 Machine Interrupt Registers (mip and mie)

The hypervisor extension gives registers mip and mie additional active bits for the hypervisor-added
interrupts. Figures 7.35 and 7.36 show the standard portions (bits 15:0) of registers mip and mie
when the hypervisor extension is implemented.
      15        13    12     11     10     9    8    7      6      5    4      3   2      1   0
            0        SGEIP MEIP VSEIP SEIP 0 MTIP VSTIP STIP 0 MSIP VSSIP SSIP 0
            3          1    1     1     1  1  1     1     1  1  1     1     1  1

**Figure 7.35:** Standard portion (bits 15:0) of mip.

```text
      15        13    12     11     10     9    8    7      6      5    4      3   2      1   0
            0        SGEIE MEIE VSEIE SEIE 0 MTIE VSTIE STIE 0 MSIE VSSIE SSIE 0
            3          1    1     1     1  1  1     1     1  1  1     1     1  1
```

**Figure 7.36:** Standard portion (bits 15:0) of mie.

Bits SGEIP, VSEIP, VSTIP, and VSSIP in mip are aliases for the same bits in hypervisor CSR
hip, while SGEIE, VSEIE, VSTIE, and VSSIE in mie are aliases for the same bits in hie.

##### 7.4.4 Machine Second Trap Value Register (mtval2)

The mtval2 register is an MXLEN-bit read/write register formatted as shown in Figure 7.37. When
a trap is taken into M-mode, mtval2 is written with additional exception-specific information,
alongside mtval, to assist software in handling the trap.
                 MXLEN-1                                                           0
                                                mtval2
                                               MXLEN

**Figure 7.37:** Machine second trap value register (mtval2).

When a guest-page-fault trap is taken into M-mode, mtval2 is written with either zero or the guest
physical address that faulted, shifted right by 2 bits. For other traps, mtval2 is set to zero, but a
future standard or extension may redefine mtval2’s setting for other traps.

If a guest-page fault is due to an implicit memory access during first-stage (VS-stage) address
translation, a guest physical address written to mtval2 is that of the implicit memory access that
faulted. Additional information is provided in CSR mtinst to disambiguate such situations.

<!-- p.270 -->

Otherwise, for misaligned loads and stores that cause guest-page faults, a nonzero guest physical
address in mtval2 corresponds to the faulting portion of the access as indicated by the virtual
address in mtval. For instruction guest-page faults on systems with variable-length instructions, a
nonzero mtval2 corresponds to the faulting portion of the instruction as indicated by the virtual
address in mtval.

mtval2 is a WARL register that must be able to hold zero and may be capable of holding only an
arbitrary subset of other 2-bit-shifted guest physical addresses, if any.

##### 7.4.5 Machine Trap Instruction Register (mtinst)

The mtinst register is an MXLEN-bit read/write register formatted as shown in Figure 7.38. When
a trap is taken into M-mode, mtinst is written with a value that, if nonzero, provides information
about the instruction that trapped, to assist software in handling the trap. The values that may
be written to mtinst on a trap are documented in Section 7.6.3.
               MXLEN-1                                                              0
                                               mtinst
                                              MXLEN

**Figure 7.38:** Machine trap instruction register (mtinst).

mtinst is a WARL register that need only be able to hold the values that the implementation
may automatically write to it on a trap.

#### 7.5 Two-Stage Address Translation

Whenever the current virtualization mode V is 1, two-stage address translation and protection
is in effect. For any virtual memory access, the original virtual address is converted in the first
stage by VS-level address translation, as controlled by the vsatp register, into a guest physical
address. The guest physical address is then converted in the second stage by guest physical address
translation, as controlled by the hgatp register, into a supervisor physical address. The two stages
are known also as VS-stage and G-stage translation. Although there is no option to disable twostage address translation when V=1, either stage of translation can be effectively disabled by zeroing
the corresponding vsatp or hgatp register.

The vsstatus field MXR, which makes execute-only pages readable, only overrides VS-stage page
protection. Setting MXR at VS-level does not override guest-physical page protections. Setting
MXR at HS-level, however, overrides both VS-stage and G-stage execute-only permissions.

When V=1, memory accesses that would normally bypass address translation are subject to Gstage address translation alone. This includes memory accesses made in support of VS-stage address
translation, such as reads and writes of VS-level page tables.

Machine-level physical memory protection applies to supervisor physical addresses and is in effect
regardless of virtualization mode.

<!-- p.271 -->

##### 7.5.1 Guest Physical Address Translation

The mapping of guest physical addresses to supervisor physical addresses is controlled by CSR
hgatp (Section 7.2.9).

When the address translation scheme selected by the MODE field of hgatp is Bare, guest physical
addresses are equal to supervisor physical addresses without modification, and no memory protection applies in the trivial translation of guest physical addresses to supervisor physical addresses.

When hgatp.MODE specifies a translation scheme of Sv32x4, Sv39x4, Sv48x4, or Sv57x4, G-stage
address translation is a variation on the usual page-based virtual address translation scheme of
Sv32, Sv39, Sv48, or Sv57, respectively. In each case, the size of the incoming address is widened
by 2 bits (to 34, 41, or 50 bits). To accommodate the 2 extra bits, the root page table (only) is
expanded by a factor of four to be 16 KiB instead of the usual 4 KiB. Matching its larger size,
the root page table also must be aligned to a 16 KiB boundary instead of the usual 4 KiB page
boundary. Except as noted, all other aspects of Sv32, Sv39, Sv48, or Sv57 are adopted unchanged
for G-stage translation. Non-root page tables and all page table entries (PTEs) have the same
formats as documented in Sections 4.3, 4.4, 4.5, and 4.6.

For Sv32x4, an incoming guest physical address is partitioned into a virtual page number (VPN)
and page offset as shown in Figure 7.39. This partitioning is identical to that for an Sv32 virtual
address as depicted in Figure 4.16 (page 74), except with 2 more bits at the high end in VPN[1].
(Note that the fields of a partitioned guest physical address also correspond one-for-one with the
structure that Sv32 assigns to a physical address, depicted in Figure 4.17.)

```text
                  33                  22 21                 12 11                       0
                          VPN[1]                 VPN[0]              page offset
                            12                     10                   12
```

**Figure 7.39:** Sv32x4 virtual address (guest physical address).

For Sv39x4, an incoming guest physical address is partitioned as shown in Figure 7.40. This
partitioning is identical to that for an Sv39 virtual address as depicted in Figure 4.19 (page 79),
except with 2 more bits at the high end in VPN[2]. Address bits 63:41 must all be zeros, or else a
guest-page-fault exception occurs.
         40                  30 29                  21 20              12 11                     0
                 VPN[2]               VPN[1]                VPN[0]                 page offset
                   11                   9                     9                       12

**Figure 7.40:** Sv39x4 virtual address (guest physical address).

For Sv48x4, an incoming guest physical address is partitioned as shown in Figure 7.41. This
partitioning is identical to that for an Sv48 virtual address as depicted in Figure 4.22 (page 81),
except with 2 more bits at the high end in VPN[3]. Address bits 63:50 must all be zeros, or else a
guest-page-fault exception occurs.

For Sv57x4, an incoming guest physical address is partitioned as shown in Figure 7.42. This
partitioning is identical to that for an Sv57 virtual address as depicted in Figure 4.25 (page 82),

<!-- p.272 -->

```text
49                        39 38                    30 29                 21 20                   12 11                    0
              VPN[3]                 VPN[2]                 VPN[1]                   VPN[0]                page offset
                11                     9                      9                        9                      12
```

**Figure 7.41:** Sv48x4 virtual address (guest physical address).

```text
except with 2 more bits at the high end in VPN[3]. Address bits 63:50 must all be zeros, or else a
guest-page-fault exception occurs.
         58              50 49             39 38            30 29            21 20            12 11                 0
                VPN[4]            VPN[3]           VPN[2]           VPN[1]           VPN[0]           page offset
                  11                9                9                9                9                 12
```

**Figure 7.42:** Sv57x4 virtual address (guest physical address).

         The page-based G-stage address translation scheme for RV32, Sv32x4, is defined to support
         a 34-bit guest physical address so that an RV32 hypervisor need not be limited in its ability to
         virtualize real 32-bit RISC-V machines, even those with 33-bit or 34-bit physical addresses. This
         may include the possibility of a machine virtualizing itself, if it happens to use 33-bit or 34-bit
         physical addresses. Multiplying the size and alignment of the root page table by a factor of four
         is the cheapest way to extend Sv32 to cover a 34-bit address. The possible wastage of 12 KiB for
         an unnecessarily large root page table is expected to be of negligible consequence for most (maybe
         all) real uses.
              A consistent ability to virtualize machines having as much as four times the physical address
         space as virtual address space is believed to be of some utility also for RV64. For a machine
         implementing 39-bit virtual addresses (Sv39), for example, this allows the hypervisor extension to
         support up to a 41-bit guest physical address space without either necessitating hardware support
         for 48-bit virtual addresses (Sv48) or falling back to emulating the larger address space using
         shadow page tables.

The conversion of an Sv32x4, Sv39x4, Sv48x4, or Sv57x4 guest physical address is accomplished
with the same algorithm used for Sv32, Sv39, Sv48, or Sv57, as presented in Section 4.3.2, except
that:

- hgatp substitutes for the usual satp;
- for the translation to begin, the effective privilege mode must be VS-mode or VU-mode;
- when checking the U bit, the current privilege mode is always taken to be U-mode; and
- guest-page-fault exceptions are raised instead of regular page-fault exceptions.

For G-stage address translation, all memory accesses (including those made to access data structures
for VS-stage address translation) are considered to be user-level accesses, as though executed in
U-mode. Access type permissions—readable, writable, or executable—are checked during G-stage
translation the same as for VS-stage translation. For a memory access made to support VS-stage
address translation (such as to read/write a VS-level page table), permissions are checked as though
for a load or store, not for the original access type. However, any exception is always reported for
the original access type (instruction, load, or store/AMO).

The G bit in all G-stage PTEs is reserved for future standard use, should be cleared by software
for forward compatibility, and must be ignored by hardware.

<!-- p.273 -->

      G-stage address translation uses the identical format for PTEs as regular address translation,
      even including the U bit, due to the possibility of sharing some (or all) page tables between G-
      stage translation and regular HS-level address translation. Regardless of whether this usage will
      ever become common, we chose not to preclude it.

##### 7.5.2 Guest-Page Faults

Guest-page-fault traps may be delegated from M-mode to HS-mode under the control of CSR
medeleg, but cannot be delegated to other operating modes. On a guest-page fault, CSR mtval or
stval is written with the faulting guest virtual address as usual, and mtval2 or htval is written
either with zero or with the faulting guest physical address, shifted right by 2 bits. CSR mtinst
or htinst may also be written with information about the faulting instruction or other reason for
the access, as explained in Section 7.6.3.

When an instruction fetch or a misaligned memory access straddles a page boundary, two different
address translations are involved. When a guest-page fault occurs in such a circumstance, the
faulting virtual address written to mtval/stval is the same as would be required for a regular
page fault. Thus, the faulting virtual address may be a page-boundary address that is higher than
the instruction’s original virtual address, if the byte at that page boundary is among the accessed
bytes.

When a guest-page fault is not due to an implicit memory access for VS-stage address translation,
a nonzero guest physical address written to mtval2/htval shall correspond to the exact virtual
address written to mtval/stval.

##### 7.5.3 Memory-Management Fences

The behavior of the SFENCE.VMA instruction is affected by the current virtualization mode V.
When V=0, the virtual-address argument is an HS-level virtual address, and the ASID argument
is an HS-level ASID. The instruction orders stores only to HS-level address-translation structures
with subsequent HS-level address translations.

When V=1, the virtual-address argument to SFENCE.VMA is a guest virtual address within the
current virtual machine, and the ASID argument is a VS-level ASID within the current virtual
machine. The current virtual machine is identified by the VMID field of CSR hgatp, and the
effective ASID can be considered to be the combination of this VMID with the VS-level ASID. The
SFENCE.VMA instruction orders stores only to the VS-level address-translation structures with
subsequent VS-stage address translations for the same virtual machine, i.e., only when hgatp.VMID
is the same as when the SFENCE.VMA executed.

Hypervisor instructions HFENCE.VVMA and HFENCE.GVMA provide additional memorymanagement fences to complement SFENCE.VMA. These instructions are described in Section 7.3.2.

Section 3.7.2 discusses the intersection between physical memory protection (PMP) and page-based
address translation. It is noted there that, when PMP settings are modified in a manner that affects either the physical memory that holds page tables or the physical memory to which page tables
point, M-mode software must synchronize the PMP settings with the virtual memory system. For
HS-level address translation, this is accomplished by executing in M-mode an SFENCE.VMA instruction with rs1=x0 and rs2=x0, after the PMP CSRs are written. If G-stage address translation
is in use and is not Bare, synchronization with its data structures is also needed. When PMP settings are modified in a manner that affects either the physical memory that holds guest-physical
page tables or the physical memory to which guest-physical page tables point, an HFENCE.GVMA
instruction with rs1=x0 and rs2=x0 must be executed in M-mode after the PMP CSRs are written.
An HFENCE.VVMA instruction is not required.

<!-- p.274 -->

#### 7.6 Traps

##### 7.6.1 Trap Cause Codes

The hypervisor extension augments the trap cause encoding. Table 7.5 lists the possible M-mode
and HS-mode trap cause codes when the hypervisor extension is implemented. Codes are added for
VS-level interrupts (interrupts 2, 6, 10), for supervisor-level guest external interrupts (interrupt 12),
for virtual instruction exceptions (exception 22), and for guest-page faults (exceptions 20, 21, 23).
Furthermore, environment calls from VS-mode are assigned cause 10, whereas those from HS-mode
or S-mode use cause 9 as usual.

      HS-mode and VS-mode ECALLs use different cause values so they can be delegated separately.

When V=1, a virtual instruction trap (not an illegal instruction trap) is taken for:

- attempts to access a counter CSR when the corresponding bit in hcounteren is 0 and the same bit in mcounteren is 1;

- attempts to execute a hypervisor instruction (HLV, HLVX, HSV, or HFENCE) or to access an implemented hypervisor CSR or VS CSR;

- in VU-mode, attempts to execute WFI or a supervisor instruction (SRET or SFENCE), or to access an implemented supervisor CSR;

- in VS-mode, attempts to execute WFI when hstatus.VTW=1 and mstatus.TW=0, unless the instruction completes within an implementation-specific, bounded time;

- in VS-mode, attempts to execute SRET when hstatus.VTSR=1; or

- in VS-mode, attempts to execute an SFENCE instruction or to access satp, when hstatus.VTVM=1.

On a virtual instruction trap, mtval or stval is written the same as for an illegal instruction trap.

      When V=1, privileged instructions that are invalid in VS-mode or VU-mode generally cause
      a virtual instruction trap instead of an illegal instruction trap. The same goes for attempts
      to access hypervisor- or supervisor-level CSRs that fail due to insufficient privilege when V=1,

<!-- p.275 -->

```text
           Interrupt   Exception Code   Description
                   1                0   Reserved
                   1                1   Supervisor software interrupt
                   1                2   Virtual supervisor software interrupt
                   1                3   Machine software interrupt
                   1                4   Reserved
                   1                5   Supervisor timer interrupt
                   1                6   Virtual supervisor timer interrupt
                   1                7   Machine timer interrupt
                   1                8   Reserved
                   1                9   Supervisor external interrupt
                   1               10   Virtual supervisor external interrupt
                   1               11   Machine external interrupt
                   1               12   Supervisor guest external interrupt
                   1            13–15   Reserved
                   1              ≥16   Designated for platform or custom use
                   0                0   Instruction address misaligned
                   0                1   Instruction access fault
                   0                2   Illegal instruction
                   0                3   Breakpoint
                   0                4   Load address misaligned
                   0                5   Load access fault
                   0                6   Store/AMO address misaligned
                   0                7   Store/AMO access fault
                   0                8   Environment call from U-mode or VU-mode
                   0                9   Environment call from HS-mode
                   0               10   Environment call from VS-mode
                   0               11   Environment call from M-mode
                   0               12   Instruction page fault
                   0               13   Load page fault
                   0               14   Reserved
                   0               15   Store/AMO page fault
                   0            16–19   Reserved
                   0               20   Instruction guest-page fault
                   0               21   Load guest-page fault
                   0               22   Virtual instruction
                   0               23   Store/AMO guest-page fault
                   0            24–31   Designated for custom use
                   0            32–47   Reserved
                   0            48–63   Designated for custom use
                   0              ≥64   Reserved
```

**Table 7.5:** Machine and supervisor cause register (mcause and scause) values when the hypervisor extension is implemented.

<!-- p.276 -->

     or attempts to access CSRs to which access has been expressly disabled by a hypervisor CSR
     (e.g. hcounteren). It is not unusual that hypervisors must emulate such instructions, to sup-
     port nested hypervisors or for other reasons. When not emulating an instruction, a hypervisor
     should convert a virtual instruction trap into an illegal instruction exception for the guest virtual
     machine.
         Machine level is expected ordinarily to delegate virtual instruction traps directly to HS-level,
     whereas illegal instruction traps are likely to be processed first in M-mode before being condi-
     tionally delegated (by software) to HS-level. Consequently, virtual instruction traps are expected
     typically to be handled faster than illegal instruction traps.

##### 7.6.2 Trap Entry

When a trap occurs in HS-mode or U-mode, it goes to M-mode, unless delegated by medeleg or
mideleg, in which case it goes to HS-mode. When a trap occurs in VS-mode or VU-mode, it goes
to M-mode, unless delegated by medeleg or mideleg, in which case it goes to HS-mode, unless
further delegated by hedeleg or hideleg, in which case it goes to VS-mode.

```text
When a trap is taken into M-mode, virtualization mode V gets set to 0, and fields MPV and MPP
in mstatus (or mstatush) are set according to Table 7.6. A trap into M-mode also writes fields
GVA, MPIE, and MIE in mstatus/mstatush and writes CSRs mepc, mcause, mtval, mtval2, and
mtinst.
                                     Previous Mode        MPV      MPP
                                     U-mode                0        0
                                     HS-mode               0        1
                                     M-mode                0        3
                                     VU-mode               1        0
                                     VS-mode               1        1
```

**Table 7.6:** Value of mstatus/mstatush fields MPV and MPP after a trap into M-mode. Upon trap return, MPV is ignored when MPP=3.

When a trap is taken into HS-mode, virtualization mode V is set to 0, and hstatus.SPV and
sstatus.SPP are set according to Table 7.7. If V was 1 before the trap, field SPVP in hstatus is
set the same as sstatus.SPP; otherwise, SPVP is left unchanged. A trap into HS-mode also writes
field GVA in hstatus, fields SPIE and SIE in sstatus, and CSRs sepc, scause, stval, htval,
and htinst.
                                      Previous Mode        SPV     SPP
                                      U-mode                0       0
                                      HS-mode               0       1
                                      VU-mode               1       0
                                      VS-mode               1       1

**Table 7.7:** Value of hstatus field SPV and sstatus field SPP after a trap into HS-mode.

When a trap is taken into VS-mode, vsstatus.SPP is set according to Table 7.8. Register hstatus
and the HS-level sstatus are not modified, and the virtualization mode V remains 1. A trap into VS-mode also writes fields SPIE and SIE in vsstatus and writes CSRs vsepc, vscause, and
vstval.

<!-- p.277 -->

```text
                                               Previous Mode       SPP
                                               VU-mode              0
                                               VS-mode              1
```

**Table 7.8:** Value of vsstatus field SPP after a trap into VS-mode.

##### 7.6.3 Transformed Instruction or Pseudoinstruction for mtinst or htinst

On any trap into M-mode or HS-mode, one of these values is written automatically into the appropriate trap instruction CSR, mtinst or htinst:

- zero;
- a transformation of the trapping instruction;
- a custom value (allowed only if the trapping instruction is nonstandard); or
- a special pseudoinstruction.

Except when a pseudoinstruction value is required (described later), the value written to mtinst
or htinst may always be zero, indicating that the hardware is providing no information in the
register for this particular trap.

          The value written to the trap instruction CSR serves two purposes. The first is to improve the
          speed of instruction emulation in a trap handler, partly by allowing the handler to skip loading
          the trapping instruction from memory, and partly by obviating some of the work of decoding and
          executing the instruction. The second purpose is to supply, via pseudoinstructions, additional
          information about guest-page-fault exceptions caused by implicit memory accesses done for VS-
          stage address translation.
              A transformation of the trapping instruction is written instead of simply a copy of the original
          instruction in order to minimize the burden for hardware yet still provide to a trap handler the
          information needed to emulate the instruction. An implementation may at any time reduce its
          effort by substituting zero in place of the transformed instruction.

On an interrupt, the value written to the trap instruction register is always zero. On a synchronous
exception, if a nonzero value is written, one of the following shall be true about the value:

- Bit 0 is 1, and replacing bit 1 with 1 makes the value into a valid encoding of a standard instruction. In this case, the instruction that trapped is the same kind as indicated by the register value, and the register value is the transformation of the trapping instruction, as defined later. For example, if bits 1:0 are binary 11 and the register value is the encoding of a standard LW (load word) instruction, then the trapping instruction is LW, and the register value is the transformation of the trapping LW instruction.

- Bit 0 is 1, and replacing bit 1 with 1 makes the value into an instruction encoding that is explicitly designated for a custom instruction (not an unused reserved encoding).

<!-- p.278 -->

     This is a custom value. The instruction that trapped is a nonstandard instruction. The
     interpretation of a custom value is not otherwise specified by this standard.

- The value is one of the special pseudoinstructions defined later, all of which have bits 1:0 equal to 00.

These three cases exclude a large number of other possible values, such as all those having bits 1:0
equal to binary 10. A future standard or extension may define additional cases, thus allowing values
that are currently excluded. Software may safely treat an unrecognized value in a trap instruction
register the same as zero.

     To be forward-compatible with future revisions of this standard, software that interprets a nonzero
     value from mtinst or htinst must fully verify that the value conforms to one of the cases listed
     above. For instance, for RV64, discovering that bits 6:0 of mtinst are 0000011 and bits 14:12
     are 010 is not sufficient to establish that the first case applies and the trapping instruction is a
     standard LW instruction; rather, software must also confirm that bits 63:32 of mtinst are all
     zeros. A future standard might define new values for 64-bit mtinst that are nonzero in bits 63:32
     yet may coincidentally have in bits 31:0 the same bit patterns as standard RV64 instructions.

     Unlike for standard instructions, there is no requirement that the instruction encoding of a
     custom value be of the same “kind” as the instruction that trapped (or even have any correlation
     with the trapping instruction).

Table 7.9 shows the values that may be automatically written to the trap instruction register for
each standard exception cause. For exceptions that prevent the fetching of an instruction, only
zero or a pseudoinstruction value may be written. A custom value may be automatically written
only if the instruction that traps is nonstandard. A future standard or extension may permit other
values to be written, chosen from the set of allowed values established earlier.

As enumerated in the table, a synchronous exception may write to the trap instruction register
a standard transformation of the trapping instruction only for exceptions that arise from explicit
memory accesses (from loads, stores, and AMO instructions). Accordingly, standard transformations are currently defined only for these memory-access instructions. If a synchronous trap occurs
for a standard instruction for which no transformation has been defined, the trap instruction register shall be written with zero (or, under certain circumstances, with a special pseudoinstruction
value).

<!-- p.279 -->

```text
                                                        Transformed                Pseudo-
                                                          Standard     Custom    instruction
            Exception                           Zero     Instruction    Value       Value
            Instruction address misaligned      Yes          No          Yes         No
            Instruction access fault            Yes          No          No          No
            Illegal instruction                 Yes          No          No          No
            Breakpoint                          Yes          No          Yes         No
            Virtual instruction                 Yes          No          Yes         No
            Load address misaligned             Yes          Yes         Yes         No
            Load access fault                   Yes          Yes         Yes         No
            Store/AMO address misaligned        Yes          Yes         Yes         No
            Store/AMO access fault              Yes          Yes         Yes         No
            Environment call                    Yes          No          Yes         No
            Instruction page fault              Yes          No          No          No
            Load page fault                     Yes          Yes         Yes         No
            Store/AMO page fault                Yes          Yes         Yes         No
            Instruction guest-page fault        Yes          No          No          Yes
            Load guest-page fault               Yes          Yes         Yes         Yes
            Store/AMO guest-page fault          Yes          Yes         Yes         Yes
```

**Table 7.9:** Values that may be automatically written to the trap instruction register (mtinst or htinst) on an exception trap.

For a standard load instruction that is not a compressed instruction and is one of LB, LBU, LH,
LHU, LW, LWU, LD, FLW, FLD, or FLQ, the transformed instruction has the format shown in
Figure 7.43.

```text
       31               25 24         20 19            15 14   12 11        76                 0
                0               0        Addr. Offset     funct3       rd           opcode
                7               5             5              3          5             7
```

**Figure 7.43:** Transformed noncompressed load instruction (LB, LBU, LH, LHU, LW, LWU, LD, FLW, FLD, or FLQ). Fields funct3, rd, and opcode are the same as the trapping load instruction.

For a standard store instruction that is not a compressed instruction and is one of SB, SH, SW,
SD, FSW, FSD, or FSQ, the transformed instruction has the format shown in Figure 7.44.

```text
       31               25 24         20 19            15 14   12 11        76                 0
                0               rs2      Addr. Offset     funct3       0            opcode
                7                5            5              3         5              7
```

**Figure 7.44:** Transformed noncompressed store instruction (SB, SH, SW, SD, FSW, FSD, or FSQ). Fields rs2, funct3, and opcode are the same as the trapping store instruction.

For a standard atomic instruction (load-reserved, store-conditional, or AMO instruction), the transformed instruction has the format shown in Figure 7.45.

<!-- p.280 -->

```text
     31              27 26    25   24           20 19             15 14        12 11          76                 0
            funct5       aq   rl         rs2           Addr. Offset      funct3         rd          opcode
              5           1   1           5                 5               3            5            7
```

**Figure 7.45:** Transformed atomic instruction (load-reserved, store-conditional, or AMO instruc- tion). All fields are the same as the trapping instruction except bits 19:15, Addr. Offset.

For a standard virtual-machine load/store instruction (HLV, HLVX, or HSV), the transformed
instruction has the format shown in Figure 7.46.

```text
       31                     25 24            20 19             15 14      12 11            76              0
                funct7                  rs2       Addr. Offset        funct3           rd          opcode
                  7                      5             5                 3              5            7
```

**Figure 7.46:** Transformed virtual-machine load/store instruction (HLV, HLVX, HSV). All fields are the same as the trapping instruction except bits 19:15, Addr. Offset.

In all the transformed instructions above, the Addr. Offset field that replaces the instruction’s rs1
field in bits 19:15 is the positive difference between the faulting virtual address (written to mtval or
stval) and the original virtual address. This difference can be nonzero only for a misaligned memory access. Note also that, for basic loads and stores, the transformations replace the instruction’s
immediate offset fields with zero.

For a standard compressed instruction (16-bit size), the transformed instruction is found as follows:

  1. Expand the compressed instruction to its 32-bit equivalent.

  2. Transform the 32-bit equivalent instruction.

  3. Replace bit 1 with a 0.

Bits 1:0 of a transformed standard instruction will be binary 01 if the trapping instruction is
compressed and 11 if not.

      In decoding the contents of mtinst or htinst, once software has determined that the register
      contains the encoding of a standard basic load (LB, LBU, LH, LHU, LW, LWU, LD, FLW, FLD,
      or FLQ) or basic store (SB, SH, SW, SD, FSW, FSD, or FSQ), it is not necessary to confirm
      also that the immediate offset fields (31:25, and 24:20 or 11:7) are zeros. The knowledge that
      the register’s value is the encoding of a basic load/store is sufficient to prove that the trapping
      instruction is of the same kind.
          A future version of this standard may add information to the fields that are currently zeros.
      However, for backwards compatiblity, any such information will be for performance purposes
      only and can safely be ignored.

For guest-page faults, the trap instruction register is written with a special pseudoinstruction value
if: (a) the fault is caused by an implicit memory access for VS-stage address translation, and (b) a
nonzero value (the faulting guest physical address) is written to mtval2 or htval. If both conditions
are met, the value written to mtinst or htinst must be taken from Table 7.10; zero is not allowed.

<!-- p.281 -->

```text
                Value            Meaning
                0x00002000       32-bit read for VS-stage address translation (RV32)
                0x00002020       32-bit write for VS-stage address translation (RV32)
                0x00003000       64-bit read for VS-stage address translation (RV64)
                0x00003020       64-bit write for VS-stage address translation (RV64)
```

**Table 7.10:** Special pseudoinstruction values for guest-page faults. The RV32 values are used when VSXLEN=32, and the RV64 values when VSXLEN=64.

The defined pseudoinstruction values are designed to correspond closely with the encodings of basic
loads and stores, as illustrated by Table 7.11.

```text
                                      Encoding         Instruction
                                      0x00002003       lw x0,0(x0)
                                      0x00002023       sw x0,0(x0)
                                      0x00003003       ld x0,0(x0)
                                      0x00003023       sd x0,0(x0)
```

**Table 7.11:** Standard instructions corresponding to the special pseudoinstructions of Table 7.10.

A write pseudoinstruction (0x00002020 or 0x00003020) is used for the case that the machine is
attempting automatically to update bits A and/or D in VS-level page tables. All other implicit
memory accesses for VS-stage address translation will be reads. If a machine never automatically
updates bits A or D in VS-level page tables (leaving this to software), the write case will never
arise. The fact that such a page table update must actually be atomic, not just a simple write, is
ignored for the pseudoinstruction.

      If the conditions that necessitate a pseudoinstruction value can ever occur for M-mode, then
      mtinst cannot be hardwired entirely to zero; and likewise for HS-mode and htinst. However,
      in that case, the trap instruction registers may minimally support only values 0 and 0x00002000
      or 0x00003000, and possibly 0x00002020 or 0x00003020, requiring as few as one or two flip-
      flops in hardware, per register.

      There is no harm here in ignoring the atomicity requirement for page table updates, because
      a hypervisor is not expected in these circumstances to emulate an implicit memory access that
      fails. Rather, the hypervisor is given enough information about the faulting access to be able to
      make the memory accessible (e.g. by restoring a missing page of virtual memory) before resuming
      execution by retrying the faulting instruction.

##### 7.6.4 Trap Return

The MRET instruction is used to return from a trap taken into M-mode. MRET first determines
what the new operating mode will be according to the values of MPP and MPV in mstatus
or mstatush, as encoded in Table 7.6. MRET then in mstatus/mstatush sets MPV=0, MPP=0,
MIE=MPIE, and MPIE=1. Lastly, MRET sets the virtualization and privilege modes as previously
determined, and sets pc=mepc.

<!-- p.282 -->

The SRET instruction is used to return from a trap taken into HS-mode or VS-mode. Its behavior
depends on the current virtualization mode.

When executed in M-mode or HS-mode (i.e., V=0), SRET first determines what the new operating
mode will be according to the values in hstatus.SPV and sstatus.SPP, as encoded in Table 7.7.
SRET then sets hstatus.SPV=0, and in sstatus sets SPP=0, SIE=SPIE, and SPIE=1. Lastly,
SRET sets the virtualization and privilege modes as previously determined, and sets pc=sepc.

When executed in VS-mode (i.e., V=1), SRET sets the privilege mode according to Table 7.8, in
vsstatus sets SPP=0, SIE=SPIE, and SPIE=1, and lastly sets pc=vsepc.

<!-- p.284 -->

### 第 8 章 “N” Standard Extension for User-Level Interrupts, Version 1.1

      This is a placeholder for a more complete writeup of the N extension, and to form a basis for
      discussion.
          An ongoing topic of discussion is whether, for systems needing only M and U privilege
      modes, the N extension should be supplanted by S-mode without virtual memory (i.e., with satp
      hardwired to zero). This approach would have similar hardware cost and would simplify the
      architecture.

This chapter presents a proposal for adding RISC-V user-level interrupt and exception handling.
When the N extension is present, and the outer execution environment has delegated designated
interrupts and exceptions to user-level, then hardware can transfer control directly to a user-level
trap handler without invoking the outer execution environment.

      User-level interrupts are primarily intended to support secure embedded systems with only M-
      mode and U-mode present, but can also be supported in systems running Unix-like operating
      systems to support user-level trap handling.
          When used in an Unix environment, the user-level interrupts would likely not replace conven-
      tional signal handling, but could be used as a building block for further extensions that generate
      user-level events such as garbage collection barriers, integer overflow, floating-point traps.

#### 8.1 Additional CSRs

New user-visible CSRs are added to support the N extension. Their encodings are listed in Table 2.2
in Chapter 2.

##### 8.1.1 User Status Register (ustatus)

The ustatus register is a UXLEN-bit read/write register formatted as shown in Figure 8.1. The
ustatus register keeps track of and controls the hart’s current operating state.

<!-- p.285 -->

```text
               UXLEN                                           5       4       3    1          0
                                  WPRI                             UPIE WPRI                  UIE
                                 UXLEN-5                            1    3                     1
```

**Figure 8.1:** User-mode status register (ustatus).

The user interrupt-enable bit UIE disables user-level interrupts when clear. The value of UIE is
copied into UPIE when a user-level trap is taken, and the value of UIE is set to zero to provide
atomicity for the user-level trap handler.

The UIE and UPIE bits are mirrored in the mstatus and sstatus registers in the same bit positions.

      There is no UPP bit to hold the previous privilege mode as it can only be user mode.

A new instruction, URET, is used to return from traps in U-mode. URET copies UPIE into UIE,
then sets UPIE, before copying uepc to the pc.

      UPIE is set after the UPIE/UIE stack is popped to enable interrupts and help catch coding
      errors.

##### 8.1.2 User Interrupt Registers (uip and uie)

The uip register is a UXLEN-bit read/write register containing information on pending interrupts,
while uie is the corresponding UXLEN-bit read/write register containing interrupt enable bits.

```text
        UXLEN-1                                       9    8       7       5        4     3         1    0
                            WPRI                          UEIP     WPRI            UTIP   WPRI          USIP
                           UXLEN-9                         1        3               1      3             1
```

**Figure 8.2:** User interrupt-pending register (uip).

```text
        UXLEN-1                                       9    8       7       5        4     3         1    0
                            WPRI                          UEIE     WPRI            UTIE   WPRI          USIE
                           UXLEN-9                         1        3               1      3             1
```

**Figure 8.3:** User interrupt-enable register (uie).

Three types of interrupts are defined: software interrupts, timer interrupts, and external interrupts.
A user-level software interrupt is triggered on the current hart by writing 1 to its user software
interrupt-pending (USIP) bit in the uip register. A pending user-level software interrupt can be
cleared by writing 0 to the USIP bit in uip. User-level software interrupts are disabled when the
USIE bit in the uie register is clear.

The ABI should provide a mechanism to send interprocessor interrupts to other harts, which will
ultimately cause the USIP bit to be set in the recipient hart’s uip register.

All bits besides USIP in the uip register are read-only.

<!-- p.286 -->

A user-level timer interrupt is pending if the UTIP bit in the uip register is set. User-level timer
interrupts are disabled when the UTIE bit in the uie register is clear. The ABI should provide a
mechanism to clear a pending timer interrupt.

A user-level external interrupt is pending if the UEIP bit in the uip register is set. User-level
external interrupts are disabled when the UEIE bit in the uie register is clear. The ABI should
provide facilities to mask, unmask, and query the cause of external interrupts.

The uip and uie registers are subsets of the mip and mie registers. Reading any field, or writing
any writable field, of uip/uie effects a read or write of the homonymous field of mip/mie. If S-mode
is implemented, the uip and uie registers are also subsets of the sip and sie registers.

##### 8.1.3 Machine Trap Delegation Registers (medeleg and mideleg)

In systems with the N extension, the medeleg and mideleg registers, described in Chapter 3, must
be implemented.

In systems that implement S-mode, medeleg and mideleg behave as described in Chapter 3. In
systems with only M and U privilege modes, setting a bit in medeleg or mideleg delegates the
corresponding trap in U-mode to the U-mode trap handler.

##### 8.1.4 Supervisor Trap Delegation Registers (sedeleg and sideleg)

For systems with both S-mode and the N extension, new CSRs sedeleg and sideleg are added.
These registers have the same layout as the machine trap delegation registers, medeleg and mideleg.

sedeleg and sideleg allow S-mode to delegate traps to U-mode. Only bits corresponding to traps
that have been delegated to S-mode are writable; the others are hardwired to zero. Setting a bit
in sedeleg or sideleg delegates the corresponding trap in U-mode to the U-mode trap handler.

##### 8.1.5 Other CSRs

The uscratch, uepc, ucause, utvec, and utval CSRs are defined analogously to the mscratch,
mepc, mcause, mtvec, and mtval CSRs.

      A more complete writeup is to follow.

#### 8.2 N Extension Instructions

The URET instruction is added to perform the analogous function to MRET and SRET.

<!-- p.287 -->

#### 8.3 Reducing Context-Swap Overhead

The user-level interrupt-handling registers add considerable state to the user-level context, yet will
usually rarely be active in normal use. In particular, uepc, ucause, and utval are only valid during
execution of a trap handler.

An NS field can be added to mstatus and sstatus following the format of the FS and XS fields
to reduce context-switch overhead when the values are not live. Execution of URET will place the
uepc, ucause, and utval back into initial state.

<!-- p.288 -->

### 第 9 章 RISC-V Privileged Instruction Set Listings

This chapter presents instruction-set listings for all instructions defined in the RISC-V Privileged
Architecture.

The instruction-set listings for unprivileged instructions, including the ECALL and EBREAK instructions, are provided in Volume I of this manual.

<!-- p.289 -->

```text
      31      27     26 25 24          20   19         15   14   12   11        7   6             0
           funct7                rs2             rs1        funct3         rd           opcode        R-type
                imm[11:0]                        rs1        funct3         rd           opcode        I-type
                                  Trap-Return Instructions
           0000000              00010     00000      000               00000            1110011       URET
           0001000              00010     00000      000               00000            1110011       SRET
           0011000              00010     00000      000               00000            1110011       MRET
```

                          Interrupt-Management Instructions
           0001000           00101     00000     000     00000                          1110011       WFI

                     Supervisor Memory-Management Instructions
           0001001            rs2      rs1    000      00000                            1110011       SFENCE.VMA

```text
                     Hypervisor Memory-Management Instructions
           0010001            rs2      rs1    000      00000                            1110011       HFENCE.VVMA
           0110001            rs2      rs1    000      00000                            1110011       HFENCE.GVMA
             Hypervisor Virtual-Machine Load and Store Instructions
           0110000        00000       rs1     100       rd       1110011                              HLV.B
           0110000        00001       rs1     100       rd       1110011                              HLV.BU
           0110010        00000       rs1     100       rd       1110011                              HLV.H
           0110010        00001       rs1     100       rd       1110011                              HLV.HU
           0110010        00011       rs1     100       rd       1110011                              HLVX.HU
           0110100        00000       rs1     100       rd       1110011                              HLV.W
           0110100        00011       rs1     100       rd       1110011                              HLVX.WU
           0110001         rs2        rs1     100     00000      1110011                              HSV.B
           0110011         rs2        rs1     100     00000      1110011                              HSV.H
           0110101         rs2        rs1     100     00000      1110011                              HSV.W
       Hypervisor Virtual-Machine Load and Store Instructions, RV64 only
          0110100         00001      rs1      100       rd       1110011                              HLV.WU
          0110110         00000      rs1      100       rd       1110011                              HLV.D
          0110111          rs2       rs1      100     00000      1110011                              HSV.D
```

**Table 9.1:** RISC-V Privileged Instructions

<!-- p.290 -->

### 第 10 章 History

#### 10.1 Research Funding at UC Berkeley

Development of the RISC-V architecture and implementations has been partially funded by the
following sponsors.

- Par Lab: Research supported by Microsoft (Award #024263) and Intel (Award #024894) funding and by matching funding by U.C. Discovery (Award #DIG07-10227). Additional support came from Par Lab affiliates Nokia, NVIDIA, Oracle, and Samsung.

- Project Isis: DoE Award DE-SC0003624.

- ASPIRE Lab: DARPA PERFECT program, Award HR0011-12-2-0016. DARPA POEM program Award HR0011-11-C-0100. The Center for Future Architectures Research (C-FAR), a STARnet center funded by the Semiconductor Research Corporation. Additional support from ASPIRE industrial sponsor, Intel, and ASPIRE affiliates, Google, Huawei, Nokia, NVIDIA, Oracle, and Samsung.

The content of this paper does not necessarily reflect the position or the policy of the US government
and no official endorsement should be inferred.

<!-- p.292 -->

Bibliography

[1] Robert P. Goldberg. Survey of virtual machine research. Computer, 7(6):34–45, June 1974.

[2] Juan Navarro, Sitaram Iyer, Peter Druschel, and Alan Cox. Practical, transparent operating
    system support for superpages. SIGOPS Oper. Syst. Rev., 36(SI):89–104, December 2002.
