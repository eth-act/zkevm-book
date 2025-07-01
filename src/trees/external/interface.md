# zkEVM Interface
Before exploring how zkEVMs can enhance the Ethereum protocol, we first clarify what they are.
Specifically, we define the functionality they offer by outlining their syntax and interfaces, as well as their security and efficiency properties.
With this definition in place, we can treat zkEVMs as a black box throughout the remainder of this part.
We invite the reader to take a pause after this section and think about which problems in Ethereum could be solved using zkEVMs.

> **Remark:** We sometimes use the terms *zkVM* and *zkEVM* interchangeably, although they historically have a slightly different meaning: while a zkVM can prove the execution of a general program, a zkEVM was meant to prove the execution of the Ethereum Virtual Machine (EVM). Our definition is actually closer to a zkVM, but the programs that we input will be the state transition functions of Ethereum clients.

## Intuition
At a high level, zkEVMs provide a *proof of correct execution*.

Imagine two parties: a computationally powerful *prover*, Alice, and a less powerful *verifier*, Bob.
Both agree on a function `f` — for example, one implemented in Rust.
They also share an input `x` and an output `y`, and Alice claims that `f(x) = y`.

How can Bob be confident this claim is correct?

- *Naive solution.* Bob could re-execute `f` on input `x` himself, but this might be prohibitively expensive.
- *Succinct argument solution.* Instead, Alice can produce a [*succinct non-interactive argument* (SNARG)](https://eprint.iacr.org/2010/610.pdf), which convinces Bob that `f(x) = y`. "Succinct" here means that the argument is small in size and inexpensive to verify - much cheaper than full re-execution.

SNARGs are fascinating cryptographic constructs, and remarkably, they can be built efficiently [and unconditionally in the random oracle model](https://eprint.iacr.org/2016/116.pdf).
For now, we treat SNARGs as given.

Importantly, SNARGs are non-interactive: Alice just sends a single convincing argument (or proof) to Bob and they do not need to interact back and forth.
Especially, the proof that a single powerful prover Alice computes, can be used to convince *arbitrarily many verifiers* Bob.

## Syntactical Interface
We now define the core interface that a zkEVM exposes:


- `Prepare(f) -> vk`. This function takes as input a description of the program `f` and outputs a verification key `vk`. This key does not need to be kept secret.
The `Prepare` step serves as a preprocessing phase, after which only the key `vk` must be retained. This can be advantageous: for example, `vk` is typically much smaller than the full description of `f`.

- `Prove(f, x) -> (y, proof)`. This function is executed by the Prover. It runs the program `f` on input `x` to compute the output `y`, and simultaneously generates a short proof (more precisely, a *succinct argument*) attesting that `f(x) = y`.

- `Verify(vk, x, y, proof) -> 0/1`. This function is executed by the Verifier. It checks the validity of the given proof with respect to the verification key `vk`. It outputs `0` (for reject) or `1` (for accept).

> **Remark:** When talking about zkEVMs in Ethereum, it is important to agree on a syntax first. In fact, a [recent project](https://github.com/eth-act/ere) aims to unify the syntax existing zkVM implementations. The interface is only minimally more complex than what we have defined.

## Properties
Now that we have a syntax at hand, we turn to the functionality, security, and efficiency properties that zkEVMs must provide.
Namely, these are *completeness* (always can generate proofs for true statements), *soundness* (cannot generate proofs for false statements), and *succinctness* (proof is small and verification is cheaper than re-execution).

> **Remark:**  Despite what the name might imply, zkEVMs generally do not provide *zero-knowledge* in the cryptographic sense, i.e., the property of revealing nothing beyond the validity of the statement. This confusion stems from a misuse of the term "ZK" within large parts of the blockchain community, where it is often used synonymously with succinctness. However, zero-knowledge and succinctness are *orthogonal* properties that a proof system may or may not possess. For example, many systems studied in the literature are zero-knowledge but not succinct.
In reality, zkEVMs would be more accurately described as *succinct EVMs* or *SNARG-EVMs*. Nevertheless, we adopt the term "zkEVM" in this book to remain consistent with prevailing community terminology, despite its technical imprecision.

### Completeness
The most basic property that zkEVMs should have is *completeness*.
Intuitively, it states that honestly generated proofs always verify.
In particular, this will be essential for ensuring liveness when we use zkEVMs in Ethereum.
Precisely, completeness means that the following always outputs `1`, for any function `f` and input `x`:
```
vk := Prepare(f)
(y, proof) := Prove(f, x)
Output Verify(vk, x, y, proof)
```

### Soundness
We now turn to the main security property of zkEVMs, namely, *soundness*.
It states that proofs for wrong claims must be rejected with overwhelmingly high probability.
Note the complementary nature of completeness and soundness: the former defines a case in which a claim should be accepted, while the latter defines a case in which a claim should be rejected.

Let us make this more precise now.
As common in cryptography, we define soundness via a probabilistic experiment (also called a *game*) that involves a hypothetical adversary `Adv`.
The experiment for soundness is as follows:
1. `Adv` receives the system parameters, e.g., descriptions of hash functions that are involved.
2. `Adv` can do some computation and then output a claim and a proof, given by `f, x, y, proof`.
3. `Adv` *wins the game*, if *f(x) != y* (the claim is false) but `Verify(vk, x, y, proof) == 1`.

That is, an adversary breaks soundness and wins the game, if it can come up with a false claim accompanied by a verifying proof, i.e., if it can fool the verifier.
Now, we say that a zkEVM is *sound*, if every *efficient* adversary has a negligible probability of winning in this game.

>**Remark:** We do not specify exactly what the terms "efficient" and "negligible" mean, and refer the curious reader to the cryptographic literature. However, we want to make explicit that computationally unbounded (i.e., inefficient) adversaries may break the security of the scheme. This is why we should actually talk about *arguments* and not about *proofs*, but we treat the terms interchangeably for this book.

>**Remark:** Note the *adaptive* nature of the soundness experiment: the adversary can choose the claim `f,x,y` *after* seeing the system parameters. This is an important detail.

>**Remark:** Note that we can alway easily design a system that is complete but not sound, or vice versa: If `Verify` always outputs `1`, the system is complete but for sure not sound. On the other hand, if it always outputs `0`, the system is clearly sound but not complete.

### Succinctness
The final property is an efficiency property known as *succinctness*.
Intuitively, it means that verifying a proof is significantly cheaper than re-executing the underlying computation.
More formally, we require that both the size of `proof` and the running time of `Verify` are sublinear in the running time of the original function `f`.

In some practical zkEVM constructions, these quantities can even be constant, depending only on a security parameter.
To give a concrete sense of scale: a proof attesting to the correctness of a large computation may be just a few hundred kilobytes.