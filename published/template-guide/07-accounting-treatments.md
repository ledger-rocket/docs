# Accounting Treatments

Accounting treatments classify the financial nature of each transfer leg and link it to accounting policy documentation. Every template must declare at least one treatment, and every leg must reference one by its `type` field.

## Structure

Each accounting treatment has three fields:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | Treatment classification identifier (e.g., `internal_transfer`, `adjustment_manual`). |
| `description` | string | Yes | Human-readable explanation of the accounting treatment. Max 300 characters. |
| `policy_refs` | array | Yes | At least one reference to an accounting policy document. |

### Policy Reference Format

Each policy reference has:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | Yes | Policy identifier (e.g., `ASC_205`, `IFRS_9`, `IAS_1`). |
| `version` | integer | No | Explicit policy version. If omitted, the active version is resolved automatically. |

## How Legs Reference Treatments

Each leg's `treatment_type` field must match the `type` of one of the template's `accounting_treatments`. A template can define multiple treatments, and different legs can reference different treatments.

```json
{
    "accounting_treatments": [
        {
            "type": "settlement_clearing",
            "policy_refs": [{ "id": "ASC_205", "version": 1 }],
            "description": "Flush Control against Suspense for reconciliation."
        },
        {
            "type": "adjustment_manual",
            "policy_refs": [{ "id": "ASC_250", "version": 1 }],
            "description": "Manual correction to resolve orphaned suspense item."
        }
    ],
    "legs": [
        {
            "leg_number": 1,
            "treatment_type": "settlement_clearing",
            "..."
        },
        {
            "leg_number": 2,
            "treatment_type": "adjustment_manual",
            "..."
        }
    ]
}
```

## Common Treatment Types

These treatment types appear in production templates:

### `internal_transfer`

Asset-to-asset or liability-to-liability reclassification within the same entity. No P&L impact and no customer balance impact.

```json
{
    "type": "internal_transfer",
    "policy_refs": [{ "id": "ASC_305", "version": 1 }],
    "description": "Reclassify confirmed inbound cash from Settled to Control for reconciliation staging; no P&L impact."
}
```

Used in: bank statement ingestion (T5401), sweep payment initiation (T5614), consumer fund release.

### `settlement_clearing`

Clearing and settlement operations that zero out suspense, control, or clearing accounts as part of the reconciliation cycle.

```json
{
    "type": "settlement_clearing",
    "policy_refs": [
        { "id": "ASC_205", "version": 1 },
        { "id": "ASC_942", "version": 1 }
    ],
    "description": "Zero out suspense against control, completing the reconciliation cycle."
}
```

Used in: reconciliation flush (T5402), sweep reconciliation (T5617), bank payment reconciliation (T5215).

### `adjustment_manual`

Manual corrections and allocations that require investigation and approval. Typically used for suspense resolution.

```json
{
    "type": "adjustment_manual",
    "policy_refs": [
        { "id": "ASC_250", "version": 1 },
        { "id": "ASC_205", "version": 1 }
    ],
    "description": "Manual, approved reconciliation adjustment to resolve an orphaned suspense item."
}
```

Used in: suspense resolve bank fee (T5403), control orphan resolution (T5406), suspense resolve inbound credit (T5405).

### `accrual_non_interest`

Recognition of non-interest income or expense items during reconciliation.

```json
{
    "type": "accrual_non_interest",
    "policy_refs": [
        { "id": "ASC_205", "version": 1 },
        { "id": "ASC_250", "version": 1 }
    ],
    "description": "Recognize bank service charges as non-interest expense in the period identified."
}
```

Used in: suspense resolve bank fee (T5403).

## Policy Reference Examples

### US GAAP (ASC) References

| ID | Description |
|----|-------------|
| `ASC_205` | Presentation of Financial Statements |
| `ASC_250` | Accounting Changes and Error Corrections |
| `ASC_305` | Cash and Cash Equivalents |
| `ASC_942` | Financial Services -- Depository and Lending |

### IFRS References

| ID | Description |
|----|-------------|
| `IAS_1` | Presentation of Financial Statements |
| `IFRS_9` | Financial Instruments |

## Real-World Example: Template 5402

This four-leg reconciliation template uses two different treatment types and references both IFRS policies:

```json
{
    "template_id": 5402,
    "name": "reconciliation_flush_clearing",
    "accounting_treatments": [
        {
            "type": "settlement_clearing",
            "policy_refs": [
                { "id": "IAS_1", "version": 1 },
                { "id": "IFRS_9", "version": 1 }
            ],
            "description": "Flush clearing balance into Suspense for reconciliation against bank statement."
        },
        {
            "type": "internal_transfer",
            "policy_refs": [{ "id": "IAS_1", "version": 1 }],
            "description": "Release consumer funds from Hold to Available after bank confirmation."
        },
        {
            "type": "settlement_clearing",
            "policy_refs": [
                { "id": "IAS_1", "version": 1 },
                { "id": "IFRS_9", "version": 1 }
            ],
            "description": "Mirror the clearing flush on the allocation ledger for FDIC position tracking."
        },
        {
            "type": "internal_transfer",
            "policy_refs": [{ "id": "IAS_1", "version": 1 }],
            "description": "Release consumer FDIC custodial position from FBO Clearing to FBO Main."
        }
    ]
}
```

Note that a template can declare multiple treatments of the same type (e.g., two `settlement_clearing` entries and two `internal_transfer` entries) when the accounting rationale differs between legs.

## Design Guidelines

1. **One treatment per accounting rationale.** If two legs have different audit justifications, declare separate treatments even if they share the same type.
2. **Always include policy references.** Every treatment must cite at least one accounting standard (ASC, IFRS, IAS, or internal policy).
3. **Write clear descriptions.** The description should explain what the treatment does and why, not just restate the type name.
4. **Use consistent type names.** Stick to the established vocabulary (`internal_transfer`, `settlement_clearing`, `adjustment_manual`, `accrual_non_interest`) unless introducing a genuinely new accounting pattern.
