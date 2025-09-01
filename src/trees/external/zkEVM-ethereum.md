# Part I: zkEVMs in Ethereum

This book takes a top-down approach to exploring zkEVMs. Accordingly, this first part introduces the technology from an external perspective, treating it as a powerful new tool for the Ethereum protocol without delving into its internal cryptographic complexities.

Our focus is on answering two fundamental questions: **What exactly is a zkEVM, and how can it be used to address Ethereum's most pressing challenges?**

To do this, we will first define zkEVMs as a "black box" by their inputs, outputs, and security guarantees. From there, we will examine their impact on the protocol, from the scalability benefits they unlock to the practical engineering and economic considerations required for their integration.

Our exploration will cover:

  * **[zkEVM Interface](interface.md):** Defining the formal syntax and the core properties (completeness, soundness, and succinctness) that make a zkEVM a secure and efficient tool.
  * **[Problems Addressed](problemsaddressed.md):** Exploring how zkEVMs provide compelling solutions to major Ethereum challenges, including limited throughput, high gas fees, and the risk of validator centralization.
  * **[Architecture](architecture.md):** Investigating how zkEVMs would integrate into Ethereum's client and consensus layers, including potential models for slot timing and proof submission.
  * **[Incentives](incentives.md):** Examining the economic layer required to build a robust and reliable ecosystem, from preventing "prover killer" blocks to ensuring censorship resistance.
  * **[Preconditions for Integration](preconditions.md):** Outlining the essential technical requirements for deploying zkEVMs on the base layer, with a deep dive into the critical need for **real-time proving**.

Upon completing this part, you will have a clear understanding of the role zkEVMs are supposed to play in Ethereum's future, the benefits they offer, and the practical roadmap for their implementation. We assume the reader has a basic familiarity with the Ethereum protocol.
