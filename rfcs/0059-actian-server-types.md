# Actian Ingres and Actian Analytics Engine server types

Champion: Jean-Georges Perrin

Authors: Jean-Georges Perrin

GitHub issue: https://github.com/bitol-io/tsc/issues/93

Applies to:
* [x] ODCS - Open Data Contract Standard
* [ ] ODPS - Open Data Product Standard
* [ ] OORS - Open Observability Results Standard
* [ ] OOCS - Open Orchestration and Control Standard
* [ ] OMMS - Open Maturity Model Standard
* [ ] OMDS - Open Metadata Difference Standard
* [ ] OSDS - Open Semantic Definition Standard

## Summary

Add two server `type` values to ODCS: `ingres`, to describe data served from Actian Ingres, and `analytics`, to describe data served from the Actian Analytics Engine (formerly Actian Vector / Vectorwise).

## Motivation

Actian Ingres is a commercially supported, enterprise-grade relational database management system (RDBMS) designed for mission-critical transactional processing (OLTP) and hybrid enterprise workloads. The Actian Analytics Engine is an enterprise-grade, high-performance, columnar analytical database management system (DBMS) designed for high-speed SQL queries and big data processing.

Both are in production use as systems of record and as analytical serving layers, and both are the physical location of data that data contracts describe. ODCS has no server type for either, so users fall back to `custom`, which hides connection details in unstructured fields and prevents structured validation and tooling support (connection generation, catalog crawling, contract linting).

The two engines are separate products with separate lifecycles, but they share the Ingres client/server plumbing — most notably the Data Access Server (DAS) and its default port `21064` — which is why they are proposed together in a single RFC. They are still two distinct `type` values: a contract consumer needs to know whether it is talking to an OLTP row store or a columnar analytical engine, and collapsing them into one type would lose that.

This aligns with the guiding value of describing the real world as practitioners find it, and follows the precedent already set for other commercial engines (`hana` in RFC 0045, `teradata` in RFC 0057, `exasol` in RFC 0058).

## Design and examples

### `ingres`

| Field      | Type    | Required | Description                                                                                                        |
| ---------- | ------- | -------- | ------------------------------------------------------------------------------------------------------------------ |
| `type`     | string  | Yes      | `ingres`                                                                                                            |
| `database` | string  | Yes      | Database name to connect to on the Ingres instance.                                                                 |
| `host`     | string  | Yes      | Hostname or IP address of the Ingres server.                                                                        |
| `port`     | integer | No       | Connection port for the Ingres Data Access Server (DAS) / SQL connections. Defaults to `21064` (installation ID `II7`). |

### `analytics`

| Field      | Type    | Required | Description                                                                                     |
| ---------- | ------- | -------- | ----------------------------------------------------------------------------------------------- |
| `type`     | string  | Yes      | `analytics`                                                                                      |
| `database` | string  | Yes      | Database name to connect to on the Analytics Engine server.                                      |
| `host`     | string  | Yes      | Hostname or IP address of the Analytics Engine server.                                           |
| `port`     | integer | No       | Connection port for the Data Access Server (DAS) / SQL connections. Defaults to `21064`.          |

Notes applying to both types:

- The port is derived from the Ingres installation identifier: the default installation ID `II` listens on `21064`, and other installation IDs on the same host listen on different ports. `port` is therefore optional but is strongly recommended whenever more than one installation runs on a host.
- No `schema` field is proposed. In both engines the namespace inside a database is the table owner, which is normally resolved from the connecting user rather than configured on the connection. If the TSC prefers symmetry with the other SQL server types, an optional `schema` field can be added; this is called out as an open question below.

### Examples

#### Example 1: Minimal — Ingres

```yaml
servers:
  - server: prod
    type: ingres
    host: ingres.acme.com
    database: sales
```

#### Example 2: Ingres with an explicit port and roles

```yaml
servers:
  - server: prod
    type: ingres
    description: Order-entry OLTP system of record
    environment: prod
    host: ingres01.acme.com
    port: 21064
    database: orders
    roles:
      - role: acme_orders_ro
        access: read
      - role: acme_orders_rw
        access: write
```

#### Example 3: Minimal — Analytics Engine

```yaml
servers:
  - server: prod
    type: analytics
    host: analytics.acme.com
    database: warehouse
```

#### Example 4: Analytics Engine, non-default installation

```yaml
servers:
  - server: preprod
    type: analytics
    description: Columnar serving layer for the sales data product
    environment: preprod
    host: vector02.acme.com
    port: 21164
    database: sales_mart
```

## Proposed schema changes

The following changes to the ODCS JSON Schema are proposed as part of this RFC. They are **not** yet applied to the standard; they will be applied to the target schema(s) if and when the TSC approves this RFC.

1. Add `analytics` and `ingres` to the server `type` enum:

```json
"enum": [
  "analytics", "api", "athena", "azure", "bigquery", "clickhouse", "databricks", "denodo", "dremio",
  "duckdb", "exasol", "glue", "hana", "cloudsql", "db2", "hive", "iceberg", "impala", "informix",
  "ingres", "kafka", "kinesis", "local",
  ...
]
```

2. Add the conditional dispatches in the server `allOf` list:

```json
{
  "if": {
    "properties": { "type": { "const": "ingres" } },
    "required": ["type"]
  },
  "then": {
    "$ref": "#/$defs/ServerSource/IngresServer"
  }
},
{
  "if": {
    "properties": { "type": { "const": "analytics" } },
    "required": ["type"]
  },
  "then": {
    "$ref": "#/$defs/ServerSource/AnalyticsServer"
  }
}
```

3. Add the `IngresServer` and `AnalyticsServer` definitions under `$defs/ServerSource`:

```json
"IngresServer": {
  "type": "object",
  "title": "IngresServer",
  "properties": {
    "database": {
      "type": "string",
      "description": "Database name to connect to on the Ingres instance.",
      "examples": ["sales"]
    },
    "host": {
      "type": "string",
      "description": "Hostname or IP address of the Ingres server.",
      "examples": ["ingres.acme.com"]
    },
    "port": {
      "$ref": "#/$defs/Port",
      "description": "Connection port for the Ingres Data Access Server (DAS) / SQL connections.",
      "default": 21064,
      "examples": [21064]
    }
  },
  "required": ["database", "host"]
},
"AnalyticsServer": {
  "type": "object",
  "title": "AnalyticsServer",
  "properties": {
    "database": {
      "type": "string",
      "description": "Database name to connect to on the Analytics Engine server.",
      "examples": ["warehouse"]
    },
    "host": {
      "type": "string",
      "description": "Hostname or IP address of the Analytics Engine server.",
      "examples": ["analytics.acme.com"]
    },
    "port": {
      "$ref": "#/$defs/Port",
      "description": "Connection port for the Data Access Server (DAS) / SQL connections.",
      "default": 21064,
      "examples": [21064]
    }
  },
  "required": ["database", "host"]
}
```

## Open questions

1. **Is `analytics` the right value?** It is the shortest name that matches the product name (Actian Analytics Engine), but it is also a very generic word for an enum whose other entries name a concrete engine or service. Candidates, with the trade-offs:
   - `analytics` — matches the current product name; risk of being read as a generic category rather than a product, and of colliding with a future generic concept.
   - `vector` — the historical product name, still widely recognised; collides conceptually with the `vector` logical type introduced by RFC 0042, which is a different thing entirely.
   - `actian` — unambiguous vendor name, but Actian ships more than one engine (Ingres, Analytics Engine, Zen — and `zen` is already in the enum), so it is ambiguous in the other direction.

   This RFC proposes `analytics` as specified, and asks the TSC to confirm or substitute.
2. **Should an optional `schema` field be added to both types?** See the note above.
3. Should the two types ship as one RFC (as proposed) or be split into two, given the different products?

## Alternatives

Keep using `custom` for both. Rejected: hides connection details in unstructured fields, so nothing can be validated or generated from the contract.

Reuse the existing `zen` server type for either engine. Rejected: `zen` is Actian Zen, an embedded/edge database with a different connection model; it shares only the vendor.

Add a single `actian` server type with a discriminator field for the engine. Rejected: it pushes an engine choice that changes query semantics, type mapping, and workload profile into a secondary field, and it is inconsistent with how every other engine in the enum is modelled.

## Decision

> The decision made by the TSC.

## Consequences

- Nonbreaking change: adds two new optional server types.
- Tooling that parses the `servers` list can detect Ingres- and Analytics-Engine-backed contracts and generate the appropriate JDBC/ODBC connection strings, instead of treating them as opaque `custom` servers.
- The `analytics` value, once shipped, is part of the public enum and cannot be renamed without a breaking change — hence the naming question above should be settled before approval, not after.

## References

- [Actian Ingres](https://www.actian.com/databases/ingres/)
- [Actian Analytics Engine](https://www.actian.com/databases/analytics-engine/)
- RFC 0045 (HANA server type), RFC 0057 (Teradata server type), RFC 0058 (Exasol server type) — structural templates for a single-engine server type.
- Existing server types in ODCS (e.g. `zen`, `informix`, `db2`).
