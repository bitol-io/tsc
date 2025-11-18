# RFC-0026b: Internal References Using Foreign Keys

> The champion is the person who is primarily supporting the RFC.

Champion: Diego C.

[Slack](https://data-mesh-learning.slack.com/archives/C08EF0M2FFV)

Jira: *replace this text with the link to the dedicated Jira ticket*.

## Summary

Define a `relationships` block structure to establish foreign key relationships between elements within and across data contracts using the `id`-based reference system introduced in RFC-0026a and RFC-0013. This expands on the relationship block to specify the formal reference format and its interaction with shorthand notation. 

## Motivation

Data contracts need a formal way to express structural constraints between data elements. Foreign key relationships:
- Enforce referential integrity constraints
- Document parent-child relationships
- Define lookup table validations
- Support composite keys
- Enable database relationship modeling

This RFC expands the foundational relationship structure, starting with `foreignKey` as the base relationship type and adding the formal references as required. 

## Prerequisites

This RFC depends on RFC-0026a (reference-id) which introduces the `id` field for stable references.

## Design

### Building the relationships block

The `relationships` block enables foreign key constraints between data contract elements.

#### Relationship type: foreignKey

ODCS | ODPS supports the `foreignKey` relationship type:

| Type | Purpose | `to` field | `from` field | Effect |
|------|---------|------------|--------------|--------|
| `foreignKey` | Enforces referential integrity | Required | Context-dependent* | Structural constraint validation |

*At property level, `from` is implicit. At schema level, `from` is required.

#### Structure

```yaml
relationships:
  - type: foreignKey
    to: <target-reference>           # Always required
    from: <source-reference>         # Only for schema-level foreignKey
    description: <human-readable-text>
    customProperties:
      - property: <name>
        value: <value>
```

#### Field definitions

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | enum | Yes | Must be: `foreignKey` |
| `to` | string or array | Yes | Target element reference |
| `from` | string or array | Context-dependent* | Source element reference. Only used for schema-level `foreignKey`. At property level, `from` is implicit. |
| `description` | string | No | Human-readable explanation of the relationship |
| `customProperties` | array | No | Additional metadata following standard custom properties structure |

*`from` is only used for schema-level `foreignKey` relationships. It is implicit at property level.

#### Where relationships can be defined

Relationships can be defined at two levels:

**Property level** (within `schema[].properties[]`):
```yaml
properties:
  - name: customer_id
    id: cust_id
    relationships:
      - type: foreignKey
        to: schema/customers/properties/id  # 'from' is implicit
```

**Schema level** (within `schema[]`):
```yaml
schema:
  - id: orders
    relationships:
      - type: foreignKey
        from: schema/orders/properties/customer_id  # 'from' required at this level
        to: schema/customers/properties/id
```

Additionally, relationships can be used in other sections (e.g., `slaProperties`, `customProperties`) to reference schema elements.

### Type: foreignKey (Structural Constraint)

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
        to: schema/customers/properties/id
        description: "Must reference valid customer"

# Composite keys
properties:
  - name: order_id
    relationships:
      - type: foreignKey
        to: ['schema/orders/properties/tenant_id', 'schema/orders/properties/order_num']
        description: "Composite foreign key"

# Schema level (explicit from/to)
schema:
  - id: orders
    relationships:
      - type: foreignKey
        from: schema/orders/properties/customer_id
        to: schema/customers/properties/id
```

### Validation rules

Implementations SHOULD validate:

1. **Type-specific field requirements:**
   - All relationship types MUST have `to` field
   - Property-level relationships must NOT have `from` field (it's implicit)
   - Schema-level `foreignKey` must have both `from` and `to`

2. **Reference resolution:**
   - Target IDs referenced in `to` must exist
   - Source IDs referenced in `from` (for schema-level foreignKey) must exist
   - Referenced paths must resolve correctly
   - External files must be accessible

3. **Type consistency (for foreignKey):**
   - Single reference: both `from` and `to` must be strings
   - Composite keys: both `from` and `to` must be arrays of equal length

### Reference Notation: Shorthand vs Fully Qualified

ODCS supports two reference notations for identifying elements in relationships, introduced in RFC-0013 and expanded in RFC-0026a:

#### Shorthand Notation (RFC-0013)

Uses the `name` field with dot notation for concise, human-readable references:

**Format:** `<schema_name>.<property_name>`

**Characteristics:**
- Uses the `name` field (required in ODCS)
- Dot-separated path: `users.account_number`
- Can be nested: `accounts.address.street`
- Concise and easy to read
- May break if `name` fields are renamed
- Established in RFC-0013

**When to use:**
- Quick references in simple contracts
- During initial development
- When names are stable and unlikely to change
- For improved readability in straightforward scenarios

#### Fully Qualified Notation (RFC-0026a/0026b)

Uses the `id` field with slash notation for stable, refactor-safe references:

**Format:** `schema/<schema_id>/properties/<property_id>`

**Characteristics:**
- Uses the `id` field (optional, recommended for references)
- Slash-separated path: `schema/users_obj/properties/account_id_field`
- Stable across renames and refactoring
- Resilient to array reordering
- More verbose but explicit
- Established in RFC-0026a

**When to use:**
- Long-lived production contracts
- Complex contracts with many references
- When refactoring is expected
- Cross-contract references
- SLA monitoring and quality rule references

#### Comparison Example

Both notations can reference the same elements:

```yaml
schema:
  - id: users_obj                    # ID for fully qualified references
    name: users                       # Name for shorthand references
    properties:
      - id: user_id_field             # ID for fully qualified references
        name: id                      # Name for shorthand references
        logicalType: integer

      - id: account_number_field      # ID for fully qualified references
        name: account_number          # Name for shorthand references
        logicalType: string
        relationships:
          # Shorthand notation (uses name)
          - type: foreignKey
            to: accounts.account_number
            description: "Shorthand reference using name fields"

          # Fully qualified notation (uses id)
          - type: foreignKey
            to: schema/accounts_obj/properties/account_number_field
            description: "Fully qualified reference using id fields"
```

#### External References

Both notations support external contract references:

```yaml
relationships:
  # Shorthand with external file
  - type: foreignKey
    to: customer-contract.yaml#customers.customer_id
    description: "Shorthand external reference"

  # Fully qualified with external file
  - type: foreignKey
    to: customer-contract.yaml#/schema/customers_tbl/properties/cust_id_pk
    description: "Fully qualified external reference"
```

#### Nested Property References

Both notations support nested structures:

```yaml
# Shorthand for nested properties
relationships:
  - type: foreignKey
    to: accounts.address.street

# Fully qualified for nested properties
relationships:
  - type: foreignKey
    to: schema/accounts_obj/properties/address_field/properties/street_field
```

### Backward compatibility with ODCS v3.1.0

The existing `foreignKey` relationship type from ODCS v3.1.0 (RFC-0013) remains fully supported:

```yaml
# ODCS v3.1.0 syntax (still valid) - Shorthand notation
relationships:
  - from: users.user_id
    to: accounts.id
    type: foreignKey  # Can be omitted (defaults to foreignKey)

# ODCS v3.2.0 with explicit type - Shorthand notation
relationships:
  - type: foreignKey
    from: users.user_id
    to: accounts.id

# ODCS v3.2.0 with fully qualified notation - ID-based
relationships:
  - type: foreignKey
    from: schema/users_obj/properties/user_id
    to: schema/accounts_obj/properties/id
```

Both name-based shorthand (`users.user_id`) and ID-based fully qualified references (`schema/users_obj/properties/user_id`) are supported. Teams can choose based on their needs:

- **Shorthand**: Better for readability, simpler contracts, stable schemas
- **Fully Qualified**: Better for stability, complex contracts, production systems

## Examples

### Example 1: Single Contract with Multiple Tables and Internal References

This example shows a complete retail database contract with multiple tables that reference each other using foreign keys.

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

      - id: cust_email
        name: email
        businessName: Email Address
        logicalType: string
        physicalType: varchar(255)
        required: true
        description: Customer email address
        classification: pii

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
            to: schema/categories_tbl/properties/cat_id_pk
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
            to: schema/categories_tbl/properties/cat_id_pk
            description: Product must belong to valid category

      - id: prod_name
        name: product_name
        businessName: Product Name
        logicalType: string
        physicalType: varchar(255)
        required: true
        description: Display name for product

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
            to: schema/customers_tbl/properties/cust_id_pk
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
        from: schema/order_items_tbl/properties/item_order_fk
        to: schema/orders_tbl/properties/ord_id_pk
        customProperties:
          - property: description
            value: Each line item must belong to valid order
          - property: cardinality
            value: many-to-one
      - type: foreignKey
        from: schema/order_items_tbl/properties/item_product_fk
        to: schema/products_tbl/properties/prod_id_pk
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
            to: schema/customers_tbl/properties/cust_id_pk
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
            to: schema/products_tbl/properties/prod_id_pk
            description: Review must be for valid product
```

**Key Patterns Demonstrated:**
- **Multiple tables in one contract**: Six related tables (customers, categories, products, orders, order_items, reviews)
- **Property-level foreign keys**: Most common pattern - implicit `from`, e.g., `ord_customer_fk` references `schema/customers_tbl/properties/cust_id_pk`
- **Schema-level foreign keys**: Order_items table uses explicit `from` + `to` syntax at schema level to group related foreign keys
- **Self-referencing relationships**: Categories table has parent-child hierarchy referencing itself
- **Consistent with ODCS patterns**: Uses `physicalType: table`, proper primary key definitions, partitioning, classifications

### Example 2: Shorthand vs Fully Qualified Notation Comparison

This example demonstrates the same relationships expressed using both shorthand and fully qualified notation. It shows when each approach is appropriate and how they coexist.

```yaml
dataContractSpecification: 3.2.0
id: user-accounts-comparison
version: 1.0.0
title: User Accounts - Notation Comparison

schema:
  # Users table
  - id: users_tbl                     # ID for fully qualified references
    name: users                        # Name for shorthand references
    logicalType: object
    description: User account information
    properties:
      - id: user_id_pk                # ID for fully qualified references
        name: id                       # Name for shorthand references
        logicalType: integer
        primaryKey: true
        description: Unique user identifier

      - id: user_username_field       # ID for fully qualified references
        name: username                 # Name for shorthand references
        logicalType: string
        required: true
        description: User's login username

      - id: user_account_num_fk       # ID for fully qualified references
        name: account_number           # Name for shorthand references
        logicalType: string
        required: true
        description: Reference to account
        relationships:
          # SHORTHAND NOTATION - concise, uses name
          - type: foreignKey
            to: accounts.account_number
            description: "Shorthand: Account reference using name fields"

      - id: user_primary_email_fk     # ID for fully qualified references
        name: primary_email_id         # Name for shorthand references
        logicalType: integer
        required: false
        description: User's primary email address
        relationships:
          # FULLY QUALIFIED NOTATION - stable, uses id
          - type: foreignKey
            to: schema/emails_tbl/properties/email_id_pk
            description: "Fully qualified: Email reference using id fields (stable)"

  # Accounts table
  - id: accounts_tbl                  # ID for fully qualified references
    name: accounts                     # Name for shorthand references
    logicalType: object
    description: Account information
    properties:
      - id: acct_id_pk                # ID for fully qualified references
        name: id                       # Name for shorthand references
        logicalType: integer
        primaryKey: true
        description: Unique account identifier

      - id: acct_number_field         # ID for fully qualified references
        name: account_number           # Name for shorthand references
        logicalType: string
        unique: true
        required: true
        description: Account number (business key)

      - id: acct_type_field           # ID for fully qualified references
        name: account_type             # Name for shorthand references
        logicalType: string
        required: true
        description: Type of account

  # Emails table
  - id: emails_tbl                    # ID for fully qualified references
    name: emails                       # Name for shorthand references
    logicalType: object
    description: Email addresses
    properties:
      - id: email_id_pk               # ID for fully qualified references
        name: id                       # Name for shorthand references
        logicalType: integer
        primaryKey: true
        description: Unique email identifier

      - id: email_address_field       # ID for fully qualified references
        name: email_address            # Name for shorthand references
        logicalType: string
        required: true
        description: Email address value

      - id: email_user_fk             # ID for fully qualified references
        name: user_id                  # Name for shorthand references
        logicalType: integer
        required: true
        description: User who owns this email
        relationships:
          # SHORTHAND NOTATION at property level
          - type: foreignKey
            to: users.id
            description: "Shorthand: Belongs to user"

  # Orders table - demonstrates schema-level relationships
  - id: orders_tbl                    # ID for fully qualified references
    name: orders                       # Name for shorthand references
    logicalType: object
    description: Customer orders
    # Schema-level relationships using BOTH notations
    relationships:
      # SHORTHAND NOTATION - simpler syntax
      - type: foreignKey
        from: orders.user_id
        to: users.id
        description: "Shorthand: Order placed by user"

      # FULLY QUALIFIED NOTATION - stable syntax
      - type: foreignKey
        from: schema/orders_tbl/properties/order_account_fk
        to: schema/accounts_tbl/properties/acct_id_pk
        description: "Fully qualified: Order billed to account (stable)"

    properties:
      - id: order_id_pk               # ID for fully qualified references
        name: id                       # Name for shorthand references
        logicalType: integer
        primaryKey: true
        description: Unique order identifier

      - id: order_user_fk             # ID for fully qualified references
        name: user_id                  # Name for shorthand references
        logicalType: integer
        required: true
        description: User who placed the order

      - id: order_account_fk          # ID for fully qualified references
        name: account_id               # Name for shorthand references
        logicalType: integer
        required: true
        description: Account to bill

      - id: order_total_field         # ID for fully qualified references
        name: total_amount             # Name for shorthand references
        logicalType: number
        required: true
        description: Order total
```

**When Shorthand Was Used:**
- `users.account_number` → `accounts.account_number`
- `orders.user_id` → `users.id`
- `emails.user_id` → `users.id`

**Advantages:**
- ✓ Concise and readable
- ✓ Quick to write
- ✓ Familiar to developers
- ✓ Good for stable, simple schemas

**Risks:**
- ✗ Breaks if `name` fields are renamed
- ✗ Less explicit about structure
- ✗ May be ambiguous in complex schemas

**When Fully Qualified Was Used:**
- `schema/users_tbl/properties/user_primary_email_fk` → `schema/emails_tbl/properties/email_id_pk`
- `schema/orders_tbl/properties/order_account_fk` → `schema/accounts_tbl/properties/acct_id_pk`

**Advantages:**
- ✓ Stable across refactoring
- ✓ Survives `name` field changes
- ✓ Explicit and unambiguous
- ✓ Better for production systems
- ✓ Good for cross-contract references

**Risks:**
- ✗ More verbose
- ✗ Requires `id` fields to be defined
- ✗ Less familiar to new users

**Recommendation:**
- Use **shorthand** for: Simple contracts, prototypes, stable internal references
- Use **fully qualified** for: Production contracts, cross-contract references, SLA monitoring, quality rule references, long-lived systems

### Example 3: Composite Foreign Keys

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
        from: schema/products_tbl/properties/prod_tenant_id
        to: schema/tenants_tbl/properties/tenant_id_pk

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
        from: [schema/orders_tbl/properties/order_tenant_id, schema/orders_tbl/properties/order_product_id]
        to: [schema/products_tbl/properties/prod_tenant_id, schema/products_tbl/properties/prod_id]
        description: Order must reference valid product within same tenant
```

### Example 3: Simple ID-Based Property References

```yaml
schema:
  - id: users_obj
    name: users
    logicalType: object
    properties:
      - name: id
        id: user_id_field
        logicalType: integer
      - name: account_id
        id: account_id_field
        logicalType: integer
        relationships:
          - type: foreignKey
            to: schema/accounts_obj/properties/account_id_field

  - id: accounts_obj
    name: accounts
    logicalType: object
    properties:
      - name: id
        id: account_id_field
        logicalType: integer
```

### Example 4: External Contract Reference

```yaml
schema:
  - id: order_items_tbl
    name: order_items
    logicalType: object
    description: Line items for orders
    properties:
      - id: item_product_id
        name: product_id
        logicalType: string
        relationships:
          # Foreign key to external contract
          - type: foreignKey
            to: product-catalog.yaml#/schema/products/properties/product_id
            description: Must reference valid product from catalog
```

## Alternatives

Alternative relationship structures were considered but rejected in favor of this explicit, typed approach that supports future extensibility with additional relationship types (see RFC-0031 and RFC-0032).

## Decision

> The decision made by the TSC.

APPROVED by TSC on Tue Nov 18. However changed to '/' notation for the fully qualified reference vs that of dot notation. Also added a section areound the shorthand vs full notation


## Consequences

> The consequences of this decision.

## References

> Prior art, inspiration, and other references you used to create this based on what's worked well before.

Formerly part of RFC 0026.
