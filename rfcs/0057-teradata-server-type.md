# Teradata server type

Champion: Simon Harrer

Contributors:
* Martin Hillebrand

Applies to:
* [x] ODCS - Open Data Contract Standard
* [ ] ODPS - Open Data Product Standard
* [ ] OORS - Open Observability Results Standard
* [ ] OOCS - Open Orchestration and Control Standard
* [ ] OMMS - Open Maturity Model Standard
* [ ] OMDS - Open Metadata Difference Standard

## Summary

Add `teradata` as a server `type` to describe data served from Teradata Vantage.

## Motivation

Teradata Vantage is a widely used enterprise data warehouse, but ODCS has no server type for it. Users fall back to `custom`, losing structured, validatable connection metadata.

## Design and examples

| Field      | Type   | Required | Description                        |
| ---------- | ------ | -------- | ---------------------------------- |
| `type`     | string | Yes      | `teradata`                         |
| `host`     | string | Yes      | Host of the Teradata server.       |
| `port`     | number | No       | Port, defaults to `1025`.          |
| `database` | string | No       | Name of the database.              |

No `schema` field: in Teradata, the database is the namespace.

```yaml
servers:
  - server: prod
    type: teradata
    host: teradata.acme.com
    port: 1025
    database: SALES
```

## Alternatives

Keep using `custom`. Rejected: hides connection details in unstructured fields.

## Decision

> The decision made by the TSC.

## Consequences

- nonbreaking change: adds a new optional server type.

## References

- [Teradata Vantage documentation](https://docs.teradata.com/)
