# Ethereum zkEVM book

This book documents research and development efforts around Ethereum's zkEVM initiative.

## Building Locally

To build and read this book locally:

1. Install [mdBook](https://rust-lang.github.io/mdBook/):

    ```bash
    cargo install mdbook
    ```

2. Clone this repository:

    ```bash
    git clone https://github.com/eth-act/zkevm-book.git
    cd zkevm-book
    ```

3. Build and serve the book:

    ```bash
    mdbook serve --open
    ```

The book will open in your default web browser at `http://localhost:3000`.

## Contributing

We welcome comments, pull requests, and contributions.
Be aware that the focus of the book is:

- to explain what zkVMs are
- to explain which problems they solve in Ethereum
- to explain how they would be used in Ethereum

The focus of the book is not:

- to explain every detail of a specific zkVM
- to explain the cryptographic concepts in full detail -- there is enough literature for that

Also, don't use this to open controversial discussions about the future of Ethereum.

## License

Dual License: APACHE + MIT
