# Data Product Type Field

Champion: Simon Harrer.

Authors: Simon Harrer.

Slack: TBD.

GitHub issue: TBD.

Applies to:
* [ ] ODCS - Open Data Contract Standard
* [x] ODPS - Open Data Product Standard
* [ ] OORS - Open Observability Results Standard
* [ ] OOCS - Open Orchestration and Control Standard
* [ ] OMMS - Open Maturity Model Standard
* [ ] OMDS - Open Metadata Difference Standard

## Summary

Add an optional `type` string field at the top level of ODPS to categorize data products by their architectural alignment. Common values include `sourceAligned`, `aggregate`, and `consumerAligned`.

## Motivation

Enable data catalog filtering and help consumers find appropriate data products based on architectural type.

## Design and examples

Add optional `type` string field at the top level of ODPS:

| Field  | Type   | Required | Default | Description                                                                                                                                    |
| ------ | ------ | -------- | ------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `type` | string | No       | null    | Architectural type of the data product. Common values: `sourceAligned`, `aggregate`, `consumerAligned`. Organizations may define custom types. |

### Example

```yaml
apiVersion: v1.0.0
kind: DataProduct
name: Customer Master Data
id: fbe8d147-28db-4f1d-bedf-a3fe9f458427
type: sourceAligned
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

## Appendix: Changelog

All notable changes to this RFC are recorded here. Dates are `YYYY-MM-DD`. Entries are listed newest-first.

| Date       | Author              | Change                                                                                                                                                                                                                                                                                                                                                                   |
| ---------- | ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 2026-04-17 | Jean-Georges Perrin | Added this Changelog appendix.                                                                                                                                                                                                                                                                                                                                           |
|            |                     | Corrected the "Applies to" section: this RFC targets ODPS (not ODCS). The design text and example already reference `kind: DataProduct` and the top level of ODPS; the checkbox had been set to ODCS.                                                                                                                                                                    |
|            |                     | Renamed example values from kebab-case to camelCase: `source-aligned` → `sourceAligned`, `consumer-aligned` → `consumerAligned` (`aggregate` unchanged). Rationale: the published ODCS/ODPS standard uses camelCase for all enum-like values; kebab-case is used only for user-chosen identifiers. Updated the field description table and the YAML example accordingly. |
| —          | Simon Harrer        | Initial draft: optional top-level `type` field for ODPS with suggested values `source-aligned`, `aggregate`, `consumer-aligned`.                                                                                                                                                                                                                                         |
