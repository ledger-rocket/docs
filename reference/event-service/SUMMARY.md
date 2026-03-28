# Event Service v6.55.0 (Go)

Transforms financial events into double-entry transfers via templates.

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/` | Root health check |
| GET | `/livez` | Liveness probe |
| GET | `/readyz` | Readiness probe |
| POST | `/api/v1/events` | Process an event |
| GET | `/api/v1/events` | List events (filter by original_event_id) |
| GET | `/api/v1/events/{event_id}` | Get event |
| GET | `/api/v1/events/{event_id}/outbox` | Event outbox record |
| GET | `/api/v1/events/{event_id}/transfers` | Transfers by event |
| GET | `/api/v1/events/{event_id}/transfers/enriched` | Enriched transfers by event |
| GET | `/api/v1/events/{event_id}/transfers:count` | Count transfers by event |
| POST | `/api/v1/events:validate` | Validate event (dry-run) |
| POST | `/api/v1/events:validate:explain` | Validate with execution trace |
| GET | `/api/v1/accounts/{account_id}/transfers` | Transfers by account |
| GET | `/api/v1/accounts/{account_id}/transfers:count` | Count transfers by account |
| GET | `/api/v1/transfers` | Query transfers |
| GET | `/api/v1/transfers/{id}` | Get transfer |
| GET | `/api/v1/transfers/{id}/enriched` | Enriched transfer |
| GET | `/api/v1/transfers:count` | Count transfers |
| GET | `/api/v1/sites/{site_id}/templates` | List templates |
| GET | `/api/v1/sites/{site_id}/templates/{template_id}` | Get template |
| POST | `/api/v1/sites/{site_id}/templates:refresh-cache` | Refresh template cache |
| GET | `/api/v1/sites/{site_id}/templates:status` | Template loader status |
| POST | `/api/v1/templates:validate` | Validate template (stateless) |
| GET | `/api/v1/admin/outbox` | List outbox records |
| GET | `/api/v1/admin/outbox/stats` | Outbox statistics |

25 endpoints total.

## Key Data Models

### Event

Represents a financial event submitted for processing.

- **event_id** -- UUIDv7 (time-sortable)
- **site_id** -- site scope
- **accounts** -- map of role-name to account ID (e.g., sender, receiver)
- **template_ids** -- list of templates to apply
- **amount** -- transaction amount
- **extra_metadata** -- arbitrary key-value metadata validated against template schema

### Transfer

A single double-entry leg produced by template execution.

- **debit_account_id** -- account debited
- **credit_account_id** -- account credited
- **amount** -- transfer amount
- **ledger_id** -- ledger scope
- **template_id** -- originating template
- **event_id** -- originating event

Enriched transfers include resolved account names, entity details, and ledger metadata.

### Template

Read-only view of accounting templates (managed by the Accounting Rule Engine).

- **variables** -- CEL expressions evaluated against the event
- **validations** -- CEL pre-conditions
- **legs** -- debit/credit mappings with amount expressions
- **metadata_schema** -- JSON Schema for event metadata validation

### AdminOutboxRecordSummary

Tracks event processing through the outbox state machine: `prepared` -> `committed` -> `published`.

### AuditState

Template lifecycle states: `DRAFT`, `PENDING_APPROVAL`, `LIVE`, `DEPRECATED`, `REJECTED`, `DELETED`.
