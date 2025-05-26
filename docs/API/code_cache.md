# Code Cache
This section provides an API reference for the R-Visor code cache management interface defined in `code_cache.h`. This module is responsible for managing the memory region where R-Visor stores translated and instrumented code (the code cache), as well as data structures for tracking basic blocks.

---
## Macros

### `CACHE_SIZE`
Defines the total size of the main code cache memory region.
```c
#define CACHE_SIZE (4 * 1024 * 1024) // 4 MB
```

---
### `BB_BLOCK_SIZE`
Defines the number of `bb_entry` structures to preallocate in chunks for the basic block pool.
```c
#define BB_BLOCK_SIZE 2048
```

* **Description**: This is used by `allocate_bb_pool` to allocate memory for `bb_entry` structures in blocks, rather than one by one.

---
## Extern Variables

These variables are globally accessible and manage the state of the code cache and related R-Visor components.

### `extern char *memory;`
A pointer to the beginning of the allocated code cache memory region where translated basic blocks and inlined instructions are stored.

---
### `extern int rvisor_memory_index;`
An integer acting as a pointer or offset into the `memory` buffer, indicating the next available byte for writing new code into the cache.

---
### `extern int rvisor_stub_code;`
Likely an offset or flag related to stub code used for context switching or linking within the code cache.

---
### `extern uint64_t rvisor_current_bb;`
Stores the address or identifier of the currently executing or most recently processed basic block.

---
### `extern bb_exit_address *bb_exit_map;`
A pointer to a hash table (`uthash`) that maps original binary addresses of basic block exits (or indirect jump targets) to their corresponding addresses within the R-Visor code cache. This is used for trace linking.

---
### `extern bb_entry *basic_blocks_map;`
A pointer to a hash table (`uthash`) that maps the starting address of an original basic block in the target binary to its corresponding `bb_entry` structure, which contains metadata and its location in the code cache.

---
### `extern bb_entry *bb_pool;`
A pointer to a preallocated pool of `bb_entry` structures. This is used to avoid frequent `malloc` calls when new basic blocks are encountered.

---
### `extern int bb_pool_index;`
An index into the `bb_pool`, indicating the next available `bb_entry` structure in the pool.

---
## Structs

### `bb_entry`
Represents an entry in the `basic_blocks_map` hash table, storing metadata for a basic block that has been processed by R-Visor.
```c
typedef struct {
    uint64_t addr;             // Key: start address of the BB in the original binary
    rvisor_basic_block metadata; // The rvisor_basic_block structure containing detailed metadata
    UT_hash_handle hh;         // Handle for uthash integration
} bb_entry;
```

* **`addr`**: The starting memory address of the basic block in the target binary. This serves as the key for the hash table.
* **`metadata`**: An `rvisor_basic_block` structure (defined in `railBasicBlock.h`) holding detailed information about the basic block, such as its original and cached addresses, instruction count, type, etc.
* **`hh`**: A handle provided by the `uthash` library to make this structure hashable.

---
### `bb_exit_address`
Represents an entry in the `bb_exit_map` hash table, used for mapping original basic block exit points (or jump targets) to their translated locations in the code cache, facilitating trace linking.
```c
typedef struct {
    uint64_t bin_addr;   // Key: Original binary address of the exit point or target
    uint64_t cache_addr; // Value: Corresponding address in R-Visor's code cache
    UT_hash_handle hh2;  // Handle for uthash integration (using a different handle name)
} bb_exit_address;
```

* **`bin_addr`**: The address in the original binary (e.g., target of a jump). This is the key.
* **`cache_addr`**: The address in the R-Visor code cache where the code corresponding to `bin_addr` (or the stub leading to it) is located.
* **`hh2`**: A handle for `uthash`.

---
## Functions

### Cache and Pool Initialization

#### `int cache_init()`
Initializes the R-Visor code cache.

* **Returns**: An integer status code (typically 0 for success).
* **Description**: This function likely allocates the main `memory` region for the code cache (using `mmap` or similar, with `CACHE_SIZE`), initializes `rvisor_memory_index`, and may initialize other related data structures like `basic_blocks_map` and `bb_exit_map` to be empty.

---
#### `void allocate_bb_pool()`
Preallocates a pool of `bb_entry` structures.

* **Description**: Allocates a chunk of memory for `BB_BLOCK_SIZE` `bb_entry` structures and assigns it to `bb_pool`. This helps in reducing dynamic memory allocation overhead during runtime when new basic blocks are discovered.

---
### Basic Block and Exit Map Operations

#### `void rvisor_insert_basic_block(uint64_t addr, rvisor_basic_block metadata)`
Inserts a new basic block's metadata into the `basic_blocks_map`.

* **Parameters**:
    * `addr`: The starting address of the basic block in the original binary (used as the key).
    * `metadata`: An `rvisor_basic_block` structure containing the metadata for this basic block.
* **Description**: A `bb_entry` is typically obtained from `bb_pool`, populated with `addr` and `metadata`, and then added to the `basic_blocks_map` hash table.

---
#### `void rvisor_insert_exit(uint64_t bin_addr, uint64_t cache_addr)`
Inserts a mapping from an original binary address (e.g., a jump target) to its corresponding code cache address into the `bb_exit_map`.

* **Parameters**:
    * `bin_addr`: The address in the original binary.
    * `cache_addr`: The corresponding address in the R-Visor code cache.

---
#### `rvisor_basic_block *find_basic_block(uint64_t addr)`
Searches for a basic block in the `basic_blocks_map` using its original starting address.

* **Parameters**:
    * `addr`: The starting address of the basic block in the original binary.
* **Returns**: A pointer to the `rvisor_basic_block` metadata if found, otherwise `NULL`.

---
#### `uint64_t *find_exit(uint64_t addr)`
Searches for a code cache address in the `bb_exit_map` corresponding to an original binary address.

* **Parameters**:
    * `addr`: The address in the original binary.
* **Returns**: A pointer to the `uint64_t` code cache address if found, otherwise `NULL`.

---
#### `void clear_basic_blocks()`
Clears all entries from the `basic_blocks_map` and potentially resets the `bb_pool`.

* **Description**: Used when the code cache is flushed or R-Visor is reinitialized.

---
### Code Cache Allocation and Insertion

#### `int rvisor_allocate_root()`
Likely allocates the very first basic block (the entry point or "root" of execution) into the code cache.

* **Returns**: An integer status code.

---
#### `int insert_stub_region(int override)`
(Conditional: `#ifdef STUBREGIONS`) Inserts a stub region into the code cache. Stub regions are shared code sequences used for context switching back to R-Visor or linking basic blocks.

* **Parameters**:
    * `override`: An integer flag, possibly to force creation or overwrite an existing region.
* **Returns**: An integer status code.

---
#### `void rvisor_insert_inst16(uint16_t newInst)`
Inserts a 16-bit instruction into the code cache at the current `rvisor_memory_index`.

* **Parameters**:
    * `newInst`: The 16-bit instruction to insert.
* **Description**: `rvisor_memory_index` is advanced by 2 bytes.

---
#### `void rvisor_insert_inst32(uint32_t newInst)`
Inserts a 32-bit instruction into the code cache at the current `rvisor_memory_index`.

* **Parameters**:
    * `newInst`: The 32-bit instruction to insert.
* **Description**: `rvisor_memory_index` is advanced by 4 bytes.

---
#### `int rvisor_allocate_bb(uint64_t binary_address)`
Allocates a single basic block starting at `binary_address` from the target binary into the R-Visor code cache.

* **Parameters**:
    * `binary_address`: The starting address of the basic block in the original binary.
* **Returns**: An integer status code, likely indicating the end of the allocated block in the cache or an error.
* **Description**: This is a core function that reads instructions from the binary, decodes them until a control-flow instruction is found (or another block termination condition is met), translates/instruments them as needed, and writes them to the code cache using `rvisor_insert_inst16`/`rvisor_insert_inst32`. It also populates and inserts a `bb_entry` into the map.

---
#### `int rvisor_allocate_trace(uint64_t binary_address, int count)`
Allocates a sequence of basic blocks (a "trace") into the code cache, starting from `binary_address`.

* **Parameters**:
    * `binary_address`: The starting address of the trace in the original binary.
    * `count`: A parameter possibly influencing the length of the trace or for recursion depth control.
* **Returns**: An integer status code or the address of the end of the trace in the cache.
* **Description**: This function repeatedly calls `rvisor_allocate_bb` or a similar mechanism to build a longer sequence of executable code in the cache, potentially linking blocks together.

---
#### `int rvisor_allocate_trace_block(uint64_t binaryAddress, rvisor_basic_block railBB, int count)`
A more specific function for allocating a block as part of a trace, possibly using already partially processed basic block metadata.

* **Parameters**:
    * `binaryAddress`: The starting address of the block in the original binary.
    * `railBB`: An `rvisor_basic_block` structure, possibly pre-filled or to be filled with metadata about the block being allocated.
    * `count`: Similar to `rvisor_allocate_trace`, possibly for trace length or recursion control.
* **Returns**: An integer status code.

<br/><br/>