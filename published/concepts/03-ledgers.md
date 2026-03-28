# Ledgers

## Ledger = Entity + Currency

A **ledger** is scoped to exactly one entity and one currency. It is the fundamental grouping for accounts and the boundary within which transfers operate.

| Field | Description |
|-------|-------------|
| `ledger_id` | Unique integer identifier (u32) |
| `entity_id` | The entity that owns this ledger |
| `currency_code` | ISO 4217 currency (e.g., USD, EUR, GBP) |
| `ledger_type` | Operational purpose: OPERATIONAL, WORKQUEUE, or TREASURY_SWEEP |
| `include_in_consolidation` | Whether balances appear in consolidated financial statements |

## Why All Transfers Must Be Within a Single Ledger

Every transfer debits one account and credits another. Both accounts must belong to the **same ledger**. This constraint ensures:

1. **Currency consistency**: Both sides of a transfer are in the same currency. There is no implicit currency conversion.
2. **Entity ownership clarity**: The transfer stays within a single entity's books.
3. **Atomicity**: TigerBeetle enforces same-ledger transfers at the storage layer.

To move value between currencies or entities, you create separate transfers on each ledger -- for example, a payment leg on a USD ledger and a receipt leg on a EUR ledger, linked by the same `event_id`.

## Creating a Ledger

```bash
curl -X POST https://ledger.dev.ledgerrocket.com/v2/ledger/api/v1/ledgers \
  -H "x-api-key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "records": [
      {
        "entity_id": "123e4567-e89b-12d3-a456-426614174000",
        "currency_code": "USD",
        "site_id": 1,
        "name": "Acme Corp USD",
        "description": "Primary USD operating ledger for Acme Corp",
        "ledger_type": "OPERATIONAL",
        "include_in_consolidation": true
      }
    ]
  }'
```

## Ledger-to-Entity Relationships

- One entity can own **multiple ledgers** (one per currency it operates in).
- One ledger belongs to **exactly one entity**.
- All accounts on a ledger share the same currency.
- The ledger response includes the full entity and currency details, eagerly loaded.

### Typical setup for a multi-currency company

```text
Acme Corp (entity)
  +-- Acme Corp USD (ledger, currency=USD)
  |     +-- Cash Account (account)
  |     +-- Revenue Account (account)
  +-- Acme Corp EUR (ledger, currency=EUR)
        +-- Cash Account (account)
        +-- Revenue Account (account)
```

Each ledger operates independently. Transfers within the USD ledger never interact with accounts on the EUR ledger.
