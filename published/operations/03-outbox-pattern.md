# Event Outbox Pattern

How events flow through the outbox state machine, and how to monitor and troubleshoot it.

## Outbox Lifecycle

Every event submitted to the Event Service passes through the outbox, a persistent state machine that guarantees exactly-once processing:

```text
prepared --> committed --> published
                |
                +--> failed
```

### States

| State | Meaning |
|-------|---------|
| `prepared` | Event accepted and validated. Transfers computed but not yet submitted to TigerBeetle. |
| `committed` | Transfers atomically committed to TigerBeetle. Balances are updated. |
| `published` | Event envelope published to the Event Store for audit and downstream consumers. Terminal success state. |
| `failed` | Processing failed at some stage. Requires investigation. |

### Normal Flow

1. Client POSTs to `/api/v1/events`
2. Event Service validates the event, resolves accounts, evaluates templates, computes transfers
3. Event enters `prepared` state
4. Transfers are submitted atomically to TigerBeetle
5. On success, event transitions to `committed`
6. Event envelope is published to the Event Store
7. On success, event transitions to `published`

The entire flow typically completes in milliseconds. The client receives a response after the `committed` stage -- they do not wait for `published`.

## Retry Behavior

If publishing to the Event Store fails, the outbox retries with exponential backoff. The event remains in `committed` state during retries. Transfers are already committed to TigerBeetle, so balances are correct regardless of publish status.

If the Event Service restarts, it scans for events stuck in `prepared` or `committed` states and resumes processing. This is the at-least-once delivery guarantee for the Event Store.

Events in `failed` state are not automatically retried. They require manual investigation and resubmission.

## Admin Endpoints

### List Outbox Records

```bash
curl -s "https://ledger.dev.ledgerrocket.com/v2/event-service/api/v1/admin/outbox?site_id=5" \
  -H "x-api-key: lr-test-20250701" | jq .
```

Returns recent outbox records with their current state. Use query parameters to filter:

```bash
# Only failed events
curl -s "https://ledger.dev.ledgerrocket.com/v2/event-service/api/v1/admin/outbox?site_id=5&state=failed" \
  -H "x-api-key: lr-test-20250701" | jq .
```

### Outbox Statistics

```bash
curl -s "https://ledger.dev.ledgerrocket.com/v2/event-service/api/v1/admin/outbox/stats" \
  -H "x-api-key: lr-test-20250701" | jq .
```

Returns aggregate counters:

```json
{
  "prepared": 0,
  "committed": 2,
  "published": 15432,
  "failed": 0,
  "total": 15434,
  "oldest_prepared_age": null,
  "oldest_committed_age": 1200
}
```

### Per-Event Outbox Record

```bash
curl -s "https://ledger.dev.ledgerrocket.com/v2/event-service/api/v1/events/EVENT_UUID/outbox" \
  -H "x-api-key: lr-test-20250701" | jq .
```

## Monitoring Outbox Health

### Key Metrics

| Metric | Healthy Value | Alert Threshold |
|--------|--------------|-----------------|
| `prepared` count | 0 | > 10 for more than 30 seconds |
| `committed` count | 0-5 (brief spikes OK) | > 50 sustained |
| `failed` count | 0 | Any non-zero value |
| `oldest_prepared_age` | null or < 5s | > 30 seconds |
| `oldest_committed_age` | null or < 10s | > 60 seconds |

### Quick Health Check

The readiness endpoint surfaces outbox health inline:

```bash
curl -s "https://ledger.dev.ledgerrocket.com/v2/event-service/readyz" \
  -H "x-api-key: lr-test-20250701" | jq '.checks.event_outbox'
```

## Failure Investigation

When events are in `failed` state:

1. **Get the failed records:**

   ```bash
   curl -s "https://ledger.dev.ledgerrocket.com/v2/event-service/api/v1/admin/outbox?site_id=5&state=failed" \
     -H "x-api-key: lr-test-20250701" | jq '.[] | {event_id, error}'
   ```

2. **Check the event details:**

   ```bash
   curl -s "https://ledger.dev.ledgerrocket.com/v2/event-service/api/v1/events/FAILED_EVENT_UUID?site_id=5" \
     -H "x-api-key: lr-test-20250701" | jq .
   ```

3. **Common failure causes:**

   | Error Pattern | Cause | Resolution |
   |---------------|-------|------------|
   | Account not found | Account ID does not exist or wrong site | Verify account exists in Ledger Service |
   | Template validation failed | Event metadata does not match template schema | Check `extra_metadata` against template `metadata_schema` |
   | TigerBeetle rejected transfer | Duplicate event_id or constraint violation | Check for duplicate submission; verify account/ledger setup |
   | Insufficient balance | Debit exceeds available balance (if constraints enabled) | Verify account has sufficient funds |

4. **After fixing the root cause**, resubmit the event with a new `event_id` (UUIDv7). Do not reuse the failed event's ID.

## Stuck Event Recovery

If events are stuck in `prepared` or `committed` after a restart, the service should automatically resume processing them. If they remain stuck after multiple restarts, check TigerBeetle connectivity and service logs for errors during the prepare-to-commit or commit-to-publish transitions.
