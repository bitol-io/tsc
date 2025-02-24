# External & Internal References 

Champion: Todd Nemanich.

## Summary

> One paragraph explanation of the RFC.

This RFC proposes to introduce internal and external relationships within the ODCS standard. 

## Motivation

> Why are we doing this? What use cases does it support? What is the expected outcome?
> How does it align with our guiding values?

Our goal is to allow for references between and within data contracts. We believe that this will allow for improved reusability within the data contract.

If we are successful, users should be able to:

- reference a dataset element in another element within a data contract 
- reference a dataset element outside a data contract

## Design and examples

> This is the bulk of the RFC.
> Explain the design in enough detail for somebody familiar with data contracts and the standard to understand. This should get into specifics and corner-cases, and include examples of how this is to be used.
> Offer at least two examples, one is minimalist, one is more structured.

We have identified four main use-cases which need to be demonstrated for each of the proposed solutions, namely:

* Internal reference from dataset element to another contract element (i.e. to DQ rule)
* Internal reference from a non-dataset element (i.e. servers->stakeholders for support)
* External reference from a dataset element (i.e. data subset)
* External reference from non-dataset element (i.e. SLA -> observability store)

### SOLUTION 1: Open API Standard 

A proposed solution mimics the existing [OpenAPI standard](https://learn.openapis.org/referencing/overview.html), with the main difference being the addition of a `ref` field to the `schema` object.

**Assumptions**
> TODO: Add any known assumptions 

**Overview**
> TODO: Add more detailed overview and usage

**PRO**
- being an open standard users are comfortable with it 
- being an open standard it is easy to implement and maintain

**CON**
- duplication of references, limited reusability 

#### Examples 

> TODO: Add OpenAPI Examples for use-cases above here. 


### SOLUTION 2: JSON-LD 'CONTEXT' inspired

A proposed solution which takes inspiration from [JSON-LD / Linked Data](https://w3c.github.io/json-ld-syntax) concepts - specifically the use of [`context`](https://w3c.github.io/json-ld-syntax/#the-context) and [`IRI`](https://w3c.github.io/json-ld-syntax/#iris)


**Assumptions**
This solution assumes the following:
- Each element within a data contract has a globally unique identifier which can be used for reference
- each element has a field which can be used as the specific identified (e.g. `name` or has a given id e.g. `quality.id` (RFC-0014))


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
context | object | No | Object. Reference related context information.|
context.name | string | No | The `term` used to reference this specific context |
context.uri | string | No | the external path or location of the given reference. If None, the current contract is assumed as the location. |
context.description| string | No | Description of the given context. |
context.ref | string | No | The object or entity wishing to reference. |

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

* Internal reference from a non-dataset element (i.e. servers->stakeholders for support)

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

* External reference from a dataset element (i.e. data subset)
```YAML
# TODO: add example
```
* External reference from non-dataset element (i.e. SLA -> observability store)
```YAML
# TODO: add example
```

## Alternatives

> Rejected alternative solutions and the reasons why.

## Decision

> The decision made by the TSC.

## Consequences

> The consequences of this decision.

## References

> Prior art, inspiration, and other references you used to create this based on what's worked well before.
