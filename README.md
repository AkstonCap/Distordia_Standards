# Distordia Open Source Asset Standards

Open specifications for blockchain-native assets on the [Nexus blockchain](https://nexus.io). These standards define how on-chain data is structured, verified, and indexed across the entire Distordia ecosystem -- and are free for anyone to adopt.

All web3 applications built by Distordia (and hopefully everyone else) follow these standards.

---

## Ecosystem Overview

```
                          DISTORDIA ASSET STANDARD ECOSYSTEM
                          ==================================

    IDENTITY LAYER                  APPLICATION LAYER                 AI LAYER
   ----------------               -------------------              -----------
  |                |             |                   |            |           |
  |   Namespace    |-- trust -->|  Content Verif.    |            |   Agent   |
  |  Attestation   |            |  Social Posts      |<-- owns --|  Registry |
  |                |-- owns --> |  Articles          |            |           |
  |  (namespace-   |            |  Product Masterdata|            | (agent-   |
  |   standard)    |            |  NFT Marketplace   |            |  standard)|
  |                |            |  Fantasy Football  |            |           |
   ----------------              -------------------               -----------
         |                              |                              |
         |                              |                              |
         v                              v                              v
   +---------------------------------------------------------------------------+
   |                  NEXUS BLOCKCHAIN (format=JSON / raw)                     |
   |                                                                           |
   |  - 1 KB max asset size       - Typed fields (uint8..uint1024, string)     |
   |  - Immutable + mutable       - No nested objects or arrays                |
   |  - Namespace-scoped           - POST API at api.distordia.com             |
   +---------------------------------------------------------------------------+
         |                    |                              |
         v                    v                              v
   +-------------------------+   +-------------------------------------------------+
   |    SWARM COORDINATION   |   |                 NEXGO P2P TRANSPORT              |
   |                         |   |                                                 |
   |  Swarm Registration     |   |  Taxi Registry -----> Ride Requests --> Ratings |
   |  Mission Contracts      |   |  (nexgo-taxi)         (nexgo-ride)    (nexgo-   |
   |  Escrow/Fees            |   |                       + Invoices API   rating)  |
   +-------------------------+   +-------------------------------------------------+
```

---

## Standards at a Glance

> **Market evaluation:** [Distordia Standards vs. Market Needs](docs/standards-market-evaluation.md) — a production-readiness and competitive-fit review of all non-product standards (vs. DID/VC, C2PA, ActivityPub/Nostr/Farcaster, Metaplex/EIP-2981, A2A/ERC-8004, Olas, and more). The product standard has its own [deep-dive](docs/product-masterdata-mrp-analysis.md). Cluster design notes: [Content & Publishing](docs/content-cluster-design-note.md) (Content · Articles · Social) and [NexGo Mobility](docs/nexgo-cluster-design-note.md) (Taxi · Ride · Rating). Nexus-platform alignment: [Namespace](docs/namespace-nexus-alignment-note.md) and [Agent & Swarm](docs/agent-swarm-nexus-alignment-note.md).

| # | Standard | File | Type Identifier | Fields | Purpose |
|---|----------|------|-----------------|--------|---------|
| 1 | [Namespace Attestation](#1-namespace-attestation) | [`namespace-standard.json`](standards/namespace-standard.json) | `namespace` | 21 | Identity, trust, and delegation |
| 2 | [Content Verification](#2-content-verification) | [`content-standard.json`](standards/content-standard.json) | `content` | 14 | Content provenance and anti-misinformation |
| 3 | [Social Posts](#3-social-posts) | [`social-standard.json`](standards/social-standard.json) | `distordia-post` | 20 | Decentralized social media |
| 4 | [Product Masterdata](#4-product-masterdata) | [`product-standard.json`](standards/product-standard.json) | `product` | 25 | Supply chain product registry (GS1/ISO) |
| 5 | [NFT Marketplace](#5-nft-marketplace) | [`nft-standard.json`](standards/nft-standard.json) | `nft` | 10 | Tokenized art and collectibles |
| 6 | [Fantasy Football Players](#6-fantasy-football-players) | [`player-standard.json`](standards/player-standard.json) | `distordia.player.v1` | 31 | Player NFT cards with live stats |
| 7 | [Agent Registration](#7-agent-registration) | [`agent-standard.json`](standards/agent-standard.json) | `agent` | 25 | AI agent identity and A2A trust |
| 8 | [Swarm Coordination](#8-swarm-coordination) | [`swarm-standard.json`](standards/swarm-standard.json) | `swarm` / `swarm-mission` | 18 / 26 | Multi-agent orchestration and contracts |
| 9 | [Articles](#9-articles) | [`article-standard.json`](standards/article-standard.json) | `distordia-article` / `distordia-article-chunk` | 11 / 5 | Long-form content via linked-list asset chains |
| 10 | [NexGo Taxi](#10-nexgo-taxi) | [`nexgo-taxi-standard.json`](standards/nexgo-taxi-standard.json) | `nexgo-taxi` | 9 | P2P ride-hailing vehicle registry |
| 11 | [NexGo Rating](#11-nexgo-rating) | [`nexgo-rating-standard.json`](standards/nexgo-rating-standard.json) | `nexgo-rating` | 3 (raw) | Passenger-to-driver rating system |
| 12 | [NexGo Ride](#12-nexgo-ride) | [`nexgo-ride-standard.json`](standards/nexgo-ride-standard.json) | `nexgo-ride` | 9 (raw) | Contractual ride requests with invoice payment |

**Total: 12 standards, 15 asset types, ~235 field definitions** (every asset type carries a `self-addr` field — see [Blockchain Constraints](#blockchain-constraints))

### v0.1.0 → v0.2.0 successor map

Every v0.1.0 file below is **deprecated** (it carries `"deprecated": true` and a `supersededBy`
pointer) and retained unchanged for history. Note that **agent** and **swarm** do not have
same-named successors — they were consolidated, and swarm was split across two:

| v0.1.0 standard | Superseded by | Nature of the change |
|---|---|---|
| `namespace-standard.json` | `namespace-standard.v0.2.0.json` | Bound to the native Nexus namespace register; hyphen-free pattern |
| `content-standard.json` | `content-standard.v0.2.0.json` | Unified anchor; off-chain body + C2PA + AI disclosure |
| `social-standard.json` | `social-standard.v0.2.0.json` | Adds follow graph + reactions |
| `article-standard.json` | `article-standard.v0.2.0.json` | Off-chain body; hardened chunk fallback |
| `product-standard.json` | `product-standard.v0.2.0.json` (+ `.solana.json`) | Minimal base identity layer + companion layers |
| `agent-standard.json` | **`agent-card-standard.v0.2.0.json`** | **Consolidated** — agent holds its own sigchain, so only discovery / endpoint binding / revocable trust remain |
| `swarm-standard.json` | **`agent-card-standard.v0.2.0.json`** (registration) **+ `agreement-standard.v0.2.0.json`** (missions) | **Split** — swarm becomes `kind:"swarm"` with token membership; missions become the generic agreement with native conditional-contract custody |
| `nexgo-taxi-standard.json` | `nexgo-taxi-standard.v0.2.0.json` | Registration only; GPS off-chain |
| `nexgo-ride-standard.json` | `nexgo-ride-standard.v0.2.0.json` + `agreement-standard.v0.2.0.json` | Domain request profile only; shared offer/agreement |
| `nexgo-rating-standard.json` | `nexgo-rating-standard.v0.2.0.json` | Verified-ride gated, typed, per-driver queryable |
| `nft-standard.json` | *(none yet)* | Market-evaluation recommendations only |
| `player-standard.json` | *(none yet)* | Market-evaluation recommendations only |

---

## Blockchain Constraints

All Distordia assets live on the Nexus blockchain under these constraints:

| Constraint | Value |
|-----------|-------|
| Maximum asset data size | **1 KB** |
| Format | `JSON` (typed fields with mutability control) or `raw` (opaque data blob) |
| Supported types | `uint8`, `uint16`, `uint32`, `uint64`, `uint256`, `uint512`, `uint1024`, `string`, `bytes` |
| Nested objects | Not supported on-chain |
| Arrays | Not supported (use comma/pipe-separated strings) |
| Booleans | `uint8` with values `0` or `1` |
| Timestamps | `uint64` Unix timestamps |
| Addressing | Register `address` is auto-assigned but **not queryable**; assets mirror it into a `self-addr` field |
| Namespace naming | Native Nexus namespaces are globally unique (1000 NXS) and allow **lowercase letters, digits and periods only — no hyphens** (`^[a-z][a-z0-9.]{2,31}$`) |
| API | POST requests to `https://api.distordia.com` |

**On-chain field format:**
```json
[
  {"name":"distordia-type","type":"string","value":"product","mutable":false,"maxlength":16},
  {"name":"self-addr","type":"string","value":"","mutable":true,"maxlength":56},
  {"name":"weight-kg","type":"uint32","value":250,"mutable":false},
  {"name":"status","type":"string","value":"active","mutable":true,"maxlength":12}
]
```

### Asset self-address (`self-addr`)

When an asset is created, Nexus auto-assigns a set of system attributes — `address` (the unique
register locator / primary key), `owner` (the creator's genesis hash), `type`, `form`, `version`,
`created`, and `modified`. These are returned by the API but are **not** part of the user-defined
field list, and the register `address` is **not a filterable column** in `register/list/assets` —
so an asset cannot be located or cross-referenced by its own address through a query.

Every Distordia asset therefore **duplicates its register address into a normal, queryable field
named `self-addr`**. Because the address is unknown until the create transaction confirms, this is a
deliberate two-step write:

```bash
# 1. Create — Nexus returns the new register address
assets/create/asset format=JSON name=product-WIDGET-2000 json='[...]'

# 2. Stamp the returned address into the asset's own self-addr field
assets/update/asset address=<returned-address> self-addr="<returned-address>"
```

`self-addr` is `mutable: true` (written exactly once, post-create, then frozen by convention). All
inter-asset links (e.g. `reply-to`, `quote`, `supersedes`, `replaced-by`) store the *target's*
`self-addr` value, so consumers resolve a link with `WHERE self-addr = '<value>'`. The field is
deliberately **not** named `address`, to avoid colliding with the auto-assigned system attribute.

---

## Verification Tiers

Trust in the ecosystem flows from verified namespaces. Every namespace must hold a tier:

```
  TIER    NAME                  STAKE        ANNUAL    VERIFICATION       DELEGATE?
  ----    ----                  -----        ------    ------------       ---------
   L0     Verified Agent          0 DIST      1 DIST  Automated          No
   L1     Verified Individual   100 DIST     10 DIST  Automated          No
   L2     Verified Organization 2,500 DIST  250 DIST  Automated + domain Yes (5 sub)
   L3     Verified Enterprise  25,000 DIST 2,500 DIST Legal entity       Yes (unlimited)
   L4     Human Attested       50,000 DIST 5,000 DIST Human expert       Yes (unlimited)
```

Higher tiers unlock more capabilities: sub-namespaces, delegation, SLA guarantees, and legal enforceability.

---

## Standard Details

### 1. Namespace Attestation

> **File:** [`standards/namespace-standard.json`](standards/namespace-standard.json)
> **Type:** `distordia-type: "namespace"`
> **Purpose:** Core identity primitive. All trust propagates from verified namespaces.

The namespace standard is the foundation of the entire Distordia ecosystem. Every organization, individual, agent, or swarm must first establish a verified namespace before registering other assets.

> ⚠️ **Nexus alignment (important):** Nexus already provides a native, globally-unique, transferable
> **namespace object register** (1000 NXS anti-squatting fee), and it permits **only lowercase
> letters, numbers, and periods — no hyphens**. v0.1.0 re-implements namespaces as an unenforced
> string and uses hyphenated examples that are *unregistrable on-chain*. See the
> [Nexus alignment note](docs/namespace-nexus-alignment-note.md) and the corrected
> [`namespace-standard.v0.2.0.json`](standards/namespace-standard.v0.2.0.json).

**Key fields:**

| Field | Type | Mutable | Description |
|-------|------|---------|-------------|
| `namespace` | string | No | Unique identifier — **Nexus-legal**: `^[a-z][a-z0-9.]{2,31}$` (lowercase, digits, periods; **no hyphens**) |
| `tier` | string | Yes | Verification level (L0-L4) |
| `stake` | uint64 | Yes | DIST tokens staked |
| `entity-type` | string | No | `individual`, `organization`, `enterprise`, `agent`, `swarm` |
| `protocols` | string | Yes | Activated protocols (e.g., `content,masterdata,a2a`) |
| `delegation-chain` | string | No | Pipe-separated delegation path |
| `reputation` | uint16 | Yes | Score 0-1000 (starts at 500) |
| `attester` | string | No | Who issued the attestation |

**Example:**
```json
{
  "distordia-type": "namespace",
  "namespace": "acme.corp",
  "tier": "L2",
  "stake": 2500,
  "entity-type": "organization",
  "protocols": "content,masterdata,a2a",
  "reputation": 850,
  "attester": "distordia"
}
```

---

### 2. Content Verification

> **File:** [`standards/content-standard.json`](standards/content-standard.json)
> **Type:** `distordia-type: "content"`
> **Purpose:** On-chain content provenance to combat AI-generated misinformation.

Register articles, videos, images, and other content with SHA-256 hashes for tamper-proof verification. Anyone can look up a URL to check if it has been attested by a verified namespace.

**Key fields:**

| Field | Type | Mutable | Description |
|-------|------|---------|-------------|
| `url` | string | No | Canonical URL of verified content |
| `title` | string | No | Content title |
| `author` | string | No | Author name/identifier |
| `hash` | string | No | `sha256:...` hash for integrity |
| `content-type` | string | No | `article`, `video`, `image`, `audio`, `document`, `dataset`, `other` |
| `status` | string | Yes | `official`, `user`, `pending`, `revoked` |
| `license` | string | No | e.g., `CC-BY-4.0`, `ARR` |
| `supersedes` | string | No | Address of previous version |

**Example:**
```json
{
  "distordia-type": "content",
  "status": "official",
  "url": "https://example.com/news/article-12345",
  "title": "Breaking: Major Discovery in Climate Research",
  "author": "Jane Smith",
  "hash": "sha256:a1b2c3d4e5f6...",
  "content-type": "article",
  "lang": "en"
}
```

---

### 3. Social Posts

> **File:** [`standards/social-standard.json`](standards/social-standard.json)
> **Type:** `distordia-type: "distordia-post"`
> **Purpose:** Decentralized social media posts stored permanently on-chain.

Supports threading (replies), quotes, reposts, media attachments, hashtags, mentions, location, and polls. Every post is an immutable blockchain asset owned by its creator.

**Key fields:**

| Field | Type | Mutable | Description |
|-------|------|---------|-------------|
| `text` | string | No | Post content (max 512 chars) |
| `distordia-status` | string | Yes | `official`, `user-post`, `deleted`, `hidden` |
| `reply-to` | string | No | Address of parent post |
| `quote` | string | No | Address of quoted post |
| `repost` | string | No | Address of original (for reposts) |
| `media-type` | string | No | `none`, `image`, `video`, `audio`, `link` |
| `media-url` | string | No | Attached media URL |
| `media-hash` | string | No | `sha256:...` integrity hash |
| `tags` | string | No | Comma-separated hashtags |
| `poll-q` / `poll-opts` / `poll-exp` | string | No | Poll question, pipe-separated options, expiry date |

**Example:**
```json
{
  "distordia-type": "distordia-post",
  "distordia-status": "user-post",
  "text": "Just deployed my first smart contract on Nexus!",
  "tags": "blockchain,nexus,web3",
  "lang": "en"
}
```

---

### 4. Product Masterdata

> **File:** [`standards/product-standard.json`](standards/product-standard.json)
> **Type:** `distordia-type: "product"`
> **Purpose:** Global product registry based on GS1, UN/CEFACT, and ISO standards for supply chain traceability.

Products are registered as immutable, timestamped blockchain assets. Each product gets a unique on-chain address that serves as a universal reference across ERP and supply chain systems.

> **In-depth review:** [Product Master Data as the Base Layer of a Decentralized MRP Stack](docs/product-masterdata-mrp-analysis.md) — frames this standard as the minimal, stable product-identity base layer, draws the line between base-layer data and the MRP application layers (BOM, sourcing, planning, warehouse…) that stack on top, and proposes the small set of base-layer improvements plus a layered companion-standard model.
>
> **Next-gen draft:** [`product-standard.v0.2.0.json`](standards/product-standard.v0.2.0.json) implements that base-layer redesign (adds `mpn`, `mat-type`, `base-uom`, `rev`, stewardship). The current `product-standard.json` is v0.1.0 and is kept unchanged for history.
>
> **Multichain:** the same v0.2.0 schema realized on Solana — [`product-standard.v0.2.0.solana.json`](standards/product-standard.v0.2.0.solana.json). Identical field names, semantics, and version; only the on-chain encoding (Borsh PDA account) and platform primitives differ.

**Key fields:**

| Field | Type | Mutable | Description |
|-------|------|---------|-------------|
| `art-nr` | string | No | Article/SKU number (unique in namespace) |
| `gtin` | string | No | Global Trade Item Number (EAN/UPC, 8-14 digits) |
| `desc` | string | Yes | Product description |
| `mfr` | string | No | Manufacturer name |
| `cat` / `subcat` | string | Yes | Category / sub-category |
| `gpc` | string | No | GS1 Global Product Classification |
| `hs` | string | No | Harmonized System customs code |
| `origin` | string | No | ISO 3166-1 alpha-2 country |
| `uom` | string | No | Unit of measure (`EA`, `KG`, `LB`, `L`, `M`, `BOX`, etc.) |
| `weight-kg` | uint32 | No | Weight in grams (divide by 1000 for kg) |
| `length-mm` / `width-mm` / `height-mm` | uint32 | No | Dimensions in millimeters |
| `hazard` / `perish` | uint8 | No | Boolean flags (0/1) |
| `status` | string | Yes | `valid`, `disc`, `recall`, `pend`, `draft` |

**Example:**
```json
{
  "distordia-type": "product",
  "status": "valid",
  "art-nr": "WIDGET-2000-BLK",
  "gtin": "5901234123457",
  "desc": "Industrial Widget 2000 Series - Black Anodized Aluminum",
  "mfr": "Acme Industries",
  "cat": "industrial-components",
  "origin": "DE",
  "uom": "EA",
  "weight-kg": 250
}
```

---

### 5. NFT Marketplace

> **File:** [`standards/nft-standard.json`](standards/nft-standard.json)
> **Type:** `distordia-type: "nft"`
> **Purpose:** Tokenized digital art and collectibles for the on-chain marketplace.

NFTs follow a two-step tokenization process (~2 NXS) before they can be traded. Supports marketplace operations: list, bid, execute, and cancel orders.

**Key fields:**

| Field | Type | Mutable | Description |
|-------|------|---------|-------------|
| `nft-name` | string | No | Display name |
| `image` | string | No | Public HTTPS image URL |
| `category` | string | Yes | `art`, `photography`, `music`, `collectible`, `gaming`, `utility`, `domain`, `meme`, `other` |
| `collection` | string | Yes | Collection grouping |
| `creator` | string | No | Creator name/namespace |
| `status` | string | Yes | `active`, `burned`, `frozen` |

**Naming:** `nft-{slug}-{timestamp}` (e.g., `nft-cosmic-dragon-1738857600000`)

**Example:**
```json
{
  "distordia-type": "nft",
  "status": "active",
  "nft-name": "Cosmic Dragon #42",
  "description": "A rare cosmic dragon from the Distordia Originals collection",
  "image": "https://images.example.com/cosmic-dragon-42.png",
  "category": "art",
  "collection": "Distordia Originals",
  "creator": "ArtistAlpha"
}
```

---

### 6. Fantasy Football Players

> **File:** [`standards/player-standard.json`](standards/player-standard.json)
> **Type:** `standard: "distordia.player.v1"`
> **Purpose:** NFT player cards for blockchain-based fantasy football with live stats.

The most field-rich standard (30 fields). Player cards have six skill attributes (pace, shooting, passing, dribbling, defending, physical), rarity tiers, edition types, and live season statistics that update on-chain.

**Key fields:**

| Field | Type | Mutable | Description |
|-------|------|---------|-------------|
| `player-id` | string | No | Unique lowercase ID |
| `player-name` | string | No | Full display name |
| `team-id` / `team-name` | string | Yes | Current team (changes on transfer) |
| `league` | string | Yes | `epl`, `laliga`, `bundesliga`, `seriea`, `ligue1`, `mls`, `eredivisie`, `liganos` |
| `position` | string | No | `GK`, `DEF`, `MID`, `FWD` |
| `overall` | uint8 | Yes | Rating 1-99 |
| `pace` / `shooting` / `passing` / `dribbling` / `defending` / `physical` | uint8 | Yes | Skill ratings 1-99 |
| `rarity` | string | No | `common`, `rare`, `epic`, `legendary` |
| `edition` | string | No | `standard`, `in-form`, `totw`, `special`, `icon` |
| `goals` / `assists` / `cleansheets` / `appearances` | uint16 | Yes | Season stats |
| `week-pts` / `total-pts` | uint16/uint32 | Yes | Fantasy points |
| `mint-num` / `max-supply` | uint32 | No | Scarcity controls |

**Rarity Distribution:**
```
  common      60-74 overall    50% drop rate
  rare        75-84 overall    30% drop rate
  epic        85-89 overall    15% drop rate
  legendary   90-99 overall     5% drop rate
```

**Naming:** `distordia:player.{league}.{team-id}.{player-id}`

**Example:**
```json
{
  "standard": "distordia.player.v1",
  "player-id": "haaland",
  "player-name": "Erling Haaland",
  "team-id": "mancity",
  "league": "epl",
  "position": "FWD",
  "overall": 91,
  "rarity": "legendary",
  "season": "2024-2025"
}
```

---

### 7. Agent Registration

> **File:** [`standards/agent-standard.json`](standards/agent-standard.json)
> **Type:** `distordia-type: "agent"`
> **Purpose:** AI agent identity, capabilities, and permissions for agent-to-agent (A2A) trust.

Agents register on-chain with their capabilities, protocol endpoints, rate limits, transaction permissions, and kill-switch configuration. This enables verifiable agent-to-agent commerce and coordination.

**Key fields:**

| Field | Type | Mutable | Description |
|-------|------|---------|-------------|
| `agent-id` | string | No | Unique agent ID |
| `namespace` | string | No | Parent namespace |
| `agent-type` | string | No | `autonomous`, `supervised`, `tool`, `swarm-member`, `orchestrator` |
| `caps` | string | Yes | Comma-separated capabilities |
| `protocols` | string | Yes | Supported protocols (`a2a`, `mcp`) |
| `can-transact` | uint8 | Yes | Transaction permission (0/1) |
| `max-tx-value` | uint64 | Yes | Max transaction value in DIST |
| `rate-rpm` / `rate-rpd` | uint32 | Yes | Rate limits (per minute / per day) |
| `kill-switch` | uint8 | Yes | Emergency shutdown enabled (default: 1) |
| `kill-auth` | string | Yes | Namespaces authorized to kill |
| `endpoint-a2a` / `endpoint-mcp` / `endpoint-health` | string | Yes | Service endpoints |
| `model` | string | Yes | AI model identifier |
| `swarm-id` / `swarm-role` | string | Yes | Swarm membership |

**Example:**
```json
{
  "distordia-type": "agent",
  "agent-id": "logistics-coordinator-01",
  "namespace": "acme.corp",
  "agent-type": "autonomous",
  "status": "active",
  "caps": "route-optimization,dispatch,tracking",
  "protocols": "a2a,mcp",
  "can-transact": 1,
  "max-tx-value": 1000,
  "kill-switch": 1
}
```

---

### 8. Swarm Coordination

> **File:** [`standards/swarm-standard.json`](standards/swarm-standard.json)
> **Type:** `distordia-type: "swarm"` and `distordia-type: "swarm-mission"`
> **Purpose:** Multi-agent swarm registration and escrow-based mission contracts.

This standard defines two asset types:

**Swarm Registration** -- a collective of agents with shared capabilities and reputation:

| Field | Type | Mutable | Description |
|-------|------|---------|-------------|
| `swarm-id` | string | No | Unique swarm ID |
| `swarm-type` | string | No | `logistics`, `mfg`, `research`, `trading`, `service`, `general` |
| `members` | string | Yes | Pipe-separated agent IDs |
| `consensus` | string | Yes | `leader`, `quorum`, `unanimous`, `weighted` |
| `stake` | uint64 | Yes | Total DIST staked |
| `missions-done` / `missions-failed` | uint32 | Yes | Track record |
| `reputation` | uint16 | Yes | Score 0-1000 |

**Mission Contract** -- an escrow-backed work agreement between swarms:

| Field | Type | Mutable | Description |
|-------|------|---------|-------------|
| `mission-id` | string | No | Unique mission ID |
| `contractor` / `contractee` | string | No | Buyer / seller namespaces |
| `deliverable` | string | No | What must be delivered |
| `payment` | uint64 | No | Amount in base units |
| `payment-type` | string | No | `upfront`, `complete`, `milestone`, `escrow` |
| `deadline-ts` | uint64 | No | Deadline timestamp |
| `progress` | uint8 | Yes | 0-100% completion |
| `status` | string | Yes | `proposed`, `accepted`, `active`, `done`, `failed`, `disputed`, `cancelled` |

**Fees:**
| Fee | Amount | Note |
|-----|--------|------|
| Swarm registration | 25 DIST | One-time |
| Mission escrow | 2% | Of escrowed amount |
| Dispute filing | 50 DIST | Refundable if upheld |

---

### 9. Articles

> **File:** [`standards/article-standard.json`](standards/article-standard.json)
> **Types:** `distordia-type: "distordia-article"` and `distordia-type: "distordia-article-chunk"`
> **Purpose:** Long-form content that exceeds the 512-character post limit, stored as a linked-list chain of on-chain assets.

Articles solve the 1KB register limit by splitting text across multiple assets. A root asset holds the title, metadata, and first text chunk, while continuation chunk assets carry subsequent text. Each asset points to the next via the `next` field, forming a singly-linked list. Chunks are created in reverse order so each can reference the address of the next one.

**Root article (`distordia-article`):**

| Field | Type | Mutable | Description |
|-------|------|---------|-------------|
| `title` | string | No | Article title (max 64 chars) |
| `text` | string | No | First text chunk (max 384 chars) |
| `distordia-status` | string | Yes | `official`, `deleted`, `hidden` |
| `cw` | string | No | Content warning |
| `reply-to` | string | No | Address of post/article being replied to |
| `quote` | string | No | Address of post/article being quoted |
| `tags` | string | No | Comma-separated hashtags |
| `lang` | string | No | ISO 639-1 language code |
| `next` | string | No | Address of first continuation chunk (empty if article fits in one asset) |

**Continuation chunk (`distordia-article-chunk`):**

| Field | Type | Mutable | Description |
|-------|------|---------|-------------|
| `text` | string | No | Text chunk (max 768 chars) |
| `distordia-status` | string | Yes | `official`, `deleted`, `hidden` |
| `next` | string | No | Address of next chunk (empty for last chunk) |

**Chunking limits:**
```
  Root asset text:    384 characters  (metadata overhead uses remaining space)
  Chunk asset text:   768 characters  (lean structure, more text per asset)
  Max article length: 5,000 characters
  Cost per asset:     1 NXS
```

**Example:**
```json
{
  "distordia-type": "distordia-article",
  "distordia-status": "official",
  "title": "Introduction to Decentralized Identity",
  "text": "Decentralized identity puts users in control of their digital presence...",
  "tags": "identity,blockchain,web3",
  "lang": "en",
  "next": "a3c4d5e6f7g8h9i0j1k2l3m4n5o6p7q8r9s0t1u2v3w4x5y6z7"
}
```

---

### 10. NexGo Taxi

> **File:** [`standards/nexgo-taxi-standard.json`](standards/nexgo-taxi-standard.json)
> **Type:** `distordia-type: "nexgo-taxi"`
> **Purpose:** Decentralized vehicle registry for the NexGo P2P ride-hailing module. Drivers register vehicles and broadcast GPS positions on-chain.

Each taxi is a JSON-format asset that drivers create and continuously update with their current GPS position and availability status. Passengers query available taxis in real-time through the Nexus register API.

**Key fields:**

| Field | Type | Mutable | Description |
|-------|------|---------|-------------|
| `vehicle-id` | string | Yes | License plate / vehicle identifier |
| `vehicle-type` | string | Yes | `sedan`, `suv`, `van`, `luxury` |
| `price-per-km` | string | Yes | Price per kilometer in NXS |
| `status` | string | Yes | `available`, `occupied`, `offline` |
| `latitude` | string | Yes | GPS latitude as decimal string |
| `longitude` | string | Yes | GPS longitude as decimal string |
| `driver` | string | Yes | Driver's wallet address or namespace |

**Naming:** `nexgo-taxi-{vehicleId}` (e.g., `nexgo-taxi-ABC-1234`)

**Example:**
```json
{
  "distordia-type": "nexgo-taxi",
  "vehicle-id": "ABC-1234",
  "vehicle-type": "sedan",
  "price-per-km": "2.5",
  "status": "available",
  "latitude": "40.712800",
  "longitude": "-74.006000",
  "driver": "john.driver"
}
```

---

### 11. NexGo Rating

> **File:** [`standards/nexgo-rating-standard.json`](standards/nexgo-rating-standard.json)
> **Type:** `distordia-type: "nexgo-rating"`
> **Format:** `raw` (state register, updatable)
> **Purpose:** Passenger-to-driver rating system. Each passenger has one rating asset that stores scores for all drivers they have rated.

Unlike JSON-format standards, this uses raw format -- the entire payload is stored as a JSON string in the asset's `data` field. This allows nested objects (the ratings map) which aren't possible with typed JSON fields. Ratings are aggregated across all passengers to compute per-driver averages.

**Data format:**
```json
{
  "distordia-type": "nexgo-rating",
  "ratings": {
    "<driver-genesis-hash>": { "score": 4, "avoid": false },
    "<driver-genesis-hash>": { "score": 1, "avoid": true }
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `distordia-type` | string | Always `nexgo-rating` |
| `ratings` | object | Map of driver genesis hash to rating entry |
| `ratings[].score` | number | Rating 1 (worst) to 5 (best) |
| `ratings[].avoid` | boolean | If true, driver is flagged as "to be avoided" |

**Naming:** `nexgo-ratings` (one per passenger, local to their signature chain)
**Cost:** 1 NXS (create) + 1 NXS (name). Updates cost only the transaction fee.

**Aggregation:** The app queries all raw assets via `register/list/assets:raw`, parses each one, and computes per-driver average scores and avoid counts.

---

### 12. NexGo Ride

> **File:** [`standards/nexgo-ride-standard.json`](standards/nexgo-ride-standard.json)
> **Type:** `distordia-type: "nexgo-ride"`
> **Format:** `raw` (state register, updatable)
> **Status:** Draft (designed, not yet implemented)
> **Purpose:** Contractual ride requests between passengers and drivers, using the Nexus Invoices API for atomic on-chain payment.

The ride standard defines the full decentralized ride flow: passenger creates a ride request, driver accepts, driver creates an invoice, passenger pays atomically on-chain, and the ride completes. Payment is guaranteed by the blockchain via conditional contracts (no escrow needed).

**Data format:**
```json
{
  "distordia-type": "nexgo-ride",
  "pickup-lat": "40.7128",
  "pickup-lng": "-74.0060",
  "dest-lat": "40.7589",
  "dest-lng": "-73.9851",
  "passengers": 2,
  "status": "requesting",
  "driver-genesis": ""
}
```

| Field | Type | Description |
|-------|------|-------------|
| `distordia-type` | string | Always `nexgo-ride` |
| `pickup-lat` / `pickup-lng` | string | Pickup GPS coordinates |
| `dest-lat` / `dest-lng` | string | Destination GPS coordinates |
| `passengers` | number | Number of passengers |
| `status` | string | `requesting`, `accepted`, `paid`, `cancelled` |
| `driver-genesis` | string | Driver's genesis hash (empty until accepted) |

**Naming:** `nexgo-ride-{timestamp}` (one per ride, local to passenger's signature chain)

**Ride lifecycle:**
1. Passenger creates ride request (status: `requesting`)
2. Driver accepts -- ride asset updated with driver genesis (status: `accepted`)
3. Driver creates invoice via `invoices/create/invoice`
4. Passenger pays via `invoices/pay/invoice` -- atomic DEBIT + CLAIM (status: `paid`)
5. Ride completes -- driver taxi asset returns to `available`, passenger can rate
6. Cancellation: driver cancels invoice, ride status becomes `cancelled`

---

## How Standards Relate

```
                            +-----------------+
                            |    Namespace     |
                            |   Attestation    |
                            | (identity root)  |
                            +--------+--------+
                                     |
              +-------------+--------+--------+-------------+
              |             |                 |             |
              v             v                 v             v
        +-----------+ +-----------+     +-----------+ +-----------+
        |  Content  | |  Product  |     |   Agent   | |   NexGo   |
        |  Verif.   | | Masterdata|     | Registry  | |   Taxi    |
        +-----------+ +-----------+     +-----+-----+ +-----------+
        |  Social   | |    NFT    |           |       |   NexGo   |
        |   Posts   | | Marketplace|          v       |   Rating  |
        +-----------+ +-----------+     +-----------+ +-----------+
        | Articles  | |  Fantasy  |     |   Swarm   | |   NexGo   |
        | (linked   | | Football  |     |  + Mission| |   Ride    |
        |  chain)   | |           |     |           | | + Invoice |
        +-----------+ +-----------+     +-----------+ +-----------+

  LEGEND:
  -------
  Namespace  -->  owns/trusts all assets below it
  Agent      -->  can join Swarms and execute Missions
  Articles   -->  linked-list of root + chunk assets for long-form content
  NexGo      -->  P2P transport: taxi registry, ratings (raw), rides + invoices
  All assets -->  live on Nexus blockchain (1KB, JSON or raw format)
```

---

## Nexus API Quick Reference

**Create an asset** (then stamp its `self-addr`, see [Asset self-address](#asset-self-address-self-addr)):
```bash
assets/create/asset format=JSON name=product-WIDGET-2000 json='[
  {"name":"distordia-type","type":"string","value":"product","mutable":false,"maxlength":16},
  {"name":"self-addr","type":"string","value":"","mutable":true,"maxlength":56},
  {"name":"art-nr","type":"string","value":"WIDGET-2000","mutable":false,"maxlength":32},
  {"name":"desc","type":"string","value":"Industrial Widget","mutable":true,"maxlength":256}
]'
# Returns the new register address; stamp it back into self-addr:
assets/update/asset address=<returned-address> self-addr="<returned-address>"
```

**Query assets:**
```javascript
const response = await fetch('https://api.distordia.com/register/list/assets:asset', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    where: "distordia-type = 'product'",
    limit: 100
  })
});
```

**Common endpoints:**

| Endpoint | Description |
|----------|-------------|
| `assets/create/asset` | Create new asset |
| `assets/update/asset` | Update mutable fields |
| `assets/get/asset` | Get asset by address |
| `register/list/assets:asset` | List/search assets |
| `assets/tokenize/asset` | Tokenize for trading |
| `market/create/ask` | List for sale |
| `market/create/bid` | Place buy order |
| `market/execute/order` | Execute matched order |

---

## Core Principles

1. **Open Standards** -- All schemas are publicly documented and free to use (MIT license)
2. **Namespace-Centric** -- Trust propagates from verified namespaces to all registered assets
3. **Agent-Readable** -- Structured typed fields for both human and AI agent consumption
4. **On-Chain First** -- All core data lives on the Nexus blockchain using `format=JSON`
5. **Versioned** -- All standards include version identifiers for forward compatibility
6. **Typed Fields** -- Proper numeric types (`uint8`-`uint1024`) avoid string parsing overhead
7. **1KB Discipline** -- Every standard is designed to fit within the blockchain's 1KB asset limit

---

## Repository Structure

```
Distordia_Standards/
  README.md                              # This file
  LICENSE                                # MIT License
  standards/
    namespace-standard.json              # Identity and trust v0.1.0 (21 fields)
    namespace-standard.v0.2.0.json       # Namespace v0.2.0 draft (bound to native Nexus namespace register)
    content-standard.json                # Content provenance v0.1.0 (14 fields)
    content-standard.v0.2.0.json         # Content v0.2.0 unified anchor draft (off-chain body + C2PA)
    social-standard.json                 # Social media posts v0.1.0 (20 fields)
    social-standard.v0.2.0.json          # Social v0.2.0 draft (+ follow graph + reactions)
    product-standard.json                # Product masterdata v0.1.0 (25 fields)
    product-standard.v0.2.0.json         # Product masterdata v0.2.0 base-layer draft (24 fields)
    product-standard.v0.2.0.solana.json  # Product masterdata v0.2.0 on Solana (same schema, Borsh PDA)
    nft-standard.json                    # NFT marketplace (10 fields)
    player-standard.json                 # Fantasy football (31 fields)
    agent-standard.json                  # AI agent registry v0.1.0 (25 fields)
    agent-card-standard.v0.2.0.json      # Agent Card v0.2.0 draft (agent + swarm; discovery/binding/trust)
    swarm-standard.json                  # Swarm + missions v0.1.0 (18 + 26 fields)
    agreement-standard.v0.2.0.json       # Agreement v0.2.0 draft (request/offer/agreement; shared by swarm + NexGo)
    article-standard.json               # Long-form articles v0.1.0 (11 + 5 fields)
    article-standard.v0.2.0.json        # Articles v0.2.0 draft (off-chain body + hardened chunk fallback)
    nexgo-taxi-standard.json            # NexGo taxi registry v0.1.0 (9 fields)
    nexgo-taxi-standard.v0.2.0.json     # NexGo taxi v0.2.0 draft (GPS off-chain, compliance creds)
    nexgo-rating-standard.json          # NexGo passenger ratings v0.1.0 (raw, 3 fields)
    nexgo-rating-standard.v0.2.0.json   # NexGo rating v0.2.0 draft (verified-ride gated, typed)
    nexgo-ride-standard.json            # NexGo ride requests v0.1.0 (raw, 9 fields)
    nexgo-ride-standard.v0.2.0.json     # NexGo ride v0.2.0 draft (request/offer/agreement, driver-signed)
```

> **v0.2.0 drafts** implement the redesigns from the cluster design notes
> ([content](docs/content-cluster-design-note.md), [NexGo](docs/nexgo-cluster-design-note.md)) and
> the [product MRP analysis](docs/product-masterdata-mrp-analysis.md), plus the Nexus alignment notes
> for [namespace](docs/namespace-nexus-alignment-note.md) and
> [agent & swarm](docs/agent-swarm-nexus-alignment-note.md). Each keeps its v0.1.0 file unchanged for
> history.
>
> **Consolidation in v0.2.0:** because a Distordia agent holds its **own sigchain**, identity,
> signing, and settlement are native Nexus capabilities. The `agent` and `swarm` standards therefore
> collapse into two: **`agent-card`** (discovery, endpoint/key binding, revocable credential — plus
> swarms via `kind: "swarm"`) and a generic **`agreement`** (request → counterparty-signed offer →
> terms-locked agreement, with custody delegated to native conditional contracts). NexGo Ride reuses
> the shared agreement instead of its own offer/agreement types.
>
> The remaining standards (nft, player) have
> [market-evaluation](docs/standards-market-evaluation.md) recommendations but no v0.2.0 draft yet.

---

## Distordia dApps Using These Standards

| Application | Standard(s) Used | Repository |
|-------------|-----------------|------------|
| Content Verification | Content, Namespace | [distordia_com](https://github.com/AkstonCap/distordia_com) |
| Distordia Social | Social, Articles, Namespace | [distordiaSocial](https://github.com/AkstonCap/distordiaSocial) |
| NexGo | NexGo Taxi, NexGo Rating, NexGo Ride, Namespace | [NexGo](https://github.com/AkstonCap/NexGo) |
| Product Masterdata | Product, Namespace | [distordia_com](https://github.com/AkstonCap/distordia_com) |
| NFT Marketplace | NFT, Namespace | [distordia_com](https://github.com/AkstonCap/distordia_com) |
| Fantasy Football | Player, Namespace | [distordia_com](https://github.com/AkstonCap/distordia_com) |
| DEX | Namespace | [DEX](https://github.com/AkstonCap/DEX) |
| Swap Bridge | Namespace | [distordia_com](https://github.com/AkstonCap/distordia_com) |
| Q-Wallet | Namespace | [q-wallet](https://github.com/AkstonCap/q-wallet) |
| Q-Mobile | Namespace | [q-mobile](https://github.com/AkstonCap/q-mobile) |
| Verification Module | Namespace | [verificationModule](https://github.com/AkstonCap/verificationModule) |

---

## License

[MIT](LICENSE) -- Use these standards freely in your own projects.
