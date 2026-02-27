# AbzuNet v2.0.1

**Post-Internet Resilient Decentralized Storage and Computation Network**

[![License](https://img.shields.io/badge/License-MIT%20OR%20Apache--2.0-blue.svg)](LICENSE)
[![Rust](https://img.shields.io/badge/Rust-1.75%2B-orange.svg)](https://www.rust-lang.org/)
[![Solidity](https://img.shields.io/badge/Solidity-0.8.24-363636.svg)](https://soliditylang.org/)

## Overview

AbzuNet (𝔸) is a mathematically rigorous, self-healing decentralized network designed to survive complete fragmentation of the global internet infrastructure. Built with military-grade cryptography and zero-knowledge verification, AbzuNet provides content-addressed storage, distributed computation, and delay-tolerant networking capabilities that function even when traditional internet connectivity fails.

### Key Features

- **Zero-Knowledge Slashing**: Groth16 proofs over Poseidon hashes verify node misbehavior without revealing sensitive network topology
- **Delay-Tolerant Networking (DTN)**: Vector clock synchronization enables offline operation with eventual consistency when connectivity returns
- **Island-Aware Replication**: Content placement guarantees local copies survive network partitions while maintaining global availability
- **Account Abstraction**: EIP-4337 integration eliminates gas payment friction for node operators
- **Mesh Transport Layer**: UDP broadcast and WebRTC enable operation over local networks without internet routing
- **Streaming Verification**: BLAKE3-based Bao streaming provides incremental cryptographic verification with O(1) memory usage

## Architecture

### Phase 1: Identity & Network Topology

Every node possesses an Ed25519 keypair. The Node ID is derived as:

```
N_id = BLAKE3(ed25519_pubkey)
```

For zero-knowledge circuit compatibility with BN254 scalar field 𝔽_p, the 256-bit Node ID is split:

```
N_high = N_id[0..16] as u128  // First 128 bits
N_low  = N_id[16..32] as u128 // Second 128 bits
```

The network implements Kademlia DHT routing with deterministic GossipSub mesh topology. Heartbeat topics are segmented by the first byte of Node ID to prevent O(N) bandwidth exhaustion:

```
topic = "abzu/heartbeat/{N_id[0]:02x}"
```

### Phase 2: Content-Addressable Storage

Files are chunked into 4MB blocks. The Root CID is computed as:

```
R_cid = BLAKE3(chunk_0 || chunk_1 || ... || chunk_n)
```

Bao streaming enables incremental verification where each chunk is validated against its expected hash before being written to disk. The WASM client enforces strict O(1) memory usage by streaming directly to FileSystemWritableFileStream rather than accumulating in heap buffers.

### Phase 3: Autonomous Favor Engine

Node reputation is computed using a 5-dimensional EWMA over 120 samples (30-minute window):

```
Dimensions: [Uptime, Latency, Bandwidth, Storage, Geo-diversity]
α = 2/121 ≈ 0.0165
Threshold: score ≥ 2500 required for shard hosting
```

Health state transitions follow a deterministic state machine:

```
Alive (0-2 missed) → Suspect (3-4) → Dying (5-7) → AbsoluteDead (8+)
```

### Phase 4: Zero-Knowledge Slashing

When a node reaches AbsoluteDead state, a Groth16 proof is generated with:

**Public Inputs** (x):
- Chain timestamp
- Accused ID Poseidon hash: H_accused = 𝒫_2(N_high, N_low)
- Three attempt Poseidon hashes: H_att_i = 𝒫_6(IP, nonce_i, AS_i, N_high, N_low, t_i)

**Private Witness** (w):
- Reporter IP address
- Accused node coordinates (N_high, N_low)
- Three nonces, BGP AS numbers, timestamps

**R1CS Constraints**:
1. Target binding: H_accused = 𝒫_2(N_high, N_low)
2. BGP diversity: AS_1 ≠ AS_2 ≠ AS_3 (enforced via inverse multiplier gadget)
3. Time bounds: t_chain - 900 ≤ t_1 ≤ t_2 ≤ t_3 ≤ t_chain
4. Attempt binding: H_att_i = 𝒫_6(...) for i ∈ {1,2,3}

### Phase 5: Cryptoeconomics

**CID-Bound Escrow**: Storage payments are locked to specific content to prevent mempool front-running:

```solidity
ID_escrow = keccak256(Payer || R_cid || BlockNumber)
```

Relayers claim escrow by submitting M-of-N Ed25519 signatures from nodes proving replication. Contracts implement strict CEI (Checks-Effects-Interactions) pattern with ReentrancyGuard.

**EIP-4337 Account Abstraction**: Operators pay gas in ABZU tokens via Paymaster, eliminating ETH dependency.

### Phase 6: Delay-Tolerant Networking

When blockchain connectivity fails, operations are queued in sled-backed persistent storage:

```
Queue key: src_nid || lamport_timestamp
```

Vector clock synchronization enables causal consistency. Upon peer reconnection, nodes exchange VectorClock maps, compute causal delta, and merge missing DtnEnvelopes.

### Phase 7: Application Layer

**Local Gateway**: HTTP proxy at 127.0.0.1:8080 translates REST requests to DHT lookups.

**Abzu Name System (ANS)**: Human-readable names map to Ed25519 Domain IDs, which point to mutable Directory CIDs.

**CRDT Synchronization**: Application state propagates via WebSocket to local node, then broadcasts as GossipSub deltas.

## Project Structure

```
abzunet-v2.0.1/
├── abzu-node/          # Core daemon (Rust + Tokio + libp2p)
│   ├── src/
│   │   ├── identity/   # Ed25519 keypairs, Node ID derivation
│   │   ├── network/    # libp2p transports, Kademlia, GossipSub
│   │   ├── storage/    # Content addressing, chunking, Bao streaming
│   │   ├── favor/      # EWMA scoring, health state machine
│   │   ├── zkp/        # Groth16 proving, Poseidon circuits
│   │   ├── dtn/        # Vector clocks, persistent queues
│   │   ├── bridge/     # Ethereum client, EIP-4337 bundler
│   │   └── gateway/    # HTTP server, ANS resolver
│   └── Cargo.toml
├── abzu-sdk/           # Client library (Rust)
│   └── src/
├── abzu-wasm/          # Browser client (WASM)
│   └── src/
├── abzu-circuits/      # Zero-knowledge circuits (arkworks)
│   ├── src/
│   │   ├── poseidon/   # Poseidon hash gadgets
│   │   ├── heartbeat/  # Slash proof circuit
│   │   └── ceremony/   # Trusted setup utilities
│   └── Cargo.toml
├── contracts/          # Solidity smart contracts
│   ├── src/
│   │   ├── AbzuStorageEscrow.sol
│   │   ├── AbzuSlashOracle.sol
│   │   ├── AbzuNodeRegistry.sol
│   │   └── AbzuPaymaster.sol
│   ├── foundry.toml
│   └── script/
├── web/                # Browser gateway UI
│   └── index.html
├── scripts/            # Deployment and utilities
│   ├── generate_poseidon_constants.py
│   ├── deploy_contracts.sh
│   └── genesis_orchestration.sh
└── docs/              # Additional documentation
    ├── SPECIFICATION.md
    ├── CRYPTOGRAPHY.md
    └── DEPLOYMENT.md
```

## Building from Source

### Prerequisites

- Rust 1.75+ with `wasm32-unknown-unknown` target
- Node.js 18+ (for contract deployment)
- Foundry (for Solidity compilation)
- Python 3.9+ (for circuit parameter generation)

### Compilation

```bash
# Clone repository
git clone https://github.com/synthicsoft/abzunet-v2.git
cd abzunet-v2

# Build workspace
cargo build --release

# Run tests
cargo test --all-features

# Build WASM client
cd abzu-wasm
wasm-pack build --target web --release
```

### Smart Contract Deployment

```bash
cd contracts
forge build
forge test

# Deploy to Arbitrum Sepolia testnet
forge script script/Deploy.s.sol --rpc-url $ARBITRUM_RPC --broadcast
```

## Running a Node

```bash
# Generate configuration
./target/release/abzu-node init --config abzu.toml

# Start daemon
RUST_LOG=info ./target/release/abzu-node start --config abzu.toml
```

Configuration example:

```toml
[identity]
keyfile = "~/.abzu/identity.key"

[network]
listen_addrs = ["/ip4/0.0.0.0/tcp/4001", "/ip4/0.0.0.0/udp/4001/quic-v1"]
bootstrap_peers = []
enable_mdns = true
enable_udp_broadcast = true

[storage]
data_dir = "~/.abzu/storage"
max_storage_gb = 100
island_tag = "home-network"

[blockchain]
arbitrum_rpc = "https://sepolia-rollup.arbitrum.io/rpc"
registry_address = "0x..."
escrow_address = "0x..."
paymaster_address = "0x..."

[gateway]
enabled = true
bind_addr = "127.0.0.1:8080"
```

## Security Considerations

This software implements cryptographic protocols and handles private keys. Review the security documentation in `docs/CRYPTOGRAPHY.md` before deploying to production. Key security properties:

- All node-to-node communication is encrypted via Noise protocol
- Content integrity is guaranteed through BLAKE3 cryptographic hashing
- Zero-knowledge proofs reveal no information beyond statement validity
- Private keys never leave local node storage
- Smart contracts undergo formal verification and external audit

## Contributing

AbzuNet is open source software under MIT OR Apache-2.0 dual license. Contributions are welcome following our development guidelines. Please review `CONTRIBUTING.md` before submitting pull requests.

### Development Priorities

Current focus areas for v2.0.x series:

1. Distributed compute layer with WASM runtime integration
2. Enhanced real-time communication over WebRTC data channels
3. Decentralized identity with verifiable credentials
4. Performance optimization for large-scale network operation
5. Comprehensive test coverage including network partition scenarios

## License

Licensed under either of:

- MIT License ([LICENSE-MIT](LICENSE-MIT))
- Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE))

at your option.

## Acknowledgments

Built with [arkworks](https://arkworks.rs/) zero-knowledge cryptography, [libp2p](https://libp2p.io/) networking stack, and [Foundry](https://getfoundry.sh/) smart contract framework.

Developed by SynthicSoft Labs for resilient infrastructure research.

---

**Warning**: This is experimental software under active development. Do not use in production without thorough security review and testing.
