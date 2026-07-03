# RFC-0047: Adding `id` to Relationship Objects

Champion: Diego Carvallo.

Authors: Jean-Georges Perrin, Diego Carvallo.

Slack: *TBD*.

GitHub issue: https://github.com/bitol-io/open-data-contract-standard/issues/277

Applies to:
* [x] ODCS - Open Data Contract Standard
* [ ] ODPS - Open Data Product Standard
* [ ] OORS - Open Observability Results Standard
* [ ] OOCS - Open Orchestration and Control Standard
* [ ] OMMS - Open Maturity Model Standard
* [ ] OMDS - Open Metadata Difference Standard

## Summary

Add an optional `id` field to relationship objects (`RelationshipBase`, surfaced as `RelationshipSchemaLevel` and `RelationshipPropertyLevel`) in ODCS, completing the work started in [RFC-0026a](./approved/odcs-v3.1.0/0026a-reference-id.md). This is a narrow, additive change that brings relationship objects in line with every other referenceable array-item object in the standard.

## Motivation

[RFC-0026a](./approved/odcs-v3.1.0/0026a-reference-id.md) introduced optional, stable `id` fields across referenceable array-item objects in ODCS — servers, schema objects, properties, support channels, roles, SLA properties, custom properties, and quality rules. The goal was to enable robust, position-independent references that survive name changes and array reordering.

Relationship objects (`RelationshipSchemaLevel` and `RelationshipPropertyLevel`) were excluded from that scope. They are currently the only array-item objects in ODCS that cannot be addressed by a stable identifier. This creates an inconsistency: every other referenceable object can be pointed to by `id`, but relationships must still be identified by position or by reconstructing a composite key from their `from`/`to` values.

This gap has practical consequences:

- **Tooling and diffs**: [OMDS](./0039-omds.md) (Open Metadata Difference Standard) needs to describe changes to specific relationships across contract versions. Without a stable `id`, a rename of a referenced column or a reordering of the relationships array makes it impossible to determine whether a relationship was modified or replaced.
- **Observability**: OORS results that flag a broken or degraded relationship cannot point to the exact relationship object in a stable, machine-readable way.
- **Imports and cross-contract references**: RFC-0032 (imports) and future reference mechanisms depend on being able to address specific array items by a stable key. Relationships are currently unaddressable.

Adding `id` to `RelationshipBase` resolves this inconsistency with a single, optional field that has zero impact on existing contracts.

> **Note on `team` members:** Team members (`team.members[]`) are out of scope for both RFC-0026a and this RFC. They are currently addressed by `username`, which is a required, naturally stable identifier for that object type. username will need to be deprecated for id in the future.

## Design and examples

Add an optional `id` to `RelationshipBase`, following the same rules established by RFC-0026a:

- `id` is optional.
- `id` MUST be unique within its containing `relationships` array.
- `id` SHOULD be stable across contract versions to preserve referential integrity.
- UUIDs are allowed for `id` values.
- An `id` cannot contain the following characters: `.` `#` `/` `\` `@` `!` `%` `&` `^`

### YAML examples

Minimal property-level example:

```yaml
schema:
  - name: orders
    properties:
      - name: customer_id
        relationships:
          - id: fk_order_customer
            to: customers.id
```

Schema-level example with composite key:

```yaml
schema:
  - name: order_items
    relationships:
      - id: fk_order_items_order
        from:
          - order_items.order_id
          - order_items.line_number
        to:
          - orders.order_id
          - orders.line_id
        type: foreignKey
```

### JSON Schema change

The change is made to `RelationshipBase`, which is inherited by both `RelationshipSchemaLevel` and `RelationshipPropertyLevel`.

**Before** (`RelationshipBase` in `odcs-json-schema-latest.json`):

```json
"RelationshipBase": {
  "type": "object",
  "description": "Base definition for relationships between properties, typically for foreign key constraints.",
  "properties": {
    "type": {
      "type": "string",
      "description": "The type of relationship. Defaults to 'foreignKey'.",
      "default": "foreignKey",
      "enum": ["foreignKey"]
    },
    "from": {
      "oneOf": [
        {
          "anyOf": [
            { "$ref": "#/$defs/ShorthandReference" },
            { "$ref": "#/$defs/FullyQualifiedReference" }
          ],
          "description": "Source property reference using fully qualified or shorthand notation."
        },
        {
          "type": "array",
          "description": "Array of source properties for composite keys.",
          "items": {
            "anyOf": [
              { "$ref": "#/$defs/ShorthandReference" },
              { "$ref": "#/$defs/FullyQualifiedReference" }
            ]
          },
          "minItems": 1
        }
      ],
      "description": "Source property or properties."
    },
    "to": {
      "oneOf": [
        {
          "anyOf": [
            { "$ref": "#/$defs/ShorthandReference" },
            { "$ref": "#/$defs/FullyQualifiedReference" }
          ],
          "description": "Target property reference using fully qualified or shorthand notation."
        },
        {
          "type": "array",
          "description": "Array of target properties for composite keys.",
          "items": {
            "anyOf": [
              { "$ref": "#/$defs/ShorthandReference" },
              { "$ref": "#/$defs/FullyQualifiedReference" }
            ]
          },
          "minItems": 1
        }
      ],
      "description": "Target property or properties to reference."
    },
    "customProperties": {
      "$ref": "#/$defs/CustomProperties"
    }
  }
}
```

**After** (add `id` as the first property in `RelationshipBase.properties`):

```json
"RelationshipBase": {
  "type": "object",
  "description": "Base definition for relationships between properties, typically for foreign key constraints.",
  "properties": {
    "id": {
      "type": "string",
      "description": "Optional stable identifier for this relationship, unique within its containing relationships array. SHOULD remain stable across contract versions. Cannot contain: . # / \\ @ ! % & ^"
    },
    "type": {
      "type": "string",
      "description": "The type of relationship. Defaults to 'foreignKey'.",
      "default": "foreignKey",
      "enum": ["foreignKey"]
    },
    "from": {
      "oneOf": [
        {
          "anyOf": [
            { "$ref": "#/$defs/ShorthandReference" },
            { "$ref": "#/$defs/FullyQualifiedReference" }
          ],
          "description": "Source property reference using fully qualified or shorthand notation."
        },
        {
          "type": "array",
          "description": "Array of source properties for composite keys.",
          "items": {
            "anyOf": [
              { "$ref": "#/$defs/ShorthandReference" },
              { "$ref": "#/$defs/FullyQualifiedReference" }
            ]
          },
          "minItems": 1
        }
      ],
      "description": "Source property or properties."
    },
    "to": {
      "oneOf": [
        {
          "anyOf": [
            { "$ref": "#/$defs/ShorthandReference" },
            { "$ref": "#/$defs/FullyQualifiedReference" }
          ],
          "description": "Target property reference using fully qualified or shorthand notation."
        },
        {
          "type": "array",
          "description": "Array of target properties for composite keys.",
          "items": {
            "anyOf": [
              { "$ref": "#/$defs/ShorthandReference" },
              { "$ref": "#/$defs/FullyQualifiedReference" }
            ]
          },
          "minItems": 1
        }
      ],
      "description": "Target property or properties to reference."
    },
    "customProperties": {
      "$ref": "#/$defs/CustomProperties"
    }
  }
}
```

The `id` field does not need to be added to `RelationshipSchemaLevel` or `RelationshipPropertyLevel` directly — both inherit it via `allOf: [{ "$ref": "#/$defs/RelationshipBase" }]`. No other changes to those definitions are required.

Document the field in `references.md` alongside the other relationship fields.

## Alternatives

Keep relying on positional/name-based identification of relationships — rejected as inconsistent with RFC-0026a and brittle under reordering.

## Decision

> The decision made by the TSC.

## Consequences

Optional and additive; no impact on existing contracts. Brings relationship objects in line with every other referenceable object in the standard.

## References

* [RFC-0026a: Adding ID Field for Stable References](./approved/odcs-v3.1.0/0026a-reference-id.md)
* [ODCS issue #277](https://github.com/bitol-io/open-data-contract-standard/issues/277)
