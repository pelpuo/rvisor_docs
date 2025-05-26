# Routine Inlining

## Description
While trace linking reduces the intervention of R-Visor during execution, its advantage can be negated when instrumentation routines are present, as one of the dispatcher's tasks is to execute routines at predetermined points through context switches. To maintain the impact of trace linking, we extended the R-Visor's API to insert the instrumentation routine directly into the basic block during allocation, which we dub *routine inlining*. 

Our version of routine inlining allows users to write short snippets in RISC-V assembly with the aid of R-Visor's encoder. This also gives users low-level control over the instrumentation routines being executed. We recommend routine inlining for short instrumentation routines where minimizing cycle overhead is critical and for users proficient in RISC-V assembly.

## Writing Inline Routines

### Collecting Metrics from Default Routine
To demonstrate routine inlining, we will implement an inline version of the [BBCounting](./../building_tools/bb_counting) Routine.

First, let us record the metrics from the default BBCounting routine. To do this we will make use of `bb_counting.cpp` in  the `routines/` folder. 

Modify `CMakeLists.txt` to enable the collection of metrics.

```cmake

# Ensure that -DMETRICS is added to the flags 

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DMETRICS -march=rv64imafd -mabi=lp64d -mno-relax")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DMETRICS -march=rv64imafd -mabi=lp64d -mno-relax")
```

Modify `CMakeLists.txt` to build the `bb_counting` routine.

```cmake

# Replace the add_executable directive with this line
add_executable(bb_counting ${ROUTINESDIR}/bb_counting.cpp ${HEADER_FILES})
```

Build the routine

```bash
cmake .
make
```

Execute the newly built routine while instrumenting one of the test binaries.

```bash
./bb_counting benchmarks/fib
```

After executing, metrics are stored in `bb_counting_logs.txt`

```
Num Basic Blocks: 1546
Total elapsed cycles: 13621698
Total instructions executed: 13621692
```

### Writing Routine in RISC-V
Inline routines are written as individual RISC-V instructions, as such, we would need to plan out the routine using RISC-V code.

The Basic Block counting routine at it's core must keep track of the number of basic blocks observed within a variable, and increment the number whenever a new basic block is encountered. Writing this in RISC-V would require a number of steps:

1. Get the memory address for our variable. We will use the `t6` register for this.

```assembly
LI t6, &num_basic_blocks
```

2. Load and Increment the value stoed within our variable. We will make use of the `t5` register.

```assembly
LD t5, 0(t6)
ADDI t5, t5, 1
```

3. Store the result in the variable

```assembly
SD t5 0(t6)
```

Our routine thus far looks like this

```assembly
LI t6, &num_basic_blocks
LD t5, 0(t6)
ADDI t5, t5, 1
SD t5 0(t6)
```

However, `t6` and `t5` are also being used by the instrumented binary. Since we do not want to corrupt the binary's state, we can store the original values of `t6` and `t5` onto the stack and restore them when the routine completes.

```assembly
#Storing t6 and t5
ADDI sp, sp, -16
SD t6, 0(sp)
SD t5, 8(sp)

LI t6, &num_basic_blocks
LD t5, 0(t6)
ADDI t5, t5, 1
SD t5 0(t6)

#Restoring t6 and t5
LD t6, 0(sp)
LD t5, 8(sp)
ADDI sp, sp, 16

```

### Adding Inline Routine in R-Visor
Now that we have the RISC-V version of our inline routine, we can modify `bb_counting.cpp` to use the inline routines. 

The R-Visor API instrumentation class provides the `addInlineInstforBBRoutine()` method for inserting inline instructions to a program. However, it takes a `uint32_t` as its parameter. We can convert our instruction to this form using R-Visor's RISC-V encoder. The full Instrumentation routine is shown below.

```c++
    railer.addInlineInstforBBRoutine(encoder.encode_Itype(OPIMM, regs::SP, RvFunct::I::ADDI, regs::SP, -16));
    railer.addInlineInstforBBRoutine(encoder.encode_Stype(STORE, RvFunct::S::SD, regs::SP, regs::T6, 0));
    railer.addInlineInstforBBRoutine(encoder.encode_Stype(STORE, RvFunct::S::SD, regs::SP, regs::T5, 8));
    railer.addInlineLiforBBRoutine(reinterpret_cast<uint64_t>(&num_basic_blocks), regs::T6);
    railer.addInlineInstforBBRoutine(encoder.encode_Itype(LOAD, regs::T5, RvFunct::I::LD, regs::T6, 0));
    railer.addInlineInstforBBRoutine(encoder.encode_Itype(OPIMM, regs::T5, RvFunct::I::ADDI, regs::T5, 1));
    railer.addInlineInstforBBRoutine(encoder.encode_Stype(STORE, RvFunct::S::SD, regs::T6, regs::T5, 0));
    railer.addInlineInstforBBRoutine(encoder.encode_Itype(LOAD, regs::T6, RvFunct::I::LD, regs::SP, 0));
    railer.addInlineInstforBBRoutine(encoder.encode_Itype(LOAD, regs::T5, RvFunct::I::LD, regs::SP, 8));
    railer.addInlineInstforBBRoutine(encoder.encode_Itype(OPIMM, regs::SP, RvFunct::I::ADDI, regs::SP, 16));
```

We must also enable [trace linking](./trace_linking) by including the following line 

```c++
railer.enableTraceLinking();
```

The full code is shown below:

```c++
#include <stdio.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <map>

#include "src/system_calls.h"
#include "src/decode.h"
#include "src/rail.h"
#include "src/regfile.h"
#include "src/printUtils.h"
#include "src/instructions.h"


using namespace rail;
static int num_basic_blocks;


#ifdef METRICS
    uint64_t start_cycle, end_cycle;
    uint64_t start_instr, end_instr;    

    void exitRoutine(uint64_t *regfile){
        asm volatile ("csrr %0, instret" : "=r"(end_instr));
        asm volatile ("rdcycle %0" : "=r"(end_cycle));

        outfile << "Number of Basic Blocks: " << num_basic_blocks << endl;

        rail::outfile << "Total elapsed cycles: " << end_cycle-start_cycle << endl;
        rail::outfile << "Total instructions executed: " << end_instr-start_instr << endl;
    }

#endif

int main(int argc, char** argv, char** envp) {

    if(argc < 2){
        cout << "Please provide a target binary" << std::endl;
        exit(1);
    }

    num_basic_blocks = 0;
    

    rail::Rail railer;

    #ifdef METRICS
        railer.setExitRoutine(exitRoutine);
        asm volatile ("rdcycle %0" : "=r"(start_cycle));
        asm volatile ("csrr %0, instret" : "=r"(start_instr));
    #endif

    railer.setTarget(argv[1]);
    railer.registerArgs(argc-1, &argv[1], &envp[0]);
    railer.setLoggingFile("tlrailBB_logs");

    railer.enableTraceLinking();

    RvEncoder encoder;
    railer.addInlineInstforBBRoutine(encoder.encode_Itype(OPIMM, regs::SP, RvFunct::I::ADDI, regs::SP, -16));
    railer.addInlineInstforBBRoutine(encoder.encode_Stype(STORE, RvFunct::S::SD, regs::SP, regs::T6, 0));
    railer.addInlineInstforBBRoutine(encoder.encode_Stype(STORE, RvFunct::S::SD, regs::SP, regs::T5, 8));
    railer.addInlineLiforBBRoutine(reinterpret_cast<uint64_t>(&num_basic_blocks), regs::T6);
    railer.addInlineInstforBBRoutine(encoder.encode_Itype(LOAD, regs::T5, RvFunct::I::LD, regs::T6, 0));
    railer.addInlineInstforBBRoutine(encoder.encode_Itype(OPIMM, regs::T5, RvFunct::I::ADDI, regs::T5, 1));rail::RvDecoder decoder;
    railer.addInlineInstforBBRoutine(encoder.encode_Stype(STORE, RvFunct::S::SD, regs::T6, regs::T5, 0));
    railer.addInlineInstforBBRoutine(encoder.encode_Itype(LOAD, regs::T6, RvFunct::I::LD, regs::SP, 0));
    railer.addInlineInstforBBRoutine(encoder.encode_Itype(LOAD, regs::T5, RvFunct::I::LD, regs::SP, 8));
    railer.addInlineInstforBBRoutine(encoder.encode_Itype(OPIMM, regs::SP, RvFunct::I::ADDI, regs::SP, 16));
    
    railer.runInstrument();
}
```

We can once again compile the routine, execute the binary and check the metrics.

*bb_counting_logs.cpp*
```
Num Basic Blocks: 1546
Total elapsed cycles: 8566194
Total instructions executed: 8566188
```

We can see a significant drop in the number of cycles and instructions, indicating that running trace linking together with inline routines effectively reduces the computational overhead of instrumentation in R-Visor.

<br><br>