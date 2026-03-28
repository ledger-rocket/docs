# Accounting Rule Engine v1.16.1 (Go)

Template and workflow management with maker-checker lifecycle.

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/livez` | Liveness probe |
| GET | `/readyz` | Readiness probe |
| GET | `/api/v1/sites/{site_id}/templates` | List templates |
| GET | `/api/v1/sites/{site_id}/templates/{template_id}` | Get template |
| PUT | `/api/v1/sites/{site_id}/templates/{template_id}` | Put template |
| POST | `/api/v1/sites/{site_id}/templates/{template_id}/approve` | Approve template |
| POST | `/api/v1/sites/{site_id}/templates/{template_id}/delete` | Delete template |
| POST | `/api/v1/sites/{site_id}/templates/{template_id}/deprecate` | Deprecate template |
| POST | `/api/v1/sites/{site_id}/templates/{template_id}/reject` | Reject template |
| POST | `/api/v1/sites/{site_id}/templates/{template_id}/submit` | Submit for approval |
| POST | `/api/v1/sites/{site_id}/templates/{template_id}/undelete` | Undelete template |
| POST | `/api/v1/sites/{site_id}/templates/{template_id}/withdraw` | Withdraw submission |
| GET | `/api/v1/sites/{site_id}/templates/{template_id}/audit-slugs` | Template audit trail |
| POST | `/api/v1/sites/{site_id}/templates:refresh-cache` | Refresh template cache |
| GET | `/api/v1/sites/{site_id}/templates:status` | Template loader status |
| POST | `/api/v1/templates:validate` | Validate template (stateless) |
| GET | `/api/v1/sites/{site_id}/workflows` | List workflows |
| POST | `/api/v1/sites/{site_id}/workflows` | Create workflow |
| GET | `/api/v1/sites/{site_id}/workflows/{workflow_id}` | Get workflow |
| PUT | `/api/v1/sites/{site_id}/workflows/{workflow_id}` | Update workflow |
| POST | `/api/v1/sites/{site_id}/workflows/{workflow_id}/approve` | Approve workflow |
| POST | `/api/v1/sites/{site_id}/workflows/{workflow_id}/delete` | Delete workflow |
| POST | `/api/v1/sites/{site_id}/workflows/{workflow_id}/deprecate` | Deprecate workflow |
| POST | `/api/v1/sites/{site_id}/workflows/{workflow_id}/reject` | Reject workflow |
| POST | `/api/v1/sites/{site_id}/workflows/{workflow_id}/submit` | Submit for approval |
| POST | `/api/v1/sites/{site_id}/workflows/{workflow_id}/undelete` | Undelete workflow |
| POST | `/api/v1/sites/{site_id}/workflows/{workflow_id}/withdraw` | Withdraw submission |
| GET | `/api/v1/sites/{site_id}/workflows/{workflow_id}/audit-slugs` | Workflow audit trail |

29 endpoints total. Templates and workflows share the same maker-checker lifecycle pattern.

## Key Data Models

### Template

Defines how a financial event produces accounting entries. Composed of:

- **variables** -- CEL expressions evaluated against the incoming event
- **validations** -- CEL-based pre-conditions that must pass
- **legs** -- debit/credit account mappings with amount expressions
- **metadata_schema** -- JSON Schema (2020-12) for event metadata validation
- **accounting_treatment** -- description, policy references, and treatment type

### Workflow

Multi-step orchestration that chains templates or actions together.

### AuditSlug

Tracks maker-checker state transitions. Each slug records the action, actor, and timestamp.

### AuditState

Lifecycle states: `DRAFT` -> `PENDING_APPROVAL` -> `LIVE` -> `DEPRECATED` / `REJECTED` / `DELETED`

### AuditAction

Transition actions: `CREATE`, `UPDATE`, `SUBMIT_FOR_APPROVAL`, `APPROVE`, `REJECT`, `WITHDRAW`, `DELETE`, `UNDELETE`, `DEPRECATE`, `ROLLBACK`

### AccountingTreatment

- **description** -- human-readable explanation
- **policy_refs** -- references to accounting policy documents
- **type** -- treatment classification

### AccountType

Standard account types: `ASSET`, `EQUITY`, `EXPENSE`, `LIABILITY`, `REVENUE`, `SUPPLEMENTAL`.
