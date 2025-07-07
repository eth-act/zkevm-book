# Incentives

Robust incentive structures are necessary to ensure zkEVMs can be implemented on Ethereum. In this section, we highlight the main incentive problems that need to be addressed. Some of these problems may also be considered protocol architecture questions whose method of analysis usually benefits from a resource pricing viewpoint which is why they are included in the Incentives section.

## Fallback Provers
At all times, there has to be at least one prover willing and able to prove blocks. That is, proving requires a 1-out-of-N trust assumption. The goal of the “Fallback Provers” project is to ensure this 1-out-of-N trust assumption is credible.  Credibility is obtained by:

1. Ensuring the required costs of proving are low enough such that many people could run a prover.
2. Ensuring provers can be spun up quickly, whether locally, in a distributed setting, or with rented GPUs. 
3. Ensuring new provers can be found by block builders.

This project is concerned with the third point and aims to build a robust in-protocol system to facilitate new provers and builders to communicate with each other, sometimes called multiplexing. The Fallback Provers project is a bottleneck for zkEVM adoption and therefore high priority.

## Prover Killers
Prover killers are blocks that are excessively hard to prove due to the large number and/or types of transactions in the block. If blocks cannot be proven, but proofs are required to validate blocks, Ethereum cannot process transactions: it loses liveness. This project is also a bottleneck for zkEVM adoption, however, the solution space is better understood than that of the fallback provers project.

This project is concerned with preventing liveness issues due to prover killers. The two ways currently explored to do so are:

1. Assigning responsibility of proving a block to the builder of the block. The builder is then not incentivized to create a prover killer. (See the [“Prover Killers Killer” post](https://ethresear.ch/t/prover-killers-killer-you-build-it-you-prove-it/22308) for the proposal).
2. Assigning proving gas costs to transactions based on how expensive they are to prove.

Note that these two methods are not mutually exclusive.

## Prover Markets
It may be desirable to incentivize builders to source proofs from an open market of provers. For example, it may decrease barriers to entry to become a prover if it were otherwise hard to find a competitively-priced prover. This project is concerned with the following subquestions:

1. How important is it for Ethereum that there is an active and open market of provers in the normal case (that is, not the fallback case)?
2. What would an active and open market of provers in the normal case look like in-protocol?

This project is not necessarily a bottleneck for zkEVM adoption. Ethereum could maintain its liveness with just fallback provers without having a competitive prover market, therefore, this project is lower priority.

## Censorship Resistance
zkEVMs leverage the asymmetry in computational power between a sophisticated prover and an unsophisticated validator set. Increased computation needed to build and prove blocks means it may not be possible for every validator to build a block locally. Ethereum may [decouple throughput from local block building](https://ethresear.ch/t/decoupling-throughput-from-local-building/22004), which means it has to source its censorship reistance in another way than local block building. While [FOCIL](https://eips.ethereum.org/EIPS/eip-7805) is a popular proposal to provide censorship resistance for regular transactions, a censorship resistance tool is still necessary for blobs. Francesco has made initial explorations on blob censorship resistance via mempool tickets, see [here](https://notes.ethereum.org/7EGS7DVtTAKnqlh9LDEWxQ?view#Multiple-independent-availability-checks) and [here](https://hackmd.io/@fradamt/blob-mempool-tickets), and a more concrete proposal from Mike and Julian [here](https://ethresear.ch/t/on-the-future-of-the-blob-mempool/22613).

## Offchain or Onchain Proofs
Proofs can live either on- or offchain. Offchain proofs may have benefits such as reduced storage costs and potentially more verifier flexibility. The benefits and drawbacks of on- or offchain proofs must be better understood. Vitalik introduced the concept in [this post](https://notes.ethereum.org/@vbuterin/enshrined_zk_evm), and Justin further explored it [here](https://ethresear.ch/t/native-rollups-superpowers-from-l1-execution/21517).

## Network throughput
zkEVMs remove the need for validators to re-execute transactions to verify a block. Still, some parties may want to re-execute (parts of) blocks, for example, to obtain the current state. There should probably still be a limit to the amount of compute a block can contain, but what should that limit be set to? This projects aims to understand the potential increases in network throughput attributable to zkEVMs.
