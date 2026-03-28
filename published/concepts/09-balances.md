# Balances

The **Balance Service** provides real-time and historical balance queries against TigerBeetle, the underlying ledger store.

## Credits Posted / Debits Posted Model

Balances are expressed as two running totals:

- `credits_posted` -- total value credited to the account (in minor units)
- `debits_posted` -- total value debited from the account (in minor units)

The **net balance** is `credits_posted - debits_posted`. For asset accounts (normal debit balance), a positive net balance means debits exceed credits. The interpretation depends on the account type.

Both values are returned as strings to avoid integer overflow in JSON for large balances.

## Real-Time Balances from TigerBeetle

Current balances are queried directly from TigerBeetle, which maintains them as transfers are committed. There is no batch reconciliation -- balances reflect all committed transfers immediately.

```bash
curl "https://ledger.dev.ledgerrocket.com/v2/balance-service/api/v1/balances/0192b4f0-7abc-7def-8000-000000000001?site_id=1" \
  -H "x-api-key: $API_KEY"
```

Response:

```json
{
  "account_id": "0192b4f0-7abc-7def-8000-000000000001",
  "credits_posted": "1500000",
  "debits_posted": "500000",
  "timestamp": 1704067200000000000
}
```

The `timestamp` is in nanoseconds since the Unix epoch, reflecting the precise point-in-time of the balance.

## Historical Timeseries

The Balance Service provides balance history as a series of snapshots over time:

```bash
curl "https://ledger.dev.ledgerrocket.com/v2/balance-service/api/v1/balances/0192b4f0-7abc-7def-8000-000000000001/timeseries?site_id=1" \
  -H "x-api-key: $API_KEY"
```

Each point in the timeseries includes `credits_posted`, `debits_posted`, and a `timestamp`, showing how the balance evolved.

## Event Chain Analysis

The Balance Service can break down an account's balance by **event chain** -- grouping transfers by `original_event_id`. This reveals how corrections and reversals affected a specific transaction's impact on the account.

```bash
curl "https://ledger.dev.ledgerrocket.com/v2/balance-service/api/v1/balances/0192b4f0-7abc-7def-8000-000000000001/chains?site_id=1" \
  -H "x-api-key: $API_KEY"
```

## Account Code Aggregation

Balances can be queried by account code rather than individual account. This returns the combined balance of all accounts with a given code, grouped by ledger:

```bash
curl "https://ledger.dev.ledgerrocket.com/v2/balance-service/api/v1/balances/account-codes/1001?site_id=1" \
  -H "x-api-key: $API_KEY"
```

Account code timeseries are also available, providing aggregated balance history across all accounts sharing a code.

## Balance Caching

The Balance Service caches **immutable historical data** -- balance snapshots at past timestamps that can never change because TigerBeetle transfers are append-only. Current (latest) balances are always fetched live from TigerBeetle. This caching strategy provides fast historical queries without risking stale data for current balances.
