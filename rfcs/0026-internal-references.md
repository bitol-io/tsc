# Identify an element within a contract

> The champion is the person who is primarily supporting the RFC.

Champion: Diego C. 

[Slack](https://data-mesh-learning.slack.com/archives/C08EF0M2FFV)

Jira: *replace this text with the link to the dedicated Jira ticket*.

## Summary

We have a need to be able to identify the specific items within a data contract. 


## Motivation

> Why are we doing this? What use cases does it support? What is the expected outcome?
> How does it align with our guiding values?

## Design and examples

Original work and design was completed under RFC-0009b - however at the TSC 
2025-05-20 - Option B | Option A - neither approved. Json Pointers were too brittle given the array structure. JSON paths were too complex in writing them effectively. TSC decision was to continue looking for options. 

This RFC continues the search for the correct internal reference structure. 

### Option: ID-based internal references (proposed)

Add an optional `id` field to most top-level and repeated objects in an ODCS contract. Internal references will use these `id` values instead of JSON Pointers/Paths to uniquely identify target elements.

Key goals:
- Keep authoring simple and human-friendly
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
- Using optional IDs avoids forcing changes on all contracts while enabling robust internal links where needed
- Array re-ordering does not break references

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
    property: generalAvailability
    value: 99.9
    unit: percent
    references:
        - to: my_contract.yaml#schema.payments_obj.properties.payment_amount.quality.dq_amount_not_null'
          type: import_reference
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

# Elsewhere: store a pointer to a specific property under a schema object
customProperties:
  - property: primary_storage_location
    references:
        - to: my_contract.yaml#servers.srv_snowflake_prod'
          type: business_reference
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