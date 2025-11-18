# RFC-0032: Add imports Relationship Type

> The champion is the person who is primarily supporting the RFC.

Champion: Diego C.

[Slack](https://data-mesh-learning.slack.com/archives/C08EF0M2FFV)

Jira: *replace this text with the link to the dedicated Jira ticket*.

## Summary

Extend the relationships structure with an `imports` relationship type for content transclusion—importing definitions from external sources and merging them into the contract as if they were defined locally.

## Motivation

Data contracts often need to reuse common definitions across multiple contracts:
- **Reusable quality rule libraries**: Standard validation rules (email format, phone format, non-negative amounts)
- **Standard property definitions**: Common audit fields (created_at, updated_at, created_by)
- **Shared SLA templates**: Organization-wide SLA configurations
- **Common authoritative definitions**: Centralized business definitions

Without `imports`, teams must:
- Copy-paste definitions across contracts (violates DRY principle)
- Manually synchronize changes across multiple contracts
- Risk inconsistencies when definitions drift

The `imports` relationship type enables DRY (Don't Repeat Yourself) patterns by transcluding external content directly into contracts.

## Prerequisites

This RFC depends on:
- RFC-0026a (reference-id) - introduces `id` field for stable references
- RFC-0026b (internal-references) - establishes the `relationships` block structure

## Design

### Type: imports (Content Transclusion)

Imports definitions from external sources and merges them into the contract as if they were defined locally. Must be an allowed structure.

| Type | Purpose | `to` field | `from` field | Effect |
|------|---------|------------|--------------|--------|
| `imports` | Imports/transclude external content | Required (import source) | Never used | Content is merged as if locally defined |

**When to use:**
- Reusable quality rule libraries
- Standard property definitions
- Shared SLA templates
- Common authoritative definitions

**What can be imported:**
- Single quality rules or arrays of quality rules
- Property definitions (including nested properties)
- SLA property configurations
- Authoritative definition links
- Custom property definitions

### Structure

```yaml
relationships:
  - type: imports
    to: <source-reference>           # Always required - points to what to import
    description: <human-readable-text>
    customProperties:
      - property: <name>
        value: <value>
```

**Important:**
- The `from` field is NEVER used with `imports`
- The `to` field points to the source to import FROM (not the target to import TO)
- Imported content is merged as if it were defined locally

### Field definitions

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | enum | Yes | Must be: `imports` |
| `to` | string | Yes | Source element reference to import from |
| `from` | N/A | Never used | Not applicable for `imports` |
| `description` | string | No | Human-readable explanation of what is being imported |
| `customProperties` | array | No | Additional metadata following standard custom properties structure |

### Validation rules

Implementations SHOULD validate:

1. **Field requirements:**
   - MUST have `to` field
   - Must NOT have `from` field (it's never used for `imports`)

2. **Import validation:**
   - Content referenced in `to` field must exist and be importable
   - Imported content type must be compatible with target location (e.g., quality rule imports into quality section)
   - Circular imports must be detected and prevented

3. **Reference resolution:**
   - External files must be accessible
   - Referenced paths must resolve correctly

## Examples

### Example 1: Shared Quality Rules Library Pattern

Demonstrates reusable quality rule definitions that can be imported across multiple contracts.

**common-quality-rules.yaml** (Shared Library):
```yaml
dataContractSpecification: 3.2.0
id: common-quality-rules
version: 1.0.0
title: Shared Quality Rules Library

# Define reusable quality rules at top level
quality:
  - id: email_validation
    name: email_format_check
    metric: pattern
    arguments:
      regex: '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    description: Standard email format validation

  - id: phone_us_format
    name: us_phone_validation
    metric: pattern
    arguments:
      regex: '^\+?1?\d{10}$'
    description: US phone number format (10 digits)

  - id: non_negative_amount
    name: amount_non_negative
    metric: invalidValues
    arguments:
      validValues: ['>=0']
    description: Ensures monetary amounts are non-negative

  - id: standard_id_checks
    name: id_validations
    metric: nullValues
    mustBe: 0
    description: Standard ID field must not be null

  - id: future_date_check
    name: no_future_dates
    metric: invalidValues
    arguments:
      validValues: ['<=NOW()']
    description: Date must not be in the future
```

**customer-contract.yaml** (Consumer Contract):
```yaml
dataContractSpecification: 3.2.0
id: customer-master-data
version: 1.5.0

schema:
  - id: customers
    name: customers
    properties:
      - id: email_field
        name: email
        logicalType: string
        relationships:
          # Import email validation from shared library
          - type: imports
            to: common-quality-rules.yaml#quality.email_validation
            description: Import standard email validation

      - id: phone_field
        name: phone
        logicalType: string
        relationships:
          # Import phone validation
          - type: imports
            to: common-quality-rules.yaml#quality.phone_us_format
            description: Import US phone format validation

      - id: account_balance
        name: balance
        logicalType: number
        relationships:
          # Import amount validation
          - type: imports
            to: common-quality-rules.yaml#quality.non_negative_amount
            description: Balance cannot be negative
```

### Example 2: Import quality rule from shared library

```yaml
properties:
  - name: email
    id: email_field
    logicalType: string
    relationships:
      - type: imports
        to: common-quality-rules.yaml#quality.email_validation
        description: "Import standard email validation rule from common library"
```

After import processing, this is equivalent to:
```yaml
properties:
  - name: email
    id: email_field
    logicalType: string
    quality:
      - id: email_validation
        name: email_format_check
        metric: pattern
        arguments:
          regex: '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
        description: Standard email format validation
```

### Example 3: Import multiple quality rules

```yaml
properties:
  - name: customer_id
    id: cust_id_field
    logicalType: integer
    relationships:
      - type: imports
        to: common-rules.yaml#quality.standard_id_checks
        description: "Import all standard ID validation rules"
```

### Example 4: Import standard audit properties

```yaml
# templates.yaml (Shared Template Library)
schema:
  - id: audit_fields_template
    name: audit_fields
    properties:
      - id: created_at_field
        name: created_at
        logicalType: timestamp
        required: true
        description: Timestamp when record was created

      - id: updated_at_field
        name: updated_at
        logicalType: timestamp
        required: true
        description: Timestamp when record was last updated

      - id: created_by_field
        name: created_by
        logicalType: string
        required: true
        description: User who created the record
```

```yaml
# orders-contract.yaml (Consumer)
schema:
  - id: orders
    name: orders
    relationships:
      - type: imports
        to: templates.yaml#/schema/audit_fields_template.properties
        description: "Import standard audit fields (created_at, updated_at, created_by)"
    properties:
      - id: order_id
        name: order_id
        logicalType: integer
      # Audit fields will be merged here automatically
```

### Example 5: Complete E-commerce Contract with Mixed Relationship Types

This example demonstrates using `foreignKey`, `relatesTo`, and `imports` together:

```yaml
dataContractSpecification: 3.2.0
id: ecommerce-orders
version: 2.1.0
title: E-commerce Orders Contract

schema:
  - id: customers_tbl
    name: customers
    logicalType: object
    description: Customer master data
    properties:
      - id: cust_id
        name: id
        logicalType: integer
        description: Primary key for customers

      - id: cust_email
        name: email
        logicalType: string
        relationships:
          # Import standard email validation from shared library
          - type: imports
            to: common-quality-rules.yaml#quality.email_validation
            description: Standard email format validation

          # Document lineage from source system
          - type: relatesTo
            to: crm-system.yaml#/schema/contacts/properties/email_address
            description: Copied from CRM system via nightly ETL pipeline

      - id: cust_segment
        name: customer_segment
        logicalType: string
        relationships:
          # Link to business definition
          - type: relatesTo
            to: business-glossary.yaml#/schema/customer_segmentation/properties/segment_type
            description: Implements Customer Segmentation business concept
        quality:
          - id: valid_segments
            metric: enumValues
            arguments:
              validValues: ['PREMIUM', 'STANDARD', 'TRIAL']

  - id: orders_tbl
    name: orders
    logicalType: object
    description: Order transactions
    properties:
      - id: order_id
        name: id
        logicalType: integer
        description: Primary key for orders
        quality:
          - id: order_id_not_null
            metric: nullValues
            mustBe: 0

      - id: order_customer_id
        name: customer_id
        logicalType: integer
        relationships:
          # Foreign key constraint
          - type: foreignKey
            to: schema/customers_tbl/properties/cust_id
            description: Must reference valid customer

      - id: order_total
        name: total_amount
        logicalType: number
        relationships:
          # Document calculation lineage
          - type: relatesTo
            to: [schema/order_items_tbl/properties/item_price, schema/order_items_tbl/properties/quantity]
            description: Calculated as SUM(item_price * quantity) from order items

          # Import standard amount validation
          - type: imports
            to: common-quality-rules.yaml#quality.non_negative_amount
            description: Amount must be non-negative

      - id: order_created
        name: created_at
        logicalType: timestamp
        relationships:
          # Import standard audit field definition
          - type: imports
            to: standard-templates.yaml#properties.audit_created_timestamp
            description: Standard audit timestamp field

  - id: order_items_tbl
    name: order_items
    logicalType: object
    description: Line items for orders
    properties:
      - id: item_order_id
        name: order_id
        logicalType: integer
        relationships:
          - type: foreignKey
            to: schema/orders_tbl/properties/order_id
            description: Must reference valid order

      - id: item_product_id
        name: product_id
        logicalType: string
        relationships:
          # Foreign key to external contract
          - type: foreignKey
            to: product-catalog.yaml#/schema/products/properties/product_id
            description: Must reference valid product from catalog

      - id: item_price
        name: price
        logicalType: number

      - id: quantity
        name: quantity
        logicalType: integer

# SLA monitoring specific quality rules
slaProperties:
  - id: sla_customer_data_quality
    property: dataQuality
    value: 99.5
    unit: percent
    relationships:
      - type: relatesTo
        to: [schema/customers_tbl/properties/cust_email/quality/email_validation,
             schema/customers_tbl/properties/cust_segment/quality/valid_segments]
        description: Monitors email validation and segment enumeration rules

  - id: sla_order_integrity
    property: dataQuality
    value: 100
    unit: percent
    relationships:
      - type: relatesTo
        to: schema/orders_tbl/properties/order_id/quality/order_id_not_null
        description: All orders must have valid IDs

customProperties:
  - property: data_governance_owner
    value: customer-data-team
    relationships:
      - type: relatesTo
        to: governance-model.yaml#roles.customer_data_steward
        description: Governed by Customer Data Stewardship team
```

### Example 6: Importing Business Validation Rules

```yaml
# business-glossary.yaml (Business Team Owned)
schema:
  - id: customer_concept
    name: Customer
    properties:
      - id: customer_status
        name: status
        businessName: Customer Status
        quality:
          - id: valid_statuses
            metric: enumValues
            arguments:
              validValues: ['ACTIVE', 'INACTIVE', 'SUSPENDED', 'CLOSED']
```

```yaml
# crm-technical-contract.yaml (IT Team Owned)
schema:
  - id: sf_customer
    name: customer__c
    properties:
      - id: sf_cust_status
        name: status__c
        physicalName: status__c
        logicalType: string
        relationships:
          # Import business validation rules
          - type: imports
            to: business-glossary.yaml#/schema/customer_concept/properties/customer_status/quality/valid_statuses
            description: Use business-defined valid status values

          # Document relationship to business concept
          - type: relatesTo
            to: business-glossary.yaml#/schema/customer_concept/properties/customer_status
            description: Implements Customer Status business definition
```

## Use Cases

Key scenarios enabled by `imports`:

1. **Centralized Quality Rule Management**: Maintain validation rules in one place, import everywhere
2. **Standard Field Templates**: Define common fields (audit timestamps, metadata) once, reuse across contracts
3. **Business Rule Consistency**: Import business-defined validation rules into technical contracts
4. **Compliance Templates**: Share regulatory compliance rules across organization
5. **Multi-System Consistency**: Ensure same validation rules apply across CRM, DWH, and BI systems
6. **Version Control**: Update shared library once, all consumers get updated rules
7. **DRY Principle**: Eliminate copy-paste duplication of common definitions

## Import Processing Behavior

When a contract is processed/validated:

1. **Resolution**: The `to` reference is resolved to the source definition
2. **Type Checking**: Verify the imported content type is compatible with target location
3. **Merging**: The imported content is merged into the target as if it were defined locally
4. **Circular Detection**: Check for circular import chains and reject if found
5. **Validation**: The merged result is validated according to ODCS schema rules

## Alternatives

Considered alternatives:

1. **Copy-paste approach**: Simple but violates DRY, creates maintenance burden
2. **JSON Schema $ref**: Too low-level, doesn't fit ODCS semantic model
3. **Template inheritance**: More complex, less explicit than targeted imports
4. **Macro expansion**: Pre-processing approach, less transparent than relationship-based imports

## Decision

> The decision made by the TSC.

## Consequences

### Positive
- Enables DRY patterns for data contracts
- Centralized management of common definitions
- Consistent validation rules across contracts
- Easier maintenance and updates
- Clear dependency tracking

### Negative
- Adds complexity to contract processing
- Requires circular import detection
- External dependencies must be managed
- Version compatibility concerns

### Neutral
- Implementations must support content transclusion
- Tooling must resolve and merge imported content
- Documentation must explain import behavior clearly

## References

> Prior art, inspiration, and other references you used to create this based on what's worked well before.

- OpenAPI $ref mechanism
- JSON Schema $ref
- YAML anchors and aliases
- Terraform modules
- DRY principle (Don't Repeat Yourself)

Formerly part of RFC 0026.
