# ISA

# PART III — THE INSTRUCTION BRIDGE

*The Instruction Set Architecture is the contract between hardware and software. Opcodes, decoders, registers, the full CPU pipeline, and protection rings — the hardware mechanism that enforces privilege and makes the OS possible.*

---

## Chapter 6: The Instruction Set Architecture — Hardware Meets Software

Chapter 03 established that logic gates respond to bit patterns. This chapter answers what those bit patterns are, how the control unit recognises them, what an opcode is and where the word came from, what the difference is between a CPU's bit width and its opcode width, how the very first programs were written before any tools existed, and how programmers knew what streams of 0s and 1s meant when there was no screen, no ASCII, and no encoding standard.

---

## The Decoder — What It Actually Does

Chapter 03 introduced the control unit and said it "decodes" instructions. Before going any further we need to understand exactly what decode means in hardware, because this is the foundation everything else rests on.

### One Gate Responding to One Pattern

Imagine you want a circuit that activates one specific output wire when it sees the bit pattern `1011` and stays silent for every other pattern.

```
Input bits: b3 b2 b1 b0

You want output = 1 ONLY when b3=1, b2=0, b1=1, b0=1

Circuit:
  b3 ───────────────────────┐
  b2 ───[NOT]───────────────┤
  b1 ───────────────────────┤──[AND]──── output
  b0 ───────────────────────┘

How it works:
  b3=1  passes directly        -> 1
  b2=0  inverted by NOT gate   -> 1  (NOT turns the 0 into a 1)
  b1=1  passes directly        -> 1
  b0=1  passes directly        -> 1
  AND of all four inputs       -> 1  (output fires)

Any other combination:
  At least one input to AND is 0
  AND output stays 0
  Wire stays silent
```

This AND gate with NOT gates on certain inputs is a **pattern detector**. It fires on exactly one binary pattern and ignores everything else. This is the fundamental building block of every decoder ever built.

### Scaling Up — How Many Patterns Are Possible

Before building a full decoder you need to know how many different patterns are possible for a given number of bits. Each bit can independently be 0 or 1, so every additional bit doubles the number of possible combinations.

```
1 bit:   0 or 1                               =  2 patterns  (2^1)
2 bits:  00, 01, 10, 11                       =  4 patterns  (2^2)
3 bits:  000, 001, 010, 011,
         100, 101, 110, 111                   =  8 patterns  (2^3)
4 bits:  0000 through 1111                    = 16 patterns  (2^4)
5 bits:  00000 through 11111                  = 32 patterns  (2^5)
6 bits:  000000 through 111111                = 64 patterns  (2^6)
7 bits:  0000000 through 1111111              = 128 patterns (2^7)
8 bits:  00000000 through 11111111            = 256 patterns (2^8)

The formula is always: 2 raised to the power of the number of bits
```

This is why an 8-bit opcode field gives exactly 256 possible patterns. It does not mean the CPU has 256 instructions — it means the CPU designer had 256 slots available to assign operations to. Some slots might be used, some might be left undefined or reserved for future use.

### The Full Decoder — One Detector Per Pattern

A CPU with an 8-bit opcode field needs up to 256 pattern detectors — one AND-gate circuit per possible pattern. All 256 evaluate simultaneously when an opcode arrives.

```
8-bit opcode arrives on 8 wires: b7 b6 b5 b4 b3 b2 b1 b0

Decoder (256 pattern detectors, each wired differently):

Pattern 00000000 detector ──► wire 0   (HALT operation)
Pattern 00111110 detector ──► wire 62  (LOAD A operation)
Pattern 10000000 detector ──► wire 128 (ADD operation)
Pattern 11000011 detector ──► wire 195 (JUMP operation)
... 252 more detectors ...

When opcode 00111110 arrives:
  Wire 0   silent
  Wire 62  FIRES  <── only this one
  Wire 128 silent
  Wire 195 silent
  ... all 252 others silent ...

Exactly one wire fires.
That wire connects to the circuit that performs that operation.
```

Each output wire from the decoder connects to the functional unit that performs that operation. Wire 62 firing enables the load-register-A circuit. Wire 128 firing signals the ALU to add. This is the complete mechanism of decoding — bit pattern in, one control wire out.

### One Instruction Executing — The Full Picture

```
Memory contains: 00111110  00000101
                 ^^^^^^^^  ^^^^^^^^
                 opcode    data (the value 5)

── FETCH ────────────────────────────────────────────────────
  Instruction pointer holds address of next instruction
  CPU reads byte 00111110 from that memory address
  Places it in the instruction register

── DECODE ───────────────────────────────────────────────────
  00111110 enters the decoder's 8 input wires
  256 pattern detectors evaluate simultaneously
  Only wire 62 fires (the LOAD A detector)
  Wire 62 activates:
    - "read next byte from memory" circuit
    - "register A write-enable" signal
    - "increment instruction pointer by 2" circuit

── EXECUTE ──────────────────────────────────────────────────
  Next byte (00000101 = 5) is read from memory
  Value 5 is driven into register A's flip-flops
  Instruction pointer advances by 2
  Register A now holds 5

── Back to FETCH ────────────────────────────────────────────
  Instruction pointer points to the next instruction
  Process repeats forever
```

The programmer in the 1940s who sat at a front panel with a notebook was doing manually what the decoder circuit does automatically. The programmer looked at `00111110` in the CPU manual, recognised it as LOAD A, and wrote "LD A" in their notes. The decoder circuit looks at `00111110` and fires the LOAD A wire. Same job. One done by a human brain. One done by AND gates.

---

## What Is an Opcode — Where the Word Came From

Opcode is short for **operation code**. The word operation describes what the instruction tells the CPU to do. The word code describes the binary encoding of that operation. Before the term was standardised, early programmers called them instruction codes, machine codes, or operation numbers. The term opcode solidified through the 1950s as the field matured.

An opcode is specifically the field of bits within an instruction that identifies **what operation to perform** — as distinct from the fields that identify **what data to operate on**. Instructions are not just one lump of bits. They have internal structure with distinct fields.

```
Example instruction: 10110000 01100001
                     ^^^^^^^^ ^^^^^^^^
                     opcode    operand

opcode  (10110000) = WHAT to do
  "Move an immediate byte value into register AL"

operand (01100001) = WHAT DATA to use
  01100001 = 97 decimal = ASCII 'a'
  "The value to move is 97"

Together: move the value 97 into register AL
```

### Instructions Have Multiple Fields

Not all instructions have the same field layout. Some instructions are just one byte — the opcode alone with no operand. Others are many bytes — opcode plus address fields plus immediate value fields. The CPU designer specifies the exact layout for every instruction.

```
No operand (instruction fully described by opcode alone):
  10000000 = ADD register B to register A
  One byte. Opcode IS the entire instruction.

One operand (opcode + one data byte):
  00111110 00000101
  ^^^^^^^^ ^^^^^^^^
  opcode   value 5
  Two bytes total.

Two operands (opcode + 16-bit address):
  11000011 00100000 00000000
  ^^^^^^^^ ^^^^^^^^ ^^^^^^^^
  opcode   addr low addr high
  Three bytes total. Address = 0x0020 = 32 decimal.
```

---

## The Intel 8080 and Z80 — The Real Architecture Behind the Examples

The opcode examples used throughout this chapter and Chapter 03 are real values from the **Intel 8080** (released 1974) and its compatible extension the **Zilog Z80** (released 1976). These were not invented for illustration. The actual Intel 8080 opcode table looks like this:

```
Intel 8080 / Z80 opcode table (selected entries):

Binary      Hex   Mnemonic      Operation
----------  ----  ------------  ------------------------------------------
00000000    0x00  NOP           Do nothing, advance instruction pointer
00111110    0x3E  LD A, n       Load immediate byte n into register A
00000110    0x06  LD B, n       Load immediate byte n into register B
00001110    0x0E  LD C, n       Load immediate byte n into register C
10000000    0x80  ADD A, B      Add register B to A, store result in A
10000001    0x81  ADD A, C      Add register C to A, store result in A
10000010    0x82  ADD A, D      Add register D to A, store result in A
10010000    0x90  SUB A, B      Subtract register B from A
10110000    0xB0  OR  A, B      Bitwise OR of B into A
10100000    0xA0  AND A, B      Bitwise AND of B into A
10101000    0xA8  XOR A, B      Bitwise XOR of B into A
00110010    0x32  LD (nn), A    Store register A to 16-bit address nn
11000011    0xC3  JP  nn        Jump unconditionally to 16-bit address nn
11001010    0xCA  JP  Z, nn     Jump to nn if zero flag is set
01111110    0x7E  LD  A, (HL)   Load A from address held in register HL
01110110    0x76  HALT          Stop execution
```

The 8080 has an 8-bit opcode field. This gives it 2^8 = 256 possible slots. The Intel 8080 uses all 256 — every pattern from 00000000 to 11111111 is assigned a defined operation. It is one of the cleanest examples of a fully populated 8-bit opcode table in computing history.

### Why the 8080 Opcodes Are Organised the Way They Are

Look at the ADD instructions in the table above:

```
10000000 = ADD A, B   (add register B to A)
10000001 = ADD A, C   (add register C to A)
10000010 = ADD A, D   (add register D to A)
10000011 = ADD A, E   (add register E to A)
10000100 = ADD A, H   (add register H to A)
10000101 = ADD A, L   (add register L to A)
10000110 = ADD A,(HL) (add memory at HL to A)
10000111 = ADD A, A   (add A to itself)
```

The top 5 bits `10000` identify the ADD operation family. The bottom 3 bits identify which register. This is not coincidence — the 8080 register encoding uses 3 bits throughout the instruction set:

```
3-bit register codes (used across many instructions):
  000 = B
  001 = C
  010 = D
  011 = E
  100 = H
  101 = L
  110 = (HL) — memory at address in HL register pair
  111 = A
```

The decoder has one circuit that fires when it sees `10000xxx` — then a smaller sub-decoder reads the bottom 3 bits to select which register. Eight ADD instructions share one top-level decoder output. This is intentional hardware economy — the designer organised the opcodes so the decoder could share logic between related operations.

---

## CPU Bit Width vs Opcode Width — A Critical Distinction

This is where most explanations go wrong and where confusion begins. The bit width in a CPU's name and the bit width of its opcodes are two completely separate design decisions. They describe different things entirely.

```
CPU "bit width" answers:           Opcode "bit width" answers:
  How wide are the registers?        How many bits identify an instruction?
  How large a number can it hold?    How many distinct instructions exist?
  How much RAM can be addressed?     How complex is the decoder?
  How wide is the data bus?          How are instruction bytes structured?

They are as unrelated as:
  The number of storeys in a building (address space)
  vs the number of rooms on each floor (instruction slots)
```

### What the CPU Bit Width Actually Means

**Register width** — how many bits each general-purpose register holds, and therefore the largest number the CPU can process in a single operation.

```
8-bit CPU (Intel 8080, 1974):
  Register A holds 8 bits
  Maximum value: 255
  Adding 200 + 100 requires handling overflow in software

16-bit CPU (Intel 8086, 1978):
  Register AX holds 16 bits
  Maximum value: 65,535

32-bit CPU (Intel 80386, 1985):
  Register EAX holds 32 bits
  Maximum value: 4,294,967,295

64-bit CPU (AMD Athlon 64, 2003):
  Register RAX holds 64 bits
  Maximum value: 18,446,744,073,709,551,615
```

**Address space** — how much RAM the CPU can address. Because addresses are stored in registers, the address space grows with register width.

```
8-bit address:  2^8  = 256 bytes of addressable RAM
16-bit address: 2^16 = 65,536 bytes (64 KB)
32-bit address: 2^32 = 4,294,967,296 bytes (4 GB)
                <- this is why 32-bit Windows capped at 4 GB RAM
64-bit address: 2^64 = 18 quintillion bytes theoretically
                (real CPUs implement 48-57 bits of this)
```

The jump from 32-bit to 64-bit is what allowed computers to use more than 4 GB of RAM. This is the single most practical consequence of bit width that most users ever experienced.

### What Opcode Width Actually Means — And How It Differs Per Architecture

The opcode field width determines how many distinct instructions are possible. A 64-bit CPU does not have 64-bit opcodes. The opcode encoding is a completely separate design decision. Two 64-bit CPUs can have completely different opcode encodings — and they do.

```
Architecture    Data width  Opcode encoding       Distinct instructions
--------------  ----------  --------------------  --------------------
Intel 8080      8-bit       always 1 byte         256 (all slots used)
Intel 8086      16-bit      1 to 3 bytes          ~300
Intel 80386     32-bit      1 to 3 bytes          ~400
Intel x86-64    64-bit      1 to 15 bytes         ~1,500-2,000
ARM64           64-bit      always 4 bytes        ~1,200
RISC-V RV64     64-bit      always 4 bytes        ~200 base + extensions
Apple M1        64-bit      always 4 bytes        ~1,200 (ARM64)

All four 64-bit architectures have completely different opcode encodings.
The 64-bit data width says nothing about how instructions are encoded.
```

---

## What Opcode Does a 64-bit CPU Use — Each Architecture Explained

### x86-64 — What Your Intel and AMD Processors Use

x86-64 uses **variable-length instructions** from 1 to 15 bytes. This is a direct consequence of 45 years of backward compatibility. The encoding grew organically as new instructions were added to old ones, never breaking compatibility with what came before.

```
x86-64 instruction structure:

[Prefix bytes] [Opcode 1-3 bytes] [ModRM] [SIB] [Displacement] [Immediate]
 0-4 bytes      1-3 bytes          0-1b    0-1b   0,1,2,4 bytes  0,1,2,4,8b

Total: 1 to 15 bytes per instruction

The opcode field itself is 1, 2, or 3 bytes depending on the instruction.
```

The x86-64 opcode space is organised in tables, each reached through escape bytes:

```
Primary opcode table (1 byte: 0x00 through 0xFF):
  256 slots
  0x90 = NOP (no operation)
  0xC3 = RET (return from function)
  0x50 = PUSH RAX
  0x0F = escape byte — "next byte is part of the opcode too"

0x0F escape table (2 bytes: 0x0F 0x00 through 0x0F 0xFF):
  256 more slots
  0x0F 0x05 = SYSCALL (enter kernel — cross into ring 0)
  0x0F 0xAF = IMUL    (signed multiply)
  0x0F 0x38 = escape  — "next byte is part of opcode too"

0x0F 0x38 table (3 bytes):
  256 more slots
  0x0F 0x38 0xF0 = MOVBE (move with byte swap)

0x0F 0x3A table (3 bytes):
  256 more slots

VEX and EVEX prefixed instructions (AVX, AVX-512 vector operations):
  Thousands more instructions for SIMD operations

Total x86-64 distinct instructions: roughly 1,500-2,000
```

A real x86-64 instruction as your CPU sees it:

```
C program:    int result = a + b;

GCC produces: 8B 45 F8   <- MOV EAX, [RBP-8]  (load variable a)
              03 45 FC   <- ADD EAX, [RBP-4]  (add variable b)
              89 45 F4   <- MOV [RBP-12], EAX (store to result)

Breaking down 8B 45 F8:
  8B     = opcode (1 byte, primary table slot 0x8B)
           "MOV register from memory"
  45     = ModRM byte — specifies EAX as destination,
           RBP-relative addressing with 8-bit displacement
  F8     = displacement = -8 in two's complement
           (variable a lives at RBP minus 8 bytes)

This is a 3-byte instruction: 1 opcode byte + 1 ModRM + 1 displacement
The CPU is 64-bit but the opcode is 1 byte.
```

### ARM64 (AArch64) — Phones, Apple Silicon, Raspberry Pi 4 and Above

ARM64 uses **fixed-length 32-bit instructions**. Every instruction is exactly 4 bytes — no variable length, no prefix bytes, no escape sequences. The decoder always knows exactly where each instruction starts and ends. This regularity makes the decoder simpler, faster, and more power-efficient.

```
ARM64 instruction: always 32 bits (4 bytes)

Example — ADD X0, X1, X2 (add registers X1 and X2, store in X0):
  Binary: 10001011 00000010 00000000 00100000
  Hex:    8B 02 00 20

Bit field breakdown:
  Bits 31-29: 100        <- sf=1 (64-bit), op=0, S=0 (no flags)
  Bits 28-24: 01011      <- identifies ADD register instruction
  Bits 23-22: 00         <- shift type (LSL)
  Bits 20-16: 00010      <- Rm = X2 (second source register)
  Bits 15-10: 000000     <- shift amount = 0
  Bits  9- 5: 00001      <- Rn = X1 (first source register)
  Bits  4- 0: 00000      <- Rd = X0 (destination register)

Every field has a fixed position in the 32-bit word.
The decoder logic is simple and uniform across all instructions.
```

The register fields (Rn, Rm, Rd) are always 5 bits wide — selecting from 32 possible registers (2^5 = 32). The opcode identification is spread across the top bits in a consistent pattern. This regularity is why ARM CPUs can be made much more power-efficient than x86 CPUs.

### RISC-V — The Open Architecture

RISC-V also uses fixed 32-bit instructions for the base ISA. Unlike ARM, the opcode field is explicitly defined as 7 bits in the low bits of every instruction.

```
RISC-V base instruction: always 32 bits

R-type instruction layout (register-to-register operations):
 31      25 24   20 19  15 14  12 11    7 6       0
  [funct7]  [rs2]  [rs1]  [funct3] [rd]  [opcode]
   7 bits    5 bits  5 bits  3 bits  5 bits  7 bits

opcode  field: 7 bits = 128 possible top-level patterns
funct3  field: 3 bits = 8 sub-operations per opcode
funct7  field: 7 bits = 128 further distinctions where needed

Example: ADD rd, rs1, rs2
  opcode  = 0110011 (register arithmetic)
  funct3  = 000     (add vs subtract distinguished here)
  funct7  = 0000000 (add, not subtract)

Example: SUB rd, rs1, rs2
  opcode  = 0110011 (same opcode — same instruction family)
  funct3  = 000     (same)
  funct7  = 0100000 (different — this makes it subtract)
```

The RISC-V design philosophy keeps the primary opcode count intentionally small and uses the funct fields for sub-operations. This makes the base decoder extremely simple while still allowing many distinct operations.


---

## The CPU Architecture — Front-End, Back-End and Every Component

The decoder does not work alone. It is one stage in a pipeline of hardware components. Understanding where the decoder sits requires seeing the whole CPU structure.

### The Physical Organisation of a Modern CPU

```
CPU CHIP
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  FRONT-END                                                  │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Fetch Unit                                          │   │
│  │    Reads instruction bytes from L1 I-cache           │   │
│  │    Uses RIP/PC to know where to fetch from           │   │
│  │    Handles branch prediction (predicts next address) │   │
│  │         ↓                                            │   │
│  │  Instruction Decoder                                 │   │
│  │    Recognises opcode patterns from byte stream       │   │
│  │    Determines: operation, source regs, dest regs     │   │
│  │    x86-64: pre-decoder finds boundaries first        │   │
│  │    Outputs: decoded instruction packets              │   │
│  │         ↓                                            │   │
│  │  Scheduler / Dispatch                                │   │
│  │    Holds decoded instructions in queue               │   │
│  │    Issues them to execution units when ready         │   │
│  │    Out-of-order: can reorder for efficiency          │   │
│  └──────────────────────────────────────────────────────┘   │
│                        ↓                                    │
│  BACK-END                                                   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Execution Units                                     │   │
│  │    ALU  — add, sub, AND, OR, XOR, compare            │   │
│  │    FPU  — floating point (32-bit and 64-bit floats)  │   │
│  │    SIMD — vector operations (SSE, AVX on x86)        │   │
│  │    AGU  — address generation for memory access       │   │
│  │         ↓                                            │   │
│  │  Registers                                           │   │
│  │    Ultra-fast storage wired directly to ALU          │   │
│  │    x86-64: RAX,RBX,RCX,RDX,RSI,RDI,RSP,RBP,R8-R15    │   │
│  │    ARM64: X0-X30, SP, XZR                            │   │
│  │    Results written here after execution              │   │
│  └──────────────────────────────────────────────────────┘   │
│                        ↓                                    │
│  MEMORY SUBSYSTEM                                           │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  L1 Cache (32-64KB, split I-cache and D-cache)       │   │
│  │  L2 Cache (256KB-1MB, per core)                      │   │
│  │  L3 Cache (8-64MB, shared across all cores)          │   │
│  │  MMU + TLB (virtual→physical address translation)    │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
         │ memory bus
         ▼
   RAM chips (DRAM — separate from CPU die)
```

Every box in this diagram is a physical circuit made from transistors and logic gates. Nothing is software. The decoder, ALU, registers, cache — all silicon.

---

## The Decoder — One Instruction Executed Step by Step

Take the x86-64 instruction `ADD EAX, EBX` — machine code bytes `01 D8`.

```
Step 1 — FETCH

  RIP (instruction pointer) holds address of this instruction
  Fetch unit reads bytes from L1 I-cache (or RAM if cache miss):
    01 D8
  Passes byte stream to decoder

Step 2 — DECODE

  Decoder receives bytes: 01 D8
  Byte 01: look up primary opcode table
    01 = ADD instruction, destination is r/m, source is register
  Byte D8: this is the ModRM byte
    Binary: 11 011 000
            ^^ ^^^ ^^^
            Mod=11 (register to register, no memory)
            Reg=011 = EBX (source register)
            RM=000  = EAX (destination register)

  Decoded meaning:
    Operation: ADD
    Source:    EBX register
    Dest:      EAX register

  Decoder outputs control signals:
    "Read EAX and EBX from register file"
    "Activate ALU add operation"
    "Write result to EAX"

Step 3 — REGISTER READ

  Register file circuits output:
    EAX value → wire A of ALU
    EBX value → wire B of ALU
  (Zero clock cycles — direct electrical connection)

Step 4 — EXECUTE (ALU)

  ALU receives two values on its input wires
  Full adder circuit computes the sum
  Result produced + flags updated:
    ZF (zero flag): 1 if result is 0
    CF (carry flag): 1 if addition overflowed
    SF (sign flag):  1 if result is negative
    OF (overflow):   1 if signed overflow

Step 5 — WRITE BACK

  Result written into EAX flip-flop circuits
  RIP incremented by 2 (length of this instruction)

Step 6 — BACK TO FETCH

  RIP now points to next instruction
  Process repeats — forever
```

### What the Assembly Programmer Knows About the Decoder

The assembly programmer does not control the decoder. They cannot modify it. It is fixed silicon. But they must understand what it does:

```
WHAT ASSEMBLY PROGRAMMER MUST KNOW:
  Which instructions exist in the ISA
  How they are encoded (opcode + ModRM + SIB + prefixes)
  How instructions affect flags
  What registers are available and their conventions
  How memory addressing modes work
  Calling conventions (which registers hold arguments)

WHAT ASSEMBLY PROGRAMMER DOES NOT CONTROL:
  The decoder circuit itself
  The ALU wiring
  Cache behaviour (directly)
  Pipeline stages
  Branch predictor hardware

WHAT ASSEMBLY PROGRAMMER IS AWARE OF:
  Instruction length (x86 variable 1-15 bytes affects decode throughput)
  Micro-ops: modern x86 CPUs decode one instruction into multiple
             internal micro-operations
             IMUL may become 3 micro-ops, simple MOV becomes 1
             Matters for performance-critical code
  Pipeline effects: instruction latency, decode throughput
```

Good analogy: the decoder is an automatic translator inside the CPU. The assembly programmer is the writer who knows the grammar. They do not build the translator but must understand how it interprets their instructions.

---

## The ISA Is a Specification — Hardware Implements It

The ISA is a document — a specification of rules. Hardware engineers build circuits that implement those rules.

```
ISA SPECIFICATION says:
  "When the decoder sees bytes 01 D8,
   it shall add the value in EBX to EAX
   and store the result in EAX"

HARDWARE ENGINEER builds:
  An AND-gate pattern detector that fires when 01 D8 is seen
  Control wires that activate the ALU add circuit
  Control wires that select EBX as source, EAX as destination
  Write-back path from ALU result to EAX flip-flops

RESULT:
  Silicon that faithfully executes the ISA specification
  When 01 D8 arrives at the decoder, EAX = EAX + EBX happens
  Every time, unconditionally, in hardware
```

The ISA specification includes:

```
1. Instruction definitions
   Name, encoding, behaviour, flags affected, exceptions raised

2. Register set
   Names, widths, special purposes (RIP, RFLAGS, CR0-CR4)

3. Memory model
   Addressing modes, alignment requirements, byte ordering

4. Privilege model
   Rings (x86) or Exception Levels (ARM)
   Which instructions require ring 0

5. Interrupt and exception behaviour
   What happens on page fault, divide by zero, etc.

6. Calling conventions (ABI)
   Which registers hold arguments, return values
   Stack alignment requirements
```

---

## Algorithms vs Primitive ISA Instructions — The Critical Separation

This is one of the most important distinctions in computing:

```
CPU ISA DEFINES:           PROGRAMS DEFINE:
  move data                  video decoding algorithm
  add numbers                scheduling algorithm
  compare values             HTML rendering algorithm
  jump somewhere             matrix multiplication
  call function              sorting algorithm
  push/pop stack             encryption algorithm
  read/write memory          network protocol

CPU never knows:           Programs encode:
  what a video is            the meaning of video
  what a kernel is           what to do when scheduling
  what a browser is          how to parse HTML
  what AI is                 what intelligence means
```

The CPU is a universal executor. It executes whatever bytes are in memory according to the ISA. The meaning of those bytes — what algorithm they implement — comes entirely from the programmer and compiler, not from the hardware.

```
Example: FFmpeg video decoding

FFmpeg source code contains the algorithm:
  for each macroblock:
    reconstruct colour from DCT coefficients
    apply motion compensation

Compiler converts this to ISA instructions:
  mov eax, [block_ptr]
  imul eax, [dct_coeff]
  add [output], eax
  ...

CPU sees:
  MOV, IMUL, ADD, MOV...

CPU does NOT know:
  what a macroblock is
  what DCT means
  that it is decoding video

The "video intelligence" exists only in the algorithm written by humans
and translated by the compiler.
```

---

## The Kernel — Same Decoder, No Magic

The kernel is not special from the CPU's perspective. The CPU decoder does not have a "kernel mode decoder" and a "user mode decoder." There is one decoder. It decodes all machine code the same way.

```
SAME CPU DECODER handles:

Linux kernel        → same decoder
nginx web server    → same decoder
bash shell          → same decoder
FFmpeg              → same decoder
your programs       → same decoder
OP-TEE TEE OS       → same decoder (in Secure World)

The decoder does NOT know what program it is executing.
It only sees bytes and decodes them according to ISA rules.
```

### What Makes the Kernel Special

```
KERNEL IS SPECIAL because of:

1. Ring 0 privilege (CPL=00 in CS register)
   Hardware checks this for every privileged instruction
   CLI (disable interrupts): allowed only in ring 0
   MOV CR3, RAX (change page table): ring 0 only
   LGDT (load GDT): ring 0 only
   IN/OUT (hardware ports): ring 0 only

2. The kernel runs the same ISA instructions as everything else
   But it is ALLOWED to run the privileged ones
   User programs executing the same bytes get a General Protection Fault

KERNEL IS NOT SPECIAL because of:
   Any different decoding
   Any different instruction set
   Any magical hardware understanding
   Any special memory access technology
```

Kernel source code compiled:

```
Kernel C code:
  current->state = TASK_RUNNING;

Compiler generates:
  mov rax, [current]
  mov dword [rax+0x10], 1

Assembler produces:
  48 8B 05 xx xx xx xx
  C7 40 10 01 00 00 00

CPU decoder sees:
  MOV (64-bit load)
  MOV (32-bit store with displacement)

Decoder does not know this is the scheduler.
It just decodes the bytes.
```

---

## The Bootstrap History — How Compilers Were Born

Understanding how compilers came to exist resolves a fundamental chicken-and-egg confusion.

### The Problem

A compiler is a program. Programs need to be compiled. Who compiled the first compiler?

### The Answer — Four Stages

```
STAGE 1 — 1940s: Machine Code Written By Hand

  Programmers consulted the CPU manual
  Calculated binary opcodes on paper
  Toggled binary into memory via front panel switches

  Example:
    Programmer wants: move value 5 into register A
    Manual says: opcode = 00111110, then the value byte
    Programmer sets switches: 00111110
    Presses DEPOSIT
    Sets switches: 00000101 (value 5)
    Presses DEPOSIT
    Repeats for every byte of the entire program

  The programmer's brain WAS the assembler.
  The programmer's notebook WAS the opcode table.

STAGE 2 — Early 1950s: First Assembler (Written in Machine Code)

  Someone wrote the first assembler as raw machine code
  Calculated every byte by hand from the ISA manual
  Toggled it into memory via switches
  The assembler did ONE thing:

    if input text is "MOV" → output opcode 00111110
    if input text is "ADD" → output opcode 10000000
    ...etc...

  It was a lookup table with a loop.
  No intelligence. Just pattern matching.

  Now programmers could type MOV instead of 00111110.

STAGE 3 — Assembler Written in Assembly

  With the first assembler running, someone wrote
  a better assembler in assembly language
  Fed it through the first assembler
  Got a better assembler binary
  Discarded the hand-toggled original

  Bootstrapping complete: assembler assembles itself.

STAGE 4 — First Compiler (Written in Assembly)

  Someone wrote a compiler for a simple language
  Written in assembly — a few hundred assembly instructions
  The compiler logic was simple rule-based matching:

    if token == '+' → emit ADD instruction
    if token == '-' → emit SUB instruction
    if token == identifier → emit LOAD instruction

  Assembled the compiler source with the assembler
  Got a compiler binary

  Now programmers could write:
    a = b + c;
  Instead of:
    MOV EAX, [b]
    ADD EAX, [c]
    MOV [a], EAX

STAGE 5 — Self-Hosting

  Compiler written in its own language
  Compiled by the previous version
  Each version more powerful than the last

  GCC today:
    Written in C
    Compiled by an older GCC
    That GCC was compiled by an even older GCC
    ...chain traces back to hand-toggled machine code...
```

### What a Compiler Actually Is — No Magic

A compiler is not intelligent. It is a rule-based transformation system:

```
COMPILER PIPELINE (every real compiler):

Source text (human language)
         |
         v
LEXER (tokeniser):
  Reads characters one at a time
  Groups them into tokens
  Uses finite automata (Chapter 07)
  "int" → KEYWORD_INT
  "x"   → IDENTIFIER
  "="   → EQUALS
  "5"   → INTEGER_LITERAL

         |
         v
PARSER:
  Takes token stream
  Recognises grammatical structure
  Uses context-free grammar (Chapter 07)
  Builds Abstract Syntax Tree (AST):
         ASSIGN
        /      \
       x        5

         |
         v
SEMANTIC ANALYSIS:
  Type checking
  Scope resolution
  Ensures x is declared, 5 is compatible type

         |
         v
IR GENERATION:
  Converts AST to Intermediate Representation
  LLVM IR, GCC GIMPLE — platform-neutral form
  %1 = alloca i32      ; allocate space for x
  store i32 5, i32* %1 ; store value 5

         |
         v
OPTIMISER:
  Transforms IR for performance
  Constant folding: 5+3 → 8 at compile time
  Dead code elimination: remove unused variables
  Inlining: replace function call with function body

         |
         v
CODE GENERATOR:
  Translates IR to target machine's instructions
  x86-64: mov DWORD PTR [rbp-4], 5
  ARM64:  str wzr, [x29, -4]  ; then str #5

         |
         v
ASSEMBLER (internal):
  Converts assembly text to machine code bytes
  x86-64: C7 45 FC 05 00 00 00

         |
         v
LINKER:
  Combines multiple object files
  Resolves cross-file references
  Produces final ELF executable

         |
         v
ELF binary on disk — machine code ready for execution
```

### Where Does Compiler Code Live in Memory?

```
When the compiler runs (e.g. gcc source.c):

DISK:
  /usr/bin/gcc → ELF binary file containing compiler machine code

OS LOADS IT:
  Kernel reads ELF header
  Creates virtual address space for gcc process
  Maps .text section (compiler code) to virtual pages
  Maps .data section (compiler data) to virtual pages
  Sets up stack and heap

RAM:
  ┌──────────────────────────────────┐
  │  compiler machine code (.text)   │ ← the compiler's own instructions
  │  compiler data (.data)           │ ← tables, string constants
  │  input buffer                    │ ← your source file being read
  │  AST in heap                     │ ← dynamically allocated tree
  │  output buffer                   │ ← machine code being generated
  │  stack                           │ ← compiler's own function calls
  └──────────────────────────────────┘

CPU EXECUTES THE COMPILER:
  Fetch compiler instruction
  Decode it (same decoder as always)
  Execute it (reads your source, builds AST, emits machine code)
  Repeat

The compiler is not special.
It is just a program that reads text and writes bytes.
Those bytes happen to be machine code for your program.
```

---

## The Complete ISA-to-Execution Flow

```
ISA SPECIFICATION (document, not hardware or software):
  "Opcode 01 D8 means ADD EAX, EBX"
  "Executing this in ring 3 is allowed"
  "Flags ZF/CF/SF/OF are updated"
         |
         | hardware engineers implement this
         v
HARDWARE (silicon, transistors, gates):
  Pattern detector for opcode 01: AND gate tree
  Control wire connected to ALU add mode
  Multiplexer selects EBX → ALU input B
  Multiplexer selects EAX → ALU input A
  Write-back path ALU result → EAX flip-flops
         |
         | software is compiled to use this hardware
         v
MACHINE CODE (bytes in memory):
  01 D8
  The bytes that activate the hardware above
         |
         | assembly language is the human-readable form
         v
ASSEMBLY LANGUAGE (text):
  ADD EAX, EBX
  One-to-one with machine code
  Assembler converts text → bytes
         |
         | C is a higher abstraction
         v
C SOURCE CODE (text):
  a = a + b;
  Compiler converts to assembly then machine code
         |
         | the compiler is itself a compiled program
         v
COMPILER (ELF binary running in RAM):
  gcc reads "a = a + b;"
  Applies rules: assignment, addition → ADD instruction
  Outputs bytes: 01 D8
  Uses same CPU decoder as everything else
         |
         | formal language theory guides compiler design
         v
FORMAL LANGUAGE THEORY (mathematics):
  Regular expressions → lexer design (Chapter 07)
  Context-free grammars → parser design (Chapter 07)
  Finite automata → lexer implementation
  Pushdown automata → parser implementation
  NOT hardware. NOT software. Mathematical framework.
         |
         | ultimately everything runs as
         v
CPU EXECUTION:
  Fetch → Decode → Execute → Write Back → Repeat
  The decoder fires the correct control wires
  The ALU computes the result
  The registers hold the values
  The MMU translates every address
  The cache reduces memory latency
  Nothing above this is in hardware
  Nothing below this is in software
```

---

## x86-64 Encoding — A Complete Explanation

The previous section showed ARM64 and RISC-V are simple to decode because they were designed with clean fixed-length encodings. x86-64 is harder to understand — not because the concept is different but because 45 years of backward-compatible additions created layers that must be understood in historical order to make sense.

### Why x86-64 Is Variable Length — The 1978 Foundation

The Intel 8086 was designed in 1978 when RAM cost thousands of dollars per kilobyte. Every byte of instruction code cost money. The designer made a deliberate choice: **common operations get the shortest possible encodings, rare operations get longer encodings.**

```
1-byte instructions (most common operations):
  0x90 = NOP          do nothing
  0xC3 = RET          return from function
  0x50 = PUSH RAX     push register onto stack
  0x40 = INC AX       increment register

2-byte instructions (operations needing a register pair):
  0x89 0xC3 = MOV BX, AX    copy AX into BX
  0x01 0xC3 = ADD BX, AX    add AX to BX

3-6 byte instructions (operations needing data values):
  0xB8 0x05 0x00 0x00 0x00 = MOV EAX, 5
       ^^^^^^^^^^^^^^^^^^^^
       the value 5 as 4 bytes
```

This compression was intentional and logical for 1978. The problem came when more instructions were needed and the encoding had to be extended without breaking existing software.

### How x86 Grew — Seven Layers of Prefixes

Every time Intel or AMD needed to add new capabilities, they added a new prefix byte — a byte that comes before the opcode and modifies its meaning. Each layer is a response to a specific need:

```
Layer 1 — Original 8086 (1978):
  Segment override prefixes: 0x26, 0x2E, 0x36, 0x3E
  Lock prefix: 0xF0 (atomic memory operations)
  Repeat prefix: 0xF3 (repeat string operations)

  Example: 0xF3 0xA4 = REP MOVSB
           ^^^^ ^^^^
           repeat  copy-one-byte opcode

Layer 2 — 80386 extends to 32-bit (1985):
  Problem: same opcode 0x89 needs to work in 16-bit AND 32-bit mode
  Solution: operand size prefix 0x66 toggles between widths

  Without 0x66:  0x89 0xC3 = MOV EBX, EAX  (32-bit)
  With    0x66:  0x66 0x89 0xC3 = MOV BX, AX   (16-bit)
  Same opcode 0x89. Prefix changes what it means.

  Address size prefix 0x67: switches between 16 and 32-bit addresses.

Layer 3 — 0x0F escape byte (added progressively 1985-2000s):
  Problem: primary opcode table (256 slots) getting full
  Solution: reserve 0x0F as escape meaning "next byte is also opcode"

  0x0F opens a second 256-slot table:
    0x0F 0xAF = IMUL   signed multiply
    0x0F 0x05 = SYSCALL enter kernel (critical for Chapter 07)
    0x0F 0xBE = MOVSX  sign-extend byte to register
    0x0F 0x38 = escape again into a third table
    0x0F 0x3A = escape into a fourth table

Layer 4 — REX prefix (AMD64, 2003):
  Problem: 64-bit mode needs 8 new registers and 64-bit operand size
  Solution: REX prefix byte = 0x40 through 0x4F

  REX byte structure (8 bits):
    0  1  0  0  W  R  X  B
    ^^^^^^^^^^^
    fixed pattern identifying this as REX

    W bit: 0 = 32-bit operands, 1 = 64-bit operands
    R bit: extends the Reg field in ModRM by 1 bit (accesses R8-R15)
    X bit: extends the Index field in SIB by 1 bit
    B bit: extends the RM field in ModRM or opcode register field

  Examples:
    0x89 0xC3         = MOV EBX, EAX    (32-bit, no REX)
    0x48 0x89 0xC3    = MOV RBX, RAX    (64-bit, REX.W=1)
    0x4C 0x89 0xC3    = MOV RBX, R8     (64-bit, REX.R=1 extends reg field)
    ^^^^
    0x48 = 0100 1000
           ^^^^ W=1 R=0 X=0 B=0

Layer 5 — VEX prefix (2011, AVX instructions):
  Problem: need 256-bit YMM registers, REX cannot encode them cleanly
  Solution: VEX is 2 or 3 bytes that replace REX + 0x0F escapes

  2-byte VEX: 0xC5 followed by one control byte
  3-byte VEX: 0xC4 followed by two control bytes

  0xC5 0xFC 0x58 0xC8 = VADDPS YMM1, YMM0, YMM1
  (add 8 floats simultaneously using 256-bit registers)

Layer 6 — EVEX prefix (2017, AVX-512):
  Problem: need 512-bit ZMM registers and 32 of them (R0-R31)
  Solution: EVEX is mandatory 4-byte prefix for all AVX-512 instructions

  0x62 [P1] [P2] [P3] [opcode] [ModRM] ...
  Encodes: register width, opmask registers, rounding control,
           zero-masking, broadcast, all 32 ZMM registers

Full instruction with all parts possible:
  [Legacy prefix 0-4 bytes][REX 0-1 byte][Opcode 1-3 bytes]
  [ModRM 0-1 byte][SIB 0-1 byte][Displacement 0-4 bytes][Immediate 0-8 bytes]
  Total: minimum 1 byte, maximum 15 bytes
```

### The ModRM Byte — Encoding Two Operands in Eight Bits

Most x86-64 instructions that reference memory or use two registers include a **ModRM byte** immediately after the opcode. This single byte encodes three fields:

```
ModRM byte layout:

  Bit:  7  6   5  4  3   2  1  0
       ┌────┬────────┬────────┐
       │ Mod│  Reg   │   RM   │
       │ 2b │  3b    │  3b    │
       └────┴────────┴────────┘

Mod field (2 bits) — what RM means:
  11 = RM is a register directly (register-to-register)
  00 = RM register is used as a memory address (no displacement)
  01 = RM register + 8-bit signed displacement added to address
  10 = RM register + 32-bit signed displacement added to address

Reg field (3 bits) — first operand register:
  000=RAX  001=RCX  010=RDX  011=RBX
  100=RSP  101=RBP  110=RSI  111=RDI
  (REX.R bit adds a 4th bit, allowing R8 through R15)

RM field (3 bits) — second operand register or address base:
  Same register encoding as Reg field
  Special cases:
    RM=100 with Mod≠11: SIB byte follows (complex addressing)
    RM=101 with Mod=00: RIP-relative addressing (used in PIE binaries)
```

### Decoding a Real Instruction Byte by Byte

```
Bytes: 48 89 45 F8

Byte 1: 0x48 = REX prefix
  Binary: 0100 1000
          ^^^^ W R X B
          W=1: use 64-bit registers
          R=0, X=0, B=0: no register extension needed

Byte 2: 0x89 = opcode
  Primary table slot 0x89 = "MOV r/m64, r64"
  Meaning: copy a register (r64) into register or memory (r/m64)
  The Reg field in ModRM is the SOURCE
  The RM field in ModRM is the DESTINATION

Byte 3: 0x45 = ModRM byte
  Binary: 01 000 101
          ^^ ^^^ ^^^
          Mod=01: RM register + 8-bit displacement
          Reg=000: RAX is the source register
          RM=101:  RBP is the base address register

Byte 4: 0xF8 = displacement
  Signed 8-bit: 0xF8 = -8

Final decoded instruction:
  MOV [RBP-8], RAX
  Store RAX into memory at address (RBP minus 8)
  This saves a local variable onto the function's stack frame
  GCC generates this for every local variable in C code
```

### The SIB Byte — For Array and Complex Addressing

When ModRM's RM field is `100` (binary), a **SIB (Scale-Index-Base) byte** follows. SIB encodes array indexing with scaling:

```
SIB byte layout:

  Bit:  7  6   5  4  3   2  1  0
       ┌─────┬────────┬────────┐
       │Scale│ Index  │  Base  │
       │ 2b  │  3b    │  3b    │
       └─────┴────────┴────────┘

Scale: multiply index register by this amount
  00=x1  01=x2  10=x4  11=x8

Effective address = Base + (Index x Scale) + Displacement

C code:           int x = array[i];
With int = 4b:    x = *(array_base + i * 4)

Assembly:         MOV EAX, [RBX + RSI*4]
  RBX = base address of array
  RSI = index variable i
  Scale = x4 (each int is 4 bytes)

ModRM: Mod=00, Reg=EAX, RM=100 (SIB follows)
SIB:   Scale=10(x4), Index=RSI, Base=RBX

Entire array access encoded in: opcode(1) + ModRM(1) + SIB(1) = 3 bytes
```

### A Complete C Function Decoded

```c
int add(int a, int b) {
    return a + b;
}
```

GCC produces (unoptimised, x86-64 Linux calling convention):

```
Offset  Hex bytes      Assembly              Explanation
------  -----------    --------------------  ----------------------------------
+0000   55             PUSH RBP              save caller's frame pointer
+0001   48 89 E5       MOV RBP, RSP          set up this function's frame
+0004   89 7D FC       MOV [RBP-4], EDI      save argument a (EDI) to stack
+0007   89 75 F8       MOV [RBP-8], ESI      save argument b (ESI) to stack
+000A   8B 45 FC       MOV EAX, [RBP-4]      load a from stack into EAX
+000D   03 45 F8       ADD EAX, [RBP-8]      add b from stack to EAX
+0010   5D             POP RBP               restore caller's frame pointer
+0011   C3             RET                   return (result is in EAX)

Decoding key instructions:

55 = PUSH RBP
  1-byte instruction. Low 3 bits of 0x50-0x57 encode register.
  0x55 = 0x50 + 5 = RBP. No ModRM needed.

48 89 E5 = MOV RBP, RSP
  0x48 = REX (W=1 for 64-bit)
  0x89 = MOV r/m64, r64
  0xE5 = ModRM: Mod=11(register), Reg=100(RSP source), RM=101(RBP dest)

89 7D FC = MOV [RBP-4], EDI
  0x89 = MOV r/m32, r32 (no REX = 32-bit operands)
  0x7D = ModRM: Mod=01(+disp8), Reg=111(EDI source), RM=101(RBP base)
  0xFC = displacement: -4 signed

03 45 F8 = ADD EAX, [RBP-8]
  0x03 = ADD r32, r/m32
  0x45 = ModRM: Mod=01(+disp8), Reg=000(EAX dest), RM=101(RBP base)
  0xF8 = displacement: -8 signed

C3 = RET
  1-byte instruction. Always pops return address from stack and jumps there.
  This is opcode 0xC3 — the byte that every ROP gadget ends with.
```

### How the x86-64 CPU Handles Variable Length

ARM64's decoder receives 32 bits and knows immediately it is one complete instruction. x86-64's decoder receives a byte stream and must determine instruction boundaries before it can decode anything.

Modern Intel and AMD CPUs use a two-stage approach:

```
Stage 1 — Pre-decoder (runs ahead of main decoder):

Receives raw byte stream from instruction cache.
Determines instruction boundaries by scanning for:
  - Is this byte a legacy prefix? Mark it, advance.
  - Is this byte REX (0x40-0x4F)? Mark it, advance.
  - Is this byte 0x0F? Two-byte opcode, read one more.
    Is next byte 0x38 or 0x3A? Three-byte opcode, read one more.
  - Now we have the opcode. Look up in pre-decoder table:
    Does this opcode require ModRM? Read it.
    Does ModRM indicate SIB (RM=100, Mod!=11)? Read it.
    Does Mod indicate displacement? Read 1 or 4 bytes.
    Does opcode require immediate? Read 1,2,4, or 8 bytes.
  - Mark the end of this instruction.
  - Next byte is start of next instruction.

Output: stream of instruction packets with boundaries marked.

Stage 2 — Main decoder:

Receives pre-identified instruction packets.
Decodes each into internal micro-operations (uops).
Sends uops to execution units.
Execution units never see x86-64 bytes — only uops.

This is why modern Intel CPUs can decode 4-6 instructions per cycle
despite variable-length encoding — the pre-decoder pipeline
runs well ahead and absorbs the complexity.
```

### Side by Side — The Same ADD in All Three Architectures

```
Operation: add two 64-bit registers, store result in first

x86-64 (Intel/AMD):
  Bytes:   48 01 F8
  Length:  3 bytes
  Decoded: REX.W=1, opcode 0x01 (ADD r/m64,r64),
           ModRM 0xF8 (Mod=11, Reg=RDI, RM=RAX)
  Meaning: ADD RAX, RDI

ARM64 (Apple Silicon, phones, Pi):
  Bytes:   00 00 01 8B
  Length:  4 bytes (always)
  Binary:  1000 1011 0000 0001 0000 0000 0000 0000
  Fields:  op=ADD, Rd=X0, Rn=X0, Rm=X1
  Meaning: ADD X0, X0, X1

RISC-V RV64 (open architecture):
  Bytes:   33 04 B5 00
  Length:  4 bytes (always)
  Binary:  0000 0000 1011 0101 0000 0100 0011 0011
  Fields:  opcode=ADD, rd=x8, rs1=x10, rs2=x11
  Meaning: ADD x8, x10, x11

Key differences:
  x86-64: 3 bytes, complex multi-field decode, REX for 64-bit
  ARM64:  4 bytes, fixed layout, fields at fixed bit positions always
  RISC-V: 4 bytes, fixed layout, opcode always bits 6-0
```

### Why This Complexity Is Not Going Away

x86-64's encoding complexity exists for one reason only: backward compatibility. Every instruction that ran on an 8086 in 1978 still runs on a 2024 Intel Core i9 with identical bytes. The encoding could never be simplified without breaking that promise.

```
Alternative that Intel could have taken (but did not):

"Starting with the next CPU, all programs must be recompiled.
We are using a clean fixed-length encoding like ARM."

Consequences:
  - Every operating system must be recompiled
  - Every application must be recompiled
  - Every library must be recompiled
  - Any program without source code is permanently lost
  - The transition period requires running two instruction sets
  - Customers must replace all software

Intel looked at this cost and chose to extend the existing encoding
instead. They made this choice repeatedly, every few years,
from 1982 to 2017.

AMD looked at Intel's choice in 2003 when designing AMD64 and
made the same decision — extend x86 rather than start clean.

ARM in 1985 had no installed base to protect.
They designed for simplicity.

RISC-V in 2010 had no installed base to protect.
They designed for simplicity.

x86-64 is the price of 45 years of unbroken backward compatibility.
It is a reasonable price. Every program ever compiled for x86 still works.
But the encoding complexity is real and permanent.
```


---

## Why the 8080 Had 8-bit Opcodes — The Coincidence Explained

The Intel 8080 happened to have both 8-bit registers and 8-bit opcodes. This coincidence caused lasting confusion about the relationship between the two. The reasons the 8080 used 8-bit opcodes had nothing to do with its 8-bit registers:

**Memory was extraordinarily expensive.** In 1974, RAM cost thousands of dollars per kilobyte. One-byte instructions were compact and conserved precious memory. Multi-byte opcodes would have wasted memory unaffordably.

**256 slots was enough.** The 8080 needed roughly 200 distinct operations. 256 slots fit them all with room to spare. No need for wider opcodes.

**The memory bus was 8 bits wide.** The 8080 fetched one byte per memory cycle. A 1-byte instruction was fetched in exactly one cycle. Longer instructions required additional fetch cycles, and the designers wanted instructions that fit naturally in one fetch where possible.

When Intel designed the 8086 to have 16-bit registers, they kept the 8-bit primary opcode structure for backward compatibility but added the 0x0F escape byte to open a second table. The register width grew to 16 bits. The opcode encoding remained 1-3 bytes. The two dimensions were already separated.

```
The coincidence that caused confusion:

Intel 8080 (1974):
  Registers: 8 bits wide
  Opcodes:   8 bits wide   <- COINCIDENCE, not a rule
  Result:    people assumed bit width = opcode width

Intel 8086 (1978):
  Registers: 16 bits wide
  Opcodes:   still 1-3 bytes  <- shows they were always separate

Intel 80386 (1985):
  Registers: 32 bits wide
  Opcodes:   still 1-3 bytes  <- confirmed as separate

AMD x86-64 (2003):
  Registers: 64 bits wide
  Opcodes:   1 to 15 bytes    <- completely separate
```

---

## Before ASCII — How Programmers Knew What Each Stream of Bits Meant

ASCII was standardised in 1963. The first stored-program computers ran in the late 1940s. For 15 years, computers operated with no standard text encoding at all. Yet programmers used them daily without confusion.

The reason: **the CPU manual was the dictionary, and the programmer's brain and notebook were the decoder.**

The programmer did not need a text encoding system to understand what `10000000` meant. They needed the Intel 8080 manual (or whatever CPU they were using) which said:

```
Binary      Hex   Operation
10000000    80    ADD A, B   <- add register B to register A
```

The programmer memorised this or kept the manual on the desk. When they wanted to add two registers, they wrote `10000000` in their notebook. The machine received the binary — it had no interest in what the programmer called it.

```
A programmer's working notes on an 8080 machine, 1975:

Address  Binary      Personal notation
-------- ----------  ------------------------------------------
0000     00111110    LD A,    <- load next byte into register A
0001     00000101    05       <- the value 5
0002     00000110    LD B,    <- load next byte into register B
0003     00000011    03       <- the value 3
0004     10000000    ADD      <- add B to A
0005     00110010    ST       <- store A to address that follows
0006     01100100    64h      <- address low byte (100 decimal)
0007     00000000    00h      <- address high byte

"Personal notation" column = for the human only
Machine receives only the binary values
Different programmers used different personal notations
The binary opcode values were the only universal language
```

Over time, teams agreed on standard notation for their machine. That agreement became the assembly language. Someone then wrote the first assembler — a program that read the standard notation and output the binary opcodes automatically.

---

## Front Panel Switches — The Keyboard of the 1940s and 1950s

Before screens, before teletypes, before keyboards as we know them — the only way to enter a program was physically toggling binary values into memory using switches on the front panel of the computer.

```
Front panel of an early computer:

ADDRESS SWITCHES (set the memory address to write to):
 [0][0][0][0][0][0][0][0][0][0][0][0]
  down=0 up=1, 12 switches for 12-bit address

DATA SWITCHES (set the binary value to write):
 [0][0][1][1][1][1][1][0]
  set these to 00111110 to write the LOAD A opcode

CONTROL BUTTONS:
 [LOAD ADDR] [DEPOSIT] [EXAMINE] [RUN] [STOP]

INDICATOR LIGHTS (show current memory or register state):
  O  O  *  *  *  *  *  O
  0  0  1  1  1  1  1  0   <- read these as binary = 00111110
```

### Entering One Instruction — The Physical Sequence

```
Goal: write the 8080 instruction LD A, 5
      which is two bytes: 0x3E (00111110) then 0x05 (00000101)
      starting at memory address 0

Step 1: Set address switches to 000000000000 (address 0)
Step 2: Press LOAD ADDR

Step 3: Set data switches to 00111110
        switch 7: down (0)
        switch 6: down (0)
        switch 5: up   (1)
        switch 4: up   (1)
        switch 3: up   (1)
        switch 2: up   (1)
        switch 1: down (0)
        switch 0: down (0)
Step 4: Press DEPOSIT
        Byte 00111110 written to address 0000
        Address register automatically increments to 0001

Step 5: Set data switches to 00000101
        switch 7: down (0)
        switch 6: down (0)
        switch 5: down (0)
        switch 4: down (0)
        switch 3: down (0)
        switch 2: up   (1)
        switch 1: down (0)
        switch 0: up   (1)
Step 6: Press DEPOSIT
        Byte 00000101 written to address 0001
        Address increments to 0002

Continue for every byte of the entire program.
Then set address switches back to 0, press LOAD ADDR, press RUN.
```

This was the daily workflow for years. A 100-instruction program required 200-300 DEPOSIT presses. Skilled operators memorised the bit patterns and became fast. Reading the indicator lights to debug — reading binary from lit and unlit bulbs and looking them up in the manual — was a normal skill.

---

## The Evolution — Five Stages From Human Decoder to Modern CPU

The entire history of programming tools is the story of automating the decoding step that the programmer originally performed by hand.

### Stage 1 — 1940s: The Human Brain Is the Decoder

```
ISA manual open on the desk
Programmer's brain: looks up binary pattern, decides what it means
Programmer's notebook: translates intent to binary by hand
Programmer's hands: toggle each byte into memory via front panel switches

The programmer IS the assembler.
The programmer IS the decoder.
The notebook IS the symbol table.
```

### Stage 2 — Late 1940s: Decoder Moves Into Hardware

```
CPU designer wires AND-gate pattern detectors in the control unit
When 00111110 arrives, wire 62 fires automatically
Wire 62 enables the load-register-A circuit

Decoding now happens automatically in hardware.
But programmer still calculates binary by hand from the manual.
Programmer is still the assembler — software has not yet taken over.
```

### Stage 3 — Early 1950s: Software Assembler Takes Over

```
First assembler bootstrapped by hand and toggled into memory.
Programmer writes:    LDA #5
Assembler reads characters, compares to its mnemonic table.
When it sees "LDA" it outputs 00111110.
When it sees "#5"  it outputs 00000101.

Human brain is no longer the decoder for opcodes.
The assembler program decodes the mnemonic notation into binary.
The hardware decoder decodes the binary into control signals.
Two automatic decoding layers.
```

### Stage 4 — 1972: Compiler Becomes the Top Layer

```
C compiler reads:    int x = a + b;
Compiler decides how to implement this for the target ISA.
Produces assembly:   MOV EAX, [RBP-8]
                     ADD EAX, [RBP-4]
                     MOV [RBP-12], EAX
Internal assembler converts to opcodes: 8B 45 F8 / 03 45 FC / 89 45 F4

Three automatic decoding layers:
  C source -> compiler -> assembly mnemonics
  Assembly mnemonics -> assembler -> opcode bytes
  Opcode bytes -> hardware decoder -> control signals
```

### Stage 5 — Modern CPUs: Microcode Adds One More Layer

```
Modern x86-64 CPUs add a fourth decoding layer.
The x86-64 instruction encoding is complex and variable-length.
Decoding it directly at full speed is power-hungry.

Modern solution:
  Pre-decoder identifies instruction boundaries in the byte stream
  Main decoder translates x86-64 opcodes into internal micro-ops (uops)
  uops are a cleaner fixed-length internal instruction set
  Execution units process only uops, never x86-64 bytes directly

Full chain from C to electrons:

C source
  | compiler decodes
  v
x86-64 opcode bytes
  | pre-decoder + main decoder decode
  v
Internal micro-operations (uops)
  | execution unit decode
  v
Control signals to ALU, registers, cache
  | gates respond
  v
Transistors switch
  | electrons flow through silicon
  v
Result

Four decoding layers between your C code and electrons.
Every layer is automatic.
The programmer sees only the top layer.
```

---

## The First Assembler — Bootstrapped From Human Hands

With the front panel switch method and the evolution of decoding understood, the bootstrapping of the first assembler makes complete sense.

### Teams Agree on Standard Notation

Different programmers at one institution had been using personal shorthand. The team standardised: one official mnemonic per opcode, agreed by everyone. This agreement is the assembly language for that machine.

```
Intel 8080 agreed standard notation (subset):

Mnemonic    Binary opcode   Operation
--------    -------------   ---------------------------------
LD A, n     00111110 n      Load immediate byte n into A
LD B, n     00000110 n      Load immediate byte n into B
ADD A, B    10000000        Add B to A
SUB A, B    10010000        Subtract B from A
JP nn       11000011 lo hi  Jump to 16-bit address nn
JP Z, nn    11001010 lo hi  Jump to nn if zero flag set
HALT        01110110        Stop execution
```

### Design the Assembler Logic on Paper

A programmer designed the logic of a program that reads the standard notation as characters and outputs the correct binary opcodes. Every step was designed on paper as flowcharts and pseudocode. Then each step was translated to binary opcodes using the ISA manual.

### Toggle the Binary Into Memory

The complete assembler program in binary was toggled into memory byte by byte using the front panel switch procedure. This took hours.

### Run It and Self-Assemble

Programmer pressed RUN. The assembler started executing. Programmer typed the assembler's source code in the agreed notation. The assembler processed it and output the binary of itself. That self-assembled version became the canonical assembler. The hand-toggled bootstrap was retired.

```
Bootstrap chain:

Human calculates assembler binary from ISA manual on paper
Human toggles binary into memory via front panel switches
         |
         v
Assembler version 0 runs (hand-entered binary)
Programmer types assembler source in standard notation
Assembler v0 outputs binary of assembler v1
         |
         v
Assembler v1 runs (cleaner, self-assembled version)
Programmer writes improved assembler v2 in notation
Assembler v1 assembles v2
         |
         v
Chain continues unbroken to present day
GCC uses an internal assembler that traces back through this chain
That chain reaches back to one person at a front panel
notebook open to the ISA manual
toggling binary into memory switch by switch
```

---

## The EDSAC — The First Time This Happened (May 6, 1949)

The EDSAC at Cambridge University, built by Maurice Wilkes, ran the first program from memory on May 6, 1949. Wilkes faced the bootstrapping problem directly. His solution was the **Initial Orders** — 31 instructions wired into a permanent read-only store. Their only purpose was to read a program from paper tape and load it into main memory.

```
EDSAC boot sequence:

Power on
    |
    v
Initial Orders in read-only store (hardwired, always present)
CPU begins executing Initial Orders from address 0
    |
    v
Initial Orders read paper tape through the tape reader
Each byte from tape stored sequentially in main memory
    |
    v
When tape ends, Initial Orders jump to address 0 of main memory
    |
    v
The program loaded from tape begins executing
```

The Initial Orders are the direct ancestor of your bootloader. GRUB Stage 1 does the same thing — a tiny fixed program that loads a bigger program and jumps to it. The BIOS ROM on your motherboard is the same concept as the EDSAC's read-only store. The chain from 1949 to your machine today is unbroken.

---

## Paper Tape and Punched Cards — The First Persistent Storage

Front panel switches loaded programs into volatile RAM. Power off, program lost. Persistent storage was needed.

**Paper tape** — a strip of paper with holes punched in rows, one byte per row. A punched hole = 1. No hole = 0. The tape punch recorded programs. The tape reader replayed them. Persistent, portable, sequential, fragile.

```
8-hole paper tape encoding (one row = one byte):

Sprocket hole (mechanical drive) ->  |
                                     v
  . * * * * . . .   <- holes punched across the width
  0 1 1 1 1 0 0 0   <- = 0x78 = 01111000

Programs stored as physical holes in paper strips.
Reels of tape could be mailed between institutions.
Dropped or torn tape = program destroyed.
```

**Punched cards** — IBM's 80-column stiff card. One card per line of source code. Holes in columns encoded characters. A deck of cards was a complete program. A dropped deck shuffled the order — programmers wrote their name and a sequence number on each card so they could resort a dropped deck.

---

## ASCII — Connecting Teletypes to Computers (1963)

By the early 1960s, teletypes were being connected to computers. Different manufacturers used different character encodings. In 1963, ASCII standardised 128 characters as 7-bit values.

```
ASCII (selected entries):

Decimal  Hex   Binary    Character   Notes
-------  ----  --------  ----------  --------------------------
      7  0x07  0000111   BEL         Ring the teletype bell
      8  0x08  0001000   BS          Backspace
     10  0x0A  0001010   LF          Line feed (advance paper)
     13  0x0D  0001101   CR          Carriage return (head left)
     27  0x1B  0011011   ESC         Escape
     32  0x20  0100000   SPACE
     48  0x30  0110000   0
     65  0x41  1000001   A
     97  0x61  1100001   a           = A + 32 (just one bit difference)
```

ASCII control characters (0-31) were designed to control teletype hardware. They survive unchanged in your terminal today — `\n`, `\r`, `\a`, `\b` are the same codes that moved mechanical print heads in 1963. The ANSI escape sequences that colour your terminal output descend from teletype control protocols.

Once ASCII existed, assemblers could compare incoming bytes to known character codes unambiguously:

```
Assembler sees byte: 01001100 = 76 = ASCII 'L'
Next byte:           01000100 = 68 = ASCII 'D'
Next byte:           01000001 = 65 = ASCII 'A'
Assembler recognises: "LDA" mnemonic
Outputs:             00111110 = Intel 8080 opcode for LD A
```

The programmer types text. The teletype encodes it as ASCII. The assembler compares ASCII bytes to its mnemonic table. The assembler outputs binary opcodes. The hardware decoder recognises opcodes and fires control signals. Three layers of automatic decoding in a chain.

---

## How the Instruction Set Shaped All of Computing

### The x86 Lock-In — 1978 to Present

In 1978 Intel published the 8086 instruction set. In 1981 IBM chose it for the IBM PC. Every subsequent Intel and AMD processor maintained backward compatibility. The same binary patterns that triggered the 8086 decoder in 1978 are still recognised by Intel CPUs in 2024.

```
x86 backward compatibility:

Intel 8086 (1978)   -> 8086 opcodes work
Intel 80286 (1982)  -> same opcodes + new ones
Intel 80386 (1985)  -> same opcodes + 32-bit extensions
Intel Pentium (1993)-> same opcodes, faster execution
AMD Athlon 64 (2003)-> same opcodes + 64-bit extensions (x86-64)
Intel Core i9 (2024)-> all previous opcodes still work

A binary compiled in 1985 runs on a 2024 CPU.
The opcode table from 1978 governs what happens in hardware today.
```

Modern x86-64 CPUs cannot decode x86 instructions efficiently in pure hardware because the encoding is too complex and variable. So they add a pre-decoder layer that translates x86-64 bytes into internal micro-operations. The CPU speaks x86-64 on the outside. It uses a clean internal instruction set on the inside. Four decoding layers to maintain 45 years of backward compatibility.

### ARM — Designed Clean, Won Mobile

ARM (1985, Acorn Computers, Cambridge) was designed with a regular, fixed-length encoding from the start. Every instruction is 32 bits. Fields are in consistent positions. The decoder is simple and power-efficient. This is why every smartphone and tablet runs ARM — battery life depends on power efficiency, and ARM's clean encoding is fundamentally more efficient to decode than x86.

### RISC-V — Open, Free, The Future

RISC-V (2010, UC Berkeley) is an open royalty-free ISA. Anyone can build a RISC-V CPU. It applies lessons from 60 years of ISA design: clean fixed-length encoding, modular extensions, fully open specification. Growing rapidly in embedded systems and beginning to appear in servers and workstations.


---

## The Meaning of x86 — Where the Name Came From

x86 is not an acronym. It does not stand for anything as words. It is a naming pattern that Intel accidentally created and then could not escape.

### Intel's Processor Numbering

Intel named its early processors with numbers:

```
1971  Intel 4004    <- 4-bit, first commercial microprocessor
1972  Intel 8008    <- 8-bit
1974  Intel 8080    <- 8-bit (the architecture used in Chapter 02 examples)
1976  Intel 8085    <- 8-bit, refined 8080
1978  Intel 8086    <- 16-bit, first x86
1979  Intel 8088    <- 8086 variant with 8-bit external bus
                       used in IBM PC 1981 — this is why x86 won
1982  Intel 80286   <- 16-bit, added protected mode
1985  Intel 80386   <- 32-bit, the "386"
1989  Intel 80486   <- 32-bit, the "486"
1993  Intel Pentium <- Intel stopped using numbers here
```

Every processor from the 8086 onward had a model number ending in 86:

```
 8086
80286
80386
80486
```

When engineers needed shorthand to refer to this processor family and their instruction set collectively, they took the common suffix — **86** — and placed **x** in front as a wildcard meaning "any of these processors." The x represented the varying prefix: 8, 80, and so on.

```
x86 = "any processor in the 8086/80286/80386/80486 family"
x   = wildcard (could be 8, 80, or anything)
86  = common suffix shared by every family member
```

It was never an official Intel product name or trademark. It was informal industry shorthand that became so universal Intel eventually adopted it in their own documentation.

### Why Intel Stopped Using Numbers

Intel tried to trademark the number "386" to prevent AMD from selling compatible processors under that name. The court ruled that numbers cannot be trademarked. AMD was free to sell the "AMD 386."

To escape this, Intel moved to brand names starting with Pentium in 1993 — from the Greek "penta" meaning five (it would have been the 586 following the pattern). But by then "x86" was permanent in industry vocabulary. The Pentium and every processor since is still described as x86 architecture because it runs the same instruction set that began with the 8086 in 1978.

### The x86-64 Extension

When AMD extended x86 to 64-bit in 2003 with the Athlon 64, they called it **AMD64**. Intel later implemented the same extension and called it **Intel 64** or **EM64T** (Extended Memory 64 Technology). The industry needed a neutral name not belonging to either company.

**x86-64** emerged as that neutral term — the x86 family extended to 64-bit. You will also see:

```
x86-64   <- most common in documentation (neutral term)
x86_64   <- used in Linux kernel source and GCC compiler flags (underscore)
amd64    <- used in Debian, Ubuntu, FreeBSD packages
             (acknowledges AMD designed this extension first)
Intel 64 <- Intel's own marketing name
EM64T    <- Intel's original name when they first adopted it

All refer to the same instruction set.

On your Pop!_OS machine:
  uname -m
  -> x86_64
```

### The Full Naming Chain

```
Intel 8086 (1978)
  Model number ends in 86
  This processor and its ISA = origin of "x86" label

Intel 8088 (1979)
  8086-compatible, 8-bit external bus
  IBM chose this for the IBM PC (1981) — x86 becomes dominant

Intel 80286 / 80386 / 80486 (1982-1989)
  All end in 86, all backward compatible
  "x86" solidifies as the family name

Intel Pentium (1993)
  Brand name replaces numbers (cannot trademark numbers)
  Still x86 architecture underneath

AMD Athlon 64 (2003)
  Extends x86 registers from 32 to 64 bits
  AMD calls it AMD64 — industry calls it x86-64

Intel Core i9 (2024)
  Still x86-64, still runs opcodes from 1978
  The name x86 now carries 46 years of history
```

The x86 in x86-64 refers directly to the 8086. The 64 refers to 64-bit registers. The name carries the entire history of the instruction set encoded inside it.

---

## Von Neumann Was Not Alone — The Real History of the Stored-Program Computer

The stored-program concept — that instructions and data live in the same memory and the CPU fetches them automatically — is credited to Von Neumann but was developed by multiple people simultaneously and independently. Von Neumann received the credit largely due to a bureaucratic accident.

### The Bureaucratic Accident — The Draft Report

In 1945, a team at the University of Pennsylvania was building the EDVAC computer. The team included John Mauchly and J. Presper Eckert as co-designers, with John von Neumann joining later as a consultant. Von Neumann wrote up the team's collective ideas in a document called the **First Draft of a Report on the EDVAC**. He put only his own name on it, intending to add all contributors in a final version.

Before the final version was written, someone distributed the draft to other researchers across the computing world. The report circulated with only Von Neumann's name on it. Everyone who read it assumed the ideas were his. Mauchly and Eckert spent years attempting to reclaim credit.

```
Timeline of the misattribution:

January 1944
  Eckert writes internal memo describing stored-program concept
  Von Neumann has not yet joined the project

1944-1945
  EDVAC team develops the architecture collectively
  Von Neumann joins as consultant

June 1945
  Von Neumann writes the First Draft — his name only
  Intends to add all contributors in final version

Late 1945
  Draft circulates widely before final version is written
  Computing world reads it as Von Neumann's ideas alone

1945 onward
  "Von Neumann architecture" enters vocabulary
  Eckert and Mauchly receive no credit

1980s-1990s
  Historical revisionism begins
  Eckert acknowledged as the originator of the stored-program idea
```

### J. Presper Eckert — The Actual First

Eckert's internal memo of January 1944 — a full year before Von Neumann joined the project — described storing programs in the same memory as data. This predates any Von Neumann contribution by over a year. Eckert is arguably the true inventor of what we call the Von Neumann architecture.

### Alan Turing — Independent and Earlier in Theory

Alan Turing described a theoretical stored-program machine in his 1936 paper **On Computable Numbers** — nine years before Von Neumann's draft. The **Turing Machine** is a theoretical model where:

```
Turing Machine (1936 — theoretical):

Tape: [ state rules | current data | results ]
       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
       program and data in the same space (the tape)

Read/write head moves along tape automatically
Reads current symbol
Looks up in state table what to do
Writes new symbol or moves head
Transitions to new state
Repeats — entirely self-driven, no human intervention

This IS the stored-program concept in theoretical form.
Nine years before the EDVAC draft.
```

Turing then applied this practically. He designed the **ACE (Automatic Computing Engine)** in 1945 — the same year as Von Neumann's draft — with ideas about subroutines and program structure more advanced than the EDVAC design. The **Manchester Mark 1**, built with Turing consulting, ran a stored program in 1948 — before the EDVAC ran.

### Konrad Zuse — Independent, Earliest in Practice

The most overlooked figure is the German engineer **Konrad Zuse**, who built a series of working programmable computers in Germany during World War II, completely isolated from British and American research:

```
Z1 (1936-1938)  mechanical computer, binary arithmetic
Z2 (1939)       electromechanical
Z3 (1941)       world's first fully functional programmable
                electromechanical computer
                read instructions from punched film tape automatically
                predates ENIAC by four years
Z4 (1944-1945)  more advanced, survived the war
```

The Z3 in 1941 was self-driving — it read instructions from punched film tape sequentially without human intervention per step. It predates every American and British stored-program computer by years. Zuse also designed **Plankalkül** in 1945 — arguably the first high-level programming language ever conceived, though not implemented until decades later.

Zuse's isolation from the western computing world by the war meant his work was largely unknown until afterward, by which time Von Neumann's draft had already defined the field's vocabulary.

### The Complete Picture — Who Did What First

```
1936  Alan Turing    Theoretical stored-program model (Turing Machine)
                     Nine years before EDVAC draft

1941  Konrad Zuse    Z3 — first working programmable computer
                     Self-driving from punched film tape
                     Four years before ENIAC

1944  J. Presper     Internal memo describing stored-program concept
      Eckert         One year before Von Neumann joined the project

1944  Howard Aiken   Harvard Mark I — self-driving from punched tape
                     (Harvard Architecture — separate instruction memory)

1945  Eckert,        EDVAC design — electronic stored-program computer
      Mauchly,       Von Neumann joins team, writes the draft
      Von Neumann    Draft circulates with only Von Neumann's name

1945  Alan Turing    ACE design — more sophisticated than EDVAC
                     Includes subroutine concepts years ahead of time

1948  Williams,      Manchester Mark 1 — runs stored program
      Kilburn,       Before EDVAC runs
      Turing

1949  Maurice Wilkes EDSAC — runs first practically useful stored program
```

### The Direct Answer

Von Neumann was not alone and was arguably not first. The stored-program self-driving computer was developed simultaneously and independently by multiple people. Von Neumann's contribution was real — he articulated the architecture with exceptional clarity — but the "Von Neumann architecture" label is a historical simplification that credits one consultant for what was a collective development across multiple continents over multiple years.

> **The stored-program self-driving computer was in the air in the mid-1940s. Multiple people reached for it simultaneously. Von Neumann's name is on it because he wrote the clearest description at the right moment — but Eckert had the idea first, Turing had the theoretical model nine years earlier, and Zuse had working hardware four years before ENIAC.**


---

## What This Means for Security

The instruction set is the hardware foundation of every security concept in this document.

**Buffer overflows** work because the `ret` instruction — opcode `0xC3` on x86-64 — is defined by the ISA as "pop address from stack, jump there, trust it completely." The decoder fires the ret-circuit when it sees `0xC3`. No verification. That is the hardware design.

**Shellcode** is machine code for a specific ISA. Shellcode for x86-64 uses `0xC3`, `0x90`, `0x48`, `0x31` and so on — real x86-64 opcodes. The same bytes on an ARM64 CPU would trigger completely different decoder wires and produce nonsense or a fault.

**Return-oriented programming** works because every `0xC3` byte in every binary on the system is a potential gadget handle. The decoder always fires the ret-circuit for `0xC3` regardless of context.

**System calls** use `0x0F 0x05` on x86-64 (SYSCALL) or `0xD4 0x00 0x00 0xD4` on ARM64 (SVC #0). These opcodes are defined by the ISA to cause the decoder to activate the privilege-level-switch circuit — changing the CPU from ring 3 to ring 0. This is a hardware mechanism in the decoder wiring, not a software check.

The instruction set is the hardware contract that defines every trust boundary and every security primitive. Understanding it is the foundation of understanding everything else in this document.
-e 
---

---

## Part B — The CPU Execution Pipeline

## Out-of-Order Execution — The CPU Does Not Execute in Order

When you write code, you assume instructions execute one after another in the order you wrote them. On every modern CPU this assumption is false. The CPU reorders instructions internally to keep its execution units busy while waiting for slow operations like memory reads.

### The Problem Out-of-Order Solves

```
Consider this sequence:
  MOV RAX, [address]    ; load from memory — takes 100-300 cycles
  ADD RBX, 1            ; add 1 to RBX — takes 1 cycle
  ADD RCX, 2            ; add 1 to RCX — takes 1 cycle

In-order CPU (simple):
  Cycle 1:   issue MOV RAX, [address]
  Cycles 2-200: STALL — waiting for RAM to return value
  Cycle 201: issue ADD RBX, 1
  Cycle 202: issue ADD RCX, 2
  Total: ~202 cycles, most of them wasted stalling

Out-of-order CPU (modern):
  Cycle 1:   issue MOV RAX, [address] (dispatched to load unit)
  Cycle 2:   issue ADD RBX, 1 (no dependency on RAX — execute now)
  Cycle 3:   issue ADD RCX, 2 (no dependency on RAX — execute now)
  Cycle 100: MOV completes, RAX now has value
  Total: ~100 cycles — 50% faster on this example
```

The CPU detects that ADD RBX and ADD RCX do not depend on the result of the memory load and executes them while waiting. From the programmer's perspective the output is identical — the CPU ensures this — but the execution order is completely different.

### The Reorder Buffer (ROB)

The ROB is a circular buffer inside the CPU that tracks all in-flight instructions in their original program order. Instructions enter the ROB in order, execute out of order, and retire (commit results) in order.

```
ROB structure (simplified):

Entry  Instruction        Status       Result
─────  ─────────────────  ───────────  ──────
 0     MOV RAX,[addr]     EXECUTING    (waiting for memory)
 1     ADD RBX, 1         COMPLETE     RBX = old_RBX + 1
 2     ADD RCX, 2         COMPLETE     RCX = old_RCX + 2
 3     MOV RDX, RAX       WAITING      (depends on entry 0)
 4     ...

Retirement happens from entry 0 first:
  Entry 0 completes (memory returns) → commit RAX, retire entry 0
  Entry 1 already complete → commit RBX, retire entry 1
  Entry 2 already complete → commit RCX, retire entry 2
  Entry 3 can now execute → dispatch to ALU

To the outside world: instructions appear to have executed in order.
Inside the CPU: they ran in a completely different order.
```

### Reservation Stations

Before an instruction can execute, it waits in a **reservation station** — a buffer attached to each execution unit. The instruction waits here until all its input operands are available.

```
Reservation stations for the ALU:

Slot  Instruction   Src1 Ready?  Src1 Value  Src2 Ready?  Src2 Value
────  ────────────  ───────────  ──────────  ───────────  ──────────
 0    ADD RBX, 1    YES          old_RBX     YES          1
 1    ADD RCX, 2    YES          old_RCX     YES          2
 2    ADD RDX, RAX  NO (RAX     ---         YES          ---
                    not ready)

Slots 0 and 1 can execute immediately.
Slot 2 waits until RAX arrives from the memory load.
When RAX becomes available, slot 2 executes.
```

---

## Register Renaming — Eliminating False Dependencies

Consider this sequence:

```
MOV RAX, 5      ; write RAX
ADD RBX, RAX    ; read RAX  ← true dependency: must wait for MOV
MOV RAX, 10     ; write RAX ← writes RAX again
ADD RCX, RAX    ; read RAX  ← reads the NEW RAX from line 3
```

The second `MOV RAX, 10` creates a **write-after-write** hazard with the first `MOV RAX, 5`. The `ADD RCX, RAX` has a **write-after-read** hazard with `ADD RBX, RAX`. These are called **false dependencies** — they only exist because both instructions happen to use the register named RAX, not because one truly needs the other's result.

Register renaming eliminates false dependencies by mapping architectural registers (the 16 registers the programmer sees: RAX-R15) to a larger pool of **physical registers** inside the CPU.

```
Physical register file on modern x86-64: ~180-512 registers
Architectural register file visible to programmer: 16 registers

Renaming example:

Instruction            Architectural  Physical register assigned
─────────────────────  ─────────────  ──────────────────────────
MOV RAX, 5             RAX            P47
ADD RBX, RAX           RAX→P47        (reads P47, writes P23 for RBX)
MOV RAX, 10            RAX            P89   ← NEW physical register!
ADD RCX, RAX           RAX→P89        (reads P89, not P47)

Now:
  ADD RBX, RAX reads P47
  MOV RAX, 10 writes P89
  These are DIFFERENT physical registers
  No false dependency
  Both can execute in parallel with each other
```

Register renaming is done by the **Register Alias Table (RAT)** — a hardware lookup table that maps each architectural register name to its current physical register. Updated on every instruction that writes a register.

---

## Speculative Execution — Running Code Before Knowing If It Should

When the CPU encounters a conditional branch, it does not know which way to go until the condition is evaluated. Evaluating the condition might take many cycles. Waiting would waste time.

**Speculative execution** means the CPU guesses which branch will be taken and starts executing the predicted path immediately without waiting for the condition to be confirmed.

```
if (x > 0) {
    result = expensive_function(x);  // branch taken
} else {
    result = 0;                      // branch not taken
}

Branch predictor guesses: "x is probably > 0, branch will be taken"
CPU immediately starts executing expensive_function(x)
Meanwhile evaluates x > 0...

IF guess correct:
  expensive_function has been running during evaluation
  No time wasted — result is ready sooner
  CPU commits the speculative results

IF guess wrong:
  CPU must discard all speculative results
  Return to architectural state before the branch
  Execute the correct path (result = 0)
  This is called a branch misprediction — costs 15-20 cycles
```

### Branch Predictors

Modern branch predictors are sophisticated pattern-recognition circuits:

```
Static prediction:
  Forward branches (if-then): predict not taken
  Backward branches (loops): predict taken
  Simple rule-based, no history

Dynamic prediction:
  2-bit saturating counter per branch:
    00 = strongly not taken
    01 = weakly not taken
    10 = weakly taken
    11 = strongly taken
  History shifts the counter on each execution

Tournament predictor (modern):
  Combines multiple predictor types
  Uses meta-predictor to choose which sub-predictor to trust
  Intel and AMD branch predictors: >98% accuracy on typical code

Branch Target Buffer (BTB):
  Caches the target address of indirect branches
  call [rax] — where does rax point?
  BTB remembers previous targets and predicts
```

### Spectre and Meltdown — When Speculation Becomes a Vulnerability

Speculative execution is the hardware basis of the **Spectre** and **Meltdown** vulnerabilities (disclosed January 2018).

```
Meltdown (CVE-2017-5754):
  CPU speculatively executes instructions AFTER a page fault
  Page table says: this kernel address is not accessible from ring 3
  CPU checks this, but speculatively executes the next instruction anyway
  The next instruction reads kernel memory into a register
  CPU discovers the fault, discards the register value
  BUT: the memory access already brought data into the cache
  Attacker measures cache timing to infer the discarded value
  Kernel memory read from ring 3 — complete privilege violation

  Fixed by: KPTI (Kernel Page Table Isolation)
    Kernel pages are entirely unmapped from user page tables
    Context switch maps and unmaps kernel pages
    Nothing to speculatively access even with misprediction

Spectre (CVE-2017-5753, CVE-2017-5715):
  More subtle — exploits branch prediction, not just speculation
  Attacker trains branch predictor with attacker-controlled branches
  Victim code's branch predictor is poisoned
  Victim speculatively executes wrong path
  Speculatively executed code accesses secret memory
  Secret leaks via cache timing side channel
  
  Fixed by: retpoline, IBRS, IBPB — software and microcode mitigations
    No clean hardware fix — speculation is fundamental to performance
```

---

## Interrupts vs Exceptions vs Traps — Three Ways to Enter the Kernel

All three transfer control from user code (or hardware) to kernel code. They are different in cause, timing, and intent.

```
INTERRUPT (asynchronous — caused by hardware):
  Source:    hardware device (keyboard, NIC, timer, DMA controller)
  When:      any time — between any two instructions
  Cause:     hardware needs CPU attention
  Examples:  keyboard key pressed, network packet arrived,
             timer fired (10ms scheduler tick),
             DMA transfer complete
  Handling:  CPU finishes current instruction
             Saves RIP and RFLAGS to kernel stack
             Loads handler address from IDT (Interrupt Descriptor Table)
             Executes interrupt handler (ISR)
             Handler services device, sends EOI to interrupt controller
             Returns to interrupted code via IRET instruction

EXCEPTION (synchronous — caused by instruction):
  Source:    CPU itself, triggered by an instruction's behaviour
  When:      immediately when the offending instruction executes
  Cause:     instruction does something illegal or exceptional
  Examples:
    Page fault (#PF):    virtual address not mapped or permission denied
    General Protection Fault (#GP): ring 3 executes privileged instruction
    Divide by Zero (#DE): DIV instruction with divisor = 0
    Invalid Opcode (#UD): CPU sees undefined opcode bytes
    Stack Fault (#SS):   stack segment limit exceeded
    Breakpoint (#BP):    INT3 instruction (debugger breakpoint)
  Handling:  same mechanism as interrupt but triggered by the instruction
             Kernel fault handler examines the cause
             Either fixes it (page fault → load page) or kills process (SIGSEGV)

TRAP (synchronous — deliberate software-initiated):
  Source:    software, intentionally executed
  When:      when the trap instruction executes
  Cause:     deliberate request for kernel service or debugger notification
  Examples:
    SYSCALL (x86-64):   system call — ring 3 requests kernel service
    SVC #0 (ARM64):     system call on ARM
    INT3:               debugger breakpoint (also an exception)
    INT 0x80 (legacy):  old Linux system call method
  Handling:  controlled entry to kernel at known safe entry points
             SYSCALL specifically uses LSTAR register for entry address

COMPARISON TABLE:
  Type        Cause           Timing        Resumable?
  ─────────   ─────────────   ───────────   ──────────────────────────
  Interrupt   Hardware        Async         Yes (code continues after)
  Exception   Bad instruction Sync          Sometimes (page fault yes,
                                            GPF usually kills process)
  Trap        Intentional     Sync          Yes (syscall returns)
```

```
Interrupt Descriptor Table (IDT):
  Array of 256 entries in memory
  Each entry: handler address + privilege requirements
  Entry 0:  divide by zero handler
  Entry 1:  debug exception handler
  Entry 3:  breakpoint handler (INT3)
  Entry 14: page fault handler
  Entry 32-255: hardware interrupt handlers
  Kernel loads IDT address into IDTR register at boot using LIDT instruction
  LIDT is a privileged instruction — ring 0 only
```

---

## The GDT — Global Descriptor Table

The **Global Descriptor Table** is an x86 data structure in memory that defines memory segments and their privilege levels. It is a legacy of the 16-bit segmented memory model but is still required on x86-64.

```
WHY IT EXISTS:
  x86 originally used segmentation (8086, 1978) to divide memory into segments
  The 80286 added protected mode with privilege enforcement via segment descriptors
  x86-64 mostly abandoned segmentation for flat memory model
  But the GDT still exists and is required for ring transitions

WHAT THE GDT CONTAINS TODAY:
  Entry 0: null descriptor (required by CPU, must be all zeros)
  Entry 1: kernel code segment (CPL=0, 64-bit, execute/read)
  Entry 2: kernel data segment (CPL=0, read/write)
  Entry 3: user code segment  (CPL=3, 64-bit, execute/read)
  Entry 4: user data segment  (CPL=3, read/write)
  Entry 5: TSS descriptor     (Task State Segment — holds kernel stack pointer)
  FS/GS base: used for thread-local storage

WHAT THE CS REGISTER HOLDS:
  The CS (Code Segment) register holds a SELECTOR — an index into the GDT
  The bottom 2 bits of the selector = CPL (Current Privilege Level)
  CS selector for kernel code: 0x08 (index 1, CPL=00)
  CS selector for user code:   0x33 (index 6, CPL=11)

  When SYSCALL fires:
    CPU loads the kernel CS selector (CPL=0) from IA32_STAR MSR
    This is how CPL changes from 3 to 0 atomically

TSS (Task State Segment):
  Structure in memory pointed to by GDT TSS entry
  Contains RSP0 — the kernel stack pointer for ring 0
  When interrupt fires in ring 3, CPU reads RSP0 from TSS
  Switches to that kernel stack before calling handler
  Prevents user stack being used for kernel operations

The kernel sets up the GDT at boot:
  lgdt [gdt_pointer]    ; load GDT address into GDTR register
  ltr  [tss_selector]   ; load TSS selector into TR register
  Both are ring 0 privileged instructions
```

---

## Context Switching — What the Kernel Actually Saves and Restores

A **context switch** is the kernel replacing one process's CPU state with another's. Every thread switch, every process switch, every time the scheduler picks a new task — a context switch happens.

```
WHAT MUST BE SAVED (the complete CPU state of the outgoing process):

General-purpose registers:
  RAX, RBX, RCX, RDX, RSI, RDI, RSP, RBP, R8-R15
  (16 registers × 8 bytes = 128 bytes)

Instruction pointer:
  RIP — where this process will resume

Flags:
  RFLAGS — condition codes (ZF, CF, SF, OF, IF...)

Segment registers:
  CS, SS, DS, ES, FS, GS
  FS.base and GS.base (used for thread-local storage)

FPU/SSE/AVX state:
  XMM0-XMM15 (128-bit SSE registers) — 256 bytes
  YMM0-YMM15 (256-bit AVX registers) — 512 bytes
  ZMM0-ZMM31 (512-bit AVX-512 registers) — 2048 bytes
  MXCSR (SSE control/status register)
  x87 FPU stack and control registers
  Saved using FXSAVE or XSAVE instruction (can save 2.5KB+ of state)

Page table pointer:
  CR3 — physical address of this process's PML4 page table
  Loading new CR3 switches the entire virtual address space
  Also flushes TLB (all cached address translations become invalid)
  TLB flush is expensive — PCID (Process Context ID) can avoid some flushes

WHERE THE SAVED STATE GOES:
  Into the kernel's per-task data structure: task_struct in Linux
  Specifically in the thread field: struct thread_struct
  Stored in kernel memory — the process cannot access its own task_struct
```

```
CONTEXT SWITCH SEQUENCE (Linux, simplified):

1. Interrupt fires (timer, 10ms tick) or process blocks (I/O wait)
2. CPU saves RIP and RFLAGS to kernel stack (hardware, automatic)
3. CPU switches to kernel stack (RSP0 from TSS)
4. Interrupt handler runs in kernel (ring 0)
5. Handler calls schedule()
6. schedule() calls context_switch(prev, next)
7. switch_mm(prev, next):
   - load new CR3 (change page tables — new virtual address space)
   - TLB flush (or update PCID)
8. switch_to(prev, next):
   - XSAVE: save FPU/SSE/AVX state of prev task
   - Save general registers of prev task to prev->thread
   - Load general registers of next task from next->thread
   - XRSTOR: restore FPU/SSE/AVX state of next task
   - Load FS.base and GS.base for next task (thread-local storage)
9. Return to next task's RIP (resume where it was interrupted)
10. IRET: restore RFLAGS, switch CS (ring 0 → ring 3), restore RSP

From the next process's perspective: nothing happened.
It resumes exactly where it left off, all registers intact.

Context switch overhead: ~1-10 microseconds
  TLB flush is the most expensive part (~1-3 microseconds)
  Threads in the same process share page tables — no TLB flush needed
  This is one reason threads are cheaper than processes
```

---

## ELF Format in Detail — What Is Inside an Executable

Every compiled program on Linux is an ELF (Executable and Linkable Format) file. Understanding ELF explains why programs work the way they do, why ASLR randomises specific addresses, and how buffer overflow exploits target specific sections.

```
ELF FILE STRUCTURE:

┌───────────────────────────────────────────────────┐
│  ELF Header (64 bytes)                            │
│    Magic: 7F 45 4C 46 ("ELF" in ASCII)            │
│    Class: 64-bit                                  │
│    Data: little-endian                            │
│    OS/ABI: Linux                                  │
│    Type: ET_EXEC (executable) or ET_DYN (PIE)     │
│    Machine: x86-64 (0x3E) or ARM64 (0xB7)         │
│    Entry point: 0x401000 (first instruction)      │
│    PHoff: offset to program header table          │
│    SHoff: offset to section header table          │
├───────────────────────────────────────────────────┤
│  Program Header Table (for the OS loader)         │
│    LOAD segment 1: .text (code) — r-x             │
│    LOAD segment 2: .data+.bss (data) — rw-        │
│    DYNAMIC segment: dynamic linking info          │
│    GNU_STACK: stack permissions (usually rw- NX)  │
│    GNU_RELRO: read-only after relocation          │
├───────────────────────────────────────────────────┤
│  .text section — compiled machine code            │
│    Your functions as opcode bytes                 │
│    Read-only + executable                         │
│    NX bit NOT set here (this IS code)             │
├───────────────────────────────────────────────────┤
│  .rodata section — read-only data                 │
│    String literals: "hello\n"                     │
│    Constant arrays                                │
│    Read-only, NOT executable                      │
├───────────────────────────────────────────────────┤ 
│  .data section — initialised global variables     │
│    int x = 5;  ← lives here (value 5 stored)      │
│    Read-write, NOT executable                     │
├───────────────────────────────────────────────────┤
│  .bss section — uninitialised global variables    │
│    int y;  ← lives here (no bytes in file)        │
│    Kernel zeroes this region when loading         │
│    Takes no space in ELF file itself              │
│    Read-write, NOT executable                     │
├───────────────────────────────────────────────────┤
│  .plt section — Procedure Linkage Table           │
│    Stubs for calling shared library functions     │
│    Covered in dynamic linking section below       │
├───────────────────────────────────────────────────┤
│  .got section — Global Offset Table               │
│    Addresses resolved at runtime by dynamic linker│
│    Covered in dynamic linking section below       │
├───────────────────────────────────────────────────┤
│  .symtab — symbol table (debug builds)            │
│    Maps function/variable names to addresses      │
│    Stripped in production binaries                │
├───────────────────────────────────────────────────┤
│  Section Header Table (for linker/debugger)       │
│    Describes all sections above                   │
│    Not needed at runtime (OS uses program hdrs)   │
└───────────────────────────────────────────────────┘
```

```bash
# Inspect an ELF binary
readelf -h /bin/ls          # ELF header
readelf -S /bin/ls          # all sections
readelf -l /bin/ls          # program headers (segments)
objdump -d /bin/ls          # disassemble .text section
objdump -s -j .rodata /bin/ls  # dump .rodata contents
size /bin/ls                # sizes of text, data, bss

# See the magic bytes
xxd /bin/ls | head -2
# 7f 45 4c 46 = ELF magic

# PIE (Position Independent Executable) vs non-PIE
file /bin/ls
# ... ELF 64-bit LSB pie executable ...
# PIE: base address randomised by ASLR each run

# Check sections of your own compiled program
gcc -o myprogram myprogram.c
readelf -S myprogram
```

---

## The Linker in Depth — Symbol Resolution, Relocation and the GOT/PLT

The linker takes multiple object files (.o) and produces a final executable. Understanding it explains how function calls across files work and why the PLT and GOT exist.

### Symbol Resolution

When you compile multiple .c files separately, each produces a .o file. References between files are **unresolved symbols** — placeholders that must be filled in.

```
File A: main.c
  calls printf() — unresolved: where is printf?
  calls helper() — unresolved: where is helper?

File B: helper.c
  defines helper() — provides the symbol
  calls malloc() — unresolved: where is malloc?

Static library: libc.a
  contains printf.o: defines printf()
  contains malloc.o: defines malloc()

Linker combines all of these:
  1. Collect all object files and needed library objects
  2. Build a symbol table: name → address
  3. Resolve every unresolved reference
  4. Assign final virtual addresses to every symbol
  5. Patch all call instructions with correct addresses
  6. Output ELF executable
```

### Static vs Dynamic Linking

```
STATIC LINKING:
  Linker copies all library code into the executable
  Final binary contains: your code + printf + malloc + everything
  Self-contained: runs without any external .so files
  Larger binary size
  No version conflicts
  Used for: embedded systems, containers, security-critical tools

DYNAMIC LINKING:
  Linker records which shared libraries are needed
  Does NOT copy library code into executable
  At runtime: OS loads the shared library (.so file) into memory
  Multiple programs share one copy of the library in memory
  Smaller binary size
  Library updates benefit all programs automatically
  Used for: almost everything on a normal Linux system

Check which type:
  file /bin/ls
  # dynamically linked
  ldd /bin/ls
  # shows all shared libraries the binary needs:
  # libc.so.6, libselinux.so.1, etc.

  ldd /bin/busybox
  # statically linked — no dependencies
```

### The PLT and GOT — Dynamic Linking at Runtime

When a dynamically linked program calls `printf()`, the address of `printf` is not known at compile time — it depends on where the dynamic linker places libc.so in memory. The PLT and GOT solve this.

```
PLT (Procedure Linkage Table) — in .plt section of executable:
  A trampoline stub for each external function
  printf@plt:
    JMP [printf@got]    ← indirect jump through GOT entry
    PUSH index          ← only executed first time
    JMP plt_resolver    ← only executed first time

GOT (Global Offset Table) — in .got section of executable:
  Array of 8-byte addresses, one per external function
  Populated by dynamic linker at program start (or lazily)
  Initially points back into PLT for lazy resolution
  After first call: contains real address of function

FIRST CALL to printf():
  1. CPU executes: CALL printf@plt
  2. PLT stub: JMP [printf@got]
  3. GOT entry points back to PLT resolver (not resolved yet)
  4. PLT resolver calls dynamic linker
  5. Dynamic linker finds printf in libc.so
  6. Dynamic linker writes real printf address into GOT
  7. Jumps to printf — function executes

SUBSEQUENT CALLS to printf():
  1. CPU executes: CALL printf@plt
  2. PLT stub: JMP [printf@got]
  3. GOT entry now has real printf address
  4. Jumps directly to printf
  No resolver involved — just two jumps

This is called lazy binding — resolution on first call only.
```

```bash
# See PLT entries
objdump -d /bin/ls | grep -A3 "@plt"

# See GOT entries
objdump -R /bin/ls          # dynamic relocations (GOT entries)
readelf -r /bin/ls          # all relocations

# Trace dynamic linking at runtime
LD_DEBUG=bindings ls 2>&1 | head -20
# Shows every symbol resolution as it happens
```

### Security Implications of PLT and GOT

```
GOT OVERWRITE ATTACK:
  If attacker can write to the GOT (writable memory region)
  They can replace printf@got with address of their shellcode
  Next call to printf() jumps to shellcode instead
  Classic ret2plt / GOT hijacking technique

RELRO (Relocation Read-Only) — defence against GOT overwrite:
  Partial RELRO: GOT is writable after dynamic linking
  Full RELRO:    dynamic linker resolves ALL symbols at startup
                 then marks GOT as read-only (mprotect)
                 GOT overwrite impossible after startup

Check RELRO:
  checksec --file=/bin/ls
  # Shows: Full RELRO, Partial RELRO, or No RELRO

In CTF/pentest: GOT overwrite only works without Full RELRO
In modern systems: most binaries have Full RELRO
```

---

## Atomic Operations and Memory Barriers

On a multi-core CPU, multiple cores execute instructions simultaneously. Without coordination, concurrent access to shared memory produces unpredictable results.

### The Race Condition at Hardware Level

```
Two threads, both executing: counter++;
counter++ compiles to:
  MOV EAX, [counter]    ; read
  ADD EAX, 1            ; increment
  MOV [counter], EAX    ; write back

Without synchronisation (both cores execute simultaneously):

Core 0:                          Core 1:
  MOV EAX, [counter]  → EAX=5
                                   MOV EAX, [counter]  → EAX=5
  ADD EAX, 1          → EAX=6
  MOV [counter], EAX  → mem=6
                                   ADD EAX, 1          → EAX=6
                                   MOV [counter], EAX  → mem=6

Result: counter=6 instead of 7
One increment was lost
```

### Atomic Operations — Hardware Support

The x86 LOCK prefix makes a read-modify-write operation atomic at the hardware level:

```
LOCK ADD [counter], 1

What happens:
  CPU asserts the LOCK# signal on the memory bus
  Other cores cannot access [counter] until this completes
  Read → increment → write happens as one indivisible operation
  Other cores see either the old value or the new value
  Never an intermediate state

x86-64 atomic instructions:
  LOCK ADD    atomic add
  LOCK XCHG   atomic exchange (always locks even without LOCK prefix)
  LOCK CMPXCHG compare-and-swap — the basis of all lock-free algorithms
  LOCK INC, LOCK DEC, LOCK OR, LOCK AND, LOCK XOR

ARM64 atomic operations:
  LDXR/STXR   load-exclusive / store-exclusive
              STXR fails if another core modified the location
              Retry loop if it fails
  LDADD, STADD  atomic add (ARMv8.1+)
  CAS         compare and swap (ARMv8.1+)
```

### Memory Barriers — Enforcing Order

Even with atomic operations, the CPU and memory system can reorder memory accesses. A **memory barrier** forces ordering constraints.

```
TYPES OF REORDERING:

Store-store reordering:
  Store to A, then store to B
  CPU may commit B to memory before A
  Other cores see B updated but A not yet

Load-load reordering:
  Load from A, then load from B
  CPU may fetch B before A
  Reads appear out of order

Store-load reordering (most common on x86):
  Store to A, then load from B
  CPU may execute the load before the store is visible

x86-64 memory model (TSO — Total Store Order):
  Most reorderings prevented by hardware
  Only store-load reordering is possible
  Relatively strong memory model

ARM64 memory model (weakly ordered):
  All four reorderings possible
  Requires explicit barriers for correct multi-core code

Memory barrier instructions:
  x86-64: MFENCE (full barrier), LFENCE (load), SFENCE (store)
  ARM64:  DMB (data memory barrier), DSB (data sync barrier)
  Linux kernel: smp_mb(), smp_rmb(), smp_wmb() — architecture-neutral

In kernel code:
  spin_lock() contains an implicit memory barrier
  All lock/unlock operations include appropriate barriers
  Needed for: device drivers, lock-free data structures, RCU
```

---

## CPUID — How Software Queries CPU Capabilities

The CPUID instruction allows software to query what features the CPU supports. The kernel uses this extensively at boot to configure itself.

```
CPUID instruction:
  MOV EAX, leaf_number    ; what information do you want?
  CPUID                   ; ask the CPU
  ; Results in EAX, EBX, ECX, EDX

Common CPUID leaves:

Leaf 0x0 (basic info):
  EAX: highest supported basic leaf
  EBX, EDX, ECX: vendor string
  GenuineIntel = 47 65 6E 75 49 6E 74 65 6C
  AuthenticAMD = 41 75 74 68 65 6E 74 69 63

Leaf 0x1 (feature flags):
  EDX bit 25: SSE    — 128-bit SIMD
  EDX bit 26: SSE2   — double-precision SIMD
  ECX bit 28: AVX    — 256-bit SIMD
  ECX bit 5:  VMX    — Intel virtualisation (VT-x)
  ECX bit 30: RDRAND — hardware random number generator

Leaf 0x7 subleaf 0 (extended features):
  EBX bit 0:  FSGSBASE — FS/GS base read/write
  EBX bit 7:  SMEP     — Supervisor Mode Execution Prevention
  EBX bit 20: SMAP     — Supervisor Mode Access Prevention
  ECX bit 2:  UMIP     — User-Mode Instruction Prevention
  ECX bit 5:  AVX512F  — AVX-512 foundation

Leaf 0x80000001 (AMD extended features):
  EDX bit 20: NX/XD    — No-Execute bit support
  EDX bit 29: LM       — Long Mode (64-bit support)
```

```bash
# Query CPUID from userspace
cpuid -1                    # all leaves (install cpuid package)
cat /proc/cpuinfo | grep flags  # kernel-parsed CPUID features

# Key flags to check for security features:
grep -m1 "flags" /proc/cpuinfo | tr ' ' '\n' | grep -E \
  "smep|smap|nx|pse|pae|vmx|svm|aes|avx|rdrand|fsgsbase"

# Check in kernel at boot time
dmesg | grep -i "CPU\|feature\|SMEP\|SMAP\|PAE\|NX"
```

The kernel reads CPUID at boot and enables features accordingly:

```
Boot sequence (kernel CPU detection):
  CPU powers on
  Kernel calls cpu_detect() or setup_cpu_features()
  CPUID leaf 0x1: check for SSE2, SSE4, AVX
  CPUID leaf 0x7: check for SMEP, SMAP, AVX-512
  If SMEP found: enable by setting CR4.SMEP bit
  If SMAP found: enable by setting CR4.SMAP bit
  If NX found:   enable by setting IA32_EFER.NXE MSR bit
  Kernel configures itself based on what hardware supports
```

---

## Compiler IR — Intermediate Representation

Between C source code and machine code, compilers use an **Intermediate Representation (IR)** — a platform-neutral form of the program that is easier to optimise than either source code or machine code.

```
WHY IR EXISTS:

Without IR:
  C frontend → x86-64 machine code directly
  To support ARM64: C frontend → ARM64 machine code directly
  To support RISC-V: C frontend → RISC-V directly
  N languages × M architectures = N×M compiler backends

With IR (LLVM approach):
  C frontend → LLVM IR
  Rust frontend → LLVM IR
  Swift frontend → LLVM IR
  LLVM IR → x86-64 backend
  LLVM IR → ARM64 backend
  LLVM IR → RISC-V backend
  N languages + M architectures = N frontends + M backends
  Any language compiles to any architecture
```

### LLVM IR — What It Looks Like

```c
// C source:
int add(int a, int b) {
    return a + b;
}
```

```llvm
; LLVM IR:
define i32 @add(i32 %a, i32 %b) {
entry:
  %result = add i32 %a, %b
  ret i32 %result
}
```

```asm
; x86-64 machine code (from LLVM IR):
add:
  lea eax, [rdi + rsi]  ; result = a + b
  ret

; ARM64 machine code (from same LLVM IR):
add:
  add w0, w0, w1        ; result = a + b
  ret
```

The same LLVM IR produces correct machine code for both architectures. The frontend (Clang, rustc) only needs to produce IR. The backend handles each architecture.

### GCC GIMPLE IR

GCC uses its own IR called **GIMPLE** — a simplified three-address form:

```c
// C source:
int x = (a + b) * (c + d);
```

```gimple
// GIMPLE (three-address code — max one operation per statement):
tmp1 = a + b;
tmp2 = c + d;
x = tmp1 * tmp2;
```

GIMPLE is what GCC optimises. After optimisation, it converts GIMPLE to RTL (Register Transfer Language) and then to machine code.

```bash
# See LLVM IR
clang -emit-llvm -S -o output.ll source.c
cat output.ll

# See GCC GIMPLE
gcc -fdump-tree-gimple source.c
cat source.c.004t.gimple

# See GCC RTL (closer to machine code)
gcc -fdump-rtl-expand source.c
```

---

## Shared Libraries and the Dynamic Linker

When a program needs printf(), it does not contain printf. It references libm.so or libc.so. At runtime, the **dynamic linker** (ld.so / ld-linux.so) loads those libraries and connects the references.

```
DYNAMIC LINKER SEQUENCE when ./myprogram starts:

1. Kernel loads myprogram ELF into memory
   Maps .text, .data, .bss, .plt, .got

2. Kernel reads PT_INTERP program header
   Contains path: /lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
   This is the dynamic linker itself

3. Kernel loads ld-linux.so into memory
   Jumps to ld-linux.so's entry point (not myprogram's yet)

4. ld-linux.so reads myprogram's PT_DYNAMIC segment
   Contains list of needed shared libraries:
     NEEDED libm.so.6
     NEEDED libc.so.6

5. ld-linux.so finds and loads each library:
   Search path: LD_LIBRARY_PATH, /etc/ld.so.cache, /lib, /usr/lib
   mmap() each .so file into process memory
   Each library gets its own virtual address region (ASLR randomised)

6. ld-linux.so performs relocations:
   For each GOT entry needing resolution:
     Find the symbol in the loaded libraries
     Write the real address into the GOT entry
   (Or defers to PLT lazy resolution for lazily-bound symbols)

7. ld-linux.so calls each library's initialisation functions
   .init_array section: constructors run before main()

8. ld-linux.so jumps to myprogram's entry point (_start)

9. _start calls __libc_start_main() which calls main()
```

```bash
# See the dynamic linker path
readelf -l /bin/ls | grep INTERP
# [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]

# Trace dynamic linker activity
LD_DEBUG=all ls 2>&1 | head -50

# See library load addresses (ASLR in effect — different each run)
ldd /bin/ls
ldd /bin/ls    # run again — addresses differ

# Preload your own library (hook functions)
LD_PRELOAD=/path/to/mylib.so ls
# Your library's functions override libc's — used for Frida-style hooking
```

### Security Implications of the Dynamic Linker

```
LD_PRELOAD hijacking:
  Set LD_PRELOAD to a malicious .so before running a program
  Your library's functions called instead of real library functions
  Can intercept: open(), read(), write(), strcmp(), crypt()
  Used by: Frida (legitimate), rootkits (malicious)
  Mitigated: setuid/setgid binaries ignore LD_PRELOAD

Library search order manipulation:
  LD_LIBRARY_PATH can redirect library loading
  Planting a malicious libcrypto.so.1 in the search path
  Intercepting crypto operations
  Mitigated: same — ignored for setuid binaries

DT_RPATH and DT_RUNPATH:
  Compiled-in library search paths (override LD_LIBRARY_PATH)
  DT_RPATH: searched before LD_LIBRARY_PATH (bad practice)
  DT_RUNPATH: searched after LD_LIBRARY_PATH
  Check with: readelf -d binary | grep -E "RPATH|RUNPATH"

Checking library integrity:
  checksec --file=/bin/ls  # shows all security properties
  ldd --verify binary      # verifies library dependencies
```


---

## Part III — The Software Foundation

---

---

## Chapter 7: Protection Rings — Hardware-Enforced Privilege

Chapters 03 and 06 explained the hardware and instruction set. Every program runs on the same CPU, using the same instructions. But not every program should be allowed to do everything. A user's web browser should not be able to read another user's files, disable interrupts, or modify the CPU's page tables. This chapter explains the hardware mechanism that enforces these boundaries — protection rings.

---

## The Problem — Everything Running at the Same Level

The earliest computers of the 1940s and 1950s had no privilege separation. Every program had complete access to everything: any memory address, any hardware register, any CPU instruction. On a time-sharing system where many users ran programs simultaneously, one user's bug or malicious code could corrupt the operating system itself, crash the machine, or read another user's private data.

Software boundaries alone cannot solve this. A program with full CPU access can overwrite any software check. The boundary must be enforced by the CPU itself — in hardware — at a level the program cannot reach or bypass.

---

## Where Rings Came From — Multics, 1964

The concept of hardware protection rings was designed for the **Multics** operating system at MIT in 1964, built in collaboration with General Electric and Bell Labs. The designers needed a way to enforce trust levels between different parts of the system. They invented the ring model and worked with GE hardware engineers to implement it in the **GE-645** processor — the first CPU with hardware ring support.

```
Multics ring model (1964):
  Ring 0: kernel            (most trusted, most privileged)
  Ring 1: memory management
  Ring 2: operating system services
  Ring 3: user programs     (least trusted, least privileged)
  Ring 4-7: further layers

The further from ring 0, the less privilege.
Moving inward toward 0 requires explicit permission.
Moving outward is automatic.
```

Multics actually used all eight rings. When Ken Thompson and Dennis Ritchie created Unix in 1969 as a deliberate simplification of Multics, they kept only two effective levels: kernel mode and user mode. This two-level model became the standard.

---

## How Rings Work in Hardware

The CPU contains a small field in a control register that indicates the current privilege level. On x86 this is called **CPL — Current Privilege Level**, stored in the bottom two bits of the CS (code segment) register.

```
CPL field — 2 bits — four possible values:

  00 = Ring 0  (kernel — most privileged)
  01 = Ring 1  (unused in practice on modern systems)
  10 = Ring 2  (unused in practice on modern systems)
  11 = Ring 3  (user programs — least privileged)
```

The CPU's decoder — the same decoder from Chapter 02 — checks the CPL field on every instruction that touches a privileged resource. If CPL is 3 and the instruction requires CPL 0, the CPU raises a **General Protection Fault** immediately, before the instruction executes.

```
Hardware privilege check on every privileged instruction:

Instruction fetched and decoded
         |
         v
Does this instruction require ring 0?

  YES: check CPL field
         CPL == 0: execute normally
         CPL == 3: raise General Protection Fault
                   CPU stops this instruction
                   CPU switches to ring 0
                   Kernel fault handler runs
                   Kernel sends SIGSEGV to the process
                   Process is terminated

  NO: execute regardless of CPL
      (normal arithmetic, memory within mapped pages —
       these require no privilege check)
```

This is not software checking. The check is wired into the control unit. A ring 3 program cannot disable it, cannot patch it, cannot bypass it. The hardware enforces it unconditionally.

---

## What Each Ring Can and Cannot Do

```
Ring 0 — Kernel Mode:
  Execute any CPU instruction without restriction
  Read and write any memory address
  Configure the CPU (page tables, interrupt handlers, control registers)
  Access hardware ports and registers directly
  Enable and disable interrupts globally
  Switch between processes (context switching)
  Map and unmap memory pages
  Modify privilege levels

Ring 3 — User Mode:
  Execute non-privileged instructions (arithmetic, logic, branches)
  Read and write its own mapped memory pages
  Make system calls (the controlled gate into ring 0)
  Use floating point and vector units
  CANNOT execute privileged instructions (instant fault)
  CANNOT access memory outside its mapped pages (page fault)
  CANNOT directly talk to hardware
  CANNOT modify page tables
  CANNOT disable interrupts
  CANNOT access other processes' memory
```

---

## The System Call Gate — The Only Legal Way Into Ring 0

A user program in ring 3 constantly needs the kernel to do things for it — open files, send network packets, create processes. But the program cannot jump arbitrarily into ring 0 code. The kernel must control exactly where ring 3 can enter.

The solution is a **controlled gate** — a specific CPU instruction that atomically switches privilege level and jumps to a kernel-defined entry point simultaneously. The program has no choice about where it enters.

On x86-64 this gate is the `SYSCALL` instruction (opcode `0x0F 0x05` from Chapter 02):

```
SYSCALL instruction execution — what happens in hardware:

Ring 3 program executes: SYSCALL

1. CPU reads the LSTAR register
   (kernel stored the syscall entry address here during boot)

2. CPU atomically performs all of these simultaneously:
   a. CPL changes from 3 to 0    <- privilege switches to kernel
   b. Stack pointer loads kernel stack
   c. Instruction Pointer jumps to LSTAR address

3. CPU is now in ring 0 executing kernel code
   Kernel reads syscall number from RAX
   Validates all arguments
   Performs the requested operation
   Puts result in RAX

4. Kernel executes SYSRET instruction
   CPU atomically:
   a. CPL changes from 0 back to 3
   b. Instruction Pointer returns to ring 3 code
   c. Stack pointer restores ring 3 stack

5. Ring 3 program resumes with result in RAX

The ring 3 program had no control over WHERE in the kernel it entered.
The CPU forced the jump to the kernel's chosen entry point.
This prevents jumping into the middle of privileged code arbitrarily.
```

Every line in `strace` output — every system call — is one complete ring 3 → ring 0 → ring 3 transition. The transition takes roughly 100-300 nanoseconds on modern hardware.

```bash
# Every line is a ring transition
strace ls 2>&1 | head -20

# Count total ring transitions a program makes
strace -c ls

# Watch ring transitions in real time on any process
strace -p $(pgrep nginx | head -1)
```

---

## Memory Protection — Rings and Page Tables Together

The ring level alone does not prevent one user program from reading another user program's memory. Both run at ring 3. Additional isolation comes from the **page table system** working alongside rings.

Each page table entry has a **User/Supervisor bit** (U/S bit):

```
Page table entry (simplified):

  [Physical address] [...flags...] [U/S] [R/W] [Present]
                                    ^^^
                                    0 = supervisor only (ring 0)
                                    1 = user accessible (ring 0 and 3)

When CPU accesses a memory address:
  1. Walk page table to find physical address
  2. Check U/S bit against current CPL:
     U/S=0 and CPL=3: page fault -> SIGSEGV
     U/S=1 and CPL=3: allowed
     CPL=0: always allowed regardless of U/S

Kernel memory pages: U/S=0
  Ring 3 cannot read kernel memory even if it knows the address
  Hardware raises page fault before the read completes

Process A's pages: U/S=1 but only in process A's page table
  Process B has a completely different page table
  Process B's page table does not map process A's physical pages
  Process B cannot reach process A's memory at all
```

This two-layer protection means:
- Ring level prevents privileged instructions (CPU instruction check)
- U/S bit prevents kernel memory access (page table check)
- Separate page tables prevent cross-process memory access (virtual memory isolation)

---

## SMEP and SMAP — Modern Ring Extensions

Modern x86-64 CPUs added two hardware features that close common exploitation techniques that abused the ring boundary:

### SMEP — Supervisor Mode Execution Prevention (Intel 2011)

When the CPU is in ring 0 (kernel mode), SMEP prevents it from executing instructions from pages marked as user pages (U/S=1).

```
Without SMEP (exploitable):
  Attacker places shellcode in user memory (ring 3 page)
  Attacker tricks kernel into jumping to that address
  Kernel (ring 0) executes shellcode from user memory
  Attacker has kernel privilege

With SMEP:
  Kernel (ring 0) attempts to execute from user page (U/S=1)
  CPU raises fault immediately — before execution
  Attack fails at hardware level
```

### SMAP — Supervisor Mode Access Prevention (Intel 2012)

When in ring 0, SMAP prevents the kernel from accidentally reading or writing user pages unless it explicitly allows it with a special instruction.

```
Without SMAP (exploitable):
  Attacker places fake kernel data structures in user memory
  Attacker tricks kernel into using a pointer into user memory
  Kernel reads attacker-controlled data and uses it for decisions
  Privilege escalation or memory corruption

With SMAP:
  Kernel reads a user-space pointer
  CPU raises fault — cannot access user memory from ring 0
  Kernel must explicitly use STAC instruction to allow user access
  Then immediately CLAC to re-enable SMAP
  Accidental use of user pointers is detected in hardware
```

```bash
# Verify your CPU has these features
grep -m1 "flags" /proc/cpuinfo | tr ' ' '\n' | grep -E "smep|smap"

# See SMAP in use — kernel uses STAC/CLAC around user memory access
sudo objdump -d /boot/vmlinuz-$(uname -r) 2>/dev/null | grep -E "stac|clac" | head -5
```

---

## Rings 1 and 2 — Why They Went Unused

Intel implemented all four rings in hardware expecting operating systems to use them all — device drivers in ring 1, OS services in ring 2, applications in ring 3. Unix never used them. The reasons:

**Performance** — crossing ring boundaries requires CPU work. Using rings 1 and 2 for device drivers means every driver call crosses two boundaries instead of one. The overhead was not worth the isolation benefit on 1970s hardware.

**Portability** — Unix was designed to run on many CPU architectures, many of which had no ring concept. Depending on rings 1 and 2 would bind Unix to specific hardware.

**Simplicity** — the two-level model (kernel trusted, user untrusted, system calls as the gate) was clean, understandable and sufficient.

Windows NT made the same choice. Every major modern operating system uses only rings 0 and 3.

---

## Where Rings 1 and 2 Actually Got Used — Virtualisation

The unused rings 1 and 2 found an unexpected use in the early 2000s for **virtual machines**. VMware needed to run a guest operating system (which expected to be in ring 0) on hardware where the host OS already occupied ring 0.

The solution: run the guest kernel in ring 1 instead. When the guest kernel executed privileged instructions (which it expected to work), they faulted because ring 1 cannot execute them. The VMware hypervisor in ring 0 caught each fault, emulated the operation, and returned control to the guest. The guest kernel never knew it was not really in ring 0.

```
Early VMware ring usage (before hardware virtualisation):

Ring 0: VMware hypervisor (real hardware control)
Ring 1: Guest OS kernel (thinks it is ring 0)
Ring 2: (sometimes guest drivers)
Ring 3: Guest user applications

This was called "ring compression" — complex and had overhead.
```

In 2005-2006, Intel (VT-x) and AMD (AMD-V) added proper hardware virtualisation with a new mode called **VMX root mode** — a privilege level more powerful than ring 0, where the hypervisor runs. Guest operating systems now get their own complete ring 0 in a separate execution context. Ring 1 and ring 2 became irrelevant for virtualisation.

---

## ARM Exception Levels — The Mobile Ring Equivalent

ARM does not use the term "ring." It uses **Exception Levels** (EL), but the concept is identical — a hardware-enforced privilege field in the CPU state that controls what instructions are allowed.

```
ARM Exception Levels:

EL0: Unprivileged (equivalent to x86 ring 3)
  User applications run here
  Cannot execute privileged instructions
  Must use SVC instruction to enter EL1 (like SYSCALL)

EL1: Privileged (equivalent to x86 ring 0)
  Operating system kernel runs here
  Full access to CPU configuration
  Manages page tables, interrupts, processes

EL2: Hypervisor (no x86 equivalent in mainstream use)
  Hypervisor (KVM on Android, Hyper-V on Windows ARM)
  Controls virtualisation
  EL1 and EL0 run inside a virtual machine it manages

EL3: Secure Monitor (no x86 equivalent)
  Most privileged mode
  Controls transitions between Normal World and Secure World
  Cannot be reached from any normal exception path
```

ARM EL0 = x86 Ring 3. ARM EL1 = x86 Ring 0. ARM adds EL2 (hypervisor) and EL3 (secure monitor) that go deeper than anything x86 exposes to mainstream software.

---

## TrustZone — A Second Dimension of Privilege

TrustZone adds an orthogonal (perpendicular) security dimension to the exception level system. Every memory access, every register, every peripheral has an additional property: **Secure** or **Non-Secure**.

```
x86 privilege model (one axis):

Ring 3 ──────────────────────────────────────► Ring 0
User programs                                Kernel
Less privilege                           More privilege

ARM privilege model (two axes):

                    Normal World         Secure World
                    (NS bit = 1)         (NS bit = 0)
                    ────────────         ────────────
EL0 (user):         Android apps         TEE user apps
EL1 (kernel):       Linux kernel         OP-TEE OS
EL2 (hypervisor):   KVM hypervisor       (rare)
EL3 (monitor):      ──── Secure Monitor controls both worlds ────

The Secure Monitor at EL3 is the most privileged code on the system.
A fully compromised Linux kernel (EL1 Normal World) cannot read
Secure World memory — the hardware enforces the NS bit on every access.
```

This is why TrustZone protects fingerprint templates, payment credentials and cryptographic keys even when Android is completely compromised. The hardware refuses to hand Normal World code access to Secure World memory, regardless of privilege level in the Normal World.

---

## How Rings Relate to Every Security Vulnerability

Every privilege escalation attack is an attempt to cross a ring boundary without using the legitimate gate:

```
Ring 3 -> Ring 0 (privilege escalation):
  Exploit kernel vulnerability from user program
  Gain code execution in ring 0
  Disable security policies, install rootkit, read all memory

  Example: kernel buffer overflow
    Ring 3 sends malicious input to kernel subsystem
    Buffer overflow corrupts kernel stack return address
    Kernel returns to attacker shellcode in ring 0
    Full kernel privilege obtained

Ring 0 -> VMX root (hypervisor escape):
  Exploit hypervisor vulnerability from guest kernel
  Gain code execution in hypervisor
  Control all guest VMs on the physical host

EL1 Normal -> Secure World (TrustZone escape):
  Exploit Secure World Trusted Application from Normal World
  Gain code execution inside TrustZone
  Read all cryptographic keys, biometric templates
  Bypass all hardware security guarantees
  Highest value target in mobile security

The syscall gate itself (ring 3 -> ring 0):
  Every system call is a controlled crossing
  Kernel must validate ALL arguments before using them
  Using unvalidated user pointers in kernel = TOCTOU race
  Returning sensitive kernel data to ring 3 = information leak
```

---

## The Complete Ring Picture — From 1964 to Today

```
1964  Multics / GE-645
  Eight hardware rings designed and implemented
  Most ambitious privilege model ever conceived

1969  Unix created (Bell Labs)
  Deliberately simplified Multics
  Uses only two levels: kernel and user
  System calls as the gate — INT 0x80 on early x86

1982  Intel 80286
  First x86 CPU with hardware ring enforcement
  Protected mode with all four rings working

1985  Intel 80386
  Rings work in 32-bit protected mode
  Unix, OS/2, Windows NT all use rings 0 and 3 only

1991  Linux kernel
  Ring 0 = kernel
  Ring 3 = all user processes
  System calls via INT 0x80

2003  AMD64 / x86-64
  Rings preserved in 64-bit mode
  SYSCALL replaces INT 0x80 (faster gate mechanism)

2005  Intel VT-x / AMD-V
  VMX root mode added — more privileged than ring 0
  Hypervisors run in VMX root
  Guest OS runs in VMX non-root with its own ring 0

2011  Intel Sandy Bridge adds SMEP
2012  Intel Broadwell adds SMAP
  Hardware defences close exploitation techniques
  that used ring boundary for shellcode and data attacks

2015  ARM ARMv8-A
  Exception Levels replace simple kernel/user split
  EL0-EL3 plus TrustZone Secure/Non-Secure worlds

Today on your Pop!_OS machine (x86-64):
  VMX root mode  <- KVM hypervisor (if running VMs)
  Ring 0         <- Linux kernel
  Ring 3         <- bash, your applications, everything else

Today on a modern Android phone (ARM64):
  EL3 Secure Monitor
  EL1/EL0 Secure World  <- TrustZone (fingerprint, keys, payments)
  EL2                   <- pKVM hypervisor
  EL1 Normal World      <- Linux kernel
  EL0 Normal World      <- all Android apps
```

---

## Verification — Seeing Rings on Your Machine

```bash
# Ring 0 privilege required — fails from ring 3 without sudo
sudo rdmsr 0x1B          # read CPU model-specific register (ring 0 only)

# The SYSCALL gate address — where ring 3 enters ring 0
sudo rdmsr 0xC0000082    # LSTAR register = address of syscall entry point

# Every strace line is a complete ring 3->ring 0->ring 3 transition
strace ls 2>&1 | head -10

# Count all ring transitions a program makes
strace -c ls

# CPU features related to rings
grep -m1 "flags" /proc/cpuinfo | tr ' ' '\n' | grep -E "smep|smap|vmx|svm"
# vmx = Intel VT-x (VMX root mode support)
# svm = AMD-V (AMD virtualisation)
# smep = supervisor mode execution prevention
# smap = supervisor mode access prevention

# See kernel using SMAP (stac/clac instructions around user memory access)
sudo grep -c "stac\|clac" /proc/kallsyms 2>/dev/null || echo "check objdump"

# Android — see TrustZone interface
# adb shell ls /dev/tee*
# adb shell ls /dev/qseecom
```

---

## Out-of-Order Execution — The CPU Does Not Execute in Order

When you write code, you assume instructions execute one after another in the order you wrote them. On every modern CPU this assumption is false. The CPU reorders instructions internally to keep its execution units busy while waiting for slow operations like memory reads.

### The Problem Out-of-Order Solves

```
Consider this sequence:
  MOV RAX, [address]    ; load from memory — takes 100-300 cycles
  ADD RBX, 1            ; add 1 to RBX — takes 1 cycle
  ADD RCX, 2            ; add 1 to RCX — takes 1 cycle

In-order CPU (simple):
  Cycle 1:   issue MOV RAX, [address]
  Cycles 2-200: STALL — waiting for RAM to return value
  Cycle 201: issue ADD RBX, 1
  Cycle 202: issue ADD RCX, 2
  Total: ~202 cycles, most of them wasted stalling

Out-of-order CPU (modern):
  Cycle 1:   issue MOV RAX, [address] (dispatched to load unit)
  Cycle 2:   issue ADD RBX, 1 (no dependency on RAX — execute now)
  Cycle 3:   issue ADD RCX, 2 (no dependency on RAX — execute now)
  Cycle 100: MOV completes, RAX now has value
  Total: ~100 cycles — 50% faster on this example
```

The CPU detects that ADD RBX and ADD RCX do not depend on the result of the memory load and executes them while waiting. From the programmer's perspective the output is identical — the CPU ensures this — but the execution order is completely different.

### The Reorder Buffer (ROB)

The ROB is a circular buffer inside the CPU that tracks all in-flight instructions in their original program order. Instructions enter the ROB in order, execute out of order, and retire (commit results) in order.

```
ROB structure (simplified):

Entry  Instruction        Status       Result
─────  ─────────────────  ───────────  ──────
 0     MOV RAX,[addr]     EXECUTING    (waiting for memory)
 1     ADD RBX, 1         COMPLETE     RBX = old_RBX + 1
 2     ADD RCX, 2         COMPLETE     RCX = old_RCX + 2
 3     MOV RDX, RAX       WAITING      (depends on entry 0)
 4     ...

Retirement happens from entry 0 first:
  Entry 0 completes (memory returns) → commit RAX, retire entry 0
  Entry 1 already complete → commit RBX, retire entry 1
  Entry 2 already complete → commit RCX, retire entry 2
  Entry 3 can now execute → dispatch to ALU

To the outside world: instructions appear to have executed in order.
Inside the CPU: they ran in a completely different order.
```

### Reservation Stations

Before an instruction can execute, it waits in a **reservation station** — a buffer attached to each execution unit. The instruction waits here until all its input operands are available.

```
Reservation stations for the ALU:

Slot  Instruction   Src1 Ready?  Src1 Value  Src2 Ready?  Src2 Value
────  ────────────  ───────────  ──────────  ───────────  ──────────
 0    ADD RBX, 1    YES          old_RBX     YES          1
 1    ADD RCX, 2    YES          old_RCX     YES          2
 2    ADD RDX, RAX  NO (RAX     ---         YES          ---
                    not ready)

Slots 0 and 1 can execute immediately.
Slot 2 waits until RAX arrives from the memory load.
When RAX becomes available, slot 2 executes.
```

---

## Register Renaming — Eliminating False Dependencies

Consider this sequence:

```
MOV RAX, 5      ; write RAX
ADD RBX, RAX    ; read RAX  ← true dependency: must wait for MOV
MOV RAX, 10     ; write RAX ← writes RAX again
ADD RCX, RAX    ; read RAX  ← reads the NEW RAX from line 3
```

The second `MOV RAX, 10` creates a **write-after-write** hazard with the first `MOV RAX, 5`. The `ADD RCX, RAX` has a **write-after-read** hazard with `ADD RBX, RAX`. These are called **false dependencies** — they only exist because both instructions happen to use the register named RAX, not because one truly needs the other's result.

Register renaming eliminates false dependencies by mapping architectural registers (the 16 registers the programmer sees: RAX-R15) to a larger pool of **physical registers** inside the CPU.

```
Physical register file on modern x86-64: ~180-512 registers
Architectural register file visible to programmer: 16 registers

Renaming example:

Instruction            Architectural  Physical register assigned
─────────────────────  ─────────────  ──────────────────────────
MOV RAX, 5             RAX            P47
ADD RBX, RAX           RAX→P47        (reads P47, writes P23 for RBX)
MOV RAX, 10            RAX            P89   ← NEW physical register!
ADD RCX, RAX           RAX→P89        (reads P89, not P47)

Now:
  ADD RBX, RAX reads P47
  MOV RAX, 10 writes P89
  These are DIFFERENT physical registers
  No false dependency
  Both can execute in parallel with each other
```

Register renaming is done by the **Register Alias Table (RAT)** — a hardware lookup table that maps each architectural register name to its current physical register. Updated on every instruction that writes a register.

---

## Speculative Execution — Running Code Before Knowing If It Should

When the CPU encounters a conditional branch, it does not know which way to go until the condition is evaluated. Evaluating the condition might take many cycles. Waiting would waste time.

**Speculative execution** means the CPU guesses which branch will be taken and starts executing the predicted path immediately without waiting for the condition to be confirmed.

```
if (x > 0) {
    result = expensive_function(x);  // branch taken
} else {
    result = 0;                      // branch not taken
}

Branch predictor guesses: "x is probably > 0, branch will be taken"
CPU immediately starts executing expensive_function(x)
Meanwhile evaluates x > 0...

IF guess correct:
  expensive_function has been running during evaluation
  No time wasted — result is ready sooner
  CPU commits the speculative results

IF guess wrong:
  CPU must discard all speculative results
  Return to architectural state before the branch
  Execute the correct path (result = 0)
  This is called a branch misprediction — costs 15-20 cycles
```

### Branch Predictors

Modern branch predictors are sophisticated pattern-recognition circuits:

```
Static prediction:
  Forward branches (if-then): predict not taken
  Backward branches (loops): predict taken
  Simple rule-based, no history

Dynamic prediction:
  2-bit saturating counter per branch:
    00 = strongly not taken
    01 = weakly not taken
    10 = weakly taken
    11 = strongly taken
  History shifts the counter on each execution

Tournament predictor (modern):
  Combines multiple predictor types
  Uses meta-predictor to choose which sub-predictor to trust
  Intel and AMD branch predictors: >98% accuracy on typical code

Branch Target Buffer (BTB):
  Caches the target address of indirect branches
  call [rax] — where does rax point?
  BTB remembers previous targets and predicts
```

### Spectre and Meltdown — When Speculation Becomes a Vulnerability

Speculative execution is the hardware basis of the **Spectre** and **Meltdown** vulnerabilities (disclosed January 2018).

```
Meltdown (CVE-2017-5754):
  CPU speculatively executes instructions AFTER a page fault
  Page table says: this kernel address is not accessible from ring 3
  CPU checks this, but speculatively executes the next instruction anyway
  The next instruction reads kernel memory into a register
  CPU discovers the fault, discards the register value
  BUT: the memory access already brought data into the cache
  Attacker measures cache timing to infer the discarded value
  Kernel memory read from ring 3 — complete privilege violation

  Fixed by: KPTI (Kernel Page Table Isolation)
    Kernel pages are entirely unmapped from user page tables
    Context switch maps and unmaps kernel pages
    Nothing to speculatively access even with misprediction

Spectre (CVE-2017-5753, CVE-2017-5715):
  More subtle — exploits branch prediction, not just speculation
  Attacker trains branch predictor with attacker-controlled branches
  Victim code's branch predictor is poisoned
  Victim speculatively executes wrong path
  Speculatively executed code accesses secret memory
  Secret leaks via cache timing side channel
  
  Fixed by: retpoline, IBRS, IBPB — software and microcode mitigations
    No clean hardware fix — speculation is fundamental to performance
```

---

## Interrupts vs Exceptions vs Traps — Three Ways to Enter the Kernel

All three transfer control from user code (or hardware) to kernel code. They are different in cause, timing, and intent.

```
INTERRUPT (asynchronous — caused by hardware):
  Source:    hardware device (keyboard, NIC, timer, DMA controller)
  When:      any time — between any two instructions
  Cause:     hardware needs CPU attention
  Examples:  keyboard key pressed, network packet arrived,
             timer fired (10ms scheduler tick),
             DMA transfer complete
  Handling:  CPU finishes current instruction
             Saves RIP and RFLAGS to kernel stack
             Loads handler address from IDT (Interrupt Descriptor Table)
             Executes interrupt handler (ISR)
             Handler services device, sends EOI to interrupt controller
             Returns to interrupted code via IRET instruction

EXCEPTION (synchronous — caused by instruction):
  Source:    CPU itself, triggered by an instruction's behaviour
  When:      immediately when the offending instruction executes
  Cause:     instruction does something illegal or exceptional
  Examples:
    Page fault (#PF):    virtual address not mapped or permission denied
    General Protection Fault (#GP): ring 3 executes privileged instruction
    Divide by Zero (#DE): DIV instruction with divisor = 0
    Invalid Opcode (#UD): CPU sees undefined opcode bytes
    Stack Fault (#SS):   stack segment limit exceeded
    Breakpoint (#BP):    INT3 instruction (debugger breakpoint)
  Handling:  same mechanism as interrupt but triggered by the instruction
             Kernel fault handler examines the cause
             Either fixes it (page fault → load page) or kills process (SIGSEGV)

TRAP (synchronous — deliberate software-initiated):
  Source:    software, intentionally executed
  When:      when the trap instruction executes
  Cause:     deliberate request for kernel service or debugger notification
  Examples:
    SYSCALL (x86-64):   system call — ring 3 requests kernel service
    SVC #0 (ARM64):     system call on ARM
    INT3:               debugger breakpoint (also an exception)
    INT 0x80 (legacy):  old Linux system call method
  Handling:  controlled entry to kernel at known safe entry points
             SYSCALL specifically uses LSTAR register for entry address

COMPARISON TABLE:
  Type        Cause           Timing        Resumable?
  ─────────   ─────────────   ───────────   ──────────────────────────
  Interrupt   Hardware        Async         Yes (code continues after)
  Exception   Bad instruction Sync          Sometimes (page fault yes,
                                            GPF usually kills process)
  Trap        Intentional     Sync          Yes (syscall returns)
```

```
Interrupt Descriptor Table (IDT):
  Array of 256 entries in memory
  Each entry: handler address + privilege requirements
  Entry 0:  divide by zero handler
  Entry 1:  debug exception handler
  Entry 3:  breakpoint handler (INT3)
  Entry 14: page fault handler
  Entry 32-255: hardware interrupt handlers
  Kernel loads IDT address into IDTR register at boot using LIDT instruction
  LIDT is a privileged instruction — ring 0 only
```

---

## The GDT — Global Descriptor Table

The **Global Descriptor Table** is an x86 data structure in memory that defines memory segments and their privilege levels. It is a legacy of the 16-bit segmented memory model but is still required on x86-64.

```
WHY IT EXISTS:
  x86 originally used segmentation (8086, 1978) to divide memory into segments
  The 80286 added protected mode with privilege enforcement via segment descriptors
  x86-64 mostly abandoned segmentation for flat memory model
  But the GDT still exists and is required for ring transitions

WHAT THE GDT CONTAINS TODAY:
  Entry 0: null descriptor (required by CPU, must be all zeros)
  Entry 1: kernel code segment (CPL=0, 64-bit, execute/read)
  Entry 2: kernel data segment (CPL=0, read/write)
  Entry 3: user code segment  (CPL=3, 64-bit, execute/read)
  Entry 4: user data segment  (CPL=3, read/write)
  Entry 5: TSS descriptor     (Task State Segment — holds kernel stack pointer)
  FS/GS base: used for thread-local storage

WHAT THE CS REGISTER HOLDS:
  The CS (Code Segment) register holds a SELECTOR — an index into the GDT
  The bottom 2 bits of the selector = CPL (Current Privilege Level)
  CS selector for kernel code: 0x08 (index 1, CPL=00)
  CS selector for user code:   0x33 (index 6, CPL=11)

  When SYSCALL fires:
    CPU loads the kernel CS selector (CPL=0) from IA32_STAR MSR
    This is how CPL changes from 3 to 0 atomically

TSS (Task State Segment):
  Structure in memory pointed to by GDT TSS entry
  Contains RSP0 — the kernel stack pointer for ring 0
  When interrupt fires in ring 3, CPU reads RSP0 from TSS
  Switches to that kernel stack before calling handler
  Prevents user stack being used for kernel operations

The kernel sets up the GDT at boot:
  lgdt [gdt_pointer]    ; load GDT address into GDTR register
  ltr  [tss_selector]   ; load TSS selector into TR register
  Both are ring 0 privileged instructions
```

---

## Context Switching — What the Kernel Actually Saves and Restores

A **context switch** is the kernel replacing one process's CPU state with another's. Every thread switch, every process switch, every time the scheduler picks a new task — a context switch happens.

```
WHAT MUST BE SAVED (the complete CPU state of the outgoing process):

General-purpose registers:
  RAX, RBX, RCX, RDX, RSI, RDI, RSP, RBP, R8-R15
  (16 registers × 8 bytes = 128 bytes)

Instruction pointer:
  RIP — where this process will resume

Flags:
  RFLAGS — condition codes (ZF, CF, SF, OF, IF...)

Segment registers:
  CS, SS, DS, ES, FS, GS
  FS.base and GS.base (used for thread-local storage)

FPU/SSE/AVX state:
  XMM0-XMM15 (128-bit SSE registers) — 256 bytes
  YMM0-YMM15 (256-bit AVX registers) — 512 bytes
  ZMM0-ZMM31 (512-bit AVX-512 registers) — 2048 bytes
  MXCSR (SSE control/status register)
  x87 FPU stack and control registers
  Saved using FXSAVE or XSAVE instruction (can save 2.5KB+ of state)

Page table pointer:
  CR3 — physical address of this process's PML4 page table
  Loading new CR3 switches the entire virtual address space
  Also flushes TLB (all cached address translations become invalid)
  TLB flush is expensive — PCID (Process Context ID) can avoid some flushes

WHERE THE SAVED STATE GOES:
  Into the kernel's per-task data structure: task_struct in Linux
  Specifically in the thread field: struct thread_struct
  Stored in kernel memory — the process cannot access its own task_struct
```

```
CONTEXT SWITCH SEQUENCE (Linux, simplified):

1. Interrupt fires (timer, 10ms tick) or process blocks (I/O wait)
2. CPU saves RIP and RFLAGS to kernel stack (hardware, automatic)
3. CPU switches to kernel stack (RSP0 from TSS)
4. Interrupt handler runs in kernel (ring 0)
5. Handler calls schedule()
6. schedule() calls context_switch(prev, next)
7. switch_mm(prev, next):
   - load new CR3 (change page tables — new virtual address space)
   - TLB flush (or update PCID)
8. switch_to(prev, next):
   - XSAVE: save FPU/SSE/AVX state of prev task
   - Save general registers of prev task to prev->thread
   - Load general registers of next task from next->thread
   - XRSTOR: restore FPU/SSE/AVX state of next task
   - Load FS.base and GS.base for next task (thread-local storage)
9. Return to next task's RIP (resume where it was interrupted)
10. IRET: restore RFLAGS, switch CS (ring 0 → ring 3), restore RSP

From the next process's perspective: nothing happened.
It resumes exactly where it left off, all registers intact.

Context switch overhead: ~1-10 microseconds
  TLB flush is the most expensive part (~1-3 microseconds)
  Threads in the same process share page tables — no TLB flush needed
  This is one reason threads are cheaper than processes
```

---

## ELF Format in Detail — What Is Inside an Executable

Every compiled program on Linux is an ELF (Executable and Linkable Format) file. Understanding ELF explains why programs work the way they do, why ASLR randomises specific addresses, and how buffer overflow exploits target specific sections.

```
ELF FILE STRUCTURE:

┌───────────────────────────────────────────────────┐
│  ELF Header (64 bytes)                            │
│    Magic: 7F 45 4C 46 ("ELF" in ASCII)            │
│    Class: 64-bit                                  │
│    Data: little-endian                            │
│    OS/ABI: Linux                                  │
│    Type: ET_EXEC (executable) or ET_DYN (PIE)     │
│    Machine: x86-64 (0x3E) or ARM64 (0xB7)         │
│    Entry point: 0x401000 (first instruction)      │
│    PHoff: offset to program header table          │
│    SHoff: offset to section header table          │
├───────────────────────────────────────────────────┤
│  Program Header Table (for the OS loader)         │
│    LOAD segment 1: .text (code) — r-x             │
│    LOAD segment 2: .data+.bss (data) — rw-        │
│    DYNAMIC segment: dynamic linking info          │
│    GNU_STACK: stack permissions (usually rw- NX)  │
│    GNU_RELRO: read-only after relocation          │
├───────────────────────────────────────────────────┤
│  .text section — compiled machine code            │
│    Your functions as opcode bytes                 │
│    Read-only + executable                         │
│    NX bit NOT set here (this IS code)             │
├───────────────────────────────────────────────────┤
│  .rodata section — read-only data                 │
│    String literals: "hello\n"                     │
│    Constant arrays                                │
│    Read-only, NOT executable                      │
├───────────────────────────────────────────────────┤
│  .data section — initialised global variables     │
│    int x = 5;  ← lives here (value 5 stored)      │
│    Read-write, NOT executable                     │
├───────────────────────────────────────────────────┤
│  .bss section — uninitialised global variables    │
│    int y;  ← lives here (no bytes in file)        │
│    Kernel zeroes this region when loading         │
│    Takes no space in ELF file itself              │
│    Read-write, NOT executable                     │
├───────────────────────────────────────────────────┤
│  .plt section — Procedure Linkage Table           │
│    Stubs for calling shared library functions     │
│    Covered in dynamic linking section below       │
├───────────────────────────────────────────────────┤
│  .got section — Global Offset Table               │
│    Addresses resolved at runtime by dynamic linker│
│    Covered in dynamic linking section below       │
├───────────────────────────────────────────────────┤
│  .symtab — symbol table (debug builds)            │
│    Maps function/variable names to addresses      │
│    Stripped in production binaries                │
├───────────────────────────────────────────────────┤
│  Section Header Table (for linker/debugger)       │
│    Describes all sections above                   │
│    Not needed at runtime (OS uses program hdrs)   │
└───────────────────────────────────────────────────┘
```

```bash
# Inspect an ELF binary
readelf -h /bin/ls          # ELF header
readelf -S /bin/ls          # all sections
readelf -l /bin/ls          # program headers (segments)
objdump -d /bin/ls          # disassemble .text section
objdump -s -j .rodata /bin/ls  # dump .rodata contents
size /bin/ls                # sizes of text, data, bss

# See the magic bytes
xxd /bin/ls | head -2
# 7f 45 4c 46 = ELF magic

# PIE (Position Independent Executable) vs non-PIE
file /bin/ls
# ... ELF 64-bit LSB pie executable ...
# PIE: base address randomised by ASLR each run

# Check sections of your own compiled program
gcc -o myprogram myprogram.c
readelf -S myprogram
```

---

## The Linker in Depth — Symbol Resolution, Relocation and the GOT/PLT

The linker takes multiple object files (.o) and produces a final executable. Understanding it explains how function calls across files work and why the PLT and GOT exist.

### Symbol Resolution

When you compile multiple .c files separately, each produces a .o file. References between files are **unresolved symbols** — placeholders that must be filled in.

```
File A: main.c
  calls printf() — unresolved: where is printf?
  calls helper() — unresolved: where is helper?

File B: helper.c
  defines helper() — provides the symbol
  calls malloc() — unresolved: where is malloc?

Static library: libc.a
  contains printf.o: defines printf()
  contains malloc.o: defines malloc()

Linker combines all of these:
  1. Collect all object files and needed library objects
  2. Build a symbol table: name → address
  3. Resolve every unresolved reference
  4. Assign final virtual addresses to every symbol
  5. Patch all call instructions with correct addresses
  6. Output ELF executable
```

### Static vs Dynamic Linking

```
STATIC LINKING:
  Linker copies all library code into the executable
  Final binary contains: your code + printf + malloc + everything
  Self-contained: runs without any external .so files
  Larger binary size
  No version conflicts
  Used for: embedded systems, containers, security-critical tools

DYNAMIC LINKING:
  Linker records which shared libraries are needed
  Does NOT copy library code into executable
  At runtime: OS loads the shared library (.so file) into memory
  Multiple programs share one copy of the library in memory
  Smaller binary size
  Library updates benefit all programs automatically
  Used for: almost everything on a normal Linux system

Check which type:
  file /bin/ls
  # dynamically linked
  ldd /bin/ls
  # shows all shared libraries the binary needs:
  # libc.so.6, libselinux.so.1, etc.

  ldd /bin/busybox
  # statically linked — no dependencies
```

### The PLT and GOT — Dynamic Linking at Runtime

When a dynamically linked program calls `printf()`, the address of `printf` is not known at compile time — it depends on where the dynamic linker places libc.so in memory. The PLT and GOT solve this.

```
PLT (Procedure Linkage Table) — in .plt section of executable:
  A trampoline stub for each external function
  printf@plt:
    JMP [printf@got]    ← indirect jump through GOT entry
    PUSH index          ← only executed first time
    JMP plt_resolver    ← only executed first time

GOT (Global Offset Table) — in .got section of executable:
  Array of 8-byte addresses, one per external function
  Populated by dynamic linker at program start (or lazily)
  Initially points back into PLT for lazy resolution
  After first call: contains real address of function

FIRST CALL to printf():
  1. CPU executes: CALL printf@plt
  2. PLT stub: JMP [printf@got]
  3. GOT entry points back to PLT resolver (not resolved yet)
  4. PLT resolver calls dynamic linker
  5. Dynamic linker finds printf in libc.so
  6. Dynamic linker writes real printf address into GOT
  7. Jumps to printf — function executes

SUBSEQUENT CALLS to printf():
  1. CPU executes: CALL printf@plt
  2. PLT stub: JMP [printf@got]
  3. GOT entry now has real printf address
  4. Jumps directly to printf
  No resolver involved — just two jumps

This is called lazy binding — resolution on first call only.
```

```bash
# See PLT entries
objdump -d /bin/ls | grep -A3 "@plt"

# See GOT entries
objdump -R /bin/ls          # dynamic relocations (GOT entries)
readelf -r /bin/ls          # all relocations

# Trace dynamic linking at runtime
LD_DEBUG=bindings ls 2>&1 | head -20
# Shows every symbol resolution as it happens
```

### Security Implications of PLT and GOT

```
GOT OVERWRITE ATTACK:
  If attacker can write to the GOT (writable memory region)
  They can replace printf@got with address of their shellcode
  Next call to printf() jumps to shellcode instead
  Classic ret2plt / GOT hijacking technique

RELRO (Relocation Read-Only) — defence against GOT overwrite:
  Partial RELRO: GOT is writable after dynamic linking
  Full RELRO:    dynamic linker resolves ALL symbols at startup
                 then marks GOT as read-only (mprotect)
                 GOT overwrite impossible after startup

Check RELRO:
  checksec --file=/bin/ls
  # Shows: Full RELRO, Partial RELRO, or No RELRO

In CTF/pentest: GOT overwrite only works without Full RELRO
In modern systems: most binaries have Full RELRO
```

---

## Atomic Operations and Memory Barriers

On a multi-core CPU, multiple cores execute instructions simultaneously. Without coordination, concurrent access to shared memory produces unpredictable results.

### The Race Condition at Hardware Level

```
Two threads, both executing: counter++;
counter++ compiles to:
  MOV EAX, [counter]    ; read
  ADD EAX, 1            ; increment
  MOV [counter], EAX    ; write back

Without synchronisation (both cores execute simultaneously):

Core 0:                          Core 1:
  MOV EAX, [counter]  → EAX=5
                                   MOV EAX, [counter]  → EAX=5
  ADD EAX, 1          → EAX=6
  MOV [counter], EAX  → mem=6
                                   ADD EAX, 1          → EAX=6
                                   MOV [counter], EAX  → mem=6

Result: counter=6 instead of 7
One increment was lost
```

### Atomic Operations — Hardware Support

The x86 LOCK prefix makes a read-modify-write operation atomic at the hardware level:

```
LOCK ADD [counter], 1

What happens:
  CPU asserts the LOCK# signal on the memory bus
  Other cores cannot access [counter] until this completes
  Read → increment → write happens as one indivisible operation
  Other cores see either the old value or the new value
  Never an intermediate state

x86-64 atomic instructions:
  LOCK ADD    atomic add
  LOCK XCHG   atomic exchange (always locks even without LOCK prefix)
  LOCK CMPXCHG compare-and-swap — the basis of all lock-free algorithms
  LOCK INC, LOCK DEC, LOCK OR, LOCK AND, LOCK XOR

ARM64 atomic operations:
  LDXR/STXR   load-exclusive / store-exclusive
              STXR fails if another core modified the location
              Retry loop if it fails
  LDADD, STADD  atomic add (ARMv8.1+)
  CAS         compare and swap (ARMv8.1+)
```

### Memory Barriers — Enforcing Order

Even with atomic operations, the CPU and memory system can reorder memory accesses. A **memory barrier** forces ordering constraints.

```
TYPES OF REORDERING:

Store-store reordering:
  Store to A, then store to B
  CPU may commit B to memory before A
  Other cores see B updated but A not yet

Load-load reordering:
  Load from A, then load from B
  CPU may fetch B before A
  Reads appear out of order

Store-load reordering (most common on x86):
  Store to A, then load from B
  CPU may execute the load before the store is visible

x86-64 memory model (TSO — Total Store Order):
  Most reorderings prevented by hardware
  Only store-load reordering is possible
  Relatively strong memory model

ARM64 memory model (weakly ordered):
  All four reorderings possible
  Requires explicit barriers for correct multi-core code

Memory barrier instructions:
  x86-64: MFENCE (full barrier), LFENCE (load), SFENCE (store)
  ARM64:  DMB (data memory barrier), DSB (data sync barrier)
  Linux kernel: smp_mb(), smp_rmb(), smp_wmb() — architecture-neutral

In kernel code:
  spin_lock() contains an implicit memory barrier
  All lock/unlock operations include appropriate barriers
  Needed for: device drivers, lock-free data structures, RCU
```

---

## CPUID — How Software Queries CPU Capabilities

The CPUID instruction allows software to query what features the CPU supports. The kernel uses this extensively at boot to configure itself.

```
CPUID instruction:
  MOV EAX, leaf_number    ; what information do you want?
  CPUID                   ; ask the CPU
  ; Results in EAX, EBX, ECX, EDX

Common CPUID leaves:

Leaf 0x0 (basic info):
  EAX: highest supported basic leaf
  EBX, EDX, ECX: vendor string
  GenuineIntel = 47 65 6E 75 49 6E 74 65 6C
  AuthenticAMD = 41 75 74 68 65 6E 74 69 63

Leaf 0x1 (feature flags):
  EDX bit 25: SSE    — 128-bit SIMD
  EDX bit 26: SSE2   — double-precision SIMD
  ECX bit 28: AVX    — 256-bit SIMD
  ECX bit 5:  VMX    — Intel virtualisation (VT-x)
  ECX bit 30: RDRAND — hardware random number generator

Leaf 0x7 subleaf 0 (extended features):
  EBX bit 0:  FSGSBASE — FS/GS base read/write
  EBX bit 7:  SMEP     — Supervisor Mode Execution Prevention
  EBX bit 20: SMAP     — Supervisor Mode Access Prevention
  ECX bit 2:  UMIP     — User-Mode Instruction Prevention
  ECX bit 5:  AVX512F  — AVX-512 foundation

Leaf 0x80000001 (AMD extended features):
  EDX bit 20: NX/XD    — No-Execute bit support
  EDX bit 29: LM       — Long Mode (64-bit support)
```

```bash
# Query CPUID from userspace
cpuid -1                    # all leaves (install cpuid package)
cat /proc/cpuinfo | grep flags  # kernel-parsed CPUID features

# Key flags to check for security features:
grep -m1 "flags" /proc/cpuinfo | tr ' ' '\n' | grep -E \
  "smep|smap|nx|pse|pae|vmx|svm|aes|avx|rdrand|fsgsbase"

# Check in kernel at boot time
dmesg | grep -i "CPU\|feature\|SMEP\|SMAP\|PAE\|NX"
```

The kernel reads CPUID at boot and enables features accordingly:

```
Boot sequence (kernel CPU detection):
  CPU powers on
  Kernel calls cpu_detect() or setup_cpu_features()
  CPUID leaf 0x1: check for SSE2, SSE4, AVX
  CPUID leaf 0x7: check for SMEP, SMAP, AVX-512
  If SMEP found: enable by setting CR4.SMEP bit
  If SMAP found: enable by setting CR4.SMAP bit
  If NX found:   enable by setting IA32_EFER.NXE MSR bit
  Kernel configures itself based on what hardware supports
```

---

## Compiler IR — Intermediate Representation

Between C source code and machine code, compilers use an **Intermediate Representation (IR)** — a platform-neutral form of the program that is easier to optimise than either source code or machine code.

```
WHY IR EXISTS:

Without IR:
  C frontend → x86-64 machine code directly
  To support ARM64: C frontend → ARM64 machine code directly
  To support RISC-V: C frontend → RISC-V directly
  N languages × M architectures = N×M compiler backends

With IR (LLVM approach):
  C frontend → LLVM IR
  Rust frontend → LLVM IR
  Swift frontend → LLVM IR
  LLVM IR → x86-64 backend
  LLVM IR → ARM64 backend
  LLVM IR → RISC-V backend
  N languages + M architectures = N frontends + M backends
  Any language compiles to any architecture
```

### LLVM IR — What It Looks Like

```c
// C source:
int add(int a, int b) {
    return a + b;
}
```

```llvm
; LLVM IR:
define i32 @add(i32 %a, i32 %b) {
entry:
  %result = add i32 %a, %b
  ret i32 %result
}
```

```asm
; x86-64 machine code (from LLVM IR):
add:
  lea eax, [rdi + rsi]  ; result = a + b
  ret

; ARM64 machine code (from same LLVM IR):
add:
  add w0, w0, w1        ; result = a + b
  ret
```

The same LLVM IR produces correct machine code for both architectures. The frontend (Clang, rustc) only needs to produce IR. The backend handles each architecture.

### GCC GIMPLE IR

GCC uses its own IR called **GIMPLE** — a simplified three-address form:

```c
// C source:
int x = (a + b) * (c + d);
```

```gimple
// GIMPLE (three-address code — max one operation per statement):
tmp1 = a + b;
tmp2 = c + d;
x = tmp1 * tmp2;
```

GIMPLE is what GCC optimises. After optimisation, it converts GIMPLE to RTL (Register Transfer Language) and then to machine code.

```bash
# See LLVM IR
clang -emit-llvm -S -o output.ll source.c
cat output.ll

# See GCC GIMPLE
gcc -fdump-tree-gimple source.c
cat source.c.004t.gimple

# See GCC RTL (closer to machine code)
gcc -fdump-rtl-expand source.c
```

---

## Shared Libraries and the Dynamic Linker

When a program needs printf(), it does not contain printf. It references libm.so or libc.so. At runtime, the **dynamic linker** (ld.so / ld-linux.so) loads those libraries and connects the references.

```
DYNAMIC LINKER SEQUENCE when ./myprogram starts:

1. Kernel loads myprogram ELF into memory
   Maps .text, .data, .bss, .plt, .got

2. Kernel reads PT_INTERP program header
   Contains path: /lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
   This is the dynamic linker itself

3. Kernel loads ld-linux.so into memory
   Jumps to ld-linux.so's entry point (not myprogram's yet)

4. ld-linux.so reads myprogram's PT_DYNAMIC segment
   Contains list of needed shared libraries:
     NEEDED libm.so.6
     NEEDED libc.so.6

5. ld-linux.so finds and loads each library:
   Search path: LD_LIBRARY_PATH, /etc/ld.so.cache, /lib, /usr/lib
   mmap() each .so file into process memory
   Each library gets its own virtual address region (ASLR randomised)

6. ld-linux.so performs relocations:
   For each GOT entry needing resolution:
     Find the symbol in the loaded libraries
     Write the real address into the GOT entry
   (Or defers to PLT lazy resolution for lazily-bound symbols)

7. ld-linux.so calls each library's initialisation functions
   .init_array section: constructors run before main()

8. ld-linux.so jumps to myprogram's entry point (_start)

9. _start calls __libc_start_main() which calls main()
```

```bash
# See the dynamic linker path
readelf -l /bin/ls | grep INTERP
# [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]

# Trace dynamic linker activity
LD_DEBUG=all ls 2>&1 | head -50

# See library load addresses (ASLR in effect — different each run)
ldd /bin/ls
ldd /bin/ls    # run again — addresses differ

# Preload your own library (hook functions)
LD_PRELOAD=/path/to/mylib.so ls
# Your library's functions override libc's — used for Frida-style hooking
```

### Security Implications of the Dynamic Linker

```
LD_PRELOAD hijacking:
  Set LD_PRELOAD to a malicious .so before running a program
  Your library's functions called instead of real library functions
  Can intercept: open(), read(), write(), strcmp(), crypt()
  Used by: Frida (legitimate), rootkits (malicious)
  Mitigated: setuid/setgid binaries ignore LD_PRELOAD

Library search order manipulation:
  LD_LIBRARY_PATH can redirect library loading
  Planting a malicious libcrypto.so.1 in the search path
  Intercepting crypto operations
  Mitigated: same — ignored for setuid binaries

DT_RPATH and DT_RUNPATH:
  Compiled-in library search paths (override LD_LIBRARY_PATH)
  DT_RPATH: searched before LD_LIBRARY_PATH (bad practice)
  DT_RUNPATH: searched after LD_LIBRARY_PATH
  Check with: readelf -d binary | grep -E "RPATH|RUNPATH"

Checking library integrity:
  checksec --file=/bin/ls  # shows all security properties
  ldd --verify binary      # verifies library dependencies
```


---

## Part III — The Software Foundation

*Chapters 6-7 explain how human-readable source code becomes machine code the CPU executes. The assembler, the compiler, the ELF format, and the complete flow from writing code to running it.*

---


---


*How human-readable code becomes machine code. The first assembler bootstrapped by hand. The compiler pipeline. The self-hosting moment. The Trusting Trust attack. The complete bootstrap chain from toggle switches to GCC.*

---
