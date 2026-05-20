# Contributing to sdl-api

Welcome! `sdl-api` is part of the **[Stellar Wave Program](https://www.drips.network/wave/stellar)** on Drips. Contributors who resolve labeled issues during a Wave earn **XLM rewards** funded by the Stellar Development Foundation.

This guide is written against the real project structure. Read it before opening a PR.

---

## Table of contents

- [Project structure](#project-structure)
- [How the API is organized](#how-the-api-is-organized)
- [Local setup](#local-setup)
- [Module anatomy](#module-anatomy)
- [How to add a new endpoint](#how-to-add-a-new-endpoint)
- [How to add a query parameter to an existing endpoint](#how-to-add-a-query-parameter-to-an-existing-endpoint)
- [Working with the WebSocket module](#working-with-the-websocket-module)
- [Testing](#testing)
- [Pull request process](#pull-request-process)
- [Code style](#code-style)

---

## Project structure

```
sdl-api/
├── dist/                        # Compiled output (generated — do not edit)
├── node_modules/
├── src/
│   ├── common/                  # Shared pipes, filters, guards, decorators
│   ├── config/                  # Environment config with validation
│   ├── database/                # Database module (pg pool wrapped as NestJS module)
│   ├── modules/
│   │   ├── activity/            # GET /activity · event log queries
│   │   ├── assets/              # GET /assets/:contractId · RWA asset data
│   │   ├── health/              # GET /health · indexer lag + status
│   │   ├── protocols/           # GET /protocols + sub-routes (TVL, pools, pairs)
│   │   └── websocket/           # WebSocket gateway for /live event streaming
│   ├── utils/                   # Shared utility functions (pagination, formatting)
│   ├── app.controller.ts        # Root controller (handles GET /)
│   ├── app.module.ts            # Root NestJS module — imports all feature modules
│   ├── app.service.ts           # Root service
│   └── main.ts                  # Bootstrap — port, CORS, global pipes, validation
├── test/
│   ├── app.e2e-spec.ts          # End-to-end tests
│   └── jest-e2e.json            # Jest config for e2e tests
├── .env.example
├── package.json
└── CONTRIBUTING.md
```

---

## How the API is organized

`sdl-api` follows **NestJS module architecture**. Each feature (protocols, assets, activity, etc.) is a self-contained module under `src/modules/`. Each module has the same internal structure:

```
modules/protocols/
├── protocols.controller.ts    # Route handlers — defines URL paths and HTTP methods
├── protocols.service.ts       # Business logic — constructs and runs DB queries
├── protocols.module.ts        # NestJS module declaration — wires controller + service
└── dto/
    ├── tvl-query.dto.ts       # Input validation for query params (class-validator)
    └── protocol-response.dto.ts  # Output type documentation (optional)
```

The data flow for every request is:

```
HTTP Request
    │
    ▼
Controller        ← validates path params + query params via DTO
    │
    ▼
Service           ← runs parameterized SQL query against PostgreSQL
    │
    ▼
JSON Response     ← serialized and returned
```

The WebSocket module (`modules/websocket/`) follows the same structure but uses NestJS `@WebSocketGateway` instead of a controller, and subscribes to the `sdl-indexer` event stream to push decoded events to connected clients.

---

## Local setup

### Requirements

- Node.js 20+
- pnpm
- A running PostgreSQL instance with the `sdl-indexer` schema migrated and data flowing

### Steps

```bash
# 1. Fork and clone
git clone https://github.com/<your-username>/sdl-api.git
cd sdl-api

# 2. Install dependencies
pnpm install

# 3. Configure environment
cp .env.example .env
# Set DATABASE_URL to your local Postgres instance
# PORT defaults to 3001

# 4. Start the API
pnpm dev    # Starts on http://localhost:3001
```

Confirm it is working:

```bash
curl http://localhost:3001/health
# { "status": "ok", "indexerLag": 4, "latestIndexedLedger": 54823905 }

curl http://localhost:3001/protocols
# [ { "id": "blend", "tvl": 100620000, ... }, ... ]
```

> **Tip:** You can develop the API without running `sdl-indexer` locally by pointing `DATABASE_URL` at the staging database (ask a maintainer for read-only credentials). This lets you build and test endpoints against real data without needing a full local stack.

### Available scripts

```bash
pnpm dev           # Start with hot reload (nest start --watch)
pnpm build         # Compile TypeScript to dist/
pnpm start         # Run compiled output
pnpm start:prod    # Production mode
pnpm test          # Unit tests (Jest)
pnpm test:watch    # Watch mode
pnpm test:cov      # Coverage report
pnpm test:e2e      # End-to-end tests (requires running DB)
pnpm lint          # ESLint
pnpm typecheck     # tsc --noEmit
```

### Environment variables

```env
# Required
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/sdl
PORT=3001

# Optional
CORS_ORIGIN=http://localhost:3000     # Comma-separated list of allowed origins
LOG_LEVEL=info                        # debug | info | warn | error
```

---

## Module anatomy

Every feature module has the same internal structure. Here is a complete worked example using the `protocols` module.

### Controller — routing and input validation

```typescript
// src/modules/protocols/protocols.controller.ts
import { Controller, Get, Param, Query } from '@nestjs/common';
import { ProtocolsService } from './protocols.service';
import { TVLQueryDto } from './dto/tvl-query.dto';

@Controller('protocols')
export class ProtocolsController {
  constructor(private readonly protocolsService: ProtocolsService) {}

  @Get()
  getAll() {
    return this.protocolsService.findAll();
  }

  @Get(':id/tvl')
  getTVL(@Param('id') id: string, @Query() query: TVLQueryDto) {
    return this.protocolsService.getTVLHistory(id, query);
  }
}
```

### DTO — input validation

```typescript
// src/modules/protocols/dto/tvl-query.dto.ts
import { IsOptional, IsIn, IsISO8601 } from 'class-validator';

export class TVLQueryDto {
  @IsOptional()
  @IsISO8601()
  from?: string;

  @IsOptional()
  @IsISO8601()
  to?: string;

  @IsOptional()
  @IsIn(['1h', '1d', '1w'])
  interval?: '1h' | '1d' | '1w' = '1d';
}
```

### Service — database queries

```typescript
// src/modules/protocols/protocols.service.ts
import { Injectable } from '@nestjs/common';
import { DatabaseService } from '../../database/database.service';
import { TVLQueryDto } from './dto/tvl-query.dto';

@Injectable()
export class ProtocolsService {
  constructor(private readonly db: DatabaseService) {}

  async getTVLHistory(protocolId: string, query: TVLQueryDto) {
    const { from, to, interval } = query;
    const result = await this.db.pool.query(`
      SELECT
        DATE_TRUNC($1, snapshot_hour) AS timestamp,
        SUM(value_usd) AS value
      FROM tvl_snapshots
      WHERE protocol = $2
        AND snapshot_hour >= $3
        AND snapshot_hour <= $4
      GROUP BY 1
      ORDER BY 1 ASC
    `, [interval, protocolId, from ?? '30 days ago', to ?? 'now']);

    return result.rows.map(row => ({
      timestamp: row.timestamp.getTime(),
      value: parseFloat(row.value),
    }));
  }
}
```

### Module declaration

```typescript
// src/modules/protocols/protocols.module.ts
import { Module } from '@nestjs/common';
import { ProtocolsController } from './protocols.controller';
import { ProtocolsService } from './protocols.service';
import { DatabaseModule } from '../../database/database.module';

@Module({
  imports: [DatabaseModule],
  controllers: [ProtocolsController],
  providers: [ProtocolsService],
})
export class ProtocolsModule {}
```

Then register the module in `src/app.module.ts`:

```typescript
@Module({
  imports: [DatabaseModule, ProtocolsModule, AssetsModule, ActivityModule, ...],
})
export class AppModule {}
```

---

## How to add a new endpoint

1. **Identify the module** — does the endpoint belong in an existing module, or does it need a new one?
2. **Add the route** in the controller with the correct HTTP method and path decorator
3. **Create a DTO** for any query params or body — use `class-validator` decorators
4. **Implement the service method** — parameterized SQL query, no raw string interpolation
5. **Register the module** in `app.module.ts` if new
6. **Write unit tests** for the service method
7. **Write or update the e2e test** in `test/app.e2e-spec.ts` covering the new route
8. **Document the endpoint** in the PR description (method, path, params, example response)

---

## How to add a query parameter to an existing endpoint

1. Add the field to the existing DTO in `dto/` with `@IsOptional()` and the appropriate validator
2. Update the service method to use the new parameter in the SQL query
3. Add a unit test case covering the new parameter
4. Add an e2e test case hitting the route with the new parameter

Example — adding `?asset=USDC` filtering to Blend pool results:

```typescript
// In dto/blend-pools-query.dto.ts
@IsOptional()
@IsString()
@Length(1, 12)
asset?: string;

// In protocols.service.ts — add conditional WHERE clause
const assetFilter = query.asset ? 'AND a.symbol = $3' : '';
const params = query.asset
  ? [poolId, 'blend', query.asset]
  : [poolId, 'blend'];
```

---

## Working with the WebSocket module

The `modules/websocket/` gateway handles the `/live` WebSocket endpoint. It:

1. Receives decoded events from `sdl-indexer` (via a shared queue or direct DB polling — see the module for the current approach)
2. Filters events by subscribed protocols per client
3. Emits typed events to subscribed clients

When adding a new event type to the live feed:

1. Add the event shape to the types file in `modules/websocket/`
2. Add the event to the gateway's emit logic
3. Update the `subscribe` message handler if new protocol filter options are needed
4. Add an integration test using a WebSocket client

The WebSocket endpoint uses Socket.io. Clients connect and subscribe:

```typescript
// Client-side subscription
socket.emit('subscribe', { protocols: ['blend', 'aquarius'] });
socket.on('event', (data) => console.log(data));
```

---

## Testing

```bash
pnpm test          # Unit tests (Jest)
pnpm test:watch    # Watch mode
pnpm test:cov      # Coverage report
pnpm test:e2e      # End-to-end tests (requires a running database)
```

### Unit tests — mock the database

Unit tests for service methods mock `DatabaseService` to return fixture data. Never make real database calls in unit tests.

```typescript
// protocols.service.spec.ts
import { Test } from '@nestjs/testing';
import { ProtocolsService } from './protocols.service';
import { DatabaseService } from '../../database/database.service';

describe('ProtocolsService', () => {
  let service: ProtocolsService;
  const mockPool = { query: jest.fn() };

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        ProtocolsService,
        { provide: DatabaseService, useValue: { pool: mockPool } },
      ],
    }).compile();
    service = module.get(ProtocolsService);
  });

  it('getTVLHistory returns Recharts-compatible timeseries', async () => {
    mockPool.query.mockResolvedValue({
      rows: [
        { timestamp: new Date('2026-05-01'), value: '98400000' },
      ],
    });
    const result = await service.getTVLHistory('blend', { interval: '1d' });
    expect(result[0]).toMatchObject({ timestamp: expect.any(Number), value: 98400000 });
  });
});
```

### E2E tests — test against a real database

E2E tests in `test/app.e2e-spec.ts` use `supertest` and a real Postgres instance. Run them after `pnpm db:migrate` with a seeded database.

```typescript
// test/app.e2e-spec.ts
it('GET /protocols returns an array', () => {
  return request(app.getHttpServer())
    .get('/protocols')
    .expect(200)
    .expect(res => {
      expect(Array.isArray(res.body)).toBe(true);
      expect(res.body[0]).toHaveProperty('id');
      expect(res.body[0]).toHaveProperty('tvl');
    });
});
```

---

## Pull request process

1. **Branch from `main`**: `git checkout -b feat/aquarius-pairs-endpoint`
2. **One PR per issue** — keep scope tight
3. **Checklist before opening:**
   - `pnpm test` passes
   - `pnpm typecheck` passes
   - `pnpm lint` passes
   - New endpoints have DTO validation (no unvalidated query params or body)
   - All SQL uses parameterized queries (no string interpolation with user input)
   - Unit tests cover the happy path and at least one error/edge case
4. **PR description must include:**
   - Link to the issue
   - HTTP method, full path, and all query params
   - Example JSON response

---

## Code style

- **NestJS conventions** — modules, controllers, services, DTOs as the framework intends
- **DTOs for all inputs** — every query param and request body goes through a DTO with `class-validator`
- **Parameterized SQL only** — no string interpolation with user-supplied values
- **Services for all DB access** — controllers never query the database directly
- **No `any`** — TypeScript strict mode throughout
- **Consistent error responses** — use NestJS built-in exceptions (`NotFoundException`, `BadRequestException`, etc.)
