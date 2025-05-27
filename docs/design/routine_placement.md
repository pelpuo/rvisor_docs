# Routine Placement

## Pre vs Post Routines

Based on the target [instrumentation granularities](../instrumentation_granularity), there are two potential positions for instrumentation routines to be placed. 

Consider a scenario where a user would want to instrument the instruction `ADD a1, a1, a2` in order to check the values of `a1` and `a2` supplied to the instruction. If the instrumentation executes after the instruction completes, then the value of `a1` would already have been changed by the instruction and the reported values, although correct, would not be those required by the user. However, in a scenario where the user would want to know the value of `a1` after the `ADD` operation, then the operation should complete before the instrumentation routine is executed. 

Based on these scenarios, R-Visor allows users to determine whether their instrumentation routine should execute before (`PRE`) or after (`POST`) an instruction. This is further detailed in the API Reference as well as the [Tutorials](../../tutorials/basic_example) setions.


### API Usage

The `PRE` vs `POST` options are declared in R-Visor through the `Rvisor_Ipoint` enum:

```c
typedef enum{
    PRE,  // Before the event
    POST  // After the event
} Rvisor_Ipoint;
```


The `Rvisor_Ipoint` is typically the second argument to the routine registration functions. For example:

1. #### rvisor_register_inst_type_routine()

* **Prototype**

```c
void rvisor_register_inst_type_routine(InstType type, void (*func)(), Rvisor_Ipoint ipoint, Rvisor_Invoke invoke)
```
Registers a routine to be called for instructions of a specific type. One routine can be registered per instruction type.

    * **Parameters**:
        * `type`: An `InstType` value (defined by ARCH-VISOR specifications) representing the instruction type to target.
        * `func`: A function pointer, typically `Rvisor_Rt_Routine_Inst` for runtime invocation or `Rvisor_Alloc_Routine_Inst` for allocator invocation.
        * `ipoint`: `PRE` or `POST`.
        * `invoke`: An `Rvisor_Invoke` enum value.
    * **Description**: Allows targeted instrumentation based on instruction formats or classes (e.g., all R-type instructions). The routines are managed internally using a hash map (`rvisor_inst_type_routine_map`).

2. #### rvisor_register_inst_routine()

* **Prototype**

```c
void rvisor_register_inst_routine(void (*func)(), Rvisor_Ipoint ipoint, Rvisor_Invoke invoke)
```

Registers a single routine to be called for every instruction. Only one such routine can be active.

    * **Parameters**:
        * `func`: A function pointer. Its signature should match `Rvisor_Alloc_Routine_Inst` if `invoke` is `ALLOCATOR`, or `Rvisor_Rt_Routine_Inst` if `invoke` is `RUNTIME`.
        * `ipoint`: `PRE` or `POST`.
        * `invoke`: An `Rvisor_Invoke` enum value (`ALLOCATOR` or `RUNTIME`).
    * **Description**: Enables fine-grained instrumentation at the individual instruction level.


---- 

## Allocator vs Runtime Routines

R-Visor provides two primary timings for executing your instrumentation routines: **Allocator** and **Runtime**. This timing is specified through the `Rvisor_Invoke` parameter (either `ALLOCATOR` or `RUNTIME`) when you register your routines.

**Allocator routines** are invoked **once** for each targeted basic block or instruction. This occurs during the allocation process, which is when R-Visor first encounters that specific piece of code and prepares it for placement into the internal code cache. Because they run only at this initial processing stage, allocator routines are well-suited for tasks like collecting static profiles of the binary, such as counting the distinct types of instructions or basic blocks that are part of the program's execution path. They can also be employed for making one-time static modifications to instructions, such as in binary translation or patching, before the code is cached.

Conversely, **runtime routines** are triggered **each time** their associated target (an instruction, basic block, or trace) is actually executed by the processor. If a particular basic block runs a thousand times, a runtime routine registered for it will also execute a thousand times. This makes them essential for dynamic analysis, where you need to observe the program's behavior as it happens. Common uses include dynamically counting the total execution frequency of code segments, tracing memory accesses, or inspecting register values at specific points during the program's live execution.

In short, `ALLOCATOR` routines are ideal for setup tasks or gathering a static view of the code as it's being prepared, while `RUNTIME` routines are necessary for observing and interacting with the dynamic, live execution of the program.


### API Usage

The `ALLOCATOR` vs `RUNTIME` options are declared in R-Visor through the `Rvisor_Invoke` enum:

```c
typedef enum{
    ALLOCATOR, // During basic block allocation (once)
    RUNTIME    // Every time the event occurs
} Rvisor_Invoke;
```

This option is typically the third argument of the routine registration functions. For example:


1. #### rvisor_register_inst_type_routine()

* **Prototype**

```c
void rvisor_register_inst_type_routine(InstType type, void (*func)(), Rvisor_Ipoint ipoint, Rvisor_Invoke invoke)
```

Registers a routine to be called for instructions of a specific type. One routine can be registered per instruction type.

    * **Parameters**:
        * `type`: An `InstType` value (defined by ARCH-VISOR specifications) representing the instruction type to target.
        * `func`: A function pointer, typically `Rvisor_Rt_Routine_Inst` for runtime invocation or `Rvisor_Alloc_Routine_Inst` for allocator invocation.
        * `ipoint`: An `Rvisor_Ipoint` enum value.
        * `invoke`: `ALLOCATOR` or `RUNTIME`.
    * **Description**: Allows targeted instrumentation based on instruction formats or classes (e.g., all R-type instructions). The routines are managed internally using a hash map (`rvisor_inst_type_routine_map`).

2. #### rvisor_register_inst_routine()

* **Prototype**

```c
void rvisor_register_inst_routine(void (*func)(), Rvisor_Ipoint ipoint, Rvisor_Invoke invoke)
```

Registers a single routine to be called for every instruction. Only one such routine can be active.

    * **Parameters**:
        * `func`: A function pointer. Its signature should match `Rvisor_Alloc_Routine_Inst` if `invoke` is `ALLOCATOR`, or `Rvisor_Rt_Routine_Inst` if `invoke` is `RUNTIME`.
        * `ipoint`: An `Rvisor_Ipoint` enum value.
        * `invoke`: `ALLOCATOR` or `RUNTIME`.
    * **Description**: Enables fine-grained instrumentation at the individual instruction level.




<br/><br/>