# Encoder 

This document provides an API reference for the RISC-V instruction encoder functions defined in R-Visor. These functions are used to construct the 32-bit binary representation of RISC-V instructions from their constituent fields (like opcode, registers, and immediate values) or from higher-level instruction mnemonics. All functions are `inline` and return a `uint32_t` representing the encoded instruction.

---
## Low-Level Format Encoders

These functions assemble a 32-bit or 16-bit instruction word based on the specified RISC-V instruction format and the provided fields.

---
### `uint32_t encode_Rtype(int opcode, int rd, int funct3, int rs1, int rs2, int funct7)`
Encodes a 32-bit R-type instruction.

* **Parameters**:
    * `opcode`: The 7-bit opcode.
    * `rd`: The 5-bit destination register.
    * `funct3`: The 3-bit funct3 field.
    * `rs1`: The 5-bit first source register.
    * `rs2`: The 5-bit second source register.
    * `funct7`: The 7-bit funct7 field.
* **Returns**: The 32-bit encoded instruction.

---
### `uint32_t encode_R4type(int opcode, int rd, int funct3, int rs1, int rs2, int funct2, int rs3)`
Encodes a 32-bit R4-type instruction (e.g., for fused multiply-add operations).

* **Parameters**:
    * `opcode`: The 7-bit opcode.
    * `rd`: The 5-bit destination register.
    * `funct3`: The 3-bit funct3 field.
    * `rs1`: The 5-bit first source register.
    * `rs2`: The 5-bit second source register.
    * `funct2`: The 2-bit funct2 field.
    * `rs3`: The 5-bit third source register.
* **Returns**: The 32-bit encoded instruction.

---
### `uint32_t encode_RAMOtype(int opcode, int rd, int funct3, int rs1, int rs2, int rl, int aq, int funct5)`
Encodes a 32-bit R-type instruction specifically for Atomic Memory Operations (AMO), incorporating `aq` and `rl` bits and a 5-bit `funct5` field instead of `funct7`.

* **Parameters**:
    * `opcode`: The 7-bit opcode.
    * `rd`: The 5-bit destination register.
    * `funct3`: The 3-bit funct3 field.
    * `rs1`: The 5-bit first source register (base address).
    * `rs2`: The 5-bit second source register (source data).
    * `rl`: The 1-bit release flag.
    * `aq`: The 1-bit acquire flag.
    * `funct5`: The 5-bit AMO function field.
* **Returns**: The 32-bit encoded instruction.

---
### `uint32_t encode_Itype(int opcode, int rd, int funct3, int rs1, int imm)`
Encodes a 32-bit I-type instruction.

* **Parameters**:
    * `opcode`: The 7-bit opcode.
    * `rd`: The 5-bit destination register.
    * `funct3`: The 3-bit funct3 field.
    * `rs1`: The 5-bit source register.
    * `imm`: The 12-bit immediate value.
* **Returns**: The 32-bit encoded instruction.

---
### `uint32_t encode_Stype(int opcode, int imm, int funct3, int rs1, int rs2)`
Encodes a 32-bit S-type instruction.

* **Parameters**:
    * `opcode`: The 7-bit opcode.
    * `imm`: The 12-bit immediate value (split into imm\[4:0\] and imm\[11:5\]).
    * `funct3`: The 3-bit funct3 field.
    * `rs1`: The 5-bit first source register (base address).
    * `rs2`: The 5-bit second source register (source data).
* **Returns**: The 32-bit encoded instruction.

---
### `uint32_t encode_Utype(int opcode, int rd, int imm)`
Encodes a 32-bit U-type instruction.

* **Parameters**:
    * `opcode`: The 7-bit opcode.
    * `rd`: The 5-bit destination register.
    * `imm`: The 20-bit immediate value (imm\[31:12\]).
* **Returns**: The 32-bit encoded instruction.

---
### `uint32_t encode_Btype(int opcode, int imm, int funct3, int rs1, int rs2)`
Encodes a 32-bit B-type instruction.

* **Parameters**:
    * `opcode`: The 7-bit opcode.
    * `imm`: The 12-bit branch offset (technically 13-bit, imm\[12|10:5|4:1|11\], but input is a signed 12-bit multiple of 2).
    * `funct3`: The 3-bit funct3 field.
    * `rs1`: The 5-bit first source register.
    * `rs2`: The 5-bit second source register.
* **Returns**: The 32-bit encoded instruction.

---
### `uint32_t encode_Jtype(int opcode, int rd, int imm)`
Encodes a 32-bit J-type instruction.

* **Parameters**:
    * `opcode`: The 7-bit opcode.
    * `rd`: The 5-bit destination register.
    * `imm`: The 20-bit jump offset (technically 21-bit, imm\[20|10:1|11|19:12\], but input is a signed 20-bit multiple of 2).
* **Returns**: The 32-bit encoded instruction.

---
### `uint32_t encode_CRtype(int op, int rs2, int rd_rs1, int funct4)`
Encodes a 16-bit CR-format compressed instruction.

* **Parameters**:
    * `op`: The 2-bit opcode.
    * `rs2`: The 5-bit second source register.
    * `rd_rs1`: The 5-bit destination/first source register.
    * `funct4`: The 4-bit function field.
* **Returns**: The 16-bit encoded instruction (as uint32_t).

---
### `uint32_t encode_CRItype(int op, int rs2p, int funct2, int rd_rs1p, int funct6)`
Encodes a 16-bit compressed instruction format, likely a variant of CR or CI for specific instructions not fitting other narrow formats. `p` might denote a restricted register set (e.g., x8-x15).

* **Parameters**:
    * `op`: The 2-bit opcode.
    * `rs2p`: A 3-bit source register field.
    * `funct2`: A 2-bit function field.
    * `rd_rs1p`: A 3-bit destination/source register field.
    * `funct6`: A 6-bit function field.
* **Returns**: The 16-bit encoded instruction.

---
### `uint32_t encode_CItype(int op, int imm, int rd_rs1, int funct3)`
Encodes a 16-bit CI-format compressed instruction.

* **Parameters**:
    * `op`: The 2-bit opcode.
    * `imm`: A 6-bit immediate value (split as imm\[5\] and imm\[4:0\]).
    * `rd_rs1`: The 5-bit destination/source register.
    * `funct3`: The 3-bit function field.
* **Returns**: The 16-bit encoded instruction.

---
### `uint32_t encode_CIStype(int op, int imm, int rd_rs1, int funct3)`
Encodes a 16-bit compressed instruction format, likely a CI variant for stack operations (e.g. C.LWSP, C.FLWSP). The immediate encoding is specific to this format.

* **Parameters**:
    * `op`: The 2-bit opcode.
    * `imm`: An immediate value (bits are specifically placed).
    * `rd_rs1`: The 5-bit destination register (for loads from stack pointer).
    * `funct3`: The 3-bit function field.
* **Returns**: The 16-bit encoded instruction.

---
### `uint32_t encode_CISDtype(int op, int imm, int rd_rs1, int funct3)`
Encodes a 16-bit compressed instruction format, likely a CI variant for stack-pointer relative double-word loads (e.g. C.LDSP, C.FLDSP).

* **Parameters**:
    * `op`: The 2-bit opcode.
    * `imm`: An immediate value with specific bit placement.
    * `rd_rs1`: The 5-bit destination register.
    * `funct3`: The 3-bit function field.
* **Returns**: The 16-bit encoded instruction.

---
### `uint32_t encode_CI16type(int op, int imm, int rd_rs1, int funct3)`
Encodes a 16-bit compressed instruction, specifically C.ADDI16SP.

* `op`: The 2-bit opcode.
* `imm`: The 6-bit non-zero immediate for C.ADDI16SP (imm\[9|4|6|8:7|5\]).
* `rd_rs1`: The destination register (must be x2/SP).
* `funct3`: The 3-bit function field.
* **Returns**: The 16-bit encoded instruction.

---
### `uint32_t encode_CSStype(int op, int rs2, int imm, int funct3)`
Encodes a 16-bit CSS-format compressed instruction (e.g., C.SWSP, C.FSWSP).

* **Parameters**:
    * `op`: The 2-bit opcode.
    * `rs2`: The 5-bit source register.
    * `imm`: An immediate value with specific bit placement for stack stores.
    * `funct3`: The 3-bit function field.
* **Returns**: The 16-bit encoded instruction.

---
### `uint32_t encode_CSSDtype(int op, int rs2, int imm, int funct3)`
Encodes a 16-bit CSS-format compressed instruction for stack-pointer relative double-word stores (e.g. C.SDSP, C.FSDSP).

* **Parameters**:
    * `op`: The 2-bit opcode.
    * `rs2`: The 5-bit source register.
    * `imm`: An immediate value with specific bit placement.
    * `funct3`: The 3-bit function field.
* **Returns**: The 16-bit encoded instruction.

---
### `uint32_t encode_CIWtype(int op, int rdp, int imm, int funct3)`
Encodes a 16-bit CIW-format compressed instruction (e.g., C.ADDIW).

* **Parameters**:
    * `op`: The 2-bit opcode.
    * `rdp`: The 3-bit destination register (from x8-x15).
    * `imm`: An immediate value with specific bit placement.
    * `funct3`: The 3-bit function field.
* **Returns**: The 16-bit encoded instruction.

---
### `uint32_t encode_CLtype(int op, int rdp, int imm, int rs1p, int funct3)`
Encodes a 16-bit CL-format compressed instruction (e.g., C.LW, C.FLW).

* **Parameters**:
    * `op`: The 2-bit opcode.
    * `rdp`: The 3-bit destination register (from x8-x15).
    * `imm`: An immediate value with specific bit placement for loads.
    * `rs1p`: The 3-bit base address register (from x8-x15).
    * `funct3`: The 3-bit function field.
* **Returns**: The 16-bit encoded instruction.

---
### `uint32_t encode_CLDtype(int op, int rdp, int imm, int rs1p, int funct3)`
Encodes a 16-bit CL-format compressed instruction for double-word loads (e.g., C.LD, C.FLD).

* **Parameters**:
    * `op`: The 2-bit opcode.
    * `rdp`: The 3-bit destination register (from x8-x15).
    * `imm`: An immediate value with specific bit placement.
    * `rs1p`: The 3-bit base address register (from x8-x15).
    * `funct3`: The 3-bit function field.
* **Returns**: The 16-bit encoded instruction.

---
### `uint32_t encode_CStype(int op, int rs2p, int imm, int rs1p, int funct3)`
Encodes a 16-bit CS-format compressed instruction (e.g., C.SW, C.FSW).

* **Parameters**:
    * `op`: The 2-bit opcode.
    * `rs2p`: The 3-bit source data register (from x8-x15).
    * `imm`: An immediate value with specific bit placement for stores.
    * `rs1p`: The 3-bit base address register (from x8-x15).
    * `funct3`: The 3-bit function field.
* **Returns**: The 16-bit encoded instruction.

---
### `uint32_t encode_CSDtype(int op, int rs2p, int imm, int rs1p, int funct3)`
Encodes a 16-bit CS-format compressed instruction for double-word stores (e.g., C.SD, C.FSD).

* **Parameters**:
    * `op`: The 2-bit opcode.
    * `rs2p`: The 3-bit source data register (from x8-x15).
    * `imm`: An immediate value with specific bit placement.
    * `rs1p`: The 3-bit base address register (from x8-x15).
    * `funct3`: The 3-bit function field.
* **Returns**: The 16-bit encoded instruction.

---
### `uint32_t encode_CBtype(int op, int imm, int rs1, int funct3)`
Encodes a 16-bit CB-format compressed instruction (e.g., C.BEQZ, C.BNEZ).

* **Parameters**:
    * `op`: The 2-bit opcode.
    * `imm`: A branch offset with specific bit placement.
    * `rs1`: The 3-bit source register (from x8-x15).
    * `funct3`: The 3-bit function field.
* **Returns**: The 16-bit encoded instruction.

---
### `uint32_t encode_CBItype(int op, int imm, int rd_rs1p, int funct2, int funct3)`
Encodes a 16-bit compressed instruction format, likely a variant of CB or CI. It might be for instructions like C.SRAI, C.ANDI etc.

* **Parameters**:
    * `op`: The 2-bit opcode.
    * `imm`: A 6-bit immediate value (split as imm\[5\] and imm\[4:0\]).
    * `rd_rs1p`: The 3-bit destination/source register (from x8-x15).
    * `funct2`: A 2-bit function field.
    * `funct3`: The 3-bit function field.
* **Returns**: The 16-bit encoded instruction.

---
### `uint32_t encode_CJtype(int op, int imm, int funct3)`
Encodes a 16-bit CJ-format compressed instruction (e.g., C.J, C.JAL).

* **Parameters**:
    * `op`: The 2-bit opcode.
    * `imm`: A jump offset with specific bit placement.
    * `funct3`: The 3-bit function field.
* **Returns**: The 16-bit encoded instruction.

---
## High-Level Instruction Encoders

These functions provide a more convenient way to encode specific RISC-V instructions by name. They typically call the appropriate low-level format encoder internally with fixed opcode and funct fields. Parameters usually include destination register (`rd`), source registers (`rs1`, `rs2`, `rs3`), and immediate values (`imm`) as applicable to the specific instruction. All return the `uint32_t` encoded instruction.

**RV32I and RV64I Base Instructions:**

* `encode_LUI(int rd, int imm)`
* `encode_AUIPC(int rd, int imm)`
* `encode_JAL(int rd, int imm)`
* `encode_JALR(int rd, int rs1, int imm)`
* `encode_BEQ(int imm, int rs1, int rs2)`
* `encode_BNE(int imm, int rs1, int rs2)`
* `encode_BLT(int imm, int rs1, int rs2)`
* `encode_BGE(int imm, int rs1, int rs2)`
* `encode_BLTU(int imm, int rs1, int rs2)`
* `encode_BGEU(int imm, int rs1, int rs2)`
* `encode_LB(int rd, int rs1, int imm)`
* `encode_LH(int rd, int rs1, int imm)`
* `encode_LW(int rd, int rs1, int imm)`
* `encode_LBU(int rd, int rs1, int imm)`
* `encode_LHU(int rd, int rs1, int imm)`
* `encode_SB(int imm, int rs1, int rs2)`
* `encode_SH(int imm, int rs1, int rs2)`
* `encode_SW(int imm, int rs1, int rs2)`
* `encode_ADDI(int rd, int rs1, int imm)`
* `encode_SLTI(int rd, int rs1, int imm)`
* `encode_SLTIU(int rd, int rs1, int imm)`
* `encode_XORI(int rd, int rs1, int imm)`
* `encode_ORI(int rd, int rs1, int imm)`
* `encode_ANDI(int rd, int rs1, int imm)`
* `encode_ADD(int rd, int rs1, int rs2)`
* `encode_SUB(int rd, int rs1, int rs2)`
* `encode_SLL(int rd, int rs1, int rs2)`
* `encode_SLT(int rd, int rs1, int rs2)`
* `encode_SLTU(int rd, int rs1, int rs2)`
* `encode_XOR(int rd, int rs1, int rs2)`
* `encode_SRL(int rd, int rs1, int rs2)`
* `encode_SRA(int rd, int rs1, int rs2)`
* `encode_OR(int rd, int rs1, int rs2)`
* `encode_AND(int rd, int rs1, int rs2)`
* `encode_FENCE(int rd, int rs1, int imm)` (Note: Parameters for FENCE might vary; typically `pred`, `succ` are in `imm`)
* `encode_ECALL(int rd, int rs1)` (Note: `rd` and `rs1` are typically zero for standard ECALL)
* `encode_EBREAK(int rd, int rs1)` (Note: `rd` and `rs1` are typically zero for standard EBREAK)

**RV64I Specific Instructions (from Zifencei, Zicsr, RV64M, RV64A, RV64F, RV64D):**

* `encode_LWU(int rd, int rs1, int imm)`
* `encode_LD(int rd, int rs1, int imm)`
* `encode_SD(int imm, int rs1, int rs2)`
* `encode_SLLI(int rd, int rs1, int shamt)` (Note: `rs2` parameter in header is likely `shamt`)
* `encode_SRLI(int rd, int rs1, int shamt)` (Note: `rs2` parameter in header is likely `shamt`)
* `encode_SRAI(int rd, int rs1, int shamt)` (Note: `rs2` parameter in header is likely `shamt`)
* `encode_ADDIW(int rd, int rs1, int imm)`
* `encode_SLLIW(int rd, int rs1, int shamt)` (Note: `rs2` parameter in header is likely `shamt`)
* `encode_SRLIW(int rd, int rs1, int shamt)` (Note: `rs2` parameter in header is likely `shamt`)
* `encode_SRAIW(int rd, int rs1, int shamt)` (Note: `rs2` parameter in header is likely `shamt`)
* `encode_ADDW(int rd, int rs1, int rs2)`
* `encode_SUBW(int rd, int rs1, int rs2)`
* `encode_SLLW(int rd, int rs1, int rs2)`
* `encode_SRLW(int rd, int rs1, int rs2)`
* `encode_SRAW(int rd, int rs1, int rs2)`

**"M" Standard Extension Instructions (Multiplication and Division):**

* `encode_MUL(int rd, int rs1, int rs2)`
* `encode_MULH(int rd, int rs1, int rs2)`
* `encode_MULHSU(int rd, int rs1, int rs2)`
* `encode_MULHU(int rd, int rs1, int rs2)`
* `encode_DIV(int rd, int rs1, int rs2)`
* `encode_DIVU(int rd, int rs1, int rs2)`
* `encode_REM(int rd, int rs1, int rs2)`
* `encode_REMU(int rd, int rs1, int rs2)`
* `encode_MULW(int rd, int rs1, int rs2)`
* `encode_DIVW(int rd, int rs1, int rs2)`
* `encode_DIVUW(int rd, int rs1, int rs2)`
* `encode_REMW(int rd, int rs1, int rs2)`
* `encode_REMUW(int rd, int rs1, int rs2)`

**"A" Standard Extension Instructions (Atomics):**

* `encode_LR_W(int rd, int rs1, int rs2, int rl, int aq)` (Note: `rs2` usually 0 for LR)
* `encode_SC_W(int rd, int rs1, int rs2, int rl, int aq)`
* `encode_AMOSWAP_W(int rd, int rs1, int rs2, int rl, int aq)`
* `encode_AMOADD_W(int rd, int rs1, int rs2, int rl, int aq)`
* `encode_AMOXOR_W(int rd, int rs1, int rs2, int rl, int aq)`
* `encode_AMOAND_W(int rd, int rs1, int rs2, int rl, int aq)`
* `encode_AMOOR_W(int rd, int rs1, int rs2, int rl, int aq)`
* `encode_AMOMIN_W(int rd, int rs1, int rs2, int rl, int aq)`
* `encode_AMOMAX_W(int rd, int rs1, int rs2, int rl, int aq)`
* `encode_AMOMINU_W(int rd, int rs1, int rs2, int rl, int aq)`
* `encode_AMOMAXU_W(int rd, int rs1, int rs2, int rl, int aq)`
* `encode_LR_D(int rd, int rs1, int rs2, int rl, int aq)` (Note: `rs2` usually 0 for LR)
* `encode_SC_D(int rd, int rs1, int rs2, int rl, int aq)`
* `encode_AMOSWAP_D(int rd, int rs1, int rs2, int rl, int aq)`
* `encode_AMOADD_D(int rd, int rs1, int rs2, int rl, int aq)`
* `encode_AMOXOR_D(int rd, int rs1, int rs2, int rl, int aq)`
* `encode_AMOAND_D(int rd, int rs1, int rs2, int rl, int aq)`
* `encode_AMOOR_D(int rd, int rs1, int rs2, int rl, int aq)`
* `encode_AMOMIN_D(int rd, int rs1, int rs2, int rl, int aq)`
* `encode_AMOMAX_D(int rd, int rs1, int rs2, int rl, int aq)`
* `encode_AMOMINU_D(int rd, int rs1, int rs2, int rl, int aq)`
* `encode_AMOMAXU_D(int rd, int rs1, int rs2, int rl, int aq)`

**"F" Standard Extension Instructions (Single-Precision Floating-Point):**

* `encode_FLW(int rd, int rs1, int imm)`
* `encode_FSW(int imm, int rs1, int rs2)`
* `encode_FMADD_S(int rd, int funct3, int rs1, int rs2, int rs3)` (Note: `funct3` is rounding mode `rm`)
* `encode_FMSUB_S(int rd, int funct3, int rs1, int rs2, int rs3)` (Note: `funct3` is `rm`)
* `encode_FNMSUB_S(int rd, int funct3, int rs1, int rs2, int rs3)` (Note: `funct3` is `rm`)
* `encode_FNMADD_S(int rd, int funct3, int rs1, int rs2, int rs3)` (Note: `funct3` is `rm`)
* `encode_FADD_S(int rd, int funct3, int rs1, int rs2)` (Note: `funct3` is `rm`)
* `encode_FSUB_S(int rd, int funct3, int rs1, int rs2)` (Note: `funct3` is `rm`)
* `encode_FMUL_S(int rd, int funct3, int rs1, int rs2)` (Note: `funct3` is `rm`)
* `encode_FDIV_S(int rd, int funct3, int rs1, int rs2)` (Note: `funct3` is `rm`)
* `encode_FSQRT_S(int rd, int funct3, int rs1)` (Note: `funct3` is `rm`)
* `encode_FSGNJ_S(int rd, int rs1, int rs2)`
* `encode_FSGNJN_S(int rd, int rs1, int rs2)`
* `encode_FSGNJX_S(int rd, int rs1, int rs2)`
* `encode_FMIN_S(int rd, int rs1, int rs2)`
* `encode_FMAX_S(int rd, int rs1, int rs2)`
* `encode_FCVT_W_S(int rd, int funct3, int rs1)` (Note: `funct3` is `rm`)
* `encode_FCVT_WU_S(int rd, int funct3, int rs1)` (Note: `funct3` is `rm`)
* `encode_FMV_X_S(int rd, int rs1)`
* `encode_FEQ_S(int rd, int rs1, int rs2)`
* `encode_FLT_S(int rd, int rs1, int rs2)`
* `encode_FLE_S(int rd, int rs1, int rs2)`
* `encode_FCLASS_S(int rd, int rs1)`
* `encode_FCVT_S_W(int rd, int funct3, int rs1)` (Note: `funct3` is `rm`)
* `encode_FCVT_S_WU(int rd, int funct3, int rs1)` (Note: `funct3` is `rm`)
* `encode_FMV_W_X(int rd, int rs1)`
* `encode_FCVT_L_S(int rd, int funct3, int rs1)` (RV64F specific; Note: `funct3` is `rm`)
* `encode_FCVT_LU_S(int rd, int funct3, int rs1)` (RV64F specific; Note: `funct3` is `rm`)
* `encode_FCVT_S_L(int rd, int funct3, int rs1)` (RV64F specific; Note: `funct3` is `rm`)
* `encode_FCVT_S_LU(int rd, int funct3, int rs1)` (RV64F specific; Note: `funct3` is `rm`)

**"D" Standard Extension Instructions (Double-Precision Floating-Point):**

* `encode_FLD(int rd, int rs1, int imm)`
* `encode_FSD(int imm, int rs1, int rs2)`
* `encode_FMADD_D(int rd, int funct3, int rs1, int rs2, int rs3)` (Note: `funct3` is `rm`)
* `encode_FMSUB_D(int rd, int funct3, int rs1, int rs2, int rs3)` (Note: `funct3` is `rm`)
* `encode_FNMSUB_D(int rd, int funct3, int rs1, int rs2, int rs3)` (Note: `funct3` is `rm`)
* `encode_FNMADD_D(int rd, int funct3, int rs1, int rs2, int rs3)` (Note: `funct3` is `rm`)
* `encode_FADD_D(int rd, int funct3, int rs1, int rs2)` (Note: `funct3` is `rm`)
* `encode_FSUB_D(int rd, int funct3, int rs1, int rs2)` (Note: `funct3` is `rm`)
* `encode_FMUL_D(int rd, int funct3, int rs1, int rs2)` (Note: `funct3` is `rm`)
* `encode_FDIV_D(int rd, int funct3, int rs1, int rs2)` (Note: `funct3` is `rm`)
* `encode_FSQRT_D(int rd, int funct3, int rs1)` (Note: `funct3` is `rm`)
* `encode_FSGNJ_D(int rd, int rs1, int rs2)`
* `encode_FSGNJN_D(int rd, int rs1, int rs2)`
* `encode_FSGNJX_D(int rd, int rs1, int rs2)`
* `encode_FMIN_D(int rd, int rs1, int rs2)`
* `encode_FMAX_D(int rd, int rs1, int rs2)`
* `encode_FCVT_S_D(int rd, int funct3, int rs1)` (Note: `funct3` is `rm`)
* `encode_FCVT_D_S(int rd, int funct3, int rs1)` (Note: `funct3` is `rm`)
* `encode_FEQ_D(int rd, int rs1, int rs2)`
* `encode_FLT_D(int rd, int rs1, int rs2)`
* `encode_FLE_D(int rd, int rs1, int rs2)`
* `encode_FCLASS_D(int rd, int rs1)`
* `encode_FCVT_W_D(int rd, int funct3, int rs1)` (Note: `funct3` is `rm`)
* `encode_FCVT_WU_D(int rd, int funct3, int rs1)` (Note: `funct3` is `rm`)
* `encode_FCVT_D_W(int rd, int funct3, int rs1)` (Note: `funct3` is `rm`)
* `encode_FCVT_D_WU(int rd, int funct3, int rs1)` (Note: `funct3` is `rm`)
* `encode_FCVT_L_D(int rd, int funct3, int rs1)` (RV64D specific; Note: `funct3` is `rm`)
* `encode_FCVT_LU_D(int rd, int funct3, int rs1)` (RV64D specific; Note: `funct3` is `rm`)
* `encode_FMV_X_D(int rd, int rs1)` (RV64D specific)
* `encode_FCVT_D_L(int rd, int funct3, int rs1)` (RV64D specific; Note: `funct3` is `rm`)
* `encode_FCVT_D_LU(int rd, int funct3, int rs1)` (RV64D specific; Note: `funct3` is `rm`)
* `encode_FMV_D_X(int rd, int rs1)` (RV64D specific)

**"C" Standard Extension Instructions (Compressed):**

* `encode_C_ADDI4SPN(int rdp, int imm)`
* `encode_C_FLD(int rdp, int imm, int rs1p)`
* `encode_C_LW(int rdp, int imm, int rs1p)`
* `encode_C_LD(int rdp, int imm, int rs1p)` (RV64C/RV32C if FLEN >= 64)
* `encode_C_FSD(int rs2p, int imm, int rs1p)`
* `encode_C_SW(int rs2p, int imm, int rs1p)`
* `encode_C_SD(int rs2p, int imm, int rs1p)` (RV64C/RV32C if FLEN >= 64)
* `encode_C_NOP()`
* `encode_C_ADDI(int imm, int rd_rs1)`
* `encode_C_ADDIW(int imm, int rd_rs1)` (RV64C/RV128C)
* `encode_C_LI(int imm, int rd_rs1)`
* `encode_C_ADDI16SP(int imm, int rd_rs1)` (Note: `rd_rs1` is implicitly SP/x2)
* `encode_C_LUI(int imm, int rd_rs1)` (Note: `rd_rs1` != x0, x2)
* `encode_C_J(int imm)`
* `encode_C_BEQZ(int imm, int rs1)`
* `encode_C_BNEZ(int imm, int rs1)`
* `encode_C_SLLI(int imm, int rd_rs1)`
* `encode_C_FLDSP(int imm, int rd_rs1)`
* `encode_C_LWSP(int imm, int rd_rs1)`
* `encode_C_LDSP(int imm, int rd_rs1)` (RV64C/RV128C)
* `encode_C_JR(int rd_rs1)`
* `encode_C_MV(int rs2, int rd_rs1)`
* `encode_C_EBREAK()`
* `encode_C_JALR(int rd_rs1)`
* `encode_C_ADD(int rs2, int rd_rs1)`
* `encode_C_FSDSP(int rs2, int imm)`
* `encode_C_SWSP(int rs2, int imm)`
* `encode_C_SDSP(int rs2, int imm)` (RV64C/RV128C)
* `encode_C_SUB(int rs2p, int rd_rs1p)`
* `encode_C_XOR(int rs2p, int rd_rs1p)`
* `encode_C_OR(int rs2p, int rd_rs1p)`
* `encode_C_AND(int rs2p, int rd_rs1p)`
* `encode_C_SUBW(int rs2p, int rd_rs1p)` (RV64C/RV128C)
* `encode_C_ADDW(int rs2p, int rd_rs1p)` (RV64C/RV128C)

---
*Note: For floating-point instructions, `funct3` often represents the rounding mode (`rm`). For atomic instructions, `rl` and `aq` are release and acquire bits. For shift immediate instructions (`SLLI`, `SRLI`, `SRAI`, `SLLIW`, `SRLIW`, `SRAIW`), the shift amount (`shamt`) is typically passed in what would be the `rs2` field for R-type or part of `imm` for I-type, depending on the exact encoding variant used by the `encode_Rtype` or `encode_Itype` functions (the specific implementation in the header shows `rs2` for these shift instructions, which is how some assemblers handle it, but the immediate shifts are I-type). Parameters like `rdp`, `rs1p`, `rs2p` in compressed instructions refer to the restricted register set x8-x15.*

<br/><br/>