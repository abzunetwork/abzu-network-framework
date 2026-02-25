[README (2).md](https://github.com/user-attachments/files/25545938/README.2.md)
***

# ABZU: A Self-Healing, Zero-Knowledge Verified Decentralized Storage Network
## Formal Protocol Specification — Yellowpaper v1.0

**Adam Rivers** — SynthicSoft Labs / 5yntaxLabs Creations
`abzunetteam@gmail.com`

***

> *"The water beneath the world."*

***

## Abstract

We present **Abzu** $\mathbb{A}$, a peer-to-peer decentralized storage and computation network engineered for extreme fault tolerance, Byzantine resistance, and browser-native edge execution. Abzu abandons centralized orchestration in favor of an autonomous, self-healing architecture: node failures are cryptographically proven via Zero-Knowledge Succinct Non-Interactive Arguments of Knowledge (zk-SNARKs) under the Groth16 proof system over the BN254 elliptic curve, prompting autonomous replication governed by an EIP-4337 v0.7 Account Abstraction paymaster layer and settled on Arbitrum One.

This paper defines the Abzu protocol in full: its BLAKE3 content-addressable storage model $\mathcal{S}$, Kademlia-based DHT topology $\mathcal{K}$, Exponentially Weighted Moving Average peer-scoring engine $\mathcal{F}$, Groth16 zero-knowledge slashing circuit $\mathcal{C}$, Poseidon hash gadgets $\mathcal{P}$ over $\mathbb{F}_p$, and its WASM-optimized $O(1)$-heap streaming architecture $\mathcal{W}$. [docs](https://docs.rs/fastcrypto-zkp/latest/$fastcrypto_zkp$/bn254/poseidon/fn.$poseidon_bytes$.html)

***

## Conventions

We adopt typographic conventions modeled on the Ethereum Yellow Paper. [ethereum.github](https://ethereum.github.io/yellowpaper/paper.pdf)

Scalar values are denoted in lowercase italic: $n$, $t$, $k$.

Sets and spaces are denoted in blackboard bold: $\mathbb{F}_p$, $\mathbb{G}_1$, $\mathbb{B}$.

Functions and algorithms are denoted in calligraphic: $\mathcal{H}$, $\mathcal{P}$, $\mathcal{F}$.

Protocol-level state structures are denoted in bold Greek: $\boldsymbol{\sigma}$, $\boldsymbol{\pi}$.

Byte sequences are denoted with double brackets: $\llbracket \cdot \rrbracket$.

Concatenation of byte sequences is denoted $\parallel$.

The BLAKE3 hash function is $\mathcal{H}_{\mathtt{B3}} : \mathbb{B}^* \rightarrow \mathbb{B}^{32}$.

The Keccak-256 hash function is $\mathcal{H}_{\mathtt{K}} : \mathbb{B}^* \rightarrow \mathbb{B}^{32}$.

The Poseidon hash function of arity $k$ over $\mathbb{F}_p$ is $\mathcal{P}_k : \mathbb{F}_p^k \rightarrow \mathbb{F}_p$.

***

## 1. Network Topology & Cryptographic Identity

### 1.1 The Scalar Field $\mathbb{F}_p$

The BN254 (alt\_bn128) elliptic curve defines a scalar field $\mathbb{F}_p$ with prime modulus: [docs.pantherprotocol](https://docs.pantherprotocol.io/docs/learn/cryptographic-primitives/poseidon)


$$
p = 21888242871839275222246405745257275088548364400416034343698204186575808495617
$$
All ZK circuit arithmetic is performed modulo $p$. Since $p < 2^{254}$, any 128-bit unsigned integer maps safely into $\mathbb{F}_p$ without reduction, which is the basis of the Node ID split in §1.2.

### 1.2 Node Identity

Each Abzu node $\mathcal{N}$ generates an **Ed25519** keypair $(K_{priv}, K_{pub})$ at genesis. The cryptographic identity is derived as:


$$
N_{id} = \mathcal{H}_{\mathtt{B3}}(K_{pub}) \in \mathbb{B}^{32}
$$
To eliminate silent field-overflow collisions during R1CS synthesis, $N_{id}$ is structurally partitioned along the 16-byte boundary:


$$
N_{high} = N_{id}\llbracket 0..15 \rrbracket, \quad N_{low} = N_{id}\llbracket 16..31 \rrbracket
$$
Both halves map injectively into $\mathbb{F}_p$, since $2^{128} \ll p \approx 2^{254}$. The Poseidon identity commitment used in the ZK circuit is then:


$$
H_{accused} = \mathcal{P}_2(N_{high},\ N_{low})
$$
**Endianness invariant.** $N_{high}$ is encoded as a 128-bit big-endian unsigned integer before field element construction. In the Rust implementation this is enforced via `u128::from_be_bytes`. In JavaScript, `BigInt` constructs from the same big-endian byte array. The on-chain Solidity verifier deserializes proof elements as 32-byte big-endian words consistent with the EVM's native encoding. Violation of this invariant causes every proof to fail at `HeartbeatVerifier.sol:verifyProof()` with no revert message.

### 1.3 Peer Identification

The libp2p PeerId $\mathtt{pid}$ is derived from the Ed25519 public key via Protobuf-encoded multihash:


$$
\mathtt{pid} = \text{Multihash}(\text{SHA2-256}, \text{Protobuf}(K_{pub}))
$$
The PeerId is distinct from $N_{id}$ and is used exclusively for transport-layer routing. $N_{id}$ is used exclusively for ZK circuit inputs and Favor score indexing.

### 1.4 Kademlia DHT Topology

Abzu employs a **Kademlia** DHT $\mathcal{K}$ for peer discovery and content routing.  The XOR distance metric on the 256-bit ID space defines the network topology: [ethereum.github](https://ethereum.github.io/yellowpaper/paper.pdf)


$$
d(A, B) = A_{id} \oplus B_{id} \in \{0, \ldots, 2^{256}-1\}
$$
Each node maintains $k$-buckets (default $k = 20$) for peers at logarithmic XOR distance intervals. Routing table refresh occurs on a 300-second interval to mitigate **Kademlia routing table rot** under long-tail uptime distributions.

**Shard location.** The DHT stores a content routing record $\mathtt{Record}(R_{cid}) \rightarrow \{N_{id}^{(1)}, \ldots, N_{id}^{(r)}\}$ mapping each Root CID to the set of nodes currently holding replicas. This record is populated by storing nodes at upload time and expires after $\tau_{ttl} = 86400$ seconds unless refreshed.

### 1.5 Targeted GossipSub Mesh

To prevent $O(N)$ bandwidth exhaustion during heartbeat broadcasts across large networks, Abzu uses **deterministic GossipSub topic segmentation**. Each node publishes heartbeats only to a subnet derived from its identity prefix:


$$
\mathtt{Topic}_{hb} = \texttt{"abzu/heartbeat/"} \parallel \text{hex}(N_{id}\llbracket 0 \rrbracket)
$$
This partitions the 256 possible prefix bytes into 256 independent GossipSub topics, bounding each node's heartbeat audience to approximately $\frac{N}{256}$ peers. At $N = 100{,}000$ nodes the effective broadcast diameter is $\approx 390$ peers per topic — sufficient for liveness detection while remaining bandwidth-linear in mesh degree rather than network size.

The GossipSub mesh is configured with parameters $(D, D_{low}, D_{high}) = (6, 4, 12)$ providing a 4-regular fault-tolerant mesh with flood fill under partition.

***

## 2. Content-Addressable Storage

### 2.1 The Storage State $\boldsymbol{\sigma}_{\mathcal{S}}$

The global storage state is defined as a mapping from Root CIDs to shard metadata:


$$
\boldsymbol{\sigma}_{\mathcal{S}} : \mathbb{B}^{32} \rightarrow \Sigma_{meta}
$$
where each metadata record $\Sigma_{meta}$ is the tuple:


$$
\Sigma_{meta} = (R_{cid},\ r,\ \{N_{id}^{(i)}\}_{i=1}^{r},\ \tau_{deadline},\ \theta)
$$
with $r$ the current replication count, $\{N_{id}^{(i)}\}$ the set of hosting nodes, $\tau_{deadline}$ the escrow expiry, and $\theta \in \{0,1,2,3\}$ the Security Tier.

### 2.2 BLAKE3 Content Addressing

Files are split into chunks $\{C_0, C_1, \ldots, C_{k-1}\}$ of constant size $\Delta_c = 2^{22}$ bytes (4 MiB). BLAKE3's internal Merkle tree provides parallelized, incremental evaluation. [github](https://github.com/ethereum/yellowpaper)

The Root Content ID $R_{cid}$ is defined:


$$
R_{cid} = \mathcal{H}_{\mathtt{B3}}(C_0 \parallel C_1 \parallel \cdots \parallel C_{k-1})
$$
Each chunk is individually addressed by its chunk CID:


$$
\mathtt{cid}_i = \mathcal{H}_{\mathtt{B3}}(C_i)
$$
The manifest $\mathcal{M}$ is the ordered tuple:


$$
\mathcal{M} = (R_{cid},\ k,\ \Delta_c,\ |D|,\ \theta,\ \langle \mathtt{cid}_0, \ldots, \mathtt{cid}_{k-1} \rangle)
$$
### 2.3 Security Tier Taxonomy

| Tier | $\theta$ | Stream Safety | Verification Strategy |
|------|-----------|--------------|----------------------|
| Public | 0 | Streaming permitted | Rolling root hash post-verification |
| Protected | 1 | Streaming permitted | Rolling root hash post-verification |
| Private | 2 | Blocked until verified | Full pre-assembly, root verification first |
| Dark | 3 | Blocked until verified | Full pre-assembly, root verification first |

For tiers $\theta \in \{2, 3\}$, the retrieve function is:


$$
\text{Retrieve}(R_{cid}) = \begin{cases} \bigoplus_{i=0}^{k-1} C_i & \text{if } \mathcal{H}_{\mathtt{B3}}\left(\bigparallel_{i=0}^{k-1} C_i\right) = R_{cid} \\ \perp & \text{otherwise} \end{cases}
$$
### 2.4 Bao-Encoded Streaming Integrity

For tiers $\theta \in \{0, 1\}$, the **Bao streaming protocol** enables incremental verification. Each chunk $C_i$ is verified against its subtree hash in $\mathcal{M}$ before being forwarded to the application layer. The streaming invariant is:


$$
\forall i \in [0, k{-}1] :\ \mathcal{H}_{\mathtt{B3}}(C_i) = \mathtt{cid}_i \implies \text{forward}(C_i)\ \text{else}\ \text{drop\_peer}()
$$
This eliminates the **Poison Pill attack**: a malicious peer cannot inject corrupt chunks that survive to disk, because verification occurs at chunk boundary, not file boundary.

### 2.5 Storage Quota Enforcement

Each node $\mathcal{N}$ declares a storage quota $Q_{bytes}$ at registration. The used byte counter $q$ is maintained as a lock-free atomic 64-bit integer, updated atomically inside the DashMap insertion closure to prevent the **TOCTOU race** between quota check and chunk write:


$$
q_{\text{new}} = q_{\text{old}} + |C_i| \leq Q_{bytes}
$$
If $q_{\text{new}} > Q_{bytes}$, the chunk is rejected with a `QuotaExceeded` error before any disk write occurs.

***

## 3. The Autonomous Favor Engine $\mathcal{F}$

### 3.1 EWMA Mathematical Model

The Favor engine applies an **Exponentially Weighted Moving Average** to each scoring dimension $k \in K$. For a stream of observations $\{x_0, x_1, \ldots\}$ the smoothed estimate $S_t$ at time step $t$ is: [github](https://github.com/ethereum/yellowpaper)


$$
S_t = \alpha \cdot x_t + (1 - \alpha) \cdot S_{t-1}, \quad S_0 = x_0
$$
The smoothing factor $\alpha$ is calibrated to a **30-minute rolling window** at 15-second heartbeat intervals ($N = 120$ samples):


$$
\alpha = \frac{2}{N + 1} = \frac{2}{121} \approx 0.01653
$$
The effective half-life of an observation is:


$$
t_{1/2} = \frac{-\ln 2}{\ln(1 - \alpha)} \approx 41.5 \text{ samples} \approx 10.4 \text{ minutes}
$$
This gives meaningful responsiveness to short-term failures while filtering transient noise from BGP reroutes and ISP congestion events.

### 3.2 Scoring Dimensions

Let $\mathbf{p}_{95}^{bw}$ denote the 95th-percentile observed bandwidth across all peers (the network baseline). The normalized per-dimension samples at each heartbeat interval are:


$$
x_{uptime} = \begin{cases} 1.0 & \text{heartbeat received} \\ 0.0 & \text{heartbeat missed} \end{cases}
x_{latency} = 1.0 - \min\!\left(1.0,\ \frac{\ell_{ms}}{2000}\right)
x_{bandwidth} = \min\!\left(1.0,\ \frac{b_{MB/s}}{\mathbf{p}_{95}^{bw}}\right)
x_{storage} = \frac{\text{chunks held}}{\text{chunks assigned}}
x_{geo} \in [0, 0.1] \quad \text{(static geographic diversity bonus)}
$$
### 3.3 Composite Favor Score

The final Favor Score $\mathcal{F}(\mathcal{N})$ is a weighted linear combination clamped to basis points scale $[0, 10000]$:


$$
\mathcal{F}(\mathcal{N}) = \left\lfloor 10000 \cdot \sum_{k \in K} w_k S_k \right\rfloor
$$
with weight vector $\mathbf{w} = (w_{up}, w_{lat}, w_{bw}, w_{st}, w_{geo}) = (0.35, 0.20, 0.20, 0.20, 0.05)$ satisfying $\sum w_k = 1.0$.

The **minimum eligibility threshold** for shard assignment and relayer bounty receipt is:


$$
\mathcal{F}_{min} = 2500 \quad (25\text{th percentile})
$$
### 3.4 Genesis Exemption Window

New nodes cannot accrue sufficient observations to exceed $\mathcal{F}_{min} = 2500$ before their EWMA converges. A **genesis exemption window** of $E = 80{,}640$ Arbitrum blocks (≈ 14 days at 15-second block time) is granted, during which all shard eligibility checks bypass the Favor threshold:


$$
\text{eligible}(\mathcal{N}) = \begin{cases} \top & \text{if } b_{current} - b_{genesis} < E \\ \mathcal{F}(\mathcal{N}) \geq \mathcal{F}_{min}\ \wedge\ n_{samples} \geq 80{,}640 & \text{otherwise} \end{cases}
$$
***

## 4. Autonomous Death Detection & Health State Machine

### 4.1 The Health State Machine

Each node $\mathcal{N}_i$ maintains a local peer state machine for every observed peer $\mathcal{N}_j$:


$$
\boldsymbol{\delta}_j = (\mathtt{health},\ m_j,\ t_{last},\ \mathcal{F}_j)
$$
where $m_j$ is the missed heartbeat counter and $t_{last}$ is the timestamp of the last received heartbeat.

State transitions are defined by the missed beat thresholds:


$$
\mathtt{health}_j = \begin{cases} \mathtt{Alive} & m_j \in [0, 2] \\ \mathtt{Suspect} & m_j \in [3, 4] \\ \mathtt{Dying} & m_j \in [5, 7] \\ \mathtt{AbsoluteDead} & m_j \geq 8 \end{cases}
$$
Transitions to $\mathtt{AbsoluteDead}$ emit a death event $\mathcal{E}_{dead}(\mathcal{N}_j)$ that triggers the ZK slashing pipeline (§5).

**Timing.** At 15-second intervals and 8 missed beats, the minimum time to declare absolute death is $8 \times 15 = 120$ seconds. This provides a 2-minute grace period for transient outages while remaining responsive to genuine Byzantine failures.

### 4.2 TOCTOU Safety in Health Polling

The health polling loop collects all peer IDs into an owned `Vec<PeerId>` before acquiring any per-peer DashMap lock, and releases all locks before awaiting the death channel send:


$$
\forall j :\ \text{lock}(\boldsymbol{\delta}_j) \rightarrow \text{read\_and\_update} \rightarrow \text{release\_lock} \rightarrow \text{send\_if\_dead}
$$
This eliminates the deadlock risk from holding a DashMap `RefMut` across an `.await` point.

***

## 5. Zero-Knowledge Fault Detection

### 5.1 Proof System Selection

Abzu employs **Groth16** — a preprocessing zk-SNARK that achieves constant proof size and pairing-based verification.  The proof $\boldsymbol{\pi}$ over the BN254 curve is: [blog.lambdaclass](https://blog.lambdaclass.com/groth16/)


$$
\boldsymbol{\pi} = (A \in \mathbb{G}_1,\ B \in \mathbb{G}_2,\ C \in \mathbb{G}_1)
$$
The serialized proof is exactly **256 bytes** (two $\mathbb{G}_1$ affine points at 64 bytes each, one $\mathbb{G}_2$ affine point at 128 bytes). On-chain verification cost is approximately **250,000 gas** on Arbitrum One.

The security of Groth16 rests on the **Knowledge of Exponent** (KoE) assumption and the hardness of the **discrete logarithm** on BN254, providing 128-bit security. [alinush.github](https://alinush.github.io/groth16)

### 5.2 The Poseidon Hash Function $\mathcal{P}$

Abzu uses the **Poseidon** hash function, designed for efficient algebraic circuit representation over large prime fields.  The permutation operates on a state vector of width $t$ over $\mathbb{F}_p$ with the HADES construction: [research.aragon](https://research.aragon.org/poseidon-noir.html)


$$
\mathcal{P}_t : \mathbb{F}_p^t \rightarrow \mathbb{F}_p
$$
The S-box is $\sigma(x) = x^5$, which is the optimal power map for BN254 since $\gcd(5, p-1) = 1$. [poseidon-hash](https://www.poseidon-hash.info)

Parameters are fixed to the **Circom-compatible BN254 configuration** to ensure the Rust R1CS prover and the Solidity `HeartbeatVerifier.sol` compute identical outputs:

| Parameter | Width-3 ($\mathcal{P}_2$) | Width-7 ($\mathcal{P}_6$) |
|-----------|---------------------------|---------------------------|
| $t$ | 3 | 7 |
| $R_F$ (full rounds) | 8 | 8 |
| $R_P$ (partial rounds) | 57 | 57 |
| $\alpha$ (S-box) | 5 | 5 |
| Rate | 2 | 6 |
| Capacity | 1 | 1 |

The round constants (ARK) and MDS matrix are generated by the Grain LFSR procedure specified in the Poseidon paper §A.2 and committed to the repository as immutable JSON artifacts (`poseidon_bn254_t3.json`, `poseidon_bn254_t7.json`). [docs.pantherprotocol](https://docs.pantherprotocol.io/docs/learn/cryptographic-primitives/poseidon)

**Invariant.** These constants MUST NOT be regenerated after the trusted setup ceremony. Any modification changes the circuit constraint count and invalidates all existing proving keys.

### 5.3 Circuit Instance & Witness

Let the **public instance** $\mathbf{x}$ (known to the EVM verifier) be:


$$
\mathbf{x} = (t_{chain},\ H_{accused},\ H_{att_1},\ H_{att_2},\ H_{att_3}) \in \mathbb{F}_p^5
$$
Let the **private witness** $\mathbf{w}$ (known only to the reporting node) be:


$$
\mathbf{w} = (IP_{rep},\ N_{high},\ N_{low},\ nonce_1, nonce_2, nonce_3,\ AS_1, AS_2, AS_3,\ t_1, t_2, t_3)
$$
where $IP_{rep} \in [0, 2^{128})$, $N_{high}, N_{low} \in [0, 2^{128})$, each $nonce_i \in [0, 2^{64})$, each $AS_i \in [0, 2^{32})$, and each $t_i \in [0, 2^{64})$. All map safely into $\mathbb{F}_p$.

### 5.4 R1CS Constraint System

The circuit $\mathcal{C}$ enforces the following R1CS constraints. An R1CS constraint is a triple $(\mathbf{a}, \mathbf{b}, \mathbf{c})$ over the witness vector $\mathbf{z}$ satisfying $\langle \mathbf{a}, \mathbf{z} \rangle \cdot \langle \mathbf{b}, \mathbf{z} \rangle = \langle \mathbf{c}, \mathbf{z} \rangle$. [docs.zkproof](https://docs.zkproof.org/pages/standards/accepted-workshop4/proposal-aggregation.pdf)

**Constraint 1 — Target Identity Binding:**


$$
H_{accused} = \mathcal{P}_2(N_{high},\ N_{low})
$$
The Poseidon sponge absorbs $(N_{high}, N_{low})$ and squeezes one field element. The output is constrained equal to the public input $H_{accused}$. If this constraint is omitted, a prover could claim any node is the accused party.

**Constraint 2 — BGP AS Path Diversity (Sybil Resistance):**

For all pairs $(i, j) \in \{(1,2), (1,3), (2,3)\}$:


$$
\exists\, inv_{ij} \in \mathbb{F}_p : (AS_i - AS_j) \cdot inv_{ij} = 1
$$
This is the standard **non-equality gadget** over $\mathbb{F}_p$. It enforces $AS_i \neq AS_j$ because $x \cdot x^{-1} = 1$ holds if and only if $x \neq 0$. The three constraints prevent a single-ISP Sybil from generating three nominally independent failure reports.

**Security note.** The constraint correctly handles the zero edge case: if $AS_i = AS_j$, then $AS_i - AS_j = 0$, which has no multiplicative inverse in $\mathbb{F}_p$. The witness assignment for $inv_{ij}$ becomes `SynthesisError::AssignmentMissing`, causing proof generation to fail before reaching the verifier.

**Constraint 3 — Temporal Window Bound:**

Failed connection attempts must lie within a 900-second window preceding the on-chain submission timestamp:


$$
t_{chain} - 900 \leq t_1 \leq t_2 \leq t_3 \leq t_{chain}
$$
This is decomposed into 5 comparison constraints using the `$enforce_cmp$` gadget (binary decomposition of the difference). The ordering sub-constraint $t_1 \leq t_2 \leq t_3$ additionally prevents out-of-order replay of a single genuine failure event.

**Constraint 4 — Attempt Commitment Binding:**

For each $i \in \{1, 2, 3\}$:


$$
H_{att_i} = \mathcal{P}_6(IP_{rep},\ nonce_i,\ AS_i,\ N_{high},\ N_{low},\ t_i)
$$
The Poseidon sponge absorbs all six inputs and squeezes one field element. The output is constrained equal to the public input $H_{att_i}$. The $nonce_i$ fields are independent random 64-bit values per attempt, providing anti-replay protection: the same (IP, node, time) triple with a different nonce produces a distinct commitment, preventing commitment recycling across multiple slash reports.

**Estimated constraint count:**


$$
|\mathcal{C}| \approx 47{,}000\text{–}55{,}000 \text{ R1CS constraints}
$$
Exact count is locked at first successful `cargo test --release -- --nocapture 2>&1 | grep $num_constraints$` execution and must be recorded before the Powers of Tau ceremony (§7). A constraint count $|\mathcal{C}| \leq 2^{16} = 65{,}536$ permits use of the `$pot16_final$.ptau` Phase 1 transcript; a count $\leq 2^{17}$ requires `$pot17_final$.ptau`.

### 5.5 Groth16 Verification Equation

The on-chain `HeartbeatVerifier.sol` checks the following pairing equation: [blog.lambdaclass](https://blog.lambdaclass.com/groth16/)


$$
e(A, B) = e(\alpha, \beta) \cdot e\!\left(\sum_{i=0}^{\ell} a_i \cdot \frac{\gamma^i}{\gamma},\ \gamma\right) \cdot e(C, \delta)
$$
where $(A, B, C) = \boldsymbol{\pi}$ is the proof, $(\alpha, \beta, \gamma, \delta)$ are the verifying key elements, $\{a_i\}_{i=0}^{\ell}$ are the public inputs $\mathbf{x}$, and $\ell = 5$ is the number of public signals. All pairing operations are computed over BN254 using the precompiles at EVM addresses `0x06` (bn256Add), `0x07` (bn256ScalarMul), and `0x08` (bn256Pairing).

***

## 6. Cryptoeconomics & Account Abstraction

### 6.1 The Token System

The Abzu network defines a native ERC-20 token $\mathtt{ABZU}$ with fixed supply $\Omega$ minted at genesis. The governance-locked derivative $\mathtt{veABZU}$ represents time-weighted voting power computed as:


$$
v_i(t) = a_i \cdot \frac{T_{lock,i} - t}{T_{max}} \quad \forall t \leq T_{lock,i}
$$
where $a_i$ is the locked amount, $T_{lock,i}$ is the unlock timestamp, and $T_{max} = 4 \text{ years}$. This is the standard **vote-escrow** construction preventing flash-loan governance attacks, since voting weight requires pre-committed lockup and is recorded in ERC20Votes checkpoints with a block-delay on weight changes.

### 6.2 EIP-4337 v0.7 PackedUserOperation

To eliminate the requirement for node operators to hold native ETH for gas, Abzu uses an **EIP-4337 v0.7 Paymaster** with the canonical EntryPoint at:


$$
\mathtt{EP}_{v0.7} = \texttt{0x0000000071727De22E5E9d8BAf0edAc6f37da032}
$$
The v0.7 `PackedUserOperation` struct packs gas limits into 256-bit words to reduce calldata cost:


$$
\mathtt{accountGasLimits} = \mathtt{verificationGasLimit} \parallel \mathtt{callGasLimit} \in \mathbb{B}^{32}
\mathtt{gasFees} = \mathtt{maxPriorityFeePerGas} \parallel \mathtt{maxFeePerGas} \in \mathbb{B}^{32}
\mathtt{paymasterAndData} = \mathtt{paymaster}_{20} \parallel \mathtt{pmVerifyGas}_{16} \parallel \mathtt{pmPostOpGas}_{16} \in \mathbb{B}^{52}
$$
The UserOperation hash is computed per EIP-712:


$$
\mathtt{UOhash} = \mathcal{H}_{\mathtt{K}}\!\left(\texttt{0x1901} \parallel \mathcal{H}_{\mathtt{K}}(\mathtt{domainSep}) \parallel \mathcal{H}_{\mathtt{K}}(\mathtt{structHash})\right)
$$
The gas conversion rate from ABZU to ETH uses a Chainlink TWAP oracle:


$$
Rate_{gas} = \frac{Price_{ETH/USD}}{Price_{ABZU/USD}}
$$
### 6.3 CID-Bound Escrow

The `AbzuStorageEscrow` contract binds payment strictly to a specific Root CID, rendering mempool front-running mathematically inert. The escrow ID is derived:


$$
ID_{escrow} = \mathcal{H}_{\mathtt{K}}(Payer \parallel R_{cid} \parallel b_{current})
$$
Relayers can only claim the escrow by providing $r_{min}$ independent node attestations that $R_{cid}$ is replicated. A valid attestation from node $\mathcal{N}_j$ is:


$$
\sigma_j = \text{Sign}_{K_{priv}^{(j)}}\!\left(\mathcal{H}_{\mathtt{K}}(ID_{escrow} \parallel R_{cid})\right)
$$
The contract verifies $\sigma_j$ via `ecrecover` and checks the attesting address against `NodeRegistry` for minimum staked collateral $\geq 500 \cdot 10^{18}$ base units.

**Re-entrancy protection.** All state-changing functions implement the **Checks-Effects-Interactions** (CEI) pattern with `nonReentrant` modifier from OpenZeppelin `ReentrancyGuard`. The escrow `amount` field is zeroed before any external token transfer:


$$
\text{State}_{escrow} \leftarrow \mathtt{Released},\ \text{amount} \leftarrow 0 \quad \text{before} \quad \mathtt{ABZU.safeTransfer}(\cdot)
$$
### 6.4 The Spawner Treasury

When an `AbsoluteDead` event is confirmed on-chain by a valid Groth16 slash proof, the `AbzuSpawnerTreasury` contract:

1. Slashes the dead node's staked collateral from `NodeRegistry`
2. Allocates a **spawn bounty** $B_{spawn}$ drawn from the treasury reserve
3. Emits a `SpawnRequest` event with target region and replication parameters
4. Transfers $B_{spawn}$ to the relayer that submitted the valid slash report

The spawn bounty is calibrated to cover 90 days of minimum-viable node operating costs in the target region, computed from an on-chain cloud pricing oracle.

***

## 7. Trusted Setup Ceremony

### 7.1 Powers of Tau

The Groth16 setup requires a **Structured Reference String** (SRS) generated in a two-phase ceremony. Phase 1 (Powers of Tau) generates the SRS independently of the circuit. Abzu uses the **Hermez Network** Phase 1 transcript (`$pot20_final$.ptau`) with 100+ independent contributors, providing 128-bit toxic waste security under the assumption that at least one contributor honestly discarded their randomness. [docs.zkproof](https://docs.zkproof.org/pages/standards/accepted-workshop4/proposal-aggregation.pdf)

### 7.2 Phase 2 Circuit-Specific Setup

Phase 2 is specific to the `HeartbeatFailureCircuit` constraint system. The Phase 2 proving key $\mathtt{zkey}_{final}$ is produced by:

1. Initializing from `$pot20_final$.ptau` and the compiled R1CS
2. Collecting contributions from $\geq 50$ independent actors, each using hardware entropy sources (HSM, air-gapped machine, physical dice)
3. Applying a **randomness beacon** from a future Ethereum block hash: $\mathtt{beacon} = \mathcal{H}_{\mathtt{K}}(\text{blockHash}(b_{future}))$

The resulting `zkey` is **verifiably non-forgeable** under the assumption that the product of all $N_{contrib}$ toxic waste scalars is non-zero, i.e., that not all contributors colluded:


$$
\tau = \prod_{i=1}^{N_{contrib}} \tau_i \neq 0 \pmod{p}
$$
### 7.3 Ceremony Transcript Publication

All Phase 2 contributions are published as a publicly verifiable transcript at `ceremony.abzu.network`. The final `$verification_key$.json` is committed to the repository and cross-verified against the on-chain `HeartbeatVerifier.sol` before mainnet activation.

***

## 8. WASM Edge Architecture

### 8.1 Linear Memory Constraint

WebAssembly enforces a 32-bit linear memory address space with a hard ceiling of:


$$
M_{max} = 2^{32}\ \text{bytes} = 4\ \text{GiB}
$$
Naively buffering a payload of size $|D| \gg M_{max}$ into WASM heap causes an **Out-Of-Memory panic** in the browser engine, which is non-recoverable and non-catchable at the JavaScript boundary.

### 8.2 Zero-Heap FSA Streaming

Abzu bypasses linear memory exhaustion via the **File System Access (FSA) API**, streaming chunk bytes directly to the host OS filesystem through a `FileSystemWritableFileStream` handle. The memory invariant for each chunk $C_i$ is:


$$
\forall C_i \in D :\ \mathcal{H}_{state} \leftarrow \mathtt{absorb}(C_i) \implies \mathtt{flush}(C_i,\ \text{Disk}) \implies \mathtt{GC}(C_i)
$$
The only persistent WASM heap allocation is the **BLAKE3 hasher state** (8 bytes of chaining value plus a 64-byte block buffer), yielding:


$$
M_{WASM}(|D|) = O(1) \quad \forall |D| \leq 2^{64}
$$
This is contrasted with the naive approach, which yields $M_{WASM}(|D|) = O(|D|)$ and OOM-panics for any $|D| > M_{max}$.

**Partial write safety.** If the FSA write fails mid-stream (e.g., user revokes disk permission), the writable stream's `abort()` method is called before the function returns an error, leaving no partial file on disk.

**Root CID post-verification.** After all $k$ chunks have been flushed to disk, the accumulated BLAKE3 state is finalized and compared against the expected $R_{cid}$:


$$
R_{computed} = \mathcal{H}_{\mathtt{B3}}\!\left(\bigoplus_{i=0}^{k-1} \mathtt{absorb}(C_i)\right)
R_{computed} \stackrel{?}{=} R_{cid}
$$
If the comparison fails, the writable stream is aborted via `abort()`, the partial file is deleted from the host filesystem, and a `RootCidMismatch` error is returned to the JavaScript caller. Under no circumstance is a file with an unverified root CID surfaced to the application layer.

### 8.3 WebRTC Transport Layer

Browser nodes communicate via **WebRTC data channels** over the `libp2p-webrtc` transport. The connection lifecycle is:


$$
\mathtt{ICE\ Gathering} \rightarrow \mathtt{DTLS\ Handshake} \rightarrow \mathtt{SCTP\ Association} \rightarrow \mathtt{DataChannel\ Open}
$$
**Critical regression — libp2p WebRTC Issue \#5986.** A bug in `rust-libp2p` versions prior to the vetted commit causes silent `DataChannel` closure without emitting a close event, isolating browser peers without triggering reconnection logic. The `Cargo.toml` MUST pin to a vetted commit hash rather than a semver range:

```toml
libp2p-webrtc = { git = "https://github.com/libp2p/rust-libp2p",
                  rev = "<vetted-sha>",
                  features = ["tokio"] }
```

Pinning to a semver version (e.g., `"0.7"`) permits `cargo update` to silently pull regressions in patch releases, violating the CI/CD immutability invariant.

### 8.4 P2P Client State Machine

The WASM P2P client implements the following connection state machine:


$$
\mathtt{Disconnected} \xrightarrow{\mathtt{connect()}} \mathtt{Connecting} \xrightarrow{\mathtt{ICE\_complete}} \mathtt{Connected} \xrightarrow{\mathtt{channel\_open}} \mathtt{Streaming}
\mathtt{Streaming} \xrightarrow{\mathtt{channel\_close}} \mathtt{Reconnecting} \xrightarrow{\mathtt{backoff}} \mathtt{Connecting}
\mathtt{Reconnecting} \xrightarrow{\mathtt{max\_retries}} \mathtt{Disconnected}
$$
Transitions to `Reconnecting` are triggered by any of: explicit channel close event, DTLS timeout, ICE failure, or absence of data for $\tau_{idle} = 30$ seconds. The reconnection backoff follows a **truncated exponential** schedule:


$$
\Delta t_n = \min\!\left(2^n \cdot \Delta t_0,\ \Delta t_{max}\right), \quad \Delta t_0 = 1\text{s},\ \Delta t_{max} = 60\text{s}
$$
***

## 9. RPC Resilience & Off-Chain Liveness

### 9.1 Rotating Provider Model

The `ContractBridge` maintains a priority-ordered list of RPC endpoints $\{E_1, E_2, \ldots, E_n\}$ with associated health state. The active provider index $\iota$ is maintained as a lock-free atomic:


$$
\iota \leftarrow \underset{i \in [n]}{\arg\min}\, \text{priority}(E_i)\ \text{s.t.}\ \text{healthy}(E_i)
$$
Provider health is assessed via a liveness probe — `$eth_blockNumber$` — before each transaction submission. On probe failure, $\iota$ rotates to the next healthy endpoint. A **background reconnection task** runs on a 60-second interval, re-establishing connections to previously failed endpoints:


$$
\forall i :\ \text{healthy}(E_i) = \bot \implies \text{attempt\_reconnect}(E_i) \text{ after } 60\text{s}
$$
This ensures the provider pool degrades gracefully under partial RPC outages without requiring node restart.

### 9.2 EIP-4337 Bundler Failover

UserOperation submission targets a primary bundler endpoint. On HTTP failure or bundler rejection (error code `AA` class), the bridge retries against the secondary bundler pool:


$$
\mathtt{bundlers} = \langle \mathtt{Stackup},\ \mathtt{Alchemy},\ \mathtt{Pimlico},\ \mathtt{self\_hosted} \rangle
$$
Bundler selection follows the same rotating priority model as the RPC provider, with the additional constraint that the same `UserOperation` nonce cannot be submitted to two bundlers simultaneously (double-submit attack prevention).

***

## 10. Security Analysis

### 10.1 Threat Model

The Abzu security model operates under the following assumptions:

**Honest majority.** At any point in time, less than one-third of staked collateral is controlled by Byzantine actors: $stake_{byzantine} < \frac{1}{3} \cdot stake_{total}$.

**Cryptographic hardness.** The **discrete logarithm** problem on BN254 is computationally infeasible. The **Knowledge of Exponent** assumption holds. The **Poseidon** hash function is collision resistant over $\mathbb{F}_p$.

**Ceremony integrity.** At least one Phase 2 contributor honestly discarded their toxic waste scalar $\tau_i$.

**Network model.** The adversary controls a subset of network links and nodes, but cannot indefinitely delay honest messages (partially synchronous network model).

### 10.2 Attack Surface Analysis

**Under-constrained ZK circuit.** If any private witness signal participates in computation but is not constrained to influence a public output, the prover can assign it to an arbitrary value and generate a valid proof for a false statement. This is the most dangerous class of ZK vulnerability because the on-chain verifier accepts the proof with no revert. Mitigation: Veridise automated constraint extraction and Trail of Bits manual circuit review (Gate 1 of the audit RFP).

**Smart contract re-entrancy.** The `release()` loop in `AbzuStorageEscrow` processes external node signatures. Without `nonReentrant` and strict CEI ordering, an attacker who controls a malicious ERC-20 token could force re-entry before state transitions to `Released`, draining the escrow. Mitigation: `ReentrancyGuard`, `SafeERC20`, and CEI enforced at all three state-changing entry points.

**Sybil attack on BGP diversity constraint.** An attacker controlling ASNs at multiple BGP prefixes could construct three nominally independent paths that are actually colocated. Mitigation: The ZK circuit enforces $AS_i \neq AS_j$ at the field level, but cannot verify that the AS numbers correspond to genuinely independent physical paths. On-chain reputation weighting of attesting nodes by their geographic diversity bonus $x_{geo}$ provides economic disincentive. Full mitigation requires integration of a decentralized BGP attestation oracle (future work).

**Slash oracle replay.** A valid slash report $(\boldsymbol{\pi}, \mathbf{x})$ could be re-submitted to slash the same node twice. Mitigation: The `AbzuSlashOracle` maintains a nullifier mapping:


$$
\mathtt{usedProofs} : \mathbb{B}^{32} \rightarrow \mathbb{B}^1
$$
where the nullifier key is $\mathcal{H}_{\mathtt{K}}(ID_{escrow} \parallel \boldsymbol{\pi})$. Any re-submission reverts.

**Eclipse attack.** An adversary fills a target node's Kademlia routing table with adversary-controlled peers, isolating it from the honest network. Mitigation: Libp2p's `Identify` protocol refreshes routing table entries from multiple independent sources. The 300-second Kademlia refresh interval (§1.4) limits the duration of a successful eclipse to the refresh window.

**Favor score manipulation.** A rational attacker could selectively respond to Favor-probing peers while failing to serve data to regular clients. Mitigation: Favor samples are drawn from actual shard retrieval events, not from synthetic probes, making selective performance computationally expensive to sustain.

**Paymaster drain via UserOp replay.** An attacker could replay a signed `PackedUserOperation` to drain the Paymaster's EntryPoint deposit. Mitigation: The EntryPoint v0.7 enforces per-sender nonce monotonicity. The Paymaster's `validatePaymasterUserOp` additionally verifies the `paymasterAndData` signature is fresh and not previously used.

### 10.3 Forward Secrecy

Node identity keys are Ed25519 keypairs stored on disk at `config/identity.key`. Compromise of a node's identity key allows an attacker to impersonate that node in the DHT and GossipSub mesh, but does not expose historical session data (WebRTC DTLS sessions use ephemeral key exchange providing forward secrecy). Compromise does not allow the attacker to forge ZK proofs, as proofs require knowledge of private witnesses, not identity keys.

***

## 11. Network Parameters

The following parameters are fixed at the genesis block and may only be modified through on-chain governance ($\mathtt{veABZU}$ supermajority vote with $\geq 14$-day timelock):

| Parameter | Symbol | Value | Rationale |
|-----------|--------|-------|-----------|
| Heartbeat interval | $\Delta_{hb}$ | 15 s | Responsive to failures within 2 min |
| Absolute death threshold | $m_{dead}$ | 8 missed beats | 120 s minimum grace period |
| EWMA span | $N$ | 120 samples | 30-min rolling window |
| Minimum Favor score | $\mathcal{F}_{min}$ | 2500 | 25th percentile threshold |
| Genesis exemption | $E$ | 80,640 blocks | ≈ 14 days on Arbitrum |
| Default chunk size | $\Delta_c$ | $2^{22}$ bytes | 4 MiB — balanced for P2P MTU |
| Maximum replicas | $r_{max}$ | 20 | Escrow contract bound |
| Escrow timeout | $\tau_{escrow}$ | 86,400 s | 24-hour window |
| ZK proof size | $|\boldsymbol{\pi}|$ | 256 bytes | BN254 Groth16 fixed size |
| On-chain verify cost | $G_{\pi}$ | ~250,000 gas | Arbitrum One estimate |
| Min node collateral | $C_{min}$ | $500 \cdot 10^{18}$ base units | 500 ABZU |
| GossipSub mesh degree | $D$ | 6 | 4-regular minimum fault tolerance |
| Kademlia $k$-bucket size | $k$ | 20 | Standard Kademlia |
| DHT refresh interval | $\tau_{kad}$ | 300 s | Routing table rot prevention |
| WASM idle timeout | $\tau_{idle}$ | 30 s | DataChannel liveness probe |
| Reconnection base delay | $\Delta t_0$ | 1 s | Exponential backoff floor |
| Reconnection max delay | $\Delta t_{max}$ | 60 s | Backoff ceiling |

***

## 12. Deployment Sequence & Gate Conditions

Abzu's path to mainnet is gated by five strictly ordered conditions. Mainnet activation is blocked until all five gates are cleared:

**Gate 1 — ZK Circuit Audit.** Independent cryptographic review of the `HeartbeatFailureCircuit` R1CS by a firm specializing in ZK constraint systems (Veridise or Nethermind Security). Deliverable: formal under-constraint proof attempts, witness malleability report, and Poseidon parameter cross-stack verification.

**Gate 2 — Smart Contract Audit.** Independent Solidity review of all six contracts plus `HeartbeatVerifier.sol` by a separate firm (Trail of Bits or Nethermind Security). Deliverable: severity-classified finding report with PoC exploits for all Critical/High findings, one remediation round included.

**Gate 3 — Trusted Setup Ceremony.** Phase 2 contributions from $\geq 50$ independent actors using hardware entropy. Public transcript published and verifiable. Ceremony integrity confirmed by at least two independent verifiers.

**Gate 4 — Testnet Operation.** 90-day continuous testnet on Arbitrum Sepolia with $\geq 10$ geographically distributed nodes. Required observations: Kademlia routing table recovery after 40% node churn, successful ZK slash proof generation and on-chain verification, escrow deposit/release cycle under adversarial conditions, WASM client upload/download of $\geq 1$ GiB payload.

**Gate 5 — Economic Parameter Review.** Independent cryptoeconomics review of spawn bounty calibration, ABZU token supply, collateral slash parameters, and Paymaster TWAP oracle manipulation surface.

***

## Appendix A: Groth16 Proof System Formal Definition

A Groth16 proof system $\Pi = (\mathtt{Setup}, \mathtt{Prove}, \mathtt{Verify})$ for an R1CS relation $\mathcal{R}$ over $\mathbb{F}_p$ is defined: [blog.lambdaclass](https://blog.lambdaclass.com/groth16/)


$$
\mathtt{Setup}(1^\lambda, \mathcal{R}) \rightarrow (\mathtt{pk}, \mathtt{vk})
\mathtt{Prove}(\mathtt{pk}, \mathbf{x}, \mathbf{w}) \rightarrow \boldsymbol{\pi} = (A \in \mathbb{G}_1,\ B \in \mathbb{G}_2,\ C \in \mathbb{G}_1)
\mathtt{Verify}(\mathtt{vk}, \mathbf{x}, \boldsymbol{\pi}) \rightarrow \{\top, \bot\}
$$
The verification equation checks:


$$
e(A, B) \stackrel{?}{=} e(\alpha \cdot G_1,\ \beta \cdot G_2) \cdot e\!\left(\sum_{i=0}^{\ell} a_i \cdot \frac{\gamma_i}{\gamma},\ \gamma \cdot G_2\right) \cdot e(C,\ \delta \cdot G_2)
$$
where $e : \mathbb{G}_1 \times \mathbb{G}_2 \rightarrow \mathbb{G}_T$ is the optimal Ate pairing over BN254, and $(\alpha, \beta, \gamma, \delta, \tau)$ are the toxic waste scalars from the trusted setup.

**Completeness.** For all $(\mathbf{x}, \mathbf{w}) \in \mathcal{R}$:


$$
\Pr[\mathtt{Verify}(\mathtt{vk}, \mathbf{x}, \mathtt{Prove}(\mathtt{pk}, \mathbf{x}, \mathbf{w})) = \top] = 1
$$
**Soundness.** For all polynomial-time adversaries $\mathcal{A}$:


$$
\Pr[\mathtt{Verify}(\mathtt{vk}, \mathbf{x}, \boldsymbol{\pi}) = \top \wedge (\mathbf{x}, \cdot) \notin \mathcal{R}] \leq \mathtt{negl}(\lambda)
$$
**Zero-knowledge.** The proof $\boldsymbol{\pi}$ reveals no information about $\mathbf{w}$ beyond what is implied by $\mathbf{x}$.

***

## Appendix B: BLAKE3 Hash Function Properties

BLAKE3 is a cryptographic hash function based on the **Bao tree** construction, providing: [github](https://github.com/ethereum/yellowpaper)

- **Security level:** 128 bits (collision resistance), 256 bits (preimage resistance)
- **Parallelism:** $O(\log k)$ depth for $k$ chunks via internal Merkle tree
- **Streaming:** Subtree hashes are independently computable — enables Bao-encoded stream verification
- **Speed:** $\sim$6.7 GB/s single-core on x86-64 (AVX-512), $\sim$0.8 GB/s on WASM (no SIMD)

The BLAKE3 compression function is not algebraically structured over $\mathbb{F}_p$ and is therefore **not suitable for ZK circuit use**. This is the precise reason Poseidon is used for ZK commitments while BLAKE3 is used for all off-circuit content addressing. [research.aragon](https://research.aragon.org/poseidon-noir.html)

***

## Appendix C: BN254 Curve Parameters

| Parameter | Value |
|-----------|-------|
| Curve equation | $y^2 = x^3 + 3$ over $\mathbb{F}_q$ |
| Field modulus $q$ | $21888242871839275222246405745257275088696311157297823662689037894645226208583$ |
| Scalar modulus $p$ | $21888242871839275222246405745257275088548364400416034343698204186575808495617$ |
| $\mathbb{G}_1$ generator $G_1$ | $(1, 2)$ |
| $\mathbb{G}_2$ generator $G_2$ | (standard BN254 $\mathbb{G}_2$ generator point) |
| Embedding degree | 12 |
| Security level | 128 bits |
| Pairing type | Optimal Ate |
| EVM precompiles | `0x06` (add), `0x07` (mul), `0x08` (pairing) |

***

## Appendix D: Poseidon Permutation Construction

The Poseidon permutation $\mathtt{PERM}_t$ over $\mathbb{F}_p^t$ applies $R_F + R_P$ rounds in sequence: [docs.pantherprotocol](https://docs.pantherprotocol.io/docs/learn/cryptographic-primitives/poseidon)

**Full rounds** ($R_F / 2$ before and after partial rounds): Apply the S-box $\sigma(x) = x^5$ to **all** $t$ state elements, add round constants $\mathbf{c}_r$, apply MDS matrix $M$:


$$
\text{state} \leftarrow M \cdot (\sigma(\text{state} + \mathbf{c}_r))
$$
**Partial rounds** ($R_P$ middle rounds): Apply the S-box $\sigma$ to only the **first** state element, add round constants, apply MDS:


$$
\text{state} \leftarrow M \cdot (\sigma(\text{state}_0 + c_{r,0}),\ \text{state}_1 + c_{r,1},\ \ldots)
$$
The sponge construction absorbs $\mathtt{rate} = t - 1$ field elements per permutation call:


$$
\mathtt{absorb}(\{x_1, \ldots, x_{rate}\}) :\ \text{state}\llbracket 0 \ldots rate-1 \rrbracket \mathrel{+}= \{x_1, \ldots, x_{rate}\},\quad \text{state} \leftarrow \mathtt{PERM}_t(\text{state})
\mathtt{squeeze}(1) :\ \text{return state}\llbracket 0 \rrbracket
$$
The MDS matrix $M$ is a $t \times t$ Cauchy matrix over $\mathbb{F}_p$, constructed as:


$$
M_{ij} = (x_i + y_j)^{-1} \pmod{p}, \quad x_i = i,\ y_j = t + j
$$
with $\{x_i\} \cup \{y_j\}$ pairwise distinct, guaranteeing the MDS property (every square submatrix is invertible).

***

## Appendix E: Glossary

| Term | Definition |
|------|-----------|
| $\mathbb{F}_p$ | BN254 scalar field with prime modulus $p$ |
| $\mathbb{G}_1, \mathbb{G}_2$ | BN254 elliptic curve groups |
| $\mathbb{G}_T$ | BN254 target group of the pairing |
| R1CS | Rank-1 Constraint System — the algebraic representation of a ZK circuit |
| SRS | Structured Reference String — the output of the trusted setup ceremony |
| EWMA | Exponentially Weighted Moving Average — the peer scoring smoothing function |
| DHT | Distributed Hash Table — Kademlia-based peer routing infrastructure |
| CAS | Content-Addressable Storage — storage indexed by BLAKE3 hash of content |
| CID | Content Identifier — the BLAKE3 root hash of a file or chunk |
| FSA | File System Access API — browser-native API for direct OS disk I/O |
| CEI | Checks-Effects-Interactions — Solidity re-entrancy prevention pattern |
| TWAP | Time-Weighted Average Price — manipulation-resistant on-chain oracle price |
| KoE | Knowledge of Exponent — cryptographic assumption underlying Groth16 soundness |
| BGP | Border Gateway Protocol — the inter-domain routing protocol defining ASNs |
| ASN | Autonomous System Number — a unique identifier for a BGP routing domain |
| Toxic waste | The trapdoor scalars $(\tau_i)$ from the trusted setup, which must be discarded |
| Nullifier | A one-time commitment preventing double-use of a ZK proof on-chain |
| veABZU | Vote-Escrowed ABZU — time-locked governance token with decaying weight |

***

## References

1. G. Wood. *Ethereum: A Secure Decentralised Generalised Transaction Ledger.* Yellow Paper, 2014. [ethereum.github](https://ethereum.github.io/yellowpaper/paper.pdf)
2. J. Groth. *On the Size of Pairing-Based Non-Interactive Arguments.* EUROCRYPT 2016. [docs.zkproof](https://docs.zkproof.org/pages/standards/accepted-workshop4/proposal-aggregation.pdf)
3. L. Grassi, D. Khovratovich, C. Rechberger, A. Roy, M. Schofnegger. *Poseidon: A New Hash Function for Zero-Knowledge Proof Systems.* USENIX Security 2021. [poseidon-hash](https://www.poseidon-hash.info)
4. P. Maymounkov, D. Mazières. *Kademlia: A Peer-to-peer Information System Based on the XOR Metric.* IPTPS 2002.
5. Y. Yohannes et al. *EIP-4337: Account Abstraction Using Alt Mempool v0.7.*
6. O. O'Connor, J. Austern, A. Beregszaszi, S. Wälde. *BLAKE3: one function, fast everywhere.* 2020. [github](https://github.com/ethereum/yellowpaper)
7. D. Bernstein. *Ed25519: High-speed high-security signatures.* 2011.
8. Abzu Network. *Genesis Orchestration Playbook.* `deploy/$genesis_orchestration$.sh`, 2026.
9. Abzu Network. *Poseidon BN254 Parameter Artifacts.* `abzu-node/params/`, 2026.
10. Abzu Network. *Security Audit Request for Proposal.* Internal document, 2026.

***

*Abzu Yellowpaper v1.0 — Specification subject to amendment pending audit findings. Mainnet parameters are not final until Gate 3 (Trusted Setup Ceremony) is complete and the constraint count is locked.*

*"The water beneath the world."*
