# Basic Example

## Basic Block Counting
The goal of this tool is to count the total number of executed basic blocks within a target binary. To follow along with this exercise, create a new file `routines/count_basic_blocks.c`.
<!-- ## Setting Up
To begin, we can create a file called `bb_counting.cpp` in the `examples` folder and paste the following code. This code counts instructions. We will be modifying this code so that we would be counting basic blocks rather than instructions. -->

The full code for the basic block counting routine is shown below. We will go through step by step to explain each part of the code.

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

int num_basic_blocks;

void bb_counting(rvisor_basic_block bb, uint64_t *regs){
  num_basic_blocks++;
}

void exit_fn(uint64_t *regfile){
    fprintf( rvisor_logger, "Number of Basic Blocks: %d\n", num_basic_blocks);
    fclose(rvisor_logger);
}

int main(int argc, char **argv, char **envp) {
  set_logging_file("basic_block_count.txt", "w");

  if (argc < 2) {
    printf("Please provide a target binary\n");
    exit(1);
  }

  rvisor_init(argv[1]);
  rvisor_register_args(argc, argv, envp);

  rvisor_register_bb_routine(bb_counting, POST, RUNTIME);
  
  rvisor_register_exit_routine(exit_fn);

  rvisor_run();

}

```

### Setting up the Instrumentation Routines
In order to count the basic blocks executed within a target binary, our routine would need a variable to keep track of the basic blocks encountered. For this we make use of `num_basic_block`, declared after the imports. Next, we use 2 different routines to count the number of basic blocks and print out the result at the end of the program's execution.

#### Basic Block Level Routine

Our routine is simple, each time a basic block is encountered, increment `num_basic_blocks`. We will call this routine `bb_counting` and it is declared as follows:

```c
void bb_counting(rvisor_basic_block bb, uint64_t *regs){
  num_basic_blocks++;
}
```

You may notice that `bb_counting` takes 2 arguments. Each instrumentation routine in R-Visor has a specified format to follow, and this includes the arguments it must have. For a basic block level routine (which `bb_counting` will be registered as) these arguments are:

* An R-Visor basic block object (`rvisor_basic_block bb`)
* A uint64 pointer to the register file (`uint64_t *regs`)

The basic block object gives us access to the metadata collected during the basic block allocation step and this includes details such as the start and ending addresses, as well as the number of instructions. 

The contents of the register file are passed as the second argument. These are modifiable by the instrumentation routine and as such, can be used to manipulate program execution. 

#### Module Level Routine
Once all the basic blocks have been counted, we must print out the total count at the end of the program. For this, we make use of a module level routine `exit_fn`, declared as follows:

```c
void exit_fn(uint64_t *regfile){
    fprintf( rvisor_logger, "Number of Basic Blocks: %d\n", num_basic_blocks);
    fclose(rvisor_logger);
}
```

Similar to the `bb_counting` routine, the arguments must follow a specified format. 

`exit_fn` and other module level routines take 1 argument - A uint64 pointer to the register file (`uint64_t *regfile`). 

Within `exit_fn` we print out the value of the `num_basic_blocks` to `rvisor_logger`, the logging file provided by R-Visor.



### Setting up R-Visor and Registering Routines
Once the routines have been created, we create the main function which is responsible for setting up R-Visor and registering our created instrumentation routines.

```c
int main(int argc, char **argv, char **envp) {
  set_logging_file("basic_block_count.txt", "w");

  if (argc < 2) {
    printf("Please provide a target binary\n");
    exit(1);
  }

  rvisor_init(argv[1]);
  rvisor_register_args(argc, argv, envp);

  rvisor_register_bb_routine(bb_counting, POST, RUNTIME);
  
  rvisor_register_exit_routine(exit_fn);

  rvisor_run();

}
```

Our main function first starts by initializing our logging file, which was used in our `exit_fn` to print out the value of the counted basic blocks. This is done using `set_logging_file("basic_block_count.txt", "w");`. The arguments provided are the name of the file and the mode ("w" for write or "a" for append).

Next we register our target binary as a command line argument:

```c
if (argc < 2) {
printf("Please provide a target binary\n");
exit(1);
}
rvisor_init(argv[1]);
```
This also includes sanity checks to ensure that a valid argument is provided.

Following this, we pass the arguments to R-Visor using `rvisor_register_args(argc, argv, envp);`

Once our arguments have been registered, we register the instrumentation routines. We start by registering the `bb_counting` routine:

```c
rvisor_register_bb_routine(bb_counting, POST, RUNTIME);
```

`rvisor_register_bb_routine` is a function provided by R-Visor. This takes 3 arguments:

* The basic block routine
* When the routine should be executed before the target (`PRE`) or after (`POST`)
* Whether the routine should be executed once during basic block allocation (`ALLOCATOR`) or every time the basic block is encountered (`RUNTIME`)

Next we register the exit routine using `rvisor_register_exit_routine(exit_fn)`

Finally, everything has been set up, so we can start R-Visor using `rvisor_run()`.


### Building the Basic Block Counting Routine
To build our tool we must modify the `CMakeLists.txt` file.

```cmake
# CMakeLists.txt

# Add this line at the end of the file
add_executable(count_basic_blocks ${ROUTINESDIR}/count_basic_blocks.c ${HEADER_FILES})
```

Next we can build our tool

```
cmake .
make
```

This will create a binary called `count_basic_blocks` in `bin/`. We can execute this on a RISC-V binary. For this example, we use one of the embench binaries:

```
qemu-riscv64 ./bin/count_basic_blocks embench/aha-mont64
```

After the tool executes, the results are stored in `basic_block_count.txt`
