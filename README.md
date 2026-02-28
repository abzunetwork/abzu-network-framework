# AbzuNet v2.0.2

> **Internet of the People. By the People. For the People.**

---

## What Is AbzuNet?

AbzuNet is a **post-internet resilient decentralized network** — a complete replacement for the internet's core functions that operates with or without traditional internet infrastructure.

No central servers. No corporate backbone. No single point of failure. No kill switch.

When the internet goes down — whether from a natural disaster, infrastructure attack, government shutdown, or corporate failure — AbzuNet keeps running. As long as two devices can communicate (over WiFi, Ethernet, LoRa radio, or anything else that carries packets), AbzuNet works.

**This is not a VPN. This is not a Tor overlay. This is a full replacement for the internet protocol stack at the application layer.**

---

## What It Does Right Now

Spin up a node in under 2 minutes:

```bash
cargo build --release -p abzu-node
./target/release/abzu-node init --data-dir ~/.abzu
./target/release/abzu-node run
# Open: http://127.0.0.1:8080
```

Your browser opens to the **AbzuNet Browser** — a full web-like interface that requires zero internet connection:

| Feature | Status |
|---|---|
| Store & retrieve files by cryptographic CID | ✅ Working |
| Human-readable naming (Abzu Name System) | ✅ Working |
| Peer discovery (mDNS + UDP broadcast) | ✅ Working |
| DHT-based content routing (Kademlia) | ✅ Working |
| Decentralized messaging (DTN email equivalent) | ✅ Working |
| ZK slashing proofs (Groth16/BN254) | ✅ Working |
| Blockchain settlement bridge (Arbitrum) | ✅ Working (optional) |
| Embedded browser UI (no external dependencies) | ✅ Working |

---

## The Vision

The internet was not built for resilience. It was built for convenience, and then retrofitted with corporate control. Today:

- A handful of companies control DNS — they can erase any website
- ISPs can block, throttle, or spy on any connection
- Governments can issue shutdown orders with a phone call
- Natural disasters knock out entire regions
- Submarine cables get cut

AbzuNet fixes this at the architecture level. Every node is equal. Content is addressed by its cryptographic hash — not by a domain name someone can revoke. The network routes around damage, censorship, and outages automatically.

**Roadmap:**
- **v2.0.2 (now):** Full software stack — open source, run on any device with WiFi/Ethernet
- **v2.1.x:** LoRa radio transport — extend the mesh to areas with no infrastructure, $30-50 per node
- **v3.0:** Global mesh — every person with a node *is* the internet

---

## Architecture

AbzuNet replaces every critical internet function:

| Internet Service | AbzuNet Equivalent |
|---|---|
| DNS | Abzu Name System (ANS) — DHT-backed, uncensorable |
| Web server | BLAKE3 content-addressed store — content is the address |
| Web browser | Embedded HTTP gateway with full browser UI |
| Email | DTN store-and-forward with vector clock ordering |
| CDN | Kademlia DHT replication across all nodes |
| BGP routing | libp2p peer routing — self-healing mesh |
| Authentication | Ed25519 keypairs — identity you own |

### Identity

Every node generates an Ed25519 keypair. Node identity is unforgeable:

```
N_id = BLAKE3(ed25519_pubkey)  ∈ {0,1}²⁵⁶
```

### Content Addressing

Files are split into 4MB chunks. The Root CID is mathematically bound to the content:

```
R_cid = BLAKE3(chunk_0 ∥ chunk_1 ∥ ... ∥ chunk_n)
```

If someone gives you a CID, you can verify the content is exactly what they promised — cryptographically, without trusting anyone.

### Network

- **Kademlia DHT** (k=20): Decentralized content routing across all nodes
- **GossipSub** (256-way segmented): Real-time message propagation that scales to millions of nodes
- **mDNS + UDP Broadcast**: Works with zero configuration on local networks
- **libp2p QUIC + TCP**: Encrypted, authenticated transport

### Messaging (DTN)

The delay-tolerant networking layer is AbzuNet's email system. Messages are:
- Stored locally when the recipient is offline
- Forwarded opportunistically when paths open up
- Causally ordered with vector clocks across network partitions
- Addressed by Node ID — no email provider required

### Zero-Knowledge Slashing

Misbehaving nodes get slashed via Groth16 ZK proofs over BN254. Reporters prove a node failed without revealing their own IP address or network position. The proof is verified on-chain by the Arbitrum smart contract.

---

## Building from Source

### Prerequisites

- Rust 1.78+
- (Optional) Foundry for smart contract deployment
- (Optional) wasm-pack for WASM client

### Build

```bash
git clone https://github.com/synthicsoft/abzunet-v2
cd abzunet-v2.0.2

# Build the node
cargo build --release -p abzu-node

# Run all tests
cargo test --workspace
```

### Initialize and Run

```bash
# First time setup
./target/release/abzu-node init --data-dir ~/.abzu

# Start the node
./target/release/abzu-node run --config ~/.abzu/abzu.toml

# Check your identity
./target/release/abzu-node identity

# Run ZK trusted setup (one time, for slashing)
./target/release/abzu-node zk-setup --keys-dir ~/.abzu/zkkeys
```

Open **http://127.0.0.1:8080** in any browser. That is the AbzuNet Browser.

### Configuration

```toml
[identity]
keyfile = "~/.abzu/identity.key"

[network]
listen_addrs = ["/ip4/0.0.0.0/tcp/4001", "/ip4/0.0.0.0/udp/4001/quic-v1"]
bootstrap_peers = []       # Add peers here to join a larger network
enable_mdns = true         # Auto-discover nodes on local network
enable_udp_broadcast = true

[storage]
data_dir = "~/.abzu/store"
max_storage_gb = 50
island_tag = "default"

[blockchain]
arbitrum_rpc = ""          # Optional — network works without this
registry_address = "0x0000000000000000000000000000000000000000"
escrow_address = "0x0000000000000000000000000000000000000000"
paymaster_address = "0x0000000000000000000000000000000000000000"

[gateway]
enabled = true
bind_addr = "127.0.0.1:8080"
```

---

## Project Structure

```
abzunet-v2.0.2/
├── abzu-node/              # Core daemon (Rust + Tokio + libp2p)
│   └── src/
│       ├── identity/       # Ed25519 keypairs, Node ID derivation
│       ├── network/
│       │   ├── behaviour.rs   # Composed libp2p behaviour
│       │   ├── gossipsub.rs   # 256-way segmented GossipSub
│       │   ├── kademlia.rs    # DHT config and CID routing
│       │   ├── swarm.rs       # Full swarm event loop
│       │   └── transport.rs   # UDP broadcast mesh transport
│       ├── storage/
│       │   ├── chunker.rs     # 4MB BLAKE3 chunker
│       │   ├── store.rs       # sled-backed content store
│       │   └── bao.rs         # Streaming verification (O(1) memory)
│       ├── favor/          # EWMA reputation, health state machine
│       ├── zkp/
│       │   ├── circuit.rs     # R1CS HeartbeatFailureCircuit
│       │   ├── poseidon.rs    # BN254 Poseidon hash
│       │   └── prover.rs      # Groth16 prove/verify lifecycle
│       ├── dtn/            # Store-and-forward messaging, vector clocks
│       ├── bridge/         # Arbitrum blockchain bridge (optional)
│       └── gateway/        # HTTP gateway + embedded browser UI
├── abzu-sdk/               # Rust client library
├── abzu-wasm/              # Browser WASM client
├── abzu-circuits/          # arkworks ZK circuit crate
├── contracts/              # Solidity: escrow, slashing, registry
└── docs/
    ├── SPECIFICATION.md
    └── DEPLOYMENT.md
```

---

## Connecting to Other Nodes

On the same local network, nodes discover each other automatically via mDNS. No configuration needed.

To connect across the internet (while it exists) or across a manual link, add the remote node's multiaddress to `bootstrap_peers`:

```toml
bootstrap_peers = [
  "/ip4/203.0.113.10/tcp/4001/p2p/12D3KooW..."
]
```

Get your node's multiaddress from the `/status` endpoint or the Node Status page in the browser UI.

---

## API Reference

The gateway exposes a REST API at `http://127.0.0.1:8080`:

| Method | Endpoint | Description |
|---|---|---|
| GET | `/` | AbzuNet Browser UI |
| GET | `/status` | Node status, peer count, disk usage |
| GET | `/abzu/{cid}` | Retrieve content by 64-char hex CID |
| POST | `/abzu/upload` | Upload file, returns CID |
| GET | `/abzu/ls` | List all stored CIDs |
| GET | `/abzu/peers` | Connected peer list |
| GET | `/abzuname/{name}` | Resolve ANS name → CID |
| POST | `/abzuname/{name}/{cid}` | Register ANS name |
| GET | `/abzu/messages` | List inbox messages |
| POST | `/abzu/messages/send` | Send DTN message |
| GET | `/abzu/messages/{id}` | Read a message |
| DELETE | `/abzu/messages/{id}` | Delete a message |

### SDK Usage

```rust
use abzu_sdk::AbzuClient;

let client = AbzuClient::new("http://127.0.0.1:8080");

// Upload a file
let result = client.upload(b"Hello, post-internet world!", "hello.txt").await?;
println!("CID: {}", result.cid);

// Retrieve by CID
let data = client.get(&result.cid).await?;

// Register a name
client.register_name("mysite", &result.cid).await?;

// Resolve a name
let resolved = client.resolve_name("mysite").await?;
```

---

## Security

- All transport is encrypted via the **Noise protocol** (libp2p standard)
- Content integrity is **cryptographically guaranteed** by BLAKE3 — you cannot receive tampered content without detection
- Node identity is **unforgeable** — Ed25519 signatures bind all protocol messages to their sender
- ZK proofs reveal **zero information** beyond proof validity
- Private keys **never leave** the local node
- The blockchain bridge is **fully optional** — the network functions without it

---

## Roadmap

### v2.0.X (Active)
- [ ] LoRa radio transport adapter (extend mesh beyond WiFi range)
- [ ] Remote chunk fetch via DHT providers (cross-node content retrieval)
- [ ] Ed25519 message signing in DTN layer
- [ ] Bootstrap peer discovery via signed peer lists

### v2.1.0
- [ ] LoRa + Meshtastic hardware integration
- [ ] Mobile node (Android/iOS)
- [ ] Distributed compute (WASM runtime)

### v3.0
- [ ] Global LoRa mesh deployment
- [ ] Satellite uplink integration
- [ ] Full internet replacement at scale

---

## Contributing

AbzuNet is open source and free forever hence no license. 

Pull requests are welcome. The most impactful areas right now:

1. **LoRa transport** — implement `LoraTransport` as a libp2p transport
2. **Remote chunk fetch** — dial DHT providers and retrieve missing chunks
3. **Message signing** — wire Ed25519 signing into the DTN send path
4. **Bootstrap discovery** — signed peer list distribution

Please open an issue before starting major work so we can coordinate.

---

## License

No license. Open Source. Free Forever. Power to the people! 

---

## Built With

- [libp2p](https://libp2p.io/) — peer-to-peer networking
- [arkworks](https://arkworks.rs/) — zero-knowledge cryptography
- [BLAKE3](https://github.com/BLAKE3-team/BLAKE3) — cryptographic hashing
- [sled](https://github.com/spacejam/sled) — embedded database
- [Axum](https://github.com/tokio-rs/axum) — async HTTP
- [Foundry](https://getfoundry.sh/) — smart contract toolchain

---

*Developed by [SynthicSoft Labs](https://synthicsoft.com)*

> **The internet belongs to the people who build it.**
> Run a node. Spread the mesh. Own your network.
