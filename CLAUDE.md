# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

**BitTok** — A peer-to-peer short video platform where AI agents autonomously trade video content through BSV micropayments. No platform lock-in for creators or viewers.

Design docs:
- `docs/design/01-conceptual-design.md` — Vision, roles, core concepts
- `docs/design/02-system-design.md` — Identity, discovery, payment, storage architecture
- `docs/design/03-detailed-design.md` — HTLC script, capsule computation, protocols, AI behavior
- `docs/design/04-test-design.md` — Test strategy, unit/integration/E2E/performance test plans

## Architecture

Four roles:
- **Creator**: Publishes encrypted 1-second video chunks to a Curator, then can go offline. Retains content ownership via on-chain Metanet identity.
- **Curator**: Runs the authorization service (HTLC+Capsule key exchange on behalf of Creator) and Overlay discovery (SHIP/SLAP). Revenue-shares with Creator via atomic on-chain split. Does NOT store content or provide download.
- **CDN Agent**: Caches popular encrypted content from Curator, resells to Viewers via x402 + Payment Channel.
- **Viewer Agent**: AI-driven recommendation, pays Curator for decryption keys (on-chain HTLC with Creator/Curator atomic split), pays CDN for data (off-chain Payment Channel).

Two-layer payment:
- **Layer 1 (on-chain)**: Viewer → Curator HTLC authorization. Claim tx atomically splits output to Creator + Curator.
- **Layer 2 (off-chain)**: Viewer → CDN download via x402 + Payment Channel.

Content discovery via Curator's API. CDN availability via MessageBox P2P. On-chain Metanet metadata for verification/ownership proof, not discovery.

## Tech Stack

- TypeScript full stack (Node.js agents + React/Next.js frontend)
- `@bsv/sdk`, `@bsv/simple`, `@bsv/simple-mcp`
- **Runar** (runar.build) for smart contracts — TypeScript compiled to Bitcoin Script. NOT sCrypt.

## BSV Conventions (MUST follow)

- **Never scan mempool or chain.** All communication via MessageBox P2P; only fetch specific transactions by txid.
- **Double-hash** for all data hashes: `SHA256(SHA256(x))`
- **Metanet data structure** for all on-chain data (video metadata)
- **satoshisPerKilobyte** for all pricing (matches `@bsv/sdk` `SatoshisPerKilobyte` class)
- Follow existing BSV ecosystem standards (BRC-100 identity, TLV encoding, x402 protocol)
- Use "peer-to-peer", never "decentralized"
