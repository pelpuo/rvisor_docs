# Routine Inlining Example

## Basic Block Counting via Inlined Instructions

The goal of this tool is to count the total number of executed basic blocks within a target binary. However, instead of registering a C function to be called for each basic block, this example demonstrates how to **inline assembly instructions directly** into the execution flow to achieve the same result. This can offer performance benefits by avoiding the overhead of a function call for very frequent events.

To follow along with this exercise, create a new file, for example, `routines/inline_bb_count.c`.

The full code for the routine inlining basic block counting is shown below. We will go through step by step to explain each part of the code.

```c
#include <fcntl.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <unistd.h>

#include "../helpers/uthash.h"
#include "../src/code_cache.h"
#include "../src/decode.h"
#include "../src/elf_reader.h"
#include "../src/logger.h"
#include "../src/railBasicBlock.h"
#include "../src/rail.h"

int num_instructions; // Unused in this specific counting mechanism
int num_basic_blocks; // Unused in this specific counting mechanism
#define CACHE_SIZE (4 * 1024 * 1024)

// This function is declared but not registered or used for counting in this example.
void bb_counting(rvisor_basic_block bb, uint64_t *regs){
  num_basic_blocks++;
}

void exitFxn(uint64_t *regfile){
    // The count is read from a specific memory address manipulated by inlined instructions.
    int *bbaddr = (int *)(0x5000000); // Address where the count is stored
    fprintf( rvisor_logger, "Number of Basic Blocks: %d\n", *bbaddr);
    fclose(rvisor_logger);
}

int main(int argc, char **argv, char **envp) {
  set_logging_file("inline_bb_count.txt", "w");

  if (argc < 2) {
    printf("Please provide a target binary\n");
    exit(1);
  }

  // Enable trace linking, which might be necessary for some inlining features.
  rvisor_trace_linking_enabled = 1;
  rvisor_init(argv[1]);

  // Initialize the memory location for the basic block counter.
  // R-Visor will manage this memory. We assume 0x5000000 is, or will be mapped to,
  // this R-Visor managed memory region.
  // For this example, we'll assume the inlined code directly uses 0x5000000.
  // If R-Visor provides a mechanism to get the address of 'memory + rvisor_memory_index'
  // that the target program can use, that would be more robust.
  // For now, let's assume the counter at 0x5000000 is initialized to zero by R-Visor
  // or the target environment, or this line correctly initializes the counter
  // at the address that will be used by the inlined code.
  // *(int *)(memory + rvisor_memory_index) = 0;
  // rvisor_memory_index +=4;
  // For simplicity in this example, we rely on the inlined code targeting a known address (0x5000000)
  // and assume it's appropriately zero-initialized before counting begins.

  rvisor_register_args(argc, argv, envp);

  // --- Inlined Instructions to Increment Basic Block Counter ---
  // These instructions will be executed after each basic block in the target.

  // 1. Save temporary registers t6 (x31) and t5 (x30) to the stack
  rvisor_add_inline_inst_bb(encode_ADDI(SP, SP, -16), POST); // addi sp, sp, -16
  rvisor_add_inline_inst_bb(encode_SD(0, SP, T6), POST);     // sd   t6, 0(sp)  (assuming T6 is x31)
  rvisor_add_inline_inst_bb(encode_SD(8, SP, T5), POST);     // sd   t5, 8(sp)  (assuming T5 is x30)

  // 2. Load current count, increment, and store back
  // lui t6, 0x5000   (t6 = 0x5000000, the address of our counter)
  rvisor_add_inline_inst_bb(0x05000fb7, POST);
  // ld t5, 0(t6)     (t5 = current_count from memory[0x5000000])
  rvisor_add_inline_inst_bb(0x000fbf03, POST);
  // addi t5, t5, 1   (t5 = current_count + 1)
  rvisor_add_inline_inst_bb(encode_ADDI(T5, T5, 1), POST);
  // sd t5, 0(t6)     (memory[0x5000000] = new_count)
  rvisor_add_inline_inst_bb(encode_SD(0, T6, T5), POST);

  // 3. Restore temporary registers t6 and t5 from the stack
  // ld t6, 0(sp)
  rvisor_add_inline_inst_bb(0x00013f83, POST); // (This encoding is for ld x31 (t6), 0(sp))
  // ld t5, 8(sp)
  rvisor_add_inline_inst_bb(0x00813f03, POST); // (This encoding is for ld x30 (t5), 8(sp))
  // addi sp, sp, 16
  rvisor_add_inline_inst_bb(encode_ADDI(SP, SP, 16), POST);

  // Register the exit routine to print the final count.
  rvisor_register_exit_routine(exitFxn);

  rvisor_run();
}
```

### Setting up the Inlined Instrumentation 
Instead of a C callback function for each basic block, we directly inject a sequence of RISC-V assembly instructions. These instructions are responsible for incrementing a counter stored at a known memory address (`0x5000000` in this example).

The global variable num_basic_blocks and the function bb_counting are present in the code but are not used for counting in this inlining example. The counting is handled entirely by the inlined assembly.

The core of the instrumentation logic lies in these R-Visor API calls:

```c
// 1. Save temporary registers t6 (x31) and t5 (x30) to the stack
rvisor_add_inline_inst_bb(encode_ADDI(SP, SP, -16), POST);
rvisor_add_inline_inst_bb(encode_SD(0, SP, T6), POST);
rvisor_add_inline_inst_bb(encode_SD(8, SP, T5), POST);

// 2. Load current count, increment, and store back
rvisor_add_inline_inst_bb(0x05000fb7, POST); // lui t6, 0x5000 (sets t6 to 0x5000000)
rvisor_add_inline_inst_bb(0x000fbf03, POST); // ld t5, 0(t6)
rvisor_add_inline_inst_bb(encode_ADDI(T5, T5, 1), POST);
rvisor_add_inline_inst_bb(encode_SD(0, T6, T5), POST);

// 3. Restore temporary registers t6 and t5 from the stack
rvisor_add_inline_inst_bb(0x00013f83, POST); // ld t6, 0(sp)
rvisor_add_inline_inst_bb(0x00813f03, POST); // ld t5, 8(sp)
rvisor_add_inline_inst_bb(encode_ADDI(SP, SP, 16), POST);
```

`rvisor_add_inline_inst_bb` is used to add individual machine instructions that will be executed after (`POST`) each basic block of the target application. The function takes a `uint32_t`, and the routine placement (`PRE` or `POST`) as argument. Rather than directly writing the integer encoding for each instruction, R-Visor's API provides helpers to encode these instructions. Examples seen are `encode_ADDI` and `encode_SD`. Within the routine, we use a mix of the direct integer encoding, and encoding through R-Visor's helpers.
The sequence performs the following actions:

1. **Saves Registers:** It first saves registers `T6` and `T5` (commonly x31 and x30 in RISC-V) to the stack. This is crucial to avoid corrupting the target program's state, as we are going to modify these registers with out instrumentation routine.
2. **Access Counter:**
`lui t6, 0x5000` (encoded as `0x05000fb7`): Loads the upper immediate value 0x5000 into register `t6`, effectively setting `t6` to the memory address `0x5000000`. This address is where our basic block counter is stored.
`ld t5, 0(t6)` (encoded as `0x000fbf03`): Loads the 8-byte value (double word) from the memory address pointed to by `t6` (which is `0x5000000`) into register `t5`. `t5` now holds the current count.
3. **Increment Counter:** `encode_ADDI(T5, T5, 1)` generates the instruction to add 1 to the value in `t5`.
4. **Store Counter:** `encode_SD(0, T6, T5)` generates the instruction to store the new value from `t5` back into memory at the address `0(t6)` (i.e., `0x5000000`).
5. **Restores Registers:** Finally, it restores the original values of `T6` and `T5` from the stack and adjusts the stack pointer.

The `encode_ADDI` and `encode_SD` functions are assumed to be helper functions provided by R-Visor or your environment to generate the correct RISC-V instruction encodings. Some instructions are provided as raw hexadecimal values.

---


### Module Level Routine (Exit Function)
Once the target program finishes execution, we need to print the total count. The exitFxn handles this:

```c
void exitFxn(uint64_t *regfile){
    int *bbaddr = (int *)(0x5000000); // Address where the count is stored
    fprintf( rvisor_logger, "Number of Basic Blocks: %d\n", *bbaddr);
    fclose(rvisor_logger);
}
```

This function is registered as an exit routine. Its key actions are:

* It declares a pointer `bbaddr` to the memory address `0x5000000`. This is the same address that our inlined assembly instructions were using to store and update the basic block count.
* It prints the integer value stored at `*bbaddr` to the `rvisor_logger`.
* It closes the logger file.

Note that `exitFxn`, like other module-level routines, takes a pointer to the register file as an argument, though it's not used in this particular function.

### Setting up R-Visor and Registering Routines

The main function orchestrates the setup and execution:

```c
int main(int argc, char **argv, char **envp) {
  set_logging_file("inline_bb_count.txt", "w"); // Output log file

  if (argc < 2) {
    printf("Please provide a target binary\n");
    exit(1);
  }

  rvisor_trace_linking_enabled = 1; // Enable trace linking
  rvisor_init(argv[1]);             // Initialize R-Visor with the target binary

  // Argument registration for the target program
  rvisor_register_args(argc, argv, envp);

  // Inlining instructions (as shown in the previous section)
  rvisor_add_inline_inst_bb(encode_ADDI(SP, SP, -16), POST);
  // ... (all other rvisor_add_inline_inst_bb calls) ...
  rvisor_add_inline_inst_bb(encode_ADDI(SP, SP, 16), POST);

  // Register the exit routine
  rvisor_register_exit_routine(exitFxn);

  // Start R-Visor and the target program execution
  rvisor_run();
}
```

Our `main` function does the following:

1. **Logging:** `set_logging_file("inline_bb_count.txt", "w");` initializes the log file where results will be written.
2. **Target Binary Registration:** The target binary is taken from command-line arguments, and `rvisor_init(argv[1]);` initializes R-Visor with it.
3. **Trace Linking Setup:** `rvisor_trace_linking_enabled = 1;` enables trace linking which reduces the execution overhead of the instrumentation routine.
4. **Argument Registration:** `rvisor_register_args(argc, argv, envp);` passes the command-line arguments and environment variables to R-Visor for the target program.
5. **Inlining Instructions:** As detailed previously, a series of `rvisor_add_inline_inst_bb()` calls inject the assembly code to perform the counting. These instructions are set to execute `POST` (after) each basic block of the target.
6. **Exit Routine:** `rvisor_register_exit_routine(exitFxn);` registers our `exitFxn` to be called when the target program terminates.
7. **Run:** `rvisor_run();` starts the instrumented execution of the target binary.

Unlike the previous basic block counting example, `rvisor_register_bb_routine()` is not used here because we are not registering a C callback for each basic block. The instrumentation is done directly with inlined assembly.


### Building the Inline Basic Block Counting Routine
To build our tool we must modify the `CMakeLists.txt` file.

```cmake
# CMakeLists.txt

# Add this line at the end of the file
add_executable(inline_bb_count ${ROUTINESDIR}/inline_bb_count.c ${HEADER_FILES})
```

Next we can build our tool

```bash
cmake .
make
```

This will create a binary called `inline_bb_count` in `bin/`. We can execute this on a RISC-V binary. For this example, we use one of the embench binaries:

```bash
qemu-riscv64 ./bin/inline_bb_count embench/aha-mont64
```

After the tool executes, the results are stored in `inline_bb_count.txt`
