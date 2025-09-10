# Guest Program Compilation
Before a program's execution can be proven, the program itself, implemented in any high-level programming language, must be compiled to a form that is understandable by a Zero-Knowledge Virtual Machine. Most current zkVM implementations do not require a guest program to be compiled to a zkVM-specific ISA (Instruction Set Architecture), but rather they use already well-known ISAs like [RISC-V](https://en.wikipedia.org/wiki/RISC-V) or [WebAssembly](https://en.wikipedia.org/wiki/WebAssembly). For the purpose of this book, we focus on the former. There are many reasons why the RISC-V ISA works well with zkVMs, but let us review a couple of the most important ones:

- Simplicity: The base RISC-V instruction set is small. A simpler instruction set can lead to a more efficient and verifiable circuit for proving its execution. See the [ISA section](./../internal/isa.md) for the detailed reasons.
- Compiler Support: Mainstream compilers like `rustc` and `clang` provide robust support for RISC-V as a compilation target, allowing developers to use familiar languages and tools.
- Modularity: RISC-V is designed with optional standard extensions (e.g., `M` for multiplication, `A` for atomic operations). A zkVM can be designed to support only the necessary extensions, which helps minimize the complexity of the final circuit.
- Open Standard: As an open-source and royalty-free ISA, RISC-V allows for permissionless innovation and collaboration, which is well-aligned with the open-source nature of many ZK and blockchain projects.

We will explain in detail how the Rust to RISC-V compilation pipeline works in the context of zkVMs.

## What is a guest program?

In the context of zkVMs, the host refers to the zkVM environment that executes the code and generates the proof. The guest program is the application code whose execution is being proven.  It can be any program implemented in a high-level programming language that can be compiled to an ISA supported by the zkVM used to prove the guest program's execution.

Like any program, the guest program can accept input, produce output, allocate memory, read or write data using file descriptors, and perform many other operations. To properly execute the guest program, runtime initialization, standard library initialization, etc., are also required. To limit the scope of possible guest programs discussed here, we assume that the program is implemented in Rust and compiled to the RISC-V ISA.

It is also worth mentioning that most existing zkVM implementations natively support guest programs implemented in the Rust language, but compiled with their own forked and customized Rust compiler.

## Forked Rust compilers

The stock Rust compiler [supports](https://github.com/rust-lang/rust/tree/master/compiler/rustc_target/src/spec/targets) many RISC-V targets for many different architectures and instruction sets. Nevertheless, almost all zkVM projects forked the stock Rust compiler and customized it to their own needs. The changes they introduced compared to the stock Rust compiler come down to a couple of zkVM-specific modifications:

- Definition of zkVM-specific compilation [targets](https://doc.rust-lang.org/nightly/rustc/platform-support.html). This is an architecture specification dedicated for a zkVM. It is very similar to the existing RISC-V related targets. 
- Implementation of zkVM-specific low-level std library functions such as panic handlers, memory allocators, environment variable access, random byte generators, and standard input/output support. These functions are implemented using zkVM-specific "operating system" ABI functions.

The full list of the changes can be found [here](https://notes.ethereum.org/@ItD_5mfmSqqKX_Fbz31OUQ/Hyau2HbBgx).

However, these changes are not related to the RISC-V target specification. The new zkVM compilation targets are similar to or the same as already defined targets in the stock Rust compiler. A possible reason why there exist so many Rust compiler forks for each zkVM is the ability to compile zkVM-specific standard library and thanks to this be able to compile any arbitrary program using the `std` library.

## Further differences between zkVMs

A direct consequence of custom compilers and unique runtime environments is that guest program binaries are generally not portable across different zkVMs. A binary compiled for one zkVM will not execute correctly on another.
This incompatibility arises from several key differences in zkVM design:
- **Program I/O:** The protocol for how the guest program receives input from and sends output to the host.
- **System Calls & Precompiles:** This includes keccak, ECC operations, other system calls and more generally the set of available functions for requesting the host to perform accelerated operations.
- **Entrypoint & Runtime Initialization:** The address of the first instruction (`_start` function) and the required setup procedures before the main program logic begins.
- **Memory Layout:** The specified organization of the program's code, stack, and data segments in memory.
- **Termination & Panicking:** The protocol for signaling successful program termination or a runtime error.

## Benefits of a common binary format

A common binary format, where a single compiled artifact could run on any compliant zkVM, would offer significant benefits to the ecosystem:
- **Portability:** Developers could choose a zkVM based on performance or cost without recompiling their application.
- **Security:** Auditing efforts could be concentrated on a single, standard compiler toolchain rather than being spread across numerous forks.
- **Developer Experience:** Developers could use the official, stable Rust compiler without needing to install and manage custom toolchains.
- **Testing and Reliability:** A standard binary could be tested across all zkVMs, making it easier to isolate bugs in either the guest program or a specific zkVM implementation.
- **Reduced Maintenance:** zkVM projects would not need to dedicate resources to maintaining compiler forks and could focus on their core proving systems.

## Getting rid of forked compilers

Forking the Rust compiler to customize it for specific zkVM needs is already an issue when maintaining a codebase that integrates different zkVM platforms, for example to test or benchmark them properly. Moreover, there are serious concerns raised by potential zkVM users about whether these forks are properly audited and tested. Only one zkVM project so far has been able to upstream their changes to the main Rust compiler repository, but still the target they defined is in [Tier 3](https://doc.rust-lang.org/nightly/rustc/platform-support.html#tier-3) of Rust target policy. This means that _the Rust project does not build or test automatically, so they may or may not work. Official builds are not available_.

There is a way for most zkVM projects to build a compatible guest program using the stock Rust compiler and compilation targets already defined there, which are in Tier 2 of Rust target policy. There are a couple of necessary steps to be done to make it possible:

1. **Select a Standard Target:** Choose a common RISC-V target supported by most ZKVMs, such as `rv32ima` which is usually supported by all zkVMs.
2. **Use `no_std`:** Add the `#![no_std]` attribute to the Rust crate to disable linking the standard library.
3. **Define an Entrypoint:** Manually define the `_start` function, which serves as the program's entrypoint, as this is normally provided by `std`.
4. **Initialize Pointers:** In the `_start` function, explicitly initialize the stack pointer (`sp`) and global pointer (`gp`).
5. **Provide Handlers:** Implement a panic handler and a memory allocator, as these are required by the language but are not available in a `no_std` context.
6. **Define the Memory Layout:** Use a linker script to instruct the linker how to arrange the program's sections (e.g., .text, .data) in the final binary's memory map to match the zkVM's expectations.
7. **Implement I/O:** If required, implement functions for reading and writing data according to the target zkVM's ABI.

These are a couple of non-obvious steps to be done to prepare a proper binary for the chosen zkVM platform, but fortunately detailed information about what exactly should happen in all these steps can be easily found in the zkVM implementations. A non-trivial example of a guest program computing a stateless validation function in Rust can be found [here](https://notes.ethereum.org/@ItD_5mfmSqqKX_Fbz31OUQ/B1llSnexSge).

## Precompiles

While RISC-V is suitable for general-purpose computation, it is inefficient for certain cryptographic primitives like keccak hashes or elliptic curve operations. Executing these operations with base RISC-V instructions would be computationally expensive and result in large, inefficient proofs.
To address this, zkVMs provide precompiles, dedicated circuits optimized for specific, computationally intensive operations. When a guest program needs to perform such an operation, it calls out to the host zkVM, which executes it using the optimized precompile circuit instead of a sequence of RISC-V instructions. 
A guest program typically invokes a precompile via an `ecall` (environment call) instruction or a custom instruction defined by the zkVM. Precompiles are important and widely used for building practical and performant ZK applications.

## Summary

The guest program compilation pipeline looks very similar for most zkVM platforms.
- They support any binary in ELF file format with RISC-V ISA.
- zkVM guest programs can be compiled with the stock Rust compiler.
- However, every platform defines their own ABI, memory layout, and precompile set, which makes the guest program binaries incompatible between different zkVMs.

