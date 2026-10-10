# Actian server types

Champion: Jean-Georges Perrin

Authors: Jean-Georges Perrin, Stephan Baumann, Andreas Renner, Ulf Schenk, Nils Behncke

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

Add four server `type` values to ODCS: `ingres` (Actian Ingres), `vectorwise` (Actian Analytics Engine), `versant` (Actian NoSQL Database) and `poet` (Actian NoSQL FastObjects). Add `btrieve` as a synonym of the existing `zen` type, and `fastobjects` as a synonym of `poet`.

## Motivation

Actian Ingres is an enterprise RDBMS for transactional processing (OLTP) and hybrid workloads. The Actian Analytics Engine is a columnar analytical DBMS for high-speed SQL and big data processing. Actian NoSQL Database and Actian NoSQL FastObjects are object database management systems (ODBMS): the first for mission-critical OLTP, the second for embedded and client/server applications.

ODCS has a type for none of the four, so users fall back to `custom` and lose validatable connection metadata.

## Design and examples

### Naming convention

> A server `type` value uses the name the product carried when it was created, in lowercase.

Enum values are permanent; marketing names are not. See [Appendix A](#appendix-a-convention-rationale) and [Appendix B](#appendix-b-conformance-of-existing-values).

### `ingres`

| Field      | Type    | Required | Description                                                                                                             |
| ---------- | ------- | -------- | ----------------------------------------------------------------------------------------------------------------------- |
| `type`     | string  | Yes      | `ingres`                                                                                                                |
| `database` | string  | Yes      | Database name to connect to on the Ingres instance.                                                                     |
| `host`     | string  | Yes      | Hostname or IP address of the Ingres server.                                                                            |
| `port`     | integer | No       | Connection port for the Ingres Data Access Server (DAS) / SQL connections. Defaults to `21064` (installation ID `II7`). |

### `vectorwise`

Created as VectorWise, renamed Actian Vector, now the Actian Analytics Engine.

| Field      | Type    | Required | Description                                                                              |
| ---------- | ------- | -------- | ---------------------------------------------------------------------------------------- |
| `type`     | string  | Yes      | `vectorwise`                                                                             |
| `database` | string  | Yes      | Database name to connect to on the Analytics Engine server.                              |
| `host`     | string  | Yes      | Hostname or IP address of the Analytics Engine server.                                   |
| `port`     | integer | No       | Connection port for the Data Access Server (DAS) / SQL connections. Defaults to `21064`. |

For `ingres` and `vectorwise`: `port` is derived from the Ingres installation identifier, so it is optional but recommended when a host runs more than one installation. No `schema` field is proposed; the namespace inside a database is the table owner, resolved from the connecting user.

### `versant`

Created as the Versant Object Database, now Actian NoSQL Database.

| Field      | Type    | Required | Description                                                                   |
| ---------- | ------- | -------- | ----------------------------------------------------------------------------- |
| `type`     | string  | Yes      | `versant`                                                                     |
| `database` | string  | Yes      | Database name to connect to on the NoSQL Database instance.                   |
| `host`     | string  | No       | Hostname or IP address of the NoSQL Database server. Defaults to `localhost`. |
| `port`     | integer | No       | Connection port for Actian NoSQL Database connections. Defaults to `5019`.    |

### `poet`

Created as POET, renamed FastObjects, now Actian NoSQL FastObjects.

| Field      | Type    | Required | Description                                                            |
| ---------- | ------- | -------- | ---------------------------------------------------------------------- |
| `type`     | string  | Yes      | `poet`                                                                 |
| `database` | string  | Yes      | Database name to connect to on the FastObjects instance.               |
| `host`     | string  | No       | Hostname or IP address of the FastObjects server. Defaults to `LOCAL`. |
| `port`     | integer | No       | Connection port for FastObjects connections. Defaults to `6001`.       |

`LOCAL` is a literal FastObjects sentinel, not a hostname: it selects the in-process embedded engine. One `host` field therefore covers both the embedded and the client/server deployment.

`fastobjects` is added as a synonym of `poet`: same fields, same `PoetServer` definition, both values valid, neither deprecated. The engine has shipped as FastObjects since 2001, so writers who know it by that name should not have to look up its creation name.

For both ODBMS types: no `schema` field is proposed. An object database has no SQL schema namespace; classes are scoped by the database.

### `btrieve`

Actian Zen was created as Btrieve, then Pervasive PSQL. `btrieve` is added as a synonym of `zen`: same fields, same `ZenServer` definition, both values valid, `zen` not deprecated. Precedent: `postgresql` and `postgres` are already synonyms sharing `PostgresServer`. Same mechanism as `fastobjects` above.

### Examples

Minimal, Ingres:

```yaml
servers:
  - server: prod
    type: ingres
    host: ingres.acme.com
    database: sales
```

Structured, Ingres:

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

Minimal, Analytics Engine:

```yaml
servers:
  - server: prod
    type: vectorwise
    host: analytics.acme.com
    database: warehouse
```

Structured, Analytics Engine on a non-default installation:

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

Minimal, NoSQL Database:

```yaml
servers:
  - server: prod
    type: versant
    host: nosql.acme.com
    database: policies
```

Structured, NoSQL Database:

```yaml
servers:
  - server: prod
    type: versant
    description: Policy administration OLTP object store
    environment: prod
    host: nosql01.acme.com
    port: 5019
    database: policies
    roles:
      - role: acme_policies_ro
        access: read
```

Minimal, FastObjects embedded (no `host`, so the in-process engine):

```yaml
servers:
  - server: device
    type: poet
    database: telemetry
```

Structured, FastObjects client/server:

```yaml
servers:
  - server: preprod
    type: poet
    description: Embedded configuration store for the CAD workstations
    environment: preprod
    host: fastobjects02.acme.com
    port: 6001
    database: parts_catalog
```

Synonym, POET addressed as FastObjects:

```yaml
servers:
  - server: prod
    type: fastobjects
    host: fastobjects.acme.com
    database: parts_catalog
```

Synonym, Zen addressed as Btrieve:

```yaml
servers:
  - server: prod
    type: btrieve
    host: zen.acme.com
    database: inventory
```

## Proposed schema changes

Proposed, not applied. They land only if the TSC approves.

1. Add `btrieve`, `fastobjects`, `ingres`, `poet`, `versant` and `vectorwise` to the server `type` enum:

```json
"enum": [
  "api", "athena", "azure", "bigquery", "btrieve", "clickhouse", "databricks", "denodo", "dremio",
  "duckdb", "exasol", "fastobjects", "glue", "hana", "cloudsql", "db2", "hive", "iceberg", "impala",
  "informix",
  "ingres", "kafka", "kinesis", "local",
  ...
  "oracle", "poet", "postgres", "postgresql",
  ...
  "redshift", "s3", "sftp", "snowflake", "sqlserver", "synapse", "trino", "vectorwise", "versant",
  "vertica", "zen", "custom"
]
```

2. Add the conditional dispatches in the server `allOf` list. `btrieve` reuses `ZenServer` and `fastobjects` reuses `PoetServer`:

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
},
{
  "if": {
    "properties": { "type": { "const": "versant" } },
    "required": ["type"]
  },
  "then": {
    "$ref": "#/$defs/ServerSource/VersantServer"
  }
},
{
  "if": {
    "properties": { "type": { "const": "poet" } },
    "required": ["type"]
  },
  "then": {
    "$ref": "#/$defs/ServerSource/PoetServer"
  }
},
{
  "if": {
    "properties": { "type": { "const": "fastobjects" } },
    "required": ["type"]
  },
  "then": {
    "$ref": "#/$defs/ServerSource/PoetServer"
  }
},
{
  "if": {
    "properties": { "type": { "const": "btrieve" } },
    "required": ["type"]
  },
  "then": {
    "$ref": "#/$defs/ServerSource/ZenServer"
  }
}
```

3. Add four definitions under `$defs/ServerSource`:

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
},
"VersantServer": {
  "type": "object",
  "title": "VersantServer",
  "properties": {
    "database": {
      "type": "string",
      "description": "Database name to connect to on the NoSQL Database instance.",
      "examples": ["policies"]
    },
    "host": {
      "type": "string",
      "description": "Hostname or IP address of the NoSQL Database server.",
      "default": "localhost",
      "examples": ["nosql.acme.com"]
    },
    "port": {
      "$ref": "#/$defs/Port",
      "description": "Connection port for Actian NoSQL Database connections.",
      "default": 5019,
      "examples": [5019]
    }
  },
  "required": ["database"]
},
"PoetServer": {
  "type": "object",
  "title": "PoetServer",
  "properties": {
    "database": {
      "type": "string",
      "description": "Database name to connect to on the FastObjects instance.",
      "examples": ["parts_catalog"]
    },
    "host": {
      "type": "string",
      "description": "Hostname or IP address of the FastObjects server. The literal LOCAL selects the in-process embedded engine.",
      "default": "LOCAL",
      "examples": ["LOCAL", "fastobjects02.acme.com"]
    },
    "port": {
      "$ref": "#/$defs/Port",
      "description": "Connection port for FastObjects connections.",
      "default": 6001,
      "examples": [6001]
    }
  },
  "required": ["database"]
}
```

## Open questions

1. Add an optional `schema` field to `ingres` and `vectorwise`, for symmetry with the other SQL types?
2. One RFC, or one per engine?
3. Record the naming convention in contributor guidance, so the next server-type RFC inherits it?

## Alternatives

| Rejected                               | Reason                                                                                      |
| -------------------------------------- | ------------------------------------------------------------------------------------------- |
| `analytics`                            | A category, not a product. Every other value names a concrete engine.                       |
| `vector`                               | Mid-life name, and overlaps the `vector` logical type of RFC 0042.                          |
| `actian`                               | Vendor name. Actian ships five of the engines named here; `zen` is already in the enum.     |
| `actian-analytics`, `actianvector`     | Track current marketing, and `actian-analytics` would be the first hyphen in the enum.      |
| `nsql`, `actian-nosql`                 | Post-acquisition branding, applied in 2017; `nsql` also reads as a category, not a product. |
| `vod`                                  | Initialism. No other enum value is one.                                                     |
| `custom` for any of them               | Hides connection details in unstructured fields.                                            |
| Reuse `zen`                            | A different engine with a different connection model. Only the vendor is shared.            |
| One `actian` type with a discriminator | Puts the engine choice in a secondary field, unlike every other engine in the enum.         |
| Rename `zen` to `btrieve`              | Breaking. A synonym reaches the same result at no cost.                                     |
| `fastobjects` as the primary value     | The 2001 rebrand, not the creation name. Kept as a synonym instead.                         |

## Decision

**Approved by the TSC on 2026-08-28.**

## Consequences

- Nonbreaking: four new optional types, two new synonyms. No existing value changes meaning.
- `vectorwise`, `versant` and `poet` do not match current marketing. Tooling should display the Analytics Engine, Actian NoSQL Database and Actian NoSQL FastObjects labels, and write the enum value.
- Two enum values map to `ZenServer` and two to `PoetServer`, as `postgresql` and `postgres` already map to `PostgresServer`. Writers pick one; readers accept both.
- The convention binds future server-type RFCs. It renames nothing.

## References

- [Actian Ingres](https://www.actian.com/databases/ingres/)
- [Actian Analytics Engine](https://www.actian.com/databases/analytics-engine/)
- [Actian NoSQL object databases](https://www.actian.com/databases/nosql/)
- RFC 0045 (HANA), RFC 0057 (Teradata), RFC 0058 (Exasol) — structural templates.
- RFC 0042 (Vector Type) — the unrelated `vector` logical type.

## Appendix A: convention rationale

An enum value cannot be renamed without a breaking change, so it is a permanent API. Product names are not: they change on rebrands and acquisitions while the connection model stays put. The Analytics Engine has been VectorWise (2010, from the CWI X100 research line), Actian Vector, and Actian Analytics Engine — three names, one engine. Zen has been Btrieve, Pervasive PSQL, Actian Zen. FastObjects has been POET (Poet Software GmbH), FastObjects by Poet (June 2001), Versant FastObjects (Versant merged with Poet Holdings in March 2004), Actian NoSQL FastObjects (Actian acquired Versant in December 2012) — four names, one engine. The Versant Object Database held its name from 1988 until the 2017 Actian NoSQL rebrand. The creation name is the only one that never changes afterwards, so it is the only one safe to commit to an enum.

## Appendix B: conformance of existing values

| Value                                                                     | Created as               | Conforms |
| ------------------------------------------------------------------------- | ------------------------ | -------- |
| `exasol`, `hana`, `vertica`, `informix`, `db2`, `snowflake`, `databricks` | same name                | Yes      |
| `zen`                                                                     | Btrieve                  | No       |
| `synapse`                                                                 | Azure SQL Data Warehouse | No       |

`zen` gains the `btrieve` synonym under this RFC. `synapse` is left alone; no synonym is proposed for it here.
