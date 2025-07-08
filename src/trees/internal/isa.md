# Instruction Set Architecture

A zkVM’s **Instruction Set Architecture (ISA)** defines the fundamental "language" understood by the virtual machine — that is, the set of basic operations (instructions) a program can use to manipulate data, control flow, and perform computation inside the zkVM.

In traditional processors, the ISA is designed to run as fast and efficiently as possible in silicon chips, minimizing energy and latency. By contrast, in a zkVM, the ISA must be optimized for **succinctness** and **prover performance**: the fewer and simpler the instructions, the fewer polynomial constraints and columns are needed to generate a SNARK proof, reducing prover cost.

At a high level, you can think of an ISA like a minimal "vocabulary" of actions a program can use. For example, in a simple ISA, a few core instructions might be:
* `ADD x, y`: Add values in registers `x` and `y`.
* `JMP addr`: Jump to instruction at address `addr`.
* `LOAD x, [addr]`: Load a value from memory into register `x`.

A program is simply a sequence of these instructions. In a zkVM context, each instruction corresponds to certain algebraic constraints that must be satisfied in a proof, so the choice and design of instructions directly impact efficiency.

Choosing an ISA for a zkVM involves delicate trade-offs between prover efficiency, compatibility with existing compilers and developer tools, and ease of writing or understanding programs.

In this section, we explore the main approaches under consideration: general-purpose ISAs such as RISC-V and MIPS, and zk-optimized ISAs designed specifically to minimize proof costs.

## RISC-V: General-Purpose Simplicity and Dominance

RISC-V is an open, modular, and minimal instruction set originally designed for conventional hardware processors. Its clean design, fixed-length instructions, and straightforward register model — with as few as 47 core instructions in its minimal base integer set (RV32I) — make it easy to implement, understand, and extend.

### Strong Tooling and Developer Familiarity

For zkVMs, RISC-V offers a strong starting point thanks to its mature and robust tooling ecosystem. Developers can write high-level programs in Rust, C, or other mainstream languages and compile them to RISC-V almost seamlessly using LLVM backends.

This broad compatibility lowers barriers to adoption and allows teams to leverage existing debugging, profiling, and other tools. It also enables rapid prototyping and smoother transition from prototype to production. In practice, this means zkVM teams can focus more on optimizing their proving systems rather than reinventing basic compilation and developer workflows.

### Challenges for SNARKs

Despite these advantages, RISC-V was not designed with SNARKs in mind. In a traditional silicon processor, registers allow fast local data access and reduce expensive memory operations. But in a zkVM, every register directly increases prover cost and proof size.

Moreover, dynamic control flow mechanisms such as conditional jumps and branches are highly efficient on hardware but become cumbersome in a zk context. Circuits must encode all possible branches and conditions to ensure correctness, which bloats the constraint system and slows down proof generation.

### The Dominant Choice Among zkVMs

Despite its inefficiencies in proof generation, RISC-V has become the dominant architecture among zkVM implementations today. A large part of this adoption is due to the immediate benefits of developer familiarity and mature infrastructure. It also reflects a pragmatic approach: by prioritizing a widely supported ISA, zkVM projects can attract more contributors, integrate mature audit and testing pipelines, and build momentum toward mainnet deployment.

Most notably, nearly all zkVMs that aim to prove Ethereum mainnet blocks in real-time today are RISC-V based. This convergence shows that, for now, the trade-off heavily favors ecosystem reuse and standardization over theoretical proof efficiency.

## MIPS: A Classic and Streamlined Alternative

MIPS is a classic RISC-style instruction set architecture, known for its simplicity and widespread use in academia and embedded systems. Even before the emergence of zkVMs, MIPS was appreciated for its minimal design and ease of analysis, which made it a frequent choice for teaching computer architecture.

### Why MIPS for zkVMs?

In the context of SNARKs, MIPS offers a few appealing properties. Its extremely straightforward design and reduced instruction set can lower execution complexity compared to more feature-rich ISAs like RISC-V. Fewer instructions and simpler decoding logic mean that the corresponding zk circuits may have fewer constraints and slightly smaller execution traces.

For example, projects like **zkMIPS** (by ZKM) have chosen MIPS specifically to capitalize on these simplifications. By using MIPS, they aim to minimize the number of cycles needed and reduce overhead during proof generation, all while preserving a strong developer experience.

Moreover, MIPS has a long-standing, mature ecosystem and has been the basis for many teaching tools and simulators. For example, a program written in Rust, Go, or other high-level languages can be compiled into MIPS machine code. This history provides a foundation of reliable tooling and makes it easier to port or compile programs.

### Limitations and Trade-offs

While MIPS offers a streamlined and simpler architecture than RISC-V, it still inherits a key limitation shared by most general-purpose ISAs: it was not designed with SNARKs in mind.

Even with its reduced instruction set, MIPS programs involve registers and explicit low-level control flow, which can lead to larger execution traces in a zk proof system. This ultimately results in higher prover costs and longer proof generation times than purpose-built zk-optimized ISAs.

Nonetheless, again, for many teams, these costs are offset by MIPS’s simplicity, mature tooling, and decades of accumulated knowledge. The trade-off often feels acceptable because it allows developers to leverage familiar compiler pipelines and focus on building applications rather than deeply optimizing circuits from scratch.

## Cairo: A zk-Optimized ISA

[Cairo](https://eprint.iacr.org/2021/1063.pdf), developed by StarkWare, represents a radical departure from conventional ISAs. It was not adapted for proofs but designed from the ground up as a **SNARK-friendly CPU architecture**. The goal was to create an instruction set that is inherently efficient to prove, which led to a design that re-imagines how instructions interact with memory, registers, and computation itself.

### An 'Algebraic RISC' Instruction Set

At its core, Cairo's ISA is an **Algebraic RISC** (Reduced Instruction Set Computer). This means its instruction set is minimal and its native data type is not a 32 or 64-bit integer, but an element in a finite field. The core instructions perform simple field arithmetic, like addition and multiplication. This "algebraic" nature is the key to its efficiency, as these instructions map directly and cheaply to the polynomial constraints of a SNARK proof.

### Memory-by-Assertion and Instruction Format

The Cairo ISA fundamentally changes how instructions handle memory. It lacks traditional `load` and `store` opcodes for a read-write memory. Instead, its primary mode of memory interaction is a form of assertion. An instruction like `[ap] = [fp - 3] + [fp - 4]` is not just a computation; it is an *assertion* that the value in memory at the address `ap` must equal the sum of the values at the other two addresses. The memory itself is "non-deterministic," meaning its entire state is provided by the prover. The ISA's job is simply to enforce the consistency of these assertions.

This design is reflected in the instruction format. Cairo instructions don't operate on general-purpose registers. Instead, instruction opcodes are encoded with offsets relative to two special-purpose *address registers*: the allocation pointer (`ap`) and the frame pointer (`fp`). This makes the instructions compact and their effect on the state easy to verify.

### Extending the ISA's Power: Hints and Builtins

While the core ISA is minimal, its practical power is amplified by two concepts that sit alongside it:

First, **prover hints** are not part of the formal ISA but are a crucial part of the programming model. They allow a prover to solve complex problems (like a division or a hash) outside the proof system, and then use the simple ISA instructions to efficiently verify the answer. The instruction set is thus focused on cheap *verification*, not expensive *computation*.

Second, **builtins** act as a form of ISA extension through memory-mapped I/O. For common but arithmetically complex operations (like range checks), a program uses standard memory-access instructions to interact with a dedicated memory segment. The builtin, which has its own optimized proof logic, enforces constraints on this segment. In this way, the simple core ISA can "invoke" powerful, pre-defined operations without needing dedicated complex instructions.

### The Trade-off: An Unconventional Instruction Set

The payoff for this unique ISA is substantial: programs can be proven with significantly smaller execution traces and faster proof generation times. However, the trade-off stems directly from its unconventional nature. Developers must learn to program with an instruction set based on field arithmetic and memory-by-assertion. This requires a new way of thinking about computation, presenting a steeper learning curve than ISAs that are simple compilation targets for familiar languages.

## Other zk-Native Custom ISAs: Valida and PetraVM

Recent zkVM research has produced new fully custom Instruction Set Architectures (ISAs) designed specifically for proof efficiency rather than silicon performance. Two strong examples are Valida and PetraVM.

### Valida: Minimal and Register-Free Design

Valida takes a radically simplified approach to ISA design by removing all general-purpose registers. It uses only a minimal set of special-purpose pointers to manage control flow and stack frames. All local variables and intermediate data are managed on the stack or in memory.

This design choice dramatically reduces the number of columns in the execution trace and simplifies constraint systems, directly improving prover performance. Additionally, Valida introduces specialized instruction formats that combine related operations to reduce branching overhead and further minimize circuit complexity.

A key feature is its tight integration with high-level languages via a dedicated LLVM backend, allowing developers to write in familiar languages like C and compile directly to Valida’s streamlined, zk-optimized ISA.

### PetraVM: Binary-Field-Oriented ISA

PetraVM is another custom ISA built specifically to leverage the advantages of binary-field SNARKs. Instead of using traditional register files, PetraVM's ISA focuses on minimizing state and simplifying control flow at the instruction level.

Its instruction set is designed to be minimal and highly composable, using static function frames and strictly defined pointer arithmetic to reduce the complexity of branching and data manipulation. PetraVM supports a flexible and extensible instruction set, which can be expanded with additional operations without increasing baseline proof costs when unused.

This focus on a clean, minimal instruction set allows PetraVM to efficiently support advanced features like recursion and field arithmetic while keeping the prover workload low.

## Hybrid and Intermediate Approaches: WASM and eBPF

Beyond purely general-purpose or purely custom ISAs, hybrid approaches also exist. One prominent example is WebAssembly (WASM). While WASM is not an ISA in the strict sense, it serves as an intermediate representation that many modern languages (including Rust and Go) can compile to. WASM features a formally specified stack model and predictable control flow, making it a convenient high-level target.

In zkVM design, WASM can serve as an initial compilation target, which is then further transformed into a zk-optimized backend representation. This approach combines the robust tooling and developer familiarity of WASM with the performance advantages of custom lower-level circuits. Some projects explore further introducing specialized stack or memory models (like Write Once Memory) during this transpilation step to optimize the final zk circuit.

eBPF, originally designed for high-speed sandboxed execution in environments like Linux and Solana, can be also considered. Its simplicity and efficiency make it attractive for certain use cases, but it has not yet been deeply explored in zk contexts, and its ultimate potential remains open for future research.

## Balancing Efficiency and Compatibility

Choosing an ISA for a zkVM requires navigating a fundamental tension: optimizing for prover efficiency versus preserving compatibility with existing developer ecosystems and tools.

General-purpose ISAs like RISC-V and MIPS offer immediate integration with existing compilers and broad language support but come at a cost in proof size and generation time. In contrast, zk-optimized ISAs like Cairo, Valida, and PetraVM can achieve drastic improvements in proof performance but require new compilers, new developer education, and often a rethinking of programming paradigms.

Today, **RISC-V has emerged as the dominant choice among zkVMs**, not only because of its strong ecosystem and tooling maturity, but also due to recent advances that have dramatically improved its proving efficiency. Several zkVM projects have successfully achieved real-time proving of Ethereum mainnet blocks using RISC-V, demonstrating that its original proof inefficiencies can be effectively mitigated through careful circuit design and engineering optimizations.

Instruction Set Architecture thus shapes the core trade-offs in zkVM design. It dictates not only how programs are written and executed but also how efficiently and succinctly they can be proven.
