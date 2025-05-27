# Advanced Example

## Measuring Basic Block Runtimes

The goal of this tool is to measure the execution time (e.g., in clock cycles) of individual basic blocks within a target binary. This is achieved by:
1.  Injecting assembly instructions to read a hardware timer (like RISC-V's `time` CSR) immediately before and after a basic block's original instructions are executed in the code cache.
2.  Storing these "pre" and "post" timestamps in dedicated memory locations.
3.  Using a C routine, called after each basic block, to calculate the difference between these timestamps.
4.  Collecting all timing data and printing it upon program termination.

To follow along, create a new file, for example, `routines/bb_timing.c`.

The full code for the basic block timing routine is shown below. We will go through each part step by step.

```c
#include <fcntl.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <unistd.h>
#include <string.h> // For strrchr, snprintf

#include "../helpers/uthash.h"
#include "../helpers/vector.h" // For rvisor_vec
#include "../src/code_cache.h"
#include "../src/decode.h"
#include "../src/elf_reader.h"
#include "../src/logger.h"
#include "../src/railBasicBlock.h"
#include "../src/rail.h"

int *basicBlockTimePost;
int *basicBlockTimePre;
int t_delta;

#define CACHE_SIZE (4 * 1024 * 1024) // Referenced in paper, but not directly used in this snippet

rvisor_vec bb_name;   // Vector to store basic block starting addresses
rvisor_vec bb_timing; // Vector to store corresponding execution times

// This routine is called after each basic block executes (and after inlined timing instructions).
void bb_time(rvisor_basic_block bb, uint64_t *regs){
  t_delta = *basicBlockTimePost - *basicBlockTimePre;
  // fprintf( rvisor_logger, "%x,%d\n", bb.first_addr, t_delta); // Alternative: log immediately
  rvisor_vec_push_back(&bb_name, bb.first_addr);   // Store BB address
  rvisor_vec_push_back(&bb_timing, t_delta);        // Store calculated time delta
}

// This routine is called once when the target program exits.
void exitFxn(uint64_t *regs) {
  int i;
  // Iterate through the collected data and print to the log file.
  for (i = 0; i < bb_name.size; i++) {
    fprintf( rvisor_logger, "0x%x,%d\n", bb_name.data[i], bb_timing.data[i]);
  }
  // Free the memory used by the vectors.
  rvisor_vec_free(&bb_name);
  rvisor_vec_free(&bb_timing);
  fclose(rvisor_logger); // Close the log file
  exit(0); // Exit the R-Visor tool
  // rvisor_exit_routine(regs); // Alternative R-Visor exit, if needed
}


int main(int argc, char **argv, char **envp) {

  if (argc < 2) {
    printf("Please provide a target binary\n");
    exit(1);
  }

  // --- Log File Setup ---
  // Dynamically create log file name based on target binary name
  char *base_name_ptr = strrchr(argv[1], '/');
  base_name_ptr = (base_name_ptr != NULL) ? base_name_ptr + 1 : argv[1];
  char log_file_name[256];
  snprintf(log_file_name, sizeof(log_file_name), "%s_bbtime_logs.csv", base_name_ptr);
  set_logging_file(log_file_name, "w");
  fprintf( rvisor_logger, "bb_address,time_cycles\n"); // CSV Header

  // --- R-Visor Initialization ---
  rvisor_init(argv[1]); // Initialize R-Visor with the target binary 

  // --- Memory Allocation for Timestamps ---
  // Allocate space in R-Visor's managed memory for pre-execution timestamp
  basicBlockTimePre = (int *)(memory + rvisor_memory_index);
  *basicBlockTimePre = 0; // Initialize
  rvisor_memory_index +=4; // Increment memory index for next allocation

  // Allocate space for post-execution timestamp
  basicBlockTimePost = (int *)(memory + rvisor_memory_index);
  *basicBlockTimePost = 0; // Initialize
  rvisor_memory_index +=4;

  rvisor_register_args(argc, argv, envp); // Pass arguments to R-Visor

  // --- Inlined Instructions to Capture PRE-BasicBlock Timestamp ---
  // These instructions are intended to run before the original basic block's instructions.
  // They read the 'time' CSR and store it into basicBlockTimePre.
  rvisor_add_inline_inst_bb(encode_ADDI(SP, SP, -16), POST);       // addi sp, sp, -16 (Save GPRs)
  rvisor_add_inline_inst_bb(encode_SD(0, SP, T6), POST);           // sd   t6, 0(sp)
  rvisor_add_inline_inst_bb(encode_SD(8, SP, T5), POST);           // sd   t5, 8(sp)
  rvisor_add_inline_li_bb((uint64_t)(basicBlockTimePre), T6, POST); // lui t6, &basicBlockTimePre
  rvisor_add_inline_inst_bb(0xc0102f73, POST);                     // csrrs T5, time, x0 (T5 = CSR[time])
  // rvisor_add_inline_inst_bb(encode_ADDI(T5, T5, 1), POST);      // Increment T5 (optional, see note)
  rvisor_add_inline_inst_bb(encode_SD(0, T6, T5), POST);           // *(&basicBlockTimePre) = T5
  rvisor_add_inline_inst_bb(0x00013f83, POST);                     // ld t6, 0(sp) (Restore GPRs)
  rvisor_add_inline_inst_bb(0x00813f03, POST);                     // ld t5, 8(sp)
  rvisor_add_inline_inst_bb(encode_ADDI(SP, SP, 16), POST);       // addi sp, sp, 16
  
  // --- Inlined Instructions to Capture POST-BasicBlock Timestamp ---
  // These instructions are intended to run after the original basic block's instructions.
  // They read the 'time' CSR and store it into basicBlockTimePost.
  rvisor_add_inline_inst_bb(encode_ADDI(SP, SP, -16), POST);       // addi sp, sp, -16
  rvisor_add_inline_inst_bb(encode_SD(0, SP, T6), POST);           // sd   t6, 0(sp)
  rvisor_add_inline_inst_bb(encode_SD(8, SP, T5), POST);           // sd   t5, 8(sp)
  rvisor_add_inline_li_bb((uint64_t)(basicBlockTimePost), T6, POST); // lui t6, &basicBlockTimePost
  rvisor_add_inline_inst_bb(0xc0102f73, POST);                     // csrrs T5, time, x0 (T5 = CSR[time])
  // rvisor_add_inline_inst_bb(encode_ADDI(T5, T5, 1), POST);      // Increment T5 (optional, see note)
  rvisor_add_inline_inst_bb(encode_SD(0, T6, T5), POST);           // *(&basicBlockTimePost) = T5
  rvisor_add_inline_inst_bb(0x00013f83, POST);                     // ld t6, 0(sp)
  rvisor_add_inline_inst_bb(0x00813f03, POST);                     // ld t5, 8(sp)
  rvisor_add_inline_inst_bb(encode_ADDI(SP, SP, 16), POST);       // addi sp, sp, 16

  // --- Initialize Vectors for Data Collection ---
  rvisor_vec_init(&bb_name, 20000);   // Pre-allocate space for 20000 BB addresses
  rvisor_vec_init(&bb_timing, 20000); // Pre-allocate space for 20000 timing entries
  
  // --- Register Instrumentation Routines ---
  // Register 'bb_time' to be called AFTER every basic block, during RUNTIME.
  rvisor_register_bb_routine(bb_time, POST, RUNTIME); 
  // Register 'exitFxn' to be called when the program finishes.
  rvisor_register_exit_routine(exitFxn); 

  // --- Start Execution ---
  rvisor_run(); 
}

```

### Global Variables and Data Storage
To manage the timing data, we use:

* `int *basicBlockTimePre;`: A pointer to an integer in R-Visor's managed memory. This will store the timestamp taken before a basic block's execution.
* `int *basicBlockTimePost;`: A pointer to an integer for the timestamp taken after a basic block's execution.
* `int t_delta;`: A temporary variable to hold the calculated time difference.
* `rvisor_vec bb_name;`: A dynamic vector (presumably from `../helpers/vector.h`) to store the starting memory addresses of the basic blocks encountered.
* `rvisor_vec bb_timing;`: A dynamic vector to store the corresponding calculated execution time (`t_delta`) for each basic block.

These vectors allow us to collect data during the program's run and process it all at the end.

----

### Instrumentation Logic
Capturing Timestamps with Inlined Instructions
The core of the timing mechanism involves injecting short sequences of RISC-V assembly instructions directly into the code cache around the original basic block's instructions. This is done using `rvisor_add_inline_inst_bb()`.  R-Visor's API allows these inlined routines to be inserted into the code cache. 


Two separate blocks of inlined instructions are registered:

#### "Pre-Timestamp" Block:
* This sequence is intended to execute before the original instructions of a basic block.
* It saves temporary registers (`T5`, `T6`) to the stack.
* Loads the memory address of basicBlockTimePre into register `T6` using `rvisor_add_inline_li_bb()`.
* Reads the RISC-V time Control and Status Register (`CSR`) into register T5. The instruction `0xc0102f73` corresponds to `csrrs T5, time, x0`, which reads the cycle counter.
(Note: The example code includes `encode_ADDI(T5, T5, 1)` commented out. If active, this would increment the timestamp by 1. While this doesn't affect the delta calculation if done for both pre and post timestamps, the raw stored timestamps would be off by one cycle.)
* Stores the value in `T5` (the timestamp) to the memory location pointed to by basicBlockTimePre.
* Restores `T5` and `T6` from the stack.

#### "Post-Timestamp" Block:
* This sequence is intended to execute after the original instructions of a basic block.
* It performs the same operations as the "Pre-Timestamp" block but uses `basicBlockTimePost` as the memory location to store the new timestamp.

Both blocks are registered with the `POST` flag. For this timing to be accurate for a basic block B, the R-Visor framework must ensure that the "Pre-Timestamp" inlined instructions effectively execute before B's original code in the cache, and the "Post-Timestamp" inlined instructions execute after B's original code, before the `bb_time` C routine is called.

#### Calculating and Storing Time Delta (`bb_time` routine)

```c
void bb_time(rvisor_basic_block bb, uint64_t *regs){
  t_delta = *basicBlockTimePost - *basicBlockTimePre;
  rvisor_vec_push_back(&bb_name, bb.first_addr);
  rvisor_vec_push_back(&bb_timing, t_delta);
}
```

This C function is registered using `rvisor_register_bb_routine(bb_time, POST, RUNTIME)`.  This means it will be called by R-Visor's dispatcher after each basic block (and after the inlined timestamping instructions have executed). 

* It calculates `t_delta` by subtracting the value stored in `*basicBlockTimePre` from `*basicBlockTimePost`.
* It then pushes the starting address of the current basic block (bb.first_addr) onto the `bb_name` vector.
* It pushes the calculated `t_delta` onto the `bb_timing` vector.

####  Printing Results (exitFxn routine)
```c
void exitFxn(uint64_t *regs) {
  int i;
  for (i = 0; i < bb_name.size; i++) {
    fprintf( rvisor_logger, "0x%x,%d\n", bb_name.data[i], bb_timing.data[i]);
  }
  rvisor_vec_free(&bb_name);
  rvisor_vec_free(&bb_timing);
  fclose(rvisor_logger);
  exit(0);
}
```

This function is registered using `rvisor_register_exit_routine(exitFxn)` and is called once when the instrumented program terminates. 

* It iterates through the `bb_name` and `bb_timing` vectors.
* For each entry, it prints the basic block's starting address and its execution time to the log file (e.g., `target_binary_bbtime_logs.csv`) in a CSV format.
* Finally, it frees the memory allocated for the vectors using `rvisor_vec_free()` and closes the log file.

----

### Setting up R-Visor (main function)
The main function orchestrates the entire process:

1. **Argument Check:** Ensures a target binary path is provided.
2. **Log File Configuration:**
* It dynamically generates a log file name based on the input binary's name (e.g., if `argv[1]` is `embench/aha-mont64`, the log will be `aha-mont64_bbtime_logs.csv`).
* It calls `set_logging_file()` to set this as R-Visor's output. 
* A CSV header (`bb_address,time_cycles`) is written to the log file.
3. **R-Visor Initialization:** `rvisor_init(argv[1])` initializes R-Visor with the target binary. 
4. **Timestamp Memory Allocation:**
* `basicBlockTimePre` and `basicBlockTimePost` pointers are assigned memory locations from R-Visor's managed memory region (`memory + rvisor_memory_index`). This shared memory is crucial for the inlined assembly to communicate timestamps to the C routines.
* `rvisor_memory_index` is incremented to reserve the allocated space.
5. **Argument Registration:** `rvisor_register_args(argc, argv, envp)` passes program arguments to R-Visor. 
6. **Inlined Instruction Registration:** The two blocks of assembly instructions for capturing "pre" and "post" timestamps are registered using `rvisor_add_inline_inst_bb()`.
7. **Vector Initialization:** `rvisor_vec_init()` is called for `bb_name` and `bb_timing` to prepare them for storing data, pre-allocating space for `20,000` entries to reduce reallocation overhead.
8. **Routine Registration:**
* `rvisor_register_bb_routine(bb_time, POST, RUNTIME)` registers the bb_time function to be called after every basic block at runtime. 
* `rvisor_register_exit_routine(exitFxn)` registers the `exitFxn` function to be called when the target program finishes. 
9. **Execution:** rvisor_run() starts the instrumented execution of the target binary.


### Building the Basic Block Execution Time Routine
To build our tool we must modify the `CMakeLists.txt` file.

```cmake
# CMakeLists.txt

# Add this line at the end of the file
add_executable(bb_timing ${ROUTINESDIR}/bb_timing.c ${HEADER_FILES})
```

Next we can build our tool

```bash
cmake .
make
```

This will create a binary called `bb_timing` in `bin/`. We can execute this on a RISC-V binary. For this example, we use one of the embench binaries:

```bash
qemu-riscv64 ./bin/bb_timing embench/aha-mont64
```

After the tool executes, the results are stored in `aha-mont64_bb_time_logs.csv`
