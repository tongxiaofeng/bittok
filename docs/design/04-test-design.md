# BitTok — Test Design

## 1. Test Strategy

### 1.1 Test Layers

| Layer | Scope | Tools | Purpose |
|-------|-------|-------|---------|
| **Unit** | Single function / class | Vitest | Verify individual components in isolation |
| **Integration** | Multiple components, real BSV transactions | Vitest + BSV testnet/regtest | Verify component interactions and on-chain behavior |
| **End-to-End** | Full system, all four roles running | Vitest + test harness | Verify complete user scenarios |

### 1.2 BSV Test Environment

- Use BSV **regtest** (local) for fast iteration during development
- Use BSV **testnet** for pre-release integration validation
- Never run automated tests against mainnet

## 2. Unit Tests

### 2.1 Encryption Module

| Test Case | Input | Expected Output |
|-----------|-------|-----------------|
| Encrypt chunk | plaintext chunk + Creator keypair | nonce(12B) \|\| ciphertext \|\| tag(16B) |
| Decrypt chunk | encrypted chunk + correct aesKey | original plaintext |
| Decrypt with wrong key | encrypted chunk + wrong aesKey | Failure / authentication error |
| Key derivation determinism | same Creator keypair + same plaintext | identical aesKey every time |
| keyHash computation | plaintext chunk | SHA256(SHA256(plaintext)) |
| Different chunks produce different keys | two different plaintexts | two different aesKeys |

### 2.2 Capsule Module

| Test Case | Input | Expected Output |
|-----------|-------|-----------------|
| Capsule computation | aesKey + Creator keypair + Buyer pubkey + nonce | 32-byte capsule |
| Capsule hash | videoId + chunkIndex + capsule | SHA256(SHA256(videoId \|\| chunkIndex \|\| capsule)) |
| Buyer key recovery | capsule + Buyer keypair + Creator pubkey + nonce | original aesKey |
| ECDH symmetry | Creator priv + Buyer pub vs Buyer priv + Creator pub | identical shared secret |
| Different buyers produce different capsules | same chunk, two different Buyer pubkeys | two different capsules |
| Nonce unlinkability | same buyer, same chunk, different nonces | different capsules |

### 2.3 HTLC Script

| Test Case | Input | Expected Output |
|-----------|-------|-----------------|
| Build HTLC locking script | invoiceId + capsuleHash + curatorPkh + buyerPkh | Valid 106-byte Bitcoin Script |
| Claim path execution | correct capsule preimage + Curator signature | Script evaluates to true |
| Claim with wrong preimage | incorrect preimage + Curator signature | Script fails |
| Claim with wrong signature | correct preimage + wrong signature | Script fails |
| Refund path execution | Buyer signature (after nLockTime) | Script evaluates to true |
| Refund before timeout | Buyer signature (before nLockTime) | Transaction rejected |
| Replay protection | duplicate invoiceId | Script fails |

### 2.4 Claim Transaction Builder

| Test Case | Input | Expected Output |
|-----------|-------|-----------------|
| Atomic split outputs | HTLC UTXO + 70/30 split | Output 0: 70% to Creator, Output 1: 30% to Curator |
| Split rounding | odd total amount + 70/30 split | floor for Creator, remainder for Curator, no dust |
| Price calculation | satoshisPerKilobyte=10, chunkSize=200KB | ceil(10 * 200KB / 1024) sats |

### 2.5 Video Chunking

| Test Case | Input | Expected Output |
|-----------|-------|-----------------|
| Chunk 30s video | 30-second video file | 30 separate 1-second chunk files |
| Chunk count matches duration | N-second video | Exactly N chunks |
| Reassembly | 30 chunks in order | Identical to original video |
| Each chunk independently playable | single chunk | Valid 1-second media segment |

### 2.6 Metanet Metadata

| Test Case | Input | Expected Output |
|-----------|-------|-----------------|
| Build video metadata tx | video info + Creator/Curator keys | Valid Metanet node with TLV payload |
| TLV encode/decode roundtrip | metadata object | Identical after encode → decode |
| videoId computation | original video file | SHA256(SHA256(file)) |
| chunkHashes computation | encrypted chunks | SHA256(SHA256(chunk_i)) for each |

### 2.7 B:// and BCAT Storage

| Test Case | Input | Expected Output |
|-----------|-------|-----------------|
| B:// transaction build | chunk data ≤ 100KB | Valid tx with B:// prefix + data |
| B:// data retrieval | storage txid | Original chunk data |
| BCAT split | chunk data > 100KB | Multiple part txs + anchor tx |
| BCAT reassembly | anchor txid | Original chunk data |

### 2.8 Curator API

| Test Case | Input | Expected Output |
|-----------|-------|-----------------|
| Submit video | video metadata + per-video key | Video stored in catalog, key stored for auth |
| List videos | GET /videos | List of available videos with metadata |
| Query by tag | tag string | List of matching videos |
| Query by Creator pubkey | Creator's public key | All videos by that Creator |

## 3. Integration Tests

### 3.1 Creator → Curator Publishing

| Test Case | Description |
|-----------|-------------|
| Full publish flow | Creator encrypts video → stores chunks on-chain (B://BCAT) → publishes Metanet metadata → submits to Curator API (metadata + per-video key) → video appears in Curator catalog |
| Metadata matches content | chunkHashes in metadata match actual stored chunk hashes |
| chunkTxIds valid | Each txid in metadata resolves to a valid B:// / BCAT transaction |
| Identity linkage | Video Metanet node parent = Creator's identity node txid |

### 3.2 Viewer → Curator Authorization (HTLC Cycle)

| Test Case | Description |
|-----------|-------------|
| Full HTLC purchase cycle | Viewer requests → Curator invoices → Viewer funds HTLC → Curator claims → Viewer extracts capsule → Viewer recovers key → Viewer decrypts chunk |
| Atomic split verified | Claim tx outputs match declared creatorSharePercent |
| Creator receives funds | Creator's UTXO balance increases by creator_share |
| Curator receives funds | Curator's UTXO balance increases by curator_share |
| Refund after timeout | Curator doesn't claim within timeout → Viewer successfully refunds |
| Multiple concurrent HTLCs | 5 parallel HTLC cycles for chunks 0-4 → all complete correctly |

### 3.3 CDN Caching and Serving

| Test Case | Description |
|-----------|-------------|
| CDN discovers via Curator API | CDN queries Curator API → receives video list with chunkTxIds |
| CDN downloads from chain | CDN fetches encrypted chunks by txid → data matches chunkHashes |
| CDN serves via x402 | Viewer requests chunk from CDN → 402 response → payment → chunk delivered |
| CDN data integrity | Chunk served by CDN matches chunk stored on-chain (hash comparison) |

### 3.4 MessageBox Communication

| Test Case | Description |
|-----------|-------------|
| Signed message delivery | Sender signs message → Receiver verifies signature against sender's pubkey |
| Purchase request/response | Viewer sends request → Curator responds with invoice (correct fields) |
| CDN availability broadcast | CDN broadcasts availability → Viewer receives and parses |
| Funding notification | Viewer sends fundingTxId → Curator receives and processes |

## 4. End-to-End Tests

### 4.1 Scenario: Single Video, Single View

**Setup**: 1 Creator, 1 Curator, 1 CDN, 1 Viewer

1. Creator publishes a 10-second video to Curator
2. Curator stores 10 encrypted chunks on-chain, publishes metadata
3. CDN discovers video via Curator API, downloads chunks from chain and caches
4. Viewer discovers video via Curator API
5. Viewer purchases authorization for chunks 0-9 from Curator (HTLC)
6. Viewer downloads encrypted chunks from CDN (x402)
7. Viewer decrypts and plays all 10 chunks

**Assertions**:
- All 10 HTLC cycles complete successfully
- Creator receives 70% × 10 × chunk_price satoshis
- Curator receives 30% × 10 × chunk_price satoshis
- CDN receives download fees for 10 chunks
- Viewer decrypts all chunks correctly
- Total on-chain transactions = 10 (funding) + 10 (claim) + 10 (B:// storage) + 1 (metadata) = 31

### 4.2 Scenario: Swipe Away

**Setup**: 1 Creator, 1 Curator, 1 CDN, 1 Viewer

1. Creator publishes a 30-second video
2. Viewer starts watching, swipes away after 5 seconds

**Assertions**:
- Only 5 HTLC cycles executed (chunks 0-4)
- Creator and Curator receive payment for exactly 5 chunks
- CDN receives download fees for exactly 5 chunks
- No payment for chunks 5-29

### 4.3 Scenario: Multiple CDNs, Price Competition

**Setup**: 1 Creator, 1 Curator, 3 CDNs (different prices), 1 Viewer

1. Creator publishes video
2. All 3 CDNs cache the content, broadcast different prices
3. Viewer discovers prices via MessageBox
4. Viewer selects cheapest CDN

**Assertions**:
- Viewer agent selects the CDN with lowest price
- All downloads go to cheapest CDN
- If cheapest CDN goes offline mid-stream, Viewer falls back to next cheapest

### 4.4 Scenario: Curator Refund Safety

**Setup**: 1 Creator, 1 Curator (simulated failure), 1 Viewer

1. Viewer funds HTLC for chunk 0
2. Curator goes offline (simulated crash, no claim)
3. nLockTime expires
4. Viewer broadcasts refund transaction

**Assertions**:
- Viewer recovers full HTLC amount
- No capsule revealed (Viewer cannot decrypt chunk)
- Viewer retries with alternate mechanism or skips chunk

### 4.5 Scenario: Multiple Viewers Concurrent

**Setup**: 1 Creator, 1 Curator, 1 CDN, 10 Viewers (parallel)

1. Creator publishes a 15-second video
2. All 10 Viewers start watching simultaneously

**Assertions**:
- All 10 Viewers successfully complete HTLC cycles
- Each Viewer gets unique capsules (different nonces)
- Creator receives correct total (10 × 15 × creator_share)
- Curator handles concurrent MessageBox requests without error
- CDN serves 10 parallel x402 streams

## 5. Performance Tests

| Test Case | Metric | Target |
|-----------|--------|--------|
| HTLC cycle latency | Time from request to decrypted chunk | < 3 seconds |
| Curator concurrent throughput | Max concurrent HTLC cycles handled | ≥ 50 |
| CDN x402 throughput | Chunks served per second | ≥ 100 |
| Viewer buffer fill time | Time to fill 3-chunk buffer | < 5 seconds |
| Curator API query latency | Time to return video list | < 500ms |
| 24h sustained throughput | Total on-chain transactions in 24h | Measure and report |

## 6. Test Data

### 6.1 Test Videos

Generated by external script (Remotion):
- 5 × 10-second videos (quick tests)
- 5 × 30-second videos (standard tests)
- 2 × 60-second videos (max length tests)

### 6.2 Test Keypairs

Pre-generated for each role:
- 2 Creator keypairs
- 1 Curator keypair
- 3 CDN keypairs
- 10 Viewer keypairs

All test keypairs stored in a fixture file, never used on mainnet.

## 7. Test Execution Order

Tests should be run in this order, as later tests depend on earlier ones passing:

1. **Unit tests** — all modules independently
2. **Integration: Publishing** — Creator → Curator → on-chain
3. **Integration: Authorization** — HTLC cycle
4. **Integration: CDN** — caching and serving
5. **Integration: MessageBox** — communication
6. **End-to-end: Single view** — full happy path
7. **End-to-end: Edge cases** — swipe away, refund, competition
8. **End-to-end: Concurrency** — multiple viewers
9. **Performance tests** — throughput and latency
