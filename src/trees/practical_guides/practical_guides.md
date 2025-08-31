# Part III: Practical Guides

This section provides hands-on guidance for different audiences interested in working with zkEVMs. Whether you're a blockchain developer, a client implementer, or a researcher, you'll find practical information to help you get started with zkEVM technology.

This section will guide you through the practical aspects of working with zkEVMs, from setting up your environment to deploying and testing applications.

### Getting Started with ERE

[ERE](https://github.com/eth-act/ere) is a powerful unified zkVM interface and toolkit that allows developers to work with multiple zkVMs through a consistent API. ERE simplifies the process of compiling, executing, proving, and verifying programs across different zkVM implementations.

### What is ERE?

ERE provides a unified Rust API that works across multiple zkVM backends. This abstraction layer allows developers to:

- Switch between different zkVM implementations with minimal code changes
- Use a consistent API for compiling, executing, proving, and verifying
- Leverage the strengths of various zkVMs for different use cases

### Developer Workflow with ERE

When working with ERE, developers typically follow this workflow:

1. **Write a guest program**: Create the program you want to execute and generate proofs for using your preferred zkVM.
2. **Use the ERE library**: Generate prover and verification keys, execute the guest program, generate proofs, and verify those proofs all through ERE's unified interface.

### Example: Building Your First zkVM Program with ERE

This example demonstrates how to compile, prove, and verify a simple program using ERE. You can choose between two setup options depending on your preference for performance or simplicity.



#### Option 1: With Local SDK Installation

This approach provides better performance and debugging capabilities by installing the necessary zkVM SDKs directly on your machine.

1.  **Install SDKs**
    First, install the SDK for the zkVM you want to use. For example, to install the SP1 SDK, run:

    ```bash
    bash scripts/sdk_installers/install_sp1_sdk.sh
    ```

2.  **Add Dependencies**
    Next, add the required ERE crates to your project's `Cargo.toml` file.

    ```toml
    # Cargo.toml
    [dependencies]
    zkvm-interface = { git = "https://github.com/eth-act/ere.git", tag = "v0.0.11" }
    ere-sp1        = { git = "https://github.com/eth-act/ere.git", tag = "v0.0.11" }
    ```

3.  **Compile & Prove Example**
    Now you can write your host program to compile the guest code and generate a proof.

    ```rust
    // main.rs
    use ere_sp1::{EreSP1, RV32_IM_SUCCINCT_ZKVM_ELF};
    use zkvm_interface::{Compiler, Input, ProverResourceType, zkVM};

    fn main() -> Result<(), Box<dyn std::error::Error>> {
        let guest_directory = std::path::Path::new("workspace/guest");

        // Compile guest
        let compiler = RV32_IM_SUCCINCT_ZKVM_ELF;
        let program = compiler.compile(guest_directory)?;

        // Create zkVM instance
        let zkvm = EreSP1::new(program, ProverResourceType::Cpu);

        // Prepare inputs
        let mut io = Input::new();
        io.write(42u32);

        // Execute
        let _report = zkvm.execute(&io)?;

        // Prove
        let (proof, _report) = zkvm.prove(&io)?;

        // Verify
        zkvm.verify(&proof)?;

        Ok(())
    }
    ```



#### Option 2: Docker-Only Setup

This method is simpler as it only requires Docker to be installed, running the zkVM operations within a container.

1.  **Add Dependencies**
    Add the `ere-dockerized` crate to your `Cargo.toml` instead of a specific backend crate.

    ```toml
    # Cargo.toml
    [dependencies]
    zkvm-interface = { git = "https://github.com/eth-act/ere.git", tag = "v0.0.11" }
    ere-dockerized = { git = "https://github.com/eth-act/ere.git", tag = "v0.0.11" }
    ```

2.  **Compile & Prove Example**
    The host program is slightly different, as it uses the Dockerized compiler and zkVM wrappers.

    ```rust
    // main.rs
    use ere_dockerized::{EreDockerizedCompiler, EreDockerizedzkVM, ErezkVM};
    use zkvm_interface::{Compiler, Input, ProverResourceType, zkVM};

    fn main() -> Result<(), Box<dyn std::error::Error>> {
        let guest_directory = std::path::Path::new("workspace/guest");

        // Compile guest using the Dockerized compiler for SP1
        let compiler = EreDockerizedCompiler::new(ErezkVM::SP1, std::path::Path::new("workspace"));
        let program = compiler.compile(guest_directory)?;

        // Create a Dockerized zkVM instance for SP1
        let zkvm = EreDockerizedzkVM::new(ErezkVM::SP1, program, ProverResourceType::Cpu)?;

        // Prepare inputs
        let mut io = Input::new();
        io.write(42u32);

        // Execute
        let _report = zkvm.execute(&io)?;

        // Prove
        let (proof, _report) = zkvm.prove(&io)?;

        // Verify
        zkvm.verify(&proof)?;

        Ok(())
    }
    ```

### ERE for client development

For Ethereum client developers, ERE serves as a crucial tool for a specific and vital task: proving the execution of an Ethereum block statelessly. This process is fundamental to evaluating how different zkVMs perform with various Execution Layer (EL) clients, which is essential for potential L1 integration.

The core idea is to treat an EL client as the **guest program**. This program is executed inside a zkVM to perform **stateless block validation**. Instead of connecting to a live network and accessing a database, the client receives a block and a "witness" containing all the necessary state data (like account balances, contract code, and storage) as inputs. The client then executes the block and, if successful, proves that the resulting post-state root and other block details are correct according to Ethereum's rules.

While you can use the ERE library directly to build custom solutions, the Ethereum Foundation's zkEVM team has developed a higher-level tool called `zkevm-benchmark-workload`. This tool is specifically designed for this purpose and uses ERE under the hood. It allows developers to:

* **Run benchmark fixtures** through various EL client and zkVM combinations.
* Use test cases from the official `execution-spec-tests` repository.
* Test performance against real mainnet blocks or custom-designed "worst-case" scenarios intended to stress-test the prover.

By using these tools, client developers can systematically benchmark how their client performs as a guest program inside different zkVMs. This work is critical for identifying performance bottlenecks, analyzing security implications, and providing concrete data to guide optimizations in both the EL clients and the zkVMs themselves.

Here is what an example guest program for stateless block verification for the SP1 zkVM looks like:

```rust 
//! SP1 guest program

#![no_main]

extern crate alloc;
use alloc::sync::Arc;

use guest_libs::mpt::SparseState;
use reth_chainspec::ChainSpec;
use reth_evm_ethereum::EthEvmConfig;
use reth_stateless::{Genesis, StatelessInput, stateless_validation_with_trie};
use tracing_subscriber::fmt;

sp1_zkvm::entrypoint!(main);
/// Entry point.
pub fn main() {
    init_tracing_just_like_println();

    println!("cycle-tracker-report-start: read_input");
    let input = sp1_zkvm::io::read::<StatelessInput>();
    let genesis = Genesis {
        config: input.chain_config.clone(),
        ..Default::default()
    };
    let chain_spec: Arc<ChainSpec> = Arc::new(genesis.into());
    let evm_config = EthEvmConfig::new(chain_spec.clone());
    println!("cycle-tracker-report-end: read_input");

    println!("cycle-tracker-report-start: validation");
    stateless_validation_with_trie::<SparseState, _, _>(
        input.block,
        input.witness,
        chain_spec,
        evm_config,
    )
    .unwrap();
    println!("cycle-tracker-report-end: validation");
}

/// Initializes a basic `tracing` subscriber that mimics `println!` behavior.
///
/// This is because we want to use tracing in the `no_std` program to capture cycle counts.
fn init_tracing_just_like_println() {
    // Build a formatter that prints *only* the message text + '\n'
    let plain = fmt::format()
        .without_time() // no timestamp
        .with_level(false) // no INFO/TRACE prefix
        .with_target(false); // no module path

    fmt::Subscriber::builder()
        .event_format(plain) // use the stripped-down format
        .with_writer(std::io::stdout) // stdout == println!
        .with_max_level(tracing::Level::INFO) // capture info! and up
        .init(); // set as global default
}

```

The tools and techniques outlined here are not just for experimentation; they are the essential building blocks for the future of Ethereum scaling and verifiable computation. We encourage you to build upon these examples, explore the different zkVM backends, and contribute to the ongoing effort to create a more scalable and secure decentralized world.


#### Resources
https://github.com/eth-act/ere
https://github.com/eth-act/zkevm-benchmark-workload
https://github.com/eth-act/zkevm-benchmark-workload/blob/master/ere-guests/stateless-validator/sp1/src/main.rs