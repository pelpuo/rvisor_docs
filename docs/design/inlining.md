# Routine Inlining

## Description
While trace linking reduces the intervention of R-Visor during execution, its advantage can be negated when C/C++ instrumentation routines are present, as one of the dispatcher's tasks is to execute these routines at predetermined points, often involving context switches[cite: 220]. To maintain the performance benefits of trace linking, R-Visor's API allows for the insertion of instrumentation routines directly into the basic block in the code cache during its allocation phase. We dub this feature *routine inlining*.

Our version of routine inlining allows users to write short snippets in RISC-V assembly with the aid of R-Visor's built-in encoder. This gives users low-level control over the instrumentation routines being executed. We recommend routine inlining for short, performance-critical instrumentation routines and for users proficient in RISC-V assembly.

----

## Writing Inline Routines

### Collecting Metrics from Default Routine (Baseline)
To demonstrate the benefits of routine inlining, we will first implement a standard basic block counting routine (similar to a C/C++ callback) and collect performance metrics. We'll then compare these to an inlined version.

Assume you have a `rvisor_bb.c` file in your `routines/` folder designed for this baseline measurement.

1.  **Enable Metrics Collection in CMake:**
    Modify your `CMakeLists.txt` to include the `-DMETRICS` compilation flag. This flag is typically used to enable sections of code (guarded by `#ifdef METRICS`) that collect and report performance data.

    ```cmake
    # Ensure that -DMETRICS is added to the flags
    # Old
    # set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -O3 -march=rv64imafd -mabi=lp64d -mno-relax")
    
    # New
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DMETRICS -O3 -march=rv64imafd -mabi=lp64d -mno-relax")
    ``` 

2.  **Register the `rvisor_bb` Routine:**
    Update your `CMakeLists.txt` to build the `rvisor_bb.c` executable.

    ```cmake
    # Replace or add the add_executable directive for rvisor_bb
    add_executable(rvisor_bb ${ROUTINESDIR}/rvisor_bb.c ${HEADER_FILES})
    ```

3.  **Build the Routine:**
    Navigate to your build directory and compile:

    ```bash
    cmake .
    make
    ```

4.  **Execute and Observe Metrics:**
    Run the newly built routine with a test binary (e.g., `benchmarks/aha-mont64`):

    ```bash
    qemu-riscv64 ./bin/rvisor_bb benchmarks/aha-mont64
    ```

    After execution, performance metrics (if the `METRICS` section is implemented to do so) might be stored in a log file, for instance, `rvisor_bb_logs.txt`. Example output is shown below:

    ```
    Num Basic Blocks: 1546
    Total elapsed cycles: 13621698
    Total instructions executed: 13621692
    ```
    This provides a baseline for comparison.

### Writing the Counting Logic in RISC-V
Inline routines are composed of individual RISC-V instructions. For basic block counting, our goal is to increment a counter variable each time a basic block is executed.

Here's the plan for the RISC-V assembly sequence:

1.  **Get the memory address of our counter variable (`num_basic_blocks`).** We'll use the `t6` register to hold this address.
    ```assembly
    LI t6, &num_basic_blocks  # Load Immediate (pseudo-instruction for address)
    ```

2.  **Load the current value of the counter, increment it.** We'll use the `t5` register for the counter's value.
    ```assembly
    LD t5, 0(t6)    # Load Doubleword from address in t6 into t5
    ADDI t5, t5, 1  # Add Immediate: increment t5
    ```

3.  **Store the new value back to the counter variable.**
    ```assembly
    SD t5, 0(t6)    # Store Doubleword from t5 to address in t6
    ```

The routine so far:
```assembly
LI t6, &num_basic_blocks
LD t5, 0(t6)
ADDI t5, t5, 1
SD t5, 0(t6)
```

However, the registers `t5` and `t6` are general-purpose temporary registers that the instrumented binary might also be using. To avoid corrupting the binary's state (a key principle of transparent instrumentation [cite: 54, 141]), we must save their original values before using them and restore them after our routine completes. We can use the stack for this:

```assembly
# Save original values of t6 and t5
ADDI sp, sp, -16  # Adjust stack pointer to make space for 2 registers (8 bytes each)
SD t6, 0(sp)      # Store t6 onto the stack
SD t5, 8(sp)      # Store t5 onto the stack (at a different offset)

# Core counting logic
LI t6, &num_basic_blocks # Target address of num_basic_blocks into t6
LD t5, 0(t6)             # Load value of num_basic_blocks into t5
ADDI t5, t5, 1           # Increment t5
SD t5, 0(t6)             # Store incremented value back to num_basic_blocks

# Restore original values of t6 and t5
LD t6, 0(sp)      # Restore t6 from the stack
LD t5, 8(sp)      # Restore t5 from the stack
ADDI sp, sp, 16   # Adjust stack pointer back
```

### Adding the Inline Routine in R-Visor (C API)

Now that we have planned our RISC-V sequence, we can use R-Visor's C API to inject these instructions. We'll create a new C file `tlrvisor_bb.c`.

R-Visor's C API provides functions like `rvisor_add_inline_inst_bb()` to insert a single instruction (as a `uint32_t` encoding) and `rvisor_add_inline_li_bb()` as a helper for loading immediate values (like addresses). We use R-Visor's C encoder functions (e.g., `encode_ADDI`, `encode_SD`, `encode_LD`) to convert our planned assembly into the required `uint32_t` instruction encodings.

Here's how the C code would look for adding the inlined instructions:

```c
    // ADDI sp, sp, -16
    rvisor_add_inline_inst_bb(encode_ADDI(SP, SP, -16), POST);
    // SD t6, 0(sp)
    rvisor_add_inline_inst_bb(encode_SD(0, SP, T6), POST);
    // SD t5, 8(sp)
    rvisor_add_inline_inst_bb(encode_SD(8, SP, T5), POST);

    // LI t6, &num_basic_blocks (R-Visor helper for loading address)
    rvisor_add_inline_li_bb((uint64_t)(&num_basic_blocks), T6, POST);
    // LD t5, 0(t6)
    rvisor_add_inline_inst_bb(encode_LD(T5, 0, T6), POST);
    // ADDI t5, t5, 1
    rvisor_add_inline_inst_bb(encode_ADDI(T5, T5, 1), POST);
    // SD t5, 0(t6)
    rvisor_add_inline_inst_bb(encode_SD(0, T6, T5), POST);

    // LD t6, 0(sp)
    rvisor_add_inline_inst_bb(encode_LD(T6, 0, SP), POST);
    // LD t5, 8(sp)
    rvisor_add_inline_inst_bb(encode_LD(T5, 8, SP), POST);
    // ADDI sp, sp, 16
    rvisor_add_inline_inst_bb(encode_ADDI(SP, SP, 16), POST);
```

### Full C++ Code with Inlining

Here's the complete `tlrvisor_bb.c` modified for inline basic block counting:

```c
#include <fcntl.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <unistd.h>
#include <string.h> // For strrchr, snprintf if used for dynamic log names

// R-Visor C Headers (adjust paths as needed)
#include "../helpers/uthash.h"   // If used by other parts not shown
#include "../src/code_cache.h" // For code cache related definitions
#include "../src/decode.h"     // For encoder functions (encode_ADDI, etc.) and register defs (SP, T5, T6)
#include "../src/elf_reader.h" // For ELF reading functionalities
#include "../src/logger.h"     // For rvisor_logger, set_logging_file
#include "../src/rail.h"       // For core R-Visor API (rvisor_init, etc.)

// If register definitions like SP, T5, T6 are not in decode.h, define/include them appropriately
// For example, from a hypothetical regs.h or directly:
// #define SP 2
// #define T5 30
// #define T6 31
// These are illustrative; use actual values from your R-Visor's ISA definition.

static int num_basic_blocks; // Our counter variable

#ifdef METRICS
    uint64_t start_cycle, end_cycle;
    uint64_t start_instr, end_instr;

    // This routine is called once when the target program exits if METRICS is defined.
    void exit_routine_metrics(uint64_t *regfile){ // regfile might be unused
        asm volatile ("csrr %0, instret" : "=r"(end_instr)); // Read instruction retired counter
        asm volatile ("rdcycle %0" : "=r"(end_cycle));      // Read cycle counter

        fprintf(rvisor_logger, "Num Basic Blocks: %d\n", num_basic_blocks);
        fprintf(rvisor_logger, "Total elapsed cycles: %lu\n", (unsigned long)(end_cycle - start_cycle));
        fprintf(rvisor_logger, "Total instructions executed: %lu\n", (unsigned long)(end_instr - start_instr));
        
        fclose(rvisor_logger); // Ensure logger is closed
    }
#else
    // Default exit routine if METRICS is not defined
    void exit_routine_default(uint64_t *regfile) {
        fprintf(rvisor_logger, "Num Basic Blocks: %d\n", num_basic_blocks);
        fclose(rvisor_logger);
    }
#endif

int main(int argc, char** argv, char** envp) {

    if(argc < 2){
        printf("Please provide a target binary\n");
        exit(1);
    }

    num_basic_blocks = 0; // Initialize counter
    
    // Set log file (can be dynamic as in your other examples)
    set_logging_file("bb_counting_inline_logs.txt", "w");

    #ifdef METRICS
        // Record start cycles and instructions retired.
        asm volatile ("rdcycle %0" : "=r"(start_cycle));
        asm volatile ("csrr %0, instret" : "=r"(start_instr));
    #endif

    rvisor_trace_linking_enabled = 1; // Enable trace linking for better performance.
    rvisor_init(argv[1]); // Initialize R-Visor with the target binary.
    
    // Register arguments for the target program.
    rvisor_register_args(argc, argv, envp); 
                                         
    // --- Injecting the Inlined Routine ---
    // Assumes POST means these instructions are added to the code cache to be executed
    // in sequence with the basic block, effectively acting as instrumentation.

    // ADDI sp, sp, -16
    rvisor_add_inline_inst_bb(encode_ADDI(SP, SP, -16), POST);
    // SD t6, 0(sp)
    rvisor_add_inline_inst_bb(encode_SD(0, SP, T6), POST);
    // SD t5, 8(sp)
    rvisor_add_inline_inst_bb(encode_SD(8, SP, T5), POST);
    // LI t6, &num_basic_blocks
    rvisor_add_inline_li_bb((uint64_t)(&num_basic_blocks), T6, POST);
    // LD t5, 0(t6)
    rvisor_add_inline_inst_bb(encode_LD(T5, 0, T6), POST);
    // ADDI t5, t5, 1
    rvisor_add_inline_inst_bb(encode_ADDI(T5, T5, 1), POST);
    // SD t5, 0(t6)
    rvisor_add_inline_inst_bb(encode_SD(0, T6, T5), POST);
    // LD t6, 0(sp)
    rvisor_add_inline_inst_bb(encode_LD(T6, 0, SP), POST);
    // LD t5, 8(sp)
    rvisor_add_inline_inst_bb(encode_LD(T5, 8, SP), POST);
    // ADDI sp, sp, 16
    rvisor_add_inline_inst_bb(encode_ADDI(SP, SP, 16), POST);
    
    // Register the appropriate exit routine.
    #ifdef METRICS
        rvisor_register_exit_routine(exit_routine_metrics);
    #else
        rvisor_register_exit_routine(exit_routine_default);
    #endif
    
    rvisor_run(); // Start R-Visor and the target program.

    return 0; 
}
```

### Comparing Metrics
After building this inlined version (ensure `CMakeLists.txt` now points to `tlrvisor_bb.c`):

```bash
cmake .
make

qemu-riscv64 ./bin/tlrvisor_bb benchmarks/aha-mont64
```

Check the output file (e.g., `rvisor_bb_inline_logs.txt` as set by `setLoggingFile`):

```
Num Basic Blocks: 1546
Total elapsed cycles: 8566194
Total instructions executed: 8566188
```

Comparing these example results:

* **Default Routine (Callback-based):**

    * Total elapsed cycles: `13621698`
    * Total instructions executed: `13621692`

* **Inlined Routine:**

    * Total elapsed cycles: `8566194`
    * Total instructions executed: `8566188`

We can observe a significant reduction in both total elapsed cycles and instructions executed. This demonstrates that using R-Visor's routine inlining feature, especially in conjunction with trace linking, can effectively reduce the computational overhead of instrumentation. This makes it a valuable technique for performance-sensitive analysis tasks.

<br/><br/>