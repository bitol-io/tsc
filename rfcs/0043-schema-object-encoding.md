# Schema Object Encoding

Champion: TBD (TSC member needed; proposed champion: Jean-Georges Perrin).

Authors: Ludo (`zorender`).

Slack: TBD.

GitHub issue: https://github.com/bitol-io/open-data-contract-standard/issues/263

Applies to:
* [x] ODCS - Open Data Contract Standard
* [ ] ODPS - Open Data Product Standard
* [ ] OORS - Open Observability Results Standard
* [ ] OOCS - Open Orchestration and Control Standard
* [ ] OMMS - Open Maturity Model Standard
* [ ] OMDS - Open Metadata Difference Standard

## Summary

This RFC proposes adding an optional `encoding` field to ODCS schema objects, not to the top-level data contract and not to individual scalar properties. The field declares the expected character encoding for dataset-like schema elements such as files, topics, tables, or object-level data structures.

## Motivation

Character encoding mismatches are a common source of data pipeline failures, especially for flat files, text payloads, event topics, and integrations with legacy systems that may produce data in encodings other than UTF-8.

Today, ODCS users can document encoding in free-text descriptions or in custom properties, but those approaches are inconsistent and difficult for tooling to consume. A standardized `encoding` field lets producers declare the expected encoding explicitly and lets consumers configure readers, validations, and quality checks accordingly.

The original discussion considered a top-level contract field. Maintainer feedback suggested that encoding should instead live at the schema or element level, especially when the element is an object / dataset. This RFC follows that direction.

## Design and examples

### Proposed field

Add an optional `encoding` field to schema objects.

| Field | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| `encoding` | string | No | Expected character encoding for this schema object. Examples: `UTF-8`, `ISO-8859-1`, `ASCII`, `UTF-16`. |

The field is intentionally free-form rather than an enum. This keeps the standard flexible for platform-specific or less common encodings while still documenting common values in examples.

### Scope

`encoding` applies to a schema object / element that represents a dataset-like structure. It is not a top-level data contract field.

It is also not proposed as a field for every scalar property. Character encoding is usually a concern of the serialized dataset, file, topic payload, or object-level representation rather than an individual column.

### Example 1: Minimal schema object encoding

```yaml
apiVersion: v3.1.0
kind: DataContract
name: customer_export
schema:
  - name: customers
    physicalType: file
    physicalName: customers.csv
    encoding: UTF-8
    properties:
      - name: customer_id
        logicalType: string
        required: true
      - name: customer_name
        logicalType: string
```

### Example 2: Legacy file encoded as ISO-8859-1

```yaml
apiVersion: v3.1.0
kind: DataContract
name: legacy_orders
schema:
  - name: orders
    physicalType: file
    physicalName: orders_legacy.csv
    encoding: ISO-8859-1
    properties:
      - name: order_id
        logicalType: string
        required: true
      - name: description
        logicalType: string
      - name: amount
        logicalType: number
```

### JSON Schema impact

Add the following optional property to `SchemaObject` definitions:

```json
"encoding": {
  "type": "string",
  "description": "The expected character encoding for this schema object, for example UTF-8, ISO-8859-1, ASCII or UTF-16.",
  "examples": ["UTF-8", "ISO-8859-1", "ASCII", "UTF-16"]
}
```

Documentation should list `encoding` under object-specific schema fields.

## Alternatives

### Top-level contract field

Rejected for this RFC. A top-level `encoding` field is simple, but it assumes a single encoding for the whole contract. ODCS contracts can describe several schema objects, and different dataset-like elements may have different encodings. Maintainer feedback also explicitly suggested avoiding the top-level location.

### Property-level field

Rejected for this RFC. A field on every scalar property would be more granular, but it is likely unnecessary for the common case and would increase schema complexity. Encoding usually describes the serialized object or dataset, not individual logical fields.

### `customProperties`

Rejected as the standard approach. `customProperties` can already represent encoding metadata, but values would not be standardized. Tooling could not reliably depend on a common key or semantics.

### Enum of allowed encodings

Rejected for now. An enum would improve validation but risks excluding valid platform-specific encodings. This RFC proposes a free-form string and documents common examples.

## Decision

TBD by TSC.

## Consequences

### Positive

- Makes encoding machine-readable in ODCS.
- Keeps the change scoped to schema objects, following maintainer feedback.
- Supports multiple schema objects with different encodings in the same contract.
- Avoids over-engineering at scalar property level.
- Remains backward compatible because the field is optional.

### Trade-offs

- Free-form values may vary in spelling or casing (`UTF-8` vs `utf8`).
- Tooling may need normalization if it wants strict behavior.
- The RFC does not define per-property encoding semantics.

## References

- Issue: https://github.com/bitol-io/open-data-contract-standard/issues/263
- Initial implementation PR: https://github.com/bitol-io/open-data-contract-standard/pull/269
- RFC process: https://github.com/bitol-io/tsc/tree/main/rfcs
