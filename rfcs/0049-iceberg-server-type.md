# Apache Iceberg server type

Champion: Patrick Beitsma

Authors: Sander Bylemans

Slack: https://app.slack.com/client/T01MWBFKW7K/C0BDZRK06GH

Applies to:
* [x] ODCS - Open Data Contract Standard
* [ ] ODPS - Open Data Product Standard
* [ ] OORS - Open Observability Results Standard
* [ ] OOCS - Open Orchestration and Control Standard
* [ ] OMMS - Open Maturity Model Standard
* [ ] OMDS - Open Metadata Difference Standard

## Summary

Add `iceberg` as a server `type` to describe data served from Apache Iceberg tables.

## Motivation

Apache Iceberg is a widely adopted open table format for large analytic datasets, used on top of object storage (S3, GCS, ADLS) and accessible through query engines such as Spark, Trino, Flink, and Dremio. ODCS currently has no dedicated server type for it. Users today fall back to the `custom` server type, losing structured, validatable connection metadata. A first-class `iceberg` type lets tooling connect, document, and validate Iceberg-backed contracts consistently and enables catalog-aware integrations.

## Design and examples

Add `iceberg` to the list of allowed server `type` values, with the following fields:

| Field            | Type   | Required | Description                                                                                                     |
| ---------------- | ------ | -------- | --------------------------------------------------------------------------------------------------------------- |
| `type`           | string | Yes      | `iceberg`                                                                                                       |
| `catalog`        | string | Yes      | Catalog name as registered in the query engine or catalog service (e.g. `my_catalog`).                          |
| `catalogType`    | string | Yes      | Catalog implementation type. Common values: `rest`, `hive`, `glue`, `jdbc`, `nessie`, `hadoop`.                |
| `catalogUri`     | string | No       | URI of the catalog service (e.g. REST catalog endpoint or Hive metastore thrift URI). Required when `catalogType` is `rest`, `hive`, `nessie`, or `jdbc`; not applicable for `glue` or `hadoop`. |
| `region`         | string | No       | AWS region of the Glue catalog (e.g. `us-east-1`). Applies to `glue` only.                                     |
| `account`        | string | No       | AWS account ID hosting the Glue catalog. Applies to `glue` only; useful for cross-account catalogs.            |
| `ref`            | string | No       | Nessie branch, tag, or commit hash to read from (e.g. `main`). Applies to `nessie` only.                       |
| `namespace`      | string | No       | Dot-separated namespace path within the catalog (e.g. `db.schema` or just `db`).                               |
| `location`       | string | No       | Base storage location of the warehouse or namespace (e.g. `s3://my-bucket/warehouse/`).                        |

### Example 1: Minimal

```yaml
servers:
  - server: prod
    type: iceberg
    catalog: prod_catalog
    catalogType: rest
    catalogUri: https://catalog.acme.com
```

### Example 2: Structured — REST catalog on S3

```yaml
servers:
  - server: prod
    type: iceberg
    catalog: prod_catalog
    catalogType: rest
    catalogUri: https://catalog.acme.com
    namespace: acme_db.sales
    location: s3://acme-warehouse/sales/
```

### Example 3: AWS Glue catalog

```yaml
servers:
  - server: prod
    type: iceberg
    catalog: glue
    catalogType: glue
    region: us-east-1
    account: "123456789012"
    namespace: sales
    location: s3://acme-warehouse/sales/
```

### Example 4: Nessie catalog

```yaml
servers:
  - server: prod
    type: iceberg
    catalog: nessie
    catalogType: nessie
    catalogUri: https://nessie.acme.com/api/v1
    ref: main
    namespace: acme_db.sales
    location: s3://acme-warehouse/sales/
```

## Alternatives

Keep using the `custom` server type. Rejected: it hides connection details in unstructured fields, preventing structured validation and tooling support.

Use the existing `s3` server type. Rejected: S3 describes raw object storage, not the Iceberg table format layer. An Iceberg table carries catalog metadata and schema evolution semantics beyond a plain S3 path.

## Decision

> The decision made by the TSC.

## Consequences

- Nonbreaking change: adds a new optional server type.
- Tooling that parses the `servers` list can now detect Iceberg-backed contracts and generate catalog-specific connection configuration automatically.

## References

- [Apache Iceberg specification](https://iceberg.apache.org/spec/)
- [Iceberg catalog documentation](https://iceberg.apache.org/concepts/catalog/)
- Existing server types in ODCS (e.g. `bigquery`, `s3`, `snowflake`).
- RFC 0045 (HANA server type) as a structural template.
