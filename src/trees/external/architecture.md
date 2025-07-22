# Architecture
> Note. Section under construction
- Stateless client
- Stateful client
- Two primary patterns have emerged for client integration (to be explored and discussed in more details)
    - Separate clients: Stateless clients (CL+EL) connect to stateful peers over a network subprotocol. Execution and verification are decoupled but remain interactive.
    - Embedded Clients: A single client combines consensus logic (CL) with a lightweight embedded EL verifier (e.g. a zkVM or stateless executor). It receives blocks and proofs via gossip and verifies locally.
- we could use https://hackmd.io/@oK3in1lRQ7-pt7b3j8nQxg/Hk9KBHsGgg here.

## Same Slot Proving
A block must be proven within the slot it is proposed. A [proposed slot architecture](https://ethresear.ch/t/prover-killers-killer-you-build-it-you-prove-it/22308) is that the builder must prove its own block. In the following we explore what a slot may look like under both Delayed Execution ([EIP-7886](https://eips.ethereum.org/EIPS/eip-7886)) and enshrined Proposer-Builder Separation (ePBS [EIP-7732](https://eips.ethereum.org/EIPS/eip-7732)). One of these two designs will likely be chosen to be included in the [Glamsterdam upgrade](https://ethereum-magicians.org/t/eip-7773-glamsterdam-network-upgrade-meta-thread/21195). The time stamps proposed here are only an indication and are likely to be changed in the future based on testing. For more information, see the [Same Slot Proving proposal](https://ethresear.ch/t/prover-killers-killer-you-build-it-you-prove-it/22308). Note that in the designs below, we assume that proofs are stored [off-chain](https://ethresear.ch/t/native-rollups-superpowers-from-l1-execution/21517), however, the design could easily be changed to include proofs onchain.

### Same Slot Proving under Delayed Execution
- Slot `n`, `t=0`: The builder of slot `n` propagates the block and its blobs.
- Slot `n`, `t=1.5`: Attesters statically validate the beacon block.
- Slot `n`, `t=10` (Proof Deadline): Attesters freeze their view of available proofs.
- Slot `n+1`, `t=0`: The builder of slot `n+1` indicates whether the proofs were timely with a flag in the block.
- Slot `n+1`, `t=1.5`: If attesters saw a valid proof by the proof deadline, but the builder indicated otherwise, they do not vote for block.

### Same Slot Proving under ePBS
With ePBS, the beacon block and execution payload are separate objects sent by the beacon proposer and the builder respectively. A proof is necessary for the execution payload. No proof is required for the beacon block. With ePBS, there is no timeliness flag necessary for the proofs of the previous block since a beacon proposer does not build on a late execution payload. Therefore, building on an execution payload is an implicit flag of proof timeliness.

- Slot `n`, `t=0`: The beacon proposer of slot `n` propagates a beacon block that commits to an execution payload hash.
- Slot `n`, `t=1.5`: Attesters vote for the beacon block.
- Slot `n`, `t=10`: The [Payload-Timeliness Committee (PTC)](https://ethresear.ch/t/payload-timeliness-committee-ptc-an-epbs-design/16054#proposer-initiated-splitting-18) votes for the availability of the execution payload, the blobs, and the proof.
- Slot `n+1`, `t=0`: The beacon proposer of slot `n+1` builds on the beacon block and execution payload of slot `n` if both were available.
- Slot `n+1`, `t=1.5`: If the PTC voted for the availability of all necessary objects, attesters only vote for the beacon block of slot `n+1` if it builds on the beacon block and execution payload of slot `n`.

