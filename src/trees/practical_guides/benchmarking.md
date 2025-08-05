# Benchmarking

When considering using zero-knowledge technology at the L1 of Ethereum, it is required to do a thorough evaluation of how it will impact the network properties and requirements. In this chapter, we explore in detail how using zkVMs can be evaluated by doing proper benchmarking to gain an understanding of this front. A similar summary of this chapter can also be found in the [following video](https://youtu.be/D2TpmD62tjQ?t=1537) section.

# Why?

When introducing a new technology or a big protocol change in Ethereum, we need to understand the implications on multiple fronts, for example:

- Does it fit the [desired hardware specs](https://blog.ethereum.org/2025/07/10/realtime-proving) for the node types that will be involved?
- How can actors in the network (e.g. provers) understand how different hardware configurations behave? (e.g. GPU models’ power efficiency).
- What new risks are introduced regarding liveness and security?
- Are there any blockers to using this technology? (i.e. non-obvious protocol changes)

To answer these questions, we can’t take a theoretical approach with napkin math, since usually the answer depends on many engineering dimensions of the solutions, e.g. how optimized are the implementations, unexpected limitations compared to theoretical possibilities, maturity, etc. Said differently, (some) devils might be in the details.

Moreover, there are many practical aspects that we need to consider:

- We’re early in the process of deciding which are the best-performing and mature zkVM implementations — we need a quick way to evaluate all existing and easily introduce new options.
- Each zkVM implementation is moving at a fast pace, so it is mandatory to understand their performance improvements and quickly catch potential regressions.
- Depending on the performance of each zkVM, we can envision trying to pair less optimized EL guest programs with more optimized zkVMs to have a balanced suite of combinations for having the most reliability.

Building a zkVM benchmarking pipeline designed specifically for Ethereum needs is a great tool to achieve our goal in the best way possible.

Note also that proving a block with zkVMs depends on two factors: the zkVM used for execution/proving, and the EL client used as a guest program. While all EL clients implement the same protocol rules, they are programmed in different languages and compiled with different tools thus the final bytecode executed in the same zkVM can have different properties/performance. Being able to benchmark the EL clients dimension is as relevant as the zkVMs dimension.

Finally, this effort wouldn’t only help Ethereum core developers to evaluate and implement any required protocol changes, but also provide concrete guidance to zkVM teams to keep optimizing the most performance-critical cases.

# How?

zkVMs are general constructions to prove generic programs that can be compiled to any ISA supported by them. This means that the Ethereum use-case is a subset of programs that they support proving. We are not interested in doing general zkVM benchmarking, but focusing on Ethereum block validation (i.e. benchmarking a program calculating the Fibonacci sequence isn’t interesting).

zkVM benchmarking for Ethereum means putting at the center fixtures that represent any valid block that can happen in the protocol, which we can separate into two main buckets:

- Average cases: these cases can be represented by mainnet blocks — this can help understand the performance in the average cases.
- Worst cases: these cases cover very rare cases that naturally happen in mainnet, or specifically designed blocks by attackers to affect the chain's liveness. A common term to describe them is *prover killers*.

Understanding both cases is crucial to evaluate potential protocol design changes (e.g. gas cost models or incentives).

Benchmarking different zkVMs can become a hot topic since it directly puts different zkVMs face-to-face. These teams work incredibly hard to provide the best version of their design, so it’s important to be responsible in doing a fair comparison. For this reason, the benchmarking pipeline should be transparent and also easily reproducible.

# Protocol design relationship with zkVMs

At the center of zkVM benchmarking is designing the cases that we should run in the zkVM and EL client combinations — we call this benchmark fixtures. To do a meaningful design of which fixtures are run, we need to understand the full protocol in detail, so as to have enough coverage of relevant ones to benchmark.

Before jumping into deeper details, it is helpful to understand what exactly is being proven by zkVMs for the Ethereum use-case. The guest program execution is an EL client executing a stateless block validation. This means receiving as inputs a block plus a witness with the required Ethereum state, and executing the block to check if the post-state root, gas usage, and other fields are valid (i.e. that the state transition function was correct).

When a block is executed, we can separate the workload into two main buckets:

- Non-EVM execution: executing a block has many tasks that do not depend on the EVM. For example, recovering senders from transactions, processing withdrawals, system contract tasks, etc.
- EVM execution: this covers the bulk of the block execution, which is the execution of transactions that call smart contracts.

This means that we need benchmark fixtures covering relevant workloads for both cases. For example, we need to understand how a block with the biggest amount of transactions possible behaves regarding recovering senders from transaction signatures. Or how a block using all the gas limit, calling the KECCAK opcode works.

Since the EVM execution is usually related to the bulk of the work when executing a block, it can also be helpful to consider the following dimensions:

- Pure computation opcodes: EVM instructions that are pure computation can have different throughput when executing inside the zkVM. For example, `ADD` and `MULMOD` do not have the same cost.
- Storage accessing opcodes: many opcodes main load involve accessing state. Recall that zkVMs prove a stateless execution of a block validation; thus, there’s no real disk access involved in the execution. The relevant part of storage accessing opcodes is computational tasks related to verifying the state in the witness, which usually involves state tree proof verifications. This workload usually involves heavy computations for zkVMs, such as proving hashes.
- Precompiles: [EVM precompiles](https://www.evm.codes/precompiled) are optimized constructions within the EVM to do heavy cryptographic tasks, such as recovering a public key from a signature, or doing elliptic curve operations. Cryptographic-related tasks are usually very heavy operations for zkVMs, so having thorough coverage is mandatory.
- Bytecode access: when a smart contract is executed in the EVM, we not only require executing instructions or precompiles, but also fetch and prove that the actual code being executed is correct. Currently, smart contract [bytecode isn’t chunkified](https://ethresear.ch/t/merkelizing-bytecode-options-tradeoffs/22255), so this opens a potential DoS vector that is very relevant to cover.

At the start of 2025, core developers started working on a big project to include benchmark tests in the [execution-spec-tests](https://github.com/ethereum/execution-spec-tests) repository. Before this effort, this repository contained an incredibly valuable set of correctness tests that had been the basis for the Ethereum zero-downtime reality. Now, [benchmarks were also introduced](https://github.com/ethereum/execution-spec-tests/tree/main/tests/benchmark) that cover scenarios in all dimensions described above. These benchmark fixtures are not only useful for zkVM benchmarking but also for [benchmarking raw execution](https://x.com/ChodoKamil/status/1915408546017018156) in EL clients.

# Tools

The zkEVM team within the EF has been working on a specific set of libraries and tools to support this initiative. The two main ones are [ere](https://github.com/eth-act/ere) and [zkevm-benchmark-workload](https://github.com/eth-act/zkevm-benchmark-workload). The former is a Rust library that provides a single API to run guest programs in many zkVMs. This allows us to abstract many of the complexities of running guest programs. The latter is a tool that uses ere, specifically designed to run Ethereum fixtures for benchmarking, e.g. run EEST benchmarks, mainnet blocks, and experimental programs.

We won’t go into full detail in this chapter on how to use these tools since both have extensive documentation in the repository. Anyone is invited to use these libraries and tools and collaborate on making them better. For example, new zkVMs can integrate into ere and immediately have a complete toolset to benchmark how Ethereum would behave in them.

The strategy used in this style of benchmarking is running real mainnet blocks, specifically crafted to target known potential attacks, or using all the gas limit in a single opcode or precompile, each under a different scenario. For example, `SSTORE` and `SLOAD` can generate different workloads if they access hot and cold data, or the `modexp` precompile underlying implementation can be sensitive to particular inputs.

When the benchmarking fixture is targeting a block that uses all the gas limit for some pure computation opcode (e.g. `ADD`, `MUL`, `MULMOD`, etc), the overhead of pre/post block processing compared to the main block execution should be very low, and is amortized the bigger the gas limit. Note that some opcodes, like `SSTORE`, `SSLOAD`, `EXTCODESIZE` involve accessing state, thus including the pre/post block workload of reading/verifying and writing into the state tree must be considered part of the benchmark run.

It is also possible (and might be desirable for some reasons) to fully isolate some particular opcodes or precompiles by only benchmarking isolated EVM opcodes implementations, but this strategy can end up being too focused on a specific implementation compared to having a shared setup that we can run in all EL EVMs. Also, if we extract the cycles/cost of proving an empty block, we can get close to a very fair take on which opcodes or precompiles can be mispriced or problematic, at least in a relevant order of magnitude. Again, this other strategy shouldn’t be discarded, and both can be complementary.

## Results

The EF zkEVM team is currently running the fixtures and collecting the results. Many parts of the stack are still changing at a fast pace (e.g. guest program optimizations, zkVM SDKs, etc). If you want to track early run results, they are being uploaded to [this repository](https://github.com/eth-act/zkevm-benchmark-runs). Other websites, like Ethproofs, are planning on presenting results about worst cases using the same tooling and benchmark fixtures — so stay tuned.

## External resources

Here is a list of other valuable resources touching on zkVM benchmarking (they may or may not be focused explicitly on Ethereum):

- [Benchmarking zkVMs: Current State and Prospects](https://fenbushicapital.medium.com/benchmarking-zkvms-current-state-and-prospects-ba859b44f560)
- [Ethproofs: Benchmarking the Path to Real-Time ZK Proving](https://www.youtube.com/watch?v=Wf2Xgi6d-I0)
- [Ethproofs Call #3 - zkEVM benchmarks](https://youtu.be/D2TpmD62tjQ?t=1537)
