# Apache Iceberg server type

Champion: Patrick Beitsma

Authors: Sander Bylemans, Patrick Beitsma, Simon Harrer, Diego C. and Jean-Georges Perrin.

Slack: https://app.slack.com/client/T01MWBFKW7K/C0BDZRK06GH

Applies to:
* [x] ODCS - Open Data Contract Standard
* [ ] ODPS - Open Data Product Standard
* [ ] OORS - Open Observability Results Standard
* [ ] OOCS - Open Orchestration and Control Standard
* [ ] OMMS - Open Maturity Model Standard
* [ ] OMDS - Open Metadata Difference Standard

## Summary

Add `iceberg` as a server `type` to describe access to Apache Iceberg catalogs (and Iceberg-managed tables) through the standardized Iceberg REST API.

## Motivation

Apache Iceberg is a widely adopted open table format and catalog specification for large analytic datasets, used on top of object storage (S3, GCS, ADLS) and accessible through query engines such as Spark, Trino, Flink, and Dremio. ODCS currently has no dedicated server type for it. Users today fall back to the `custom` server type, losing structured, validatable connection metadata. A first-class `iceberg` type lets tooling connect, document, and validate Iceberg-backed contracts consistently and enables catalog-aware integrations.

## Design and examples

Add `iceberg` to the list of allowed server `type` values, with the following fields:

| Field            | Type   | Required | Description                                                                                                     |
| ---------------- | ------ | -------- | --------------------------------------------------------------------------------------------------------------- |
| `type`           | string | Yes      | `iceberg`                                                                                                      |
| `catalog`        | string | Yes      | Catalog name as registered in the query engine or catalog service (e.g. `my_catalog`).                          |
| `catalogUrl`     | string | Yes      | URL of the Iceberg compatible REST catalog service (Polaris, S3 Tables, Nessie, Unity Catalog, Glue, etc.)      |
| `namespace`      | string | No       | Dot-separated namespace path within the catalog (e.g. `db.schema` or just `db`).                                |
| `warehouse`      | string | No       | Base storage location of the warehouse (e.g. `s3://my-bucket/warehouse/`).                                      |

### Example 1: Minimal

```yaml
servers:
  - server: prod
    type: iceberg
    catalog: prod_catalog
    catalogUrl: https://catalog.acme.com
```

### Example 2: Structured — REST catalog on S3 Tables

```yaml
servers:
  - server: prod
    type: iceberg
    catalog: prod_catalog
    catalogUrl: https://catalog.acme.com
    namespace: acme_db.sales
    warehouse: s3://acme-warehouse/sales/
```

### Example 3: AWS Glue catalog (supported by the Glue Server type, but included here for completeness)

```yaml
servers:
  - server: prod
    type: glue
    account: "123456789012"
    namespace: sales
    location: s3://acme-warehouse/sales/
```

When using Glue to expose an Iceberg REST API compatible catalog, the `iceberg` type should be used.

### Example 4: Nessie catalog

```yaml
servers:
  - server: prod
    type: nessie
    catalogUrl: https://nessie.acme.com/api/v1
    ref: main
    namespace: acme_db.sales
    warehouse: s3://acme-warehouse/sales/
```

Above is just an example and is not supported as no 'nessie' server type exists yet. If Nessie is used as an Iceberg compatible REST catalog, the `iceberg` type should be used.

## Proposed schema changes

The following changes to the ODCS JSON Schema are proposed as part of this RFC. They are **not** yet applied to the standard; they will be applied to the target schema(s) if and when the TSC approves this RFC.

1. Add `iceberg` to the server `type` enum:

```json
"enum": [
  "api", "athena", "azure", "bigquery", "clickhouse", "databricks", "denodo", "dremio",
  "duckdb", "glue", "cloudsql", "db2", "hive", "iceberg", "impala", "informix", "kafka", "kinesis", "local",
  "mysql", "oracle", "postgresql", "postgres", "presto", "pubsub",
  "redshift", "s3", "sftp", "snowflake", "sqlserver", "synapse", "trino", "vertica", "zen", "custom"
]
```

2. Add a conditional dispatch in the server `allOf` list:

```json
{
  "if": {
    "properties": { "type": { "const": "iceberg" } },
    "required": ["type"]
  },
  "then": {
    "$ref": "#/$defs/ServerSource/IcebergServer"
  }
}
```

3. Add the `IcebergServer` definition under `$defs/ServerSource`:

```json
"IcebergServer": {
  "type": "object",
  "title": "IcebergServer",
  "properties": {
    "catalog": {
      "type": "string",
      "description": "Catalog name as registered in the query engine or catalog service.",
      "examples": ["my_catalog"]
    },
    "catalogUrl": {
      "type": "string",
      "format": "uri",
      "description": "URL of the Iceberg compatible REST catalog service (Polaris, S3 Tables, Nessie, Unity Catalog, Glue, etc.)",
      "examples": ["https://my-catalog-service.com"]
    },
    "namespace": {
      "type": "string",
      "description": "Dot-separated namespace/schema path within the catalog (e.g. db.schema or just db)."
    },
    "warehouse": {
      "type": "string",
      "description": "Logical catalog or base location for the catalog to use.",
      "examples": ["s3://my-bucket/warehouse/sales/", "production", "development"]
    }
  }
}
```

## Alternatives

Keep using the `custom` server type. Rejected: it hides connection details in unstructured fields, preventing structured validation and tooling support.

Use the existing `s3` server type. Rejected: S3 describes raw object storage, not the Iceberg table format layer. An Iceberg table carries catalog metadata and schema evolution semantics beyond a plain S3 path.

Add multiple new server types for each Iceberg catalog implementation (e.g. `glue`, `nessie`, `polaris`). Rejected: as the Iceberg REST API is a stable, standardized interface that can be used across multiple catalogs, a single `iceberg` type is sufficient to support multiple catalog implementations. Those catalogs that do not support the Iceberg REST API (e.g. Hadoop, Nessies old API) can be supported by using the `custom` type or a separate server type if needed.

## Decision

**Approved by the TSC on 2026-07-14.**

The `iceberg` server type requires `catalog` and `catalogUrl`; `namespace` and `warehouse` are optional.

## Consequences

- Nonbreaking change: adds a new optional server type.
- Tooling that parses the `servers` list can now detect Iceberg-backed contracts and generate catalog-specific connection configuration automatically.

## References

- [Apache Iceberg specification](https://iceberg.apache.org/spec/)
- [Iceberg catalog documentation](https://iceberg.apache.org/concepts/catalog/)
- Existing server types in ODCS (e.g. `bigquery`, `s3`, `snowflake`).
- RFC 0045 (HANA server type) as a structural template.
