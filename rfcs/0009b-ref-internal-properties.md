# Identify an element within a contract

> The champion is the person who is primarily supporting the RFC.

Champion: Diego C. & Simon Harrer.

[RFC-0009 Slack](https://data-mesh-learning.slack.com/archives/C08EF0M2FFV)

[Jira Ticket](https://bitol-io.atlassian.net/browse/ODCS-48?atlOrigin=eyJpIjoiNWNkM2M2MGYxZmM5NDYyMmI0NGEyYzY1ZjA2Yzg1MDEiLCJwIjoiaiJ9)

## Summary

We have a need to be able to identify the specific items within a data contract. 


## Motivation

The decision required in this RFC is: 

1) The structure of the reference (Options A vs Option B)
2) Allowed Shortcuts (YES / NO ) - Option D

## Design and examples

In order to reference an item inside a data contract - we need to be able to identify its location within the contract. There are three proposed solutions. 


### OPTION A - JSON Pointers (Recommendation)

JSON Pointer (RFC 6901) is a string syntax for identifying a specific value within a JSON document. It uses a sequence of reference tokens separated by / characters to navigate through the document hierarchy. For arrays, numeric indices are used; for objects, property names serve as identifiers.

Refs:
- [JSON POINTER](https://datatracker.ietf.org/doc/html/rfc6901)

<<add description of what a JSON pointer is>>

``` examples
schema:
  - name: job_post
    physicalName: job_post_1_0_0
    description: Contains the information related to a single job posting by a single organisation.
    properties:                     # /schema/0/properties
      - name: id                    # /schema/0/properties/0/name
        required: true              # /schema/0/properties/0/required
        logicalType: string         # /schema/0/properties/0/logicalType
```

Pros:

- Standardized: Well-defined in RFC 9535 with clear semantics
- Simple Implementation: Easy to implement and parse
- Deterministic: References are unambiguous and exact
- Language Support: Libraries available in most programming languages
- Invariant to Structure Changes: Using indices ensures references remain valid even if property names change

Cons:

- Poor Readability: Index-based references like `/schema/0/properties/0` are not intuitive to humans
- Fragility to Reordering: Array indices break if elements are reordered
- Maintenance Burden: Requires updating references when structural changes occur
- Limited Semantics: No query capabilities beyond direct addressing

Reference Implementations:

- OpenAPI: Uses JSON Pointer in its $ref implementation
- JSON Schema: Uses JSON Pointer for referencing definitions
- Kubernetes: API paths follow similar index-based hierarchical patterns



### OPTION B - JSON Paths

Refs:
- [JSON PATH](https://datatracker.ietf.org/doc/rfc9535/)

JSONPath extends beyond JSON Pointer by adding query capabilities. It allows for more expressive selection of elements through filter expressions, wildcards, and recursive descent operators.

``` examples

# Example reference:
# $.schema[?(@.name=="job_post")].properties[?(@.name=="id")]

schema:
  - name: job_post
    physicalName: job_post_1_0_0
    description: Contains the information related to a single job posting by a single organisation.
    properties:                     # $.schema[?(@.name=="job_post")].properties
      - name: id                    # $.schema[?(@.name=="job_post")].properties[?(@.name=="id")]
        required: true              # $.schema[?(@.name=="job_post")].properties[?(@.name=="id")].required
        logicalType: string         # $.schema[?(@.name=="job_post")].properties[?(@.name=="id")].logicalType
```

Pros:

- Query Capabilities: Supports filtering, conditionals, and expressions
- Name-Based References: Can reference by properties, not just indices
- Robustness: More resilient to reordering of array elements
- Wildcard Support: Can select multiple elements matching criteria
- Expressive Power: Can create complex, targeted selections

Cons:

- Complex Syntax: More verbose and harder to read, especially for nested structures
- Performance Overhead: Query evaluation is more resource-intensive than direct pointer addressing
- Multiple Standards: Several competing JSONPath implementations with subtle differences
- Learning Curve: More difficult to master than simpler pointer notation
- Verbosity: Expressions like [?(@.name=="job_post")] are cumbersome for frequent use

Reference Implementations:

- Jayway JSONPath: Popular Java implementation of JSONPath
- jq: Command-line JSON processor uses similar query expressions
- PowerShell: ConvertFrom-Json supports JSONPath-like queries
- MongoDB: Query syntax influenced by JSONPath concepts
- Splunk: Search queries incorporate JSONPath-like features


### OPTION D - Allowed "shortcuts"

While we will conform to a single standard there are a set of shorthand notation options which we would like to support. 


Shorthand 1: `table.column`  - reference one column
Shorthand 2: `table.(column 1, column2)` - reference two or more columns on the same table

These shorthand options are only available for properties, but we see this notation already in our existing contract.  Existing "shortcuts" are already used within the data contract (e.g. SLAs):
- table.column

``` examples
schema:
  - name: job_post
    physicalName: job_post_1_0_0
    description: Contains the information related to a single job posting by a single organisation.
    properties:                     # /schema/0/properties
      - name: id                    # /job_post.id
        required: true              # /job_post.id/required
        logicalType: string         # /job_post.id/logicalType
```


=> Tools should first try the shortcut, and then fallback to using JSON Pointer (recommended solution)

These shortcuts also allow for foreign key designations (RFC-13) 

```mock implementation
relationships:
  - type: foreignKey  # Optional, defaults to "include"
    to: 'target-table.(target-column-1, target-column-2)'
    from: 'source-table.(source-column-1, source-column-2)'
```

 
## Alternatives

### OPTION C - JSON Pointers with "shortcuts"

REJECTED: Creating a New Standard 


This approach introduces a customized reference mechanism that maintains the hierarchical structure of JSON Pointer but uses name-based identifiers instead of indices. This would require identifier uniqueness as a validation requirement in data contracts (possibly through a linter).

For resolution - it may be possible to quickly convert Option C to Option A format for tools to reference. 


``` examples

# Example reference:
# schema.job_post.properties.id

schema:
  - name: job_post
    physicalName: job_post_1_0_0
    description: Contains the information related to a single job posting by a single organisation.
    properties:                     # schema.job_post.properties
      - name: id                    # schema.job_post.properties.id
        required: true              # schema.job_post.properties.id.required
        logicalType: string         # schema.job_post.properties.id.logicalType
```

Existing "shortcuts" are already used within the data contract (e.g. SLAs):
- table.column


For this to work, we define a mapping table determining which fields to use as identifiers for each type of collection:

When resolving references, the system follows this hierarchy:

1) Use the name field to determine element identity
2) Fall back to the mapping table if name is not present
3) If not found in the mapping table, use index-based reference

The default mapping rules are::

|Collection|Identifier Field|
|:-----|:-----|
schema | name|
properties | name |
quality | id (RFC0012)
roles | role|
channels | channel |
servers | server|

Pros:

- Human Readability: References like `schema.job_post.properties.id` are intuitive
- Semantic Clarity: References carry meaning through names rather than positions
- Reordering Resilience: Not affected by array element reordering
- Conciseness: Shorter than equivalent JSONPath expressions
- Customizable: Mapping table allows tailored identity resolution for different object types
- Familiar Syntax: Similar to programming language object references

Cons:

- Non-Standard: Custom solution not directly aligned with existing standards
- Ambiguity Risk: Requires unique names within collections to avoid conflicts
- Implementation Complexity: Requires custom resolver logic and mapping tables
- Consistency Challenges: May lead to inconsistent implementations across tools
- Migration Difficulty: If identifier field values change, all references need updating

Reference Implementations:

GraphQL: Uses named field resolution in a similar manner
XPath: Named node selection with custom axes
YAML Anchors: Uses named references for similar purposes
CSS Selectors: Uses attribute-based targeting similar to mapping rules
Terraform: HCL language uses similar named reference patterns




## Decision

2025-05-20 - Option D - table.column shortcut approved for schema properties only
2025-05-20 - Option B | Option A - neither approved. Json Pointers were too brittle given the array structure. JSON paths were too complex in wirtting them effectively. TSC decision was to continue looking for options. 

## Consequences

> The consequences of this decision.

## References

> Prior art, inspiration, and other references you used to create this based on what's worked well before.
