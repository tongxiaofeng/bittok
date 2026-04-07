# BitTok — System Design

## 1. Identity Model

### 1.1 Public Key as Identity

Each role's public key (BRC-100) IS the identity. No on-chain registration node required — the public key naturally appears in every action the role performs.

### 1.2 Identity in Context

| Context | How Identity Appears |
|---------|---------------------|
| Video Metanet metadata | `creatorPubKey` in TLV payload |
| HTLC script | `curatorPkh` (claim path), `buyerPkh` (refund path) |
| HTLC claim tx outputs | Creator address (from `P_creator`), Curator address (from `P_curator`) |
| MessageBox | Messages signed with sender's private key, verified by public key |
| Overlay queries | Search by `creatorPubKey` |
| CDN broadcasts | Availability signed with `P_cdn`, verifiable by any peer |

## 2. Content Model

### 2.1 Video Structure

- Duration: 15–60 seconds
- Chunking: 1 second per chunk
- Encryption: Each chunk independently encrypted with AES-256-GCM
- Key derivation: Method 42 ECDH
  - `aesKey_i = HKDF(ECDH(D_creator, P_creator), keyHash_i, "bittok-chunk-encryption")`
  - `keyHash_i = SHA256(SHA256(plaintext_chunk_i))`

### 2.2 On-Chain Content Storage

Encrypted chunks are stored on-chain by Creator using "An Immutable File and Data Store":

- **B:// protocol** (chunks ≤ 100KB): Single transaction per chunk
  ```
  OP_RETURN
    19HxigV4QyBv3tHpQVcUEQyq1pzZVdoAut   # B:// prefix
    <encrypted_chunk_data>
    application/octet-stream
    binary
  ```
- **BCAT protocol** (chunks > 100KB): Split across multiple transactions with anchor tx

### 2.3 Video Metadata (Metanet Node)

```
Metanet Node Structure:
  OP_RETURN
    <metanet_flag>
    <P_video>                  # Video node's public key
    <TxID_parent>              # Creator's identity node TxID

  TLV Payload:
    videoId                    # SHA256(SHA256(original video file))
    creatorPubKey              # Creator's BRC-100 public key
    title
    description
    tags                       # JSON array
    chunkCount                 # e.g., 30 for a 30s video
    chunkHashes[]              # SHA256(SHA256(encrypted_chunk_i))
    chunkTxIds[]               # On-chain storage txid per chunk (B:// or BCAT)
    keyHashes[]                # keyHash per chunk (for capsule derivation)
    satoshisPerKilobyte        # Authorization fee (satoshis/KB)
    creatorSharePercent        # Creator's share of authorization fee (e.g., 70)
    timestamp
```

### 2.4 Content Publishing Flow

1. Creator encrypts video chunks locally (key derived from per-video derived key, not master key)
2. Creator stores encrypted chunks on-chain (B:// or BCAT)
3. Creator publishes Metanet metadata on-chain (includes chunk storage txids, split ratio)
4. Creator transfers per-video derived key to Curator (via secure channel). Derived as `D_video = HKDF(D_creator, videoId)`. Master key `D_creator` never leaves Creator.
5. Creator can go offline

**Future improvement**: Replace per-video key delegation with Proxy Re-Encryption (PRE) so Curator can facilitate key exchange without ever knowing decryption keys.

## 3. Discovery Architecture

### 3.1 Content Discovery — Overlay Service (SHIP/SLAP)

Curator runs a topic-specific Overlay Service node that indexes transactions matching the `bittok` protocol prefix.

- **SHIP (Simple Hosting and Indexing Protocol)**: Indexes BitTok Metanet nodes (video metadata)
- **SLAP (Service Lookup Availability Protocol)**: Allows agents to discover available Curator overlay nodes

**Flow**:
1. Curator publishes video metadata → Metanet node tx broadcast to network
2. Curator's overlay node indexes the tx (protocol-prefix matching)
3. Viewer/CDN agents query Curator's overlay to discover available videos
4. Overlay returns video metadata (title, tags, pricing, chunk txids, etc.)
5. Agents verify data against on-chain Metanet node if needed (by txid)

### 3.2 Node Discovery — MessageBox

Dynamic, real-time information flows through MessageBox P2P:

- **CDN availability**: "I have video X chunks 1-30 available, download price: 1 sats/KB"
- **Pricing updates**: CDN agents broadcast updated pricing
- **Purchase requests**: Viewer → Curator for HTLC key exchange

### 3.3 Discovery Summary

| What | Where | Purpose |
|------|-------|---------|
| Video catalog | Curator Overlay (SHIP) | Content discovery, search, browse |
| Node availability | MessageBox | Real-time pricing, seller selection |
| Payment + key exchange | MessageBox + on-chain HTLC | Authorization |
| Permanent record | On-chain Metanet nodes | Verification, ownership proof |

## 4. Payment Architecture

### 4.1 Two-Layer Model

```
┌──────────────────────────────────────────────────┐
│  Layer 1: Authorization Fee (On-chain)           │
│  Viewer ──→ Curator (handles HTLC)               │
│  HTLC + Capsule → buyer-specific decryption key  │
│  Claim tx atomic split: Creator % + Curator %    │
│  High on-chain transaction throughput             │
├──────────────────────────────────────────────────┤
│  Layer 2: Download Fee (Off-chain)               │
│  Viewer ──→ CDN                                  │
│  x402 + Payment Channel → encrypted chunk data   │
│  100% to data provider                           │
└──────────────────────────────────────────────────┘
```

### 4.2 Authorization (Layer 1) — HTLC + Capsule

Curator handles HTLC key exchange on behalf of Creator, using Creator's delegated keys. Capsule mechanism ensures buyer-specific key delivery.

**Claim transaction atomic split**:
```
Claim Transaction:
  Input:   HTLC UTXO (Viewer's funding)
  Output 0: creator_share → Creator address   (creatorSharePercent %)
  Output 1: curator_share → Curator address   (100 - creatorSharePercent %)
```

Split ratio declared in on-chain Metanet metadata, verifiable by anyone.

### 4.3 Download (Layer 2) — x402 + Payment Channel

CDN serves encrypted chunk data via HTTP 402 protocol:
```
Viewer → CDN:  GET /chunk/{videoId}/{chunkIndex}
CDN → Viewer:  402 Payment Required
               X-Price-Per-KB: <sats>
               X-File-Size: <bytes>
               X-Invoice-Id: <hex>
Viewer → CDN:  Payment via Payment Channel
CDN → Viewer:  200 OK + encrypted chunk data
```

### 4.4 Pricing

- Authorization fee: `satoshisPerKilobyte` set by Creator, split with Curator on-chain
- Download fee: `satoshisPerKilobyte` set independently by each CDN (competitive market)
- Viewer agent optimizes for lowest total cost + best reliability

## 5. CDN Economics

- CDN downloads encrypted chunks from chain (by txid)
- CDN caches and serves encrypted chunks to Viewers at its own download price
- CDN never holds decryption keys — serves only encrypted data
- Profit = (download fees x viewers served) - storage/bandwidth costs

## 6. Curator Economics

- Curator receives `(100 - creatorSharePercent)%` of every authorization fee (atomic on-chain split)
- Curator runs Overlay Service infrastructure
- Curator does NOT store content or provide download service — Creator stores on-chain, CDN handles distribution

### Trust Model

- Creator delegates per-video derived keys to Curator, not master key (limited trust: compromise affects only that video)
- Revenue split is atomic on-chain (no trust required for payment)
- Creator can verify all transactions on-chain
- Creator can switch Curators — ownership proven by Metanet identity
- Multiple Curators compete for Creators

## 7. Streaming Pipeline

### 7.1 Parallel Pipeline

Authorization (Layer 1) and download (Layer 2) proceed in parallel:

```
Authorization (Layer 1)          Download (Layer 2)
HTLC chunk 0  ─────────┐        fetch chunk 0  ────────┐
HTLC chunk 1  ─────────┤        fetch chunk 1  ────────┤
HTLC chunk 2  ─────────┤        fetch chunk 2  ────────┤
                        ↓                                ↓
               Keys ready               Encrypted data ready
                        └───────── Decrypt + Play ───────┘
```

### 7.2 Buffering Strategy

```
t=0s  ──── Initial buffer (3-5 HTLC cycles in parallel) ────
      HTLC chunk 0  →  claim  →  decrypt  ┐
      HTLC chunk 1  →  claim  →  decrypt  ├─ buffer ready → start playback
      HTLC chunk 2  →  claim  →  decrypt  ┘

t=1s  Play chunk 0  │  HTLC chunk 3
t=2s  Play chunk 1  │  HTLC chunk 4
...
```

Initial latency: ~3-5 seconds. After buffer filled: smooth playback with rolling pre-purchase.

## 8. Transaction Volume

| Metric | Value |
|--------|-------|
| Chunk duration | 1 second |
| On-chain txs per chunk purchase | 2 (HTLC funding + claim) |
| 30s video full view | 60 on-chain txs |
| CDN data retrieval | From chain (by txid) or cached |

## 9. Technology Stack

- **Language**: TypeScript (full stack)
- **BSV Libraries**: `@bsv/sdk`, `@bsv/simple`, `@bsv/simple-mcp`
- **Smart Contracts**: Runar (TypeScript → Bitcoin Script compiler, runar.build)
- **Frontend**: React / Next.js
- **Agent Runtime**: Node.js
- **AI**: LLM API calls for agent decision-making
- **Video Processing**: ffmpeg for chunking
- **Test Content**: Remotion for AI video generation (external script)

## 10. System Boundary

| Component | In System (BitTok Core) | External |
|-----------|------------------------|----------|
| Creator publish API | Encrypt → transfer to Curator | — |
| AI video generation | — | Remotion + call publish API |
| Curator Service | Overlay, HTLC auth, on-chain storage | — |
| CDN Agent | Full implementation | — |
| Viewer Agent | Full implementation | — |
| Web UI | Real-time dashboard | — |
