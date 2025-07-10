## Real-Time Proving

A fundamental prerequisite for integrating zkEVMs directly into Ethereum’s base layer is **real-time proving**: the ability to generate a succinct validity proof for a block within the same slot in which it is proposed (today, 12 seconds).

### Why Real-Time Proving is Important

In Ethereum today, each validator fully re-executes all transactions in a block before attesting. zkEVMs aim to remove this redundancy by shifting execution to a single prover who produces a succinct proof, while validators verify it cheaply. However, for this paradigm to preserve Ethereum’s liveness and finality guarantees, proofs must be ready **within each slot’s time budget**.

If a proof arrives late — even by a few seconds — validators cannot attest immediately, leading to missed attestations, weaker finality, and increased vulnerability to reorgs. Worse, if slots are missed systematically, it undermines the security assumptions of the beacon chain and the overall Ethereum consensus.

Thus, real-time proving is an **absolute requirement** to replace re-execution at L1 and maintain the same block time and finality cadence.

### Technological Foundations

Achieving real-time proving relies on a combination of cryptographic and engineering advances. At the heart of this effort are special virtual machines called zkVMs, which allow us to run Ethereum’s state transition logic and produce a short proof that the computation was done correctly.

To make proofs fast enough, these zkVMs must be carefully designed to minimize unnecessary work and handle large blocks efficiently. In practice, proving a block requires breaking it into smaller parts that can be processed in parallel and then combined into one final proof. The final proof should also be small enough to be posted onchain.

Hardware also plays an important role: proofs are generated using powerful computers, often with many processors working together. The overall challenge is to ensure that all these steps — running the computation, generating the trace, and assembling the proof — can be completed within the strict time limit of each Ethereum block slot.

### Guarding Against Prover Killers

Real-time proving must also protect against so-called **prover killer blocks**, blocks that are very hard to prove. To prevent this, the protocol can set strict time limits for when proofs must be submitted. This forces block builders to make sure their blocks can be proven quickly, avoiding blocks that might slow down or stop the network.

### Toward Future Slot Times

Ethereum’s long-term roadmap envisions reducing slot times (for example to 6 seconds or even lower), which means proofs will need to be generated even faster. zkEVMs must guarantee that proofs are always ready within these tighter time frames, even for the heaviest blocks — not just on average.

Recent advances show that proving technology is evolving very rapidly. As slot times shorten, proving systems are expected to keep improving in parallel, aiming for proofs in 6 seconds, 3 seconds, or even under 1 second in the future. This makes fast proving a moving target that will continue to push technical progress forward.

### Decentralization Implications

Beyond speed, real-time proving has important consequences for decentralization. If proof systems require massive GPU clusters or expensive specialized hardware, only a few large operators could run them, introducing new centralization risks.

To address this, ongoing work focuses on reducing both hardware and energy costs. The goal is to make provers small and efficient enough to run on modest clusters, and eventually even on consumer-grade machines such as powerful desktops or laptops. In the long term, this could allow individuals, small organizations, or community groups to participate in proving, supporting a more open and decentralized network.
