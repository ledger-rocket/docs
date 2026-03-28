# Store Metadata

## Overview

You can attach up to 10 tags to Ledger Entries to preserve arbitrary key-value pairs, such as identifiers from your system.

## Schema Entry Tags

Tags can be defined directly within your Ledger Entry type definitions in the Schema. Use template parameters to populate tag values dynamically:

```json
{
  "tags": [
    {
      "key": "user",
      "value": "{{user_id}}"
    },
    {
      "key": "deposit_flow",
      "value": "{{deposit_flow_id}}"
    }
  ]
}
```

When posting an entry, supply the parameter values in your request. The system automatically applies the schema-defined tags with interpolated values to the resulting entry.

## Runtime Entry Tags

Tags can also be supplied at posting time, independent of schema definitions:

```json
{
  "tags": [
    {
      "key": "operator",
      "value": "alice"
    }
  ]
}
```

If you define tags both in the schema and at runtime, the entry receives the combined set. You may only repeat a tag key if both instances share identical values.

## Updating Entry Tags

After posting, you can modify tags using the `updateLedgerEntry` mutation. This operation supports both adding/updating and removing tags.

**Adding or updating tags:** Provide the tag key-value pairs to modify. Existing tags with the same key are updated; new keys are added; unmentioned tags remain unchanged.

**Removing tags:** Use the `tagsToRemove` field, specifying both key and value for each tag to delete. If a tag doesn't exist, the removal is silently ignored.

You can perform both additions and removals in a single update call, though you cannot add and remove the same tag simultaneously.

**Limitation:** Each entry supports a maximum of 10 updates throughout its lifecycle.
