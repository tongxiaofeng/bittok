# BitTok — Decentralized Short Video Platform

## Overview

BitTok is a decentralized short video platform where AI agents autonomously trade video content through BSV micropayments. Creators publish encrypted short videos, CDN agents cache and distribute content for profit, and Viewer agents learn user preferences to recommend and stream videos — paying per second of content consumed.

**Core Value Proposition**: Pay only for what you watch, at 1-second granularity. No subscriptions, no platform cut, no censorship.

## Agent Architecture

### Three Agent Types

| Agent | Goal | AI Behavior | Online Requirement |
|-------|------|-------------|--------------------|
| **Creator Agent (Daemon)** | Maximize content revenue | Dynamic pricing, analytics | Always online (automated daemon) |
| **CDN Agent** | Profit from content distribution | Predict popular content, competitive pricing | Always online |
| **Viewer Agent** | Maximize user satisfaction | Learn preferences, recommend videos, find best deals | Online when user is active |

### Agent Identity

Each agent has a BRC-100 wallet with independent BSV identity (public/private keypair). All inter-agent communication uses MessageBox P2P.

## Content Model

### Video Format

- **Duration**: 15–60 seconds (short video format, TikTok-style)
- **Chunking**: 1 second per chunk
- **Encryption**: Each chunk independently encrypted with AES-256-GCM
- **Key Derivation**: Method 42 ECDH (same as BitFS)
  - `aesKey_i = HKDF(ECDH(D_creator, P_creator), keyHash_i, "bittok-chunk-encryption")`
  - Each chunk has a unique `keyHash_i = SHA256(SHA256(plaintext_chunk_i))`

### On-Chain Video Metadata (OP_RETURN)

```
OP_RETURN
  <protocol_prefix>         # "bittok" protocol identifier
  <action: "publish">
  <videoId>                  # SHA256(original video file)
  <creatorPubKey>            # Creator's BRC-100 public key
  <title>
  <description>
  <tags>                     # JSON array
  <chunkCount>               # e.g., 30 for a 30s video
  <chunkHashes[]>            # SHA256 hash of each encrypted chunk
  <keyHashes[]>              # keyHash per chunk (for capsule derivation)
  <pricePerChunk>            # Creator's authorization fee (satoshis)
  <selfHostUrl>              # x402 endpoint (optional; omitted if on-chain)
  <timestamp>
```

### Content Storage Options

1. **Self-hosted** (default): Creator's daemon serves encrypted chunks via x402 HTTP endpoint
2. **On-chain** (most expensive): Encrypted chunks stored in OP_RETURN transactions

## Discovery Mechanism (Fully P2P)

### Content Discovery — On-Chain

Agents read on-chain video metadata transactions to discover available content. Video metadata is permanent, tamper-proof, and universally accessible.

**Note**: This is NOT chain scanning. Agents use overlay services / topic-based subscription to receive new video publication transactions matching the `bittok` protocol prefix.

### Node Discovery — MessageBox

Dynamic information flows through MessageBox P2P:

- **CDN availability broadcasts**: "I have video X chunks 1-30 available, download price: 5 sats/chunk"
- **Pricing updates**: CDN agents broadcast updated pricing
- **Purchase requests**: Viewer → Creator daemon for HTLC key exchange

### No Central Registry

All discovery is peer-to-peer. No registration authority or directory server.

## Payment Architecture

### Two-Layer Payment (No Revenue Sharing)

```
┌──────────────────────────────────────────────────┐
│  Layer 1: Authorization Fee (On-chain)           │
│  Viewer ──→ Creator                              │
│  HTLC + Capsule → buyer-specific decryption key  │
│  100% to Creator                                 │
│  = 1.5M+ on-chain transaction source             │
├──────────────────────────────────────────────────┤
│  Layer 2: Download Fee (Off-chain)               │
│  Viewer ──→ CDN (or Creator)                     │
│  x402 + Payment Channel → encrypted chunk data   │
│  100% to data provider                           │
└──────────────────────────────────────────────────┘
```

### Layer 1: HTLC + Capsule Authorization (BitFS Pattern)

Directly reuses BitFS's proven HTLC + Capsule mechanism, adapted for MessageBox communication (no mempool/chain scanning).

#### Capsule Computation

```
# Creator-side (per buyer, per chunk)
capsule_i = aesKey_i XOR buyerMask

where:
  aesKey_i  = HKDF(ECDH(D_creator, P_creator), keyHash_i, "bittok-chunk-encryption")
  buyerMask = HKDF(ECDH(D_creator, P_buyer), keyHash_i || nonce, "bittok-buyer-mask")
  nonce     = per-invoice random value (unlinkability)

capsuleHash_i = SHA256(videoId || chunkIndex || capsule_i)
```

#### HTLC Script (106 bytes, same as BitFS)

```
<invoiceId>  OP_DROP
OP_IF
  OP_SHA256  <capsuleHash>  OP_EQUALVERIFY
  OP_DUP  OP_HASH160  <creatorPkh>  OP_EQUALVERIFY  OP_CHECKSIG
OP_ELSE
  OP_DUP  OP_HASH160  <buyerPkh>  OP_EQUALVERIFY  OP_CHECKSIG
OP_ENDIF
```

- **Claim path (IF)**: Creator daemon reveals capsule preimage + signature
- **Refund path (ELSE)**: Buyer reclaims after nLockTime timeout (default 72 blocks)

#### Purchase Flow (via MessageBox, Zero Chain Scanning)

```
Viewer Agent              MessageBox            Creator Daemon
    │                         │                       │
    │ ① Request               │                       │
    │ {videoId, chunkIdx,     │                       │
    │  buyerPubKey}           │                       │
    │────────────────────────>│──────────────────────>│
    │                         │                       │
    │                         │  ② Compute capsule    │
    │                         │  (ECDH + HKDF)        │
    │                         │  Build invoice        │
    │                         │                       │
    │ ③ Invoice               │                       │
    │ {capsuleHash, price,    │                       │
    │  htlcScript, nonce,     │                       │
    │  invoiceId}             │                       │
    │<────────────────────────│<──────────────────────│
    │                         │                       │
    │ ④ Build & broadcast     │                       │
    │    HTLC funding tx      │                       │
    │                         │                       │
    │ ⑤ Notify fundingTxId    │                       │
    │────────────────────────>│──────────────────────>│
    │                         │                       │
    │                         │  ⑥ Fetch tx by txid   │
    │                         │  Validate HTLC        │
    │                         │  Build & broadcast    │
    │                         │  claim tx             │
    │                         │                       │
    │ ⑦ Return claimTxId      │                       │
    │<────────────────────────│<──────────────────────│
    │                         │                       │
    │ ⑧ Fetch claim tx by     │                       │
    │    txid, extract        │                       │
    │    capsule from         │                       │
    │    scriptSig            │                       │
    │                         │                       │
    │ ⑨ Recover key:          │                       │
    │ buyerMask = HKDF(       │                       │
    │   ECDH(D_buyer,         │                       │
    │        P_creator),      │                       │
    │   keyHash||nonce,       │                       │
    │   "bittok-buyer-mask")  │                       │
    │ aesKey = capsule        │                       │
    │          XOR buyerMask  │                       │
    │                         │                       │
    │ ⑩ Decrypt chunk, play   │                       │
```

#### Key Properties

| Property | Mechanism |
|----------|-----------|
| Atomicity | HTLC: Creator gets paid ⟺ capsule revealed |
| Buyer-specific | Capsule = aesKey XOR buyerMask(ECDH); on-chain but only buyer can recover aesKey |
| Unlinkability | Per-invoice nonce in HKDF prevents purchase correlation |
| Refund safety | nLockTime-based refund if Creator daemon fails to claim |
| Zero scanning | All communication via MessageBox; only specific txid lookups |
| Replay protection | invoiceId embedded in HTLC script |

### Layer 2: x402 + Payment Channel Download

CDN (or Creator) serves encrypted chunk data via HTTP 402 protocol:

```
Viewer → CDN:  GET /chunk/{videoId}/{chunkIndex}
CDN → Viewer:  402 Payment Required
               X-Price: <sats>
               X-Invoice-Id: <hex>
Viewer → CDN:  Payment via Payment Channel
CDN → Viewer:  200 OK + encrypted chunk data
```

- Payment Channels enable high-throughput off-chain micropayments
- CDN never holds decryption keys — serves only encrypted data
- Viewer decrypts locally after obtaining key from Layer 1

### Pricing & Competition

```
Example:
  Creator authorization fee:    50 sats/chunk
  Creator self-host download:   10 sats/chunk (x402)
  CDN-A download price:         8 sats/chunk
  CDN-B download price:         5 sats/chunk  ← Viewer agent picks cheapest

  Total cost per chunk:
    Via Creator direct: 50 + 10 = 60 sats
    Via CDN-B:          50 + 5  = 55 sats
```

- Authorization fee always goes to Creator (fixed by Creator)
- Download fee goes to data provider (competitive market)
- Viewer agent optimizes for lowest total cost + best reliability

## CDN Agent Economics

### Revenue Model

- CDN pays Creator's x402 endpoint to acquire encrypted chunks (one-time cost)
- CDN serves encrypted chunks to Viewers at its own download price (recurring revenue)
- Profit = (download fees × viewers served) − acquisition cost − operational costs

### CDN Does NOT Need Decryption Keys

CDN only handles encrypted data. It cannot see the content. This is a feature:
- Creator's IP is protected even from CDN
- CDN is a pure bandwidth/storage provider
- No trust required between Creator and CDN

## On-Chain Reviews

Only Viewers who have paid (have on-chain HTLC payment proof) can review.

### Review Transaction (OP_RETURN)

```
OP_RETURN
  <protocol_prefix>
  <action: "review">
  <videoId>
  <reviewerPubKey>
  <paymentTxRef>         # Reference to HTLC funding tx (proof of payment)
  <rating>               # 1–5
  <comment>              # Optional short text
```

On-chain verifiable: anyone can check `paymentTxRef` to confirm reviewer actually paid.

### Review Data Usage

- **Viewer Agent**: Trusted signal for recommendation (paid reviews only)
- **CDN Agent**: Content quality signal for caching decisions
- **Creator Agent**: Feedback for pricing strategy

## Agent AI Behaviors

### CDN Agent Decision Loop

```
[Periodic — every N minutes, LLM-driven]
  1. Scan on-chain metadata for new videos
  2. Evaluate each: creator reputation, reviews, category demand, competition
  3. Decide: cache or skip
  4. If cache: set download price (markup over acquisition cost)
  5. Broadcast availability via MessageBox

[Continuous — deterministic rules]
  - Serve x402 requests
  - Monitor revenue vs. cost per video
  - Drop underperforming content
  - Adjust prices based on demand (competitive response)
```

### Viewer Agent Recommendation

```
[Per-video decision — LLM-driven]
  Input:
    - User preference profile (liked tags, watch-through history, ratings given)
    - Candidate videos (on-chain metadata)
    - On-chain reviews (trusted, paid-only)
    - Available sellers + pricing (MessageBox)
  Output:
    - Ranked recommendation list with reasoning

[Per-chunk — deterministic]
  - Select best seller (weighted: price, latency, reliability)
  - Execute HTLC purchase + x402 download
  - After each chunk: continue or "swipe away"
  - Update preference profile based on behavior
```

### "Swipe Away" Mechanism

1-second chunks enable instant stop. Viewer agent decides after each chunk:
- **Continue**: preference signal strengthened
- **Swipe away**: preference signal weakened, stop paying immediately
- Zero waste — never pay for content you didn't watch

### LLM vs. Deterministic Boundary

| Behavior | Execution | Frequency |
|----------|-----------|-----------|
| CDN content selection | LLM | Every N minutes |
| CDN pricing strategy | LLM | Every N minutes |
| Viewer recommendation | LLM | Each time new video needed |
| Viewer post-watch review decision | LLM | After each video |
| Per-chunk payment + download | Deterministic | Every second |
| Seller selection | Weighted scoring | Each seller switch |
| MessageBox communication | Deterministic | Continuous |

## Streaming Pipeline

### Initial Buffering

```
t=0s  ──── Initial buffer (3-5 HTLC cycles in parallel) ────
      HTLC chunk 0  →  claim  →  decrypt  ┐
      HTLC chunk 1  →  claim  →  decrypt  ├─ buffer ready
      HTLC chunk 2  →  claim  →  decrypt  ┘
      → Start playback

t=1s  Play chunk 0  │  HTLC chunk 3
t=2s  Play chunk 1  │  HTLC chunk 4
t=3s  Play chunk 2  │  HTLC chunk 5
...
```

- Multiple HTLC cycles run in parallel (Creator daemon handles concurrently)
- Initial latency: ~3-5 seconds (buffer fill time)
- After buffer filled: smooth playback with rolling pre-purchase

### CDN Download Pipeline (Parallel)

While HTLC authorization proceeds, encrypted chunk downloads from CDN happen in parallel:

```
Authorization (Layer 1)          Download (Layer 2)
HTLC chunk 0  ─────────┐        x402 chunk 0  ─────────┐
HTLC chunk 1  ─────────┤        x402 chunk 1  ─────────┤
HTLC chunk 2  ─────────┤        x402 chunk 2  ─────────┤
                        ↓                                ↓
               Keys ready               Encrypted data ready
                        └───────── Decrypt + Play ───────┘
```

## Transaction Volume Estimation

| Metric | Value |
|--------|-------|
| Chunk duration | 1 second |
| On-chain txs per chunk purchase | 2 (HTLC funding + claim) |
| 30s video full view | 60 on-chain txs |
| Reviews per video | ~1 tx |
| CDN acquisition (off-chain) | 0 on-chain txs |
| Target: 1.5M txs | ~25,000 full 30s views |
| With multiple Viewer agents 24h | Achievable |

## System Boundary

| Component | In System (BitTok Core) | Out of System (Demo Script) |
|-----------|------------------------|-----------------------------|
| Creator publish API | ✅ Receive video → chunk → encrypt → on-chain metadata → x402 endpoint | — |
| Creator AI video generation | — | ✅ Remotion AI + call publish API |
| Creator Daemon | ✅ HTLC handling, x402 serving | — |
| CDN Agent | ✅ Full implementation | — |
| Viewer Agent | ✅ Full implementation | — |
| Web UI | ✅ Real-time dashboard | — |

## Technology Stack

- **Language**: TypeScript (full stack)
- **BSV Libraries**: `@bsv/sdk`, `@bsv/simple`, `@bsv/simple-mcp`
- **Frontend**: React / Next.js
- **Agent Runtime**: Node.js
- **AI**: LLM API calls for agent decision-making
- **Video Processing**: ffmpeg for chunking
- **Demo Content**: Remotion for AI video generation (external script)

## Demo Scenario

1. **Setup**: External script generates N short videos via Remotion, publishes through Creator publish API
2. **Creator Daemon**: Runs automatically, handles HTLC claims and serves content via x402
3. **CDN Agents**: Autonomously discover content, evaluate profitability, cache popular videos, compete on pricing
4. **Viewer Agents**: Each with unique preference profiles, autonomously browse/discover/pay/watch/review
5. **Web UI**: Real-time visualization of agent activity, transaction flow, recommendation decisions, market dynamics
6. **Human Interaction**: Humans can also browse and watch through the Web UI, triggering their own Viewer agent
