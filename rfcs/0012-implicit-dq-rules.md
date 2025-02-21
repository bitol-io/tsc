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

### What set of rules will be included

- Null
- Unique
- Equal
- Min
- Max
- Less than
- Greater than
- Between
- Pattern/regex
- Min length
- Max length
- Field exists
- Field data type
- Referential integrity
- Row count
(16 rules)

### Format of the rules

Below are some ideas on the format of the rules. It could be a combination of the following:

#### 1. Define rule along with the condition it should adhere to

```yaml
quality:
  - rule: null
    mustBeLessThan: 10
    description: "There must be less than 10 null values in the column."
  - rule: notNull
    mustBe: 0
    description: "There must be no null values in the column."
  - rule: duplicateCount
    mustBeGreaterThan: 0
  - rule: validValues
    validValues: ['pounds']
```

#### 2. Separate rules for each negation/alternative

```yaml
quality:
  - rule: null
  - rule: notNull
  - rule: lessThan
    value: 100
  - rule: lessThanOrEqualTo
    value: 100
```

#### 3. Rules with additional parameters for negation/alternatives

```yaml
quality:
  - rule: null
  - rule: null
    description: "The value must be not null."
    negate: true
  - rule: lessThan
    description: "The value must be less than 100."
    value: 100
  - rule: lessThan
    description: "The value must be less than or equal to 100."
    strictly: false
    value: 100
  - rule: between
    description: "The value must be between 0 and 100."
    min: 0
    max: 100
```

#### 4. Include a threshold for the rule

Threshold is similar to [format 1](#1-define-rule-along-with-the-condition-it-should-adhere-to) but always represents `mustBeLessThan`.
If not specified, it is assumed to be 0.

```yaml
quality:
  - rule: null
    threshold: 10
    description: "There must be less than 10 null values in the column."
  - rule: notNull
    threshold: 0
    description: "There must be no null values in the column."
  - rule: lessThan
    value: 100
    threshold: 20
    description: "There must be less than 20 values less than 100."
```

### Comparison table

| Rule         | Description                               | Format 1                                               | Format 2                                           | Format 3                                           | Format 4                                                   |
|--------------|-------------------------------------------|--------------------------------------------------------|----------------------------------------------------|----------------------------------------------------|------------------------------------------------------------|
| Null         | Check if the value is null                | <pre>- rule: null<br>  mustBe: 0</pre>                 | <pre>- rule: null</pre>                            | <pre>- rule: null</pre>                            | <pre>- rule: null</pre>                                    |
| Not Null     | Check if the value is not null            | <pre>- rule: notNull<br>  mustBe: 0</pre>              | <pre>- rule: notNull</pre>                         | <pre>- rule: null<br>  negate: true</pre>          | <pre>- rule: notNull</pre>                                 |
| < 10 null    | Less than 10 values can be null           | <pre>- rule: null<br>  mustBeLessThan: 10</pre>        | <pre>- rule: null<br>  mustBeLessThan: 10</pre>    | <pre>- rule: null<br>  mustBeLessThan: 10</pre>    | <pre>- rule: null<br>  threshold: 10</pre>                 |
| > 10 null    | More than 10 values can be null           | <pre>- rule: null<br>  mustBeGreaterThan: 10</pre>     | <pre>- rule: null<br>  mustBeGreaterThan: 10</pre> | <pre>- rule: null<br>  mustBeGreaterThan: 10</pre> | <pre>- rule: notNull<br>  threshold: total_rows - 10</pre> |

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
- [Data Contract CLI](https://github.com/datacontract/datacontract-specification/blob/main/datacontract.schema.json#L1797)
