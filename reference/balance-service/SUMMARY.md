# Balance Service v1.0 (Go)

Real-time and historical balance queries backed by TigerBeetle.

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/livez` | Liveness probe |
| GET | `/readyz` | Readiness probe |
| GET | `/api/v1/accounts/{account_id}/balance` | Get account balance |
| GET | `/api/v1/accounts/{account_id}/balance-history` | Account balance history |
| GET | `/api/v1/accounts/{account_id}/chain-balances` | Account chain balances |
| GET | `/api/v1/account-codes/{code}/balance` | Balance by account code |
| GET | `/api/v1/account-codes/{code}/balance-history` | History by account code |
| POST | `/api/v1/balance-history:query` | Bulk balance history query |
| POST | `/api/v1/chain-summary:query` | Batch chain summary |
| GET | `/api/v1/events/{event_id}/balance` | Event chain balance |
| GET | `/api/v1/events/{event_id}/chain` | Event chain detail |
| GET | `/api/v1/cache/stats` | Cache statistics |
| POST | `/api/v1/uuids` | Generate UUIDs |

13 endpoints total.

## Key Data Models

### BalanceResponse

Single account balance snapshot.

- **account_id** -- UUID of the account
- **credits_posted** -- total posted credits (string-encoded integer, minor units)
- **debits_posted** -- total posted debits (string-encoded integer, minor units)
- **timestamp** -- balance timestamp in nanoseconds

### BalancePoint

Timeseries snapshot for balance history queries. Represents a single point in time with the balance state at that moment.

### AccountChainBalancesResponse

Per-chain breakdown of balances for a single account. Backed by Athena for historical chain queries.

### AccountCodeBalanceResponse

Aggregate balance across all accounts sharing a given account code, spanning multiple ledgers.

### CacheStatsResponse

Operational metrics for the balance cache layer.

### Amount Encoding

All monetary amounts are string-encoded integers in minor units (e.g., `"1500"` represents 15.00 in a currency with scale 2). This avoids floating-point precision issues.
