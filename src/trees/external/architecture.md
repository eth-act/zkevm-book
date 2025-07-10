# Architecture
> Note. Section under construction
- Stateless client
- Stateful client
- Two primary patterns have emerged for client integration (to be explored and discussed in more details)
    - Separate clients: Stateless clients (CL+EL) connect to stateful peers over a network subprotocol. Execution and verification are decoupled but remain interactive.
    - Embedded Clients: A single client combines consensus logic (CL) with a lightweight embedded EL verifier (e.g. a zkVM or stateless executor). It receives blocks and proofs via gossip and verifies locally.
