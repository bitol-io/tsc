# HANA server type

Champion: Simon Harrer

Applies to:
* [x] ODCS - Open Data Contract Standard
* [ ] ODPS - Open Data Product Standard
* [ ] OORS - Open Observability Results Standard
* [ ] OOCS - Open Orchestration and Control Standard
* [ ] OMMS - Open Maturity Model Standard
* [ ] OMDS - Open Metadata Difference Standard

## Summary

Add `hana` as a server `type` to describe data served from SAP HANA.

## Motivation

SAP HANA is widely used as an enterprise data platform, but ODCS has no dedicated server type for it. Today users fall back to the `custom` server type, losing structured, validatable connection metadata. A first-class `hana` type lets tooling connect, document, and validate HANA-backed contracts consistently.

## Design and examples

Add `hana` to the list of allowed server `type` values, with the following fields:

| Field      | Type   | Required | Description                          |
| ---------- | ------ | -------- | ------------------------------------ |
| `type`     | string | Yes      | `hana`                               |
| `host`     | string | Yes      | Host of the HANA server.             |
| `port`     | number | No       | Port of the HANA server.             |
| `database` | string | No       | Name of the database (tenant).       |
| `schema`   | string | No       | Name of the schema.                  |

### Example 1: Minimal

```yaml
servers:
  - server: prod
    type: hana
    host: hana.acme.com
```

### Example 2: Structured

```yaml
servers:
  - server: prod
    type: hana
    host: hana.acme.com
    port: 30015
    database: HXE
    schema: SALES
```

## Alternatives

Keep using the `custom` server type. Rejected: it hides connection details in unstructured fields, preventing structured validation and tooling support.

## Decision

> The decision made by the TSC.

## Consequences

- nonbreaking change: adds a new optional server type.

## References

- [SAP HANA SQL connection properties](https://help.sap.com/docs/SAP_HANA_PLATFORM)
- Existing server types in ODCS (e.g. `postgresql`, `snowflake`).
