# Deprecated flag for schema and properties

> The champion is the person who is primarily supporting the RFC.

Champion: Simon Harrer

## Summary

Add an optional `deprecated` boolean flag to schema objects and properties to indicate when data elements are no longer recommended for use. The flag defaults to `false` when not specified.

## Motivation

Data contracts evolve over time. Fields get replaced, schemas get refactored. Currently there's no standardized way to mark elements as deprecated while maintaining backward compatibility.

**Use cases:**
- Signal to consumers which fields/objects are being phased out
- Enable tooling (linters, validators, IDEs) to warn about deprecated usage
- Track technical debt and plan migrations
- Document lifecycle status for governance

Aligns with industry standards (OpenAPI, JSON Schema) for deprecation patterns.

## Design and examples

### Specification

Add optional `deprecated` boolean field (defaults to `false`) to:
- Schema objects (`schema[]`)
- Properties (`schema[].properties[]` including nested properties)

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `deprecated` | boolean | No | `false` | Indicates this element is deprecated and should not be used in new implementations |

**Guidelines:**
- Deprecated elements remain documented and validated for backward compatibility
- Implementations MAY warn when deprecated elements are used
- Use `description` field to provide migration guidance

### Example 1: Deprecated property with replacement

Deprecate a single field and point to its replacement:

```yaml
schema:
  - name: customers
    logicalType: object
    properties:
      - name: customer_id
        logicalType: integer
        description: "Unique customer identifier"

      - name: email_address
        logicalType: string
        deprecated: true
        description: "DEPRECATED: Use 'primary_email' instead. Will be removed in next major version"

      - name: primary_email
        logicalType: string
        description: "Primary email address for the customer"
```

### Example 2: Deprecated schema object

Deprecate an entire schema object and provide migration guidance:

```yaml
schema:
  - name: legacy_orders
    deprecated: true
    logicalType: object
    description: "DEPRECATED: Use 'orders_v2' instead. Will be removed in next major version. Migration guide: https://docs.example.com/migration/orders-v2"
    properties:
      - name: order_id
        logicalType: integer
        description: "Order identifier"

      - name: order_date
        logicalType: date
        description: "Date order was placed"

  - name: orders_v2
    logicalType: object
    description: "Replacement for legacy_orders table with improved structure"
    properties:
      - name: order_id
        logicalType: integer
        description: "Order identifier"

      - name: created_at
        logicalType: timestamp
        description: "Timestamp when order was created"

      - name: updated_at
        logicalType: timestamp
        description: "Timestamp when order was last updated"
```

### Example 3: Migrating from flat to nested structure

Deprecate a flat field while introducing a structured replacement:

```yaml
schema:
  - name: products
    logicalType: object
    properties:
      - name: product_id
        logicalType: integer
        description: "Unique product identifier"

      - name: product_name
        logicalType: string
        description: "Product name"

      - name: category_id
        logicalType: integer
        deprecated: true
        description: "DEPRECATED: Use 'category' object instead. Will be removed in next major version"

      - name: category
        logicalType: object
        description: "Product category information"
        properties:
          - name: id
            logicalType: integer
            description: "Category identifier"

          - name: name
            logicalType: string
            description: "Category name"
```

## Decision

> The decision made by the TSC.

## Consequences

- nonbreaking change

## References

**Industry standards using boolean deprecation flags:**
- [OpenAPI 3.1.0](https://spec.openapis.org/oas/v3.1.0#fixed-fields-9) - `deprecated: true` on operations, parameters, schemas
- [JSON Schema 2020-12](https://json-schema.org/draft/2020-12/json-schema-validation.html#name-deprecated) - `deprecated: true` annotation
- [GraphQL](https://spec.graphql.org/October2021/#sec--deprecated) - `@deprecated` directive on fields and enum values

**Common pattern:** Boolean flag for machine readability + description field for human-readable migration guidance.
