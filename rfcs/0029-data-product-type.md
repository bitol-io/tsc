# Data Product Type Field

Champion: Simon Harrer

## Summary

Add an optional `type` string field at the top level of ODPS to categorize data products by their architectural alignment. Common values include `source-aligned`, `aggregate`, and `consumer-aligned`.

## Motivation

Enable data catalog filtering and help consumers find appropriate data products based on architectural type.

## Design and examples

Add optional `type` string field at the top level of ODPS:

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `type` | string | No | null | Architectural type of the data product. Common values: `source-aligned`, `aggregate`, `consumer-aligned`. Organizations may define custom types. |

### Example

```yaml
apiVersion: v1.0.0
kind: DataProduct
name: Customer Master Data
id: fbe8d147-28db-4f1d-bedf-a3fe9f458427
type: source-aligned
domain: customer
status: active

outputPorts:
  - name: customer-master
    version: 1.0.0
    contractId: c2798941-1b7e-4b03-9e0d-955b1a872b32
```

## Alternatives

Use an enum with fixed values. Rejected because organizations may have different taxonomies.

## Decision

> The decision made by the TSC.

## Consequences

Non-breaking change.

## References

- [Implementing Data Mesh](https://www.oreilly.com/library/view/implementing-data-mesh/9781098130190/) - Perrin & Broda
