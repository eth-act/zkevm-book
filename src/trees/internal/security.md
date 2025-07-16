## Security

Security is critical to the success of zkEVMs, especially as they move toward becoming part of Ethereum’s core infrastructure. To ensure a safe and resilient transition, the Ethereum community plans a phased rollout: zkEVM proving will be optional at first and become mandatory only once systems have been thoroughly vetted. Diversity is also a cornerstone of this approach—we aim to support multiple independent zkEVM clients so that if one encounters issues, others can continue to provide correct proofs. To guide this effort, there is a proposed set of [security guidelines](https://docs.google.com/document/d/1W6Y_JPxkt9D2P2sJe29FPOe8IuNiaPUvySpuzWcvnwk/edit?tab=t.0#heading=h.5vymhic1kh9x) rooted in best practices from the broader cryptography and software industries, helping projects design, implement, and evaluate zkEVMs with robustness in mind.

### Understanding Knowledge Soundness

The goal of a zkEVM is to ensure that when someone claims they executed some EVM bytecode, they really did. That is, if a proof says “this transaction was processed correctly,” we should be confident that the prover *knows* how that transaction was executed. This property is called **knowledge soundness**, and it’s what prevents malicious provers from fabricating proofs out of thin air.

To achieve this, zkEVMs are built in layers. Here’s roughly how that structure looks:

1. The EVM bytecode is run through a virtual machine using a low-level instruction set (an ISA).
2. That execution is arithmetised into a set of algebraic constraints.
3. An **Interactive Oracle Proof (IOP)** is used to check that those constraints are satisfied.
4. The IOP is compiled into an **interactive argument** using a **polynomial commitment scheme**.
5. Finally, the protocol is made non-interactive using the **Fiat-Shamir transform**, so it can be verified with just a single message.

The first three layers—the ISA execution, the constraints, and the IOP—are designed to be **information-theoretically** sound. That means, assuming everything is implemented correctly, *even an attacker with infinite computing power* couldn’t forge a convincing proof. This is a very strong guarantee, but it only holds in the idealised interactive world where the verifier has access to imaginary oracles.

To make this practical, we compile the IOP using cryptographic primitives. For example, the BCS transform replaces oracles with polynomial commitments. At this point, we step into the realm of **computational assumptions**. For the compiled zkEVM to remain knowledge sound, the underlying **polynomial commitment scheme (PCS)** must satisfy two crucial properties:

- **Evaluation binding**: once the prover commits to a polynomial, they can’t later claim it evaluates to something else.
- **Knowledge extractability**: if the prover can convince the verifier of a claim, there must be a way to “extract” the polynomial they committed to.

The last step is to eliminate interaction entirely. We do this using the **Fiat-Shamir transform**, which replaces the verifier’s random challenges with deterministic hashes of the proof transcript. Non-interactive arguments are essential for Ethereum, where the prover and verifier cannot interact, but this convenience comes with a caveat: we must assume the hash function behaves like a **random oracle**. While this is a strong heuristic assumption, it is widely used in practice. Some systems even target the **Quantum Random Oracle Model (QROM)** to prepare for potential quantum threats.

Putting it all together: the early layers of the zkEVM stack provide strong, ideal-world guarantees. The later layers compile those guarantees into something practical and succinct, grounded in well-studied cryptographic assumptions. If each layer is carefully constructed, we get a system where every accepted proof genuinely reflects a valid execution—succinctly, securely, and efficiently.


### Knowledge Soundness in the Literature

The above structure for building knowledge-sound zkEVMs appears across many modern proof systems.

* [**Marlin**](https://eprint.iacr.org/2019/1047) builds an IOP for the Rank-1 Constraint System (R1CS) and compiles it into a SNARK using the KZG polynomial commitment scheme. Knowledge soundness in Marlin depends on the extractability of KZG and the security of the Fiat-Shamir transform in the Random Oracle Model.

* [**Binius**](https://eprint.iacr.org/2023/1784) constructs a multilinear polynomial commitment scheme suitable for polynomials over very small fields, including the binary field. They apply their PCS to a binary variant of the Plonkish constraint system, enabling efficient proof systems even in tiny fields.

* [**WHIR**](https://eprint.iacr.org/2024/1586.pdf), [**BaseFold**](https://eprint.iacr.org/2023/1705), and [**FRI**](https://eccc.weizmann.ac.il/report/2017/134/) construct IOPs of proximity, which are used to verify that a function is *close* to one satisfying a set of algebraic constraints. These protocols form the basis for building prover-efficient polynomial commitment schemes, often used in STARKs.

* [**Jolt**](https://eprint.iacr.org/2023/1217) focuses on the RISC-V virtual machine. It arithmetises RISC-V execution using lookup arguments and instantiates them via [**Lasso**](https://eprint.iacr.org/2023/1216), which itself is built using polynomial commitments.

In all of these protocols, the same layered pattern appears: an interactive, information-theoretic proof at the foundation, compiled into a cryptographic argument using polynomial commitments and the Fiat-Shamir transform.

### Targeting 128 Bits of Security
Before proving becomes mandatory on Ethereum L1, zkEVM systems are expected to achieve at least **128 bits of security**. This level of assurance ensures that even powerful, well-resourced adversaries cannot forge proofs or compromise the integrity of the system. During the initial rollout—where proofs may be optional and value at risk is low—some projects may temporarily operate at lower thresholds (e.g., 100-bit security) to test APIs and proving infrastructure. But the target remains clear: 128-bit security is a requirement before mandatory proving can be considered safe.

So what does 128-bit security mean? In cryptographic terms, it means that the best known attack would require an adversary to perform at least $2^{128}$ operations to succeed. This benchmark reflects what the cryptographic community considers a safe margin, even against nation-state level adversaries.

To put this in context, [NIST’s guidelines on key management](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-57pt1r5.pdf) cite 128 bits as the minimum acceptable security level for long-term use, particularly for systems intended to remain secure beyond 2031. This standard underlies a wide range of protocols and systems, including AES-128, elliptic curve cryptography over 256-bit fields, TLS, IPsec, and others.

For a concrete sense of what’s computationally feasible, consider Bitcoin mining. As of 2025, the most difficult known Bitcoin block ([Block 756951](https://blockchair.com/bitcoin/block/756951)) had a hash with 97 leading zero bits. Statistically, achieving such a result requires around $2^{97}$ SHA-256 evaluations. This gives a tangible measure of the scale of effort possible with globally distributed hashpower.

### Measuring Bit Security in zkEVMs

Assessing the bit security of a zkEVM means understanding how soundness can degrade through each layer of the system. Even if individual components are secure, the overall system may be weaker when composed. Key contributors include:

- **Challenge soundness in the IOP**  
  The verifier sends random challenges to the prover. If these are predictable or repeatable, an adversary may get “lucky” and bypass detection. Bit security here depends on the size of the field the challenges are sampled from.

- **Polynomial commitment scheme (PCS)**  
  Most zkEVMs rely on polynomial commitments to compile their IOP into a SNARK. The bit security of the system is bounded by the hardness of forging a commitment or opening it inconsistently. 

- **Hash function assumptions (Fiat-Shamir)**  
  The Fiat-Shamir transform replaces verifier randomness with hash outputs. Soundness here assumes the hash behaves like a **random oracle**, which is a heuristic assumption. Bit security is typically estimated from the preimage or collision resistance of the hash function, though it's often slightly weaker in practice due to modeling gaps.

- **Grindability of Fiat-Shamir**  
  In some systems, adversaries can try multiple inputs to the hash function—called **grinding**—to find a challenge that helps them cheat. If the success probability per attempt is $p$, and the adversary can try $n$ transcripts, then the overall success probability is bounded by $p \cdot n$, reducing effective bit security.

In practice, the total bit security of a zkEVM is limited by **its weakest link**. Even if most components offer 128-bit security, a single step with only 110 bits of assurance could lower the system’s effective strength. Because of this, security analysis should account for all components, their assumptions, and how they compose.

### Specification and Formal Verification

For zkEVMs to be secure, two things must hold: the cryptographic **security proof must be correct**, and the **implementation must match the scheme that was proven secure**. These are both difficult to guarantee in practice. For example, a [critical flaw](https://github.com/Plonky3/Plonky3/security/advisories/GHSA-f69f-5fx9-w9r9) was recently discovered in the Plonky3 FRI verifier—despite the underlying theory being sound. One of the most useful tools for bridging this gap is a **written specification**. A good specification is the foundation for both verification and auditing.

Specifications serve three critical roles:

- First, they allow implementers, auditors, and researchers to understand what the system is supposed to do without needing to reverse engineer the codebase.
- Second, they allow us to formally verify that the implementation matches the intended design, and that this design aligns with the cryptographic scheme that has been proven secure.
- Third, they enable interoperability, allowing other codebases to include compatible but distinct implementations.

Specifications become especially important when **optimisations** are applied. Performance tweaks often restructure or rewrite parts of the system in non-obvious ways. Without a spec, it is difficult to tell whether such changes preserve correctness—or whether they introduce subtle inconsistencies that break the soundness of the whole.

The Ethereum Foundation is currently leading the [Verified ZK-EVM](https://verified-zkevm.org/) initiative, which supports the use of formal methods across the Ethereum ecosystem. The most important step zkEVM projects can take to support this effort is to write **high-quality, accurate, and unambiguous specifications**—with clear semantics and deterministic behavior.

Formal methods are widely used in high-stakes engineering: from aircraft control software, to CPU instruction set verification, to operating system kernels. zkEVMs, which aim to secure smart contract execution on Ethereum’s base layer, face similar stakes. On this topic, the following [xkcd](https://xkcd.com/2030/) seems highly relevant.

### Conclusion

As zkEVMs mature from experimental prototypes into core components of Ethereum’s infrastructure, security must be treated as a first-class concern. Achieving this requires more than just strong cryptographic building blocks—it also demands rigorous security proofs, clear specifications, careful engineering, and a realistic understanding of what can go wrong when these pieces are composed.


