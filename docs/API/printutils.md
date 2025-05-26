# Print Utils

This section provides an API reference for the utility functions defined in R-Visor. These functions are designed to print decoded RISC-V instructions in a human-readable format, likely to the R-Visor logger or standard output. They take an `RvInst` object (presumably defined in `instructions.h`) as input, which contains the decoded fields of an instruction.

---
## Generic Instruction Printing Function

### `void printInstruction(RvInst inst)`
Prints a generically decoded RISC-V instruction.

* **Parameters**:
    * `inst`: An `RvInst` object containing the already decoded instruction.
* **Description**: This function likely acts as a dispatcher, calling a more specific print function (e.g., `printRType`, `printIType`) based on the instruction's opcode or determined format stored within the `RvInst` object. It aims to provide a comprehensive, human-readable representation of the instruction, potentially including its mnemonic and operands.

---
## Standard RISC-V Format-Specific Printing Functions

These functions print instructions belonging to the standard RISC-V base and extension formats. The header shows duplicate declarations for `printRType` through `printJType`; the documentation below assumes each is a single function intended to print the details of an instruction of that type, which may include its fields in hexadecimal or a disassembled format.

### `void printRType(RvInst r_type)`
Prints a decoded R-type instruction.

* **Parameters**:
    * `r_type`: An `RvInst` object, decoded as an R-type instruction, containing fields like opcode, rd, funct3, rs1, rs2, and funct7.
* **Description**: Displays the components or the assembly representation of an R-type instruction.

---
### `void printIType(RvInst i_type)`
Prints a decoded I-type instruction.

* **Parameters**:
    * `i_type`: An `RvInst` object, decoded as an I-type instruction, containing fields like opcode, rd, funct3, rs1, and imm.
* **Description**: Displays the components or the assembly representation of an I-type instruction.

---
### `void printSType(RvInst s_type)`
Prints a decoded S-type instruction.

* **Parameters**:
    * `s_type`: An `RvInst` object, decoded as an S-type instruction, containing fields like opcode, imm, funct3, rs1, and rs2.
* **Description**: Displays the components or the assembly representation of an S-type instruction.

---
### `void printUType(RvInst u_type)`
Prints a decoded U-type instruction.

* **Parameters**:
    * `u_type`: An `RvInst` object, decoded as a U-type instruction, containing fields like opcode, rd, and imm.
* **Description**: Displays the components or the assembly representation of a U-type instruction.

---
### `void printBType(RvInst b_type)`
Prints a decoded B-type instruction.

* **Parameters**:
    * `b_type`: An `RvInst` object, decoded as a B-type instruction, containing fields like opcode, imm, funct3, rs1, and rs2.
* **Description**: Displays the components or the assembly representation of a B-type instruction.

---
### `void printJType(RvInst j_type)`
Prints a decoded J-type instruction.

* **Parameters**:
    * `j_type`: An `RvInst` object, decoded as a J-type instruction, containing fields like opcode, rd, and imm.
* **Description**: Displays the components or the assembly representation of a J-type instruction.

---
### `void printR4Type(RvInst r4_type)`
Prints a decoded R4-type instruction (typically for instructions with three source registers like FMA).

* **Parameters**:
    * `r4_type`: An `RvInst` object, decoded as an R4-type instruction.
* **Description**: Displays the components or the assembly representation of an R4-type instruction.

---
## Compressed RISC-V Format-Specific Printing Functions

These functions print decoded 16-bit compressed RISC-V instructions.

### `void printCRType(RvInst cr_type)`
Prints a decoded CR-format compressed instruction.

* **Parameters**:
    * `cr_type`: An `RvInst` object representing a CR-format instruction.

---
### `void printCIType(RvInst ci_type)`
Prints a decoded CI-format compressed instruction.

* **Parameters**:
    * `ci_type`: An `RvInst` object representing a CI-format instruction.

---
### `void printCSSType(RvInst css_type)`
Prints a decoded CSS-format compressed instruction.

* **Parameters**:
    * `css_type`: An `RvInst` object representing a CSS-format instruction.

---
### `void printCIWType(RvInst ciw_type)`
Prints a decoded CIW-format compressed instruction.

* **Parameters**:
    * `ciw_type`: An `RvInst` object representing a CIW-format instruction.

---
### `void printCLType(RvInst cl_type)`
Prints a decoded CL-format compressed instruction.

* **Parameters**:
    * `cl_type`: An `RvInst` object representing a CL-format instruction.

---
### `void printCSType(RvInst cs_type)`
Prints a decoded CS-format compressed instruction.

* **Parameters**:
    * `cs_type`: An `RvInst` object representing a CS-format instruction.

---
### `void printCBType(RvInst cb_type)`
Prints a decoded CB-format compressed instruction.

* **Parameters**:
    * `cb_type`: An `RvInst` object representing a CB-format instruction.

---
### `void printCJType(RvInst cj_type)`
Prints a decoded CJ-format compressed instruction.

* **Parameters**:
    * `cj_type`: An `RvInst` object representing a CJ-format instruction.

---
### `void printCRIType(RvInst cl_type)`
Prints a decoded compressed instruction, likely a specific variant of CR or CI format.

* **Parameters**:
    * `cl_type`: An `RvInst` object representing the instruction. *(Note: The parameter name `cl_type` might be a slight misnomer if it's intended for a general "CRI" format, but it matches the header.)*

<br/><br/>