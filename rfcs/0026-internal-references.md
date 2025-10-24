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
- `id` SHOULD be snake_case and technical in nature for readability and stability
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

##### v4.0.0 Future Direction

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
| `type` | enum | Yes | One of: `foreignKey`, `relatesTo`, `imports` |
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

Below examples illustrate adding `id` and referencing those IDs across sections.

#### 1) Schema object with `id` and property relationships by `id`

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
