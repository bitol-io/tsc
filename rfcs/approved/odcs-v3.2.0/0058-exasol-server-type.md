# Exasol server type

Champion: Jochen Christ

Applies to:
* [x] ODCS - Open Data Contract Standard
* [ ] ODPS - Open Data Product Standard
* [ ] OORS - Open Observability Results Standard
* [ ] OOCS - Open Orchestration and Control Standard
* [ ] OMMS - Open Maturity Model Standard
* [ ] OMDS - Open Metadata Difference Standard

## Summary

Add `exasol` as a server `type` to describe data served from Exasol.

## Motivation

Exasol is an in-memory MPP analytics database used as an enterprise data warehouse, but ODCS has no server type for it. Users fall back to `custom`, losing structured, validatable connection metadata.

## Design and examples

| Field    | Type   | Required | Description                                                                          |
| -------- | ------ | -------- | ------------------------------------------------------------------------------------ |
| `type`   | string | Yes      | `exasol`                                                                             |
| `host`   | string | Yes      | Host of the Exasol server. May be a cluster connection range, e.g. `n11..14.acme.com`. |
| `port`   | number | No       | Port, defaults to `8563`.                                                            |
| `schema` | string | No       | Name of the schema.                                                                  |

No `database` field: an Exasol cluster runs a single database, and the schema is the namespace.

```yaml
servers:
  - server: prod
    type: exasol
    host: exasol.acme.com
    port: 8563
    schema: SALES
```

## Alternatives

Keep using `custom`. Rejected: hides connection details in unstructured fields.

## Decision

**Approved by the TSC on 2026-08-05.**

## Consequences

- nonbreaking change: adds a new optional server type.

## References

- [Exasol documentation](https://docs.exasol.com/)
