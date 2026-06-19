# graphiview

## Project Overview

**Graphiview** is a Rust-based CLI tool designed for architectural code review. It analyzes and compares knowledge graphs generated from codebase snapshots to help developers identify architectural risks before code merges.

## Key Features

- **Graph Differencing**: Computes structural differences between base and head branches
- **Architectural Metrics**: Calculates blast radius, god nodes, community sprawl, and more
- **Command-Line Interface**: Modern CLI using `clap` with multiple subcommands
- **Cross-Language Support**: Designed to work with various languages via AST parsing

## Getting Started

### Prerequisites

- Rust 2021 edition or newer
- Cargo package manager

### Installation

```bash
# Clone the repository
git clone <repository-url>
cd graphiview

# Build the project
cargo build
```

### Usage

```bash
# Analyze a pull request
graphiview pr --base main --head feature/branch

# Compare two graphs
graphiview diff --base graph-main.json --head graph-pr.json

# Calculate blast radius
graphiview affected --file src/main.rs --graph graph.json
```

## Directory Structure

```
graphiview/
├── .gitignore                    # Git ignore patterns
├── Cargo.toml                    # Project manifest
├── README.md                     # Project documentation
├── docs/                         # Existing documentation
│   ├── ARCHITECTURE.md
│   ├── PROBLEM_STATEMENT.md
│   ├── SOLUTION.md
│   ├── USAGE.md
│   ├── AGENT_INTEGRATION.md
│   └── METRICS.md
├── plans/                        # Project plans
│   └── project-setup.md
├── src/
│   ├── main.rs                   # CLI entry point
│   ├── cli.rs                    # CLI argument definitions
│   ├── graph/                    # Graph construction
│   ├── analysis/                 # Analysis modules
│   ├── diff/                     # Graph differencing
│   ├── report/                   # Report generation
│   └── utils/                    # Utility functions
└── tests/                        # Tests
```

## Technology Stack

- **Language**: Rust 2021
- **CLI Framework**: `clap`
- **Graph Processing**: `petgraph`
- **AST Parsing**: `tree-sitter`
- **Serialization**: `serde`
- **Error Handling**: `thiserror`, `anyhow`
- **Async Runtime**: `tokio`
- **Git Operations**: `git2`

## Contributing

Contributions are welcome! Please see our [contributing guidelines](CONTRIBUTING.md) for more information.

## License

This project is licensed under the terms of the [MIT license](LICENSE) or [Apache 2.0 license](LICENSE-APACHE).

## Project Status

This project is currently under active development. The initial project setup has been completed, and core modules are being implemented.

## References

- [Rust CLI Book](https://rust-cli.github.io/book/)
- [petgraph Documentation](https://docs.rs/petgraph/)
- [clap Documentation](https://docs.rs/clap/)
- [tree-sitter Documentation](https://tree-sitter.github.io/tree-sitter/)