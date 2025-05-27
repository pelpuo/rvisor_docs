# R-Visor

**R-Visor** is an extensible modular dynamic binary instrumentation (DBI) tool designed for open Instruction Set Architecture (ISA) ecosystems, facilitating architectural research and development.

Open ISAs such RISC-V are being widely adopted as the lack of a licensing requirement provides extensibility; Users can freely develop custom instructions and extensions. However, traditional instrumentation tools, built for closed ISAs, lack the adaptability to easily support new extensions. Therefore, a key requirement for modern instrumentation tools is easy extensibility.

R-Visor addresses this challenge by leveraging the *ArchVisor* DSL, which allows users to readily incorporate new ISA extensions. Additionally, R-Visor provides a modular, highly extensible platform with a user-friendly instrumentation API.   

## Key Advantages of R-Visor

R-Visor is built from the ground up with the unique challenges and opportunities of open ISAs in mind:

  * **Extensibility**: Leveraging **ARCH-VISOR**, a novel Domain-Specific Language (DSL), R-Visor allows users to effortlessly integrate new ISA extensions. This reduces the effort and code required to support new or custom instructions compared to traditional DBI tools.
  * **High-Performance, Low Overhead**: R-Visor employs a cache-based Just-In-Time (JIT) execution model, optimized to minimize performance and memory overheads. This ensures that instrumentation is both powerful and efficient.
  * **Flexible and User-Friendly Instrumentation**: R-Visor provides a rich Application Programming Interface (API), that supports diverse instrumentation capabilities at multiple granularities including module, instruction, instruction type, and basic block. This allows for fine-grained control and simplifies the creation of complex analysis routines.

-----

## Support

R-Visor provides initial support for the RISC-V architecture, specifically `rv64gc`. It supports the instrumentation of static 64-bit binaries, compiled with the newlib toolchain (`riscv64-unknown-elf-gcc`). 


-----

## Quick Links

- [Installation](installation)
- [Tutorials](tutorials/basic_example)
- [Architecture](design/architecture)
- [API Reference](API/r_visor)
<!-- - [Contributing Guide](contributing.md) -->
