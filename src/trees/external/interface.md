# zkEVM Interface
Before exploring how zkEVMs can enhance the Ethereum protocol, we first clarify what they are.
Specifically, we define the functionality they offer by outlining their syntax and interfaces, as well as their security and efficiency properties.
With this definition in place, we can treat zkEVMs as a black box throughout the remainder of this part.

> **Remark:** We sometimes use the terms *zkVM* and *zkEVM* interchangeably, although they historically have a slightly different meaning: while a zkVM can prove the execution of a general program, a zkEVM was meant to prove the execution of the Ethereum Virtual Machine (EVM). Our definition is actually closer to a zkVM, but the programs that we input will be Ethereum clients.

## Intuition
At a high level, zkEVMs provide a *proof of correct execution*.

Imagine two parties: a powerful *prover*, Alice, and a less powerful *verifier*, Bob.
Both agree on a function `f` — for example, one implemented in Rust.
They also share an input `x` and an output `y`, and Alice claims that `f(x) = y`.

How can Bob be confident this claim is correct?

- *Naive solution.* Bob could re-execute `f` on input `x` himself, but this might be prohibitively expensive.
- *Succinct argument solution.* Instead, Alice can produce [*succinct non-interactive argument* (SNARG)](https://eprint.iacr.org/2010/610.pdf), which convinces Bob that `f(x) = y`. "Succinct" here means that the argument is small in size and inexpensive to verify - much cheaper than full re-execution.

SNARGs are fascinating cryptographic constructs, and remarkably, they can be built efficiently [and unconditionally in the random oracle model](https://eprint.iacr.org/2016/116.pdf).
For now, we treat SNARGs as given.

Importantly, because SNARGs are non-interactive, the proof that a single powerful prover Alice computes, can be used to convince *arbitrarily many verifiers* Bob.

## Syntactical Interface
We now define the core interface that a zkEVM exposes:


- `Prepare(f) -> vk`. This function takes as input a description of the program `f` and outputs a verification key `vk`. This key does not need to be kept secret.
The `Prepare` step serves as a preprocessing phase, after which only the key `vk` must be retained. This can be advantageous: for example, `vk` is typically much smaller than the full description of `f`.

- `Prove(f, x) -> (y, proof)`. This function is executed by the Prover. It runs the program `f` on input `x` to compute the output `y`, and simultaneously generates a short proof (more precisely, a *succinct argument*) attesting that `f(x) = y`.

- `Verify(vk, x, y, proof) -> 0/1`. This function is executed by the Verifier. It checks the validity of the given proof with respect to the verification key `vk`. It outputs `0` (for reject) or `1` (for accept).

> **Remark:** When talking about zkEVMs in Ethereum, it is important to agree on a syntax first. In fact, a [recent project](https://github.com/eth-act/ere) aims to unify the syntax existing zkVM implementations. The interface is only minimally more complex than what we have defined.

## Semantical Properties
- what are the properties that they provide?
> **Remark:**  note that they are not zk!

### Completeness

### Soundness and Knowledge Soundness

### Succinctness
