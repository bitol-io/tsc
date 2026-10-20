# OData server type

Champion: Simon Harrer

Authors: Johan Åhlén

Slack: *TBD*

GitHub issue: *TBD*

Applies to:
* [x] ODCS - Open Data Contract Standard
* [ ] ODPS - Open Data Product Standard
* [ ] OORS - Open Observability Results Standard
* [ ] OOCS - Open Orchestration and Control Standard
* [ ] OMMS - Open Maturity Model Standard
* [ ] OMDS - Open Metadata Difference Standard
* [ ] OSDS - Open Semantic Definition Standard

## Summary

Add `odata` as a server `type` to describe data served through an [OData](https://www.odata.org/) service, and define how OData entity sets map to existing schema fields. No new schema-level fields are introduced.

## Motivation

OData (OASIS standard, ISO/IEC 20802) is a protocol for exposing data and its underlying model. It is widely used by SAP (S/4HANA, BW, Datasphere), Microsoft Dynamics 365 and Dataverse, SharePoint, and many public-sector and open-data portals. ODCS has no dedicated server type for it. Users today fall back to `api` and put the protocol version, metadata URL and entity-set URLs in `customProperties`, where nothing is validated and every tool invents its own keys.

OData provides a self-describing schema (similar to INFORMATION_SCHEMA in a relational database), an SQL-like standardized query language, and uniform responses and paging. A first-class `odata` type lets tooling connect to the service, resolve each schema object to its entity set, and validate a data contract.

## Design and examples

Add `odata` to the list of allowed server `type` values, with the following fields:

| Field             | Type   | Required | Description                                                                                                        |
| ----------------- | ------ | -------- | ------------------------------------------------------------------------------------------------------------------ |
| `type`            | string | Yes      | `odata`                                                                                                            |
| `location`        | string | Yes      | Service root URL of the OData service.                                                                             |
| `protocolVersion` | string | Yes      | OData protocol version the service must support: `2.0`, `3.0` or `4.0`; compatible later versions also satisfy the requirement. A 4.01 service satisfies `4.0`.    |

The ODCS server identifies the OData service, and each ODCS schema object identifies one OData entity set or singleton in it. URLs are resolved by the OData conventions and are not stored:

- Metadata document: `{location}/$metadata`
- Entity set or singleton: `{location}/{physicalName}`

### Example 1: Minimal

```yaml
servers:
  - server: prod
    type: odata
    location: https://services.odata.org/V4/TripPinService/
    protocolVersion: "4.0"
```

### Example 2: Structured — WHO reference data

```yaml
servers:
  - server: uat
    type: odata
    environment: uat
    location: https://xmart-api-public-uat.who.int/refmart/
    protocolVersion: "4.0"
schema:
  - name: REF_COUNTRY
    physicalName: REF_COUNTRY
    physicalType: entitySet
    logicalType: object
    properties:
      - name: CODE_ISO_3
        physicalType: Edm.String
        logicalType: string
        primaryKey: true
        primaryKeyPosition: 1
        required: true
        logicalTypeOptions:
          maxLength: 3
      - name: CODE_ISO_NUMERIC
        physicalType: Edm.Int32
        logicalType: integer
      - name: DATE_START
        physicalType: Edm.Date
        logicalType: date
```

## Proposed schema changes

The following changes to the ODCS JSON Schema are proposed as part of this RFC. They have **not** yet been applied to the standard; they will be applied to the target schema(s) if and when the TSC approves this RFC.

1. Add `odata` to the server `type` enum:

```json
"enum": [
  "api", "athena", "azure", "bigquery", "btrieve", "clickhouse", "databricks", "denodo", "dremio",
  "duckdb", "exasol", "fastobjects", "glue", "hana", "cloudsql", "db2", "hive", "iceberg", "impala",
  "informix", "ingres", "kafka", "kinesis", "local", "mysql", "odata", "oracle", "poet", "postgresql",
  "postgres", "presto", "pubsub", "redshift", "s3", "sftp", "snowflake", "sqlserver", "synapse",
  "teradata", "trino", "vectorwise", "versant", "vertica", "zen", "custom"
]
```

2. Add a conditional dispatch in the server `allOf` list:

```json
{
  "if": {
    "properties": { "type": { "const": "odata" } },
    "required": ["type"]
  },
  "then": {
    "$ref": "#/$defs/ServerSource/ODataServer"
  }
}
```

3. Add the `ODataServer` definition under `$defs/ServerSource`:

```json
"ODataServer": {
  "type": "object",
  "title": "ODataServer",
  "properties": {
    "location": {
      "type": "string",
      "format": "uri",
      "description": "Service root URL of the OData service.",
      "examples": ["https://services.odata.org/V4/TripPinService/"]
    },
    "protocolVersion": {
      "type": "string",
      "description": "OData protocol version the service must support; compatible later versions also satisfy the requirement."
    }
  },
  "required": ["location", "protocolVersion"]
}
```

4. Add `entitySet` and `singleton` to the documented examples of the schema object's `physicalType`.

## Alternatives

Keep using the `api` server type with `customProperties`. Rejected: `api` identifies an endpoint by its URL but does not standardize what an OData contract needs: the protocol version, the location of the metadata document, and how schema objects map to entity sets. These end up in tool-specific custom properties that cannot be validated, and a reader cannot tell whether `location` is the service root or an entity-set URL.

Use one `api` server per entity set, with `location` pointing to the entity-set URL. Rejected: it splits one service into many servers and loses the link to `$metadata`. `api` also does not describe the OData response envelope (`value`) or server-driven paging (`@odata.nextLink`), so each tool has to infer them. RFC 0064 (MCP server type) rejects `api` for a similar reason: a single `location` does not describe a protocol with its own structure.

Extend `api` with a `protocol` field (`rest`, `odata`, `graphql`). Rejected: the fields and resolution rules that follow are protocol-specific, which would require conditional sub-schemas inside `api`. Existing ODCS practice is to define a separate server type for each protocol or format (e.g. `iceberg` beside `s3`).

## Decision

> The decision made by the TSC.

## Consequences

- Nonbreaking change: adds a new optional server type.
- Tooling can test a contract against the service's `$metadata` (schema checks without fetching data) and fetch records per entity set, following OData paging.

## References

- [ISO/IEC 20802-1:2016](https://www.iso.org/standard/69208.html)
- [OData Version 4.01 Part 1: Protocol](https://docs.oasis-open.org/odata/odata/v4.01/odata-v4.01-part1-protocol.html)
- [OData Version 4.01 Part 2: URL Conventions](https://docs.oasis-open.org/odata/odata/v4.01/odata-v4.01-part2-url-conventions.html)
- [OData Common Schema Definition Language (CSDL) XML Version 4.01](https://docs.oasis-open.org/odata/odata-csdl-xml/v4.01/odata-csdl-xml-v4.01.html)
