# RFC-0026a: Adding ID Field for Stable References

> The champion is the person who is primarily supporting the RFC.

Champion: Diego C.

[Slack](https://data-mesh-learning.slack.com/archives/C08EF0M2FFV)

Jira: *replace this text with the link to the dedicated Jira ticket*.

## Summary

Add an optional `id` field to referenceable objects within ODCS and ODPS contracts to enable stable, human-readable references that are resilient to name changes and array reordering.

## Motivation

ODCS data contracts and ODPS need a way to uniquely identify elements for referencing purposes. Current primary key fields (`name`, `server`, `channel`, `role`, etc.) are inconsistent across object types and may change during refactoring.

Adding stable identifiers enables:
- Robust references that survive name changes
- References that don't break when arrays are reordered
- Foundation for relationship definitions (foreign keys, lineage, imports)
- Consistent reference mechanism across all object types

## Design

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
- UUIDs are allowed for `id` values
- `id` SHOULD be stable across versions to preserve referential integrity
- An id cannot contain the following objects or symbols: '.', '#', '/', '\', \@\, '!', '%', '&', '^'

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

**Objects with Optional Name Fields**

| Object Type | Location | Name Field Status | Notes |
|-------------|----------|------------------|-------|
| **Quality rules** | `schema[].quality[]` or `schema[].properties[].quality[]` | `name` (optional) | Also has `metric` field; currently identified by position in array |
| **AuthoritativeDefinition** | Multiple locations | No name field | Has `type` and `url` (both required) |
| **TeamMember** | `team[]` | `name` (optional) | Primary key is `username` (required) |

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

#### Reference syntax (proposed - see RFC-0026b)

References use a slash-based path notation anchored by section names and ids. General patterns:

- Same-contract reference to a section item by `id`:
  - `section/<id>`
- Schema property reference by object `id` and property `id`:
  - `schema/<object_id>/properties/<property_id>`
- External contract reference:
  - `<file>#/section/<id>` or `<file>#/schema/<object_id>/properties/<property_id>`

Where `section` can be: `schema`, `quality`, `sla`, `servers`, `roles`, `support`, `customProperties`.

Notes:
- For references within schema, use `schema/<object_id>/properties/<property_id>` to reference a property-level element
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

### Examples

#### Example 1: Comprehensive Data Quality Monitoring with Stable ID References

This example demonstrates the core value proposition of `id` fields: **stable references that survive refactoring, reordering, and name changes**. It shows a real-world scenario where SLA monitoring depends on specific quality rules, and how `id` fields keep these references intact even as the contract evolves.

**Initial Contract Version (v1.0.0)**
```yaml
dataContractSpecification: 3.1.0
id: customer-analytics
version: 1.0.0
title: Customer Analytics Database

schema:
  - id: customers_tbl
    name: customers
    physicalName: dim_customers
    physicalType: table
    description: Customer dimension table
    properties:
      - id: pk_customer_id
        name: customer_id
        primaryKey: true
        logicalType: integer
        physicalType: bigint
        required: true
        quality:
          - id: dq_cust_id_not_null
            name: customer_id_completeness
            metric: nullValues
            mustBe: 0
            description: Customer ID must never be null
            dimension: completeness
            severity: error

      - id: fld_email_address
        name: email
        logicalType: string
        physicalType: varchar(255)
        required: true
        classification: pii
        quality:
          - id: dq_email_format
            name: email_format_validation
            metric: pattern
            arguments:
              regex: '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
            description: Email must match valid format
            dimension: validity
            severity: error

          - id: dq_email_not_null
            name: email_completeness
            metric: nullValues
            mustBe: 0
            description: Email must not be null
            dimension: completeness
            severity: error

      - id: fld_signup_date
        name: signup_date
        logicalType: date
        physicalType: date
        required: true
        quality:
          - id: dq_signup_not_future
            name: signup_date_validity
            metric: invalidValues
            arguments:
              validValues: ['<=NOW()']
            description: Signup date cannot be in the future
            dimension: validity
            severity: error

      - id: fld_customer_status
        name: status
        logicalType: string
        physicalType: varchar(20)
        required: true
        quality:
          - id: dq_status_enum
            name: status_enumeration
            metric: enumValues
            arguments:
              validValues: ['ACTIVE', 'INACTIVE', 'SUSPENDED', 'CHURNED']
            description: Status must be one of valid values
            dimension: validity
            severity: error

  - id: transactions_tbl
    name: transactions
    physicalName: fact_transactions
    physicalType: table
    description: Transaction fact table
    properties:
      - id: pk_transaction_id
        name: transaction_id
        primaryKey: true
        logicalType: string
        physicalType: varchar(50)
        required: true
        quality:
          - id: dq_txn_id_format
            name: transaction_id_format
            metric: pattern
            arguments:
              regex: '^TXN-[0-9]{10}$'
            description: Transaction ID must match format TXN-XXXXXXXXXX
            dimension: validity
            severity: error

      - id: fld_amount
        name: amount
        logicalType: number
        physicalType: decimal(15,2)
        required: true
        quality:
          - id: dq_amount_positive
            name: amount_positivity
            metric: invalidValues
            arguments:
              validValues: ['>0']
            description: Transaction amount must be positive
            dimension: validity
            severity: error

# SLA Properties that reference specific quality rules using stable IDs
slaProperties:
  - id: slo_critical_customer_data_quality
    property: dataQuality
    value: 100
    unit: percent
    description: Critical customer fields must have 100% quality
    driver: regulatory
    businessImpact: critical
    schedule: "0 * * * *"
    scheduler: cron
    customProperties:
      - property: monitored_rules
        value: |
          This SLA monitors the following quality rules using stable ID references:
          - dq_cust_id_not_null: Customer ID completeness
          - dq_email_format: Email format validation
          - dq_email_not_null: Email completeness
          - dq_status_enum: Status enumeration

  - id: slo_transaction_data_quality
    property: dataQuality
    value: 99.5
    unit: percent
    description: Transaction data must meet quality thresholds
    driver: operational
    schedule: "0 */4 * * *"
    scheduler: cron
    customProperties:
      - property: monitored_rules
        value: |
          References quality rules:
          - dq_txn_id_format: Transaction ID format
          - dq_amount_positive: Amount positivity
```

**Evolved Contract Version (v2.0.0) - After Refactoring**

Now imagine the team needs to:
1. **Rename fields** for clarity (email → email_address, status → customer_status)
2. **Reorder properties** alphabetically for better organization
3. **Add new quality rules** and reorder existing ones
4. **Rename quality rules** for naming consistency

Without `id` fields, name-based references would break. With `id` fields, all references remain stable:

```yaml
dataContractSpecification: 3.2.0
id: customer-analytics
version: 2.0.0  # Major version bump due to refactoring
title: Customer Analytics Database

schema:
  - id: customers_tbl  # ID unchanged - stable reference
    name: customers
    physicalName: dim_customers_v2  # Physical name changed
    physicalType: table
    description: Customer dimension table with enhanced tracking
    properties:
      # Properties now alphabetically ordered - different array positions
      - id: fld_customer_status  # ID unchanged - reference still works!
        name: customer_status    # NAME CHANGED: was "status"
        logicalType: string
        physicalType: varchar(20)
        required: true
        quality:
          # Quality rules reordered - new rules added first
          - id: dq_status_not_null
            name: customer_status_completeness
            metric: nullValues
            mustBe: 0
            description: Customer status must not be null
            dimension: completeness
            severity: error

          - id: dq_status_enum  # ID unchanged - SLA reference still works!
            name: customer_status_enumeration  # NAME CHANGED: was "status_enumeration"
            metric: enumValues
            arguments:
              validValues: ['ACTIVE', 'INACTIVE', 'SUSPENDED', 'CHURNED', 'PENDING']  # Value added
            description: Status must be one of valid values
            dimension: validity
            severity: error

      - id: pk_customer_id  # ID unchanged
        name: customer_id
        primaryKey: true
        logicalType: integer
        physicalType: bigint
        required: true
        quality:
          - id: dq_cust_id_not_null  # ID unchanged
            name: customer_identifier_completeness  # NAME CHANGED
            metric: nullValues
            mustBe: 0
            description: Customer ID must never be null
            dimension: completeness
            severity: error

      - id: fld_email_address  # ID unchanged
        name: email_address  # NAME CHANGED: was "email"
        logicalType: string
        physicalType: varchar(320)  # Spec changed to RFC 5321 max length
        required: true
        classification: pii
        quality:
          # New quality rule added at beginning
          - id: dq_email_uniqueness
            name: email_uniqueness_check
            metric: duplicateValues
            mustBe: 0
            description: Email addresses must be unique
            dimension: uniqueness
            severity: warning

          - id: dq_email_not_null  # ID unchanged
            name: email_address_completeness  # NAME CHANGED
            metric: nullValues
            mustBe: 0
            description: Email must not be null
            dimension: completeness
            severity: error

          - id: dq_email_format  # ID unchanged
            name: email_address_format_validation  # NAME CHANGED
            metric: pattern
            arguments:
              regex: '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
            description: Email must match valid RFC 5322 format
            dimension: validity
            severity: error

      - id: fld_signup_date  # ID unchanged
        name: signup_date
        logicalType: date
        physicalType: date
        required: true
        quality:
          - id: dq_signup_not_future  # ID unchanged
            name: signup_date_temporal_validity  # NAME CHANGED
            metric: invalidValues
            arguments:
              validValues: ['<=NOW()']
            description: Signup date cannot be in the future
            dimension: validity
            severity: error

  - id: transactions_tbl  # ID unchanged
    name: transactions
    physicalName: fact_transactions_v2
    physicalType: table
    description: Transaction fact table
    properties:
      - id: fld_amount  # ID unchanged - array position changed
        name: amount
        logicalType: number
        physicalType: decimal(15,2)
        required: true
        quality:
          - id: dq_amount_positive  # ID unchanged
            name: transaction_amount_positivity_check  # NAME CHANGED
            metric: invalidValues
            arguments:
              validValues: ['>0']
            description: Transaction amount must be positive
            dimension: validity
            severity: error

      - id: pk_transaction_id  # ID unchanged - array position changed
        name: transaction_id
        primaryKey: true
        logicalType: string
        physicalType: varchar(50)
        required: true
        quality:
          - id: dq_txn_id_format  # ID unchanged
            name: transaction_identifier_format_validation  # NAME CHANGED
            metric: pattern
            arguments:
              regex: '^TXN-[0-9]{10}$'
            description: Transaction ID must match format TXN-XXXXXXXXXX
            dimension: validity
            severity: error

# SLA Properties remain UNCHANGED - references still work via stable IDs!
slaProperties:
  - id: slo_critical_customer_data_quality
    property: dataQuality
    value: 100
    unit: percent
    description: Critical customer fields must have 100% quality
    driver: regulatory
    businessImpact: critical
    schedule: "0 * * * *"
    scheduler: cron
    customProperties:
      - property: monitored_rules
        value: |
          This SLA monitors the following quality rules using stable ID references:
          - dq_cust_id_not_null: Customer ID completeness ✓ STILL WORKS
          - dq_email_format: Email format validation ✓ STILL WORKS
          - dq_email_not_null: Email completeness ✓ STILL WORKS
          - dq_status_enum: Status enumeration ✓ STILL WORKS

  - id: slo_transaction_data_quality
    property: dataQuality
    value: 99.5
    unit: percent
    description: Transaction data must meet quality thresholds
    driver: operational
    schedule: "0 */4 * * *"
    scheduler: cron
    customProperties:
      - property: monitored_rules
        value: |
          References quality rules:
          - dq_txn_id_format: Transaction ID format ✓ STILL WORKS
          - dq_amount_positive: Amount positivity ✓ STILL WORKS
```

**Key Benefits Demonstrated:**

1. **Resilient to Naming Changes**
   - `email` → `email_address`: References using `fld_email_address` ID still work
   - `status` → `customer_status`: References using `fld_customer_status` ID still work
   - Quality rule names all changed, but ID-based references remain valid

2. **Resilient to Array Reordering**
   - Properties reordered alphabetically (customer_status is now first instead of last)
   - Quality rules reordered (new rules added at beginning)
   - Transaction properties reordered
   - All references continue to resolve correctly via stable IDs

3. **Resilient to Schema Evolution**
   - New quality rules added
   - Physical names changed
   - Data types updated
   - SLA monitoring configurations remain untouched and functional

4. **Clean Reference Syntax**
   - Future RFCs can reference these elements using stable IDs:
   - `schema/customers_tbl/properties/fld_email_address/quality/dq_email_format`
   - `schema/transactions_tbl/properties/fld_amount/quality/dq_amount_positive`
   - These paths remain valid across contract versions

**Contrast with Name-Based References (Without IDs):**

If we had used name-based references instead:
- Renaming `email` → `email_address` would break all references to that field
- Renaming quality rules would break SLA monitoring configurations
- Array reordering could break positional references
- Refactoring would require coordinated updates across all referencing contracts

**Conclusion:**

The `id` field provides the stability needed for long-lived data contracts that evolve over time. It decouples the technical reference identifier from the human-readable names and physical implementation details, enabling safe refactoring and maintenance.

#### Example 2: Schema with ID-based references

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

  - id: accounts_obj
    name: accounts
    logicalType: object
    properties:
      - name: id
        id: account_id_field
        logicalType: integer
```

#### Example 3: Referencing specific items (stable across reordering)

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
```

#### Example 4: Quality rule items with `id`

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
```

### Validation considerations

Implementations SHOULD validate:
- Uniqueness of `id` within the containing array or object collection
- Existence of referenced `id`
- Property path resolution when using `schema/<object_id>/properties/<property_id>`

### Backward compatibility

- Existing name-based and dot-notation references continue to be supported in ODCS 3.1
- `id` references are optional and additive
- Authors can progressively adopt `id` without breaking existing contracts

### Fully qualified references (requirement)

Even if an element `id` could be made globally unique, ODCS requires all references to use fully qualified, schema-anchored paths for explicitness and readability.

Rules:
- Always start references at `schema/<schema_id>` or `section/<id>` and include the full path to the target (e.g., `/properties/<property_id>`, `/quality/<rule_id>`, etc.)
- If only ONE schema/top_parent is present in a data contract - no schema_id is needed
- For external files, include the file and anchor with the fully qualified schema path, e.g., `other.yaml#/schema/<schema_id>/properties/<property_id>`

Rationale:
- Makes the reference unambiguous and self-explanatory in context
- Preserves readability by showing where in the schema hierarchy the target resides

## Alternatives

REJECTED:

[RFC-009b](./approved/odcs-v3.1.0/0009b-ref-internal-properties.md) Option A & Option B - JSON Pointers and JSON Paths

## Decision

> The decision made by the TSC.

APPROVED - TSC - NOV 18 2025

## Consequences

> The consequences of this decision.

Allow the addition of an optional `id` field to be used. 

## References

> Prior art, inspiration, and other references you used to create this based on what's worked well before.

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
