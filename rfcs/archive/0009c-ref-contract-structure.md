# Reference Internal and External Structure

> The champion is the person who is primarily supporting the RFC.

Champion: Diego C. & Simon Harrer.

[RFC-0009 Slack](https://data-mesh-learning.slack.com/archives/C08EF0M2FFV)

[Jira Ticket](https://bitol-io.atlassian.net/browse/ODCS-48?atlOrigin=eyJpIjoiNWNkM2M2MGYxZmM5NDYyMmI0NGEyYzY1ZjA2Yzg1MDEiLCJwIjoiaiJ9)

# Decision 

The initial use of relationships was moved to RFC-13 under foreign keys using the relationship block until a concrete decision can be made on RFC-0009b internal contract structure. 

## Summary

> One paragraph explanation of the RFC.
As part of configuring references within the ODCS standard - we need to define the internal structure that someone uses to best define the data contract. 

## Motivation

The ODCS standard currently lacks a formal mechanism for reusing elements within and across data contracts.

Current users face the following key problems:

- Maintenance burden
- Excessive contract size (repeated elements)
- Limited contract relationships
- Fragmented governance

Our goal is to allow for references between and within data contracts. We believe that this will allow for improved reusability within the data contract. A [proposed solution](#main-proposed-solution) is located below. 

If we are successful, users should be able to:

- reference a dataset element in another element within a data contract 
- reference a dataset element outside a data contract
- maintain single sources of truth for common elements like quality rules, stakeholders, and schemas
- express complex data relationships that reflect their actual architecture

We need to be 

## Design and examples


## OPTION A: to/from
*Standard Format*

|Key | UX Label | Required | Description|
|:----|:----|:----|:----|
|relationships| relationships | optional| Array. A list of elements which to reference|
|relationships.type | type | optional | One of `foreignKey`, `include`. Default is `include`|
|relationships.to | to | optional | A reference to a contract object. Cannot be used in conjunction with ref. Must follow correct [reference structure](#basic-format) |
| relationships.from| from | optional |A reference to a contract object.  Cannot be used in conjunction with ref. Must follow correct [reference structure](#basic-format)|

From is optional within property values. 

Example usage:

```yaml
# foreign keys 
relationships:
  - type: foreignKey  # Optional, defaults to "include"
    to: '#/target-table/properties/target-column'
    from: '#/source-table/properties/source-column'

# simple inclusion
relationships:
  - ref: '#/my-table/properties/my-other-column'

# shorthand notation (proposed - only for properties)
relationships:
  - type: foreignKey  # Optional, defaults to "include"
    to: 'target-table.target-column'
    from: 'source-table.source-column' # if at property level - from is optional or maybe even forbidden at property level

# shorthand notation (proposed - second option, composite keys)
relationships:
  - type: foreignKey  # Optional, defaults to "include"
    to: 'target-table.(target-column-1, target-column-2)'
    from: 'source-table.(source-column-1, source-column-2)'

```
Pros:

- Explicit Directionality: Clear distinction between source and target
- Relationship Typing: Supports different relationship semantics (foreign key vs inclusion)
- Composite Key Support: Can represent multi-column relationships
- Familiar Pattern: Similar to relational database foreign key definitions
- Extensible: Additional relationship types can be added as needed

Cons:

- Verbosity: More verbose than simple references
- Redundancy: Both sides of relationship must be specified
- Schema Complexity: Adds additional nesting level
- Learning Curve: Users must understand from/to directionality

## Option B - relationship block and ref shorthand

*Shorthand Format*

For simpler usage, a $ref shorthand can be used to automatically include the referenced object:

``` yaml

# Standard Relationship Object
relationships:
  - type: include # one of (foreignKey, include)
    ref: "../somefolder/contract_v2.yaml#schema.my-table.properties.my-other-column"


# Equivalent Shorthand
$ref: "../somefolder/contract_v2.yaml#schema.my-table.properties.my-other-column"

```

Pros:

- Simplicity: More concise for simple inclusion cases
- Standards Alignment: Similar to JSON Schema and OpenAPI $ref pattern
- Familiarity: Developers familiar with $ref notation can adopt quickly
- Less Redundancy: Single reference line instead of relationships block
- Convenience: Shorthand notation reduces typing burden

Cons:

- Limited Semantics: Harder to represent complex relationships like foreign keys
- Dual Syntax: Creates two ways to express similar concepts
- Implicit Assumptions: Type of relationship is assumed rather than explicit
- Extensibility Challenges: Adding new relationship types complicates shorthand
- Composite Key Limitations: Less clear how to handle multi-column relationships

### OPTION C - Typed References with Bidirectional Support

This proposed option combines elements from both A and B, offering flexibility while maintaining semantic clarity.

```
# Foreign key relationship (full syntax)
relationships:
  - type: foreignKey
    reference: "../somefolder/contract_v2.yaml#schema.target-table.properties.target-column"
    referencedBy: "schema.source-table.properties.source-column"  # Optional, can be inferred at property level
    cardinality: "one-to-many"  # Optional (one-to-one, one-to-many, many-to-one, many-to-many)
    integrity: "cascade"  # Optional (restrict, cascade, setNull, noAction)

# Simple inclusion (full syntax)
relationships:
  - type: include  
    reference: "../somefolder/contract_v2.yaml#schema.my-table.properties.my-other-column"

# Simple inclusion (shorthand)
$include: "../somefolder/contract_v2.yaml#schema.my-table.properties.my-other-column"

# Foreign key (shorthand with composite keys)
$foreignKey: 
  reference: "schema.target-table.properties.(col1, col2)"
  cardinality: "one-to-many"
```

Pros:

- Semantic Clarity: Explicit relationship types with appropriate attributes
- Simplified Naming: reference and referencedBy are more intuitive than to/from
- Cardinality Support: Explicit cardinality declaration for relationship modeling
- Referential Integrity: Can specify actions for integrity enforcement
- Flexible Shorthand: Typed shortcuts ($include, $foreignKey) maintain semantics
- Self-Documenting: More explicit about relationship meaning

Cons:

- More Complex Schema: Additional fields increase schema complexity
- New Syntax: Requires learning modified syntax and shorthand prefixes
- Implementation Effort: Tools need to understand additional semantics
- Standard Deviation: Further from common $ref standard pattern

## Alternatives

> Rejected alternative solutions and the reasons why.

## Decision

> The decision made by the TSC.

## Consequences

> The consequences of this decision.

## References

> Prior art, inspiration, and other references you used to create this based on what's worked well before.
