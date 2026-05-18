# sdl-api

> NestJS REST API and WebSocket server for Stellar DeFi Lens.

<p>
  <a href="https://www.drips.network/wave/stellar"><img src="https://img.shields.io/badge/Stellar-Wave%20Program-blueviolet?style=flat-square" /></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-MIT-22c55e?style=flat-square" /></a>
  <img src="https://img.shields.io/badge/NestJS-10.x-E0234E?style=flat-square" />
  <img src="https://img.shields.io/badge/TypeScript-5.x-3178C6?style=flat-square" />
</p>

Part of [Stellar DeFi Lens](https://github.com/stellar-defi-lens) — the open-source DeFi analytics layer for Stellar.

---

## Overview

`sdl-api` is the data-serving layer of Stellar DeFi Lens. It reads the metrics indexed by `sdl-indexer` from PostgreSQL and exposes them via:

- A **REST API** for protocol TVL, DEX volume, pool analytics, and asset data
- A **WebSocket gateway** for real-time streaming of live on-chain events

Both `sdl-web` (the dashboard) and `sdl-sdk` (the npm package) consume this API.

---

## API reference

### Protocols

```
GET /protocols
```
Returns all tracked protocols with summary stats.

```json
[
  {
    "id": "blend",
    "name": "Blend Protocol",
    "type": "lending",
    "tvl": 100620000,
    "volume24h": 4830000,
    "change24h": 2.4,
    "change7d": -1.1
  },
  {
    "id": "aquarius",
    "name": "Aquarius AMM",
    "type": "dex",
    "tvl": 37300000,
    "volume24h": 9120000,
    "change24h": 5.7,
    "change7d": 12.3
  }
]
```

---

```
GET /protocols/:id
```
Full protocol details.

Params: `id` — `blend` | `aquarius` | `soroswap` | `templar`

---

```
GET /protocols/:id/tvl
```
TVL timeseries — Recharts-compatible `[{ timestamp, value }]`.

| Query param | Type | Default | Description |
|-------------|------|---------|-------------|
| `from` | ISO 8601 | 30 days ago | Start of range |
| `to` | ISO 8601 | now | End of range |
| `interval` | `1h` \| `1d` \| `1w` | `1d` | Aggregation interval |

```json
[
  { "timestamp": 1746057600000, "value": 98400000 },
  { "timestamp": 1746144000000, "value": 99100000 },
  { "timestamp": 1746230400000, "value": 100620000 }
]
```

---

```
GET /protocols/:id/volume
```
Volume timeseries. Same query params as `/tvl`.

---

```
GET /protocols/blend/pools
```
All active Blend lending pools with per-asset rates.

| Query param | Type | Description |
|-------------|------|-------------|
| `sortBy` | `tvl` \| `borrowRate` \| `utilization` | Sort field (default: `tvl`) |
| `order` | `asc` \| `desc` | Sort direction (default: `desc`) |

```json
[
  {
    "poolAddress": "CBLEND1...",
    "assets": [
      {
        "symbol": "USDC",
        "contractId": "CCW67TT...",
        "totalSupply": 45200000,
        "totalBorrow": 38700000,
        "supplyApy": 4.21,
        "borrowApy": 6.83,
        "utilization": 0.856
      }
    ]
  }
]
```

---

```
GET /protocols/aquarius/pairs
GET /protocols/soroswap/pairs
```
DEX liquidity pairs sorted by TVL.

| Query param | Type | Description |
|-------------|------|-------------|
| `sortBy` | `tvl` \| `volume24h` \| `apy` | Sort field |
| `order` | `asc` \| `desc` | Sort direction |
| `limit` | number | Max results (default: 50, max: 200) |
| `cursor` | string | Cursor for next page (cursor-based pagination) |

```json
{
  "pairs": [
    {
      "pairId": "CAQUA...",
      "token0": { "symbol": "XLM", "contractId": "native" },
      "token1": { "symbol": "USDC", "contractId": "CCW67TT..." },
      "tvl": 8430000,
      "volume24h": 2140000,
      "fees24h": 6420,
      "apy": 27.8
    }
  ],
  "nextCursor": "eyJpZCI6..."
}
```

---

### Assets

```
GET /assets/:contractId
```
Asset detail — supply, price, holders, 24h volume.

```json
{
  "contractId": "BENJI_CONTRACT...",
  "name": "Franklin OnChain US Govt Money Fund",
  "symbol": "BENJI",
  "decimals": 6,
  "totalSupply": 94300000,
  "holderCount": 2841,
  "priceUsd": 1.0,
  "marketCap": 94300000,
  "volume24h": 1230000,
  "yieldApy": 5.02
}
```

---

```
GET /assets/:contractId/holders?limit=20
```
Top holder distribution.

```json
[
  { "address": "GABC...", "balance": 12400000, "percentage": 13.15 },
  { "address": "GDEF...", "balance": 8700000, "percentage": 9.23 }
]
```

---

```
GET /assets/:contractId/transfers
```
Paginated transfer history.

| Query param | Type | Description |
|-------------|------|-------------|
| `from` | ISO 8601 | Start of range |
| `to` | ISO 8601 | End of range |
| `cursor` | string | Pagination cursor |
| `limit` | number | Max results (default: 50) |

---

### Activity

```
GET /activity
```
Recent significant on-chain events (swaps > $10K, liquidations, large deposits).

| Query param | Type | Description |
|-------------|------|-------------|
| `protocol` | string | Filter by protocol ID |
| `type` | string | Filter by event type (`swap`, `supply`, `borrow`, `liquidation`) |
| `limit` | number | Max results (default: 50) |

---

```
WebSocket /live
```
Real-time stream of decoded Soroban events.

**Subscribe to specific protocols:**
```json
{ "event": "subscribe", "data": { "protocols": ["blend", "aquarius"] } }
```

**Incoming event shape:**
```json
{
  "protocol": "blend",
  "type": "supply",
  "asset": "USDC",
  "assetContractId": "CCW67TT...",
  "amountUsd": 96000,
  "user": "GABC...",
  "ledger": 54823901,
  "timestamp": 1746230400000
}
```

---

### System

```
GET /health
```
```json
{
  "status": "ok",
  "latestIndexedLedger": 54823901,
  "networkLatestLedger": 54823905,
  "indexerLag": 4
}
```

```
GET /metadata
```
List of all tracked protocols and their contract IDs.

---

## Project structure

```
sdl-api/
├── src/
│   ├── protocols/
│   │   ├── protocols.controller.ts   # GET /protocols routes
│   │   ├── protocols.service.ts      # DB queries + data assembly
│   │   ├── protocols.module.ts
│   │   └── dto/
│   │       ├── tvl-query.dto.ts      # Validated query params
│   │       └── protocol-response.dto.ts
│   ├── assets/
│   │   ├── assets.controller.ts
│   │   ├── assets.service.ts
│   │   └── dto/
│   ├── activity/
│   │   ├── activity.controller.ts
│   │   ├── activity.service.ts
│   │   └── activity.gateway.ts       # WebSocket gateway
│   ├── health/
│   │   └── health.controller.ts
│   ├── db/
│   │   └── db.module.ts              # Shared pg pool (NestJS module)
│   └── main.ts                       # Bootstrap, CORS, global pipes
├── test/
│   └── *.e2e-spec.ts
├── .env.example
├── package.json
├── tsconfig.json
└── CONTRIBUTING.md
```

---

## Getting started

### Prerequisites

- Node.js 20+
- pnpm
- `sdl-indexer` running with a populated PostgreSQL database

### Setup

```bash
git clone https://github.com/stellar-defi-lens/sdl-api.git
cd sdl-api
pnpm install
cp .env.example .env
pnpm dev    # starts on http://localhost:3001
```

Confirm it's working:

```bash
curl http://localhost:3001/health
curl http://localhost:3001/protocols
```

### Environment variables

```env
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/sdl
PORT=3001
CORS_ORIGIN=http://localhost:3000
```

### Available scripts

```bash
pnpm dev          # Start with hot reload
pnpm build        # Compile TypeScript
pnpm start        # Run compiled output
pnpm test         # Unit tests (Jest)
pnpm test:e2e     # End-to-end tests
pnpm test:cov     # Coverage report
pnpm lint         # ESLint
pnpm typecheck    # tsc --noEmit
```

---

## Key dependencies

| Package | Purpose |
|---------|---------|
| `@nestjs/core`, `@nestjs/common` | NestJS framework |
| `@nestjs/websockets`, `@nestjs/platform-socket.io` | WebSocket gateway |
| `class-validator`, `class-transformer` | DTO validation + transformation |
| `pg` | PostgreSQL client |

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). Issues labeled `wave:trivial`, `wave:medium`, `wave:high` are part of the Stellar Wave Program on Drips.

---

## License

MIT © Stellar DeFi Lens contributors
