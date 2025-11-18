# RFC-0031: Add relatesTo Relationship Type

> The champion is the person who is primarily supporting the RFC.

Champion: Diego C.

[Slack](https://data-mesh-learning.slack.com/archives/C08EF0M2FFV)

Jira: *replace this text with the link to the dedicated Jira ticket*.

## Summary

Extend the relationships structure (introduced in RFC-0026b) with a `relatesTo` relationship type for documenting semantic relationships, data lineage, and business context without enforcing structural constraints.

## Motivation

Beyond structural constraints (foreign keys), data contracts need to document:
- **Data lineage**: Where data came from, how it was derived or transformed
- **Business ↔ technical mapping**: Links between business concepts and technical implementations
- **Data flows**: Documentation of ETL/ELT processes and transformations
- **Governance ownership**: Who is responsible for specific data elements
- **Semantic relationships**: Synonyms, equivalents, dependencies between elements

Unlike `foreignKey` which enforces validation rules, `relatesTo` is purely informational—enabling rich metadata without imposing runtime constraints.

## Prerequisites

This RFC depends on:
- RFC-0026a (reference-id) - introduces `id` field for stable references
- RFC-0026b (internal-references) - establishes the `relationships` block structure

## Design

### Type: relatesTo (Semantic Documentation)

Documents relationships for lineage, governance, and business context. Does not enforce constraints—purely informational.

| Type | Purpose | `to` field | `from` field | Effect |
|------|---------|------------|--------------|--------|
| `relatesTo` | Documents semantic relationships | Required | Never used | Informational only (lineage, documentation) |

**When to use:**
- Data lineage tracking (derived from, copied from, transformed by)
- Business ↔ technical mapping (implements business definitions)
- Documenting data flows
- Governance ownership
- Semantic relationships (synonyms, equivalents, dependencies)

### Structure

```yaml
relationships:
  - type: relatesTo
    to: <target-reference>           # Always required
    description: <human-readable-text>
    customProperties:
      - property: <name>
        value: <value>
```

**Important:** The `from` field is NEVER used with `relatesTo`. The source is always implicit based on where the relationship is defined.

### Field definitions

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | enum | Yes | Must be: `relatesTo` |
| `to` | string or array | Yes | Target element reference |
| `from` | N/A | Never used | Not applicable for `relatesTo` - source is implicit |
| `description` | string | No | Human-readable explanation of the relationship |
| `customProperties` | array | No | Additional metadata following standard custom properties structure |

### Validation rules

Implementations SHOULD validate:

1. **Field requirements:**
   - MUST have `to` field
   - Must NOT have `from` field (it's never used for `relatesTo`)

2. **Reference resolution:**
   - Target IDs referenced in `to` must exist
   - Referenced paths must resolve correctly
   - External files must be accessible

## Examples

### Example 1: Data lineage - derived field

```yaml
properties:
  - name: monthly_total
    id: monthly_total_field
    logicalType: number
    relationships:
      - type: relatesTo
        to: schema/daily_sales/properties/amount
        description: "Derived from: Aggregated SUM of daily transaction amounts"
    customProperties:
      - property: aggregation_logic
        value: "SUM(amount) GROUP BY MONTH(transaction_date)"
```

### Example 2: Business definition mapping

```yaml
schema:
  - id: customer_table
    name: customers
    physicalName: crm_prod.customers
    relationships:
      - type: relatesTo
        to: business-glossary.yaml#/schema/customer_concept
        description: "Implements the Customer business entity concept"
    properties:
      - id: status_field
        name: status
        relationships:
          - type: relatesTo
            to: business-glossary.yaml#/schema/customer_concept/properties/customer_status
            description: "Implements Customer Status business definition"
```

### Example 3: Multiple semantic relationships

```yaml
properties:
  - name: email
    id: email_field
    logicalType: string
    relationships:
      - type: relatesTo
        to: crm-system.yaml#/schema/contacts/properties/email
        description: "Copied from CRM system via nightly ETL"

      - type: relatesTo
        to: etl-pipeline.yaml#jobs.data_sync_job
        description: "Transformed by data sync pipeline"

      - type: relatesTo
        to: governance-model.yaml#roles.data_privacy_officer
        description: "Governed by Data Privacy Officer"
```

### Example 4: Calculated field with lineage

```yaml
schema:
  - id: orders_tbl
    name: orders
    properties:
      - id: ord_total
        name: order_total
        businessName: Order Total Amount
        logicalType: number
        physicalType: decimal(12,2)
        required: true
        description: Total amount for entire order
        relationships:
          # Document lineage: calculated from order_items
          - type: relatesTo
            to: schema/order_items_tbl/properties/item_line_total
            description: Calculated as SUM(line_total) from order_items table
        customProperties:
          - property: calculationLogic
            value: SUM(order_items.line_total) WHERE order_items.order_id = orders.order_id
```

### Example 5: Data Lineage Across Multi-Tier Architecture

Demonstrates tracking data flow through bronze/silver/gold layers:

**bronze-layer.yaml** (Raw Ingestion):
```yaml
dataContractSpecification: 3.2.0
id: bronze-transactions
version: 1.0.0

schema:
  - id: raw_transactions
    name: transactions_raw
    properties:
      - id: raw_txn_id
        name: transaction_id
        logicalType: string

      - id: raw_amount
        name: amount
        logicalType: string
        description: Raw amount as ingested (string format)
```

**silver-layer.yaml** (Cleaned/Validated):
```yaml
dataContractSpecification: 3.2.0
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
          - type: relatesTo
            to: bronze-transactions.yaml#/schema/raw_transactions/properties/raw_txn_id
            description: Converted from string to integer during cleansing

      - id: clean_amount
        name: amount_usd
        logicalType: number
        relationships:
          - type: relatesTo
            to: bronze-transactions.yaml#/schema/raw_transactions/properties/raw_amount
            description: Parsed from string, validated, converted to numeric USD
```

**gold-layer.yaml** (Business Aggregates):
```yaml
dataContractSpecification: 3.2.0
id: gold-daily-revenue
version: 1.0.0

schema:
  - id: daily_revenue
    name: daily_revenue_summary
    properties:
      - id: revenue_date
        name: business_date
        logicalType: date

      - id: total_revenue
        name: total_revenue_usd
        logicalType: number
        relationships:
          - type: relatesTo
            to: silver-transactions.yaml#/schema/cleaned_transactions/properties/clean_amount
            description: Aggregated SUM of cleaned transaction amounts by date

          - type: relatesTo
            to: bronze-transactions.yaml#/schema/raw_transactions/properties/raw_amount
            description: Ultimate source is raw transaction amounts from bronze layer
        customProperties:
          - property: aggregation_logic
            value: SUM(amount_usd) GROUP BY DATE(transaction_timestamp)
```

### Example 6: Business Glossary to Technical Implementation Mapping

**business-glossary.yaml** (Business Team Owned):
```yaml
dataContractSpecification: 3.2.0
id: business-data-glossary
version: 3.0.0
title: Enterprise Business Data Glossary

schema:
  - id: customer_concept
    name: Customer
    businessName: Customer Entity
    description: |
      A customer is any individual or organization that has
      entered into a business relationship with our company.
    properties:
      - id: customer_identifier
        name: customer_id
        businessName: Customer Identifier
        description: Unique identifier assigned to each customer

      - id: customer_status
        name: status
        businessName: Customer Status
        description: Current lifecycle status of the customer relationship
```

**crm-technical-contract.yaml** (IT Team Owned):
```yaml
dataContractSpecification: 3.2.0
id: crm-customer-table
version: 2.3.0
title: CRM Customer Table (Salesforce)

schema:
  - id: sf_customer
    name: customer__c
    physicalName: crm_prod.salesforce.customer__c
    logicalType: object
    relationships:
      # Link entire table to business concept
      - type: relatesTo
        to: business-glossary.yaml#/schema/customer_concept
        description: Technical implementation of Customer business entity

    properties:
      - id: sf_cust_id
        name: customer_id__c
        physicalName: customer_id__c
        logicalType: string
        relationships:
          # Link to specific business definition
          - type: relatesTo
            to: business-glossary.yaml#/schema/customer_concept/properties/customer_identifier
            description: Implements Customer Identifier business definition

      - id: sf_cust_status
        name: status__c
        physicalName: status__c
        logicalType: string
        relationships:
          # Document relationship to business concept
          - type: relatesTo
            to: business-glossary.yaml#/schema/customer_concept/properties/customer_status
            description: Implements Customer Status business definition
```

**dwh-technical-contract.yaml** (IT Team Owned - Different System):
```yaml
dataContractSpecification: 3.2.0
id: dwh-customer-dimension
version: 4.1.0
title: Data Warehouse Customer Dimension

schema:
  - id: dim_customer
    name: dim_customer
    physicalName: analytics.dimensions.dim_customer
    logicalType: object
    relationships:
      # Same business concept, different technical implementation
      - type: relatesTo
        to: business-glossary.yaml#/schema/customer_concept
        description: DWH implementation of Customer business entity

      # Document data lineage from source system
      - type: relatesTo
        to: crm-technical-contract.yaml#/schema/sf_customer
        description: Data sourced from Salesforce CRM via ETL pipeline

    properties:
      - id: dwh_cust_id
        name: customer_id
        physicalName: customer_business_id
        logicalType: string
        relationships:
          # Business definition link
          - type: relatesTo
            to: business-glossary.yaml#/schema/customer_concept/properties/customer_identifier
            description: Implements Customer Identifier business definition

          # Technical lineage
          - type: relatesTo
            to: crm-technical-contract.yaml#/schema/sf_customer/properties/sf_cust_id
            description: Populated from Salesforce customer_id__c field
```

### Example 7: Mixed relationship types (foreignKey + relatesTo)

A single property can have multiple relationships of different types:

```yaml
properties:
  - name: customer_id
    id: cust_id_field
    logicalType: integer
    relationships:
      # Structural constraint
      - type: foreignKey
        to: schema/customers/properties/id
        description: "Must reference valid customer"

      # Document where it came from
      - type: relatesTo
        to: source-system.yaml#/schema/orders/properties/cust_no
        description: "Copied from source system customer number field"
```

### Example 8: SLA monitoring with relatesTo

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
      - type: relatesTo
        to: schema/orders_tbl/properties/order_total/quality/amount_positive
        description: Monitors the positive amount validation rule
```

## Use Cases

Key scenarios enabled by `relatesTo`:

1. **Data Lineage Tracking**: Document transformations across bronze/silver/gold layers
2. **Business-Technical Alignment**: Link technical implementations to business glossary terms
3. **Multi-System Integration**: Track data flow across CRM → DWH → BI systems
4. **Governance Mapping**: Associate data elements with responsible parties/roles
5. **Impact Analysis**: Understand dependencies when making changes
6. **Calculation Documentation**: Explain how derived/calculated fields are produced
7. **SLA Monitoring**: Link SLAs to specific quality rules they monitor

## Alternatives

Considered embedding lineage and semantic information directly in fields like `description` or `customProperties`, but a structured relationship type provides:
- Machine-readable lineage graphs
- Consistent tooling support
- Validation of referenced elements
- Clear separation from structural constraints

## Decision

> The decision made by the TSC.

## Consequences

> The consequences of this decision.

## References

> Prior art, inspiration, and other references you used to create this based on what's worked well before.

- Data lineage standards (Apache Atlas, OpenLineage)
- Business glossary patterns (Collibra, Alation)
- Semantic web relationships (RDF, SKOS)

Formerly part of RFC 0026.
