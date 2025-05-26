# Elf Reader
This section provides an API reference for the ELF file reader interface defined in R-Visor. This module is responsible for parsing Executable and Linkable Format (ELF) files, extracting relevant information such as headers, sections, and symbols, and providing an interface to access instruction data.

---
## Structs

### `funcBounds`
Represents the boundaries (start and end addresses) of a function.
```c
typedef struct {
    uint32_t startAddr;
    uint32_t endAddr;
    char* funcName;
} funcBounds;
```
* **`startAddr`**: The starting memory address of the function.
* **`endAddr`**: The ending memory address of the function.
* **`funcName`**: A pointer to a string containing the name of the function.

---
### `Elf_Reader`
Holds the state and data extracted from an ELF file.
```c
typedef struct {
    FILE *elf_file;             // File pointer to the opened ELF file
    Elf64_Ehdr elf_header;      // ELF file header
    Elf64_Phdr *program_headers; // Array of program headers
    Elf64_Shdr *sectionHeaders; // Array of section headers
    Elf64_Shdr shstrtabHeader;  // Section header for the section header string table
    char *shstrtab;             // Section header string table (contains names of sections)
    char *text_section;         // Buffer containing the .text section (executable code)
    int text_section_offset;    // Starting offset/address of the .text section in memory
    int text_section_size;      // Size of the .text section in bytes
    int program_counter;        // R-Visor's internal program counter for iterating through instructions
    int pc_increment;           // Amount to increment the program_counter by (usually instruction size)
} Elf_Reader;
```

---
## Global Variables

### `extern Elf_Reader rvisor_elf_reader;`
A global instance of the `Elf_Reader` struct, presumably used by R-Visor to manage the currently loaded ELF file.

---
## Functions

### Initialization and Cleanup

#### `int rvisor_init_rvisor_elf_reader(const char *filename)`
Initializes the global `rvisor_elf_reader` instance by opening and parsing the specified ELF file.

* **Parameters**:
    * `filename`: Path to the ELF file.
* **Returns**: An integer indicating success (usually 0) or an error code.
* **Description**: This function reads the ELF header, program headers, section headers, and section header string table. It prepares the `rvisor_elf_reader` for other operations.

---
#### `void free_rvisor_elf_reader(Elf_Reader *reader)`
Frees dynamically allocated memory within an `Elf_Reader` instance.

* **Parameters**:
    * `reader`: A pointer to the `Elf_Reader` struct whose resources are to be freed.
* **Description**: This should be called when ELF file processing is complete to prevent memory leaks, closing the file and freeing allocated buffers like `program_headers`, `sectionHeaders`, `shstrtab`, and `text_section`.

---
### Printing and Display

#### `void printHeaders()`
Prints all section headers information from the currently loaded ELF file.

* **Description**: Iterates through `rvisor_elf_reader.sectionHeaders` and displays details for each section. The output format is not specified but typically includes section name, type, address, size, etc.

---
#### `void printSectionNames()`
Prints the names of all sections in the currently loaded ELF file.

* **Description**: Uses the section header string table (`shstrtab`) to display the names of all sections.

---
#### `void printSection(const char *sectionName)`
Prints the content (e.g., as a hex dump) of a specific section from the ELF binary.

* **Parameters**:
    * `sectionName`: The name of the section whose content is to be printed.

---
#### `void print_sections(FILE* outfile, Elf64_Shdr* sectionHeaders, size_t section_count, const char* shstrtab)`
A utility function to print information about ELF sections to a specified output file.

* **Parameters**:
    * `outfile`: The `FILE` stream to print to.
    * `sectionHeaders`: A pointer to an array of section headers.
    * `section_count`: The number of section headers in the array.
    * `shstrtab`: A pointer to the section header string table.

---
### Program Counter (PC) and Instruction Navigation

#### `int jumpToMain()`
Sets R-Visor's internal program counter to the address of the `main` function.

* **Returns**: An integer, likely the address of `main` or a status code.
* **Description**: This involves looking up the "main" symbol in the symbol table and updating `rvisor_elf_reader.program_counter`.

---
#### `int jumpToEntry()`
Sets R-Visor's internal program counter to the entry point address specified in the ELF header.

* **Returns**: An integer, likely the entry point address or a status code.
* **Description**: Updates `rvisor_elf_reader.program_counter` to `rvisor_elf_reader.elf_header.e_entry`.

---
#### `int getProgramCounter()`
Returns the current value of R-Visor's internal program counter.

* **Returns**: The current program counter value.

---
#### `void setProgramCounter(int newPC)`
Sets R-Visor's internal program counter to an arbitrary address.

* **Parameters**:
    * `newPC`: The new value for the program counter.

---
#### `uint32_t rvisor_get_next_instruction()`
Retrieves the instruction at the current internal program counter from the loaded `.text` section.

* **Returns**: The 32-bit instruction word.
* **Description**: This function reads from `rvisor_elf_reader.text_section` based on `rvisor_elf_reader.program_counter` and `rvisor_elf_reader.text_section_offset`. It also likely updates `rvisor_elf_reader.pc_increment` based on whether the fetched instruction is compressed or not, though this detail is internal. The commented-out older signature suggests more explicit parameter passing, while the current one relies on the global `rvisor_elf_reader` state.

---
#### `void incProgramCounter()`
Increments R-Visor's internal program counter.

* **Description**: The amount of increment is determined by `rvisor_elf_reader.pc_increment` (which is set by `rvisor_get_next_instruction` based on instruction size, e.g., 2 for compressed, 4 for standard).

---
#### `void decProgramCounter()`
Decrements R-Visor's internal program counter.

* **Description**: This likely decrements by a fixed amount (e.g., 4 or a previously recorded `pc_increment`), but the exact behavior might depend on how R-Visor handles instruction sizes for decrements.

---
### Section Data Retrieval

#### `int rvisor_get_data_sections(char *buffer, int *bound)`
Loads all non-executable data sections (e.g., `.data`, `.rodata`, `.bss`) from the ELF file into the provided `buffer`.

* **Parameters**:
    * `buffer`: A character buffer where the concatenated data sections will be stored.
    * `bound`: A pointer to an integer that will be updated with the total size of data loaded into the buffer.
* **Returns**: An integer status code.
* **Description**: This function iterates through the section headers, identifies data sections, reads their content from the ELF file, and appends them to the `buffer`. The commented-out older signature suggests it previously took more explicit ELF details as arguments.

---
#### `void rvisor_get_text_section()`
Loads the `.text` section (executable instructions) from the ELF file into `rvisor_elf_reader.text_section`.

* **Description**: It also populates `rvisor_elf_reader.text_section_offset` and `rvisor_elf_reader.text_section_size`.

---
#### `int rvisor_get_text_sectionOffset()`
Returns the starting memory address (or offset) of the `.text` section.

* **Returns**: The offset/address of the text section.

---
### Symbol and Function Information

#### `int getSymbols(const char *symtabName)`
Retrieves and potentially prints or processes symbols from a specified symbol table section (e.g., ".symtab", ".dynsym").

* **Parameters**:
    * `symtabName`: The name of the symbol table section to process.
* **Returns**: An integer status code.

---
#### `const char* getFunctionName(int addr)`
Retrieves the name of a function from the symbol table that contains the given address.

* **Parameters**:
    * `addr`: The memory address to look up.
* **Returns**: A pointer to a string containing the function's name if found, otherwise `NULL` or an empty string.

---
#### `Elf64_Sym getSymbol(const char *symName)`
Retrieves the `Elf64_Sym` structure for a symbol given its name.

* **Parameters**:
    * `symName`: The name of the symbol to retrieve.
* **Returns**: An `Elf64_Sym` structure. If the symbol is not found, the contents of the returned structure might be invalid or zeroed.

---
#### `void getFunctions()`
Likely processes the symbol table (e.g., ".symtab") to identify all function symbols and might populate an internal list or print them.

* **Description**: The exact behavior (e.g., populating `funcBounds` structures) is not detailed by the signature alone but is implied by the `funcBounds` struct.

<br/><br/>