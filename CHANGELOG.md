# AbzuNet Changelog
**SynthicSoft Labs \u00b7 February 2026**

---

## v2.0.3 \u2014 LoRa Transport + Autonomous Node Reward System
*February 2026*

v2.0.3 introduces two major systems that together complete AbzuNet as both a technical and economic internet replacement: LoRa radio transport (Mode A via Meshtastic) and the Autonomous Node Reward System (ANRS) for continuous, automatic ABZU token emission.

### Added: LoRa Transport Layer

- `transport/fragment.rs` \u2014 Packet fragmentation / reassembly for LoRa's 237-byte MTU
- `transport/lora_meshtastic.rs` \u2014 Meshtastic serial bridge (Mode A): ride the existing global Meshtastic mesh
- `transport/lora.rs` \u2014 libp2p Transport trait implementation with Noise encryption inherited from swarm
- Automatic EU 868 MHz / 433 MHz duty cycle enforcement (10% per rolling hour)
- Auto-detection of Meshtastic hardware on common serial ports at startup
- Airtime estimation formula accounting for spreading factor and bandwidth

**Supported hardware:**
- Heltec LoRa 32 v3 (ESP32-S3 + SX1262, ~$25)
- TTGO T-Beam v1.2 (ESP32 + SX1276 + GPS, ~$35)
- RAK WisBlock 4631 (nRF52840 + SX1262, ~$40)
- Any Meshtastic-compatible LoRa hardware with USB serial

### Added: Autonomous Node Reward System (ANRS)

- `anrs/score.rs` \u2014 5-dimensional ServiceScore: uptime 20%, storage 30%, serving 25%, routing 15%, LoRa 10%
- `anrs/challenge.rs` \u2014 Storage challenge-response via cryptographic proof of data possession using BLAKE3
- `anrs/epoch.rs` \u2014 Epoch manager: 1-hour epochs, halving schedule, automatic on-chain reporting
- `anrs/balance.rs` \u2014 Local ABZU balance tracking with deterministic Arbitrum address derivation
- `EpochActivityCounters` \u2014 Zero-overhead atomic counters wired into gateway and swarm

**ABZU Token Economics:**
- Total supply: 1,000,000,000 ABZU (fixed forever)
- Node allocation: 700,000,000 (70%) emitted over ~20 years
- Halving: every 17,520 epochs (2 years)
- Chain: Arbitrum (low gas, fast finality)
- Address: `keccak256(ed25519_pubkey)[12..]` \u2014 derived from node identity, no wallet setup required
- Withdrawal: `./abzu-node withdraw [address]`

### Added: Smart Contracts

- `contracts/src/AbzuToken.sol` \u2014 ERC-20 ABZU with burn-from mechanism
- `contracts/src/AbzuNodeRewards.sol` \u2014 Epoch distribution with halving schedule
- `contracts/src/AbzuDataCredits.sol` \u2014 Non-transferable usage credits, burn-and-mint equilibrium

### Added: New Gateway Endpoints

- `GET  /anrs/status` \u2014 Current ServiceScore breakdown and epoch earnings
- `GET  /anrs/balance` \u2014 ABZU balance (pending + confirmed)
- `POST /anrs/withdraw` \u2014 Withdraw to any Ethereum-compatible address
- `GET  /anrs/history` \u2014 Epoch-by-epoch earning history
- `GET  /lora/status` \u2014 LoRa hardware status, signal quality, relay count
- `GET  /lora/peers` \u2014 Peers discovered via LoRa radio

### Changed

- `config.rs` \u2014 Added `[lora]` and `[anrs]` sections with sane defaults
- `abzu.toml.example` \u2014 Updated with full v2.0.3 configuration reference
- `Cargo.toml` \u2014 Added `tokio-serial = "6.0"` and `sha3 = "0.10"` dependencies

### Architecture Notes

- LoRa transport is connectionless. Virtual connections are per-peer NodeID.
- The DTN layer is a natural fit for LoRa's intermittent, low-bandwidth nature \u2014 messages store-and-forward automatically across radio links.
- ANRS does NOT require blockchain configuration to run. Scoring and local balance tracking work fully offline. On-chain reporting activates when `arbitrum_rpc` is set.
- The LoRa 10% score weight directly incentivizes physical mesh expansion \u2014 nodes with LoRa hardware earn more than WiFi-only nodes by design.
- Mode B (direct SX127x/SX126x without Meshtastic firmware) is planned for v2.1.

---

## v2.0.2 \u2014 Production Network Stack
*February 2026*

v2.0.2 completed the full production network stack \u2014 every protocol component was wired end-to-end, the ZK slashing system was fully implemented, and DTN messaging was introduced as AbzuNet's email-equivalent communication layer.

### Added: Network Integration

- `network/swarm.rs` \u2014 Complete libp2p swarm event loop with `select!` multi-protocol dispatch
- `SwarmHandle` channel API for safe external interaction with the swarm
- UDP broadcast mesh discovery bridged to swarm dial queue
- Heartbeat publication (15 s interval) via segmented GossipSub topics
- Kademlia bootstrap on startup with periodic re-bootstrap every 5 min

### Added: Kademlia Content Announcement

- `SwarmCommand::ProvideContent` triggers `kad.start_providing()` on every upload
- `SwarmCommand::FindProviders` triggers provider lookup; gateway falls through to DHT on local miss
- GossipSub content announcement on `abzu/content` topic for immediate mesh propagation

### Added: ZK Slashing Circuit

- `zkp/poseidon.rs` \u2014 Native BN254 Poseidon hash (`poseidon2`, `poseidon_attempt_hash`)
- `zkp/circuit.rs` \u2014 Full R1CS `HeartbeatFailureCircuit` with all 4 constraint categories
- `zkp/prover.rs` \u2014 Complete Groth16 prove / verify / serialize lifecycle
- `abzu-circuits/src/poseidon/mod.rs` \u2014 Full S-box, MDS, and permutation gadgets
- CLI: `abzu-node zk-setup` for one-time trusted setup

### Added: Blockchain Bridge

- `bridge/mod.rs` \u2014 `ContractBridge` with `deposit_escrow`, `get_staked_collateral`, `submit_slash`
- `BridgeConfig` struct with optional signer key for EIP-4337 integration
- Graceful offline mode when no Arbitrum RPC is configured

### Added: ANS DHT Propagation

- `SwarmCommand::DhtPutName` \u2192 `kad.put_record()` on every ANS registration
- `SwarmCommand::DhtGetName` \u2192 `kad.get_record()` with pending query tracking
- Gateway ANS handlers consult DHT on local cache miss

### Added: DTN Messaging (Email Equivalent)

- `DtnMessage` \u2014 Full message type with vector clock, hop count, TTL, reply threading
- `VectorClock` \u2014 Merge-correct causal ordering for partitioned networks
- `DtnQueue` \u2014 Persistent inbox / outbox / relay queue (sled-backed)
- Routing: local delivery vs. relay queue based on recipient Node ID
- Sync digest protocol for island reconnection handshake
- 30-second relay forwarding loop via GossipSub `abzu/dtn` topic

### Added: Gateway v2.0.2

- Messaging API: `GET` / `POST` / `DELETE /abzu/messages`, `POST /abzu/messages/send`
- ANS handlers propag
