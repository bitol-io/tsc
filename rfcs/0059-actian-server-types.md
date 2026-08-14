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

Add two server `type` values to ODCS: `ingres`, to describe data served from Actian Ingres, and `vectorwise`, to describe data served from the Actian Analytics Engine (created as VectorWise, later Actian Vector).

## Motivation

Actian Ingres is a commercially supported, enterprise-grade relational database management system (RDBMS) designed for mission-critical transactional processing (OLTP) and hybrid enterprise workloads. The Actian Analytics Engine is an enterprise-grade, high-performance, columnar analytical database management system (DBMS) designed for high-speed SQL queries and big data processing.

Both are in production use as systems of record and as analytical serving layers, and both are the physical location of data that data contracts describe. ODCS has no server type for either, so users fall back to `custom`, which hides connection details in unstructured fields and prevents structured validation and tooling support (connection generation, catalog crawling, contract linting).

The two engines are separate products with separate lifecycles, but they share the Ingres client/server plumbing — most notably the Data Access Server (DAS) and its default port `21064` — which is why they are proposed together in a single RFC. They are still two distinct `type` values: a contract consumer needs to know whether it is talking to an OLTP row store or a columnar analytical engine, and collapsing them into one type would lose that.

This aligns with the guiding value of describing the real world as practitioners find it, and follows the precedent already set for other commercial engines (`hana` in RFC 0045, `teradata` in RFC 0057, `exasol` in RFC 0058).

## Design and examples

### Naming convention for server `type` values

This RFC proposes, and applies, a convention for naming server `type` values:

> **A server `type` value uses the name the product carried when it was created, in lowercase.**

An enum value is a permanent API: once shipped it cannot be renamed without a breaking change. Product marketing names, by contrast, churn — often several times, and often as a result of acquisitions. The creation name is the one point in a product's branding history that never changes afterwards, so it is the only name that can be committed to an enum without betting on a vendor's future marketing.

The Analytics Engine is exactly this case. It was created as **VectorWise** (2010, out of the CWI X100 research line), was renamed **Actian Vector**, and is today the **Actian Analytics Engine** — three names for one engine, in a product whose connection model never changed. Pinning the enum to `vectorwise` costs nothing and survives the next rename.

Applied to this RFC: `ingres` (created as Ingres, at UC Berkeley) and `vectorwise` (created as VectorWise).

Most of the existing enum already follows this convention — `exasol`, `hana`, `vertica`, `informix`, `db2`, `snowflake`, `databricks`. Two entries do not, and are worth naming honestly rather than glossing: `zen` (Actian Zen was created as **Btrieve**, then Pervasive PSQL) and `synapse` (created as Azure SQL Data Warehouse, and since renamed again under Microsoft Fabric). Both predate this convention; this RFC does **not** propose renaming them, since that would be a breaking change for no functional gain. They are precisely the pattern the convention exists to stop repeating.

### `ingres`

| Field      | Type    | Required | Description                                                                                                             |
| ---------- | ------- | -------- | ----------------------------------------------------------------------------------------------------------------------- |
| `type`     | string  | Yes      | `ingres`                                                                                                                |
| `database` | string  | Yes      | Database name to connect to on the Ingres instance.                                                                     |
| `host`     | string  | Yes      | Hostname or IP address of the Ingres server.                                                                            |
| `port`     | integer | No       | Connection port for the Ingres Data Access Server (DAS) / SQL connections. Defaults to `21064` (installation ID `II7`). |

### `vectorwise`

| Field      | Type    | Required | Description                                                                              |
| ---------- | ------- | -------- | ---------------------------------------------------------------------------------------- |
| `type`     | string  | Yes      | `vectorwise`                                                                             |
| `database` | string  | Yes      | Database name to connect to on the Analytics Engine server.                              |
| `host`     | string  | Yes      | Hostname or IP address of the Analytics Engine server.                                   |
| `port`     | integer | No       | Connection port for the Data Access Server (DAS) / SQL connections. Defaults to `21064`. |

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
    type: vectorwise
    host: analytics.acme.com
    database: warehouse
```

#### Example 4: Analytics Engine, non-default installation

```yaml
servers:
  - server: preprod
    type: vectorwise
    description: Columnar serving layer for the sales data product
    environment: preprod
    host: vector02.acme.com
    port: 21164
    database: sales_mart
```

## Proposed schema changes

The following changes to the ODCS JSON Schema are proposed as part of this RFC. They are **not** yet applied to the standard; they will be applied to the target schema(s) if and when the TSC approves this RFC.

1. Add `ingres` and `vectorwise` to the server `type` enum:

```json
"enum": [
  "api", "athena", "azure", "bigquery", "clickhouse", "databricks", "denodo", "dremio",
  "duckdb", "exasol", "glue", "hana", "cloudsql", "db2", "hive", "iceberg", "impala", "informix",
  "ingres", "kafka", "kinesis", "local",
  ...
  "redshift", "s3", "sftp", "snowflake", "sqlserver", "synapse", "trino", "vectorwise", "vertica", "zen", "custom"
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
    "properties": { "type": { "const": "vectorwise" } },
    "required": ["type"]
  },
  "then": {
    "$ref": "#/$defs/ServerSource/VectorwiseServer"
  }
}
```

3. Add the `IngresServer` and `VectorwiseServer` definitions under `$defs/ServerSource`:

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
"VectorwiseServer": {
  "type": "object",
  "title": "VectorwiseServer",
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

1. **Should an optional `schema` field be added to both types?** See the note above.
2. **Should the two types ship as one RFC (as proposed) or be split into two, given the different products?**
3. **Should the naming convention be recorded outside this RFC?** If the TSC adopts it, it belongs in contributor guidance (`CONTRIBUTING.md` or the schema documentation) rather than only in the RFC that first applied it, so the next server-type RFC inherits it.

## Alternatives

### Naming of the Analytics Engine type

`analytics`. Rejected: too vague. It is a category word, not a product, in an enum whose every other entry names a concrete engine or service — and it would age badly the moment the enum needs a genuinely generic concept.

`vector`. Rejected: it is a mid-life marketing name rather than the creation name, so it fails the convention above, and it shares a word with the `vector` logical type introduced by RFC 0042. There is no technical collision (different field, different enum), but the overlap is a needless source of reader confusion.

`actian`. Rejected: Actian ships more than one engine — Ingres, the Analytics Engine, and Zen, and `zen` is already in the enum — so a vendor name is ambiguous in the other direction. No other value in the enum is a bare vendor name.

`actian-analytics` / `actianvector`. Rejected: both track current or recent marketing rather than the creation name, and `actian-analytics` would introduce the first hyphen into an enum that is otherwise all single lowercase words.

### Structural alternatives

Keep using `custom` for both. Rejected: hides connection details in unstructured fields, so nothing can be validated or generated from the contract.

Reuse the existing `zen` server type for either engine. Rejected: `zen` is Actian Zen, an embedded/edge database with a different connection model; it shares only the vendor.

Add a single `actian` server type with a discriminator field for the engine. Rejected: it pushes an engine choice that changes query semantics, type mapping, and workload profile into a secondary field, and it is inconsistent with how every other engine in the enum is modelled.

## Decision

> The decision made by the TSC.

## Consequences

- Nonbreaking change: adds two new optional server types.
- Tooling that parses the `servers` list can detect Ingres- and Analytics-Engine-backed contracts and generate the appropriate JDBC/ODBC connection strings, instead of treating them as opaque `custom` servers.
- `vectorwise` does not match the product's current marketing name. This is deliberate and is the point of the convention: the value is stable across renames, and the documentation carries the current name. Tooling and documentation should present the Analytics Engine label to users while writing `vectorwise` to the contract.
- Adopting the naming convention sets a precedent that binds future server-type RFCs. It does not change any existing value: `zen` and `synapse` stay as they are, since renaming them would be breaking.

## References

- [Actian Ingres](https://www.actian.com/databases/ingres/)
- [Actian Analytics Engine](https://www.actian.com/databases/analytics-engine/)
- RFC 0045 (HANA server type), RFC 0057 (Teradata server type), RFC 0058 (Exasol server type) — structural templates for a single-engine server type.
- RFC 0042 (Vector Type) — the unrelated `vector` logical type, one of the reasons `vector` was rejected as the enum value.
- Existing server types in ODCS (e.g. `zen`, `informix`, `db2`).
