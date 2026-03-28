# Events

An **event** is a financial business occurrence submitted to the Event Service for processing. The Event Service validates the event, evaluates templates, and produces double-entry transfers in TigerBeetle.

## Event Structure

| Field | Type | Description |
|-------|------|-------------|
| `event_id` | UUIDv7 | Unique identifier and idempotency key. Time-ordered for natural sorting. |
| `site_id` | integer | Tenant boundary (positive integer) |
| `template_ids` | integer[] | Templates to process atomically (all succeed or all fail) |
| `amount` | integer | Transaction amount in minor currency units (cents, satoshis) |
| `accounts` | array | Purpose-to-account mappings (e.g., `"from_account"` -> account UUID) |
| `occurred_at` | integer | When the business event happened (Unix milliseconds) |
| `event_source` | string | System or service that originated the event |
| `extra_metadata` | object | Additional data validated against the template's metadata schema |
| `original_event_id` | UUIDv7 | Links to the first event in a correction/reversal chain (optional) |
| `previous_event_id` | UUIDv7 | Links to the immediate predecessor event (optional) |
| `directives` | string[] | Processing directives (e.g., `skip_rollup_processing`) |

## Event Lifecycle

1. **Submit**: Client sends the event to `POST /api/v1/events`.
2. **Validate**: The Event Service resolves accounts via Ledger Service, validates against each template (metadata schema, account code masks, CEL validations).
3. **Process**: Template variables are evaluated, transfer legs are computed.
4. **Transfer creation**: All transfers are submitted atomically to TigerBeetle. Either every transfer succeeds or none do.
5. **Publish**: The processed event envelope is published to the Event Store for audit.

## Idempotency

The `event_id` is the idempotency key. If processing fails, **retry with the same `event_id`**. Never generate a new `event_id` for a retry. TigerBeetle deduplicates transfers by ID, so resubmitting a previously successful event is safe.

## Corrections and Reversals

The `original_event_id` and `previous_event_id` fields create an event chain for corrections and reversals. The original event ID links back to the first event in the series, while the previous event ID links to the immediate predecessor. This chain enables the Balance Service to analyze the full history of adjustments to a transaction.

## Submitting an Event

```bash
curl -X POST https://ledger.dev.ledgerrocket.com/v2/event-service/api/v1/events \
  -H "x-api-key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "$schema": "https://ledger-rocket.github.io/schemas/domain/event.schema.json",
    "event_id": "0192b4f0-7abc-7def-8000-000000000001",
    "site_id": 1,
    "template_ids": [532],
    "amount": 100000,
    "occurred_at": 1760000000000,
    "event_source": "payment_gateway",
    "accounts": [
      {"purpose": "from_account", "account_id": "0192b4f0-7abc-7def-8000-000000000010"},
      {"purpose": "to_account", "account_id": "0192b4f0-7abc-7def-8000-000000000020"}
    ]
  }'
```

The response includes the event echo, generated transfers, and the templates that were applied.
