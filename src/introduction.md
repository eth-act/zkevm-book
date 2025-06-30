# Introduction
Ethereum’s vision is to be a *secure* and *scalable* decentralized backbone of the modern economy.
One major step toward achieving this vision is the integration of zkEVMs into the Ethereum protocol.

In this book, we explore this idea in depth - what zkEVMs are, how they can be applied to Ethereum, and how theoretical advances in cryptography, combined with significant engineering effort, make zkEVMs a reality.

## Slots, Re-Execution and Gas
One of Ethereum's core functionalities is maintaining *consensus* on a shared state in a decentralized way.
To achieve this, Ethereum operates in discrete time intervals called *slots*, during which the blockchain’s state can be updated at most once.
In each slot, participants in the Ethereum protocol work to agree on a state transition, which is represented by a *block*.
Each block contains, among other things, a list of *transactions*.
Here, a transaction isn't just a monetary transfer - it can be any programmatic action, such as calling a smart contract function that executes code and potentially alters the shared blockchain state.
Intuitively, the more transactions included in a single block, the lower the amortized overhead imposed by the consensus algorithm per transaction.

A simplified version of what happens in a slot is as follows:
1. A *proposer* is selected and distributes a signed block of transactions over the gossip network.
2. Other participants, called *validators*, independently *re-execute* the transactions in the block to verify its correctness. If the block is valid, they generate and distribute attestations.
3. Once enough attestations are collected, the block is considered confirmed or finalized by consensus (ignoring many details here).

To ensure that this verification process completes in a timely manner, Ethereum imposes limits on how much computation can occur per block.
This is where the concept of *gas* becomes essential.

Gas is a unit that measures the amount of computational effort required to execute a transaction or smart contract on Ethereum.
Each operation has an associated gas cost.
When you submit a transaction, you specify a *gas limit* (the maximum you're willing to spend) and a *gas fee* (what you're willing to pay per unit of gas).
If the transaction runs out of gas, it fails, but still consumes all the gas spent up to that point.

More importantly, there is also a *block gas limit*, which caps the total gas that can be consumed by all transactions in a block.
This upper bound is essential to ensure that the validator's re-execution can be completed in time.
It also effectively limits Ethereum’s throughput: only a certain amount of computation can happen per block.

## Scaling Naively
One might think that increasing the block gas limit would immediately solve Ethereum’s scalability issues: more gas per block means more transactions, which means higher throughput.
However, recall that all validators must re-execute the block to verify its correctness within the fixed duration of a slot.
Raising the gas limit naively increases the computational load on validators.
One way to address this is by requiring validators to have more powerful hardware.
However, this approach has limits: if hardware demands become too high, it raises the barrier to entry for validators, undermining decentralization.

Thus, a fundamental bottleneck in Ethereum’s scalability is that *every validator must re-execute every transaction*.

## zkEVMs to the Rescue
Fortunately, zkEVMs offer a powerful solution to this bottleneck.
They rely on a cryptographic tool known as a *succinct non-interactive argument of knowledge (SNARK)*.
The key idea is simple: instead of requiring all validators to re-execute every transaction, we allow a single party, typically the proposer or a specialized prover, to execute the block and generate a *short cryptographic proof* that shows the correctness of this execution.
This proof is then included with the block.
Importantly, verifying this proof is much cheaper than re-executing the entire block.
By verifying the proof instead of re-executing every transaction, validators dramatically reduce their computational and hardware demands. This allows Ethereum to safely raise the gas limit — and thus increase throughput — without compromising decentralization or validator accessibility.


## Structure of this Book
This book is structured into four parts, two of which form the conceptual core:
- Part I explains how a zkEVM can be used Ethereum. It explains what kind of problems it solves and how. In particular, here we treat the zkEVM as a magical black box with a specific interface and desired efficiency and security properties.
- Part II explains the cryptographic principles underlying the construction of most zkEVMs. While different constructions of zkEVMs differ in details, most of them follow a common framework, which we outline in this part.
- Part III contains sections that are tailored to specific audiences, e.g., Ethereum client developers, or readers interested in benchmarking.
- Part IV tracks the current status of the zkEVM effort and is meant to be updated regularly.
