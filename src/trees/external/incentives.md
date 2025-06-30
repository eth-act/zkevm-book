# Incentives

Robust incentive structures are necessary to ensure zkEVMs can be implemented on Ethereum. In this section, we highlight the three main incentive problems that need to be addressed.

### ❗ Bottlenecks

Ethereum cannot adopt zkEVMs without addressing the following issues. These issues are high priority. Colour indicates how well-understood the problems are going form least understood, 🔴, to solved, 🟢. 🟠 indicates research and implementation work still needs to be done, and 🟡 indicates research is done while implementation still needs to happen.
- 🔴 **Fallback Provers.** A system that allows provers to connect with builders when existing builders and/or provers go offline.
- 🟠 **Prover Killers.** Blocks that are excessively hard to prove, prover killers, should not cause missed slots.

### ✅ Improvements

To improve the robustness of Ethereum, the following issues should be addressed:

- 🔴 **Prover Markets.** An in-protocol market that allows builders to reliably and trustlessly buy proofs for every block.

## Fallback Provers
There has to be only one prover willing and able to prove a block at all times. That is, proving is a 1-out-of-N trust assumption. The goal of the “Fallback Provers” project is to ensure this 1-out-of-N trust assumption is credible. Credibility is obtained by:

1. Ensuring the required costs of proving are low enough such that many people could run a prover.
2. Ensuring provers can be spun up quickly, whether locally, in a distributed setting, or with rented GPUs. 
3. Ensuring new provers can be found by block builders.

This project is concerned with the third point and aims to build a robust in-protocol system to facilitate new provers and builders to communicate with each other, sometimes called multiplexing.

## Prover Killers
Prover killers are blocks that are excessively hard to prove due to the large number and/or types of transactions in the block. If blocks cannot be proven, but proofs are required to validate blocks, Ethereum cannot process transactions: it loses liveness.

This project is concerned with preventing liveness issues due to prover killers. The two ways currently explored to do so are:

1. Assigning responsibility of proving a block to the builder of the block. The builder is then not incentivized to create a prover killer. (See the [“Prover Killers Killer” post](https://ethresear.ch/t/prover-killers-killer-you-build-it-you-prove-it/22308) for the proposal).
2. Assigning proving gas costs to transactions based on how expensive they are to prove.

Note that these two methods are not mutually exclusive.

## Prover Markets
It may be desirable to incentivize builders to source proofs from an open market of provers. For example, it may decrease barriers to entry to become a prover if it were otherwise hard to find a competitively-priced prover. This project is concerned with the following subquestions:

1. How important is it for Ethereum that there is an active and open market of provers in the normal case (that is, not the fallback case)?
2. What would an active and open market of provers in the normal case look like in-protocol?
