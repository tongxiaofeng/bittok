# BitTok — Detailed Design

## 1. Encryption

### 1.1 Chunk Encryption

Each 1-second chunk is independently encrypted with AES-256-GCM.

**Key derivation (Method 42 ECDH, per-video derived key)**:
```
# Per-video key derivation (master key never leaves Creator)
D_video = HKDF-SHA256(
  ikm  = D_creator,
  salt = videoId,                           # SHA256(SHA256(original video file))
  info = "bittok-video-key",
  len  = 32
)
P_video = D_video * G                       # corresponding public key

# Per-chunk encryption key
aesKey_i = HKDF-SHA256(
  ikm  = ECDH(D_video, P_video).x,         # shared secret x-coordinate
  salt = keyHash_i,                         # SHA256(SHA256(plaintext_chunk_i))
  info = "bittok-chunk-encryption",
  len  = 32
)
```

Creator delegates `D_video` to Curator. Curator can derive all chunk keys for that video, but cannot access other videos or Creator's master key.

**Encryption output format**:
```
nonce(12 bytes) || ciphertext || tag(16 bytes)
```

**AAD (Additional Authenticated Data)**: `keyHash_i` — binds ciphertext to content.

**Integrity verification**: After decryption, verify `SHA256(SHA256(plaintext)) == keyHash_i`.

### 1.2 Key Properties

- Each chunk has a unique AES key (derived from unique `keyHash_i`)
- Keys are deterministic — same Creator keypair + same plaintext always produces the same key
- Creator's private key `D_creator` never leaves the Creator's control (delegated to Curator for capsule computation)

## 2. HTLC + Capsule Protocol

### 2.1 Capsule Computation

Curator computes capsule using the per-video derived key (`D_video`) delegated by Creator:

```
# Per buyer, per chunk
capsule_i = aesKey_i XOR buyerMask

where:
  aesKey_i  = HKDF(ECDH(D_video, P_video), keyHash_i, "bittok-chunk-encryption")
  buyerMask = HKDF(ECDH(D_video, P_buyer), keyHash_i || nonce, "bittok-buyer-mask")
  nonce     = per-invoice random value (16 bytes)

capsuleHash_i = SHA256(SHA256(videoId || chunkIndex || capsule_i))
```

**Buyer-side key recovery**:
```
buyerMask = HKDF(ECDH(D_buyer, P_video), keyHash_i || nonce, "bittok-buyer-mask")
aesKey_i  = capsule_i XOR buyerMask
```

Works because `ECDH(D_video, P_buyer) == ECDH(D_buyer, P_video)` (ECDH symmetry).

Buyer needs `P_video` (included in invoice response) to recover the key.

### 2.2 HTLC Script (106 bytes)

```
<invoiceId>  OP_DROP
OP_IF
  OP_SHA256  <capsuleHash>  OP_EQUALVERIFY
  OP_DUP  OP_HASH160  <curatorPkh>  OP_EQUALVERIFY  OP_CHECKSIG
OP_ELSE
  OP_DUP  OP_HASH160  <buyerPkh>  OP_EQUALVERIFY  OP_CHECKSIG
OP_ENDIF
```

**Parameters**:
- `invoiceId` (16 bytes): Replay protection
- `capsuleHash` (32 bytes): SHA256(SHA256(videoId || chunkIndex || capsule))
- `curatorPkh` (20 bytes): Curator's P2PKH address (claim path)
- `buyerPkh` (20 bytes): Buyer's P2PKH address (refund path)

**Execution paths**:

| Path | Selector | Unlocking Script | Use Case |
|------|----------|-----------------|----------|
| Claim (IF) | `OP_TRUE` | `<sig> <curatorPubKey> <videoId\|\|chunkIdx\|\|capsule> OP_TRUE` | Curator claims, reveals capsule |
| Refund (ELSE) | `OP_FALSE` | `<sig> <buyerPubKey> OP_FALSE` | Buyer refunds after timeout |

**Timeout**: Enforced via `nLockTime` on refund transaction (default 72 blocks, range [6, 288]).

### 2.3 Claim Transaction (Atomic Split)

```
Claim Transaction:
  Input 0:  HTLC UTXO
            scriptSig: <curatorSig> <curatorPubKey> <videoId||chunkIdx||capsule> OP_TRUE

  Output 0: creator_share → P2PKH(P_creator)     # creatorSharePercent %
  Output 1: curator_share → P2PKH(P_curator)     # (100 - creatorSharePercent) %
```

Price calculation:
```
total = ceil(satoshisPerKilobyte * chunkSizeBytes / 1024)
creator_share = floor(total * creatorSharePercent / 100)
curator_share = total - creator_share
```

### 2.4 Purchase Flow (Direct API to Curator)

```
Viewer Agent                               Curator Service
    │                                            │
    │ ① REQUEST (HTTP API)                       │
    │ {                                          │
    │   videoId,                                 │
    │   chunkIndex,                              │
    │   buyerPubKey                              │
    │ }                                          │
    │───────────────────────────────────────────>│
    │                                            │
    │                            ② Curator computes:
    │                            - capsule (ECDH+HKDF)
    │                            - capsuleHash
    │                            - HTLC script
    │                            - invoice
    │                                            │
    │ ③ INVOICE                                  │
    │ {                                          │
    │   capsuleHash,                             │
    │   satoshisPerKilobyte,                     │
    │   chunkSizeBytes,                          │
    │   htlcScript,                              │
    │   nonce,                                   │
    │   invoiceId,                               │
    │   P_video,                                 │
    │   creatorSharePercent                      │
    │ }                                          │
    │<───────────────────────────────────────────│
    │                                            │
    │ ④ Viewer builds HTLC                       │
    │    funding tx:                             │
    │    Input: Viewer UTXO                      │
    │    Output: HTLC script                     │
    │    Broadcast to network                    │
    │                                            │
    │ ⑤ FUNDING_NOTIFY                           │
    │ { fundingTxId }                            │
    │───────────────────────────────────────────>│
    │                                            │
    │                            ⑥ Curator:
    │                            - Fetch tx by txid
    │                            - Validate HTLC
    │                            - Build claim tx
    │                              (atomic split)
    │                            - Broadcast
    │                                            │
    │ ⑦ CLAIM_NOTIFY                             │
    │ { claimTxId }                              │
    │<───────────────────────────────────────────│
    │                                            │
    │ ⑧ Viewer:                                  │
    │ - Fetch claim tx by txid                   │
    │ - Extract capsule from scriptSig           │
    │ - Compute buyerMask                        │
    │ - Recover aesKey_i                         │
    │ - Decrypt chunk                            │
    │ - Play                                     │
```

### 2.5 Security Properties

| Property | Mechanism |
|----------|-----------|
| Atomicity | HTLC: Curator gets paid ⟺ capsule revealed |
| Atomic split | Claim tx outputs enforced by Curator; verifiable against on-chain metadata |
| Buyer-specific | Capsule uses ECDH(D_creator, P_buyer); on-chain but only buyer can recover aesKey |
| Unlinkability | Per-invoice nonce prevents purchase correlation |
| Refund safety | nLockTime-based unilateral refund |
| Zero scanning | All communication via MessageBox; only specific txid lookups |
| Replay protection | invoiceId embedded in HTLC script |
| Verifiable split | creatorSharePercent declared in on-chain Metanet metadata |

## 3. x402 Download Protocol

### 3.1 Request/Response

```
GET /chunk/{videoId}/{chunkIndex} HTTP/1.1
Host: cdn.example.com

HTTP/1.1 402 Payment Required
X-Price-Per-KB: 1
X-File-Size: 204800
X-Invoice-Id: a3f2...
X-Expiry: 1712500000
```

### 3.2 Price Calculation

```
total = ceil(X-Price-Per-KB * X-File-Size / 1024)
```

### 3.3 Payment Channel

Viewer and CDN maintain an off-chain payment channel for high-throughput micropayments. Individual chunk downloads settle within the channel; on-chain settlement happens periodically.

## 4. Agent AI Behaviors

### 4.1 CDN Agent

```
[Periodic — every N minutes, LLM-driven]
  1. Query Curator API for new videos
  2. Evaluate: creator reputation, category demand, existing competition
  3. Decide: cache or skip
  4. If cache: download encrypted chunks from chain (by txid)
  5. Set download price (markup over storage/bandwidth cost)
  6. Broadcast availability via MessageBox

[Continuous — deterministic]
  - Serve x402 requests
  - Monitor revenue vs. cost per video
  - Drop underperforming content
  - Adjust prices in response to competition
```

### 4.2 Viewer Agent

```
[Per-video decision — LLM-driven]
  Input:
    - User preference profile (liked tags, watch-through history)
    - Candidate videos (from Curator API)
    - Available sellers + pricing (from MessageBox)
  Output:
    - Ranked recommendation list

[Per-chunk — deterministic]
  - Select best seller: weighted score(price, latency, reliability)
  - Execute HTLC purchase (via Curator) + x402 download (via CDN)
  - After each chunk: continue or "swipe away"
  - Update preference profile
```

### 4.3 "Swipe Away"

After each 1-second chunk, Viewer agent evaluates:
- **Continue**: Pay next chunk, strengthen preference signal
- **Swipe away**: Stop paying immediately, weaken preference signal

### 4.4 LLM vs. Deterministic Boundary

| Behavior | Execution | Frequency |
|----------|-----------|-----------|
| CDN content selection | LLM | Every N minutes |
| CDN pricing strategy | LLM | Every N minutes |
| Viewer recommendation | LLM | Each new video needed |
| Per-chunk payment + download | Deterministic | Every second |
| Seller selection | Weighted scoring | Each seller switch |
| MessageBox communication | Deterministic | Continuous |

## 5. On-Chain Data Hashing Convention

All data hashes throughout the system use double-hash (BSV standard):

```
hash(x) = SHA256(SHA256(x))
```

Applied to:
- `videoId = SHA256(SHA256(original_video_file))`
- `chunkHashes[i] = SHA256(SHA256(encrypted_chunk_i))`
- `keyHash_i = SHA256(SHA256(plaintext_chunk_i))`
- `capsuleHash_i = SHA256(SHA256(videoId || chunkIndex || capsule_i))`

## 6. Pricing Convention

All pricing uses `satoshisPerKilobyte`, consistent with `@bsv/sdk` `SatoshisPerKilobyte` class:

```
fee(data) = ceil(satoshisPerKilobyte * sizeInBytes / 1024)
```
