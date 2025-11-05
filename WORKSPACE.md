# Bitfold Workspace Architecture

This document describes the modular workspace architecture of the Bitfold networking library.

## Overview

Bitfold has been modularized into a Cargo workspace with 6 distinct packages, each with clear responsibilities and dependencies. This architecture promotes:

- **Separation of Concerns**: Each package has a single, well-defined purpose
- **Independent Development**: Packages can be developed and tested independently
- **Flexible Reuse**: Users can depend on specific packages if they don't need the full library
- **Better Build Parallelization**: Cargo can build packages in parallel
- **Clearer Dependency Boundaries**: Workspace structure enforces architectural layers

## Workspace Structure

```
bitfold/
├── Cargo.toml                          # Workspace root
├── crates/
│   ├── bitfold-core/                   # Core types and configuration
│   ├── bitfold-protocol/               # Protocol logic
│   ├── bitfold-peer/                   # Peer state machine
│   ├── bitfold-utilities/              # Utility functions
│   ├── bitfold-host/                   # Host and session management
│   └── bitfold/                        # Main facade library
├── examples/                           # Usage examples
└── tests/                              # Integration tests
```

## Package Descriptions

### 1. bitfold-core

**Purpose**: Foundation layer providing core types, configuration, and shared utilities.

**Key Components**:
- Configuration types (`Config`)
- Error handling (`ErrorKind`, `DecodingErrorKind`, etc.)
- Protocol constants (MTU, header sizes, version)
- Memory pooling (`PacketAllocator`, `CompressionBufferPool`)
- Shared types (`SharedBytes` for zero-copy slicing)
- Transport abstraction (`Socket` trait)
- Interceptor trait for packet interception

**Dependencies**:
- `byteorder` - Binary serialization
- `rand` - Random number generation
- `tracing` - Logging

**Dependents**: All other packages

### 2. bitfold-protocol

**Purpose**: Pure protocol logic with no I/O dependencies.

**Key Components**:
- Packet types and delivery guarantees
- Protocol command definitions
- Command codec (encoding/decoding)
- Acknowledgment handling
- Congestion control
- Bandwidth management
- Multi-channel abstraction
- Compression (Zlib, LZ4)
- CRC32 checksums

**Dependencies**:
- `bitfold-core` - Core types
- `byteorder` - Binary serialization
- `crc32fast` - Checksums
- `flate2` - Zlib compression
- `lz4` - LZ4 compression
- `tracing` - Logging

**Dependents**: `bitfold-peer`, `bitfold-host`, `bitfold`

**Design Philosophy**: This package is intentionally I/O-free, making it fully testable and reusable in different contexts.

### 3. bitfold-peer

**Purpose**: Per-peer state machine for managing remote endpoints.

**Key Components**:
- Peer state machine (`Peer`)
- Command queue for batching
- Flow control (sliding window)
- Bandwidth throttling
- Fragment reassembly
- Per-channel state tracking
- Unsequenced duplicate detection
- PMTU (Path MTU) discovery
- Connection statistics

**Dependencies**:
- `bitfold-core` - Core types
- `bitfold-protocol` - Protocol logic
- `tracing` - Logging

**Dependents**: `bitfold-host`, `bitfold`

**Design Philosophy**: Each peer is independent with no shared mutable state between peers.

### 4. bitfold-utilities

**Purpose**: Optional utility functions for DNS and address operations.

**Key Components**:
- DNS hostname resolution
- Reverse DNS lookup
- IP string parsing and formatting

**Dependencies**:
- `dns-lookup` - DNS operations
- `socket2` - Socket utilities

**Dependents**: `bitfold-host`, `bitfold`

**Design Philosophy**: Separated to keep the core library dependency-free and allow users to opt-in to these utilities.

### 5. bitfold-host

**Purpose**: Socket I/O and session management.

**Key Components**:
- `Host` - High-level UDP socket API
- `SessionManager` - Manages multiple peer sessions
- Session lifecycle management
- Event emission (`SocketEvent`)
- Manual and background polling modes
- Throughput monitoring
- Time utilities

**Dependencies**:
- `bitfold-core` - Core types
- `bitfold-protocol` - Protocol logic
- `bitfold-peer` - Peer state machine
- `bitfold-utilities` - Address utilities
- `socket2` - Low-level socket operations
- `crossbeam-channel` - Multi-producer channels
- `tracing` - Logging

**Dependents**: `bitfold`

**Design Philosophy**: This is the I/O layer that brings together all other packages to provide a complete networking solution.

### 6. bitfold (Main Facade)

**Purpose**: Public API facade that re-exports commonly used types.

**Key Components**:
- Re-exports from all workspace packages
- Convenience prelude module
- Examples and integration tests

**Dependencies**: All workspace packages

**Dependents**: User applications

**Design Philosophy**: Provides a clean, stable API surface while maintaining modularity internally.

## Dependency Graph

```
bitfold (facade)
├── bitfold-host
│   ├── bitfold-peer
│   │   ├── bitfold-protocol
│   │   │   └── bitfold-core
│   │   └── bitfold-core
│   ├── bitfold-protocol
│   │   └── bitfold-core
│   ├── bitfold-utilities
│   └── bitfold-core
└── [all packages via re-exports]
```

## Layered Architecture

The workspace enforces a strict layered architecture:

```
┌─────────────────────────────────────────┐
│  APPLICATION LAYER                      │
│  (User code using bitfold)              │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│  BITFOLD (Facade)                       │
│  Re-exports public API                  │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│  HOST LAYER (bitfold-host)              │
│  - Socket I/O                           │
│  - Session management                   │
│  - Event coordination                   │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│  PEER LAYER (bitfold-peer)              │
│  - Per-peer state machine               │
│  - Command queuing                      │
│  - Flow control                         │
│  - Fragment reassembly                  │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│  PROTOCOL LAYER (bitfold-protocol)      │
│  - Command encoding/decoding            │
│  - Acknowledgments                      │
│  - Congestion control                   │
│  - Compression                          │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│  CORE LAYER (bitfold-core)              │
│  - Configuration                        │
│  - Error types                          │
│  - Constants                            │
│  - Memory pooling                       │
└─────────────────────────────────────────┘
```

**Utilities** are a separate concern that can be used by any layer.

## Usage Patterns

### Full Library Usage (Recommended)

```toml
[dependencies]
bitfold = "0.1.2"
```

```rust
use bitfold::{Host, Packet, SocketEvent, DeliveryGuarantee};

let mut host = Host::bind_any().unwrap();
// ... use the library
```

### Selective Package Usage

If you only need protocol logic without I/O:

```toml
[dependencies]
bitfold-core = "0.1.2"
bitfold-protocol = "0.1.2"
```

```rust
use bitfold_protocol::{Packet, DeliveryGuarantee};
use bitfold_core::Config;
```

### Module Re-exports

The main `bitfold` crate re-exports all workspace packages:

```rust
use bitfold::core::Config;
use bitfold::protocol::Packet;
use bitfold::peer::Peer;
use bitfold::host::Host;
use bitfold::utilities::resolve_host;
```

## Building the Workspace

### Build all packages:
```bash
cargo build --workspace
```

### Build a specific package:
```bash
cargo build -p bitfold-core
cargo build -p bitfold-protocol
```

### Run tests:
```bash
cargo test --workspace
```

### Run examples:
```bash
cargo run --example server
cargo run --example client
```

## Development Workflow

1. **Making changes to a package**: Edit files in `crates/{package-name}/src/`
2. **Testing changes**: `cargo test -p {package-name}`
3. **Checking the entire workspace**: `cargo check --workspace`
4. **Formatting code**: `cargo fmt --all`
5. **Linting**: `cargo clippy --workspace --all-targets`

## Migration Notes

### For Existing Code

Existing code using the monolithic `bitfold` crate will continue to work without changes:

```rust
// Old code (still works)
use bitfold::{Host, Packet, SocketEvent};
```

The workspace modularization is an internal refactoring that doesn't affect the public API.

### Import Path Changes

If you were using internal modules directly (not recommended), update imports:

```rust
// Old (monolithic crate)
use bitfold::core::Config;
use bitfold::protocol::Packet;

// New (workspace)
use bitfold::core::Config;  // Still works via re-exports
use bitfold::protocol::Packet;  // Still works via re-exports

// Or use workspace packages directly
use bitfold_core::Config;
use bitfold_protocol::Packet;
```

## Benefits of This Architecture

1. **Clearer Boundaries**: Package boundaries enforce architectural layers
2. **Testability**: Each package can be tested independently
3. **Build Performance**: Cargo can parallelize builds across packages
4. **Selective Usage**: Users can depend on only what they need
5. **Independent Evolution**: Packages can evolve at different rates
6. **Better Documentation**: Each package has focused documentation
7. **Reduced Coupling**: Circular dependencies are impossible
8. **Easier Maintenance**: Smaller, focused packages are easier to understand

## Future Enhancements

Potential future improvements enabled by this architecture:

1. **Feature Flags**: Enable/disable packages via features
2. **Alternative Implementations**: Swap out packages (e.g., different protocol codecs)
3. **Plugin System**: Users can provide custom implementations of traits
4. **WebAssembly Support**: Build only core + protocol for WASM targets
5. **No-std Support**: Make core + protocol no-std compatible
6. **Versioning**: Version packages independently if needed

## Conclusion

The modular workspace architecture maintains Bitfold's excellent performance and reliability while improving code organization, testability, and flexibility. The public API remains stable, ensuring a smooth transition for existing users.
