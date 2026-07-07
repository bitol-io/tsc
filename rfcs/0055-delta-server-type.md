# Delta Lake server type

Champion: Patrick Beitsma

Authors: Jean-Georges Perrin, Patrick Beitsma

Slack: https://app.slack.com/client/T01MWBFKW7K/C0BDZRK06GH

Applies to:
* [x] ODCS - Open Data Contract Standard
* [ ] ODPS - Open Data Product Standard
* [ ] OORS - Open Observability Results Standard
* [ ] OOCS - Open Orchestration and Control Standard
* [ ] OMMS - Open Maturity Model Standard
* [ ] OMDS - Open Metadata Difference Standard

## Summary

Add `delta` as a server `type` to describe data served from Delta Lake tables.

## Motivation

Delta Lake is a widely adopted open table format for large analytic datasets, used on top of object storage (S3, ADLS, GCS) and accessible through query engines such as Spark, Databricks, Trino, Flink, and DuckDB. ODCS currently has no dedicated server type for it. Today `delta` exists only as a `format` value under the `s3`, `azure`, and `sftp` server types, and users otherwise fall back to the `databricks` or `custom` server types, losing structured, validatable connection metadata for platform-independent Delta tables. A first-class `delta` type lets tooling connect, document, and validate Delta-backed contracts consistently — including catalog-managed and path-based tables — and enables time-travel-aware integrations.

## Design and examples

Add `delta` to the list of allowed server `type` values, with the following fields:

| Field         | Type    | Required | Description                                                                                                                                              |
| ------------- | ------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `type`        | string  | Yes      | `delta`                                                                                                                                                  |
| `location`    | string  | No       | Storage location of the Delta table directory containing the `_delta_log` folder (e.g. `s3://my-bucket/warehouse/sales/`). Required for path-based tables; optional when the table is resolved through a catalog. |
| `catalog`     | string  | No       | Catalog name as registered in the query engine or catalog service (e.g. Unity Catalog name). Applies to catalog-managed tables.                         |
| `catalogType` | string  | No       | Catalog implementation type. Common values: `unity`, `hive`, `glue`, `hadoop`. Omit for purely path-based tables.                                       |
| `namespace`   | string  | No       | Dot-separated namespace/schema path within the catalog (e.g. `db.schema` or just `db`). Applies to catalog-managed tables.                              |
| `table`       | string  | No       | Table name within the namespace. Applies to catalog-managed tables.                                                                                     |
| `version`     | integer | No       | Delta table version to read for time travel (e.g. `12`). Mutually exclusive with `timestamp`.                                                           |
| `timestamp`   | string  | No       | Point-in-time (ISO 8601) to read for time travel (e.g. `2024-01-01T00:00:00Z`). Mutually exclusive with `version`.                                      |

## Examples

### Example 1: Minimal — path-based table

```yaml
servers:
  - server: prod
    type: delta
    location: s3://acme-warehouse/sales/
```

### Example 2: Catalog-managed — Unity Catalog

```yaml
servers:
  - server: prod
    type: delta
    catalog: acme_catalog
    catalogType: unity
    namespace: acme_db.sales
    table: orders
```

### Example 3: AWS Glue catalog on S3

```yaml
servers:
  - server: prod
    type: delta
    catalog: glue
    catalogType: glue
    namespace: sales
    table: orders
    location: s3://acme-warehouse/sales/
```

### Example 4: Path-based table with time travel

```yaml
servers:
  - server: prod
    type: delta
    location: abfss://warehouse@acme.dfs.core.windows.net/sales/
    version: 42
```

## Proposed schema changes

The following changes to the ODCS JSON Schema are proposed as part of this RFC. They are **not** yet applied to the standard; they will be applied to the target schema(s) if and when the TSC approves this RFC.

1. Add `delta` to the server `type` enum:

```json
"enum": [
  "api", "athena", "azure", "bigquery", "clickhouse", "databricks", "delta", "denodo", "dremio",
  ...
]
```

2. Add a conditional dispatch in the server `allOf` list:

```json
{
  "if": {
    "properties": { "type": { "const": "delta" } },
    "required": ["type"]
  },
  "then": {
    "$ref": "#/$defs/ServerSource/DeltaServer"
  }
}
```

3. Add the `DeltaServer` definition under `$defs/ServerSource`:

```json
"DeltaServer": {
  "type": "object",
  "title": "DeltaServer",
  "properties": {
    "location": {
      "type": "string",
      "format": "uri",
      "description": "Storage location of the Delta table directory containing the _delta_log folder. Required for path-based tables; optional when the table is resolved through a catalog.",
      "examples": ["s3://my-bucket/warehouse/sales/"]
    },
    "catalog": {
      "type": "string",
      "description": "Catalog name as registered in the query engine or catalog service (e.g. Unity Catalog name). Applies to catalog-managed tables."
    },
    "catalogType": {
      "type": "string",
      "description": "Catalog implementation type. Omit for purely path-based tables.",
      "examples": ["unity", "hive", "glue", "hadoop"]
    },
    "namespace": {
      "type": "string",
      "description": "Dot-separated namespace/schema path within the catalog (e.g. db.schema or just db). Applies to catalog-managed tables."
    },
    "table": {
      "type": "string",
      "description": "Table name within the namespace. Applies to catalog-managed tables."
    },
    "version": {
      "type": "integer",
      "description": "Delta table version to read for time travel. Mutually exclusive with timestamp."
    },
    "timestamp": {
      "type": "string",
      "description": "Point-in-time (ISO 8601) to read for time travel. Mutually exclusive with version.",
      "examples": ["2024-01-01T00:00:00Z"]
    }
  }
}
```

## Alternatives

Keep using `delta` as a `format` under the `s3`, `azure`, or `sftp` server types. Rejected: it treats the Delta table as raw object storage, hiding the table-format layer (transaction log, schema evolution, time travel) and preventing catalog-managed tables from being expressed.

Keep using the `databricks` server type. Rejected: Delta Lake is an open, platform-independent format that also runs outside Databricks (Spark, Trino, Flink, DuckDB, standalone readers); tying it to a single vendor server type misrepresents these deployments.

Keep using the `custom` server type. Rejected: it hides connection details in unstructured fields, preventing structured validation and tooling support.

## Decision

> The decision made by the TSC.

## Consequences

- Nonbreaking change: adds a new optional server type.
- Tooling that parses the `servers` list can now detect Delta-backed contracts and generate the appropriate path-based or catalog-specific connection configuration automatically, including time-travel reads.

## References

- [Delta Lake documentation](https://docs.delta.io/latest/index.html)
- [Delta Lake protocol specification](https://github.com/delta-io/delta/blob/master/PROTOCOL.md)
- Existing server types in ODCS (e.g. `databricks`, `s3`, `snowflake`).
- RFC 0049 (Iceberg server type) as a structural template.
