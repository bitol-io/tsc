# Implicit Data Quality Rules

> The champion is the person who is primarily supporting the RFC.

Champion: Jochen Christ.

[Slack channel](https://data-mesh-learning.slack.com/archives/C08C9G9PDA7).

## Summary

> One paragraph explanation of the RFC.

This RFC proposes to add predefined data quality rules to the standard. These rules are commonly used in data quality 
management. It was first proposed as part of [RFC-0007](0007-data-quality.md#implicit-or-predefined-data-quality-rules).

## Motivation

> Why are we doing this? What use cases does it support? What is the expected outcome?
> How does it align with our guiding values?

- **We favor a small standard over a large one**
  - Define a small set of rules that are commonly used in data quality management.
  - Look to the community to gain inspiration for ways rules can be extended.
- **We favor interoperability over readability**
  - Common set of rules across all data contracts.
  - Agnostic to the underlying data storage and data quality engine.

## Design and examples

> This is the bulk of the RFC.
> Explain the design in enough detail for somebody familiar with data contracts and the standard to understand. This should get into specifics and corner-cases, and include examples of how this is to be used.
> Offer at least two examples, one is minimalist, one is more structured.

## Alternatives

> Rejected alternative solutions and the reasons why.

## Decision

> The decision made by the TSC.

## Consequences

> The consequences of this decision.

## References

> Prior art, inspiration, and other references you used to create this based on what's worked well before.

- [Great Expectations](https://greatexpectations.io/expectations/)
- [Soda Metrics and Checks](https://docs.soda.io/soda-cl/metrics-and-checks.html#list-of-sodacl-metrics-and-checks)
- [OpenMetadata Data Quality](https://docs.open-metadata.org/latest/how-to-guides/data-quality-observability/quality/tests-yaml)
- [Deequ](https://github.com/awslabs/deequ)
