# RvInst

This document provides an API reference for the `RvInst` struct, which is used to hold the decoded components of a RISC-V instruction.

---
## Structs

### `RvInst`
Represents a decoded RISC-V instruction, containing its various fields extracted from its binary encoding, as well as metadata assigned during the decoding or instrumentation process.

```c
typedef struct{
    int32_t funct6;      // 6-bit function field (used in some compressed or custom formats)
    int32_t rs1;         // 5-bit first source register identifier
    int32_t op;          // 2-bit opcode (primarily for 16-bit compressed instructions)
    int32_t rd_rs1p;     // 3-bit destination or source register identifier, typically for compressed instructions using registers x8-x15 (rd'/rs1')
    int32_t aq;          // 1-bit acquire bit (for atomic instructions)
    int32_t funct4;      // 4-bit function field (used in some compressed formats like CR)
    int32_t funct7;      // 7-bit function field (used in R-type instructions)
    int32_t rdp;         // 3-bit destination register identifier, typically for compressed instructions using registers x8-x15 (rd')
    int32_t rs3;         // 5-bit third source register identifier (used in R4-type instructions like FMA)
    int32_t rd;          // 5-bit destination register identifier
    int32_t funct2;      // 2-bit function field (used in R4-type or some compressed formats)
    int32_t rl;          // 1-bit release bit (for atomic instructions)
    int32_t rd_rs1;      // 5-bit destination or source register identifier (can be rd or rs1 depending on instruction format, e.g., in CI-type compressed instructions or as a general field)
    int32_t rs2;         // 5-bit second source register identifier
    int32_t imm;         // 32-bit field to store the various forms of immediate values, sign-extended or zero-extended as appropriate
    int32_t opcode;      // 7-bit opcode (for 32-bit instructions)
    int32_t rs2p;        // 3-bit second source register identifier, typically for compressed instructions using registers x8-x15 (rs2')
    int32_t funct3;      // 3-bit function field
    int32_t funct5;      // 5-bit function field (used in AMO instructions in place of funct7)
    int32_t rs1p;        // 3-bit first source register identifier, typically for compressed instructions using registers x8-x15 (rs1')
    uint64_t group;      // Custom group identifier for the instruction (e.g., assigned via ARCH-VISOR for targeted instrumentation)
    uint64_t address;    // Memory address of the instruction in the original binary
    InstName name;       // Enum or type representing the mnemonic name of the instruction (e.g., ADD, LW, BEQ)
    InstType type;       // Enum or type representing the format/type of the instruction (e.g., R_TYPE, I_TYPE, C_LW_TYPE)
} RvInst;
```

* **`funct6`**: A 6-bit function field, often used in specific compressed instruction formats (e.g., C.ANDI's `funct2` combined with other bits, or parts of other custom instruction encodings).
* **`rs1`**: The 5-bit identifier for the first source register operand (e.g., for R, I, S, B types).
* **`op`**: A 2-bit opcode field, primarily used for identifying 16-bit compressed instructions (e.g., C.ADDI uses `op=01`).
* **`rd_rs1p`**: A 3-bit register field, typically representing `rd'` or `rs1'` (registers x8-x15) in certain compressed instruction formats.
* **`aq`**: The 1-bit "acquire" flag used in atomic memory operations (AMO) to specify memory ordering constraints.
* **`funct4`**: A 4-bit function field, used in some compressed instruction formats like CR-type (e.g., C.ADD, C.MV).
* **`funct7`**: The 7-bit function field used in 32-bit R-type instructions to differentiate operations (e.g., ADD vs SUB).
* **`rdp`**: A 3-bit register field, typically representing `rd'` (registers x8-x15) in certain compressed instruction formats (e.g., CIW-type like C.ADDIW).
* **`rs3`**: The 5-bit identifier for the third source register operand, used in R4-type instructions (e.g., floating-point fused multiply-add instructions).
* **`rd`**: The 5-bit identifier for the destination register operand (e.g., for R, I, U, J types).
* **`funct2`**: A 2-bit function field, used in R4-type instructions or some compressed formats.
* **`rl`**: The 1-bit "release" flag used in atomic memory operations (AMO) to specify memory ordering constraints.
* **`rd_rs1`**: A 5-bit register field that can serve as either `rd` or `rs1` depending on the instruction context, common in CI-type compressed instructions.
* **`rs2`**: The 5-bit identifier for the second source register operand (e.g., for R, S, B types).
* **`imm`**: A field (typically `int32_t` to hold sign-extended values) used to store the various forms of immediate values extracted from instructions. The specific bits and their interpretation depend on the instruction type (I, S, B, U, J, CI, etc.).
* **`opcode`**: The 7-bit primary opcode for 32-bit instructions, or can store the effective 7-bit opcode for expanded compressed instructions.
* **`rs2p`**: A 3-bit register field, typically representing `rs2'` (registers x8-x15) in certain compressed instruction formats.
* **`funct3`**: The 3-bit function field used in many 32-bit and 16-bit instruction formats to further specify the operation (e.g., distinguishing ADDI from SLTI, or LW from LH).
* **`funct5`**: A 5-bit function field used in atomic memory operations (AMO) instead of `funct7` to define the atomic operation.
* **`rs1p`**: A 3-bit register field, typically representing `rs1'` (registers x8-x15) in certain compressed instruction formats.
* **`group`**: A 64-bit identifier for a custom instruction group. This can be assigned via tools like ARCH-VISOR to categorize instructions for targeted instrumentation or analysis.
* **`address`**: The original memory address of this instruction within the instrumented binary.
* **`name`**: An `InstName` (likely an enum or typedef) representing the assembly mnemonic or a unique identifier for the instruction (e.g., `ADD`, `LW`, `C_BEQZ`).
* **`type`**: An `InstType` (likely an enum or typedef defined via ARCH-VISOR) representing the decoded instruction format or a more specific classification (e.g., `R_TYPE`, `I_TYPE_LOAD`, `C_BRANCH_TYPE`).

<br/><br/>