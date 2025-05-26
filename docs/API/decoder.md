# Decoder 
This section provides an API reference for the RISC-V instruction decoder functions within R-Visor. These functions are responsible for parsing a raw 32-bit or 16-bit instruction encoding and extracting its constituent fields into an `RvInst` structure. These functions are generated directly from the ArchVisor sources at compile time.

---
## Decoding Functions

All `decode_<Format>type` functions, as well as `decode_instruction32` and `decode_instruction16`, take a single 32-bit unsigned integer representing the instruction bits and return an `RvInst` structure. The `RvInst` structure (presumably defined in `instructions.h`) contains the decoded fields like opcode, funct3, funct7, registers (rd, rs1, rs2), and immediate values.

The primary dispatching function `decode_instruction` additionally takes a debug flag.

---
### Standard RISC-V Format Decoders

These functions decode instructions belonging to the standard RISC-V instruction formats (RV32I, RV64I, and common extensions).

#### `RvInst decode_Rtype(uint32_t instruction)`
Decodes a 32-bit R-type instruction.

* **Parameters**:
    * `instruction`: The 32-bit raw instruction encoding.
* **Returns**: An `RvInst` structure populated with the decoded fields (opcode, rd, funct3, rs1, rs2, funct7).

---
#### `RvInst decode_R4type(uint32_t instruction)`
Decodes a 32-bit R4-type instruction (typically used for FMADD, FMSUB, etc., involving three source registers rs1, rs2, rs3).

* **Parameters**:
    * `instruction`: The 32-bit raw instruction encoding.
* **Returns**: An `RvInst` structure populated with decoded fields.

---
#### `RvInst decode_RAMOtype(uint32_t instruction)`
Decodes a 32-bit R-type instruction specifically for Atomic Memory Operations (AMO). While structurally similar to R-type, this distinguishes them for clarity or specific field interpretation (like `aq` and `rl` bits).

* **Parameters**:
    * `instruction`: The 32-bit raw instruction encoding.
* **Returns**: An `RvInst` structure.

---
#### `RvInst decode_Itype(uint32_t instruction)`
Decodes a 32-bit I-type instruction (e.g., ADDI, LW, JALR).

* **Parameters**:
    * `instruction`: The 32-bit raw instruction encoding.
* **Returns**: An `RvInst` structure (opcode, rd, funct3, rs1, imm).

---
#### `RvInst decode_Stype(uint32_t instruction)`
Decodes a 32-bit S-type instruction (e.g., SW, SB).

* **Parameters**:
    * `instruction`: The 32-bit raw instruction encoding.
* **Returns**: An `RvInst` structure (opcode, imm\[4:0\], funct3, rs1, rs2, imm\[11:5\]).

---
#### `RvInst decode_Utype(uint32_t instruction)`
Decodes a 32-bit U-type instruction (e.g., LUI, AUIPC).

* **Parameters**:
    * `instruction`: The 32-bit raw instruction encoding.
* **Returns**: An `RvInst` structure (opcode, rd, imm\[31:12\]).

---
#### `RvInst decode_Btype(uint32_t instruction)`
Decodes a 32-bit B-type instruction (e.g., BEQ, BNE).

* **Parameters**:
    * `instruction`: The 32-bit raw instruction encoding.
* **Returns**: An `RvInst` structure (opcode, imm\[11\], imm\[4:1\], funct3, rs1, rs2, imm\[10:5\], imm\[12\]).

---
#### `RvInst decode_Jtype(uint32_t instruction)`
Decodes a 32-bit J-type instruction (e.g., JAL).

* **Parameters**:
    * `instruction`: The 32-bit raw instruction encoding.
* **Returns**: An `RvInst` structure (opcode, rd, imm\[19:12\], imm\[11\], imm\[10:1\], imm\[20\]).

---
### Compressed Instruction Format Decoders (RISC-V 'C' Extension)

These functions decode 16-bit compressed instructions. Note that they still take a `uint32_t` as input, but will typically only process the lower 16 bits.

#### `RvInst decode_CRtype(uint32_t instruction)`
Decodes a 16-bit CR-format compressed instruction (e.g., C.MV, C.ADD).

* **Parameters**:
    * `instruction`: The 16-bit raw instruction encoding (passed as uint32_t).
* **Returns**: An `RvInst` structure.

---
#### `RvInst decode_CRItype(uint32_t instruction)`
This seems to be a typo and might intend to be `decode_CItype` for instructions like C.ADDI or specific immediate versions of CR format. Assuming it's distinct from `decode_CItype`, it decodes a specific 16-bit compressed format involving an immediate.

* **Parameters**:
    * `instruction`: The 16-bit raw instruction encoding.
* **Returns**: An `RvInst` structure.

---
#### `RvInst decode_CItype(uint32_t instruction)`
Decodes a 16-bit CI-format compressed instruction (e.g., C.LI, C.LUI, C.ADDI).

* **Parameters**:
    * `instruction`: The 16-bit raw instruction encoding.
* **Returns**: An `RvInst` structure.

---
#### `RvInst decode_CIStype(uint32_t instruction)`
Decodes a 16-bit CI-format compressed instruction specifically for stack-pointer based loads/stores if `rd`/`rs1` is SP (e.g. C.LWSP, C.SWSP). The naming suggests it might be a more specific CI variant or a typo for CSS-type. Given `decode_CSStype` exists, this is likely a specific CI sub-format.

* **Parameters**:
    * `instruction`: The 16-bit raw instruction encoding.
* **Returns**: An `RvInst` structure.

---
#### `RvInst decode_CISDtype(uint32_t instruction)`
Likely decodes a 16-bit CI-format compressed instruction for stack-pointer based double-word loads (e.g., C.LDSP) from RV64C/RV128C.

* **Parameters**:
    * `instruction`: The 16-bit raw instruction encoding.
* **Returns**: An `RvInst` structure.

---
#### `RvInst decode_CI16type(uint32_t instruction)`
Decodes a 16-bit CI-format compressed instruction like C.ADDI16SP.

* **Parameters**:
    * `instruction`: The 16-bit raw instruction encoding.
* **Returns**: An `RvInst` structure.

---
#### `RvInst decode_CSStype(uint32_t instruction)`
Decodes a 16-bit CSS-format compressed instruction (e.g., C.SWSP).

* **Parameters**:
    * `instruction`: The 16-bit raw instruction encoding.
* **Returns**: An `RvInst` structure.

---
#### `RvInst decode_CSSDtype(uint32_t instruction)`
Decodes a 16-bit CSS-format compressed instruction for stack-pointer based double-word stores (e.g., C.SDSP) from RV64C/RV128C.

* **Parameters**:
    * `instruction`: The 16-bit raw instruction encoding.
* **Returns**: An `RvInst` structure.

---
#### `RvInst decode_CIWtype(uint32_t instruction)`
Decodes a 16-bit CIW-format compressed instruction (e.g., C.ADDIW).

* **Parameters**:
    * `instruction`: The 16-bit raw instruction encoding.
* **Returns**: An `RvInst` structure.

---
#### `RvInst decode_CLtype(uint32_t instruction)`
Decodes a 16-bit CL-format compressed instruction (e.g., C.LW, C.FLW).

* **Parameters**:
    * `instruction`: The 16-bit raw instruction encoding.
* **Returns**: An `RvInst` structure.

---
#### `RvInst decode_CLDtype(uint32_t instruction)`
Decodes a 16-bit CL-format compressed instruction for double-word loads (e.g., C.LD, C.FLD) from RV64C/RV128C.

* **Parameters**:
    * `instruction`: The 16-bit raw instruction encoding.
* **Returns**: An `RvInst` structure.

---
#### `RvInst decode_CStype(uint32_t instruction)`
Decodes a 16-bit CS-format compressed instruction (e.g., C.SW, C.FSW).

* **Parameters**:
    * `instruction`: The 16-bit raw instruction encoding.
* **Returns**: An `RvInst` structure.

---
#### `RvInst decode_CSDtype(uint32_t instruction)`
Decodes a 16-bit CS-format compressed instruction for double-word stores (e.g., C.SD, C.FSD) from RV64C/RV128C.

* **Parameters**:
    * `instruction`: The 16-bit raw instruction encoding.
* **Returns**: An `RvInst` structure.

---
#### `RvInst decode_CBtype(uint32_t instruction)`
Decodes a 16-bit CB-format compressed instruction (e.g., C.BEQZ, C.BNEZ, C.ADDIW).

* **Parameters**:
    * `instruction`: The 16-bit raw instruction encoding.
* **Returns**: An `RvInst` structure.

---
#### `RvInst decode_CBItype(uint32_t instruction)`
This seems to be a typo or a specific sub-format of CB. It likely decodes a 16-bit compressed branch format that also incorporates an immediate field not covered by the general CB, or specific CB instructions like C.SLLI.

* **Parameters**:
    * `instruction`: The 16-bit raw instruction encoding.
* **Returns**: An `RvInst` structure.

---
#### `RvInst decode_CJtype(uint32_t instruction)`
Decodes a 16-bit CJ-format compressed instruction (e.g., C.J, C.JAL).

* **Parameters**:
    * `instruction`: The 16-bit raw instruction encoding.
* **Returns**: An `RvInst` structure.

---
### General Instruction Decoders

These functions handle the initial dispatch or decode based on instruction size.

#### `RvInst decode_instruction32(uint32_t instruction)`
Decodes a 32-bit instruction by determining its specific format (R, I, S, U, B, J) and calling the appropriate format-specific decoder.

* **Parameters**:
    * `instruction`: The 32-bit raw instruction encoding.
* **Returns**: An `RvInst` structure.

---
#### `RvInst decode_instruction16(uint32_t instruction)`
Decodes a 16-bit compressed instruction by determining its specific format (CR, CI, CS, etc.) and calling the appropriate compressed format-specific decoder. Processes the lower 16 bits of the input.

* **Parameters**:
    * `instruction`: The 16-bit raw instruction encoding (passed as uint32_t).
* **Returns**: An `RvInst` structure.

---
#### `RvInst decode_instruction(uint32_t instruction, int debug)`
The main dispatch function for decoding an instruction. It first determines if the instruction is 16-bit (compressed) or 32-bit based on its lowest two bits and then calls either `decode_instruction16` or `decode_instruction32`.

* **Parameters**:
    * `instruction`: The raw instruction encoding (can be 16-bit or 32-bit, passed as uint32_t).
    * `debug`: An integer flag; if non-zero, it may enable debug printing (using `printUtils.h`) during the decoding process.
* **Returns**: An `RvInst` structure containing the decoded fields.

<br/><br/>