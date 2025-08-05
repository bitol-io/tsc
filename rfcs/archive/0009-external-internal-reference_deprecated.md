# External & Internal References 

Champion: Diego C. & Simon Harrer.


## THIS RFC WAS split into RFC-0009a, RFC-009b, RFC-009c 

## Summary

> One paragraph explanation of the RFC.

This RFC proposes to introduce internal and external relationships within the ODCS standard. 

This feature will allow data contract authors to create references between elements within a single contract as well as across different contracts. By implementing a standardized referencing system, we can significantly reduce duplication, improve maintainability, and enable more sophisticated contract relationships. These references will support use cases ranging from simple reuse of quality rules to complex cross-contract data lineage tracking, ultimately making data contracts more powerful and manageable as their complexity and ecosystem grow.

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

## Design and examples

We have identified four main use-cases which need to be demonstrated for each of the proposed solutions, namely:

* Internal reference from dataset element to another contract element (i.e. to DQ rule or to a business definition that is used multiple times within the same contract)
* Internal reference from a non-dataset element (i.e. servers->stakeholders for support)
* External reference from a dataset element (i.e. data subset or the description of the technical heritage of a dataset element aka lineage or to a business definition within another contract)
* External reference from non-dataset element (i.e. SLA -> observability store)

Other possible use cases noted:

* foreign keys


### Identified Hurdles
-> There needs to be a standardized way to be able to identify a given object or property within a data contract so it can be referenced. 

- Identification of the data contract itself (use pattern matching for versioning?)
  - by file (major version is part of the filename)
  - by URI (includes ID and (major) version)
- Identification of elements in a data contract (e.g., a property in a schema of a data contract)
  - by YAML path (JSON Path: /schema[0]/properties[0] )
  - by implicit URI (/schema/$objectName/properties/$propertyName)
    - we need to define the URI scheme for all elements (a lot of work)
    - to reduce amount of work, we can provide a mapping from JSON Path to URI, with each element in a list required to have an local name, or a combination of properties
  - by explicit IDs (unique within the contract), URIs (enterprise-wide unique) or even UUIDs (universally unique)

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


**Assumptions**
* All objects that need to be referenced have unique paths within their contract
* External contracts are accessible and compatible with the reference syntax
* References are validated at contract validation time
* Circular references are detected and prevented

### MAIN PROPOSED SOLUTION ###

This proposed solution takes inspiration from the [rejected solutions](#alternatives) below and addresses two key concerns around the solution.

- Defining the references
  - What is the structure for a reference?
  - What is the unique identifier for a reference?
  - What is the reference fallback?
- Usage of a reference
  - How is a reference used for inclusion?
  - How is a reference used for 'linking'?


#### Key Concepts ####

A reference allows one element in a data contract to connect/link to or include another element or data contract object. References can point to:

- Objects within the same contract
- Objects in external contracts
- Different types of elements (schemas, properties, quality definitions, etc.)

#### Basic Format ####

A reference follows the following structure:
```yaml
<file><anchor><item><separator><item><separator>...
```

Where:

- < file > : Path to the contract file (optional for same-contract references)
- < anchor >: '#' symbol to mark entry into a contract (optional for same-contract)
- < item >: Contract element identifier
- < separator >: Dot '.' for navigating the hierarchy (following dot notation)


#### Path Components ####

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

*Anchor Component*

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

*Item Components*

Items use json-pointer notation to navigate the contract hierarchy and requires an anchor to denote entry to a local contract.

[JSON PATHS](https://www.ietf.org/archive/id/draft-goessner-dispatch-jsonpath-00.html)
[JSON PATH WIKI](https://en.wikipedia.org/wiki/JSONPath)


```yaml
# Reference a table
'#/schema/my-table'

# Reference a column in a table
'#/schema/my-table/properties/my-column'

# Reference a nested property
'#/schema/my-table/properties/my-array-column/items/properties/id'

# OP - json path

../../localfolder/localfile/myfile.yaml#schema.my-table.properties.my-array-column.items.id

# OP - json pointer
../../localfolder/localfile/myfile.yaml#schema/my-table/properties/my-array-column/items/id
```

==> bringing to TSC on json Pointer syntax for internal items

Arrays can also be accessed with an index. A sample contract with locations is below:

``` yaml
# Full example below with dot notation
id: my-id
version: 1.2.3
description: # description
  limitations: cannot be used alone #  .description.limitations
schema: # schema
- name: my-table # schema/my-table or schema/my-table/name
  logicalType: object # schema/my-table/logicalType
  properties: #  schema/my-table/properties
  - name: my-column #  schema.my-table.properties.my-column
    logicalType: string # schema.my-table.properties.my-column.logicalType
    quality: # .schema.my-table.properties.my-column.quality
    - type: sql #  .schema.my-table.properties.my-column.quality[0]
      query: |
        SELECT COUNT(*) FROM ${object} WHERE ${property} IS NOT NULL
      mustBeLessThan: 3600
  - name: my-array-column #  .schema.my-table.properties.my-array-column
    logicalType: array # .schema.my-table.properties.my-array-column.logicalType
    items:  # .schema.my-table.properties.my-array-column.items
      logicalType: object 
      properties: #  schema.my-table.properties.my-array-column.items.properties
        - name: id  # schema.my-table.properties.my-array-column.items.id
          logicalType: string  # .schema.my-table.properties.my-array-column.items.id.logicalType
          physicalType: VARCHAR(40) # .schema.my-table.properties.my-array-column.items.id.physicalType
        - name: zip # .schema.my-table.properties.my-array-column.items.zip
          logicalType: string
          physicalType: VARCHAR(15)
```

#### Resolving A Reference ####

When resolving references, the system follows this hierarchy:

1) Use the name field to determine element identity
2) Fall back to the mapping table if name is not present
3) If not found in the mapping table, use index-based reference

The default mapping rules are:

|Collection|Identifier Field|
|:-----|:-----|
schema | name|
properties | name |
quality | id (RFC0012)
roles | role|
channels | channel |
servers | server|

*NOTE* There is the option to introduce breaking changes by requiring a `name` field on all elements. This would remove the mapping table. A user can always use the index fallback. 

``` examples

# my_table and my_column would be 0, 0 - this is not a valid json pointer
ref: #/schema/0/properties/0 - valid json pointer
ref: #/schema/my_table/properties/my_column  - invalid json pointer

schema:
  - name: job_post
    physicalName: job_post_1_0_0
    description: Contains the information related to a single job posting by a single organisation.
    properties:
      - name: id
        required: true
        logicalType: string

#/schema/name="job_post"/properties/id
#/schema/job_post/properties/id
#/schema/0/properties/0
#schema.[?(@.name="job_post")].properties.[?(@.name="id")]
=> 

=> If we know the element in the array - then we can use the `name` to identify to make it independent on re-ordering
```

#### Reference Implementation ####

*Standard Format*

|Key | UX Label | Required | Description|
|:----|:----|:----|:----|
|relationships| relationships | optional| Array. A list of elements which to reference|
|relationships.type | type | optional | One of `foreignKey`, `include`. Default is `include`|
|relationships.to | to | optional | A reference to a contract object. Cannot be used in conjunction with ref. Must follow correct [reference structure](#basic-format) |
| relationships.from| from | optional |A reference to a contract object.  Cannot be used in conjunction with ref. Must follow correct [reference structure](#basic-format)|
|relationships.ref| ref | optional |A reference to a contract object. Cannot be used in conjunction with to & from. Must follow correct [reference structure](#basic-format)|

*NOTE* Given the shorthand - we could remove the `ref` and require either a to/from for an include reference? 

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

##### Reference Types #####

*Foreign Key* 

Foreign key references establish relationships between data elements, indicating that values in one element relate to values in another:

``` yaml

# using to/from 
relationships:
  - type: foreignKey
    to: 'my-table.properties.my-column'

# using standard ref object 
relationships:
  - type: foreignKey
    ref: 'my-table.properties.my-column'
```

*Inclusions* 

Inclusion references incorporate the referenced element's definition into the current context:

```yaml
relationships:
  - type: include
    ref: 'my-table.properties.my-other-column'
```


#### Implementation Considerations ####

*Validation Rules*
Reference implementations must validate:

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

##### Examples #####


###### Simple Same Contract Foreign Key ######
```yaml

# Table definition
schema:
  - name: customers
    logicalType: object
    properties:
      - name: customer_id
        logicalType: string
      - name: customer_name
        logicalType: string

  - name: orders
    logicalType: object
    properties:
      - name: order_id
        logicalType: string
      - name: customer_id
        logicalType: string
        # Reference to customers table as foreign key
        relationships:
          - type: foreignKey
            to: 'schema.customers.properties.customer_id'
```


###### Simple Same Contract Inclusion######
```yaml

# Table definition
schema:
  - name: customers
    logicalType: object
    properties:
      - name: customer_id
        logicalType: string
        quality:
        - type: library
          rule: mustBeUnique
      - name: customer_name
        logicalType: string


  - name: orders
    logicalType: object
    properties:
      - name: order_id
        logicalType: string
      - name: customer_id
        logicalType: string
        # Reference to include quality field
        quality:
        - relationships:
          - type: include
            to: 'schema.customers.properties.customer_id.quality[0]'

# equivalent reference:
  - name: orders
    logicalType: object
    properties:
      - name: order_id
        logicalType: string
      - name: customer_id
        logicalType: string
        # Reference to include quality field
        quality:
        - $ref: 'schema.customers.properties.customer_id.quality[0]'

```
@SimonHarrer - thoughts on the above for including a quality definition?


###### External Contract ######
``` yaml
# In contract: product-catalog.yaml
schema:
  - name: products
    logicalType: object
    properties:
      - name: product_id
        logicalType: string
      - name: product_name
        logicalType: string
      - name: created_at
        logicalType: date
        physicalType: DATETIME
      - name: updated_at
        logicalType: date
        physicalType: DATETIME

# In contract: orders.yaml
schema:
  - name: order_items
    logicalType: object
    properties:
      - name: order_id
        logicalType: string
      - name: product_id
        logicalType: string
        # Reference to external contract
        relationships:
          - type: foreignKey
            to: 'product-catalog.yaml#schema.products.properties.product_id'
          - type: include
            ref: 'product-catalog.yaml#schema.products.properties.created_at'
          - type: include
            ref: 'product-catalog.yaml#schema.products.properties.updated_at'
```

###### External Contract - non schema fields  ######

```yaml 
# In contract: product-catalog.yaml

servers:
  - server: my-server-name
    type: my-server
    description: server-description
    environment: production

# In current contract: orders.yaml

servers:
  # reference to servers block including entire servers object
  - relationships:
    - type: include
      ref: 'product-catalog.yaml#servers' 

# equivalent to 
servers:
  $ref: 'product-catalog.yaml#servers' 

schema:
  - name: order_items
    logicalType: object
    properties:
      - name: order_id
        logicalType: string
      - name: product_id
        logicalType: string

```
@simonHarrer - how do we want to reference non-schema objects. Ie. should the reference live under the object? Overwrite the object?


## Alternatives

> Rejected alternative solutions and the reasons why.

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

OPTION 3

Context - This solution is below. However, the naming has confused users.

#### Examples ####

``` yaml 

# internal reference within contract (using dot notation and $ as anchor)
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
        relationships:
        - ref: $users.first_name.quality.q160512f7
          type: embed

# Usage of foreign keys 
schema:
- name: users
  properties:
  - name: id
    type: integer
    primaryKey: true
  - name: supervisor_id
    type: integer
    relationships:
    - ref: $users.id
      type: foreignKey
      
- name: posts
  properties:
  - name: id
    type: integer
    primaryKey: true
  - name: user_id
    type: integer
    relationships:
    - ref: '#users.id' # other schema
      type: foreignKey
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

### Solution 3: Somewhere in the middle

This proposed solution is somewhere in the middle between a classic semantic (context) approach and a openAPI based approach. 
We leverage *implicit* uris when identifying elements in a data contract and limit external references to  include a specific filename. 
By using implicit uris we can use linting rules to ensure that uris are valid and exist and unique within the specific scope. 

This solution would require use to:
- Define a clear method for creating uris 
- Define a hierarchy as part of the above (e.g. name is a property we can use to identify an object etc. )
- Arrays would require a `key` field in order to be correctly identified

We are not tied to the format (e.g we could use '/', '::', '.' etc. ) However we need:
- an anchor to denote internal references (# Proposed)
- a path separator to demote different items within the scope (e.g. json-path or dot notation)

```yaml
id: my-id
version: 1.2.3
description: # /description
  limitations: bla bla # /description/limitations
schema: # schema
- name: my-table # /schema/my-table  or  /schema/my-table/name
  logicalType: object # /schema/my-table/logicalType
  properties: # /schema/my-table/properties
  - name: my-column # /schema/my-table/properties/my-column
    logicalType: string # /schema/my-table/properties/my-column/logicalType
    quality: # /schema/my-table/properties/my-column/quality
    - type: sql # /schema/my-table/properties/my-column/quality/???
      query: |
        SELECT COUNT(*) FROM ${object} WHERE ${property} IS NOT NULL
      mustBeLessThan: 3600
  - name: my-array-column # /schema/my-table/properties/my-array-column
    logicalType: array # /schema/my-table/properties/my-array-column/logicalType
    items:  # /schema/my-table/properties/my-array-column/items
      logicalType: object 
      properties: #  /schema/my-table/properties/my-array-column/items/properties
        - name: id  #  /schema/my-table/properties/my-array-column/items/id
          logicalType: string  # /schema/my-table/properties/my-array-column/items/id/logicalType
          physicalType: VARCHAR(40) # /schema/my-table/properties/my-array-column/items/id/physicalType
        - name: zip #  /schema/my-table/properties/my-array-column/items/zip
          logicalType: string
          physicalType: VARCHAR(15)
```

For identifying the _data contract_ itself, we propose the use of a single file. This means that we are inherently referencing a single contract with a given version. 
To reference a property we can combine the file path with a local uri: "/path/to/data-contract-v1.yaml#/schema/my-table/properties/my-column" using the proposed # as a separator.

The actual format of how a reference is defined is still in process, but could be one of the following:

- define a context (as proposed above)
  - context may be too broad and is better served by more specific items e.g. `authoritativeDefinitions`
- use OpenAPI `$ref` specification
- use a `relationships` field, which contains a `type` and a `ref`
- use a `businessDefinition` field 


```yaml
context: # relationships, authoritativeDefinitions, ...
- type: foreignKey
  ref: "pass in a URI"
- type: businessDefinition
  ref: "pass in a URI"

schema:
  - name: my-table
    properties:
      - context:
        - type: embed
          ref: "#/schema/my-table/properties/my-column"
      - $ref: "#/schema/my-table/properties/my-column"
      - name: my-column2
        logicalType: string
        businessDefinition: "#/schema/my-table/properties/my-column/businessDefinition" # using business definition
        relationships: # using a 'relationships' block
        - type: foreignKey
          ref: "#my-table.my-column"
```

## Decision

> The decision made by the TSC.

## Consequences

> The consequences of this decision.

## References

> Prior art, inspiration, and other references you used to create this based on what's worked well before.


## Outstanding Questions
QQ > How do we deal with external access/auth considerations references? 
  - this is an implementation issue, and is noted in implementation considerations
QQ > How do we deal across different data contract versions?
  -> by linking to a specific contract, the version is inhearent. To update contract versions, the references should be changed
