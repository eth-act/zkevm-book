# Problems Addressed by zkEVMs

Ethereum is one of the most widely used decentralized platforms, but it faces several key challenges in scaling while preserving security and decentralization. zkEVMs leverage succinct proofs of correct execution to tackle these bottlenecks. In this section, we list the core problems in today’s protocol, provide a brief explanation of each, and outline how zkEVMs address them. Later chapters will explore each problem in depth.

## Limited Throughput and Rising Costs

### The Problem

Ethereum’s base layer (L1) processes only about 15–20 transactions per second, constrained by the **block gas limit**, a cap on total resources consumption per block. Under this limit, validators must re-run every transaction to confirm correctness. Raising the gas limit naively would force all validators to execute more work in the same slot time, requiring more powerful (and expensive) hardware and risking centralization as only large operators can keep pace.

### How zkEVMs help

A single **builder** executes the entire block—performing all contract calls and state updates, and passes the resulting execution trace to a **prover**, which generates a succinct proof of correctness without needing full chain state. Validators then perform only a **proof verification** step, which is orders of magnitude cheaper than full re-execution. This shift allows Ethereum to increase the gas limit (and thus throughput) without raising validator hardware requirements or compromising decentralization.

## Risk of Validator Centralization

### The Problem

As block complexity and gas limits increase, only entities with high-end machines can keep up with full re-execution. This hardware barrier shrinks the pool of potential validators, undermining the network’s decentralized security model.

### How zkEVMs help

By offloading heavy computation to provers, zkEVMs reduce validator requirements to lightweight proof checking. Even a modest device (some researchers suggest a **Raspberry Pi**) could verify proofs. By drastically lowering the hardware cost barrier, zkEVMs preserve a broad validator set alongside existing requirements (e.g., 32 ETH stake), ensuring that no single group dominates consensus.

## Finality Delays and Missed Slots

### The Problem

In the classical model, validators must re-execute each block before attesting. If a validator cannot complete re-execution in time, they simply miss the opportunity to submit a vote (resulting in a missed attestation) rather than delaying finality. Ethereum reaches economic finality once at least 66% of validators (directly or via child blocks) have attested.

### How zkEVMs help

The **succinctness** property ensures that proof verification stays fast and constant-time regardless of block complexity. While the prover still needs time to generate the proof (within the existing slot duration) validators no longer re-execute and can attest as soon as the proof is published. Finality remains tied to the fixed slot length (e.g., 12 s), but zkEVMs eliminate re-execution overhead, reducing missed votes and making confirmation windows more predictable—especially if slot times are later optimized.

## Gas Price Volatility and Accessibility

### The Problem

Limited block capacity drives competition for inclusion, causing gas fees to spike under heavy demand and pricing out some users. Even with EIP-1559’s base-fee mechanism smoothing sudden jumps, sustained high demand can still raise fees enough to deter participation.

### How zkEVMs help

With higher safe gas limits, zkEVMs expand block capacity and spread demand over a larger supply of inclusion slots. Lower contention means users need not overbid each other, stabilizing and reducing gas prices. The network becomes more affordable for everyday use, without sacrificing security.

> **Remark:** Lower gas costs may attract even more users and applications, potentially bringing demand back towards capacity limits. zkEVMs could address this by pairing higher throughput with ongoing protocol upgrades (such as parallel mini-block execution and sharding) ensuring that capacity scales with adoption.

## Threat of Prover Killer Blocks

Blocks containing extremely heavy or pathological workloads, so-called **prover killers**, can overwhelm provers, risking a stall in block finalization (liveness failure).

> **What makes a block a “prover killer”?** Blocks containing operations whose on-chain gas costs don’t account for their proving complexity, i.e. cheap opcodes or interactions that incur heavy work at proof time but remain underpriced in gas.

Even the heaviest blocks are reduced to succinct proofs, shielding validators from worst-case workloads. Moreover, assigning proving responsibility to block builders aligns incentives: builders must generate proofs themselves, so they avoid constructing blocks that are too costly to prove. This mechanism preserves liveness and prevents network stalls.

## Big Picture: A Unified Execution-Validation Architecture

By cleanly separating execution (proving) from validation (proof checking), zkEVMs transform Ethereum’s scalability model:

* **Competitive execution market:** Encourages specialized prover implementations to optimize speed and cost.
* **Massive validator diversity:** Empowers lightweight nodes to secure the chain, strengthening decentralization.
* **Safe, elastic throughput:** Allows base-layer capacity to grow without undermining security or validator accessibility.
