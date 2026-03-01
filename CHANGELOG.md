# AbzuNet Changelog
**SynthicSoft Labs · February 2026**

---

## v2.0.3 — LoRa Transport + Autonomous Node Reward System
*February 2026*

v2.0.3 introduces two major systems that together complete AbzuNet as both a technical and economic internet replacement: LoRa radio transport (Mode A via Meshtastic) and the Autonomous Node Reward System (ANRS) for continuous, automatic ABZU token emission.

### Added: LoRa Transport Layer

- `transport/fragment.rs` — Packet fragmentation / reassembly for LoRa's 237-byte MTU
- `transport/lora_meshtastic.rs` — Meshtastic serial bridge (Mode A): ride the existing global Meshtastic mesh
- `transport/lora.rs` — libp2p Transport trait implementation with Noise encryption inherited from swarm
- Automatic EU 868 MHz / 433 MHz duty cycle enforcement (10% per rolling hour)
- Auto-detection of Meshtastic hardware on common serial ports at startup
- Airtime estimation formula accounting for spreading factor and bandwidth

**Supported hardware:**
- Heltec LoRa 32 v3 (ESP32-S3 + SX1262, ~$25)
- TTGO T-Beam v1.2 (ESP32 + SX1276 + GPS, ~$35)
- RAK WisBlock 4631 (nRF52840 + SX1262, ~$40)
- Any Meshtastic-compatible LoRa hardware with USB serial

### Added: Autonomous Node Reward System (ANRS)

- `anrs/score.rs` — 5-dimensional ServiceScore: uptime 20%, storage 30%, serving 25%, routing 15%, LoRa 10%
- `anrs/challenge.rs` — Storage challenge-response via cryptographic proof of data possession using BLAKE3
- `anrs/epoch.rs` — Epoch manager: 1-hour epochs, halving schedule, automatic on-chain reporting
- `anrs/balance.rs` — Local ABZU balance tracking with deterministic Arbitrum address derivation
- `EpochActivityCounters` — Zero-overhead atomic counters wired into gateway and swarm

**ABZU Token Economics:**
- Total supply: 1,000,000,000 ABZU (fixed forever)
- Node allocation: 700,000,000 (70%) emitted over ~20 years
- Halving: every 17,520 epochs (2 years)
- Chain: Arbitrum (low gas, fast finality)
- Address: `keccak256(ed25519_pubkey)[12..]` — derived from node identity, no wallet setup required
- Withdrawal: `./abzu-node withdraw [address]`

### Added: Smart Contracts

- `contracts/src/AbzuToken.sol` — ERC-20 ABZU with burn-from mechanism
- `contracts/src/AbzuNodeRewards.sol` — Epoch distribution with halving schedule
- `contracts/src/AbzuDataCredits.sol` — Non-transferable usage credits, burn-and-mint equilibrium

### Added: New Gateway Endpoints

- `GET  /anrs/status` — Current ServiceScore breakdown and epoch earnings
- `GET  /anrs/balance` — ABZU balance (pending + confirmed)
- `POST /anrs/withdraw` — Withdraw to any Ethereum-compatible address
- `GET  /anrs/history` — Epoch-by-epoch earning history
- `GET  /lora/status` — LoRa hardware status, signal quality, relay count
- `GET  /lora/peers` — Peers discovered via LoRa radio

### Changed

- `config.rs` — Added `[lora]` and `[anrs]` sections with sane defaults
- `abzu.toml.example` — Updated with full v2.0.3 configuration reference
- `Cargo.toml` — Added `tokio-serial = "6.0"` and `sha3 = "0.10"` dependencies

### Architecture Notes

- LoRa transport is connectionless. Virtual connections are per-peer NodeID.
- The DTN layer is a natural fit for LoRa's intermittent, low-bandwidth nature — messages store-and-forward automatically across radio links.
- ANRS does NOT require blockchain configuration to run. Scoring and local balance tracking work fully offline. On-chain reporting activates when `arbitrum_rpc` is set.
- The LoRa 10% score weight directly incentivizes physical mesh expansion — nodes with LoRa hardware earn more than WiFi-only nodes by design.
- Mode B (direct SX127x/SX126x without Meshtastic firmware) is planned for v2.1.

---

## v2.0.2 — Production Network Stack
*February 2026*

v2.0.2 completed the full production network stack — every protocol component was wired end-to-end, the ZK slashing system was fully implemented, and DTN messaging was introduced as AbzuNet's email-equivalent communication layer.

### Added: Network Integration

- `network/swarm.rs` — Complete libp2p swarm event loop with `select!` multi-protocol dispatch
- `SwarmHandle` channel API for safe external interaction with the swarm
- UDP broadcast mesh discovery bridged to swarm dial queue
- Heartbeat publication (15 s interval) via segmented GossipSub topics
- Kademlia bootstrap on startup with periodic re-bootstrap every 5 min

### Added: Kademlia Content Announcement

- `SwarmCommand::ProvideContent` triggers `kad.start_providing()` on every upload
- `SwarmCommand::FindProviders` triggers provider lookup; gateway falls through to DHT on local miss
- GossipSub content announcement on `abzu/content` topic for immediate mesh propagation

### Added: ZK Slashing Circuit

- `zkp/poseidon.rs` — Native BN254 Poseidon hash (`poseidon2`, `poseidon_attempt_hash`)
- `zkp/circuit.rs` — Full R1CS `HeartbeatFailureCircuit` with all 4 constraint categories
- `zkp/prover.rs` — Complete Groth16 prove / verify / serialize lifecycle
- `abzu-circuits/src/poseidon/mod.rs` — Full S-box, MDS, and permutation gadgets
- CLI: `abzu-node zk-setup` for one-time trusted setup

### Added: Blockchain Bridge

- `bridge/mod.rs` — `ContractBridge` with `deposit_escrow`, `get_staked_collateral`, `submit_slash`
- `BridgeConfig` struct with optional signer key for EIP-4337 integration
- Graceful offline mode when no Arbitrum RPC is configured

### Added: ANS DHT Propagation

- `SwarmCommand::DhtPutName` → `kad.put_record()` on every ANS registration
- `SwarmCommand::DhtGetName` → `kad.get_record()` with pending query tracking
- Gateway ANS handlers consult DHT on local cache miss

### Added: DTN Messaging (Email Equivalent)

- `DtnMessage` — Full message type with vector clock, hop count, TTL, reply threading
- `VectorClock` — Merge-correct causal ordering for partitioned networks
- `DtnQueue` — Persistent inbox / outbox / relay queue (sled-backed)
- Routing: local delivery vs. relay queue based on recipient Node ID
- Sync digest protocol for island reconnection handshake
- 30-second relay forwarding loop via GossipSub `abzu/dtn` topic

### Added: Gateway v2.0.2

- Messaging API: `GET` / `POST` / `DELETE /abzu/messages`, `POST /abzu/messages/send`
- ANS handlers propagate to DHT and report source (local vs. dht)
- Content retrieval falls through to DHT provider list on local miss
- Node status endpoint includes DTN queue statistics

### Added: Browser UI v2.0.2

- Full Messages tab: inbox, compose, view, reply, delete
- ANS resolver displays source (local / DHT)
- Peers panel queries live swarm peer list
- DTN compose queues via `abzu/dtn` GossipSub topic

---

## v2.0.1 — Foundation Release
*February 2026*

v2.0.1 established the core AbzuNet node architecture, moving the codebase from a well-specified skeleton of stubs into a buildable, running system. All cryptographic primitives, the EWMA reputation model, the ZK slashing circuit design, and the network layer structure were already specified; v2.0.1 delivered the production implementations for each module.

### Added: Storage Layer

- `storage/chunker.rs` — Full 4 MB BLAKE3 content-addressed chunker with streaming file support and roundtrip reassembly
  - Root CID computed as `BLAKE3(chunk_0 ‖ chunk_1 ‖ … ‖ chunk_n)`
  - `ChunkManifest` captures root CID, chunk count, total bytes, per-chunk hashes and sizes
  - Reassembly verifies each chunk hash before writing; supports out-of-order arrival
- `storage/store.rs` — sled-backed Content-Addressed Store (CAS) with dual-tree design
  - Chunk tree keyed by BLAKE3 hash; manifest tree keyed by root CID
  - API: `store_file_chunks`, `retrieve_file`, `list_root_cids`, `chunk_count`, `size_on_disk`, `delete_file`
- `storage/bao.rs` — Bao-style incremental streaming verifier
  - `BaoStreamVerifier`: validates chunks in any arrival order against known `ChunkManifest`
  - `BaoStreamWriter`: bounded-memory `BTreeMap` buffering with in-order sink writes

### Added: Network Layer

- `network/kademlia.rs` — Kademlia DHT configuration (k=20, 48 hr TTL, 12 hr re-advertisement)
  - Root CID → Kademlia record key encoding; provider value serialization (JSON multiaddrs)
- `network/gossipsub.rs` — 256-way deterministic topic segmentation
  - Heartbeat topic assignment: `abzu/heartbeat/{node_id[0]:02x}` (O(N/256) per topic)
- `network/behaviour.rs` — Composed `AbzuBehaviour` (Kademlia + GossipSub + Identify + mDNS + Ping)

### Added: Gateway & SDK

- `gateway/mod.rs` — Full HTTP gateway: content browse, upload, ANS register/resolve, peer status
- `abzu-sdk/client.rs` — HTTP client: `get`, `upload`, `resolve_name`, `register_name`, `status`, `list`
- `main.rs` — Full CLI (`init` / `run` / `identity`), config loader, all subsystems wired and connected

---

## Version Summary

| Version | Theme | Key Deliverables |
|---------|-------|-----------------|
| v2.0.1 | Foundation Release | BLAKE3 CAS, Kademlia DHT, GossipSub mesh, HTTP gateway, SDK, full CLI |
| v2.0.2 | Production Network Stack | Swarm event loop, ZK slashing (Groth16), blockchain bridge, DTN messaging, browser UI |
| v2.0.3 | LoRa + ANRS | Meshtastic LoRa transport, 5-dimensional node scoring, ABZU tokenomics, Solidity contracts |

---

*AbzuNet · SynthicSoft Labs · February 2026*
