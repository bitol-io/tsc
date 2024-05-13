# Support for versioning at the attribute level

> The champion is the person who is primarily supporting the RFC.

Champion: Martin Meermeyer & Jean-Georges Perrin.

## Summary

> There are two use cases.
> 1) Due to changes in business structures and processes (e. g. new counties, new warehouse, etc.) data payloads change as well which could heavily affect subsequent business processes. This holds even in the case that the technical definition of a field carrying such information remains the same. A versioning allows to keep track of such changes. This being said, this usecase could be covered by versioning of the corresponding data quality rules, which are assosiated with the attributes.
> 2) If more than one version of a business payload is active at the same time a versioning allows to keep things in one data contract. 

## Motivation

> Communication: Substantial changes in business data payloads should be announced within an organisation. Refering to an explicit new version which is already "published" in a data contracts helps to foster communication.
> 
> Documentation: Especially for retrospective data usage (reporting, business analysis, data science applications, balancing/auditing) it is important to have a proper and easily accessible documentation of business payload in the past.
>
> Readability: Hiding versions of data descriptions in versioning tools like GIT or SVN is often not feasible for a non-technical audience. The standard should be inclusive for business people to the greatest extent to achieve widespread acceptance of the data contract concept as a whole.
>
> Principal of Parsimony: Versioning on attribute level is more parsimonious than versioning on contract level. This being said, versioning on data quality rule level is even more parsimonious.  
>
> How does it align with our guiding values? 

## Design and examples

> This is the bulk of the RFC.
> Explain the design in enough detail for somebody familiar with data contracts and the standard to understand. This should get into specifics and corner-cases, and include examples of how this is to be used.
> Offer at least two examples, one is minimalist, one is more structured.

YAML example taken from rfc-0004, option F

The usage over time is important, consider the following scenario.

A business starts in January 2020 and  delivers it's products in the US only. For a transactinal system the dc from the very beginning looks like this:

```YAML
schema:
 - object: TransactionDetail
    kind: table
    physicalName: trx_details
    description: Contains transactions.
    attributes:
    - attribute: Receiver country code
      kind: field
      description: Receiver country. Currently this is always 'USA'.
      version_number: 1
      version_validFrom: 2020-01-01
      version_flagActive: true
...
 ```

In the first half of 2023 the business decision was made to expand to Mexico on 2023-06-01
The data producers of the transactional systems learn about this on in the first quater of 2023 and documents the projected changes as soon as possible:

```YAML
schema:
 - object: TransactionDetail
    kind: table
    physicalName: trx_details
    description: Contains transactions.
    attributes:
    - attribute: Receiver country code
      kind: field
      description: Receiver country. Currently this is always 'USA'.
      version_number: 1
      version_validFrom: 2020-01-01
      version_flagActive: true
    - attribute: Receiver country code
      kind: field
      description: Receiver country. Could be 'USA' or 'MEX'. This is very important for customs, VAT and shipping fee regulations.
      version_number: 2
      version_validFrom: 2023-01-01
      version_flagActive: false
...
 ```






## Alternatives

> Rejected alternative solutions and the reasons why.

## Decision

> The decision made by the TSC.

## Consequences

> The consequences of this decision.

## References

> Prior art, inspiration, and other references you used to create this based on what's worked well before.
