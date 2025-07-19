# Arithmetization

Before a zkVM can prove a computation, it must first translate that computation into a language that cryptographic proof systems understand: the language of polynomials. This translation process is called **arithmetization**. It is the step that converts a statement about computational integrity, like *"this program was executed correctly"*, into an equivalent algebraic statement about a system of polynomial equations.

The core goal is to create a set of polynomial constraints that are satisfied **if and only if** the original computation was valid. Once we have this algebraic representation, the subsequent layers of the proof system, such as a **Polynomial IOP**, can take over to generate a succinct proof.

## The Core Components: Execution Trace & Constraints

At the heart of arithmetizing a machine's computation are two key components: the trace and the constraints that govern it.

### The Execution Trace

The first step is to record the computation's every move. This record is called the **execution trace**. Conceptually, it's a table where each row represents the complete state of the virtual machine at a single point in time (a clock cycle), and each column represents a specific register or memory cell tracked over time.

For example, a simple computation that calculates a Fibonacci sequence would have a trace with one column, where each row `n` contains the `n`-th Fibonacci number. This detailed table is the **execution trace**.

To understand why this trace is also called the **witness**, it helps to step back and look at the general cryptographic meaning of the term. A witness is the secret data that a prover possesses that makes a public statement true. For a zkVM, the public statement might be *"running this program with public input `x` results in public output `y`"*. The witness is the answer to the question *"how?"*. It's the complete set of intermediate variables and secret values that demonstrate the computation's validity from start to finish.

Therefore, the execution trace (as a complete, step-by-step record of the machine's internal state) is the concrete form this witness takes for a program's execution. It is the core evidence the prover will use to construct the proof.

| Clock Cycle (Row) | Register 0 (Column) |
| :---------------- | :------------------ |
| 0                 | 1                   |
| 1                 | 1                   |
| 2                 | 2                   |
| 3                 | 3                   |
| 5                 | 5                   |
| ...               | ...                 |

### Constraints: The Rules of the Computation

An execution trace is only valid if it follows the rules of the computation. These rules are formalized as **constraints**, which are simply polynomial equations that must hold true.

* **Transition Constraints:** These enforce the rules for moving from one state (row) to the next. For our Fibonacci example, the transition constraint is `Trace[n+2] = Trace[n+1] + Trace[n]`. This rule must be satisfied for every applicable row transition.

* **Boundary Constraints:** These enforce conditions on the start and end states. In the Fibonacci example, a boundary constraint would be `Trace[0] = 1` and `Trace[1] = 1`, fixing the initial state.

If every transition and boundary constraint is satisfied, the computation is valid. The power of arithmetization lies in expressing both the trace and these constraints using polynomials.

## Arithmetization Schemes

Once a computation has been captured in an execution trace, the next step is to encode its correctness as a set of polynomial constraints. This encoding, called an *arithmetization scheme*, is the interface between the computation and the proof system. Different zkVMs adopt different schemes depending on the nature of the computation and the proof system they target.

We now describe the three dominant families of arithmetization: **R1CS**, **Plonkish**, and **AIR**.


### R1CS (Rank-1 Constraint Systems)

R1CS is the classic circuit-based arithmetization used in many SNARKs. It turns a computation into a collection of simple algebraic constraints, each one shaped like a small multiplication gate followed by a subtraction. These constraints are then checked to ensure the computation was done correctly.

Each R1CS constraint has the form:

\begin{equation}
(\mathbf{s} \cdot \mathbf{a}) \cdot (\mathbf{s} \cdot \mathbf{b}) = (\mathbf{s} \cdot \mathbf{c})
\end{equation}

or rearranged as:

\begin{equation}
(\mathbf{s} \cdot \mathbf{a}) \cdot (\mathbf{s} \cdot \mathbf{b}) - (\mathbf{s} \cdot \mathbf{c}) = 0
\end{equation}

Let’s break this down:

* \\(\mathbf{s}\\) is the **witness vector**. It contains all the values used in the computation: inputs, outputs, and any intermediate results.
* \\(\mathbf{a}\\), \\(\mathbf{b}\\), and \\(\mathbf{c}\\) are vectors that select which elements of \\(\mathbf{s}\\) are used in this constraint. They are usually sparse (mostly zeros), and act like masks.

This single constraint checks whether a product equals another value, based on selected parts of the witness. To represent a full computation, we create one such constraint for each small step, forming a kind of "circuit" of arithmetic gates.

#### Example: Computing \\(w = x^4\\)

We can’t do this in one step, because R1CS only supports one multiplication per constraint. So we break it into smaller steps:

1. First, compute \\(w_1 = x \cdot x\\)
2. Then, compute \\(w = w_1 \cdot w_1\\)

Each of these steps becomes one R1CS constraint. That means computing \\(x^4\\) requires **two constraints** and **one intermediate variable** \\(w_1\\).

#### Why Use R1CS?

R1CS is powerful because it’s simple and general-purpose: you can represent **any** computation this way, given enough constraints and intermediate variables. It’s the backbone of many SNARK proof systems like Groth16 and Marlin.

However, it also comes with limitations:

* **Rigid structure**: Every operation must be written as a multiplication and subtraction. Even simple operations like XOR or range checks may require several constraints and helper variables.

* **Gadget overhead**: To represent common programming features like conditionals, booleans, or comparisons, you need to build "gadgets", i.e. small collections of constraints designed to implement that logic.

So while R1CS is a flexible and foundational technique, it can become inefficient for complex logic or memory-heavy computations. That’s where newer schemes like Plonkish and AIR offer improvements.


### Plonkish Arithmetization

Plonkish arithmetization, used in systems like Plonk and UltraPlonk, builds on R1CS but introduces more flexibility. Instead of hardwiring each constraint into a fixed quadratic form, it views computation as a table of values (a trace), where each row represents one step of the computation. Constraints are then applied row-by-row using selector polynomials.

#### Key Features:

* **Custom Gates via Selectors**:
  Instead of one fixed constraint form, Plonk allows general linear combinations of trace columns with selector polynomials:

  \begin{equation}
  q_L \cdot a + q_R \cdot b + q_M \cdot a \cdot b + q_O \cdot c + q_C = 0,
  \end{equation}

  where
  - \\(a\\), \\(b\\), and \\(c\\) are witness values from the current row,
  - \\(q_L\\), \\(q_R\\), \\(q_M\\), \\(q_O\\), \\(q_C\\) are selectors that define the operation.

  This allows each row of the trace to act like a different type of gate, depending on the selectors: addition, multiplication... It reduces constraint count by unifying multiple operations under a single constraint form.

* **Wiring via Permutation Checks**:
  Plonkish systems verify that outputs from one step are correctly reused as inputs in another using a *permutation argument*. Instead of tracking every connection manually, the prover shows that two lists (e.g., inputs and outputs) are just permutations of each other. This enables efficient wiring checks without complex gadgets.

* **Lookup Arguments**:
  For operations like bitwise logic, comparisons, or range checks (which are expensive to encode algebraically) Plonkish systems allow the prover to simply *lookup* values from a precomputed table. The prover proves that its inputs match some row in a trusted dictionary of valid values (e.g., all valid `(a, b, a ⊕ b)` tuples for an XOR gate).

Together, these features make Plonkish systems more expressive than R1CS while remaining SNARK-friendly and efficient for a wide class of computations.

### AIR (Algebraic Intermediate Representation)

AIR is the native arithmetization scheme used by STARK-based zkVMs. Rather than viewing computation as a circuit, AIR treats it as a sequence of machine states evolving over time. Each column of the execution trace corresponds to a register or memory cell, and constraints express algebraic relations between rows.

#### How AIR Works:

1.  **Trace Polynomials**:
    Each column of the execution trace is interpolated into a univariate polynomial \\(P_i(x)\\). This interpolation is performed over a specific subset of the finite field called the **evaluation domain**. To make the math efficient, this domain is chosen to be a *cyclic multiplicative subgroup*, which means it has a **generator** \\(g\\) whose powers (e.g., \\(g^0, g^1, g^2, \ldots\\)) can produce every element in the domain. The input trace thus becomes a vector of polynomials:

    $$
    P_0(x), P_1(x), \ldots, P_{w-1}(x)
    $$

2. **Polynomial Constraints**:
   Transition and boundary constraints are expressed as polynomial identities involving shifted evaluations. For instance, the rule

   $$
   \texttt{Reg}_C[n+1] = \texttt{Reg}_A[n] + \texttt{Reg}_B[n]
   $$

   becomes:

   $$
   P_C(gx) - P_A(x) - P_B(x) = 0
   $$

   where \\(g\\) is a generator of the evaluation domain, representing a one-step shift in time.

   To connect this to a real zkVM, imagine the trace has columns for the current instruction's opcode and various machine registers. The polynomial constraints are responsible for enforcing the entire CPU instruction set. For instance:

    * **Instruction Logic**: For a row where the opcode is `ADD`, a constraint enforces that the values from two source register columns sum to the value in the destination register column. This is often written conditionally, like $$\text{is_add_opcode} \cdot (P_{dest} - (P_{source1} + P_{source2})) = 0,$$ so the constraint only applies when the `ADD` instruction is active.

    * **Program Counter (PC)**: The PC register, which points to the next instruction, is also constrained. For most rows, the constraint is $$P_{PC}(g \cdot x) - (P_{PC}(x) + 1) = 0,$$ ensuring it simply increments. For a `JUMP` instruction, however, the constraint would change to enforce that the next PC value matches the jump destination specified in the instruction.

    In this way, a complete set of polynomial constraints collectively describes the behavior of every possible instruction, ensuring the entire program execution is valid.

3.  **Quotient Argument**:
    To prove that each constraint polynomial \\(C(x)\\) is zero on its specific enforcement domain \\(D\\), the prover uses an algebraic argument. They construct the vanishing polynomial \\(V_D(x)\\) (which by definition is zero on all \\(x \in D\\)) and compute the quotient:

    $$
    Q(x) = \frac{C(x)}{V_D(x)}
    $$

    If \\(C(x)\\) correctly vanishes over \\(D\\), then the division is clean and \\(Q(x)\\) is a valid low-degree polynomial. If not, the division would result in a rational function (not a polynomial), and the prover's dishonesty would be revealed. This process transforms the set of original constraint claims into an equivalent set of claims that each \\(Q_i(x)\\) is a valid polynomial.

4.  **Composition Polynomial**:
    Instead of testing each individual quotient polynomial \\(Q_i(x)\\) for being low-degree (which would be inefficient), they are all merged into a single polynomial using random coefficients \\(\alpha_i\\) provided by the verifier:

    $$
    C_{\text{comp}}(x) = \sum_i \alpha_i \cdot Q_i(x)
    $$

    This single, randomly-combined composition polynomial is what the proof system ultimately tests. If this final polynomial is low-degree, it implies with very high probability that all the individual quotient polynomials were also low-degree, and thus all the original constraints were satisfied. This polynomial is then passed to a protocol like FRI for a final low-degree test.

> **Note on Multilinear Polynomials**: While treating each column as a separate univariate polynomial is common, more modern AIR-based systems can view the entire execution trace as a single multilinear polynomial. As described in recent work on [Jagged Polynomial Commitments](https://eprint.iacr.org/2025/917), this approach allows the system to commit to the entire trace as one object, which can drastically reduce the verification cost and prover overhead associated with managing many individual column commitments.

AIR’s tight alignment with machine semantics, support for high-degree constraints, and STARK-compatibility make it the arithmetization of choice for zkVMs that prioritize scalability and transparency.


## Resources for Further Information

- [Study of Arithmetization Methods for STARKs](https://eprint.iacr.org/2023/661)
- [Converting Algebraic Circuits to R1CS (Rank One Constraint System)](https://rareskills.io/post/rank-1-constraint-system)
- [Arithmetization schemes for ZK-SNARKs](https://blog.lambdaclass.com/arithmetization-schemes-for-zk-snarks/)
- [Arithmetization in STARKs an Intro to AIR](https://threesigma.xyz/blog/zk/arithmetization-starks-algebraic-intermediate-representation)
- [Quadratic Arithmetic Programs: from Zero to Hero](https://medium.com/@VitalikButerin/quadratic-arithmetic-programs-from-zero-to-hero-f6d558cea649)
- [Proving AIRs with multivariate sumchecks](https://solvable.group/posts/air-multivariate-sumcheck/)
- [The Elegant Foundation: Plonkish Arithmetization](https://www.bigsky.gg/posts/mina_3/)
