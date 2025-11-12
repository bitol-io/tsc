# Identify an element within a contract

> The champion is the person who is primarily supporting the RFC.

Champion: Diego C. 

[Slack](https://data-mesh-learning.slack.com/archives/C08EF0M2FFV)

Jira: *replace this text with the link to the dedicated Jira ticket*.

## Summary

We have a need to be able to identify the specific items within a data contract. At the same time this is the prerequisit to link between elements living in different data contracts.


## Motivation

Adding the formal capability to establish links between elements witin or between contracts dramatically extends the versatility of the entire standard. Potential use cases include but are not limited to:
- Describe lineages between technical elements (e. g. on field or table level) within a system landscape if this could not be retrieved or maintained otherwise.
- Describe lineages between the elements of a pure non-technical business data model on arbitrary aggregation levels. This could be very useful for complex organisations acting in highly regulated industries, like banking or pharma.
- Keep business data definitions in separate and lean data contracts for business departments use only. These definitions are then linked to from within other data contracts of pure technical nature owned by IT departments in a potential 1:n manner. This is the ONLY way to avoid redundant business documentations in complex system landscapes where the data travel between different systems (e. g. from a SAP EWM through a DWH into a BI tool).
- Human readable und primary-key-like links to Business Data definitions could be used within technical environments (e. g. within comments in a Snowflake environment) to be read out by meta data crawlers to easily join these definitons prior to upload these into a Data Catalog.
- Establish and maintain abstract Data Governance models whose elements describe C-level responsibilities for data assest. Linking business data models and/or technical data models to the elements of DG models establish corresponding management responsibilities.
- If data quality rules are maintained within data contracts these can be easily referred to from arbitrary DQ engines within different technical environments.


## Design and examples

Original work and design was completed under RFC-0009b - however at the TSC 
2025-05-20 - Option B | Option A - neither approved. Json Pointers were too brittle given the array structure. JSON paths were too complex in writing them effectively. TSC decision was to continue looking for options. 

This RFC continues the search for a feasible internal reference structure. 

### Option E: ID-based internal references (proposed)

Add an optional `id` field to most top-level and repeated objects in an ODCS contract. Internal references will use these `id` values instead of JSON Pointers/Paths to uniquely identify target elements.

Key goals:
- Keep authoring simple and human-readable to the greatest extent
- Support addressing specific items within arrays
- Avoid brittleness from re-ordering arrays

#### Scope (where `id` is allowed)

IDs are allowed on addressable items that may be targets of references:
- Contract-level repeated blocks: `schema` objects, `servers` items, `roles` items, `support` items, `slaProperties` items, `quality` items, `customProperties` items
- Schema structures: `SchemaObject` (objects/tables) and `SchemaProperty` (properties/columns)

Notes:
- Do not add `id` to trivial leaf fields like `logicalType`, `physicalType`, etc.; add it to the containing item (object/property/rule) instead

Rules:
- `id` is optional everywhere it is allowed
- `id` MUST be unique within its containing array/object collection
- ~`id` SHOULD be snake_case and technical in nature for readability and stability~ (UUIDs should be allowed)
- `id` SHOULD be stable across versions to preserve referential integrity

Rationale:
- Using optional IDs avoids forcing changes on all contracts while enabling robust internal and external links where needed
- Array re-ordering does not break references

#### Object Inventory

This section documents all major objects and arrays within ODCS and ODPS, identifying their current primary/key fields and highlighting opportunities for `id`-based referencing.

##### Open Data Contract Standard (ODCS) v3.1.0

**Top-Level Arrays and Objects**

| Object/Array | Current Primary Field | Field Type | Description |
|--------------|----------------------|------------|-------------|
| **servers** | `server` | string (required) | Identifier of the server |
| **schema** | `name` | string (required) | Name of the schema object (table/object) |
| **support** | `channel` | string (required) | Channel name or identifier |
| **team** | `username` | string (required) | User's username or email (flat array of members) |
| **roles** | `role` | string (required) | Name of the IAM role |
| **slaProperties** | `property` | string (required) | Specific SLA property name |
| **customProperties** | `property` | string (required) | Name of the custom property |

**Nested Arrays within Schema**

| Object/Array Path | Current Primary Field | Field Type | Description |
|------------------|----------------------|------------|-------------|
| **schema[].properties** | `name` | string (required) | Name of the property/column |
| **schema[].properties[].properties** | `name` | string (required) | Nested properties (for object/array types) |
| **schema[].quality** | `name` | string (optional) | Short name for the quality rule |
| **schema[].properties[].quality** | `name` | string (optional) | Short name for the quality rule |
| **schema[].relationships** | `from` + `to` | string/array (required) | Source and target references |
| **schema[].properties[].relationships** | `to` | string/array (required) | Target reference (`from` is implicit) |

**Objects with Optional Name Fields**

| Object Type | Location | Name Field Status | Notes |
|-------------|----------|------------------|-------|
| **Quality rules** | `schema[].quality[]` or `schema[].properties[].quality[]` | `name` (optional) | Also has `metric` field; currently identified by position in array |
| **AuthoritativeDefinition** | Multiple locations | No name field | Has `type` and `url` (both required) |
| **TeamMember** | `team[]` | `name` (optional) | Primary key is `username` (required) |
| **Relationship** | Schema or property level | No name field | Identified by `from`/`to` references |

##### Open Data Product Standard (ODPS) v1.0.0

**Top-Level Arrays and Objects**

| Object/Array | Current Primary Field | Field Type | Description |
|--------------|----------------------|------------|-------------|
| **inputPorts** | `name` | string (required) | Name of the input port |
| **outputPorts** | `name` | string (required) | Name of the output port |
| **managementPorts** | `name` | string (required) | Endpoint identifier or unique name |
| **support** | `channel` | string (required) | Channel name or identifier |
| **team** | N/A (single object) | N/A | Container object, not an array |
| **team.members** | `username` | string (required) | User's username or email |

**Objects with Optional Name Fields**

| Object Type | Location | Name Field Status | Notes |
|-------------|----------|------------------|-------|
| **Team** | `team` | `name` (optional) | Single object, not an array |
| **TeamMember** | `team.members[]` | `name` (optional) | Primary key is `username` (required) |
| **InputContract** | `outputPorts[].inputContracts[]` | No name field | Has `id` and `version` (both required) |
| **SBOM** | `outputPorts[].sbom[]` | No name field | Has `type` and `url` |

##### Candidates for `id` Field Support

**High Priority** (Arrays with required primary fields that may need stable references):

ODCS:
- `servers[]` - current key: `server`
- `schema[]` - current key: `name`
- `schema[].properties[]` - current key: `name`
- `schema[].quality[]` - current key: `name` (optional, problematic)
- `schema[].properties[].quality[]` - current key: `name` (optional, problematic)
- `support[]` - current key: `channel`
- `roles[]` - current key: `role`
- `slaProperties[]` - current key: `property`
- `team[]` - current key: `username`

ODPS:
- `inputPorts[]` - current key: `name`
- `outputPorts[]` - current key: `name`
- `managementPorts[]` - current key: `name`
- `support[]` - current key: `channel`
- `team.members[]` - current key: `username`

**Medium Priority** (Objects that appear in multiple locations):
- `customProperties[]` - current key: `property`
- `authoritativeDefinitions[]` - no current key (has `type` + `url`)

**Low Priority** (Relationship objects):
- `schema[].relationships[]` - identified by `from` + `to` combination
- `schema[].properties[].relationships[]` - identified by `to` (`from` is implicit)


#### Reference syntax

References use a dot-notation anchored by section names and ids. General patterns:

- Same-contract reference to a section item by `id`:
  - `section.<id>`
- Schema property reference by object `id` and property `id`:
  - `schema.<object_id>.properties.<property_id>`
- External contract reference:
  - `<file>#section.<id>` or `<file>#schema.<object_id>.properties.<property_id>`

Where `section` can be: `schema`, `quality`, `sla`, `servers`, `roles`, `support`, `customProperties`.

Notes:
- For foreign keys within schema, use `schema.<object_id>.properties.<property_id>` to reference a property-level element
- Backward-compatible name/path references remain valid where supported by ODCS 3.1; `id` references are additive
- To be referenceable with the `id`-based form, the target item (object, property, quality rule, SLA item, etc.) MUST define an `id`. The `id` field remains optional overall for backward compatibility.

#### Authoring guidance

- Prefer adding `id` to any array item that may be a reference target (e.g., a specific `quality` rule, a `schema` object, an SLA item)
- Do not add `id` to simple leaf fields; add it only to the containing item
- When both `id` and name-based shorthand exist, prefer `id` for long-term stability

#### Naming consistency and migration path

##### Current State

ODCS currently has inconsistent primary key fields across different object types:
- `schema[]` uses `name`
- `servers[]` uses `server`
- `support[]` uses `channel`
- `roles[]` uses `role`
- `slaProperties[]` uses `property`
- `team[]` uses `username`

Additionally, schema elements can have three different name-related fields (`name`, `businessName`, `physicalName`) with unclear semantic boundaries.

##### v3.X.0 Approach (This RFC)

To maintain backward compatibility while enabling stable references:

1. **Add optional `id` field** to all referenceable objects
2. **Keep all existing primary fields** (`name`, `server`, `channel`, `role`, `property`, `username`, etc.)
3. **Keep `businessName` and `physicalName`** for backward compatibility
4. **Clarify semantic roles**:

```yaml
# Recommended pattern for v3.2.0:
schema:
  - id: customer_data          # Optional: Stable technical identifier for references
    name: customers             # Required: Logical name (current primary key)
    businessName: Customers     # Optional: Business-facing display name
    physicalName: cust_tbl_v2   # Optional: Physical implementation name
```

**Semantic guidance for v3.X.0:**
- `id` - Stable technical identifier (optional, recommended for all references)
- `name`/`server`/`channel`/etc. - Logical identifier (required, existing primary keys)
- `businessName` - Business-facing display name (optional)
- `physicalName` - Physical implementation name (optional)

**Recommendation:** Authors should prefer `id`-based references for long-term stability, as primary key fields may change during refactoring.


#### Building the relationships block

The `relationships` block enables three fundamental operations between data contract elements: enforcing structural constraints, documenting relationships, and importing reusable definitions.

##### Relationship types

ODCS | ODPS supports three base relationship types:

| Type | Purpose | `to` field | `from` field | Effect |
|------|---------|------------|--------------|--------|
| `foreignKey` | Enforces referential integrity | Required | Context-dependent* | Structural constraint validation |
| `relatesTo` | Documents semantic relationships | Required | Never used | Informational only (lineage, documentation) |
| `imports` | Imports/transclude external content | Required (import source) | Never used | Content is merged as if locally defined |

*At property level, `from` is implicit. At schema level, `from` is required.

##### Structure

```yaml
relationships:
  - type: foreignKey|relatesTo|imports
    to: <target-reference>           # Always required
    from: <source-reference>         # Only for schema-level foreignKey
    description: <human-readable-text>
    customProperties:
      - property: <name>
        value: <value>
```

##### Field definitions

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | enum | Yes | One of: `foreignKey`, `relatesTo`, `imports`, `definition`[capture business definition] |
| `to` | string or array | Yes | Target element reference. For `foreignKey` and `relatesTo`, this is the related element. For `imports`, this is the source to import from. |
| `from` | string or array | Context-dependent* | Source element reference. Only used for schema-level `foreignKey`. At property level, `from` is implicit. Never used for `relatesTo` or `imports`. |
| `description` | string | No | Human-readable explanation of the relationship |
| `customProperties` | array | No | Additional metadata following standard custom properties structure |

*`from` is only used for schema-level `foreignKey` relationships. It is implicit at property level and never used for `relatesTo` or `imports`.

##### Where relationships can be defined

Relationships can be defined at two levels:

**Property level** (within `schema[].properties[]`):
```yaml
properties:
  - name: customer_id
    id: cust_id
    relationships:
      - type: foreignKey
        to: schema.customers.properties.id  # 'from' is implicit
```

**Schema level** (within `schema[]`):
```yaml
schema:
  - id: orders
    relationships:
      - type: foreignKey
        from: schema.orders.properties.customer_id  # 'from' required at this level
        to: schema.customers.properties.id
```

Additionally, relationships can be used in other sections (e.g., `slaProperties`, `customProperties`) to reference schema elements or external definitions.

##### Type 1: foreignKey (Structural Constraint)

Enforces referential integrity between data elements. Values in the source field must exist in the target field.

**When to use:**
- Primary key to foreign key relationships
- Lookup table validations
- Parent-child hierarchies

**Common patterns:**
```yaml
# Property level (most common)
properties:
  - name: customer_id
    relationships:
      - type: foreignKey
        to: schema.customers.properties.id
        description: "Must reference valid customer"

# Composite keys
properties:
  - name: order_id
    relationships:
      - type: foreignKey
        to: ['schema.orders.properties.tenant_id', 'schema.orders.properties.order_num']
        description: "Composite foreign key"

# Schema level (explicit from/to)
schema:
  - id: orders
    relationships:
      - type: foreignKey
        from: schema.orders.properties.customer_id
        to: schema.customers.properties.id
```

##### Type 2: relatesTo (Semantic Documentation)

Documents relationships for lineage, governance, and business context. Does not enforce constraints or import content—purely informational.

**When to use:**
- Data lineage tracking (derived from, copied from, transformed by)
- Business ↔ technical mapping (implements business definitions)
- Documenting data flows
- Governance ownership
- Semantic relationships (synonyms, equivalents, dependencies)

**Examples:**
```yaml
# Data lineage - derived field
properties:
  - name: monthly_total
    relationships:
      - type: relatesTo
        to: schema.daily_sales.properties.amount
        description: "Derived from: Aggregated SUM of daily transaction amounts"

# Business definition mapping
schema:
  - id: customer_table
    relationships:
      - type: relatesTo
        to: business-glossary.yaml#schema.customer_concept
        description: "Implements the Customer business entity concept"

# Multiple semantic relationships
properties:
  - name: email
    relationships:
      - type: relatesTo
        to: crm-system.yaml#schema.contacts.properties.email
        description: "Copied from CRM system via nightly ETL"

      - type: relatesTo
        to: etl-pipeline.yaml#jobs.data_sync_job
        description: "Transformed by data sync pipeline"

      - type: relatesTo
        to: governance-model.yaml#roles.data_privacy_officer
        description: "Governed by Data Privacy Officer"
```

##### Type 3: imports (Content Transclusion)

Imports definitions from external sources and merges them into the contract as if they were defined locally. Enables DRY (Don't Repeat Yourself) patterns. Must be an allowed structure. 

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

**Examples:**
```yaml
# Import quality rule from shared library
properties:
  - name: email
    relationships:
      - type: imports
        to: common-quality-rules.yaml#quality.email_validation
        description: "Import standard email validation rule from common library"

# Import multiple quality rules
properties:
  - name: customer_id
    relationships:
      - type: imports
        to: common-rules.yaml#quality.standard_id_checks
        description: "Import all standard ID validation rules"

# Import standard audit properties
schema:
  - id: orders
    relationships:
      - type: imports
        to: templates.yaml#properties.audit_fields
        description: "Import standard audit fields (created_at, updated_at, created_by)"
```

##### Validation rules

Implementations SHOULD validate:

1. **Type-specific field requirements:**
   - All relationship types MUST have `to` field
   - Property-level relationships must NOT have `from` field (it's implicit)
   - Schema-level `foreignKey` must have both `from` and `to`
   - `relatesTo` and `imports` never use `from` field at any level

2. **Reference resolution:**
   - Target IDs referenced in `to` must exist
   - Source IDs referenced in `from` (for schema-level foreignKey) must exist
   - Referenced paths must resolve correctly
   - External files must be accessible

3. **Type consistency (for foreignKey):**
   - Single reference: both `from` and `to` must be strings
   - Composite keys: both `from` and `to` must be arrays of equal length

4. **Import validation:**
   - Content referenced in `to` field must exist and be importable
   - Imported content type must be compatible with target location (e.g., quality rule imports into quality section)
   - Circular imports must be detected and prevented

##### Mixed relationship types

A single property can have multiple relationships of different types:

```yaml
properties:
  - name: customer_id
    relationships:
      # Structural constraint
      - type: foreignKey
        to: schema.customers.properties.id
        description: "Must reference valid customer"

      # Document where it came from
      - type: relatesTo
        to: source-system.yaml#schema.orders.properties.cust_no
        description: "Copied from source system customer number field"

      # Import validation rules
      - type: imports
        to: common-rules.yaml#quality.id_format_check
        description: "Import standard ID format validation"
```

##### Backward compatibility with ODCS v3.1.0

The existing `foreignKey` relationship type from ODCS v3.1.0 remains fully supported:

```yaml
# ODCS v3.1.0 syntax (still valid)
relationships:
  - from: users.user_id
    to: accounts.id
    type: foreignKey  # Can be omitted (defaults to foreignKey)

# ODCS v3.2.0 equivalent with explicit type
relationships:
  - type: foreignKey
    from: schema.users_obj.properties.user_id
    to: schema.accounts_obj.properties.id
```

Both name-based shorthand (`users.user_id`) and ID-based fully qualified references (`schema.users_obj.properties.user_id`) are supported during the transition period.

### Examples

Below examples illustrate adding `id` and referencing those IDs across sections. These range from simple property-level references to complex cross-contract scenarios.

#### Example 1: Single Contract with Multiple Tables and Internal References

This example shows a complete retail database contract with multiple tables that reference each other using the new `id`-based reference system. This demonstrates the most common pattern for defining relationships within a single database.

```yaml
version: 1.0.0
kind: DataContract
id: retail-pos-system
status: active
apiVersion: v3.1.0
domain: retail
dataProduct: point-of-sale

description:
  purpose: Core retail database tables for point-of-sale operations
  limitations: Does not include inventory or supply chain data
  usage: Transaction processing and customer analytics

servers:
  - server: retail-postgres-prod
    type: postgres
    host: prod-db.retail.example.com
    port: 5432
    database: retail_db
    schema: public

schema:
  # Table 1: Customers master data
  - id: customers_tbl
    name: customers
    physicalName: customers
    physicalType: table
    businessName: Customer Master Data
    description: Core customer information and contact details
    tags: ['master-data', 'pii']
    properties:
      - id: cust_id_pk
        name: customer_id
        primaryKey: true
        primaryKeyPosition: 1
        businessName: Customer Identifier
        logicalType: integer
        physicalType: int
        required: true
        description: Unique customer identifier
        classification: restricted
        quality:
          - id: cust_id_not_null
            metric: nullValues
            mustBe: 0
            description: Customer ID must never be null
            dimension: completeness
            severity: error

      - id: cust_email
        name: email
        businessName: Email Address
        logicalType: string
        physicalType: varchar(255)
        required: true
        description: Customer email address
        classification: pii
        quality:
          - id: email_format
            metric: pattern
            arguments:
              regex: '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
            description: Email must be valid format
            dimension: validity
            severity: error

      - id: cust_created_at
        name: created_at
        businessName: Account Creation Date
        logicalType: timestamp
        physicalType: timestamp
        required: true
        description: When customer record was created
        classification: public

  # Table 2: Product categories (with self-referencing hierarchy)
  - id: categories_tbl
    name: categories
    physicalName: product_categories
    physicalType: table
    businessName: Product Categories
    description: Hierarchical product category structure
    tags: ['master-data', 'taxonomy']
    properties:
      - id: cat_id_pk
        name: category_id
        primaryKey: true
        primaryKeyPosition: 1
        businessName: Category Identifier
        logicalType: integer
        physicalType: int
        required: true
        description: Unique category identifier

      - id: cat_name
        name: category_name
        businessName: Category Name
        logicalType: string
        physicalType: varchar(100)
        required: true
        description: Display name for category

      - id: cat_parent_fk
        name: parent_category_id
        businessName: Parent Category
        logicalType: integer
        physicalType: int
        required: false
        description: Parent category for hierarchical structure (NULL for top-level)
        relationships:
          # Self-referencing foreign key
          - type: foreignKey
            to: schema.categories_tbl.properties.cat_id_pk
            description: References parent category in same table for hierarchy

  # Table 3: Products
  - id: products_tbl
    name: products
    physicalName: products
    physicalType: table
    businessName: Product Catalog
    description: All products available for sale
    tags: ['master-data', 'catalog']
    properties:
      - id: prod_id_pk
        name: product_id
        primaryKey: true
        primaryKeyPosition: 1
        businessName: Product Identifier
        logicalType: integer
        physicalType: int
        required: true
        description: Unique product identifier

      - id: prod_sku
        name: sku
        businessName: Stock Keeping Unit
        logicalType: string
        physicalType: varchar(50)
        required: true
        unique: true
        description: Unique SKU for inventory tracking

      - id: prod_category_fk
        name: category_id
        businessName: Product Category
        logicalType: integer
        physicalType: int
        required: true
        description: Product category classification
        relationships:
          # Foreign key to categories table
          - type: foreignKey
            to: schema.categories_tbl.properties.cat_id_pk
            description: Product must belong to valid category

      - id: prod_name
        name: product_name
        businessName: Product Name
        logicalType: string
        physicalType: varchar(255)
        required: true
        description: Display name for product

      - id: prod_price
        name: unit_price
        businessName: Unit Price
        logicalType: number
        physicalType: decimal(10,2)
        required: true
        description: Current selling price per unit
        quality:
          - id: price_positive
            metric: invalidValues
            arguments:
              validValues: ['>0']
            description: Price must be greater than zero
            dimension: validity
            severity: error

  # Table 4: Orders
  - id: orders_tbl
    name: orders
    physicalName: orders
    physicalType: table
    businessName: Customer Orders
    description: Order header information
    tags: ['transactional']
    properties:
      - id: ord_id_pk
        name: order_id
        primaryKey: true
        primaryKeyPosition: 1
        businessName: Order Identifier
        logicalType: integer
        physicalType: int
        required: true
        description: Unique order identifier

      - id: ord_customer_fk
        name: customer_id
        businessName: Customer
        logicalType: integer
        physicalType: int
        required: true
        description: Customer who placed the order
        relationships:
          # Foreign key to customers table
          - type: foreignKey
            to: schema.customers_tbl.properties.cust_id_pk
            description: Order must be placed by valid customer

      - id: ord_date
        name: order_date
        businessName: Order Date
        logicalType: date
        physicalType: date
        required: true
        partitioned: true
        partitionKeyPosition: 1
        description: Date order was placed

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
            to: schema.order_items_tbl.properties.item_line_total
            description: Calculated as SUM(line_total) from order_items table
        customProperties:
          - property: calculationLogic
            value: SUM(order_items.line_total) WHERE order_items.order_id = orders.order_id

  # Table 5: Order Items (join/junction table)
  - id: order_items_tbl
    name: order_items
    physicalName: order_items
    physicalType: table
    businessName: Order Line Items
    description: Individual line items for each order
    tags: ['transactional', 'detail']
    # Schema-level composite relationships
    relationships:
      - type: foreignKey
        from: schema.order_items_tbl.properties.item_order_fk
        to: schema.orders_tbl.properties.ord_id_pk
        customProperties:
          - property: description
            value: Each line item must belong to valid order
          - property: cardinality
            value: many-to-one
      - type: foreignKey
        from: schema.order_items_tbl.properties.item_product_fk
        to: schema.products_tbl.properties.prod_id_pk
        customProperties:
          - property: description
            value: Each line item must reference valid product
          - property: cardinality
            value: many-to-one
    properties:
      - id: item_id_pk
        name: order_item_id
        primaryKey: true
        primaryKeyPosition: 1
        businessName: Order Item Identifier
        logicalType: integer
        physicalType: int
        required: true
        description: Unique line item identifier

      - id: item_order_fk
        name: order_id
        businessName: Order Reference
        logicalType: integer
        physicalType: int
        required: true
        description: Reference to parent order

      - id: item_product_fk
        name: product_id
        businessName: Product Reference
        logicalType: integer
        physicalType: int
        required: true
        description: Product being ordered

      - id: item_quantity
        name: quantity
        businessName: Quantity
        logicalType: integer
        physicalType: int
        required: true
        description: Number of units ordered
        quality:
          - id: qty_positive
            metric: invalidValues
            arguments:
              validValues: ['>0']
            description: Quantity must be at least 1
            dimension: validity
            severity: error

      - id: item_unit_price
        name: unit_price
        businessName: Unit Price at Time of Sale
        logicalType: number
        physicalType: decimal(10,2)
        required: true
        description: Price per unit at time of order (may differ from current product price)
        relationships:
          - type: relatesTo
            to: schema.products_tbl.properties.prod_price
            description: Snapshot of product price at order time

      - id: item_line_total
        name: line_total
        businessName: Line Total
        logicalType: number
        physicalType: decimal(12,2)
        required: true
        description: Total for this line item
        relationships:
          # Document calculation within same table
          - type: relatesTo
            to: [schema.order_items_tbl.properties.item_quantity, schema.order_items_tbl.properties.item_unit_price]
            description: Calculated as quantity * unit_price
        customProperties:
          - property: calculationLogic
            value: quantity * unit_price

  # Table 6: Product Reviews
  - id: reviews_tbl
    name: product_reviews
    physicalName: product_reviews
    physicalType: table
    businessName: Product Reviews
    description: Customer reviews and ratings for products
    tags: ['user-generated-content']
    properties:
      - id: rev_id_pk
        name: review_id
        primaryKey: true
        primaryKeyPosition: 1
        businessName: Review Identifier
        logicalType: integer
        physicalType: int
        required: true
        description: Unique review identifier

      - id: rev_customer_fk
        name: customer_id
        businessName: Reviewer
        logicalType: integer
        physicalType: int
        required: true
        description: Customer who wrote the review
        relationships:
          - type: foreignKey
            to: schema.customers_tbl.properties.cust_id_pk
            description: Review must be authored by valid customer

      - id: rev_product_fk
        name: product_id
        businessName: Reviewed Product
        logicalType: integer
        physicalType: int
        required: true
        description: Product being reviewed
        relationships:
          - type: foreignKey
            to: schema.products_tbl.properties.prod_id_pk
            description: Review must be for valid product

      - id: rev_rating
        name: rating
        businessName: Star Rating
        logicalType: integer
        physicalType: int
        required: true
        description: Rating from 1 to 5 stars
        quality:
          - id: rating_range
            metric: enumValues
            arguments:
              validValues: [1, 2, 3, 4, 5]
            description: Rating must be between 1 and 5
            dimension: validity
            severity: error

      - id: rev_text
        name: review_text
        businessName: Review Content
        logicalType: string
        physicalType: text
        required: false
        description: Optional text review content
        classification: public

# SLA Properties
slaProperties:
  # SLO 1: Overall data quality across critical fields
  - id: slo_overall_data_quality
    property: dataQuality
    value: 99.9
    unit: percent
    description: Overall data quality SLO across all critical validation rules
    relationships:
      - type: relatesTo
        to: [
          schema.customers_tbl.properties.cust_id_pk.quality.cust_id_not_null,
          schema.customers_tbl.properties.cust_email.quality.email_format,
          schema.products_tbl.properties.prod_price.quality.price_positive,
          schema.order_items_tbl.properties.item_quantity.quality.qty_positive,
          schema.reviews_tbl.properties.rev_rating.quality.rating_range
        ]
        description: Aggregated quality score must meet 99.9% across these specific quality rules
    driver: operational
    schedule: "0 */6 * * *"
    scheduler: cron

  # SLO 2: Customer data quality (specific to customer table)
  - id: slo_customer_completeness
    property: dataQuality
    value: 100
    unit: percent
    description: Customer critical fields must be 100% complete
    element: customers.customer_id
    relationships:
      - type: relatesTo
        to: schema.customers_tbl.properties.cust_id_pk.quality.cust_id_not_null
        description: Customer ID completeness must be 100% - no nulls allowed
      - type: relatesTo
        to: schema.customers_tbl.properties.cust_email.quality.email_format
        description: All customer emails must pass format validation
    driver: regulatory
    businessImpact: critical
    schedule: "0 * * * *"
    scheduler: cron

  # SLO 3: Product pricing validity
  - id: slo_product_pricing_validity
    property: dataQuality
    value: 100
    unit: percent
    description: All product prices must be valid (positive values)
    element: products.unit_price
    relationships:
      - type: relatesTo
        to: schema.products_tbl.properties.prod_price.quality.price_positive
        description: Tracks the positive price validation rule - zero tolerance for invalid prices
    driver: operational
    businessImpact: critical
    schedule: "0 */4 * * *"
    scheduler: cron

  # SLO 4: Order data integrity
  - id: slo_order_validity
    property: dataQuality
    value: 100
    unit: percent
    description: Order line items must pass all quantity validations
    element: order_items.quantity
    relationships:
      - type: relatesTo
        to: schema.order_items_tbl.properties.item_quantity.quality.qty_positive
        description: All order quantities must be positive integers
    driver: operational
    schedule: "0 */2 * * *"
    scheduler: cron

  # SLO 5: Review ratings validity
  - id: slo_review_ratings_validity
    property: dataQuality
    value: 99.5
    unit: percent
    description: Product reviews must have valid star ratings
    element: product_reviews.rating
    relationships:
      - type: relatesTo
        to: schema.reviews_tbl.properties.rev_rating.quality.rating_range
        description: Monitors enumeration rule ensuring ratings are 1-5 stars only
    driver: operational
    schedule: "0 0 * * *"
    scheduler: cron

  # SLO 6: Referential integrity
  - id: slo_referential_integrity
    property: dataIntegrity
    value: 100
    unit: percent
    description: All foreign key relationships must be valid with no orphaned records
    driver: regulatory
    businessImpact: critical
    schedule: "0 */1 * * *"
    scheduler: cron
    customProperties:
      - property: monitored_relationships
        value: "orders->customers, order_items->orders, order_items->products, reviews->customers, reviews->products"

  # SLO 7: Data freshness for orders
  - id: slo_order_data_freshness
    property: latency
    value: 5
    unit: minutes
    element: orders.order_date
    description: New orders must appear in system within 5 minutes of creation
    driver: operational
    businessImpact: high
    schedule: "*/5 * * * *"
    scheduler: cron

  # SLO 8: Data availability
  - id: slo_system_availability
    property: availability
    value: 99.95
    unit: percent
    description: Database must be available and queryable
    driver: operational
    businessImpact: critical

  # SLO 9: Data retention compliance
  - id: slo_data_retention
    property: retention
    value: 7
    unit: years
    element: orders.order_date
    description: Order data must be retained for 7 years per regulatory requirements
    driver: regulatory
    businessImpact: critical

# Team
team:
  name: retail-data-platform
  description: Retail data platform engineering team
  members:
    - username: jsmith
      role: Data Engineer
      dateIn: "2023-01-15"
    - username: alee
      role: Database Administrator
      dateIn: "2023-03-01"

# Support
support:
  - channel: '#retail-data-support'
    tool: slack
  - channel: retail-data-team
    tool: email
    url: mailto:retail-data-team@example.com

# Custom Properties
customProperties:
  - property: database_version
    value: PostgreSQL 15.2
  - property: backup_schedule
    value: Daily at 02:00 UTC
```

**Key Patterns Demonstrated:**
- **Multiple tables in one contract**: Six related tables (customers, categories, products, orders, order_items, reviews)
- **Property-level foreign keys**: Most common pattern - implicit `from`, e.g., `ord_customer_fk` references `schema.customers_tbl.properties.cust_id_pk`
- **Schema-level foreign keys**: Order_items table uses explicit `from` + `to` syntax at schema level to group related foreign keys
- **Self-referencing relationships**: Categories table has parent-child hierarchy referencing itself
- **Cross-table calculated fields**: Order total documents lineage from order_items line totals
- **Same-table calculations**: Line total references quantity and unit_price from same table
- **SLA monitoring across tables**: Single SLA can monitor quality rules from multiple tables using fully qualified paths
- **Consistent with ODCS patterns**: Uses `physicalType: table`, proper primary key definitions, partitioning, classifications

#### Example 2: Complete E-commerce Contract with Multiple Relationship Types

This example demonstrates a realistic e-commerce data contract using all three relationship types:

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
            to: crm-system.yaml#schema.contacts.properties.email_address
            description: Copied from CRM system via nightly ETL pipeline

      - id: cust_segment
        name: customer_segment
        logicalType: string
        relationships:
          # Link to business definition
          - type: relatesTo
            to: business-glossary.yaml#schema.customer_segmentation.properties.segment_type
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
            to: schema.customers_tbl.properties.cust_id
            description: Must reference valid customer

      - id: order_total
        name: total_amount
        logicalType: number
        relationships:
          # Document calculation lineage
          - type: relatesTo
            to: [schema.order_items_tbl.properties.item_price, schema.order_items_tbl.properties.quantity]
            description: Calculated as SUM(item_price * quantity) from order items
        quality:
          - id: amount_positive
            metric: invalidValues
            arguments:
              validValues: [">0"]

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
            to: schema.orders_tbl.properties.order_id
            description: Must reference valid order

      - id: item_product_id
        name: product_id
        logicalType: string
        relationships:
          # Foreign key to external contract
          - type: foreignKey
            to: product-catalog.yaml#schema.products.properties.product_id
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
        to: [schema.customers_tbl.properties.cust_email.quality.email_validation,
             schema.customers_tbl.properties.cust_segment.quality.valid_segments]
        description: Monitors email validation and segment enumeration rules

  - id: sla_order_integrity
    property: dataQuality
    value: 100
    unit: percent
    relationships:
      - type: relatesTo
        to: schema.orders_tbl.properties.order_id.quality.order_id_not_null
        description: All orders must have valid IDs

customProperties:
  - property: data_governance_owner
    value: customer-data-team
    relationships:
      - type: relatesTo
        to: governance-model.yaml#roles.customer_data_steward
        description: Governed by Customer Data Stewardship team
```

#### Example 2: Shared Quality Rules Library Pattern

Demonstrates reusable quality rule definitions that can be imported across multiple contracts:

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

#### Example 3: Business Glossary to Technical Implementation Mapping

Shows how business definitions can be maintained separately and linked to technical implementations:

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
        quality:
          - id: valid_statuses
            metric: enumValues
            arguments:
              validValues: ['ACTIVE', 'INACTIVE', 'SUSPENDED', 'CLOSED']

  - id: order_concept
    name: Order
    businessName: Sales Order
    description: A request by a customer to purchase products or services
    properties:
      - id: order_value
        name: order_amount
        businessName: Order Value
        description: Total monetary value of the order in base currency
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
        to: business-glossary.yaml#schema.customer_concept
        description: Technical implementation of Customer business entity

    properties:
      - id: sf_cust_id
        name: customer_id__c
        physicalName: customer_id__c
        logicalType: string
        relationships:
          # Link to specific business definition
          - type: relatesTo
            to: business-glossary.yaml#schema.customer_concept.properties.customer_identifier
            description: Implements Customer Identifier business definition

      - id: sf_cust_status
        name: status__c
        physicalName: status__c
        logicalType: string
        relationships:
          # Import business validation rules
          - type: imports
            to: business-glossary.yaml#schema.customer_concept.properties.customer_status.quality.valid_statuses
            description: Use business-defined valid status values

          # Document relationship to business concept
          - type: relatesTo
            to: business-glossary.yaml#schema.customer_concept.properties.customer_status
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
        to: business-glossary.yaml#schema.customer_concept
        description: DWH implementation of Customer business entity

      # Document data lineage from source system
      - type: relatesTo
        to: crm-technical-contract.yaml#schema.sf_customer
        description: Data sourced from Salesforce CRM via ETL pipeline

    properties:
      - id: dwh_cust_key
        name: customer_key
        physicalName: customer_sk
        logicalType: integer
        description: Surrogate key for data warehouse

      - id: dwh_cust_id
        name: customer_id
        physicalName: customer_business_id
        logicalType: string
        relationships:
          # Business definition link
          - type: relatesTo
            to: business-glossary.yaml#schema.customer_concept.properties.customer_identifier
            description: Implements Customer Identifier business definition

          # Technical lineage
          - type: relatesTo
            to: crm-technical-contract.yaml#schema.sf_customer.properties.sf_cust_id
            description: Populated from Salesforce customer_id__c field

      - id: dwh_status
        name: customer_status
        physicalName: current_status
        logicalType: string
        relationships:
          # Import business validation from glossary
          - type: imports
            to: business-glossary.yaml#schema.customer_concept.properties.customer_status.quality.valid_statuses
            description: Use business-defined status enumeration

          # Technical lineage
          - type: relatesTo
            to: crm-technical-contract.yaml#schema.sf_customer.properties.sf_cust_status
            description: Derived from Salesforce status__c with transformation logic
```

#### Example 4: Data Lineage Across Multi-Tier Architecture

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
            to: bronze-transactions.yaml#schema.raw_transactions.properties.raw_txn_id
            description: Converted from string to integer during cleansing

      - id: clean_amount
        name: amount_usd
        logicalType: number
        relationships:
          - type: relatesTo
            to: bronze-transactions.yaml#schema.raw_transactions.properties.raw_amount
            description: Parsed from string, validated, converted to numeric USD
        quality:
          - id: amount_valid
            metric: invalidValues
            arguments:
              validValues: ['>0']
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
            to: silver-transactions.yaml#schema.cleaned_transactions.properties.clean_amount
            description: Aggregated SUM of cleaned transaction amounts by date

          - type: relatesTo
            to: bronze-transactions.yaml#schema.raw_transactions.properties.raw_amount
            description: Ultimate source is raw transaction amounts from bronze layer
        customProperties:
          - property: aggregation_logic
            value: SUM(amount_usd) GROUP BY DATE(transaction_timestamp)
```

#### Example 5: Composite Foreign Keys

Demonstrates multi-column foreign key relationships:

```yaml
dataContractSpecification: 3.2.0
id: multi-tenant-orders
version: 1.0.0

schema:
  - id: tenants_tbl
    name: tenants
    properties:
      - id: tenant_id_pk
        name: tenant_id
        logicalType: string
      - id: tenant_name
        name: name
        logicalType: string

  - id: products_tbl
    name: products
    description: Products are scoped per tenant
    properties:
      - id: prod_tenant_id
        name: tenant_id
        logicalType: string

      - id: prod_id
        name: product_id
        logicalType: string

      - id: prod_name
        name: name
        logicalType: string
    relationships:
      # Composite key: (tenant_id) references (tenants.tenant_id)
      - type: foreignKey
        from: schema.products_tbl.properties.prod_tenant_id
        to: schema.tenants_tbl.properties.tenant_id_pk

  - id: orders_tbl
    name: orders
    properties:
      - id: order_id_pk
        name: order_id
        logicalType: integer

      - id: order_tenant_id
        name: tenant_id
        logicalType: string

      - id: order_product_id
        name: product_id
        logicalType: string
    relationships:
      # Composite foreign key: (tenant_id, product_id) -> products(tenant_id, product_id)
      - type: foreignKey
        from: [schema.orders_tbl.properties.order_tenant_id, schema.orders_tbl.properties.order_product_id]
        to: [schema.products_tbl.properties.prod_tenant_id, schema.products_tbl.properties.prod_id]
        description: Order must reference valid product within same tenant
```

#### Simple Examples: Basic ID-Based References

#### 6) Schema object with `id` and property relationships by `id`

```yaml
schema:
  - id: users_obj
    name: users
    logicalType: object
    properties:
      - name: id
        id: id_fields
        logicalType: integer
      - name: account_id
        logicalType: integer
        id: account_id
        relationships:
          - to: 'my_contract.yaml#schema.accounts_obj.properties.id_field'   # target by object id + property id

  - id: accounts_obj
    name: accounts
    logicalType: object
    properties:
      - name: id
        id: id_field
        logicalType: integer
```

Reference resolution rules for properties when using `schema.<object_id>.properties.<property_id>`:
- The `<object_id>` must match a `schema` item `id`
- The `<property_id>` must match the `id` of a property within that object

#### 2) Schema-level relationship using `id`

```yaml
schema:
  - id: orders_obj
    name: orders
    logicalType: object
    properties:
      - name: customer_id
        id: customer_id
    relationships:
      - from: 'my_contract.yaml#schema.orders_obj.properties.customer_id'
        to:   'my_contract.yaml#schema.customers_obj.properties.id'

  - id: customers_obj
    name: customers
    logicalType: object
    properties:
      - name: id
        id: id
```

#### 3) Quality rule items with `id` and SLA referencing them

Attach `quality` to either objects or properties as usual, but give each rule an `id` so other sections can reference it.

```yaml
schema:
  - id: payments_obj
    name: payments
    logicalType: object
    properties:
      - name: amount
        id: payment_amount
        logicalType: number
        quality:
          - id: dq_amount_not_null
            metric: nullValues
            mustBe: 0
          - id: dq_amount_range
            metric: invalidValues
            arguments:
              validValues: [">=0"]

slaProperties:
  - id: slo_amount_quality
    property: dataQuality
    value: 99.9
    unit: percent
    relationships:
      - type: relatesTo
        to: schema.payments_obj.properties.payment_amount.quality.dq_amount_not_null
        description: "SLA monitors the null value quality check for payment amounts"
```

#### 4) Referencing specific schema items (stable across reordering)

Using `id` on array items like `servers` gives stable references regardless of their order in the array.

```yaml
servers:
  - id: srv_snowflake_prod
    server: snowflake-prod
    type: snowflake
    account: my_account
    database: SALES
    schema: PUBLIC

  - id: srv_bigquery_dev
    server: bq-dev
    type: bigquery
    project: my_project
    dataset: staging

# Elsewhere: reference a specific server from custom properties
customProperties:
  - property: primary_storage_location
    value: snowflake-prod
    relationships:
      - type: relatesTo
        to: servers.srv_snowflake_prod
        description: "Data is physically stored on the production Snowflake server"
```

#### 5) External contract by file + id

```yaml
# Reference a quality rule in another contract by file and fully-qualified schema path
customProperties:
  - property: external_quality_ref
    value: 'https://example.com/data-contract.yaml#schema.schema_id.properties.my_field.quality.dq_global_rule_01'
```

### Validation considerations

Implementations SHOULD validate:
- Uniqueness of `id` within the containing array or object collection
- Existence of referenced `id`
- Property path resolution when using `schema.<object_id>.properties.<property_id>`

### Backward compatibility

- Existing name-based and dot-notation references continue to be supported in ODCS 3.1
- `id` references are optional and additive
- Authors can progressively adopt `id` without breaking existing contracts

### Fully qualified references (requirement)

Even if an element `id` could be made globally unique, ODCS requires all references to use fully qualified, schema-anchored paths for explicitness and readability.

Rules:
- Always start references at `schema.<schema_id>` or `<section>.<id> `and include the full path to the target (e.g., `.properties.<property_id>`, `.quality.<rule_id>`, etc.)
- if only ONE schema/top_parent is present in a data contract - no schema_id is needed
- For external files, include the file and anchor with the fully qualified schema path, e.g., `other.yaml#schema.<schema_id>.properties.<property_id>`

Rationale:
- Makes the reference unambiguous and self-explanatory in context
- Preserves readability by showing where in the schema hierarchy the target resides


## Alternatives

REJECTED:

[RFC-009b](./approved/odcs-v3.1.0/0009b-ref-internal-properties.md) Option A & Option B - JSON Pointers an JSON Paths 


## Decision

> The decision made by the TSC.

## Consequences

> The consequences of this decision.

## References

> Prior art, inspiration, and other references you used to create this based on what's worked well before.

Formerly known as RFC 0009D.

## APPENDIX:

##### v4.0.0 Future Direction (Possible Migration Strategy - Out of Scope of RFC Decision)

To achieve full consistency across ODCS, the following breaking changes are recommended for v4.0.0:

1. **Make `id` required** on all referenceable objects
2. **Standardize naming fields** across all object types:
   - `id` (required) - Stable technical identifier for references
   - `name` (required) - Canonical human-readable name
   - `physicalName` (optional) - Physical implementation name when it differs from logical name

3. **Remove inconsistent fields: (DEPRECATED)** 
   - `businessName` (redundant with `name`)
   - `server` (replaced by `name`)
   - `channel` (replaced by `name`)
   - `role` (replaced by `name`)
   - `property` (replaced by `name`)

```yaml
# Future v4.0.0 pattern:
schema:
  - id: customer_data          # Required: Stable identifier for references
    name: Customers             # Required: The canonical human-readable name
    physicalName: cust_tbl_v2   # Optional: Physical implementation when different

servers:
  - id: prod_snowflake         # Required: Stable identifier
    name: snowflake-prod        # Required: Human-readable name (was "server" field)
    type: snowflake
    physicalName: sf_prod_01    # Optional: Physical server identifier

support:
  - id: primary_slack          # Required: Stable identifier
    name: "#data-contracts"     # Required: Human-readable name (was "channel" field)
    url: https://...
```

**Benefits of v4.0.0 approach:**
- Single stable identifier (`id`) for all references
- One canonical human name (`name`) for all objects
- Clear separation between logical and physical naming
- Consistent structure across all object types
- Simpler mental model: one ID, one name, one physical name

##### Migration Strategy

**v3.2.0 to v4.0.0 migration path:**

1. Start adding `id` fields to objects in v3.2.0 contracts
2. Migrate references to use `id`-based syntax
3. In v4.0.0, transform existing primary keys:
   - `server` → `name`
   - `channel` → `name`
   - `role` → `name`
4. Set `businessName` values to `name` where they differ
5. Make `id` required
6. Remove `businessName`

This provides immediate value from stable `id` references while establishing a clear path to full consistency in v4.0.0.

