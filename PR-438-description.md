# PR: Fix - Add dependency status to /health endpoint

## Category
Operations - Health Checks

## Priority
Medium

## Description
The health endpoint (`GET /health`) previously only reported a static status (`"ok"`) and network name, making partial outages invisible to orchestrators and operators. This PR enriches the endpoint with real-time dependency health information and separates liveness from readiness semantics.

## Changes Made

### 1. Added dependency health fields to `/health`
- **Database** (`dependencies.db`) — reflects whether the Prisma connection was established
- **Cache** (`dependencies.cache`) — reflects whether Redis (or in-memory) cache is operational
- **Indexer** (`dependencies.indexer`) — reflects whether the indexer service is running
- **Cold Storage** (`dependencies.coldStorage`) — reflects whether archive storage is initialized
- **Worker** (`dependencies.worker`) — new dependency tracking the bridge worker status

The response also includes:
- `indexer.healthy` + optional `indexer.failureReason` — detailed indexer health
- `worker.running` — whether the bridge worker poll loop is active
- `uptime` — process uptime in seconds

### 2. Separated liveness from readiness semantics

| Endpoint | Purpose | Returns 503? |
|----------|---------|-------------|
| `GET /health` | Liveness probe — is the process alive? | Only on shutdown |
| `GET /readyz` | Readiness probe — are all deps ready? | When any dep is not ready |
| `GET /ready` | Indexer-specific health | When indexer has failed |

**Key behavioral change**: `/health` now returns HTTP 200 with dependency status as informational fields, even if some dependencies are down. This allows orchestrators (Kubernetes, Docker, etc.) to distinguish between "process is alive but degraded" vs "process is dead". Use `/readyz` for the readiness gate that controls traffic routing.

### 3. Worker dependency tracking
- Added `worker` to the `DependencyName` type in `src/readiness.ts`
- Worker readiness is set to `true` when `startBridgeWorker()` succeeds in `main()`
- Worker status (`isBridgeWorkerRunning()`) is reported in the `/health` response

### 4. OpenAPI / Swagger documentation
Added full `@swagger` JSDoc annotation for `GET /health` documenting:
- Request parameters (none)
- Successful response (200) shape with all fields
- Shutdown response (503) shape
- Schema definitions for dependencies, indexer, and worker objects

### 5. Updated tests
All test cases in `tests/readyz.test.ts` updated to validate the new `worker` dependency:
- Initial state assertion (worker is `false`)
- Partial readiness assertion (worker is `false`)
- Full readiness assertions (worker is `true`)
- Shutdown override assertion
- Readiness transition tests

## Affected Files

| File | Change |
|------|--------|
| `src/index.ts` | Updated `/health` handler with dependency fields, liveness semantics, Swagger docs; added worker readiness tracking |
| `src/readiness.ts` | Added `worker` to `DependencyName` type and state map |
| `tests/readyz.test.ts` | Updated all test assertions to include `worker` dependency |

## Response Contract

### `GET /health` — 200 OK

```json
{
  "status": "ok",
  "network": "testnet",
  "uptime": 1234.56,
  "dependencies": {
    "db": true,
    "cache": true,
    "indexer": true,
    "coldStorage": true,
    "worker": true
  },
  "indexer": {
    "healthy": true
  },
  "worker": {
    "running": true
  }
}
```

### `GET /health` — 503 Service Unavailable (shutting down)

```json
{
  "status": "shutting_down"
}
```

### `GET /readyz` — 200 OK (all deps ready)

```json
{
  "status": "ready",
  "dependencies": {
    "db": true,
    "cache": true,
    "indexer": true,
    "coldStorage": true,
    "worker": true
  }
}
```

### `GET /readyz` — 503 Service Unavailable (partial readiness)

```json
{
  "status": "not_ready",
  "dependencies": {
    "db": true,
    "cache": false,
    "indexer": true,
    "coldStorage": true,
    "worker": true
  }
}
```

## Testing
- All existing `/readyz` tests pass with updated worker assertions
- New response contract verified via Swagger documentation
- Liveness probe now returns 200 with dependency info (not 503) when deps are down

closes #438
