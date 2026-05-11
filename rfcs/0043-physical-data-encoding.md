# Physical Data Encoding

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

This RFC proposes standardizing how ODCS declares the expected character encoding of the physical data payload, for example `UTF-8`, `ISO-8859-1`, `ASCII`, or `UTF-16`.

The proposal distinguishes two concerns:

1. The ODCS contract document itself should be UTF-8 encoded.
2. The data described by the contract may have its own encoding, especially for files, topics, and other serialized payloads.

After community feedback, the preferred direction is to model data payload encoding close to existing physical access details such as `format` and `delimiter`, i.e. on relevant server definitions. This is because the same contract can expose data through different servers or platforms, and each physical representation may have a different encoding.

## Motivation

Character encoding mismatches are a common source of data pipeline failures, especially for flat files, text payloads, event topics, and integrations with legacy systems that may produce data in encodings other than UTF-8.

Today, ODCS users can document encoding in free-text descriptions or in custom properties, but those approaches are inconsistent and difficult for tooling to consume. A standardized `encoding` field lets producers declare the expected encoding explicitly and lets consumers configure readers, validations, and quality checks accordingly.

The original issue proposed a top-level contract field. Early feedback suggested avoiding the top-level location and considering schema / element level. Further feedback pointed out that physical data encoding plays in the same area as file `format` and `delimiter`, which are currently modeled on server definitions such as S3-compatible, Azure, local, Kafka, and similar infrastructure servers.

This RFC therefore frames `encoding` as physical payload metadata and proposes server-level placement as the preferred direction.

## Design and examples

### ODCS document encoding

The ODCS metadata document itself SHOULD be encoded as UTF-8.

This is separate from the encoding of the physical data described by the contract. For example, an ODCS YAML or JSON document can be UTF-8 while describing a legacy CSV file encoded as `ISO-8859-1`.

### Proposed field

Add an optional `encoding` field to server definitions that describe serialized data payloads and already carry physical representation details such as `format` or `delimiter`.

| Field | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| `encoding` | string | No | Expected character encoding for data payloads exposed through this server. Examples: `UTF-8`, `ISO-8859-1`, `ASCII`, `UTF-16`. |

The field is intentionally free-form rather than an enum. This keeps the standard flexible for platform-specific or less common encodings while still documenting common values in examples.

### Candidate server definitions

The field is most relevant for servers where ODCS already describes physical serialization concerns, for example:

- Amazon S3 and S3-compatible servers
- Azure Blob Storage / ADLS servers
- local filesystem servers
- Kafka or event/message servers where message format is declared
- other file, topic, or record-oriented servers that expose `format` and/or `delimiter`

The implementation should avoid adding `encoding` to server types where character encoding is not meaningful or where the storage engine abstracts it away.

### Example 1: UTF-8 CSV on S3

```yaml
apiVersion: v3.1.0
kind: DataContract
name: customer_export
servers:
  production:
    type: s3
    location: s3://my-bucket/customer_export/*.csv
    format: csv
    delimiter: ","
    encoding: UTF-8
schema:
  - name: customers
    physicalType: table
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
servers:
  production:
    type: local
    location: /data/legacy/orders_*.csv
    format: csv
    delimiter: ";"
    encoding: ISO-8859-1
schema:
  - name: orders
    physicalType: table
    properties:
      - name: order_id
        logicalType: string
        required: true
      - name: description
        logicalType: string
      - name: amount
        logicalType: number
```

### Example 3: Same logical contract, different server encodings

```yaml
apiVersion: v3.1.0
kind: DataContract
name: customer_export
servers:
  modern:
    type: s3
    location: s3://modern-bucket/customers/*.csv
    format: csv
    encoding: UTF-8
  legacy:
    type: s3
    location: s3://legacy-bucket/customers/*.csv
    format: csv
    encoding: ISO-8859-1
schema:
  - name: customers
    physicalType: table
    properties:
      - name: customer_id
        logicalType: string
      - name: customer_name
        logicalType: string
```

This example demonstrates why server-level placement can be more precise than a top-level or schema-object-level field: the logical schema is the same, while the physical representations differ.

### JSON Schema impact

For each relevant server definition, add an optional property similar to:

```json
"encoding": {
  "type": "string",
  "description": "The expected character encoding for data payloads exposed through this server, for example UTF-8, ISO-8859-1, ASCII or UTF-16.",
  "examples": ["UTF-8", "ISO-8859-1", "ASCII", "UTF-16"]
}
```

Documentation should list `encoding` alongside physical representation fields such as `format` and `delimiter` for the applicable server types.

## Alternatives

### Top-level contract field

Rejected for this RFC. A top-level `encoding` field is simple, but it assumes a single encoding for the whole contract. ODCS contracts can describe several physical locations or servers, and different payloads may have different encodings. It also mixes the encoding of the ODCS metadata document itself with the encoding of the data being described.

### Schema object / element field

Considered in the initial discussion and in the first RFC draft. This location is closer to the logical dataset or object than the top level and may be useful when a schema object directly represents a dataset.

However, community feedback pointed out that encoding behaves like physical access metadata, similar to file `format` and `delimiter`. Since these are already modeled on server definitions, server-level placement better supports cases where the same logical schema is exposed through multiple physical servers with different encodings.

### Property-level field

Rejected for this RFC. A field on every scalar property would be more granular, but it is likely unnecessary for the common case and would increase schema complexity. Encoding usually describes the serialized payload, file, topic, or record representation rather than individual logical fields.

### `customProperties`

Rejected as the standard approach. `customProperties` can already represent encoding metadata, but values would not be standardized. Tooling could not reliably depend on a common key or semantics.

### Enum of allowed encodings

Rejected for now. An enum would improve validation but risks excluding valid platform-specific encodings. This RFC proposes a free-form string and documents common examples.

## Decision

TBD by TSC.

## Consequences

### Positive

- Makes data payload encoding machine-readable in ODCS.
- Separates ODCS document encoding from physical data encoding.
- Aligns encoding with existing physical representation fields such as `format` and `delimiter`.
- Supports multiple servers with different encodings in the same contract.
- Remains backward compatible because the field is optional.

### Trade-offs

- Free-form values may vary in spelling or casing (`UTF-8` vs `utf8`).
- Tooling may need normalization if it wants strict behavior.
- Implementers need to decide which server definitions should expose `encoding`.
- Server-level placement may be less convenient for contracts with a single schema object and no explicit physical server details.

## References

- Issue: https://github.com/bitol-io/open-data-contract-standard/issues/263
- Initial implementation PR: https://github.com/bitol-io/open-data-contract-standard/pull/269
- RFC PR: https://github.com/bitol-io/tsc/pull/59
- RFC process: https://github.com/bitol-io/tsc/tree/main/rfcs
- Infrastructure server fields: https://bitol-io.github.io/open-data-contract-standard/latest/infrastructure-servers/
