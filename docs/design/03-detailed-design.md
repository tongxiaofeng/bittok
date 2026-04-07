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

Creator delegates `D_video` to Curator via ECIES on-chain. Curator can derive all chunk keys for that video, but cannot access other videos or Creator's master key.

Note: `ECDH(D_video, P_video)` is self-ECDH — the shared secret is deterministic. This is intentional for compatibility with the Method 42 library API, which expects an ECDH-based key derivation pattern.

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

### 2.1 Batch Authorization

To reduce on-chain transaction volume, chunks are authorized in **batches** rather than individually. One HTLC covers multiple consecutive chunks.

**Batch size strategy (adaptive, like TCP slow start)**:
- First batch: 1-3 chunks (fast start, minimize risk)
- Subsequent batches: 5-10 chunks (if viewer continues watching)
- Viewer can stop at any batch boundary — pays for current batch, not the rest

A 30-second video with batch size 5 produces 6 batches = 12 on-chain txs (vs 60 without batching).

### 2.2 Capsule Computation

Curator computes a **batch capsule** that unlocks all chunk keys in the batch:

```
# Per buyer, per batch
batchKey = HKDF(ECDH(D_video, P_video), videoId || batchIndex, "bittok-batch-key")

# Individual chunk keys derived from batchKey
aesKey_i = HKDF(batchKey, keyHash_i, "bittok-chunk-encryption")

# Capsule wraps the batchKey (not individual chunk keys)
buyerMask = HKDF(ECDH(D_video, P_buyer), videoId || batchIndex || nonce, "bittok-buyer-mask")
capsule   = batchKey XOR buyerMask
nonce     = per-invoice random value (16 bytes)

capsuleHash = SHA256(SHA256(videoId || batchIndex || capsule))
```

**Buyer-side key recovery**:
```
buyerMask = HKDF(ECDH(D_buyer, P_video), videoId || batchIndex || nonce, "bittok-buyer-mask")
batchKey  = capsule XOR buyerMask
aesKey_i  = HKDF(batchKey, keyHash_i, "bittok-chunk-encryption")   # for each chunk in batch
```

Works because `ECDH(D_video, P_buyer) == ECDH(D_buyer, P_video)` (ECDH symmetry).

Buyer needs `P_video` (included in invoice response) to recover the key.

### 2.3 Free First Chunk (Instant Playback)

To eliminate startup latency, chunk 0 of every video is **unencrypted** (or its key is publicly available in the metadata). This enables:
- Instant playback when user taps a video (no HTLC needed for first second)
- HTLC for batch 0 (chunks 1-N) proceeds in background during first-second playback
- By the time chunk 0 finishes playing, batch 0 keys are ready

### 2.4 HTLC Script (108 bytes)

```
<invoiceId>  OP_DROP
OP_IF
  OP_HASH256  <capsuleHash>  OP_EQUALVERIFY
  OP_DUP  OP_HASH160  <curatorPkh>  OP_EQUALVERIFY  OP_CHECKSIG
OP_ELSE
  <locktime>  OP_CHECKLOCKTIMEVERIFY  OP_DROP
  OP_DUP  OP_HASH160  <buyerPkh>  OP_EQUALVERIFY  OP_CHECKSIG
OP_ENDIF
```

**Parameters**:
- `invoiceId` (16 bytes): Replay protection
- `capsuleHash` (32 bytes): SHA256(SHA256(videoId || batchIndex || capsule)) — verified by `OP_HASH256` which computes double-SHA256
- `curatorPkh` (20 bytes): Curator's P2PKH address (claim path)
- `buyerPkh` (20 bytes): Buyer's P2PKH address (refund path)

**Execution paths**:

| Path | Selector | Unlocking Script | Use Case |
|------|----------|-----------------|----------|
| Claim (IF) | `OP_TRUE` | `<sig> <curatorPubKey> <videoId\|\|batchIdx\|\|capsule> OP_TRUE` | Curator claims, reveals batch capsule |
| Refund (ELSE) | `OP_FALSE` | `<sig> <buyerPubKey> OP_FALSE` | Buyer refunds after timelock |

**Timeout**: Enforced by `OP_CHECKLOCKTIMEVERIFY` in the ELSE branch (default 6 blocks / ~1 hour, range [6, 72]). Short locktime is appropriate for micropayments — minimizes fund lockup for the Viewer while giving Curator sufficient time to claim.

### 2.5 Claim Transaction (Atomic Split with Covenant)

```
Claim Transaction:
  Input 0:  HTLC UTXO
            scriptSig: <curatorSig> <curatorPubKey> <videoId||batchIdx||capsule> OP_TRUE

  Output 0: creator_share → P2PKH(P_creator)     # creatorSharePercent %
  Output 1: curator_share → P2PKH(P_curator)     # (100 - creatorSharePercent) %
```

**Covenant enforcement**: The HTLC claim path uses `OP_PUSH_TX` (sighash preimage introspection) to verify that the claim transaction's outputs match the required split ratio. This is cryptographically enforced in-script — Curator cannot construct a claim tx that deviates from the declared `creatorSharePercent`. Implemented via Runar smart contract.

Price calculation:
```
batchSizeBytes = sum(chunkSizes[i] for i in batch)
total = ceil(satoshisPerKilobyte * batchSizeBytes / 1024)
creator_share = floor(total * creatorSharePercent / 100)
curator_share = total - creator_share
```

### 2.6 Purchase Flow (Direct API to Curator)

```
Viewer Agent                               Curator Service
    │                                            │
    │ ① REQUEST (HTTP API)                       │
    │ {                                          │
    │   videoId,                                 │
    │   batchIndex,                              │
    │   batchSize,                               │
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
    │   batchSizeBytes,                           │
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
    │ - Recover batchKey                          │
    │ - Derive aesKey_i for each chunk in batch  │
    │ - Decrypt chunks, play                     │
```

### 2.7 Security Properties

| Property | Mechanism |
|----------|-----------|
| Atomicity | HTLC: Curator gets paid ⟺ capsule revealed |
| Atomic split | Claim tx outputs enforced by OP_PUSH_TX covenant in-script; Curator cannot deviate |
| Buyer-specific | Capsule uses ECDH(D_video, P_buyer); on-chain but only buyer can recover aesKey |
| Unlinkability | Per-invoice nonce prevents purchase correlation |
| Refund safety | OP_CHECKLOCKTIMEVERIFY-enforced timelock in refund path |
| Zero scanning | CDN discovery via MessageBox, authorization via Curator API; only specific txid lookups on-chain |
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

### 3.3 Payment Method

Each x402 chunk download is paid via direct BSV micropayment (single on-chain transaction). The Viewer includes payment proof in the follow-up request.

**Future improvement**: For frequent Viewer↔CDN interactions, introduce payment channels or a CDN aggregator to reduce per-request on-chain overhead. A Viewer would maintain a single long-lived channel with an aggregator that routes payments to multiple CDNs.

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

## 5. Video Encoding Requirements

Chunk boundaries must align with video keyframes (I-frames) to ensure:
- Each chunk is independently decodable (no dependency on previous chunks)
- No decoding artifacts at chunk boundaries
- Optimal compression efficiency within each chunk

In practice, the video encoder (ffmpeg) is configured with `force_key_frames` at 1-second intervals to ensure GOP (Group of Pictures) boundaries coincide with chunk boundaries. Actual chunk sizes may vary due to scene complexity — this is why `chunkSizes[]` is included in the metadata.

## 6. On-Chain Data Hashing Convention

All data hashes throughout the system use double-hash (BSV standard):

```
hash(x) = SHA256(SHA256(x))
```

Applied to:
- `videoId = SHA256(SHA256(original_video_file))`
- `chunkHashes[i] = SHA256(SHA256(encrypted_chunk_i))`
- `keyHash_i = SHA256(SHA256(plaintext_chunk_i))`
- `capsuleHash = SHA256(SHA256(videoId || batchIndex || capsule))`

## 7. Pricing Convention

All pricing uses `satoshisPerKilobyte`, consistent with `@bsv/sdk` `SatoshisPerKilobyte` class:

```
fee(data) = ceil(satoshisPerKilobyte * sizeInBytes / 1024)
```
