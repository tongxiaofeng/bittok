# BitTok — Conceptual Design

## 1. Vision

BitTok is a peer-to-peer short video platform. Creators publish videos, Viewers pay to watch — per second, with no subscriptions and no platform lock-in. AI agents act on behalf of each participant, autonomously discovering content, negotiating prices, and executing micropayments over BSV.

**Core Value Proposition**: No platform lock-in. Creators own their content and set their own terms — free to switch service providers at any time. Viewers pay only for what they watch, at 1-second granularity. Recommendation is powered by the Viewer's own AI agent — all browsing history and preference data belong to the user, never to a platform. No subscriptions, no censorship, no data harvesting — just peers trading value directly.

## 2. Problem Statement

Current video platforms (YouTube, TikTok, etc.) create lock-in for both sides:

- **Creators** depend on the platform for distribution, monetization, and audience. The platform sets the rules, takes a cut, and can demonetize or remove content at will.
- **Viewers** pay fixed subscriptions regardless of actual consumption. Their attention is the product — sold to advertisers, not serving the viewer's interest. Browsing history, preferences, and behavioral data are captured by the platform, used to train its recommendation algorithm and sold to third parties. The user has no ownership, no portability, and no control over their own data.

## 3. Core Concepts

### 3.1 Per-Second Micropayment

Videos are split into 1-second chunks. Viewers pay per chunk consumed. Stop watching = stop paying. Zero waste.

### 3.2 Separation of Roles

Four distinct roles, each with clear responsibilities and independent economic incentives:

| Role | Responsibility | Economic Model |
|------|---------------|----------------|
| **Creator** | Produce content, set pricing | Earns authorization fees (per-KB) from every view |
| **Curator** | Run authorization service, provide content discovery | Earns a share of authorization fees (atomic on-chain split with Creator) |
| **CDN** | Cache and distribute encrypted content | Earns download fees from Viewers |
| **Viewer** | Discover and consume content | Pays per second of content consumed |

### 3.3 Identity

Every participant is identified by a public key (BRC-100). This identity is:
- **Self-sovereign**: No registration with any authority. The public key itself is the identity.
- **Naturally present**: Appears in every transaction and message the role participates in
- **Portable**: Creator can switch Curators; Viewer can switch CDNs

### 3.4 Content Ownership

Content ownership is established by the Creator's Metanet identity. The Creator's public key is the parent node of all their videos. Even when content is hosted by a Curator, the on-chain Metanet structure proves who created it.

### 3.5 Two-Layer Payment

| Layer | What | Who Pays → Who | Mechanism |
|-------|------|----------------|-----------|
| Authorization | Decryption key for a chunk | Viewer → Creator + Curator (atomic split) | On-chain HTLC + Capsule |
| Download | Encrypted chunk data | Viewer → CDN | Off-chain x402 + Payment Channel |

Authorization fees are the content creator's revenue. Download fees are the infrastructure provider's revenue. The two are independent.

### 3.6 User-Owned Recommendation

In centralized platforms, the recommendation algorithm is a black box owned by the platform. User behavior data (watch history, preferences, interactions) is harvested and monetized by the platform, not by the user.

In BitTok, recommendation is performed by the Viewer's own AI agent:

- **Preference data stays local**: Watch history, like/dislike signals, "swipe away" patterns — all stored on the Viewer's own device, never uploaded to any server
- **Algorithm is the user's own agent**: The Viewer agent runs locally, learns the user's taste, and makes recommendations. The user can inspect, adjust, or replace their agent
- **No filter bubble imposed by platform**: The platform has no algorithm to push content that maximizes ad revenue or engagement metrics. The Viewer agent optimizes for the user's actual preferences
- **Portable**: If the user switches clients or agents, they take their preference data with them

### 3.7 No Platform Lock-In

- **Creators** can publish to any Curator, or multiple Curators simultaneously. Switching Curators does not affect content ownership (Metanet proves authorship).
- **Curators** compete on service quality and split ratio. They are replaceable infrastructure, not gatekeepers.
- **CDNs** compete on price and reliability. Any node can become a CDN by caching content.
- **Viewers** are free to use any CDN and any Curator's discovery service.

## 4. Role Interactions

```
Creator                 Curator          Blockchain          CDN               Viewer
  │                       │                  │                │                  │
  │ Encrypt chunks        │                  │                │                  │
  │ Store on-chain ──────────────────────>   │                │                  │
  │ Publish metadata ────────────────────>   │                │                  │
  │ Transfer keys ───────>│                  │                │                  │
  │ (can go offline)      │                  │                │                  │
  │                       │ Runs Overlay     │                │                  │
  │                       │ Runs HTLC auth   │                │                  │
  │                       │                  │                │                  │
  │                       │                  │<── download ───│                  │
  │                       │                  │  (by txid)     │                  │
  │                       │                  │                │                  │
  │                       │<─────────────────────────────────────── discover ───│
  │                       │<─────────────────────────────────────── auth fee ───│
  │                       │  claim tx ──────>│                │                  │
  │<── creator share ─────│  (atomic split)  │                │                  │
  │                       │                  │                │<── download fee ─│
  │                       │                  │                │  (x402+PayChan)  │
```

## 5. AI Agent Behavior (Conceptual)

Each role can be operated by an AI agent that makes autonomous decisions:

- **Creator Agent**: Chooses Curator, sets pricing strategy, adjusts split ratios based on market feedback
- **Curator Agent**: Curates content quality, manages infrastructure, optimizes service
- **CDN Agent**: Predicts popular content, decides what to cache, competes on pricing
- **Viewer Agent**: Learns user preferences, recommends videos, selects cheapest/fastest source

The division between AI decision-making and deterministic execution:
- **LLM-driven**: Strategic decisions (what to cache, what to recommend, how to price)
- **Deterministic**: Per-chunk operations (payment, download, decrypt, play) — executed every second, cannot afford LLM latency
