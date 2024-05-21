# Support for versioning at the attribute level

> The champion is the person who is primarily supporting the RFC.

Champion: Martin Meermeyer & Jean-Georges Perrin.

## Summary

> There are two use cases.
> 1) Due to changes in business structures and processes (e. g. new counties, new warehouse, etc.) data payloads change as well which could heavily affect subsequent business processes. This holds even in the case that the technical definition of a field carrying such information remains the same. A versioning allows to keep track of such changes. This being said, this usecase could also be covered by versioning of corresponding data quality rules, which are assosiated with the attributes.
> 2) If more than one version of a business payload is active at the same time a versioning allows to keep different versions of the same payload in one data contract. 

## Motivation

> Communication: Substantial changes in business data payloads should be announced within an organisation toward each and every consumer of a particular data source. Refering to an explicit new version which is already "published" in a data contract helps to foster communication.
> 
> Documentation: Especially for retrospective data usage (reporting/controlling, balancing/auditing, business analysis, data science applications) it is important to have a proper and for non-technical people easy-to-understand documentation of actual and past business payloads.
>
> Readability: Hiding versions of data descriptions in versioning tools like GIT or SVN is often not feasible for a non-technical audience. The standard should be inclusive for business people to the greatest extent to achieve widespread acceptance of the data contract concept as a whole.
>
> Principal of Parsimony: Versioning on attribute level is more parsimonious than versioning on contract level. This being said, versioning on data quality rule level is even more parsimonious.  


## Design and examples

> This is the bulk of the RFC.
> Explain the design in enough detail for somebody familiar with data contracts and the standard to understand. This should get into specifics and corner-cases, and include examples of how this is to be used.
> Offer at least two examples, one is minimalist, one is more structured.

# Option A

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

In the first half of 2023 the business decision was made to expand to Canada on 2023-06-01
The data producers of the transactional systems learn about this in the first quarter of 2023. The projected changes as incorporated into the DC as soon as possible:

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
      description: Receiver country. Could be 'USA' or 'CAN'. This is very important for customs, VAT and shipping fee regulations.
      version_number: 2
      version_validFrom: 2023-06-01
      version_flagActive: false
...
 ```
Announcement to every consumer in an appropriate channel (also as soon as possible) could look like this: "Dear transaction data consumer, please note the upcoming change in our payload (attribute 'Receiver country code') due to the scheduled expansion to Canada starting from 2023-06-01. For details please refer to the description in our data contract."

Finally, on the 2023-06-01 the field 'version_flagActive' must be toggled:
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
      version_flagActive: false
    - attribute: Receiver country code
      kind: field
      description: Receiver country. Could be 'USA' or 'CAN'. This is very important for customs, VAT and shipping fee regulations.
      version_number: 2
      version_validFrom: 2023-06-01
      version_flagActive: true
...
 ```

# Option B

Instead of 'version_flagActive' a field 'version_validTo' would also be possible. This would require less changes in the contract itself when the new version becomes active. Communication scheme is identical to Option A
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
      version_validTo: 2023-05-31
    - attribute: Receiver country code
      kind: field
      description: Receiver country. Could be 'USA' or 'CAN'. This is very important for customs, VAT and shipping fee regulations.
      version_number: 2
      version_validFrom: 2023-06-01
      version_validTo: 2099-12-31
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
