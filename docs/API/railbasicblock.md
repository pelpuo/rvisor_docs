# R-Visor Basic Block
This section provides an API reference for the definitions related to basic blocks in R-Visor, as specified in R-Visor.

---
## Enums

### `rvisor_bb_type`
Defines the type of a basic block, typically characterized by its terminating instruction or its creation context.
```c
typedef enum {
    BRANCH,             // Basic block ends with a conditional branch instruction
    DIRECT_JUMP,        // Basic block ends with a direct jump (e.g., JAL with known offset)
    INDIRECT_JUMP,      // Basic block ends with an indirect jump (e.g., JALR to a register value)
    SEGMENTED           // Basic block was segmented due to reasons other than a control flow instruction (e.g., max size, instrumentation point)
} rvisor_bb_type;
```
* **`BRANCH`**: The basic block terminates with a conditional branch. Its execution may lead to a "taken" path or a "fall-through" path.
* **`DIRECT_JUMP`**: The basic block terminates with a jump instruction where the target address can be determined statically.
* **`INDIRECT_JUMP`**: The basic block terminates with a jump instruction where the target address is determined at runtime (e.g., from a register).
* **`SEGMENTED`**: The basic block was ended by R-Visor for reasons other than encountering a typical control-flow terminating instruction. This could be due to reaching a maximum block size, an explicit instrumentation point, or other internal R-Visor logic.

---
## Static Data

### `static const char *basic_block_type_str[6]`
An array of strings corresponding to the `rvisor_bb_type` enum values, intended for converting the enum to a human-readable string.
```c
static const char *basic_block_type_str[6] = {
        "Branch", 
        "Direct Jump", 
        "Indirect Jump",
        "Segmented Block"
        };
```

---
## Structs

### `rvisor_basic_block`
Represents a basic block processed and managed by R-Visor. It contains metadata about the original basic block from the target binary and its representation within R-Visor's code cache.
```c
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

* **`first_addr`**: The memory address of the first instruction of this basic block as it appears in the original, uninstrumented binary.
* **`last_addr`**: The memory address of the last instruction (typically a control-flow instruction) of this basic block in the original binary.
* **`start_location_in_cache`**: The offset or address within R-Visor's code cache where the translated/instrumented version of this basic block begins.
* **`end_location_in_cache`**: The offset or address within R-Visor's code cache where the translated/instrumented version of this basic block ends.
* **`num_instructions`**: The count of original RISC-V instructions that constitute this basic block.
* **`taken_block`**: A flag (or indicator) that is set if this basic block was entered because a preceding conditional branch was taken.
* **`type`**: An enum of type `rvisor_bb_type` classifying the basic block based on its terminating instruction or context.
* **`resume`**: A flag or state variable used for managing execution resumption, possibly after handling interrupts, system calls, or other R-Visor interventions.
* **`start_inst`**: The raw 32-bit (or 16-bit if compressed) encoding of the first instruction of this basic block.
* **`terminal_inst`**: The raw 32-bit (or 16-bit if compressed) encoding of the terminating instruction of this basic block.
* **`basic_block_address`**: If this `rvisor_basic_block` structure represents a segment of a larger logical basic block (i.e., `type` is `SEGMENTED`), this field stores the starting address of that original, larger basic block.
* **`taken_addr`**: If the basic block ends in a conditional branch or a jump, this field holds the target address if the branch/jump is taken.
* **`fall_through_addr`**: If the basic block ends in a conditional branch, this field holds the address of the instruction that would execute if the branch is not taken. For non-branching instructions that are part of a segmented block, this might point to the next sequential instruction.
* **`ecall_next`**: If the basic block terminates with an `ECALL` (syscall) instruction, this field stores the address where execution should resume after the system call is handled by R-Visor.

<br/><br/>