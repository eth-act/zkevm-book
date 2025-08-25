# Part II: zkVM Internals

In Part I, we treated zkVMs as a black box, defining their interface and exploring their applications within the Ethereum protocol. We now transition from the *what* to the *how*. This second part of the book opens that black box to examine the internal components and core design principles that make these systems work.

Our focus is on answering the central question: **How do zkVMs function internally?**

While each implementation makes unique architectural choices, they are all constructed from a common set of foundational layers. The goal here is not to provide an exhaustive manual for a single zkVM. Rather, it is to demystify these shared layers and equip you with a robust conceptual framework for understanding the entire field.

To achieve this, we will explore the essential building blocks and concepts that form a modern zkVM, in a logical progression from high-level architecture down to the cryptographic primitives:

* **[Instruction Set Architecture (ISA)](isa.md):** The fundamental "language" the virtual machine understands, and the trade-offs between general-purpose ISAs like RISC-V and zk-optimized designs like Cairo.
* **[Arithmetization](arithmetization.md):** The crucial first step of translating a program's execution trace into a system of polynomial equations—the mathematical language that proof systems understand.
* **[Polynomial IOPs](polyiops.md):** The abstract, information-theoretic protocol that allows a prover to convince a verifier of a computational claim using polynomial "oracles."
* **[Polynomial Commitment Schemes](polycoms.md):** The cryptographic tool used to commit to large polynomials with a small piece of data, forming the foundation for succinctness in modern SNARKs.
* **[The BCS Transform](bcs.md):** The cryptographic compiler that transforms an abstract, interactive protocol into a single, non-interactive, and compact proof using hashing and Merkle trees.
* **[Security Considerations](security.md):** An analysis of the security guarantees, assumptions, and potential vulnerabilities of a complete zkVM system, from knowledge soundness to the importance of formal verification.

Upon completing this part, you will grasp the key technologies, understand the critical design trade-offs, and be prepared to confidently explore the technical details of any specific zkVM implementation.
