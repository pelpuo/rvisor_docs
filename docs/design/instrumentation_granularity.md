# Instrumentation Granularity
Instrumentation routines in R-Visor are executed periodically at a frequency determined by the user. The frequency or granularity of instrumentation routines can be one of the following:

* Module  
* Basic Block 
* Instruction 
* Instruction Type
* Instruction Group

## Module Granularity
The **module level** routines execute once per program. A typical use case for this is an exit routine which only executes at the end of a program to print out collected data or perform some other function. 

### API Usage

#### rvisor_register_exit_routine()

* **Prototype**

```c
void rvisor_register_exit_routine(void (*func)())
```
Registers a single routine to be called when the target program finishes execution.

* **Parameters**:
    * `func`: A function pointer matching the `Rvisor_Exit_Routine` signature (`void (*func)(uint64_t *regs)`).
* **Description**: The registered function is invoked just before R-Visor itself terminates, allowing for cleanup or final analysis output.

----

## Basic Block Granularity
The **basic block level** routines execute whenever there is a control flow change within the instrumented binary. Since, by default, R-Visor interrupts the program execution when control flow instructions are encountered, there are no additional context switches required to execute **basic block level** routines. When instrumenting at this granularity, users have access to basic block metadata collected by R-Visor. The metadata is of the following form:

```c
typedef enum {
    BRANCH,             // Basic block ends with a conditional branch instruction
    DIRECT_JUMP,        // Basic block ends with a direct jump (e.g., JAL with known offset)
    INDIRECT_JUMP,      // Basic block ends with an indirect jump (e.g., JALR to a register value)
    SEGMENTED           // Basic block was segmented due to reasons other than a control flow instruction (e.g., max size, instrumentation point)
} rvisor_bb_type;

typedef struct {
    uint64_t first_addr;              // First address of the block in the original binary
    uint64_t last_addr;               // Last address of the block in the original binary
    uint64_t start_location_in_cache; // Starting rvisor_memory_index where this block (and its instrumentation) begins in R-Visor's code cache
    uint64_t end_location_in_cache;   // Ending rvisor_memory_index where this block (and its instrumentation) ends in R-Visor's code cache
    int num_instructions;             // Number of original instructions in this basic block
    int taken_block;                  // Flag indicating if control switched to this block as a result of a taken branch from a previous block
    rvisor_bb_type type;              // The type of basic block (e.g., BRANCH, DIRECT_JUMP), defined by the rvisor_bb_type enum
    int resume;                       // Flag or value related to resuming execution, possibly after an event like a syscall
    // bool taken = true;              // Commented out; likely intended for branch prediction or actual branch outcome
    uint32_t start_inst;              // Raw byte encoding of the first instruction of the basic block
    uint32_t terminal_inst;           // Raw byte encoding of the last (terminating) instruction of the basic block
    uint64_t basic_block_address;     // If the block is 'SEGMENTED', this stores the original starting address of the first instruction of the logical basic block it belongs to
    uint64_t taken_addr;              // Target address if the terminating branch/jump is taken (if applicable)
    uint64_t fall_through_addr;       // Address of the next sequential instruction if the terminating branch is not taken or if it's not a branch (if applicable)
    uint64_t ecall_next;              // Address to resume after a syscall, if this block ends in a syscall
    // unordered_map<int, uint64_t> bbExits; // Commented out; likely intended for storing multiple exit points or targets
} rvisor_basic_block;
```

### API Usage

#### rvisor_register_bb_routine()

* **Prototype**

```c
void rvisor_register_bb_routine(void (*func)(), Rvisor_Ipoint ipoint, Rvisor_Invoke invoke)
```

Registers a single routine to be called for basic blocks. Only one such routine can be active.

* **Parameters**:
    * `func`: A function pointer. Its signature should match `Rvisor_Alloc_Routine_BB` if `invoke` is `ALLOCATOR`, or `Rvisor_Rt_Routine_BB` if `invoke` is `RUNTIME`.
    * `ipoint`: An `Rvisor_Ipoint` enum value (`PRE` or `POST`) specifying whether the routine runs before or after the basic block.
    * `invoke`: An `Rvisor_Invoke` enum value (`ALLOCATOR` or `RUNTIME`) specifying when the routine is called.
* **Description**: Allows instrumentation at the granularity of basic blocks.



<!-- Example routines implemented at this granularity are [BBProfile](./../example_tools/bb_profile) and [BBFrequency](./../example_tools/bb_frequency) -->

----

## Instruction Level Granularity
Routines can be placed at **instruction level** granularity when a user requires statistics for each individual instruction executed in the target binary. By default, a context switch would need to be invoked after every instruction for this instrumentation granularity, unless [inline routines](./../optimizations/routine_inlining) are used. As such, this is the most computationally expensive type of routine. At this granularity, users have access to the decoded instruction as a struct created by R-Visor. This has the following form:

```c++
typedef struct {
    int opcode;
    int rd;
    int funct3;
    int rs1;
    int rs2;
    int funct7;
    int imm;
    int address;

    InstType type;
    InstName name;
} RvInst;
```

### API Usage

#### rvisor_register_inst_routine()

* **Prototype** 

 ```c
 void rvisor_register_inst_routine(void (*func)(), Rvisor_Ipoint ipoint, Rvisor_Invoke invoke)
 ```

Registers a single routine to be called for every instruction. Only one such routine can be active.

* **Parameters**:
    * `func`: A function pointer. Its signature should match `Rvisor_Alloc_Routine_Inst` if `invoke` is `ALLOCATOR`, or `Rvisor_Rt_Routine_Inst` if `invoke` is `RUNTIME`.
    * `ipoint`: An `Rvisor_Ipoint` enum value (`PRE` or `POST`).
    * `invoke`: An `Rvisor_Invoke` enum value (`ALLOCATOR` or `RUNTIME`).
* **Description**: Enables fine-grained instrumentation at the individual instruction level.


----

## Instruction Type Granularity
The RISC-V instruction set defines 6 distinct instruction types (for uncompressed instructions) namely:

* R Type
* I Type
* S Type
* U Type
* B Type
* J Type

Based on these, a modified version of the instruction level granularity was created in R-Visor, in order for users to instrument at just one of these types of instructions, rather than for the whole program. These types are also modifiable by ArchVisor. For example, **R Type** was extended to **R4 Type** due to instructions such as `FMADD.S` and `FMSUB.S`

### API Usage

####  rvisor_register_inst_type_routine()

* **Prototype**

```c
void rvisor_register_inst_type_routine(InstType type, void (*func)(), Rvisor_Ipoint ipoint, Rvisor_Invoke invoke)
```
Registers a routine to be called for instructions of a specific type. One routine can be registered per instruction type.

* **Parameters**:
    * `type`: An `InstType` value (defined by ARCH-VISOR specifications) representing the instruction type to target.
    * `func`: A function pointer, typically `Rvisor_Rt_Routine_Inst` for runtime invocation or `Rvisor_Alloc_Routine_Inst` for allocator invocation.
    * `ipoint`: An `Rvisor_Ipoint` enum value.
    * `invoke`: An `Rvisor_Invoke` enum value.
* **Description**: Allows targeted instrumentation based on instruction formats or classes (e.g., all R-type instructions). The routines are managed internally using a hash map (`rvisor_inst_type_routine_map`).

----

## Instruction Group Granularity
In addition to the standard RISC-V instruction types, R-Visor also allows users to instrument based on custom defined groups. These are defined in the ArchVisor sources in [decodeFile.rdec](https://github.com/stamcenter/r-visor/blob/main/src/decodeFile.rdec). The base R-Visor includes two pseudotypes namely:

* Memory Access Type: For Load and Store instructions and
* System Type: For `ECALL` and `EBREAK` instructions

The R-Visor Codebase can also be extended to include additional groups based on the user's required functionality.


### API Usage

#### rvisor_register_inst_group_routine()

* **Prototype**

```c
void rvisor_register_inst_group_routine(RvInstGroup group, void (*func)(), Rvisor_Ipoint ipoint, Rvisor_Invoke invoke)
```

Registers a routine to be called for instructions belonging to a specific group (defined via ARCH-VISOR). One routine can be registered per group.

* **Parameters**:
    * `group`: An `RvInstGroup` value (an extensible construct specified by ARCH-VISOR sources) representing the instruction group.
    * `func`: A function pointer, typically `Rvisor_Alloc_Routine_Inst` for allocator invocation or `Rvisor_Rt_Routine_Inst` for runtime invocation.
    * `ipoint`: An `Rvisor_Ipoint` enum value.
    * `invoke`: An `Rvisor_Invoke` enum value.
* **Description**: Provides a flexible way to instrument custom-defined sets of instructions. Routines are managed internally using `rvisor_inst_group_routine_map`.



<br/><br/>