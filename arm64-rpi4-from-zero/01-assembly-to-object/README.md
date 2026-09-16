**# 01 — Assembly Source to Object File**

This stage is intentionally tiny.

The goal is to understand one transformation:

```text

answer.s

   │

   │  GNU ARM64 assembler

   ▼

answer.o

```

The source file used here is:

```asm

.global answer

answer:

    mov x0, #42

    ret

```

The command used to assemble it is:

```bash

aarch64-linux-gnu-as answer.s -o answer.o

```

The generated file can be identified with:

```bash

file answer.o

```

Example output:

```text

answer.o: ELF 64-bit LSB relocatable, ARM aarch64, version 1 (SYSV), not stripped

```

The important point is that `answer.o` is **not yet a finished executable**. It is an ELF **relocatable object file** containing ARM64 machine code plus metadata that a linker can use later.

**---**

**## Inspecting the ELF sections**

Running:

```bash

aarch64-linux-gnu-readelf -S answer.o

```

for this particular `answer.s` produced:

```text

There are 7 section headers, starting at offset 0x110:

Section Headers:

  [Nr] Name              Type             Address           Offset

       Size              EntSize          Flags  Link  Info  Align

  [ 0]                   NULL             0000000000000000  00000000

       0000000000000000  0000000000000000           0     0     0

  [ 1] .text             PROGBITS         0000000000000000  00000040

       0000000000000008  0000000000000000  AX       0     0     4

  [ 2] .data             PROGBITS         0000000000000000  00000048

       0000000000000000  0000000000000000  WA       0     0     1

  [ 3] .bss              NOBITS           0000000000000000  00000048

       0000000000000000  0000000000000000  WA       0     0     1

  [ 4] .symtab           SYMTAB           0000000000000000  00000048

       0000000000000090  0000000000000018           5     5     8

  [ 5] .strtab           STRTAB           0000000000000000  000000d8

       000000000000000b  0000000000000000           0     0     1

  [ 6] .shstrtab         STRTAB           0000000000000000  000000e3

       000000000000002c  0000000000000000           0     0     1

```

Flattened into a more readable table:

| Nr | Name | Type | Address | Offset | Size | EntSize | Flags | Link | Info | Align |

|---:|---|---|---:|---:|---:|---:|---|---:|---:|---:|

| 0 | — | NULL | `0x0000000000000000` | `0x00000000` | `0x0` | `0x0` | — | 0 | 0 | 0 |

| 1 | `.text` | PROGBITS | `0x0000000000000000` | `0x00000040` | `0x8` | `0x0` | AX | 0 | 0 | 4 |

| 2 | `.data` | PROGBITS | `0x0000000000000000` | `0x00000048` | `0x0` | `0x0` | WA | 0 | 0 | 1 |

| 3 | `.bss` | NOBITS | `0x0000000000000000` | `0x00000048` | `0x0` | `0x0` | WA | 0 | 0 | 1 |

| 4 | `.symtab` | SYMTAB | `0x0000000000000000` | `0x00000048` | `0x90` | `0x18` | — | 5 | 5 | 8 |

| 5 | `.strtab` | STRTAB | `0x0000000000000000` | `0x000000d8` | `0xb` | `0x0` | — | 0 | 0 | 1 |

| 6 | `.shstrtab` | STRTAB | `0x0000000000000000` | `0x000000e3` | `0x2c` | `0x0` | — | 0 | 0 | 1 |

**---**

**## How those section values map into the actual `answer.o` file**

For this object file, `.text` begins at file offset `0x40` and has a size of `0x8` bytes.

The two AArch64 instructions are 4 bytes each:

```text

answer.o

file offset

0x00  ┌──────────────────────────────┐

      │ ELF header / metadata        │

      │ before .text                 │

      │                              │

0x40  ├──────────────────────────────┤

      │ mov x0, #42                  │ 4 bytes

0x44  │ ret                          │ 4 bytes

0x48  ├──────────────────────────────┤

      │ .data                        │ 0 bytes

      ├──────────────────────────────┤

      │ .bss                         │ 0 bytes

0x48  ├──────────────────────────────┤

      │ .symtab                      │ 0x90 bytes

      │ ...                          │

0xd8  ├──────────────────────────────┤

      │ .strtab                      │ 0x0b bytes

0xe3  ├──────────────────────────────┤

      │ .shstrtab                    │ 0x2c bytes

      └──────────────────────────────┘

```

The arithmetic lines up exactly:

```text

.text begins at 0x40

.text size     = 0x08

0x40 + 0x08 = 0x48

```

Therefore the next section begins at file offset `0x48`.

Likewise:

```text

.symtab begins at 0x48

.symtab size   = 0x90

0x48 + 0x90 = 0xD8

```

and `.strtab` indeed begins at `0xD8`.

Then:

```text

.strtab begins at 0xD8

.strtab size   = 0x0B

0xD8 + 0x0B = 0xE3

```

and `.shstrtab` begins at `0xE3`.

This makes the `readelf` output much less mysterious: the `Offset` and `Size` columns describe where the corresponding blocks are located **inside the object file on disk**.

**---**

**## File offset vs memory address**

This distinction is fundamental.

**### Offset**

`Offset` answers:

> Where does this section begin inside the file `answer.o`?

For example:

```text

.text Offset = 00000040

```

means:

```text

.text begins 0x40 bytes from the beginning of answer.o

```

Since:

```text

0x40 = 64 decimal

```

the first ARM64 instruction is stored beginning at byte 64 of the object file.

**### Address**

`Address` answers:

> At what memory address should this section exist when the program is loaded?

At this stage every address is:

```text

0000000000000000

```

because `answer.o` is only a relocatable object file.

No linker has yet decided where `.text`, `.data`, and the other allocatable sections will live in memory.

So at this stage:

```text

file offset -> already meaningful

memory address -> not final yet

```

**---**

**## Why does `readelf` show offsets with 8 hex digits but addresses with 16?**

Example:

```text

Address           Offset

0000000000000000  00000040

```

The leading zeroes do not change the value.

Therefore:

```text

00000040 = 0x40

```

`readelf` uses fixed-width columns mainly for readability and alignment.

A hexadecimal digit represents exactly 4 bits because one hex digit has 16 possible values:

```text

0 ... F

```

and:

```text

2^4 = 16

```

So every group of four binary bits maps exactly to one hexadecimal digit:

```text

Binary   Hex

0000  -> 0

0001  -> 1

...

1010  -> A

...

1111  -> F

```

For example:

```text

0x10

```

corresponds directly to:

```text

0001 0000

```

or:

```text

0b00010000

```

A full 64-bit value needs:

```text

64 / 4 = 16

```

hexadecimal digits.

That is why a 64-bit address-shaped field can naturally be displayed as:

```text

0000000000000000

```

which contains 16 hex digits.

`readelf` prints the `Offset` column more compactly here using 8 hex digits because the actual offsets in this small object file are tiny.

However, this does **not** mean ELF64 file offsets are fundamentally limited to 32 bits. ELF64 file offsets are themselves capable of being 64-bit values. The shorter display is just `readelf` formatting for this output.

**---**

**## Why is ARM64 called ARM64?**

The important point is:

> ARM64 refers to ARM's 64-bit instruction-set architecture, AArch64.

It does **not** simply mean "the physical address bus has 64 wires."

One of the clearest manifestations of the "64" is the width of the general-purpose `x` registers:

```text

x0

x1

x2

...

x30

```

Each `x` register is 64 bits wide.

Our first instruction already uses one:

```asm

mov x0, #42

```

So `x0` is a 64-bit container receiving the integer value `42`.

AArch64 also provides a 32-bit view of these registers:

```text

x0 -> 64-bit view

w0 -> 32-bit view

```

The architecture supports 64-bit integer operations and uses 64-bit registers for pointers and addresses in its programming model.

That still does **not** imply that every ARM64 processor implements all possible 64 physical address bits.

For example, the Cortex-A72 used in the Raspberry Pi 4 implements a smaller physical address space than the full numerical range of a 64-bit register.

So these are separate ideas:

```text

64-bit AArch64 register

        │

        ├── can hold 2^64 different bit patterns

        │

        └── may contain an integer or an address

                         │

                         ▼

               actual address support

               depends on the CPU/system

```

A register can therefore hold a value such as:

```text

0xFFFFFFFFFFFFFFFF

```

even though that bit pattern is not necessarily a usable physical memory address.

**---**

**## Raspberry Pi 4: address space vs installed RAM**

This distinction is useful when thinking about the Raspberry Pi 4.

An 8 GiB Raspberry Pi 4 physically contains:

```text

8 × 2^30 bytes

= 2^33 bytes

= 0x200000000 bytes

```

of DRAM.

That is a **memory capacity**, not a maximum pointer value.

A Linux process can contain virtual pointer values numerically much larger than:

```text

0x200000000

```

because the MMU translates virtual addresses into physical memory locations.

So these are different concepts:

```text

AArch64 register width        -> 64 bits

CPU-supported address space   -> architecture / implementation dependent

Installed Raspberry Pi RAM    -> e.g. 8 GiB

Process virtual addresses     -> can numerically be much larger than RAM size

```

An 8 GiB machine can have at most roughly 8 GiB of information physically resident in its DRAM at one instant, with some of that memory used by the kernel, firmware, other processes, and other system needs.

But the numerical value of a pointer is not limited to `0x200000000`.

**---**

**## From assembly to the actual bytes**

From the ELF section table, we already know:

```text

.text Offset = 0x40

.text Size   = 0x08

```

So we can inspect exactly those eight bytes inside `answer.o`:

```bash

xxd -s 0x40 -l 8 answer.o

```

Output:

```text

00000040: 4005 80d2 c003 5fd6                      @....._.

```

The command means:

```text

-s 0x40  -> start reading at file offset 0x40

-l 8     -> display 8 bytes

```

Therefore the `.text` section physically contains:

```text

40 05 80 d2  c0 03 5f d6

```

AArch64 instructions are 32 bits, or 4 bytes, wide. We can therefore split the eight bytes into two instructions:

```text

40 05 80 d2    c0 03 5f d6

└──────┬───┘    └──────┬───┘

       │               │

 instruction 1     instruction 2

```

**### Little-endian storage**

The ELF file identifies itself as:

```text

elf64-littleaarch64

```

The instruction bytes are therefore stored little-endian.

The four bytes:

```text

40 05 80 d2

```

form the 32-bit instruction word:

```text

0xd2800540

```

Similarly:

```text

c0 03 5f d6

```

form:

```text

0xd65f03c0

```

This does not mean that the bits inside each byte are reversed. The **order of the bytes** is what changes when interpreting the 4-byte value as a little-endian 32-bit word.

**---**

**## Disassembling the machine code**

GNU `objdump` can interpret those instruction words as AArch64 instructions:

```bash

aarch64-linux-gnu-objdump -S answer.o

```

Output:

```text

answer.o:     file format elf64-littleaarch64



Disassembly of section .text:

0000000000000000 <answer>:

   0:   d2800540        mov     x0, #0x2a                       // #42

   4:   d65f03c0        ret

```

Now the relationship is visible end-to-end:

| Assembly source | 32-bit instruction word | Raw bytes in `answer.o` |

|---|---|---|

| `mov x0, #42` | `0xd2800540` | `40 05 80 d2` |

| `ret` | `0xd65f03c0` | `c0 03 5f d6` |

Or visually:

```text

SOURCE

mov x0, #42

      │

      │ assembler

      ▼

32-BIT AARCH64 INSTRUCTION

0xd2800540

      │

      │ stored little-endian

      ▼

BYTES IN answer.o

40 05 80 d2

```

and:

```text

SOURCE

ret

 │

 │ assembler

 ▼

32-BIT AARCH64 INSTRUCTION

0xd65f03c0

 │

 │ stored little-endian

 ▼

BYTES IN answer.o

c0 03 5f d6

```

`objdump` performs the useful reverse interpretation:

```text

raw machine instruction

        │

        │ disassembler

        ▼

human-readable assembly

```

The hexadecimal encodings themselves do **not** need to be memorized.

What matters is understanding that each AArch64 instruction is ultimately a 32-bit bit pattern. Different fields inside that bit pattern encode information such as:

- what operation should be performed,

- which register or registers are involved,

- immediate values embedded in the instruction,

- and other instruction-specific information.

For this learning journey, it is useful to manually decode one simple instruction once, to understand what an instruction encoding actually is. After that, the assembler and disassembler should normally handle these encodings for us.

**---**

**
---

## One manual instruction dissection: `mov x0, #42`

It is useful to decode one instruction manually once, not because the hexadecimal encoding should be memorized, but to understand what an instruction-set architecture actually defines.

For:

```asm
mov x0, #42
```

`objdump` showed the 32-bit instruction word:

```text
0xd2800540
```

Written in binary:

```text
11010010100000000000010101000000
```

For this instruction encoding, the 32 bits can be separated as:

| Bits | Binary value | Meaning |
|---|---|---|
| `31` | `1` | 64-bit form (`X` register) |
| `30:29` | `10` | `MOVZ` operation within the move-wide family |
| `28:23` | `100101` | move-wide immediate instruction family |
| `22:21` | `00` | shift amount = 0 |
| `20:5` | `0000000000101010` | 16-bit immediate = 42 |
| `4:0` | `00000` | destination register number 0 → `x0` |

Visually:

```text
11010010100000000000010101000000

1 | 10 | 100101 | 00 | 0000000000101010 | 00000
│    │      │       │          │             │
│    │      │       │          │             └─ destination = x0
│    │      │       │          └─────────────── immediate = 42
│    │      │       └────────────────────────── shift = 0
│    │      └────────────────────────────────── move-wide family
│    └───────────────────────────────────────── MOVZ operation
└────────────────────────────────────────────── 64-bit form
```

### The number `42` is literally encoded inside the instruction

The immediate field is:

```text
0000000000101010
```

Ignoring leading zeroes:

```text
101010
```

and:

```text
101010₂ = 32 + 8 + 2 = 42
```

So the constant written in `mov x0, #42` is represented directly by a field inside the 32-bit machine instruction.

### The destination register is also encoded in the instruction

The final five bits are:

```text
00000
```

Five bits can represent `2^5 = 32` different values. Here `00000₂ = 0`, so the destination register number is `0`, giving `x0` in the 64-bit form.

### Why `objdump` says `mov` although the encoding is `MOVZ`

For this constant, `mov x0, #42` is an assembler-friendly alias for a move-wide-immediate instruction whose underlying operation is `MOVZ` ("move wide with zero").

Conceptually, it produces:

```text
x0 =
0000000000000000
0000000000000000
0000000000000000
0000000000101010
```

The important lesson is **not** to memorize:

```text
mov x0, #42 = 0xd2800540
```

Instead, remember the structure:

```text
human-readable instruction
        │
        ▼
assembler applies AArch64 encoding rules
        │
        ▼
32-bit instruction containing fields for
operation + width + shift + immediate + register
        │
        ▼
machine instruction
```

For this example:

```text
mov x0, #42
        │
        ▼
MOVZ-compatible encoding
        │
        ▼
11010010100000000000010101000000
        │
        ▼
0xd2800540
        │
        ▼
stored little-endian in answer.o
        │
        ▼
40 05 80 d2
```

This is deep enough for the current stage. Normal ARM64 development should rely on the assembler and disassembler rather than memorizing machine-code encodings.

## Current checkpoint**

We have now followed the complete first-stage transformation:

```text

answer.s

    │

    │ aarch64-linux-gnu-as

    ▼

answer.o

    │

    ├── ELF metadata

    ├── symbol table

    └── .text

          │

          ├── 40 05 80 d2

          └── c0 03 5f d6

                 │

                 │ interpreted as AArch64

                 ▼

          mov x0, #42

          ret

```

The object file has been inspected using:

```bash

file answer.o

aarch64-linux-gnu-readelf -s answer.o

aarch64-linux-gnu-readelf -S answer.o

xxd -s 0x40 -l 8 answer.o

aarch64-linux-gnu-objdump -S answer.o

```

At this point we understand where the ARM64 machine instructions live inside the ELF object file and how the bytes produced by the assembler map back to the original assembly instructions.

We have now also manually inspected the bit fields of one instruction — `mov x0, #42` — to understand how an AArch64 machine instruction is structured, without treating the encoding as something that needs to be memorized.