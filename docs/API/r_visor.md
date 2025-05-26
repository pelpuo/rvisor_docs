# Instrumentation API

This document provides an API reference for the R-Visor C interface defined in `rail.h`.

---
## Enums

### `Rvisor_Ipoint`
Defines the instrumentation point relative to an event (e.g., basic block execution, instruction execution).
```c
typedef enum{
    PRE,  // Before the event
    POST  // After the event
} Rvisor_Ipoint;
```
* **`PRE`**: The instrumentation or inlined code executes *before* the associated basic block or instruction.
* **`POST`**: The instrumentation or inlined code executes *after* the associated basic block or instruction.

---
### `Rvisor_Invoke`
Defines when an instrumentation routine should be invoked.
```c
typedef enum{
    ALLOCATOR, // During basic block allocation (once)
    RUNTIME    // Every time the event occurs
} Rvisor_Invoke;
```
* **`ALLOCATOR`**: The routine is called once when the basic block or instruction is first processed and allocated into the code cache. Useful for static analysis or one-time modifications.
* **`RUNTIME`**: The routine is called every time the associated basic block or instruction is executed.

---
## Typedefs (Routine Signatures)

R-Visor uses function pointers to define the signatures for different types of instrumentation routines.

### `Rvisor_Rt_Routine_Inst`
Signature for a **runtime** routine at the **instruction** level.
```c
typedef void (*Rvisor_Rt_Routine_Inst)(RvInst inst, uint64_t *regs);
```
* `inst`: An `RvInst` object representing the current instruction.
* `regs`: A pointer to the register file, allowing observation or modification of register states.

---
### `Rvisor_Alloc_Routine_Inst`
Signature for an **allocator** routine at the **instruction** level.
```c
typedef int (*Rvisor_Alloc_Routine_Inst)(RvInst inst);
```
* `inst`: An `RvInst` object representing the current instruction.
* Return: An integer, potentially to signal success/failure or modify R-Visor's behavior.

---
### `Rvisor_Rt_Routine_BB`
Signature for a **runtime** routine at the **basic block** level.
```c
typedef void (*Rvisor_Rt_Routine_BB)(rvisor_basic_block bb, uint64_t *regs);
```
* `bb`: An `rvisor_basic_block` object representing the current basic block.
* `regs`: A pointer to the register file.

---
### `Rvisor_Alloc_Routine_BB`
Signature for an **allocator** routine at the **basic block** level.
```c
typedef void (*Rvisor_Alloc_Routine_BB)();
```
* This routine takes no arguments.

---
### `Rvisor_Exit_Routine`
Signature for a routine to be called when the instrumented program exits.
```c
typedef void (*Rvisor_Exit_Routine)(uint64_t *regs);
```
* `regs`: A pointer to the final state of the register file.

---
## Core API Functions

### Initialization and Execution

#### `void rvisor_init(char *target)`
Initializes the R-Visor framework.

* **Parameters**:
    * `target`: A string representing the path to the target binary to be instrumented.
* **Description**: This function must be called before most other R-Visor API functions. It sets up internal structures, parses the target ELF binary, and prepares for instrumentation and execution.

---
#### `void rvisor_register_args(int argc, char **argv, char **envp)`
Registers the command-line arguments and environment variables for the target program.

* **Parameters**:
    * `argc`: The argument count for the target program.
    * `argv`: The argument vector for the target program.
    * `envp`: The environment variables for the target program.
* **Description**: This allows the instrumented program to receive its intended command-line arguments and environment.

---
#### `void rvisor_run()`
Starts the execution of the instrumented target binary under R-Visor's control.

* **Description**: R-Visor begins JIT compilation, execution, and invocation of registered instrumentation routines. This function typically does not return until the target program terminates.

---
### Routine Registration

#### `void rvisor_register_exit_routine(void (*func)())`
Registers a single routine to be called when the target program finishes execution.

* **Parameters**:
    * `func`: A function pointer matching the `Rvisor_Exit_Routine` signature (`void (*func)(uint64_t *regs)`).
* **Description**: The registered function is invoked just before R-Visor itself terminates, allowing for cleanup or final analysis output.

---
#### `void rvisor_register_bb_routine(void (*func)(), Rvisor_Ipoint ipoint, Rvisor_Invoke invoke)`
Registers a single routine to be called for basic blocks. Only one such routine can be active.

* **Parameters**:
    * `func`: A function pointer. Its signature should match `Rvisor_Alloc_Routine_BB` if `invoke` is `ALLOCATOR`, or `Rvisor_Rt_Routine_BB` if `invoke` is `RUNTIME`.
    * `ipoint`: An `Rvisor_Ipoint` enum value (`PRE` or `POST`) specifying whether the routine runs before or after the basic block.
    * `invoke`: An `Rvisor_Invoke` enum value (`ALLOCATOR` or `RUNTIME`) specifying when the routine is called.
* **Description**: Allows instrumentation at the granularity of basic blocks.

---
#### `void rvisor_register_inst_routine(void (*func)(), Rvisor_Ipoint ipoint, Rvisor_Invoke invoke)`
Registers a single routine to be called for every instruction. Only one such routine can be active.

* **Parameters**:
    * `func`: A function pointer. Its signature should match `Rvisor_Alloc_Routine_Inst` if `invoke` is `ALLOCATOR`, or `Rvisor_Rt_Routine_Inst` if `invoke` is `RUNTIME`.
    * `ipoint`: An `Rvisor_Ipoint` enum value (`PRE` or `POST`).
    * `invoke`: An `Rvisor_Invoke` enum value (`ALLOCATOR` or `RUNTIME`).
* **Description**: Enables fine-grained instrumentation at the individual instruction level.

---
#### `void rvisor_register_inst_type_routine(InstType type, void (*func)(), Rvisor_Ipoint ipoint, Rvisor_Invoke invoke)`
Registers a routine to be called for instructions of a specific type. One routine can be registered per instruction type.

* **Parameters**:
    * `type`: An `InstType` value (defined by ARCH-VISOR specifications) representing the instruction type to target.
    * `func`: A function pointer, typically `Rvisor_Rt_Routine_Inst` for runtime invocation or `Rvisor_Alloc_Routine_Inst` for allocator invocation.
    * `ipoint`: An `Rvisor_Ipoint` enum value.
    * `invoke`: An `Rvisor_Invoke` enum value.
* **Description**: Allows targeted instrumentation based on instruction formats or classes (e.g., all R-type instructions). The routines are managed internally using a hash map (`rvisor_inst_type_routine_map`).

---
#### `void rvisor_register_inst_group_routine(RvInstGroup group, void (*func)(), Rvisor_Ipoint ipoint, Rvisor_Invoke invoke)`
Registers a routine to be called for instructions belonging to a specific group (defined via ARCH-VISOR). One routine can be registered per group.

* **Parameters**:
    * `group`: An `RvInstGroup` value (an extensible construct specified by ARCH-VISOR sources) representing the instruction group.
    * `func`: A function pointer, typically `Rvisor_Alloc_Routine_Inst` for allocator invocation or `Rvisor_Rt_Routine_Inst` for runtime invocation.
    * `ipoint`: An `Rvisor_Ipoint` enum value.
    * `invoke`: An `Rvisor_Invoke` enum value.
* **Description**: Provides a flexible way to instrument custom-defined sets of instructions. Routines are managed internally using `rvisor_inst_group_routine_map`.

---
### Inline Instruction API

These functions allow for injecting individual RISC-V instructions directly into the execution flow at basic block or instruction granularity. The injected instructions are added to internal vectors (`rvisor_inline_inst_bb_vec_pre`, `rvisor_inline_inst_bb_vec_post`, etc.) and then woven into the code cache.

#### `void rvisor_add_inline_inst_bb(uint32_t instruction, Rvisor_Ipoint ipoint)`
Adds a raw 32-bit encoded RISC-V instruction to be inlined at basic block boundaries.

* **Parameters**:
    * `instruction`: The 32-bit encoded RISC-V instruction.
    * `ipoint`: An `Rvisor_Ipoint` enum value (`PRE` or `POST`), determining if the instruction is placed before or after the original basic block's instructions in the code cache.
* **Description**: Useful for inserting short assembly sequences for high-performance instrumentation.

---
#### `void rvisor_add_inline_inst_inst(uint32_t instruction, Rvisor_Ipoint ipoint)`
Adds a raw 32-bit encoded RISC-V instruction to be inlined at individual instruction boundaries.

* **Parameters**:
    * `instruction`: The 32-bit encoded RISC-V instruction.
    * `ipoint`: An `Rvisor_Ipoint` enum value (`PRE` or `POST`).

---
#### `void rvisor_add_inline_li_bb(uint64_t value, GPregs reg, Rvisor_Ipoint ipoint)`
Helper function to add instructions (typically `LUI` and `ADDI` or appropriate sequence for RV64) to load a 64-bit immediate value into a general-purpose register, inlined at basic block boundaries.

* **Parameters**:
    * `value`: The 64-bit immediate value to load.
    * `reg`: The target general-purpose register (`GPregs` enum) to load the value into.
    * `ipoint`: An `Rvisor_Ipoint` enum value (`PRE` or `POST`).
* **Description**: Simplifies loading constants or addresses for use by other inlined instructions.

---
#### `void rvisor_add_inline_li_inst(uint64_t value, GPregs reg, Rvisor_Ipoint ipoint)`
Helper function to add instructions to load a 64-bit immediate value into a general-purpose register, inlined at individual instruction boundaries.

* **Parameters**:
    * `value`: The 64-bit immediate value to load.
    * `reg`: The target general-purpose register (`GPregs` enum).
    * `ipoint`: An `Rvisor_Ipoint` enum value (`PRE` or `POST`).

---
## Internal Hash Map Management (Primarily for R-Visor Internals)

The header also declares functions for managing hash maps used by `rvisor_register_inst_type_routine` and `rvisor_register_inst_group_routine`. These are generally not called directly by the user if the registration functions are used.

* `void add_inst_type_routine(InstType key, Rvisor_Rt_Routine_Inst routine);`
* `Rvisor_Rt_Routine_Inst get_inst_type_routine(InstType key);`
* `void add_inst_group_routine(RvInstGroup key, Rvisor_Alloc_Routine_Inst routine);`
* `Rvisor_Alloc_Routine_Inst get_inst_group_routine(RvInstGroup key);`

---
## Extern Variables (Primarily for R-Visor Internals)

The header declares several `extern` variables. These are primarily for R-Visor's internal bookkeeping and are manipulated by the API functions. Direct modification by user code is generally not recommended or necessary.

* Pointers to registered routines (e.g., `rvisor_bb_routine_post_alloc`, `rvisor_inst_routine_pre_rt`, `rvisor_exit_routine`).
* Vectors for storing inlined instructions (e.g., `rvisor_inline_inst_bb_vec_pre`, `rvisor_inline_inst_inst_vec_post`).
* Pointers to hash maps for instruction type and group routines (e.g., `rvisor_inst_type_routine_map`, `rvisor_inst_group_routine_map`).
* Counters for registered routines (`inst_group_routine_count`, `inst_type_routine_count`).

---

<br/><br/>