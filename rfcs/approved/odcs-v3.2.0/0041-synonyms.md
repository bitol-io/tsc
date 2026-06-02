# Synonyms

Champion: Jean-Georges Perrin.

Authors: Massil Chabane, Denis Arnaud, Gilles Guglielmoni, and Jean-Georges Perrin.

Slack: https://data-mesh-learning.slack.com/archives/C0B334HJQV7

GitHub issue: TBD.

Applies to:
* [x] ODCS - Open Data Contract Standard
* [x] ODPS - Open Data Product Standard
* [ ] OORS - Open Observability Results Standard
* [ ] OOCS - Open Orchestration and Control Standard
* [ ] OMMS - Open Maturity Model Standard
* [ ] OMDS - Open Metadata Difference Standard

## Summary

This RFC introduces a standard `synonyms` field in the Bitol standards. Synonyms are alternative names for any named object (properties, schemas, measures, dimensions, data products, output ports, …) that help AI/LLM tools, catalogs, and natural language interfaces resolve the business vocabulary to the underlying object. This RFC defines **only the shape of a synonym** and where it may appear; it does not define semantic guidance mechanisms (see RFC-0038 for the broader `context` block).

This RFC was split out of RFC-0034 (Measures and Dimensions) after the TSC meeting on 2026-04-21, because synonyms apply more broadly than just to measures and dimensions.

## Motivation

### Why are we doing this?

Business users rarely refer to a metric or attribute by its technical name. "Total Revenue" may also be called "Turnover", "TO", "Sales volume", or — in a French-speaking subsidiary — "Chiffre d'affaires". Without a standardized way to record these alternative names, every consumer (BI tool, AI assistant, data catalog, semantic layer) reinvents synonym handling in a proprietary format, breaking interoperability.

### Use cases

1. **Natural-language querying**: An LLM or BI tool resolves a user's question ("What was our turnover last quarter?") to the right measure by matching the question text against the property's `synonyms`.
2. **Catalog search**: A data catalog indexes `synonyms` so that searching for "ABV" surfaces the `average_basket_value` measure.
3. **Cross-locale discovery**: Multi-national organizations attach locale-tagged synonyms so that users in different regions can discover the same asset using their native vocabulary.
4. **Glossary integration**: Synonyms can be sourced from an enterprise glossary, with `source` and `id` fields allowing round-trip synchronization.

### Alignment with guiding values

- **Small standard over large**: This RFC adds a single optional field. No new top-level structures.
- **Interoperability over readability**: The object form carries the metadata needed for multi-locale, versioned, source-tracked synonyms to round-trip across tools (Databricks Metric Views, dbt, OSI, glossaries, ontologies).

## Design and examples

### Core concept

`synonyms` is an optional array attached to any named object in ODCS and ODPS where alternative naming is useful. The primary expected locations are:

- Properties (columns, measures, dimensions — see RFC-0034).
- Schema objects (tables, topics) in ODCS.
- Output ports and the data product itself in ODPS.

Tools MUST NOT accept `synonyms` at additional locations; the standard forbids them.

### Field shape

Each `synonyms[]` entry is an **object** carrying the synonymous term and optional metadata. The TSC adopted the object form (over a flat array of strings) on 2026-06-01: the object form carries locale, source, and lifecycle metadata natively and is extensible without breaking changes.

```yaml
synonyms:
  - synonym: TO
    description: "Common abbreviation used by the finance team"
  - synonym: Sales
    locale: en-US
  - id: sales-fr
    synonym: "Chiffre d'affaires"
    locale: fr-FR
    description: "French equivalent used in French-speaking subsidiaries"
```

#### Synonym object fields

| Field              | Required | Type   | Description                                                                                                                          |
| ------------------ | -------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------ |
| `id`               | No       | string | Stable identifier for the synonym, useful when referencing or deduplicating synonyms across tools.                                   |
| `synonym`          | Yes      | string | The synonymous term. The TSC adopted `synonym` (over `name`) on 2026-06-01 because it is self-describing inside a `synonyms[]` array and avoids overloading `name` (which elsewhere refers to the owning object). |
| `description`      | No       | string | Short human-readable note about when or why this synonym is used.                                                                    |
| `locale`           | No       | string | [BCP 47](https://datatracker.ietf.org/doc/html/rfc5646) language tag (e.g., `en-US`, `fr-FR`) when the synonym is language-specific. |
| `source`           | No       | string | Origin of the synonym (e.g., `glossary`, `finance-team`, `legacy-system`).                                                           |
| `status`           | No       | string | Lifecycle status of the synonym (e.g., `active`, `deprecated`).                                                                      |
| `customProperties` | No       | array  | Extension point, per the standard pattern.                                                                                           |

### Example 1: Minimal — synonyms on a property

```yaml
properties:
  - name: total_revenue
    logicalType: number
    synonyms:
      - synonym: Revenue
      - synonym: Sales
      - synonym: TO
```

### Example 2: Detailed — localized and sourced synonyms on a measure

```yaml
properties:
  - name: total_turnover_euros
    implementationType: measure
    logicalType: number
    transformLogic: SUM(turnover_euros)
    businessName: TurnOver (Euros)
    synonyms:
      - synonym: TO
        description: "Common abbreviation used by the finance team"
        source: finance-team
      - synonym: Sales
        locale: en-US
      - synonym: Sales volume
        locale: en-US
      - id: sales-fr
        synonym: "Chiffre d'affaires"
        locale: fr-FR
        description: "French equivalent used in European subsidiaries"
      - synonym: Total number of orders
        status: deprecated
```

### Example 3: Synonyms on a schema object

```yaml
schema:
  - name: turnover
    physicalName: metrics_turnover
    synonyms:
      - synonym: Sales metrics
      - synonym: Revenue metrics
      - synonym: Chiffre d'affaires
        locale: fr-FR
```

### Example 4: Synonyms on an ODPS output port

```yaml
kind: DataProduct
name: Customer Master Data
outputPorts:
  - name: customer-master
    synonyms:
      - synonym: Master customer
      - synonym: Golden customer record
      - synonym: 360 customer view
```

### Compatibility with external tools

Databricks Metric Views accept `synonyms` as a flat list of strings. When exporting from ODCS/ODPS to Databricks, tools should project each synonym object to its `synonym` value. When importing from Databricks, tools should expand each string into an object `{ synonym: <string> }`.

OSI's `ai_context.synonyms` and glossary-term references from enterprise catalogs map naturally onto the object form via the `source` and `id` fields.

### Relationship to other RFCs

- **RFC-0034 (Measures and Dimensions)**: This RFC was factored out of RFC-0034. RFC-0034 defines `implementationType` on properties; RFC-0041 defines `synonyms` on any named object, including the measures and dimensions introduced by RFC-0034.
- **RFC-0038 (Context block)**: `synonyms` lives at the schema level (on the object itself), not inside the `context` block. RFC-0038 explicitly delegates synonym definition to this RFC. Tools needing synonym-based disambiguation MUST read `synonyms` from the named object, not from `context`.

## Alternatives

### Alternative A: Keep synonyms inside RFC-0034 (rejected)

Leave `synonyms` defined inside RFC-0034, scoped to measures and dimensions only.

**Rejected because**: synonyms are useful on regular columns, schemas, output ports, and data products — not only on measures and dimensions. Scoping them to RFC-0034 would either force duplication in a second RFC for non-measure uses, or push authors into `customProperties` for the other cases.

### Alternative B: Put synonyms inside the `context` block (rejected)

Define `synonyms` as a sub-field of RFC-0038's `context` block.

**Rejected because**: `context` is concerned with interpretive guidance for AI agents (instructions, constraints, verified Q&A). Synonyms are identity metadata on the named object itself, applicable even to consumers that have no AI context processing. Keeping `synonyms` adjacent to `name`, `businessName`, and `description` is more discoverable and more consistent with how other naming fields are organized.

### Alternative C: String-only form (rejected)

Use an array of strings instead of an array of objects:

```yaml
synonyms:
  - TO
  - Sales
  - Sales volume
```

**Rejected because**: shorter to author by hand, but every future need (locale, deprecation, provenance) would force either a breaking change or an escape into `customProperties`. The TSC selected the object form (Option B) on 2026-06-01.

### Alternative D: `name` as the term-carrying key (rejected)

Use `name` inside each synonym object, matching the rest of ODCS/ODPS where named objects use `name`.

**Rejected because**: ambiguous in isolation — `name: TO` inside a `synonyms[]` entry could be misread as the *owning* object's name in logs, diffs, and partial extracts. The TSC selected `synonym` on 2026-06-01 for its self-describing nature.

## Decision

**Approved by the TSC on 2026-06-01.** Both previously open items were resolved at the same vote:

1. **Field shape**: array of objects (Option B) was adopted over the string-only form. The object form's native carriage of locale, source, and lifecycle metadata outweighs the verbosity cost.
2. **Term-carrying key**: `synonym` was adopted over `name`, because it is self-describing inside a `synonyms[]` array and avoids overloading `name`.

## Consequences

- **Non-breaking change**: `synonyms` is an optional field. Existing contracts and products remain valid.
- **Cross-standard**: Applies to both ODCS and ODPS.
- **Tool interoperability**: Enables round-trip with Databricks Metric Views, dbt, OSI, and enterprise glossaries.
- **Clean separation of concerns**: RFC-0034 owns measure/dimension semantics; RFC-0038 owns interpretive context; RFC-0041 owns naming synonymy. Each can evolve independently.

## References

- [RFC-0034: Measures and Dimensions](../../0034-measures-and-dimensions.md) — original home of `synonyms` before the 2026-04-21 split.
- [RFC-0038: Context](0038-context.md) — semantic guidance for AI, which delegates synonym definition to this RFC.
- [Databricks Semantic Metadata](https://docs.databricks.com/aws/en/metric-views/data-modeling/semantic-metadata) — `synonyms` as a flat string list on dimensions and measures.
- [Open Semantic Interchange (OSI)](https://opensemanticinterchange.org/) — `ai_context.synonyms` at field level.
- [BCP 47 — Tags for Identifying Languages](https://datatracker.ietf.org/doc/html/rfc5646) — locale identifiers used in the `locale` field.

## Appendix: Changelog

All notable changes to this RFC are recorded here. Dates are `YYYY-MM-DD`. Entries are listed newest-first.

| Date       | Author              | Change                                                                                                                                                                                                                                                                          |
| ---------- | ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 2026-06-01 | Jean-Georges Perrin | TSC approved the RFC. Resolved both open items: adopted the object form (Option B) and the `synonym` key (over `name`). Examples, field table, and Alternatives section updated; discussion items removed. Soft-moved into `approved/odcs-v3.2.0/`.                              |
| 2026-05-19 | Jean-Georges Perrin | Opened a second discussion item: the term-carrying key inside a synonym object could be `name` (consistent with the rest of ODCS/ODPS) or `synonym` (self-describing inside `synonyms[]`). Updated the field table and Decision section; called for a vote at the next TSC meeting. |
| 2026-04-21 | Jean-Georges Perrin | Initial draft, factored out of RFC-0034 after the TSC meeting on 2026-04-21. Broadened scope from measures/dimensions to any named object in ODCS and ODPS. Added schema-level and output-port example sections.                                                                |
