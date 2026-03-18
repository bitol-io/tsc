# RFC-0031: Informational Relationship Types

Champion: Diego C.

[Slack](https://data-mesh-learning.slack.com/archives/C08EF0M2FFV)

## Summary

Extend the relationships structure (introduced in RFC-0026b) with one or more informational relationship types for documenting data lineage, business context, and governance ownership—without enforcing structural constraints.

This RFC presents two options for the working group to evaluate.

## Motivation

Beyond structural constraints (foreign keys), data contracts need to document:
- **Data lineage**: Where data came from, how it was derived or transformed
- **Business ↔ technical mapping**: Links between business concepts and technical implementations
- **Data flows**: Documentation of ETL/ELT processes and transformations
- **Governance ownership**: Who is responsible for specific data elements

Unlike `foreignKey` which enforces validation rules, informational relationships are purely documentary—enabling rich metadata without imposing runtime constraints.

## Prerequisites

This RFC depends on:
- RFC-0026a (reference-id) - introduces `id` field for stable references
- RFC-0026b (internal-references) - establishes the `relationships` block structure

---

## Option A: Single Generic Type (`relatesTo`)

### Overview

One catch-all informational type. The nature of the relationship is expressed through `description` and optionally through a `customProperty` subtype discriminator.

### Structure

```yaml
relationships:
  - type: relatesTo
    to: <target-reference>           # Always required
    description: <human-readable-text>
    customProperties:
      - property: subtype
        value: lineage | businessDefinition | governedBy  # optional, community convention
```

### When to use

- Data lineage (derived from, copied from, transformed by)
- Business ↔ technical mapping (implements a business definition)
- Governance ownership (governed by a role or party)
- Any other informational relationship not covered by `foreignKey`

### Field definitions

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | enum | Yes | Must be: `relatesTo` |
| `to` | string or array | Yes | Target element reference |
| `from` | N/A | Never used | Source is implicit from where the relationship is defined |
| `description` | string | Recommended | Human-readable explanation of the relationship |
| `customProperties` | array | No | Additional metadata; can carry a `subtype` discriminator by convention |

### Validation rules

Implementations SHOULD validate:
1. MUST have `to` field
2. Must NOT have `from` field
3. Target IDs referenced in `to` must exist and paths must resolve correctly

### Trade-offs

| | |
|---|---|
| **Pros** | Simple — one new type, minimal additions to the spec. Flexible for unforeseen use cases. Easy to adopt incrementally. |
| **Cons** | Semantically weak — tooling cannot distinguish lineage from governance without relying on an unofficial `subtype` convention. Less discoverable for authors. |

---

## Option B: Three Specific Informational Types

### Overview

Replace `relatesTo` with three named types, each with a defined purpose and scope. No generic fallback.

| Type | Purpose | Typical `to` target |
|------|---------|---------------------|
| `lineage` | Documents data provenance — where data originated, how it was derived or transformed | Field or table in a source contract |
| `businessDefinition` | Maps a technical element to a business concept, glossary term, or authoritative definition | Element in a business glossary contract |
| `governedBy` | Associates a data element with a responsible role, party, or policy | Role, team, or policy reference |

### Common structure (all three types)

```yaml
relationships:
  - type: lineage | businessDefinition | governedBy
    to: <target-reference>           # Always required
    description: <human-readable-text>
    customProperties:
      - property: <name>
        value: <value>
```

**Important:** The `from` field is NEVER used with any of these types. The source is always implicit from where the relationship is defined.

---

### Type: `lineage`

**Scope:** Documents the provenance of a data element—its origin, transformations applied, or how it is calculated. Enables machine-readable lineage graphs across systems and pipeline layers.

**Use when:**
- A field is copied from a source system
- A field is derived by transformation or aggregation
- Tracking data flow through bronze/silver/gold layers
- Documenting ETL/ELT pipeline dependencies

With enterprise data catalogs in mind the lineage on field level could be especially useful in the presence of system breaks. Most of the data catalogs struggle to extract technical lineages above and beyond system boundaries. The proposed relationship type `lineage` would offer an easy, lightweight, and machine readable remedy for this fundamental issue. The description of lineage on higher abstraction levels (tables, objects, etc.) would also be possible.

An alternative would be the defintion of input port data contracts within the ODPS framework. But this option does not provide the same granularity.

**Field definitions**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | enum | Yes | Must be: `lineage` |
| `to` | string or array | Yes | Source field, table, or pipeline reference |
| `from` | N/A | Never used | Source is implicit from where the relationship is defined |
| `description` | string | Recommended | Describes the transformation or derivation logic |
| `customProperties` | array | No | e.g. `calculationLogic`, `aggregation_logic` |

---

### Type: `businessDefinition`

**Scope:** Maps a technical data element to its authoritative business meaning—a concept in a business glossary, data dictionary, or canonical data model. Enables semantic governance and tooling that auto-populates descriptions or enforces glossary coverage.

**Use when:**
- A table implements a business entity concept
- A field implements a specific business term or definition
- Linking a technical contract to a business-owned glossary contract (see RFC-0015)

**Field definitions**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | enum | Yes | Must be: `businessDefinition` |
| `to` | string or array | Yes | Reference to the business concept or glossary element |
| `from` | N/A | Never used | Source is implicit from where the relationship is defined |
| `description` | string | No | Optional clarifying note |
| `customProperties` | array | No | Additional metadata |

---

### Type: `governedBy`

**Scope:** Associates a data element with the role, team, or policy responsible for its governance. Purely informational—does not enforce access control or ownership in the runtime system.

**Use when:**
- A field is subject to data privacy rules and a named officer/role is accountable
- A domain or table has a designated data owner
- Documenting compliance relationships (e.g. GDPR, CCPA accountability)

**Field definitions**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | enum | Yes | Must be: `governedBy` |
| `to` | string or array | Yes | Reference to a role, team, or policy element |
| `from` | N/A | Never used | Source is implicit from where the relationship is defined |
| `description` | string | No | Optional clarifying note |
| `customProperties` | array | No | Additional metadata |

---

### Validation rules (all three types)

Implementations SHOULD validate:
1. MUST have `to` field
2. Must NOT have `from` field
3. Target IDs referenced in `to` must exist and paths must resolve correctly

### Trade-offs

| | |
|---|---|
| **Pros** | Semantically precise — tooling can act on type without parsing descriptions. Discoverable — authors know exactly which type to choose. Machine-readable lineage graphs. Consistent with how `foreignKey` names a specific semantic intent. |
| **Cons** | Three new enum values instead of one. Edge cases may not fit neatly — authors may feel forced or may misuse types. A future fourth type would require another RFC amendment. |

---

## Comparison

| Concern | Option A (`relatesTo`) | Option B (3 specific types) |
|---------|----------------------|----------------------------|
| Spec simplicity | One new type | Three new types |
| Tooling support | Requires parsing descriptions or `subtype` convention | Native — type is machine-readable |
| Author clarity | Ambiguous — what goes here? | Clear — each type has a defined scope |
| Extensibility | Any relationship fits | New use cases may require spec amendment |
| Alignment with `foreignKey` pattern | Diverges (broad vs. specific) | Consistent (each type = specific semantic intent) |

---

## Examples

### Example 1: Data lineage — derived field

**Option A:**
```yaml
properties:
  - name: monthly_total
    id: monthly_total_field
    logicalType: number
    relationships:
      - type: relatesTo
        to: schema/daily_sales/properties/amount
        description: "Aggregated SUM of daily transaction amounts"
        customProperties:
          - property: subtype
            value: lineage
          - property: aggregation_logic
            value: "SUM(amount) GROUP BY MONTH(transaction_date)"
```

**Option B:**
```yaml
properties:
  - name: monthly_total
    id: monthly_total_field
    logicalType: number
    relationships:
      - type: lineage
        to: schema/daily_sales/properties/amount
        description: "Aggregated SUM of daily transaction amounts"
        customProperties:
          - property: aggregation_logic
            value: "SUM(amount) GROUP BY MONTH(transaction_date)"
```

---

### Example 2: Business definition mapping

**Option A:**
```yaml
schema:
  - id: customer_table
    name: customers
    relationships:
      - type: relatesTo
        to: business-glossary.yaml#/schema/customer_concept
        description: "Implements the Customer business entity concept"
        customProperties:
          - property: subtype
            value: businessDefinition
    properties:
      - id: status_field
        name: status
        relationships:
          - type: relatesTo
            to: business-glossary.yaml#/schema/customer_concept/properties/customer_status
            description: "Implements Customer Status business definition"
            customProperties:
              - property: subtype
                value: businessDefinition
```

**Option B:**
```yaml
schema:
  - id: customer_table
    name: customers
    relationships:
      - type: businessDefinition
        to: business-glossary.yaml#/schema/customer_concept
        description: "Implements the Customer business entity concept"
    properties:
      - id: status_field
        name: status
        relationships:
          - type: businessDefinition
            to: business-glossary.yaml#/schema/customer_concept/properties/customer_status
```

---

### Example 3: Governance ownership

**Option A:**
```yaml
properties:
  - name: email
    id: email_field
    logicalType: string
    relationships:
      - type: relatesTo
        to: governance-model.yaml#roles.data_privacy_officer
        description: "Governed by Data Privacy Officer"
        customProperties:
          - property: subtype
            value: governedBy
```

**Option B:**
```yaml
properties:
  - name: email
    id: email_field
    logicalType: string
    relationships:
      - type: governedBy
        to: governance-model.yaml#roles.data_privacy_officer
        description: "Data Privacy Officer accountable for this field"
```

---

### Example 4: Mixed relationship types (all options)

A single property can carry multiple relationships of different types:

**Option A:**
```yaml
properties:
  - name: customer_id
    id: cust_id_field
    logicalType: integer
    relationships:
      # Structural constraint
      - type: foreignKey
        to: schema/customers/properties/id

      # Lineage
      - type: relatesTo
        to: source-system.yaml#/schema/orders/properties/cust_no
        description: "Copied from source system customer number field"
        customProperties:
          - property: subtype
            value: lineage

      # Business definition
      - type: relatesTo
        to: business-glossary.yaml#/schema/customer_concept/properties/customer_identifier
        customProperties:
          - property: subtype
            value: businessDefinition
```

**Option B:**
```yaml
properties:
  - name: customer_id
    id: cust_id_field
    logicalType: integer
    relationships:
      # Structural constraint
      - type: foreignKey
        to: schema/customers/properties/id

      # Lineage
      - type: lineage
        to: source-system.yaml#/schema/orders/properties/cust_no
        description: "Copied from source system customer number field"

      # Business definition
      - type: businessDefinition
        to: business-glossary.yaml#/schema/customer_concept/properties/customer_identifier
```

---

### Example 5: Data lineage across multi-tier architecture (bronze/silver/gold)

**silver-layer.yaml** (Option B shown; Option A substitutes `type: lineage` → `type: relatesTo` with `subtype: lineage`):
```yaml
dataContractSpecification: 3.2.0
kind: DataContract
id: silver-transactions
version: 1.0.0

schema:
  - id: cleaned_transactions
    name: transactions_cleaned
    properties:
      - id: clean_txn_id
        name: transaction_id
        logicalType: integer
        relationships:
          - type: lineage
            to: bronze-transactions.yaml#/schema/raw_transactions/properties/raw_txn_id
            description: Converted from string to integer during cleansing

      - id: clean_amount
        name: amount_usd
        logicalType: number
        relationships:
          - type: lineage
            to: bronze-transactions.yaml#/schema/raw_transactions/properties/raw_amount
            description: Parsed from string, validated, converted to numeric USD
```

**gold-layer.yaml:**
```yaml
dataContractSpecification: 3.2.0
kind: DataContract
id: gold-daily-revenue
version: 1.0.0

schema:
  - id: daily_revenue
    name: daily_revenue_summary
    properties:
      - id: total_revenue
        name: total_revenue_usd
        logicalType: number
        relationships:
          - type: lineage
            to: silver-transactions.yaml#/schema/cleaned_transactions/properties/clean_amount
            description: Aggregated SUM of cleaned transaction amounts by date

          - type: lineage
            to: bronze-transactions.yaml#/schema/raw_transactions/properties/raw_amount
            description: Ultimate source is raw transaction amounts from bronze layer
        customProperties:
          - property: aggregation_logic
            value: SUM(amount_usd) GROUP BY DATE(transaction_timestamp)
```

---

### Example 6: Business glossary to technical implementation (RFC-0015 Option C alignment)

See RFC-0015 for the full business glossary contract definition. The technical contract links to it as follows:

**Option B:**
```yaml
dataContractSpecification: 3.2.0
kind: DataContract
id: crm-customer-table
version: 2.3.0

schema:
  - id: sf_customer
    name: customer__c
    physicalName: crm_prod.salesforce.customer__c
    relationships:
      - type: businessDefinition
        to: business-glossary.yaml#/schema/customer_concept
        description: Technical implementation of Customer business entity

    properties:
      - id: sf_cust_id
        name: customer_id__c
        logicalType: string
        relationships:
          - type: businessDefinition
            to: business-glossary.yaml#/schema/customer_concept/properties/customer_identifier

      - id: sf_cust_status
        name: status__c
        logicalType: string
        relationships:
          - type: businessDefinition
            to: business-glossary.yaml#/schema/customer_concept/properties/customer_status
```

---

### Example 7: SLA monitoring

```yaml
schema:
  - id: orders_tbl
    name: orders
    properties:
      - id: order_total
        name: total_amount
        logicalType: number
        quality:
          - id: amount_positive
            metric: invalidValues
            arguments:
              validValues: [">0"]

slaProperties:
  - id: slo_order_validity
    property: dataQuality
    value: 100
    unit: percent
    description: Order amounts must pass all validations
    relationships:
      # Option A
      - type: relatesTo
        to: schema/orders_tbl/properties/order_total/quality/amount_positive
        description: Monitors the positive amount validation rule

      # Option B equivalent
      # - type: lineage
      #   to: schema/orders_tbl/properties/order_total/quality/amount_positive
      #   description: Monitors the positive amount validation rule
```

---

## Open Questions for the Working Group

1. **Option A vs. Option B**: Does the community prefer simplicity (one generic type) or precision (three named types)? How important is native tooling support for distinguishing lineage vs. governance without description parsing?

2. **`relatesTo` as fallback in Option B**: Should Option B include `relatesTo` as a generic escape hatch for cases that don't fit the three named types, at the cost of reintroducing ambiguity?

3. **`businessDefinition` vs. `authoritativeDefinitions`**: ODCS already has an `authoritativeDefinitions` field on properties. If Option B is adopted, should `businessDefinition` in relationships supersede or coexist with `authoritativeDefinitions`? (Related: RFC-0015)

4. **Naming**: Is `lineage` the right name, or does `derivedFrom` / `sourceOf` better reflect directionality? Is `governedBy` clear, or does `ownedBy` / `managedBy` fit better?

---

## Alternatives Considered

Embedding lineage and semantic information directly in `description` or `customProperties` was considered, but a structured relationship type provides:
- Machine-readable lineage graphs
- Consistent tooling support
- Validation of referenced elements
- Clear separation from structural constraints

---

## Decision

> The decision made by the TSC.

## Consequences

> The consequences of this decision.

## References

- Data lineage standards (Apache Atlas, OpenLineage)
- Business glossary patterns (Collibra, Alation)
- Semantic web relationships (RDF, SKOS)
- RFC-0015 (Business Definitions) — Option C proposes `businessDefinition` as a relationship type
- RFC-0026a (reference-id)
- RFC-0026b (internal-references)

Formerly part of RFC 0026.
