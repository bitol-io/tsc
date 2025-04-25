# Identify an external contract

> The champion is the person who is primarily supporting the RFC.

Champion: Diego C. & Simon Harrer.

[RFC-0009 Slack](https://data-mesh-learning.slack.com/archives/C08EF0M2FFV)

[Jira Ticket](https://bitol-io.atlassian.net/browse/ODCS-48?atlOrigin=eyJpIjoiNWNkM2M2MGYxZmM5NDYyMmI0NGEyYzY1ZjA2Yzg1MDEiLCJwIjoiaiJ9)

## Summary

> One paragraph explanation of the RFC.
As part of configuring references within the ODCS standard - we need to define how one can identify an external data contract from within a given contract. 


## Motivation

> Why are we doing this? What use cases does it support? What is the expected outcome?
> How does it align with our guiding values?

## Design and examples

> This is the bulk of the RFC.
> Explain the design in enough detail for somebody familiar with data contracts and the standard to understand. This should get into specifics and corner-cases, and include examples of how this is to be used.
> Offer at least two examples, one is minimalist, one is more structured.

### Option A: File-Based ⭐ *Our Recommendation*

A reference follows the following structure:
```yaml
<file><anchor><item-path-within-contract>...
```

Where:

- < file > : Path to the contract file (optional for same-contract references)
- < anchor >: '#' symbol to mark entry into a contract (optional for same-contract)
- < item-path-within-contract >: The defined path the contract reference

To identify an external contract, use one of these formats:

``` yaml
# Same folder as current contract
data-contract-v1.yaml

# Full path
file:///path/to/data-contract-v1.yaml

# URL
https://example.com/data-contract-v1.yaml

# Relative path
../../path/to/data-contract-v1.yaml
```

When referencing an element within the same contract, the file component can be omitted.

The anchor ('#') marks the entry into the data contract structure:

```yaml
# Reference to an external contract
'external-contract.yaml#schema.my-table'

# Reference within the same contract (anchor optional)
'#schema.my-table'
'schema.my-table'

```

If referencing the same schema within the same data contract, the file component, anchor can be omitted. 

```yaml

# Full Reference 
'this-contract.yaml#schema.my-table.my-column'

# equivalent if referenced within the same schema
'schema.my-table.my-column'

```

Reference Implementations:

- OpenAPI Specification: Uses $ref with URLs and fragments (e.g., $ref: 'http://example.com/schemas/user.json#/definitions/user')
- JSON Schema: Uses URI references with fragments (e.g., {"$ref": "definitions.json#/address"})
- HTML/XML: Uses XPath and ID references

Pros:

- Familiarity: Mimics widely understood URI/URL referencing patterns
- Flexibility: Can reference local files, network resources, or within the same document
- Path Traversal: Clear mechanism for drilling down into nested structures
- Tool Support: Many existing parsers understand this format
- Orthogonality: Storage mechanism is separate from identifier mechanism
- Human Readable: Easy to understand where the reference points to


Cons:

- File Dependency: Coupling to file system concepts may be problematic in non-file contexts
- URI Escaping: Special characters in paths require proper escaping
- Path Maintenance: Relative paths can break when files are moved
- Versioning Challenge: No explicit version handling without incorporating it into filenames
- Security Concerns: External URL references may pose security risks without proper validation


- We beleive that using Option A allows for users to leverage outside tooling to implement. 

### Option B: Package-style Reference


A reference follows the following structure:
```yaml
contract:
  - name: "contract_name"
  - version: "contract version"

or 

{"contract name", version="version"}
```

Reference Implementations:

- NPM/Yarn: Package references like "dependency": "package-name@^1.2.3"
- Maven: Uses groupId:artifactId format
- Python pip: Requirements syntax package==1.0.0
- Terraform: Module references with source and version attributes
- Semantic Import Versioning: Go modules use semver in import paths

Pros:

- Version Explicit: Version is a first-class property in the reference
- Registry Compatible: Works well with centralized contract registries
- Transport Agnostic: No assumptions about storage mechanism (file, URL, etc.)
- Namespace Support: Can incorporate organization/space concepts easily
- Explicit Dependency: Clear declaration of dependency relationships
- Resolution Flexibility: Implementation can decide how to resolve the contract

Cons:

- Resolution Ambiguity: Doesn't specify how to locate the contract without additional lookup logic
- Path Complexity: Requires separate mechanism for referencing within a contract
- Less Direct: More indirection compared to direct file references
- Registry Dependency: Often requires a central registry or resolution system
- Implementation Overhead: Needs more sophisticated tooling to handle resolution

### Option C: Custom Contract Wide "context" 💀 _moving_to_new_rfc_

```yaml
context:  # top-level context the data contract 
- name: this_contract # contract name to reference
  alias: myc  # contract alias - used within the contract
  description: 'This is the current contract'
  version:  2.3 # contract version number
```

Reference Implementations:

- XML Namespaces: Uses xmlns attributes to define context
- GraphQL: Schema definitions with types and namespaces
- Protobuf: Package declarations and import statements
- SQL: Schema qualifications for table references
- JSON-LD: Uses @context to define semantic mappings

Example:

```yaml
context:  # top-level context the data contract 
- name: this_contract # contract name to reference
  alias: myc  # contract alias - used within the contract
  description: 'This is the current contract'
  version:  2.3 # contract version number

# Usage

ref: myc#this_schema.that_column.property


## OPTION A&C
ref: myc#this_schema.that_column.property
ref: https://my_contract.yaml#this_schema.that_column.property
```


Pros:

Rich Metadata: Can include additional metadata beyond just identification
Alias Support: Shorthand references via aliases improve readability
Self-Documentation: Contract includes its own identity information
Namespace Control: Explicit namespace management
Internal References: Clear mechanism for referencing within the same contract
Semantic Clarity: Explicit metadata about the contract's purpose and content

Cons:

Format Specific: Less aligned with general-purpose reference mechanisms
External Reference Challenge: Needs additional syntax for referencing external contracts
Verbosity: More verbose for simple references
Indirection: Adds a layer of indirection that needs to be understood
Custom Parsing: May require custom parsing logic rather than using existing URI parsers
Duplication Risk: Context information may be duplicated across systems

## Alternatives

> Option C - is an alaternative point which can be added on later. However in terms of resolution, you would still resolve the context to a url - then expand.
> Option C allows us to reuse items across the data contract - it could be extremely powerful (macros/globals) - it could its own RFC depending on how it is built

> Option B - could be better lent to the ODPS where we are _grouping_ data contracts together into a given data product. 

## Decision

Requested Decisions:
- Do we like Option A or Option B

## Consequences

> The consequences of this decision.

## References

> Prior art, inspiration, and other references you used to create this based on what's worked well before.
