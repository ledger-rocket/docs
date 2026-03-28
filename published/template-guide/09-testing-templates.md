# Testing Templates

LedgerRocket provides three validation endpoints for testing templates without affecting live data. Use these during development to verify template logic, debug expression errors, and confirm transfer generation before submitting for approval.

## Validation Endpoints

### Template Validation (Stateless)

```text
POST /api/v1/templates:validate
```

Validates the template structure and compiles CEL expressions without requiring a running ledger or account data. This is a pure structural check.

**What it checks:**

- JSON structure validity against the template schema
- CEL expression compilation (variables, validations)
- Leg field reference resolution
- Accounting treatment consistency (legs reference declared treatments)
- Account code validation structure

**Request body:** The full template JSON, optionally with an `event_request` and `expected_transfers` for transfer generation testing.

```json
{
    "template": {
        "$schema": "https://ledger-rocket.github.io/schemas/domain/template.schema.json",
        "site_id": 5,
        "template_id": 9999,
        "name": "my_test_template",
        "..."
    }
}
```

**Response includes:**

- `validation_errors`: array of simple error strings (for basic tooling)
- `validation_details`: array of structured error objects with:
  - `category`: error type (`JSON_STRUCTURE`, `CEL_EXPRESSION`, `BUSINESS_LOGIC`, `REFERENCE`, `TYPE`, `RANGE`)
  - `field`: specific field path (e.g., `variables[0].expression`, `legs[2].amount`)
  - `message`: human-readable error description
  - `suggestion`: actionable fix recommendation

**With event_request:** If you include an `event_request` in the body, the endpoint generates transfers using the template and returns them as `generated_expected_transfers`. This lets you verify transfer generation logic.

**With expected_transfers:** If you also include `expected_transfers`, the endpoint compares generated transfers against expected ones and reports any mismatches.

**Auto-generation:** If you provide `event_request` without `expected_transfers`, the endpoint auto-generates expected transfers that you can copy into your template's `examples` array.

### Event Validation (Dry Run)

```text
POST /api/v1/events:validate
```

Runs the full processing pipeline -- account enrichment, metadata schema validation, variable evaluation, validations, leg generation -- **without** submitting transfers to the ledger. This requires connectivity to the Ledger Service for account lookup.

**What it checks:**

- Everything from template validation, plus:
- Account existence and type matching (via Ledger Service)
- Account code mask matching against `account_code_validations`
- Metadata schema validation against the event's `extra_metadata`
- Variable evaluation with real account data
- Validation expression evaluation
- Transfer generation with real account IDs

**Request body:** A standard event request:

```json
{
    "event": {
        "$schema": "https://ledger-rocket.github.io/schemas/domain/event.schema.json",
        "site_id": 5,
        "template_ids": [5403],
        "event_id": "018f3f0b-4a6f-7b31-9a7a-2b9b1d2a5c01",
        "event_source": "api",
        "occurred_at": 1705315800000,
        "amount": 2500,
        "accounts": [
            { "account_id": "50050000-0000-4001-a000-000000000009", "purpose": "bank_fees_account" },
            { "account_id": "50050000-0000-4001-a000-000000000004", "purpose": "suspense_account" }
        ],
        "extra_metadata": {
            "anomaly_type": "BANK_FEE",
            "investigation_reference": "JIRA-18422",
            "approver": "ops_analyst_17",
            "original_suspense_event_id": "550e8400-e29b-41d4-a716-446655440001"
        }
    }
}
```

**Response:** Validation result with generated transfers (but no actual ledger writes). The `trace` field is `null` for this endpoint.

### Event Validation Trace (Explain Mode)

```text
POST /api/v1/events:validate:explain
```

Identical to the dry-run endpoint but additionally returns a detailed execution trace. Use this for debugging complex templates.

**Request body:** Same as `POST /events:validate`.

**Response includes the `trace` object:**

- **Variables**: each CEL expression and its resolved value per template
- **Validations**: pass/fail status with error details for each validation expression
- **Legs**: condition outcomes, skip reasons, resolved debit/credit account IDs, and generated amounts

The trace is deterministic -- running the same event through the same template always produces the same trace. Use it to understand exactly how the engine interpreted your template.

## Template Development Workflow

### Step 1: Write the Template

Start with the structure from [Template Structure](01-template-structure.md). Define your accounts, metadata schema, variables, validations, and legs.

### Step 2: Validate Structure

Use `POST /templates:validate` to check for structural errors and CEL compilation issues. This is fast and does not require account data:

```bash
curl -X POST https://ledger.dev.ledgerrocket.com/v2/event-service/api/v1/templates:validate \
  -H "Content-Type: application/json" \
  -H "x-api-key: lr-test-20250701" \
  -d @my-template.json
```

Fix any errors reported in `validation_errors` or `validation_details`.

### Step 3: Test with Dry Run

Once the template passes structural validation, test it with a real event using `POST /events:validate`. This verifies that:

- The accounts you reference actually exist
- Account codes match the masks in your `account_code_validations`
- The metadata schema accepts your test metadata
- Variables compute expected values
- Validations pass with your test data
- Legs generate the correct transfers

### Step 4: Use Explain Mode for Debugging

If transfers are not what you expect, use `POST /events:validate:explain` to see the full execution trace. Check:

- Did variables evaluate to the expected values?
- Did any validations fail unexpectedly?
- Were any legs skipped due to conditions?
- Are the debit/credit accounts resolved correctly?

### Step 5: Add Examples

Once your template generates correct transfers, add example scenarios to the template's `examples` array. Each example includes an `event_request` and `expected_transfers`:

```json
"examples": [
    {
        "name": "standard_bank_fee_resolution",
        "description": "$25.00 wire transfer fee identified in suspense.",
        "event_request": {
            "$schema": "https://ledger-rocket.github.io/schemas/domain/event.schema.json",
            "site_id": 5,
            "template_ids": [5403],
            "event_id": "018f3f0b-4a6f-7b31-9a7a-2b9b1d2a5c01",
            "event_source": "api",
            "occurred_at": 1705315800000,
            "amount": 2500,
            "accounts": [
                { "account_id": "50050000-0000-4001-a000-000000000009", "purpose": "bank_fees_account" },
                { "account_id": "50050000-0000-4001-a000-000000000004", "purpose": "suspense_account" }
            ],
            "extra_metadata": {
                "anomaly_type": "BANK_FEE",
                "investigation_reference": "JIRA-18422",
                "approver": "ops_analyst_17",
                "original_suspense_event_id": "550e8400-e29b-41d4-a716-446655440001"
            }
        },
        "expected_transfers": [
            {
                "amount": 2500,
                "debit_account_id": "50050000-0000-4001-a000-000000000009",
                "credit_account_id": "50050000-0000-4001-a000-000000000004",
                "leg_number": 1
            }
        ]
    }
]
```

You can use `POST /templates:validate` with an `event_request` to auto-generate the `expected_transfers` array.

### Step 6: Submit for Approval

Once the template is tested and includes examples, submit it for maker-checker review:

```bash
# Create/update the template (DRAFT)
curl -X PUT https://ledger.dev.ledgerrocket.com/v2/event-service/api/v1/sites/5/templates/5403 \
  -H "Content-Type: application/json" \
  -H "x-api-key: lr-test-20250701" \
  -d @my-template.json

# Submit for approval
curl -X POST https://ledger.dev.ledgerrocket.com/v2/event-service/api/v1/sites/5/templates/5403/submit \
  -H "x-api-key: lr-test-20250701"
```

See [Template Lifecycle](08-template-lifecycle.md) for the full approval workflow.

## Error Categories

When validation fails, errors are categorized to help you find the root cause:

| Category | Description | Example |
|----------|-------------|---------|
| `JSON_STRUCTURE` | Template JSON does not match the expected schema | Missing required field `legs` |
| `CEL_EXPRESSION` | A CEL expression failed to compile | Syntax error in `variables[0].value` |
| `BUSINESS_LOGIC` | Template logic is internally inconsistent | Leg references undefined treatment type |
| `REFERENCE` | A reference to another field or entity is broken | Leg references undefined variable name |
| `TYPE` | Type mismatch in an expression or field | Amount expression evaluates to string instead of int |
| `RANGE` | A numeric value is outside its valid range | `template_id` exceeds 65535 |

## Tips

- **Start with a known-good template.** Copy an existing production template and modify it, rather than starting from scratch.
- **Test incrementally.** Validate after each change, not after writing the entire template.
- **Use the explain endpoint sparingly.** It returns detailed traces that are useful for debugging but add response overhead.
- **Check account code masks.** A common error is sending an account with code `10100` when the mask requires `10102`.
- **Verify metadata field names.** Typos in `extra_metadata` field names cause validation failures because `additionalProperties: false` rejects unknown fields.
