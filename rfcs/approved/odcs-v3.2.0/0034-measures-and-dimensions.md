# Measures and Dimensions

Champion: Jean-Georges Perrin.

Authors: Massil Chabane, Denis Arnaud, Gilles Guglielmoni, and Jean-Georges Perrin.

Slack: https://data-mesh-learning.slack.com/archives/C0B334HJQV7

GitHub issue: TBD.

Applies to:
* [x] ODCS - Open Data Contract Standard
* [ ] ODPS - Open Data Product Standard
* [ ] OORS - Open Observability Results Standard
* [ ] OOCS - Open Orchestration and Control Standard
* [ ] OMMS - Open Maturity Model Standard
* [ ] OMDS - Open Metadata Difference Standard

## Summary

This RFC adds support for **measures** and **dimensions** to ODCS by extending the existing `properties` array with a new optional `semanticType` field. A property can be a `column` (default), a `measure` (aggregated value, e.g., `SUM(revenue)`), or a `dimension` (categorical attribute for grouping/filtering). This allows data contracts to express business metrics as first-class properties, reusing all existing property fields, so they can be consumed consistently across reporting, analytics, and AI tools.

Synonyms, originally proposed as part of this RFC, have been factored out into [RFC-0041](0041-synonyms.md) so they apply uniformly to any named object (columns, schemas, output ports, data products) rather than only to measures and dimensions.

## Motivation

### Why are we doing this?

Data contracts today describe the physical and logical structure of data (schemas, properties, types, quality rules) but do not capture the **semantic meaning of business metrics**. Organizations commonly define KPIs like "Total Revenue", "Average Basket Value", or "Number of Orders" — these are computed from underlying properties using aggregations and groupings. Without a standard representation, each team or tool reinvents these definitions, leading to inconsistent metrics across dashboards, reports, and AI-driven insights.

### Use cases

1. **Consistent metric definitions**: A data contract owner defines "Total Revenue = SUM(revenue_euros)" once. All downstream tools (BI, AI assistants, data catalogs) consume the same definition.
2. **Semantic layer interoperability**: Tools like Databricks Metric Views, dbt Semantic Layer, and others define metrics in proprietary formats. ODCS can serve as a portable, tool-agnostic interchange format.
3. **Self-service analytics**: Data consumers can discover available measures and dimensions from the contract without needing to understand the underlying SQL or table structure.
4. **AI/LLM readability**: Business names and (via [RFC-0041](0041-synonyms.md)) synonyms help AI assistants understand what a metric means in business terms, enabling natural language querying.

### Alignment with guiding values

- **We favor a small standard over a large one**: This RFC adds a single optional field (`semanticType`) to existing properties. No new top-level structures are introduced. Advanced features (joins, grains, materialization) are left to tool-specific extensions via `customProperties`.
- **We favor interoperability over readability**: The design is compatible with Databricks Metric Views, dbt metrics, and similar semantic layer tools, enabling import/export between ODCS and these tools.

## Design and examples

### Core concept

Measures and dimensions are **properties**. Rather than introducing new top-level arrays, this RFC adds an optional `semanticType` field to existing properties. This means measures and dimensions inherit all existing property fields (`name`, `logicalType`, `logicalTypeOptions`, `businessName`, `description`, `transformLogic`, `customProperties`, `authoritativeDefinitions`, etc.) with zero duplication.

### New fields

#### `semanticType` (optional, string, on properties)

Specifies the semantic role the property plays. One of:

| Value       | Description                                                                                                                                                                       |
| ----------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `column`    | A physical column in the underlying data store. **This is the default** — if `semanticType` is omitted, the property is a column.                                                 |
| `measure`   | An aggregated value computed via a SQL aggregation expression (e.g., `SUM(revenue)`, `COUNT(DISTINCT order_id)`). The `transformLogic` field contains the aggregation expression. |
| `dimension` | A categorical attribute used for grouping and filtering. The `transformLogic` field contains the SQL expression or property reference.                                            |

### Example 1: Minimal — a single measure and dimension

A simple e-commerce contract with one revenue measure and one dimension, alongside regular column properties.

```yaml
apiVersion: v3.2.0
kind: DataContract
id: turnover-metrics
name: Turnover Metrics

schema:
  - name: orders
    physicalName: fact_orders
    properties:
      - name: order_id
        logicalType: string
        primaryKey: true
      - name: revenue_euros
        logicalType: number
      - name: country_code
        logicalType: string
      - name: country_code_dim
        semanticType: dimension
        logicalType: string
        transformLogic: country_code
        description: "Country where the order was placed"
      - name: total_revenue
        semanticType: measure
        logicalType: number
        transformLogic: SUM(revenue_euros)
        description: "Total revenue in Euros"
```

### Example 2: Detailed — turnover metrics with formatting and computed measures

> The `synonyms` entries shown below are defined by [RFC-0041](0041-synonyms.md); they are included here to illustrate how measures and dimensions compose with synonyms in a realistic contract.


Inspired by the [data-engineering-helpers semantic layer example](https://github.com/data-engineering-helpers/semantic-layer/blob/main/odcs/metrics-turnover.yml).

```yaml
apiVersion: v3.2.0
kind: DataContract
id: turnover-detailed-metrics
name: Turnover Detailed Metrics

schema:
  - name: turnover
    physicalName: metrics_turnover
    description: "Turnover (TO) metrics for sales and financial analysis"
    properties:
      # Columns
      - name: turnover_ts
        logicalType: date
        description: "Timestamp when the TurnOver is calculated"
      - name: order_id
        logicalType: string
        description: "Order ID"
      - name: turnover_euros
        logicalType: number
        description: "TurnOver converted in Euros"
      - name: country_code
        logicalType: string
        description: "ISO-3166-2 country code"
      - name: product_type
        logicalType: string
        description: "Type of product sold"

      # Dimensions
      - name: turnover_ts_dim
        semanticType: dimension
        logicalType: date
        transformLogic: turnover_ts
        description: "Timestamp when the TurnOver is calculated"
        businessName: TurnOver Date
      - name: order_id_dim
        semanticType: dimension
        logicalType: string
        transformLogic: order_id
        description: "Order ID"
        businessName: Order ID
      - name: country_code_dim
        semanticType: dimension
        logicalType: string
        transformLogic: country_code
        description: "Country code where the order was made"
        businessName: Country Code
        synonyms:
          - synonym: "ISO-3166-2 country code"
            source: glossary
      - name: product_type_dim
        semanticType: dimension
        logicalType: string
        transformLogic: product_type
        description: "Type of product sold"
        businessName: Product Type

      # Measures
      - name: total_turnover_euros
        semanticType: measure
        logicalType: number
        logicalTypeOptions:
          format: currency
          currencyCode: EUR
          decimalPlaces: 2
        transformLogic: SUM(turnover_euros)
        description: "Total TurnOver in Euros"
        businessName: TurnOver (Euros)
        synonyms:
          - synonym: TO
            description: "Common abbreviation used by the finance team"
            source: finance-team
          - synonym: Sales
            locale: en-US
          - synonym: Sales volume
            locale: en-US
          - synonym: Chiffre d'affaires
            locale: fr-FR
            description: "French equivalent used in European subsidiaries"
      - name: nb_receipts
        semanticType: measure
        logicalType: integer
        logicalTypeOptions:
          decimalPlaces: 0
        transformLogic: COUNT(DISTINCT order_id)
        description: "Number of receipts"
        businessName: Number of Receipts
        synonyms:
          - synonym: Number of orders
          - synonym: Order Count
          - synonym: Total number of orders
            status: deprecated
      - name: average_basket_value
        semanticType: measure
        logicalType: number
        logicalTypeOptions:
          format: currency
          currencyCode: EUR
          decimalPlaces: 2
        transformLogic: "SUM(turnover_euros) / NULLIF(COUNT(DISTINCT order_id), 0)"
        description: "Average basket value in Euros"
        businessName: Average Basket Value
        synonyms:
          - synonym: ABV
            source: glossary
```

### Example 3: Dimension with a SQL expression

Dimensions can use SQL expressions in `transformLogic`, not just direct property references.

```yaml
    properties:
      # ... columns omitted for brevity ...
      - name: order_month
        semanticType: dimension
        logicalType: date
        transformLogic: DATE_TRUNC('MONTH', order_date)
        description: "Month of the order"
        businessName: Order Month
      - name: order_status_label
        semanticType: dimension
        logicalType: string
        transformLogic: |
          CASE
            WHEN order_status = 'O' THEN 'Open'
            WHEN order_status = 'P' THEN 'Processing'
            WHEN order_status = 'F' THEN 'Fulfilled'
          END
        description: "Human-readable order status"
        businessName: Order Status
```

### Compatibility with Databricks Metric Views

The proposed structure maps to Databricks Metric Views YAML. In Databricks, measures and dimensions are separate arrays; in ODCS, they are unified as properties distinguished by `semanticType`:

| ODCS property field            | Databricks Metric View field |
| ------------------------------ | ---------------------------- |
| `semanticType: dimension`      | entry in `dimensions[]`      |
| `semanticType: measure`        | entry in `measures[]`        |
| `name`                         | `name`                       |
| `transformLogic`               | `expr`                       |
| `description`                  | `comment`                    |
| `businessName`                 | `display_name`               |
| `logicalTypeOptions`           | `format`                     |

ODCS reuses existing property field names (`transformLogic`, `businessName`, `logicalType`, `logicalTypeOptions`) for consistency with the rest of the standard, while Databricks uses its own naming (`expr`, `display_name`, `format`). ODCS uses `camelCase` per convention; Databricks uses `snake_case`.

Synonym interoperability with Databricks Metric Views is covered by [RFC-0041](0041-synonyms.md).

## Alternatives

### Alternative A: Separate `measures` and `dimensions` arrays at the schema level (rejected)

Add two new arrays (`measures` and `dimensions`) alongside `properties` at the schema level, each with their own field definitions.

**Rejected because**: Measures and dimensions are conceptually properties — they have names, types, descriptions, business names, and transform logic, just like columns. Creating separate arrays would duplicate the property schema and require maintaining parallel field definitions. Using `semanticType` on existing properties is simpler and more consistent with the guiding value of favoring a small standard.

### Alternative B: Reuse the `quality` block for metrics (rejected)

Express measures as data quality rules with a special type.

**Rejected because**: Measures are fundamentally different from quality checks. Quality checks validate data; measures aggregate data for business consumption. Overloading `quality` would confuse semantics and make tool interoperability harder.

### Alternative C: Use `customProperties` only (rejected)

Store measures and dimensions as custom properties without standardized fields.

**Rejected because**: This defeats the purpose of a standard. Every tool would use different property names, making interoperability impossible. The guiding value "we favor interoperability over readability" demands a standardized structure.

### Alternative D: Include joins and source in ODCS (rejected for now)

Databricks Metric Views support `source`, `joins`, and `filter` at the metric view level.

**Rejected for now because**: ODCS already defines schema structure, servers, and relationships (via RFC-0013, RFC-0026b). Adding a parallel join/source mechanism would duplicate existing capabilities. The `transformLogic` field in measures and dimensions can reference properties defined in the schema. Cross-schema joins could be addressed in a follow-up RFC if needed, keeping this RFC small per our guiding values.

### Alternative E: Other names for the field (rejected)

Six finalists were on the ballot at the 2026-06-01 TSC vote, after earlier elimination of weaker candidates (`propertyType`, `role`, `kind`, `propertyRole`, `semanticRole`, `propertyImplementation`, `category` / `propertyCategory`, `type`):

- `usageType`
- `interpretationType`
- `reportingType`
- `metricType`
- `semanticType` — **adopted**
- `implementationType`

**`semanticType` was adopted** because it most accurately conveys the role the property plays in the *semantic model* (column / measure / dimension), independent of how the value is physically implemented or surfaced in reports. `implementationType` was second-closest but skewed toward the physical layer; `metricType` and `reportingType` were too narrow; `usageType` and `interpretationType` were too generic.

## Decision

**Approved by the TSC on 2026-06-01 for inclusion in ODCS v3.2.0.** The **concept** was approved earlier (2026-05-19); the **field name** `semanticType` was ratified at the 2026-06-01 vote among six finalists.

## Consequences

- **Non-breaking change**: `semanticType` is an optional field on properties. Existing contracts remain valid — all current properties are implicitly `semanticType: column`.
- **No schema duplication**: Measures and dimensions reuse the full property schema (name, logicalType, logicalTypeOptions, businessName, description, transformLogic, customProperties, authoritativeDefinitions, etc.).
- **Tool interoperability**: Tools supporting Databricks Metric Views, dbt Semantic Layer, or similar can import/export measures and dimensions via ODCS.
- **Schema evolution**: Future RFCs may extend this with joins across schemas, filter expressions, or grain definitions, building on this foundation.
- **ODPS alignment**: A follow-up RFC could bring the same structure to ODPS for data product metric definitions.

## References

- [Databricks Metric Views](https://docs.databricks.com/aws/en/metric-views/) — Unity Catalog metric views with YAML-based measures and dimensions.
- [Databricks YAML Syntax Reference](https://docs.databricks.com/aws/en/metric-views/data-modeling/syntax) — Full YAML syntax for metric view definitions.
- [Databricks Semantic Metadata](https://docs.databricks.com/aws/en/metric-views/data-modeling/semantic-metadata) — Display names, synonyms, and format specifications.
- [data-engineering-helpers semantic layer example](https://github.com/data-engineering-helpers/semantic-layer/blob/main/odcs/metrics-turnover.yml) — ODCS-inspired turnover metrics example.
- [dbt Semantic Layer / MetricFlow](https://docs.getdbt.com/docs/build/about-metricflow) — dbt's semantic layer for defining metrics alongside data models.
- [Open Semantic Interchange (OSI)](https://opensemanticinterchange.org/) — Open standard for sharing semantic models across tools and platforms.
- [AtScale SML](https://www.atscale.com/blog/introduction-to-sml-a-standard-semantic-modeling-language/) — Open-source Semantic Modeling Language for portable metric definitions.
- [RFC-0002 Data Types](../odcs-v3.0.0/0002-types.md) — ODCS logical/physical types.
- [RFC-0013 Relationships](../odcs-v3.1.0/0013-relationships-between-properties.md) — Relationships between properties.

## Appendix: Vendor Landscape for Measures and Dimensions

The concept of measures and dimensions is central to the semantic layer ecosystem. This appendix catalogs vendors and tools that define or consume measures and dimensions, demonstrating the broad industry need for a standardized, portable representation.

### Warehouse-Native Semantic Layers

| Vendor              | Product                                                                                                          | Measures/Dimensions Support                                                                          | Notes                                                   |
| ------------------- | ---------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- | ------------------------------------------------------- |
| **Databricks**      | [Metric Views](https://docs.databricks.com/aws/en/metric-views/)                                                 | YAML-based measures, dimensions, joins, filters, semantic metadata (synonyms, display names, format) | Unity Catalog native. Primary inspiration for this RFC. |
| **Google BigQuery** | [BigQuery BI Engine / Looker Semantic Layer](https://cloud.google.com/looker/docs/reference/param-measure-types) | Measures (with typed aggregations), dimensions via LookML                                            | Embedded in Looker/LookML modeling language.            |
| **Snowflake**       | [Semantic Views](https://docs.snowflake.com/en/user-guide/views-semantic/overview)                               | Metrics (aggregated measures), dimensions, facts as schema-level objects                             | GA since Snowflake Summit 2025. Native to Snowflake.    |

### Standalone Semantic Layer Platforms

| Vendor        | Product                                                                                | Measures/Dimensions Support                                                                                       | Notes                                                                                           |
| ------------- | -------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| **AtScale**   | [AtScale / SML](https://www.atscale.com/glossary/semantic-modeling-language/)          | Measures, dimensions, hierarchies, semi-additive measures, many-to-many relationships via open-source SML (YAML). | Enterprise-grade. First open-source semantic modeling language (2024). Joined OSI in Jan 2026.  |
| **Cube**      | [Cube Semantic Layer](https://cube.dev/docs/product/data-modeling/reference/measures)  | Measures and dimensions defined in YAML or JavaScript. Pre-aggregations, caching, multi-tenancy.                  | API-first / headless BI. Supports 20+ BI tools.                                                 |
| **dbt Labs**  | [dbt Semantic Layer / MetricFlow](https://docs.getdbt.com/docs/build/about-metricflow) | Measures, dimensions, entities in YAML semantic models. Metrics defined as compositions of measures.              | Powers the OSI proprietary standard. Integrates with Snowflake, Databricks, BigQuery, Redshift. |
| **Lightdash** | [Lightdash Metrics Explorer](https://www.lightdash.com/)                               | Measures and dimensions on top of dbt models. Metrics Explorer for governed metric reuse.                         | Open-source. Deep dbt integration.                                                              |

### BI Tools with Embedded Semantic Layers

| Vendor                   | Product                                                                                                         | Measures/Dimensions Support                                                                         | Notes                                                                    |
| ------------------------ | --------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| **Apache**               | [Apache Superset](https://github.com/apache/superset/issues/35003)                                              | Lightweight semantic layer with metrics and dimensions via SQL-first datasets.                      | Open-source. SIP-182 proposes deeper semantic layer support.             |
| **Google**               | [Looker / LookML](https://cloud.google.com/looker/docs/reference/param-measure-types)                           | Dimensions and measures as core LookML building blocks. Typed aggregations (sum, avg, count, etc.). | LookML is a YAML-like modeling language.                                 |
| **Metabase**             | [Metabase](https://www.metabase.com/)                                                                           | Simple semantic layer with aggregated metrics and certified measures.                               | Open-source. Best for non-technical users.                               |
| **Microsoft**            | [Power BI Semantic Models](https://learn.microsoft.com/en-us/power-bi/connect-data/service-datasets-understand) | Measures (DAX), dimensions, hierarchies in semantic models (formerly datasets).                     | Fabric IQ (2025) elevates semantic models into ontologies for AI agents. |
| **Omni Analytics**       | [Omni](https://omni.co/)                                                                                        | Measures and dimensions with semantic modeling. OSI founding member.                                | Modern BI tool with shared semantic definitions.                         |
| **Salesforce / Tableau** | [Tableau Semantic Layer](https://www.salesforce.com/blog/agentic-future-demands-open-semantic-layer/)           | Dimensions, measures, calculated fields. Participating in OSI initiative.                           | Salesforce actively advocates for open semantic layer standards.         |
| **ThoughtSpot**          | [ThoughtSpot](https://www.thoughtspot.com/)                                                                     | Measures and dimensions for AI-driven natural language search. Participating in OSI.                | Search-first BI.                                                         |

### Summary

The measures-and-dimensions pattern is a universal concept across the entire data analytics stack — from warehouse-native implementations (Databricks, Snowflake) to standalone semantic layers (dbt, Cube, AtScale) to BI tools (Power BI, Tableau, Looker, Superset). Adding measures and dimensions to ODCS positions the standard at the center of this ecosystem, enabling data contracts to serve as a portable, tool-agnostic semantic interchange format.

## Appendix: Changelog

All notable changes to this RFC are recorded here. Dates are `YYYY-MM-DD`. Entries are listed newest-first.

| Date       | Author                                                                | Change                                                                                                                                                                                                                                                                                                                                                                |
| ---------- | --------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 2026-06-01 | Jean-Georges Perrin                                                   | TSC ratified the field name: **`semanticType`** (over `usageType`, `interpretationType`, `reportingType`, `metricType`, `implementationType`). All examples, the field table, the Databricks compatibility table, and the Consequences section updated. Discussion section folded into a new Alternative E. Synonym examples in Example 2 updated to use `synonym:` (per the approved RFC-0041 key). Relative reference paths fixed for the new `approved/odcs-v3.2.0/` location. Soft-moved into `approved/odcs-v3.2.0/`. |
| 2026-05-19 | Jean-Georges Perrin                                                   | TSC approved the **concept** for ODCS v3.2.0; the **field name** was not ratified. Narrowed naming candidates to six finalists (`usageType`, `interpretationType`, `reportingType`, `metricType`, `semanticType`, `implementationType`) for a vote at the next TSC meeting. RFC stays in `tsc/rfcs/` until the name is settled. Deduplicated RFC-0041 pointers and trimmed Example 1. |
| 2026-04-21 | Jean-Georges Perrin                                                   | Split `synonyms` out of this RFC into [RFC-0041](0041-synonyms.md) following the TSC meeting on 2026-04-21. Removed the `synonyms` field definition and its discussion item; kept `synonyms` in Example 2 with a pointer to RFC-0041 to show how measures/dimensions compose with synonyms. Updated Summary, Use cases, Alignment-with-guiding-values, RFC-0038 note, Databricks compatibility table, and Consequences. No change to `implementationType` or its values.                                                                                                             |
| 2026-04-17 | Jean-Georges Perrin                                                   | Reframed `synonyms` as a **single-form** field (not dual-form). Two candidate shapes — array of strings vs. array of objects — are now presented as a TSC discussion item. The WG recommends the object form (with fields `id`, `name`, `description`, `locale`, `source`, `status`, `customProperties`); the rest of the RFC assumes that recommendation is adopted. |
|            |                                                                       | Bumped example `apiVersion` from `v3.1.0` to `v3.2.0` to reflect that this RFC targets the next ODCS minor release.                                                                                                                                                                                                                                                   |
|            |                                                                       | Added this Changelog appendix.                                                                                                                                                                                                                                                                                                                                        |
| —          | Massil Chabane, Denis Arnaud, Gilles Guglielmoni, Jean-Georges Perrin | Initial draft: `implementationType` on properties (`column`, `measure`, `dimension`), `synonyms` as array of strings, two worked examples, Databricks Metric Views mapping, alternatives, vendor landscape appendix.                                                                                                                                                  |
