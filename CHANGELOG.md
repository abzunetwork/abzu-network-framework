# AbzuNet Changelog

## v2.0.3 — LoRa Transport + Autonomous Node Reward System
*Released: February 2026 — SynthicSoft Labs*

### Added: LoRa Transport Layer
- `transport/fragment.rs` — Packet fragmentation/reassembly for LoRa's 237-byte MTU
- `transport/lora_meshtastic.rs` — Meshtastic serial bridge (Mode A): ride the existing global Meshtastic mesh
- `transport/lora.rs` — libp2p Transport trait implementation with Noise encryption inherited from swarm
- Automatic EU 868MHz/433MHz duty cycle enforcement (10% per rolling hour)
- Auto-detection of Meshtastic hardware on common serial ports
- Airtime estimation formula for LoRa spreading factor and bandwidth

**Supported hardware:**
- Heltec LoRa 32 v3 (ESP32-S3 + SX1262, ~$25)
- TTGO T-Beam v1.2 (ESP32 + SX1276 + GPS, ~$35)
- RAK WisBlock 4631 (nRF52840 + SX1262, ~$40)
- Any Meshtastic-compatible LoRa hardware with USB serial

### Added: Autonomous Node Reward System (ANRS)
- `anrs/score.rs` — 5-dimensional ServiceScore (uptime 20%, storage 30%, serving 25%, routing 15%, LoRa 10%)
- `anrs/challenge.rs` — Storage challenge-response: cryptographic proof of data possession using BLAKE3
- `anrs/epoch.rs` — Epoch manager: 1-hour epochs, halving schedule, automatic on-chain reporting
- `anrs/balance.rs` — Local ABZU balance tracking with deterministic Arbitrum address derivation
- `EpochActivityCounters` — Zero-overhead atomic counters wired into gateway and swarm

**ABZU Token:**
- Total supply: 1,000,000,000 (fixed forever)
- Node allocation: 700,000,000 (70%) emitted over ~20 years
- Halving: every 17,520 epochs (2 years)
- Chain: Arbitrum (low gas, fast finality)
- Address: `keccak256(ed25519_pubkey)[12..]` — derived from node identity, no wallet setup needed
- Withdrawal: CLI command `./abzu-node withdraw [address]`

### Added: Smart Contracts
- `contracts/src/AbzuToken.sol` — ERC-20 ABZU with burn-from mechanism
- `contracts/src/AbzuNodeRewards.sol` — Epoch distribution with halving schedule
- `contracts/src/AbzuDataCredits.sol` — Non-transferable usage credits, burn-and-mint equilibrium

### Added: New Gateway Endpoints
- `GET /anrs/status` — Current ServiceScore breakdown and epoch earnings
- `GET /anrs/balance` — ABZU balance (pending + confirmed)
- `POST /anrs/withdraw` — Withdraw to any Ethereum-compatible address
- `GET /anrs/history` — Epoch-by-epoch earning history
- `GET /lora/status` — LoRa hardware status, signal quality, relay count
- `GET /lora/peers` — Peers discovered via LoRa radio

### Changed
- `config.rs` — Added `[lora]` and `[anrs]` sections with sane defaults
- `abzu.toml.example` — Updated with full v2.0.3 configuration reference
- `Cargo.toml` — Added `tokio-serial = "6.0"` and `sha3 = "0.10"` dependencies

### Architecture Notes
- LoRa transport is connectionless. Virtual connections are per-peer-NodeID.
- The DTN (Delay-Tolerant Networking) layer is a natural fit for LoRa's intermittent,
  low-bandwidth nature — messages store-and-forward automatically across radio links.
- ANRS does NOT require blockchain configuration to run. Scoring and local balance
  tracking work offline. On-chain reporting activates when `arbitrum_rpc` is set.
- The LoRa 10% score weight directly incentivizes physical mesh expansion — nodes
  with LoRa hardware earn more than WiFi-only nodes by design.

---

## v2.0.2 — Production Network Stack
*Previous release — see docs/SPECIFICATION.md*

Full Kademlia DHT, GossipSub mesh, BLAKE3 CAS, ANS, DTN, ZK slashing,
EIP-4337 escrow, embedded browser UI, favor/health scoring system.
