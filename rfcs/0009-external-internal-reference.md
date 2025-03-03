# External & Internal References 

Champion: Todd Nemanich.

## Summary

> One paragraph explanation of the RFC.

This RFC proposes to introduce internal and external relationships within the ODCS standard. 

This feature will allow data contract authors to create references between elements within a single contract as well as across different contracts. By implementing a standardized referencing system, we can significantly reduce duplication, improve maintainability, and enable more sophisticated contract relationships. These references will support use cases ranging from simple reuse of quality rules to complex cross-contract data lineage tracking, ultimately making data contracts more powerful and manageable as their complexity and ecosystem grow.

## Motivation

> Why are we doing this? What use cases does it support? What is the expected outcome?
> How does it align with our guiding values?

The ODCS standard currently lacks a formal mechanism for reusing elements within and across data contracts.
Current users face the following key problems:

- Maintenance burden
- Excessive contract size (repeated elements)
- Limited contract relationships
- Fragmented governance

Our goal is to allow for references between and within data contracts. We believe that this will allow for improved reusability within the data contract.

If we are successful, users should be able to:

- reference a dataset element in another element within a data contract 
- reference a dataset element outside a data contract
- maintain single sources of truth for common elements like quality rules, stakeholders, and schemas
- express complex data relationships that reflect their actual architecture

## Design and examples

> This is the bulk of the RFC.
> Explain the design in enough detail for somebody familiar with data contracts and the standard to understand. This should get into specifics and corner-cases, and include examples of how this is to be used.
> Offer at least two examples, one is minimalist, one is more structured.

We have identified four main use-cases which need to be demonstrated for each of the proposed solutions, namely:

* Internal reference from dataset element to another contract element (i.e. to DQ rule or to a business definition that is used multiple times within the same contract)
* Internal reference from a non-dataset element (i.e. servers->stakeholders for support)
* External reference from a dataset element (i.e. data subset or the description of the technical heritage of a dataset element aka lineage or to a business definition within another contract)
* External reference from non-dataset element (i.e. SLA -> observability store)

Other possible use cases noted:

* foreign keys


### Identified Hurdles
-> There needs to be a standardized way to be able to identify a given object or property within a data contract so it can be referenced. 

Proposed Options:
 - implicit ID using `name` field on objects (or username etc.), however may not be unique across all objects ina  data contract. 
 - add `id` field on objects to allow for referencing
 - add `customProperty` on fields for which we want reference. 


### Reference Solution Process
Both proposed solutions follow this general resolution process:

1) Identify the reference syntax in the contract
2) Determine if the reference is internal or external

3) For external references:

* Locate the external contract
* Validate access permissions
* Load the referenced contract

4) Resolve the path to the specific element
5) Apply the appropriate action based on the reference



### SOLUTION 1: Open API Standard 

A proposed solution mimics the existing [OpenAPI standard](https://learn.openapis.org/referencing/overview.html), with the main difference being the addition of a `ref` field to the `schema` object.

**Assumptions**
* All objects that need to be referenced have unique paths within their contract
* External contracts are accessible and compatible with the reference syntax
* References are validated at contract validation time
* Circular references are detected and prevented

**Overview**
The OpenAPI approach uses JSON Pointer syntax to specify references:

Internal references: #/path/to/element
External references: external-contract.yaml#/path/to/element

Implementing this solution would mimic that syntax within the ODCS standard.

**PRO**
- being an open standard users are comfortable with it 
- widely implemented and supported


**CON**
- duplication of references, limited reusability 
- using index e.g. `/schema/0/properties` could lead to inconsistent references
- Potential for path brittleness if structure changes
- less mantic information

#### Examples 

1) Internal reference from dataset element to another contract element (i.e. to DQ rule)

``` yaml
schema:
- name: users
  properties:
    - name: first_name
      businessName: User's First Name
      logicalType: string
      physicalType: varchar(26)
      quality:
      - description: This column should not contain null values
        dimension: completeness
        severity: error
        rule: nullCheck
        businessImpact: operational
        id: q160512f7
    - name: last_name
      businessName: User's last name
      logicalType: string
      physicalType: varchar(26)
      quality:
      - ref: "#/schema/0/properties/0/quality/0"
        # This references the quality rule from first_name
```
2) Internal reference from a non-dataset element (i.e. servers->stakeholders for support)

``` YAML
stakeholders:
  - name: data_team
    email: data-team@company.com
    id: support-team-1
    
servers:
  - server: my-azure
    type: azure
    location: az://my-data-location
    format: parquet
    support:
      ref: "#/stakeholders/0"
      # This references the data team stakeholder for support
```

3) External reference from a dataset element (i.e. data subset)
```yaml
schema:
- name: users
  properties:
  - name: user_profile
    ref: "common-data-contract.yaml#/schema/0"
    # This references a user profile schema from another contract
```

4) External reference from non-dataset element (i.e. SLA -> observability store)

```yaml
sla:
  monitoring:
    ref: "observability-contract.yaml#/monitoring/dataQuality"
    # This references an external observability store for SLA monitoring
```

### SOLUTION 2: JSON-LD 'CONTEXT' inspired

A proposed solution which takes inspiration from [JSON-LD / Linked Data](https://w3c.github.io/json-ld-syntax) concepts - specifically the use of [`context`](https://w3c.github.io/json-ld-syntax/#the-context) and [`IRI`](https://w3c.github.io/json-ld-syntax/#iris)


**Assumptions**
This solution assumes the following:
- Each element within a data contract has a globally unique identifier which can be used for reference
- each element has a field which can be used as the specific identified (e.g. `name` or has a given id e.g. `quality.id` (RFC-0014))
- Context definitions have contract-wide or element-specific scope

**Overview**

The solution is composed of a high-lever section, `context` and schema-level sections `context` which define the internal references. 

A reference is composed of a `context` which defines:
    - the term to be used within the specific contract
    - the base location of the external reference 
    - additional information regarding the external reference 
    OR
    - the reference and it's location

A context is scoped to the appropriate level within the data contract. 

E.G. A context defined at the schema level will be applicable for the entire schema, whereas a context defined at the object level will be applicable only to that object.

| Key                              | UX label                 | Required | Description                                                                                                                                                             |
|----------------------------------|--------------------------|----------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
context | object | No | Object. Reference related context information. The level where context is defined creates an allowed scope for that context to apply.|
context.name | string | No | The `term` used to reference this specific context |
context.uri | string | No | the external path or location of the given reference. If None, the current contract is assumed as the location. |
context.description| string | No | Description of the given context. |
context.ref | string | No | The object or entity wishing to reference. |
context.type | Literal | No | `embed`,`foreignKey`,`reference`. Default is `embed`.|


The proposed will support the following types:
| Type | Description | Action | embed | Include the referenced content directly | Copy the referenced element and validate it fits the schema | foreignKey | Create a foreign key relationship | Validate the relationship and add appropriate metadata | reference | Create a logical reference without embedding | Add metadata about the reference without copying content |

(These types could be common across either solution)


**Examples**


*Defining a Top Level Context* 
```YAML
# Data Contract 

context:  # top-level context defining where some_external_file is located.
- name: some_external_file  # Term used within this contract
   uri: 'location_of_file/that_is_unique' # URI location of the file 
   description: 'location of another contract reference' # additional       information of this reference

```

*Using a context to reference a field*
```YAML
context:  # top-level context defining where some_external_file is located.
- name: some_external_file
   uri: 'location_of_file/that_is_unique'
   description: 'location of another contract reference'

schema:
- name: users
  properties:
    - name: age
      context: # The actual reference itself
      - ref: some_external_file.age.birth_year # imports specific field  based on its URI

# EQUIVALENT TO:

schema:
- name: users
  properties:
    - name: age
      context: # The actual reference itself
      - ref: location_of_file/that_is_unique/age/#birth_year # imports specific field  based on its URI
```

*Usage for foreign Keys*

```YAML
schema:
- name: users
  properties:
  - name: id
    type: integer
    primaryKey: true
  - name: supervisor_id
    type: integer
    context:
    - ref: id # same schema
      type: foreignKey
      
- name: posts
  properties:
  - name: id
    type: integer
    primaryKey: true
  - name: user_id
    type: integer
    context:
    - ref: users.id # other schema
      type: foreignKey
```

*Use Case Examples*

1) Internal reference from dataset element to another contract element (i.e. to DQ rule)

```YAML

# This solution is more declarative and creates a context with uri being None, which defaults to the current contract. The second option, does not define a context - which again, assumes the current contract when we implement a reference 


context:  # top-level context defining where some_external_file is located.
- name: this_contract
  uri: None # an empty URI assumes the current contract
  description: 'This is the current contract'

schema:
- name: users
  properties:
    - name: first_name
      businessName: User's First Name
      logicalType: string
      physicalType: varchar(26)
      quality:
      - description: This column should not contain null values
        dimension: completeness
        severity: error
        rule: nullCheck
        businessImpact: operational
        id: q160512f7
    - name: last_name
      businessName: User's last name
      logicalType: string
      physicalType: varchar(26)
      quality:
      - description: Reference no-null values quality rule 
        context:
        - ref: this_contract.users.first_name.quality.q160512f7

## Equivalent Solution (without defining top-level context)

schema:
- name: users
  properties:
    - name: first_name
      businessName: User's First Name
      logicalType: string
      physicalType: varchar(26)
      quality:
      - description: This column should not contain null values
        dimension: completeness
        severity: error
        rule: nullCheck
        businessImpact: operational
        id: q160512f7
    - name: last_name
      businessName: User's last name
      logicalType: string
      physicalType: varchar(26)
      quality:
      - description: Reference no-null values quality rule  
        context:
        - ref: users.first_name.quality.q160512f7

            # since we did not define a context (e.g. there is no `users` context) we default to the current contract as reference
```

2) Internal reference from a non-dataset element (i.e. servers->stakeholders for support)

```YAML
support:
  - channel: my_custom_channel # Simple Slack communication channel
    url: https://aidaug.slack.com/archives/C05UZRSBKLY

servers:
  - server: my-azure
    type: azure
    location: az://my-data-location
    format: parquet
    context:
        - ref: support.my_custom_channel
```

3) External reference from a dataset element (i.e. data subset)
```YAML


schema:
- name: account
  properties:
  - name: account_number
    type: integer
    primaryKey: true
  - name: account_type
    type: integer
    primaryKey: true
 
- name: users
  context:  # defining context here as a single scoped item
  - name: common
    uri: ./my_company/shared/common_data_contract/
    description: 'This is the current contract'
    ref: common.users

```
4) External reference from non-dataset element (i.e. SLA -> observability store)

```YAML
# External reference from non-dataset element (i.e. SLA -> observability store)

sla:
  context:
  - name: observability
    uri: ./my_company/shared/observability_contract/
    description: 'Company-wide observability stack configuration'
  
  metrics:
    - name: freshness
      threshold: 24h
      context:
      - ref: observability.dataQuality.monitors.freshness
        type: reference
  
  alerting:
    channel: slack
    context:
    - ref: observability.alertChannels.dataQualityAlerts
      type: embed
```

**PRO**
- easily readable 
- context can be re-used across same data contract
- provides more semantic meaning to references
- flexible scoping, enabling localized reference patterns without affecting the entire document.

**CON**
- decoupling of paths (split location vs item in contract)
- implementation logic could be more complex
- less familiarity vs openAPI specification

## Implementation Considerations
*Validation Rules*
Both solutions require validation of references:

1) Reference Existence: Referenced elements must exist
2) Type Compatibility: Referenced elements must be compatible with the referencing context
3) Circular References: Detect and prevent circular references
4) Access Control: Check that the user has access to referenced contracts

*Error Handling* 

Implementation should handle these error cases:

1) Missing Reference: Provide clear error message with reference details
2) Inaccessible External Contract: Indicate access issues with troubleshooting info
3) Circular References: Detect and prevent with clear cycle information
4) Incompatible Types: Explain the type mismatch
5) Broken References: Detect when referenced elements have been removed or changed

## Alternatives

> Rejected alternative solutions and the reasons why.

## Decision

> The decision made by the TSC.

## Consequences

> The consequences of this decision.

## References

> Prior art, inspiration, and other references you used to create this based on what's worked well before.


## Outstanding Questions
QQ > How do we deal with external access/auth considerations references? 
QQ > How do we deal across different data contract versions?
