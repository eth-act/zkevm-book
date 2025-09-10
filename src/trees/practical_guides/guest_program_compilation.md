# Guest Program Compilation
Before a program's execution can be proven, the program itself, implemented in any high-level programming language, must be compiled to a form that is understandable by a Zero-Knowledge Virtual Machine. Most current ZKVM implementations do not require a guest program to be compiled to a ZKVM-specific ISA (Instruction Set Architecture), but rather they use already well-known ISAs like [RISC-V](https://en.wikipedia.org/wiki/RISC-V) or [WebAssembly](https://en.wikipedia.org/wiki/WebAssembly). For the purpose of this book, we focus on the former. There are many reasons why the RISC-V ISA works well with ZKVMs, but let us review a couple of the most important ones:

- This ISA is very simple (the basic instruction set is small) and easy to implement in the ZKVM context
- It is well supported by the most popular high-level programming language compilers like `rustc` or `clang`
- TODO: Add more reasons.

We will explain in detail how the Rust to RISC-V compilation pipeline works in the context of ZKVMs.

## What is a guest program?

A guest program can be any program implemented in a high-level programming language that can be compiled to an ISA supported by the ZKVM used to prove the guest program's execution.

Like any program, the guest program can accept input, produce output, allocate memory, read or write data using file descriptors, and perform many other operations. To properly execute the guest program, runtime initialization, standard library initialization, etc., are also required. To limit the scope of possible guest programs discussed here, we assume that the program is implemented in Rust and compiled to the RISC-V ISA.

It is also worth mentioning that most existing ZKVM implementations natively support guest programs implemented in the Rust language, but compiled with their own forked and customized Rust compiler.

## Forked Rust compilers

The stock Rust compiler [supports](https://github.com/rust-lang/rust/tree/master/compiler/rustc_target/src/spec/targets) many RISC-V targets for many different architectures and instruction sets. Nevertheless, almost all ZKVM projects forked the stock Rust compiler and customized it to their own needs. The changes they introduced compared to the stock Rust compiler come down to a couple of ZKVM-specific modifications:

- Definition of ZKVM-specific compilation targets
- Implementation of ZKVM-specific low-level std library functions such as panic handlers, memory allocators, environment variable access, random byte generators, and standard input/output support. These functions are implemented using ZKVM-specific "operating system" ABI functions.

The full list of the changes can be found [here](https://notes.ethereum.org/@ItD_5mfmSqqKX_Fbz31OUQ/Hyau2HbBgx).

However, these changes are not related to the RISC-V target specification. The new ZKVM compilation targets are similar to or the same as already defined targets in the stock Rust compiler. A possible reason why there exist so many Rust compiler forks for each ZKVM is the ability to compile ZKVM-specific standard library and thanks to this be able to compile any arbitrary program using the `std` library.

## Further differences between ZKVMs

In addition to forked and customized Rust compilers, the ZKVM projects define their own guest program format and the zk system ABI they use. The most important differences are:
- program termination and panicking,
- program I/O interaction
- calling precompiles (keccak, ECC, etc.) and other system calls
- program entrypoint (`_start` function) and runtime initialization
- program binary memory layout

All of these differences make it impossible to compile a program into a binary independently of a specific ZKVM and then prove its execution on any freely chosen ZKVM implementation.

## Benefits of a common binary format

Having a common guest program binary format has a lot of benefits for the ecosystem.

- The user can choose which ZKVM to use to prove the execution without a need to recompile the program each time.
- Potential forked compiler security issue likelihood is limited
- The user can use a stock compiler to build a program whose execution they want to prove.
- The same binary can be tested across all ZKVM implementations and make ZKVM testing more precise. Some bugs can remain undiscovered because recompilation with a buggy stack can hide them.
- ZKVMs will not need to maintain their own compilation stacks. Constant rebasing on the newest stock Rust version will not be needed.
- A standard library can be compiled using one compiler configuration for all the ZKVMs.
- TODO: Add more

## Getting rid of forked compilers

Forking the Rust compiler to customize it for specific ZKVM needs is already an issue when maintaining a codebase that integrates different ZKVM platforms, for example to test or benchmark them properly. Moreover, there are serious concerns raised by potential ZKVM users about whether these forks are properly audited and tested. Only one ZKVM project so far has been able to upstream their changes to the main Rust compiler repository, but still the target they defined is in [Tier 3](https://doc.rust-lang.org/nightly/rustc/platform-support.html#tier-3) of Rust target policy. This means that _the Rust project does not build or test automatically, so they may or may not work. Official builds are not available_.

There is a way for most ZKVM projects to build a compatible guest program using the stock Rust compiler and compilation targets already defined there, which are in Tier 2 of Rust target policy. There are a couple of necessary steps to be done to make it possible:

- A proper compilation target from the stock Rust compiler has to be chosen. It is often `rv32ima` as this instruction set is usually supported by all ZKVMs.
- The guest program cannot use the `std` Rust library. Fortunately, many libraries have a feature which allows building them without `std`.
- Since the program entrypoint in Rust is defined in the `std` library, it has to be defined manually. To do this, a `_start` function needs to be defined in the binary.
- The `_start` function has to properly initialize the `sp (stack pointer)` register value, but also the `gp (global pointer)` register value. The global pointer is used by the linker to apply linker-level (_linker relaxation_) optimization.
- A panic handler and a memory allocator have to be defined. Depending on the specific ZKVM platform, they can be implemented differently.
- The final binary memory layout has to be defined according to the ZKVM platform specification. This is usually defined by a linker script or `rustc` compiler flags.
- If the program uses input and output, both of these handlers need to be defined accordingly.

These are a couple of non-obvious steps to be done to prepare a proper binary for the chosen ZKVM platform, but fortunately detailed information about what exactly should happen in all these steps can be easily found in the ZKVM implementations. A non-trivial example of a guest program computing a stateless validation function in Rust can be found [here](https://notes.ethereum.org/@ItD_5mfmSqqKX_Fbz31OUQ/B1llSnexSge).

## Precompiles

As mentioned earlier, the RISC-V ISA is very convenient for ZKVMs, but to increase computational efficiency of advanced cryptography primitives like keccak, ECC operations, or Poseidon hash functions, ZKVM platforms implement their own dedicated circuits for proving execution of these functions. These dedicated circuits are called precompiles.
Depending on the ZKVM specification, precompiles are called by the guest program via `ecall` instruction or by defining a dedicated custom RISC-V instruction. 

## Summary

The guest program compilation pipeline looks very similar for most ZKVM platforms.
- They support any binary in ELF file format with RISC-V ISA.
- ZKVM guest programs can be compiled with the stock Rust compiler.
- However, every platform defines their own ABI, memory layout, and precompile set, which makes the guest program binaries incompatible between different ZKVMs.

