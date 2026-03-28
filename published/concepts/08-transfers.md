# Transfers

A **transfer** is the fundamental unit of record in LedgerRocket -- a double-entry movement of value that debits one account and credits another.

## Double-Entry Model

Every transfer has exactly two sides:

- A **debit account** -- the account being debited
- A **credit account** -- the account being credited
- An **amount** -- the value moving, in integer minor units

This guarantees that the books always balance: total debits across all accounts always equal total credits.

## Same-Ledger Constraint

Both the debit and credit account must belong to the **same ledger**. This is enforced at the TigerBeetle storage layer. Since a ledger is scoped to one entity and one currency, this constraint ensures:

- No implicit currency conversion happens within a transfer
- Each transfer operates within a single entity's books

Cross-currency or cross-entity flows require multiple transfers on different ledgers, linked by a common `event_id`.

## Integer Minor Units

All transfer amounts are **integer minor units** -- cents for USD (scale=2), pence for GBP, satoshis for BTC (scale=8). There are no floating-point values anywhere in the transfer pipeline.

For example, $1,500.00 USD is represented as `150000` (integer cents).

## Transfer Grouping by Event

Transfers are grouped by `event_id`. A single event processed through one or more templates may generate multiple transfers (one per template leg). All transfers from an event are submitted atomically to TigerBeetle -- either all succeed or none do.

## Enriched Transfers

When querying transfers, the API returns **enriched transfers** that include resolved reference data alongside the raw transfer:

| Field | Description |
|-------|-------------|
| `transfer_id` | UUID assigned by TigerBeetle |
| `event_id` | The originating event |
| `amount` | Transfer amount in minor units |
| `debit_account_id` | UUID of the debited account |
| `debit_account_name` | Human-readable name of the debited account |
| `debit_account_code` | Account code of the debited account |
| `debit_account_type` | ASSET, LIABILITY, etc. |
| `credit_account_id` | UUID of the credited account |
| `credit_account_name` | Human-readable name of the credited account |
| `credit_account_code` | Account code of the credited account |
| `credit_account_type` | ASSET, LIABILITY, etc. |
| `currency_code` | ISO currency from the ledger |
| `currency_scale` | Minor unit scale (e.g., 100 for USD) |
| `template_name` | Template that generated this transfer |
| `template_product` | Product category (e.g., PAYMENTS) |
| `leg_number` | Which template leg produced this transfer |
| `is_rollup` | Whether this is a rollup aggregation transfer |
| `occurred_at` | When the business event occurred (Unix milliseconds) |

## Querying Transfers

```bash
curl "https://ledger.dev.ledgerrocket.com/v2/event-service/api/v1/transfers?site_id=1&event_id=0192b4f0-7abc-7def-8000-000000000001" \
  -H "x-api-key: $API_KEY"
```

The response includes the full enriched transfer list, so consumers do not need to join against the Ledger Service to display meaningful transfer details.
