# Metadata Schemas

Every template defines a `metadata_schema` that validates the `extra_metadata` field on incoming events. This schema uses [JSON Schema 2020-12](https://json-schema.org/draft/2020-12/schema) and is compiled at template load time for high-performance validation.

## Schema Structure

The `metadata_schema` field is a standard JSON Schema object:

```json
"metadata_schema": {
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "type": "object",
    "additionalProperties": false,
    "properties": {
        "field_name": {
            "type": "string",
            "description": "What this field represents."
        }
    },
    "required": ["field_name"]
}
```

Key conventions:

- **Always use `$schema: "https://json-schema.org/draft/2020-12/schema"`** -- this is the standard across the platform.
- **Always set `"additionalProperties": false`** -- this prevents unrecognized fields from being silently accepted.
- **Always set `"type": "object"`** -- metadata is always a JSON object.

## How Validation Works

When an event is submitted:

1. The engine extracts the event's `extra_metadata` field.
2. For each template referenced in `template_ids`, the engine validates `extra_metadata` against that template's compiled `metadata_schema`.
3. If validation fails, the event is rejected with a detailed error message identifying the failing field and constraint.

Metadata schema validation happens during the enrichment phase, before variables and validations are evaluated.

## Required vs. Optional Fields

### Required Fields

Fields listed in the `required` array must be present in every event:

```json
"required": ["anomaly_type", "investigation_reference", "approver"]
```

### Optional Fields

Fields defined in `properties` but not listed in `required` are optional -- the event is valid whether they are present or absent:

```json
"properties": {
    "marketplace": {
        "type": "string",
        "description": "White-labelled product line identifier. Optional.",
        "minLength": 1,
        "maxLength": 100
    },
    "payment_rail": {
        "type": "string",
        "description": "Inbound payment rail. Optional.",
        "minLength": 1,
        "maxLength": 50
    },
    "statement_file_id": {
        "type": "string",
        "description": "Source bank statement file. Required."
    }
},
"required": ["statement_file_id"]
```

In this example from **template 5401**, `marketplace` and `payment_rail` are optional, while `statement_file_id` is required.

## Common Field Types and Constraints

### String Fields

```json
{
    "type": "string",
    "description": "Reference to the investigation ticket.",
    "minLength": 1,
    "maxLength": 200
}
```

### UUID Fields

```json
{
    "type": "string",
    "format": "uuid",
    "description": "Event ID (UUID) of the original suspense-creating event."
}
```

### Enum Fields

```json
{
    "type": "string",
    "enum": ["BANK_FEE"],
    "description": "Classification of the suspense anomaly (must be BANK_FEE)."
}
```

### Integer Fields

```json
{
    "type": "integer",
    "minimum": 1,
    "description": "Number of return entries expected in this file."
}
```

### Nullable String Fields

```json
{
    "type": ["string", "null"],
    "description": "Optional human-readable description from the bank statement line."
}
```

### Direction Enum

```json
{
    "type": "string",
    "enum": ["out", "in"],
    "description": "Direction of the sweep movement."
}
```

## Real-World Examples

### Template 5403: Suspense Resolve Bank Fee

This template requires full investigation metadata for audit compliance:

```json
"metadata_schema": {
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "type": "object",
    "additionalProperties": false,
    "properties": {
        "original_suspense_event_id": {
            "type": "string",
            "format": "uuid",
            "description": "Event ID (UUID) of the original suspense-creating event being resolved."
        },
        "anomaly_type": {
            "type": "string",
            "enum": ["BANK_FEE"],
            "description": "Classification of the suspense anomaly being resolved (must be BANK_FEE)."
        },
        "investigation_reference": {
            "type": "string",
            "minLength": 1,
            "maxLength": 200,
            "description": "Reference to the investigation ticket (e.g., JIRA-12345)."
        },
        "approver": {
            "type": "string",
            "minLength": 1,
            "maxLength": 200,
            "description": "Identifier of the approving operations analyst."
        }
    },
    "required": [
        "anomaly_type",
        "investigation_reference",
        "approver",
        "original_suspense_event_id"
    ]
}
```

A matching event would include:

```json
"extra_metadata": {
    "anomaly_type": "BANK_FEE",
    "investigation_reference": "JIRA-18422",
    "approver": "ops_analyst_17",
    "original_suspense_event_id": "550e8400-e29b-41d4-a716-446655440001"
}
```

### Template 5401: Bank Statement Ingestion

This template mixes required and optional fields:

```json
"metadata_schema": {
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "type": "object",
    "additionalProperties": false,
    "properties": {
        "marketplace": {
            "type": "string",
            "description": "White-labelled product line identifier.",
            "minLength": 1,
            "maxLength": 100
        },
        "payment_rail": {
            "type": "string",
            "description": "Inbound payment rail (e.g., ACH, WIRE).",
            "minLength": 1,
            "maxLength": 50
        },
        "line_item_description": {
            "type": ["string", "null"],
            "description": "Optional human-readable description from the bank statement line."
        },
        "statement_file_id": {
            "type": "string",
            "description": "External_id of the source bank statement file."
        },
        "transaction_ref": {
            "type": "string",
            "description": "Bank-provided transaction reference identifier."
        },
        "line_count": {
            "type": "integer",
            "description": "Number of line items in the bank statement file."
        },
        "total_amount": {
            "type": "string",
            "description": "Total amount of all line items as a decimal string (e.g., \"333.50\")."
        }
    },
    "required": ["statement_file_id", "transaction_ref"]
}
```

Only `statement_file_id` and `transaction_ref` are required. All other fields are optional analytics and traceability metadata.

### Template 5602: Sweep Customer Allocation

This template requires five fields for sweep tracking:

```json
"metadata_schema": {
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "type": "object",
    "additionalProperties": false,
    "properties": {
        "fbo_balance": {
            "type": "string",
            "description": "Master FBO balance formatted as dollars with cents."
        },
        "network_position": {
            "type": "string",
            "description": "Net network position formatted as dollars with cents."
        },
        "consumer_external_id": {
            "type": "string",
            "description": "Consumer entity external identifier.",
            "minLength": 1,
            "maxLength": 100
        },
        "sweep_batch_id": {
            "type": "string",
            "description": "Unique identifier for the sweep batch.",
            "minLength": 1,
            "maxLength": 100
        },
        "direction": {
            "type": "string",
            "enum": ["out", "in"],
            "description": "Direction of the sweep movement."
        }
    },
    "required": [
        "sweep_batch_id",
        "consumer_external_id",
        "direction",
        "fbo_balance",
        "network_position"
    ]
}
```

## Design Guidelines

1. **Be explicit about required fields.** Every field that is essential for audit or processing should be in the `required` array.
2. **Use `additionalProperties: false`.** This catches typos and prevents undocumented data from entering the system.
3. **Add descriptions to every field.** These descriptions appear in validation error messages and documentation.
4. **Use `minLength` / `maxLength` on strings.** This prevents empty strings and unbounded input.
5. **Use `enum` for controlled vocabularies.** When a field has a fixed set of valid values, constrain it with an enum.
6. **Use `format: "uuid"` for UUID fields.** This validates UUID format at submission time.
