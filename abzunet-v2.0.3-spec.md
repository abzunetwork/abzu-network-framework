# AbzuNet v2.0.3 — Design Specification
## LoRa Transport + Autonomous Node Reward System (ANRS)

*SynthicSoft Labs — February 2026*

---

## Overview

v2.0.3 introduces two major systems that together complete AbzuNet as both a technical and economic internet replacement:

1. **LoRa Transport Layer** — A libp2p transport adapter connecting AbzuNet nodes to LoRa radio hardware, enabling the network to operate over the globally-deployed Meshtastic mesh without internet infrastructure.

2. **Autonomous Node Reward System (ANRS)** — A continuous, automatic token emission model that pays every node operator proportionally for their real contribution to the network. No manual claiming. No mining rigs. Just run a node and earn.

---

## Part 1: LoRa Transport

### 1.1 What We Learned From Research

The RTradeLtd libp2p-lora-transport (Go) proved the concept works but has two weaknesses AbzuNet must avoid:
- No security — data transmitted in cleartext unless manually encrypted
- Arduino serial bridge creates unnecessary hardware dependency

AbzuNet's approach: implement LoRa as a proper libp2p transport with Noise protocol encryption inherited from the swarm layer. Security is automatic — no manual encryption needed because the Noise handshake happens above the transport layer exactly as it does with TCP and QUIC.

### 1.2 Meshtastic Integration Strategy

Meshtastic's packet format:
- Protobuf-encoded `MeshPacket` with raw byte header
- Node IDs = bottom 4 bytes of Bluetooth MAC address
- AES-256-CTR encryption at the channel layer
- Managed flooding with hop limit (default 3)
- Max payload: ~237 bytes per packet (LoRa physical limit)

**The key insight from research:** Meshtastic is a messaging layer, not a raw transport. For AbzuNet we need two modes:

**Mode A — Meshtastic Overlay:** Use Meshtastic's existing network as a transport. AbzuNet packets are wrapped in Meshtastic `PRIVATE_APP` portnum messages. This lets AbzuNet nodes communicate through existing Meshtastic infrastructure immediately. Bandwidth: ~1KB/s.

**Mode B — Direct LoRa:** Connect directly to LoRa hardware via serial (SX1276/SX1262 chipsets) bypassing Meshtastic firmware entirely. Faster, lower overhead, but requires dedicated hardware not running Meshtastic. Bandwidth: up to 5.5KB/s with SF7.

We implement Mode A first (v2.0.3) since Meshtastic hardware is already globally deployed. Mode B follows in v2.1.

### 1.3 Hardware Support

Target hardware for v2.0.3:
- **Heltec LoRa 32 v3** — ESP32-S3, SX1262, USB-C, $25
- **TTGO T-Beam v1.2** — ESP32, SX1276, GPS, $35
- **RAK WisBlock 4631** — nRF52840, SX1262, modular, $40
- **Raspberry Pi + dragino LoRa HAT** — SX1276, GPIO, $45

Connection: USB serial at 115200 baud. The node detects LoRa hardware on startup by scanning serial ports for Meshtastic device handshake.

### 1.4 Transport Implementation

```rust
// abzu-node/src/transport/lora.rs

use libp2p_core::{
    transport::{ListenerId, TransportEvent},
    Multiaddr, Transport,
};
use tokio::sync::mpsc;
use tokio_serial::SerialPortBuilderExt;

/// LoRa transport multiaddress format:
/// /lora/915mhz/{node_id_hex}
/// /lora/868mhz/{node_id_hex}
/// /lora/433mhz/{node_id_hex}

pub struct LoraTransport {
    /// Serial port connected to LoRa hardware
    port_path:    String,
    /// Operating frequency (region-dependent)
    frequency:    LoraFrequency,
    /// Inbound packet channel from serial reader task
    inbound_rx:   mpsc::Receiver<LoraPacket>,
    /// Outbound packet channel to serial writer task
    outbound_tx:  mpsc::Sender<LoraPacket>,
    /// Local node ID (AbzuNet Ed25519-derived)
    local_id:     [u8; 32],
    /// Active "connections" (virtual, LoRa is connectionless)
    connections:  HashMap<[u8; 32], LoraConnection>,
}

#[derive(Debug, Clone)]
pub struct LoraPacket {
    /// Source AbzuNet Node ID
    pub from:    [u8; 32],
    /// Destination AbzuNet Node ID (or broadcast)
    pub to:      [u8; 32],
    /// Chunk index for fragmented payloads
    pub chunk:   u16,
    /// Total chunks in this message
    pub total:   u16,
    /// Message ID (for reassembly)
    pub msg_id:  u32,
    /// Payload bytes (max 200 bytes per chunk)
    pub payload: Vec<u8>,
}

#[derive(Debug)]
pub enum LoraFrequency {
    Mhz433,  // Asia, some EU
    Mhz868,  // EU (10% duty cycle)
    Mhz915,  // US, Australia
}
```

### 1.5 Packet Fragmentation

LoRa's maximum payload is ~237 bytes. libp2p protocol messages are much larger. AbzuNet implements a fragmentation layer:

```
AbzuNet protocol message (variable size)
    │
    ▼
Fragment into 200-byte chunks
    │
    ▼
Add LoraPacket header (from, to, chunk_idx, total, msg_id)
    │
    ▼
Serialize to ~237 bytes
    │
    ▼
Transmit over LoRa radio
```

Reassembly uses `msg_id` to group fragments. Incomplete messages are held for 60 seconds before being discarded. This is identical to how IP fragmentation works, and AbzuNet's DTN layer already handles message loss gracefully.

### 1.6 DTN + LoRa = Perfect Match

The research confirmed what the architecture already assumed: LoRa's intermittent, low-bandwidth nature is exactly the environment DTN was invented for. The key numbers:

```
LoRa bandwidth (LONG_FAST preset): ~1.07 kbps
AbzuNet DTN message (avg):         ~2KB
Transmission time per message:     ~15 seconds
Multi-hop relay (3 hops):         ~45 seconds end-to-end
```

For text messages, documents, and routing updates — entirely acceptable. For streaming video — not applicable (that's what WiFi is for). AbzuNet's LoRa mode is the emergency internet, the rural internet, the disaster internet. It doesn't need to stream Netflix. It needs to deliver messages and content when nothing else works.

### 1.7 Multiaddress and Config

```toml
# abzu.toml — LoRa configuration
[lora]
enabled = true
port = "/dev/ttyUSB0"          # Linux
# port = "COM3"               # Windows
# port = "/dev/cu.usbserial-0001"  # macOS
frequency = "915mhz"          # us | eu868 | eu433 | au915
mode = "meshtastic"           # meshtastic | direct
spreading_factor = 9          # 7-12, higher = longer range, slower
bandwidth = 125               # kHz: 125, 250, 500
tx_power = 20                 # dBm, max 30 (check local regulations)
hop_limit = 3                 # Meshtastic hop count
```

---

## Part 2: Autonomous Node Reward System (ANRS)

### 2.1 What We Learned From Research

| Project | Model | Key Lesson |
|---|---|---|
| Bitcoin | Proof of Work — burn electricity, earn block reward | Work should be *useful*, not just computationally expensive |
| Filecoin | Proof of Spacetime — prove you're storing data, earn FIL | Correct principle but requires sealing (high compute) and complex deal markets |
| Helium | Proof of Coverage — prove you're providing radio coverage, earn HNT | Closest to AbzuNet's vision — grassroots, hardware-based, automatic |
| Storj | Pay-per-use — earn STORJ for actual data transferred | Too passive; nodes don't earn for being available, only for being used |
| NodeOps | Bond-and-earn — stake tokens to join, earn from revenue | Too capital-intensive; excludes people without capital |

**AbzuNet's synthesis:** Take Helium's grassroots model (anyone with hardware earns automatically), add Filecoin's cryptographic proof of actual storage (not just coverage claims), remove the capital requirements (no bonding, no staking required to start), and make it work without internet using the existing favor score system already built into v2.0.2.

**The critical lesson from Helium's mistakes:** Helium's Proof-of-Coverage was gamed heavily early on — people spoofed locations, clustered hotspots, and earned without providing real coverage. AbzuNet avoids this because:
- Nodes are proven by the content they actually serve, not their claimed location
- BLAKE3 CID verification is unforgeable — you either have the content or you don't
- The existing ZK slashing system penalizes misbehavior cryptographically

### 2.2 AbzuNet Proof of Service (PoS)

AbzuNet introduces **Proof of Service** — a composite proof that a node is:

1. **Online** — Responding to heartbeats (already tracked in v2.0.2 favor system)
2. **Storing** — Holding content CIDs that others need (verifiable via challenge-response)
3. **Serving** — Successfully delivering content to requesting peers (logged in gateway)
4. **Routing** — Relaying DTN messages and DHT queries (tracked in swarm event loop)

Each dimension is already measured by the v2.0.2 EWMA favor system. ANRS turns those measurements into automatic token emission.

### 2.3 ABZU Token Design

```
Name:           ABZU
Total Supply:   1,000,000,000 (1 billion) — fixed forever
Decimals:       8 (like Bitcoin)
Chain:          Arbitrum (existing bridge infrastructure)

Distribution:
  Node Rewards:     70%  (700,000,000 ABZU) — emitted over ~20 years
  SynthicSoft Labs: 15%  (150,000,000 ABZU) — 4-year vesting, funds development
  Ecosystem Fund:   10%  (100,000,000 ABZU) — grants, bounties, community
  Reserve:           5%  (50,000,000 ABZU)  — emergency, liquidity
```

**Why 70% to nodes:** The network IS the nodes. The people running the infrastructure should own the majority of it. Every comparable project (Filecoin: 55% miners, Helium: no premine, Storj: community-first) confirms this is the right principle. AbzuNet goes further than most.

**Why fixed supply:** Predictable economics. No inflation surprises. Like Bitcoin, scarcity is built in from day one. Unlike Bitcoin, the work is useful — you're storing real content and routing real messages.

### 2.4 Emission Schedule

Inspired by Bitcoin's halving but tied to network epochs rather than arbitrary time:

```
Epoch Duration:     1 hour (3600 seconds)
Epoch Reward Pool:  Decreases by halving every 2 years (17,520 epochs)

Year 1:   ~95,890 ABZU per epoch   (~840M ABZU/year total)
Year 2:   ~95,890 ABZU per epoch
Year 3:   ~47,945 ABZU per epoch   (first halving)
Year 4:   ~47,945 ABZU per epoch
Year 5:   ~23,972 ABZU per epoch   (second halving)
...
Year 20+: Tail emission (1% of remaining supply per year)
```

Tail emission after year 20 ensures nodes always have an incentive to participate even when the primary emission approaches zero — preventing the "security budget" problem Bitcoin will eventually face.

### 2.5 Per-Node Reward Calculation

Each epoch, the reward pool is distributed proportionally based on each node's **Service Score**:

```
ServiceScore(node) = w_uptime   × UptimeScore
                   + w_storage  × StorageScore  
                   + w_serving  × ServingScore
                   + w_routing  × RoutingScore
                   + w_lora     × LoraScore      // bonus for LoRa nodes

Weights (sum to 1.0):
  w_uptime   = 0.20
  w_storage  = 0.30   // largest weight — storage is the core service
  w_serving  = 0.25   // rewarding actual data delivery
  w_routing  = 0.15
  w_lora     = 0.10   // LoRa bonus encourages mesh expansion

NodeReward(epoch) = (ServiceScore(node) / Σ ServiceScore(all_nodes)) × EpochPool
```

The LoRa bonus is critical: nodes running LoRa hardware get a 10% weight boost. This directly incentivizes people to deploy LoRa nodes, accelerating global mesh expansion. Every person who buys a $40 LoRa module earns more than a node running only on WiFi.

### 2.6 Score Components

**UptimeScore** — derived directly from existing EWMA favor system:
```
UptimeScore = min(favor_score / 5000, 1.0)  // normalized to [0,1]
```

**StorageScore** — based on verified content held:
```
StorageScore = verified_bytes_held / max_storage_pledged
```
"Verified" means the node successfully responded to a random CID challenge in the last epoch. The challenge picks 10 random CIDs from the DHT and asks the node to prove it holds them. Unforgeable — you either return the correct BLAKE3 chunk or you don't.

**ServingScore** — based on successful content deliveries:
```
ServingScore = chunks_served_this_epoch / (chunks_requested + 1)
```
Logged by the gateway's request handler. Incremented on every successful `GET /abzu/{cid}` that returned actual content (not 404).

**RoutingScore** — based on DTN relay and DHT participation:
```
RoutingScore = (dtn_messages_relayed + dht_queries_answered) / epoch_activity_target
```

**LoraScore** — binary + quality:
```
LoraScore = lora_active ? (1.0 + lora_packets_relayed / lora_target) : 0.0
```

### 2.7 Challenge-Response Storage Verification

To prevent nodes from claiming to store content they don't have, ANRS runs a periodic challenge:

```
Every epoch:
1. Network selects 10 random CIDs from the global DHT
2. Broadcasts challenge to all nodes: "prove you hold CID X"
3. Node must return: BLAKE3(chunk_data) within 30 seconds
4. Correct response = storage verified for that CID
5. Score calculated from % of challenges answered correctly
```

This is a simplified version of Filecoin's Proof of Data Possession (PDP) — same principle, implemented using AbzuNet's existing BLAKE3 infrastructure rather than requiring external proof systems.

### 2.8 Automatic Payment — No Claiming Required

**This is the critical design decision that makes AbzuNet different from everything else.**

In Filecoin and Storj, nodes must manually claim rewards, submit proofs, and pay gas fees. This is friction. It excludes people who don't understand crypto. It requires ETH for gas.

In AbzuNet:

```
Every epoch end:
1. Smart contract calculates ServiceScore for every registered node
2. ABZU tokens are transferred directly to each node's Ed25519-derived address
3. Zero user action required
4. Gas paid by the epoch reward contract itself (self-funding from reserve)
```

The node's Arbitrum wallet address is deterministically derived from its Ed25519 public key:
```
abzu_address = keccak256(ed25519_pubkey)[12..]  // last 20 bytes = Ethereum address
```

The node operator never needs to configure a wallet. Their node IS their wallet. Tokens accumulate automatically. They can check their balance with `./abzu-node balance` and withdraw to any Ethereum-compatible wallet at any time.

### 2.9 Burn-and-Mint Equilibrium

Inspired by Helium's BME model, AbzuNet introduces **Data Credits**:

```
1 Data Credit = $0.001 USD worth of ABZU (price oracle from Chainlink)

To store 1GB on the network:     costs 10 Data Credits
To retrieve 1GB from network:    costs 1 Data Credit  
To register an ANS name:         costs 5 Data Credits
To send a DTN message:           costs 0.1 Data Credits
```

When a user pays in ABZU for Data Credits:
- The ABZU is **burned** (permanently removed from supply)
- Data Credits are minted for that user (non-transferable)
- The burn reduces circulating supply, supporting token value
- The epoch reward pool continues emitting to nodes regardless

This creates a sustainable flywheel:
```
More nodes → better network → more users → more Data Credits purchased
→ more ABZU burned → reduced supply → higher ABZU value
→ more valuable node rewards → more nodes join
```

### 2.10 Node Registration

Nodes register with the on-chain `AbzuNodeRegistry` contract:

```solidity
function registerNode(
    bytes32 node_id,        // BLAKE3(ed25519_pubkey)
    bytes memory pubkey,    // Ed25519 public key
    uint32 storage_pledge,  // GB of storage pledged
    bool lora_enabled       // LoRa hardware present
) external {
    // No stake required. No fee. Just registration.
    require(!nodes[node_id].registered, "Already registered");
    nodes[node_id] = NodeRecord({
        pubkey:         pubkey,
        storage_pledge: storage_pledge,
        lora_enabled:   lora_enabled,
        registered:     true,
        registered_at:  block.timestamp,
        total_earned:   0
    });
    emit NodeRegistered(node_id, storage_pledge, lora_enabled);
}
```

Registration is free. No ETH required. The node calls this automatically on first startup if `[blockchain]` config is set. If the user hasn't configured blockchain, the node still works — it just doesn't earn ABZU until registered.

---

## Part 3: v2.0.3 File Structure

```
abzu-node/src/
├── identity/
├── network/
│   ├── behaviour.rs
│   ├── gossipsub.rs
│   ├── kademlia.rs
│   ├── swarm.rs
│   └── transport.rs
├── transport/                    ← NEW
│   ├── mod.rs                    ← Transport registry
│   ├── lora.rs                   ← LoRa transport implementation
│   ├── lora_meshtastic.rs        ← Meshtastic protocol bridge
│   ├── lora_direct.rs            ← Direct SX126x/SX127x serial
│   └── fragment.rs               ← Packet fragmentation/reassembly
├── storage/
│   ├── chunker.rs
│   ├── store.rs
│   └── bao.rs
├── anrs/                         ← NEW — Autonomous Node Reward System
│   ├── mod.rs                    ← ANRS coordinator
│   ├── score.rs                  ← ServiceScore calculation
│   ├── challenge.rs              ← Storage challenge-response
│   ├── epoch.rs                  ← Epoch timing and reward emission
│   └── balance.rs                ← Local balance tracking
├── favor/
├── zkp/
├── dtn/
├── bridge/
│   ├── mod.rs
│   ├── registry.rs               ← Updated: registerNode with LoRa flag
│   ├── escrow.rs
│   ├── anrs_contract.rs          ← NEW: ANRS reward contract client
│   └── paymaster.rs
└── gateway/
    └── mod.rs                    ← Updated: /balance, /anrs/status endpoints
```

---

## Part 4: New Smart Contracts

### AbzuNodeRewards.sol
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.24;

contract AbzuNodeRewards {
    uint256 public constant EPOCH_DURATION = 3600;      // 1 hour in seconds
    uint256 public constant TOTAL_NODE_SUPPLY = 700_000_000e8; // 70% of 1B
    uint256 public constant HALVING_EPOCHS = 17520;     // 2 years of epochs
    
    uint256 public epochNumber;
    uint256 public lastEpochTime;
    uint256 public totalEmitted;
    
    mapping(bytes32 => NodeScore) public epochScores;
    mapping(bytes32 => uint256) public balances;
    
    struct NodeScore {
        uint32 uptime;
        uint32 storage;
        uint32 serving;
        uint32 routing;
        bool   lora;
    }
    
    /// Called by oracle network at end of each epoch
    /// Scores are computed off-chain by nodes, validated by oracle
    function distributeEpoch(
        bytes32[] calldata node_ids,
        NodeScore[] calldata scores
    ) external onlyOracle {
        require(block.timestamp >= lastEpochTime + EPOCH_DURATION);
        
        uint256 pool = currentEpochPool();
        uint256 totalScore = computeTotalScore(scores);
        
        for (uint i = 0; i < node_ids.length; i++) {
            uint256 nodeScore = computeNodeScore(scores[i]);
            uint256 reward = (nodeScore * pool) / totalScore;
            balances[node_ids[i]] += reward;
            totalEmitted += reward;
        }
        
        epochNumber++;
        lastEpochTime = block.timestamp;
        emit EpochDistributed(epochNumber, pool, node_ids.length);
    }
    
    /// Nodes withdraw to their Ethereum address
    function withdraw(bytes32 node_id, bytes memory signature) external {
        address recipient = nodeIdToAddress(node_id);
        require(verifySignature(node_id, signature), "Invalid signature");
        uint256 amount = balances[node_id];
        balances[node_id] = 0;
        ABZU_TOKEN.transfer(recipient, amount);
    }
    
    function currentEpochPool() public view returns (uint256) {
        uint256 halvings = epochNumber / HALVING_EPOCHS;
        uint256 baseEmission = 95890e8; // Year 1 hourly emission
        return baseEmission >> halvings; // Right shift = halving
    }
}
```

---

## Part 5: Updated Config

```toml
# abzu.toml — v2.0.3 full configuration

[identity]
keyfile = "~/.abzu/identity.key"

[network]
listen_addrs = [
  "/ip4/0.0.0.0/tcp/4001",
  "/ip4/0.0.0.0/udp/4001/quic-v1"
]
bootstrap_peers = []
enable_mdns = true
enable_udp_broadcast = true

[lora]                             # NEW
enabled = false
port = "/dev/ttyUSB0"
frequency = "915mhz"
mode = "meshtastic"
spreading_factor = 9
bandwidth = 125
tx_power = 20
hop_limit = 3

[storage]
data_dir = "~/.abzu/store"
max_storage_gb = 50
island_tag = "default"

[anrs]                             # NEW
enabled = true
storage_pledge_gb = 50            # How much storage you're committing
auto_register = true              # Auto-register with on-chain registry
withdraw_threshold = 100          # Auto-withdraw when balance > 100 ABZU

[blockchain]
arbitrum_rpc = ""
registry_address = "0x..."
rewards_address = "0x..."
escrow_address = "0x..."
paymaster_address = "0x..."

[gateway]
enabled = true
bind_addr = "127.0.0.1:8080"
```

---

## Part 6: New Gateway Endpoints

```
GET  /anrs/status       Node's current service scores and epoch earnings
GET  /anrs/balance      Current ABZU balance on-chain
POST /anrs/withdraw     Withdraw earned ABZU to wallet address
GET  /anrs/history      Epoch-by-epoch earning history
GET  /lora/status       LoRa hardware status, signal quality, packets relayed
GET  /lora/peers        Peers discovered via LoRa
```

---

## Part 7: Implementation Order

**Week 1 — LoRa Transport:**
1. `transport/fragment.rs` — packet fragmentation/reassembly
2. `transport/lora_meshtastic.rs` — Meshtastic serial protocol bridge
3. `transport/lora.rs` — libp2p Transport trait implementation
4. Wire into `network/behaviour.rs` as additional transport
5. Add `/lora/status` gateway endpoint
6. Add `[lora]` config parsing

**Week 2 — ANRS Core:**
1. `anrs/score.rs` — ServiceScore calculation from existing favor data
2. `anrs/challenge.rs` — CID challenge-response protocol
3. `anrs/epoch.rs` — Epoch timing, score aggregation
4. `anrs/balance.rs` — Local balance tracking
5. `anrs/mod.rs` — ANRS coordinator, wired into swarm loop

**Week 3 — Smart Contracts + Bridge:**
1. `AbzuNodeRewards.sol` — Epoch distribution contract
2. `AbzuToken.sol` — ERC-20 ABZU token with burn mechanism
3. `AbzuDataCredits.sol` — Non-transferable Data Credit system
4. Update `bridge/mod.rs` to include ANRS contract client
5. Auto-registration on node startup

**Week 4 — Integration + UI:**
1. New gateway endpoints for ANRS status
2. Embedded browser UI: add "Earnings" tab
3. Add `./abzu-node balance` CLI command
4. Add `./abzu-node withdraw <address>` CLI command
5. End-to-end testing: run 3 nodes, verify reward distribution

---

## Part 8: What This Means for the Vision

When v2.0.3 ships, AbzuNet is complete as an economic system:

**For a person in rural Africa:**
- Buy a $35 Raspberry Pi + $40 LoRa module = $75 total
- Install AbzuNet, configure LoRa, connect to existing Meshtastic mesh
- Immediately begin earning ABZU for providing storage and routing
- Withdraw earnings to any Ethereum-compatible wallet
- No bank account required. No ISP required. No employer required.

**For the network:**
- Every new LoRa node expands the physical mesh
- Every new storage node makes content more available
- Every new routing node makes messages more deliverable
- The grassroots economy self-funds the grassroots infrastructure

**The loop:**
```
Person buys cheap hardware
    → Runs AbzuNet node
        → Earns ABZU automatically
            → ABZU has value because others need Data Credits
                → More people want to earn ABZU
                    → More nodes join
                        → Better network
                            → More users
                                → More Data Credits burned
                                    → ABZU more scarce
                                        → Higher earnings per node
                                            → More people buy cheap hardware
```

This is the grassroots economy. Not a company paying workers. Not a VC funding infrastructure. People funding people, automatically, through cryptographic proof of useful work.

---

*AbzuNet v2.0.3 Design Specification*
*SynthicSoft Labs — February 2026*
*No license. Open source. Free forever.*
