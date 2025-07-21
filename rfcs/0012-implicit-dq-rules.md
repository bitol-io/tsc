# Implicit Data Quality Rules

> The champion is the person who is primarily supporting the RFC.

Champion: Jochen Christ.

[Slack channel](https://data-mesh-learning.slack.com/archives/C08C9G9PDA7).

## Summary

> One paragraph explanation of the RFC.

This RFC proposes to add a list of predefined data quality rules to the standard. These rules are commonly used in data quality 
management.

## Motivation

> Why are we doing this? What use cases does it support? What is the expected outcome?
> How does it align with our guiding values?

- **We want it to make it simple for users to define commonly used data quality rules**
  - No need to define the rule from scratch every time.
  - No need to write SQL queries or custom code manually.
- **We want to make the rules simple to implement by tools**
  - Every tool that wants to support ODCS should be able to implement the rules.
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

### Design Principles

- Simple: The rule should be simple and easy to understand.
- Executable: The rules should be executable by data quality tools
- Metrics-based: The rules should be defined in a way to result in a numeric value that can be compared to a threshold
- Threshold-based: The rule should have a threshold that defines the acceptable value. Tools can report how many rows/objects are violating the rule. Thresholds can be skipped if it is implicit from the rule.
- Percentage and absolute values: The rule should be able to express both percentage and absolute values.


### Quality Type

The quality type is `library`, which is also the default.

```yaml
quality:
  - type: library
    rule: noNullValues
```

is the same as:

```yaml
quality:
  - rule: noNullValues
```

### Predefined Rules

- Property
  - Null (`noNullValues`, `nullValues`)
  - Missing values (empty strings, etc.) (`noMissingValues`, `missingValues`)
  - Unique/duplicates (`noDuplicates`, `duplicateValues`)
  - Valid values (enum, regex, etc.) (`validValues`, `invalidValues`)
- Row Count
- Statistical
  - avg (`avg`)
  - sum (`sum`)
  - stddev (`stddev`)
  - median (`median`)
- Freshness


### Examples

#### Null values

```yaml
quality:
  - rule: noNullValues
    description: "There must be no null values in the column."
  - rule: nullValues
    mustBeLessThan: 10
    description: "There must be less than 10 null values in the column."
  - rule: nullValues
    mustBeLessThan: 1
    unit: percent
    description: "There must be less than 1% null values in the column."
```

#### Missing values

```yaml
quality:
  - rule: noMissingValues
    missingValues: [null, '', 'N/A', 'n/a']
    description: "There must be no missing values in the column."
  - rule: missingValues
    missingValues: [null, '', 'N/A', 'n/a']
    mustBeLessThan: 100
    unit: absolute # absolute (default) or percent
```


#### Valid values

Level: String Property

```yaml
properties:
  - name: line_item_unit
    quality:
     - rule: noInvalidValues
       description: "The value must be either 'pounds' or 'kg'."
       validValues: ['pounds', 'kg']
     - rule: invalidValues
       validValues: ['pounds', 'kg']
       mustBeLessThan: 5
       unit: absolute # absolute (default) or percent
```

Using a pattern:

```yaml
properties:
  - name: iban
    quality:
     - rule: noInvalidValues
       description: "The value must be an IBAN."
       pattern: '^[A-Z]{2}[0-9]{2}[A-Z0-9]{4}[0-9]{7}([A-Z0-9]?){0,16}$'
```



#### Duplicate values

Level: Property

```yaml
quality:
  - rule: noDuplicateValues
    description: "There must be no duplicate values in the column."
  - rule: duplicateValues 
    mustBe: 0 # same as noDuplicateValues
    description: "There must be no duplicate values in the column."
  - rule: duplicateValues
    mustBeLessThan: 10
    description: "There must be less than 10 duplicate values in the column."
  - rule: duplicateValues
    mustBeLessThan: 1
    unit: percent
    description: "There must be less than 1% duplicate values in the column."
```


Level: Schema

```yaml
quality:
  - rule: noDuplicateValues
    description: "There must be no duplicate values for a specific tenant."
    properties:
      - tenant_id
      - order_id
```


#### Row Count

Level: Schema

```yaml
quality:
  - rule: rowCount
    mustBeGreaterThan: 1000000
    description: "There must be more than 1 million values in the table."
```

#### Avg

Level: Numeric Property

```yaml
properties:
  - name: order_total
    logicalType: number
    quality:
      - rule: avg
        mustBeBetween: [100, 200]
        description: "The average value of this column must be between 100 and 200."
```

Todo: same for median, max, min, sum?

### Freshness

Level: Timestamp Property

```yaml
properties:
  - name: created_at
    logicalType: timestamp
    quality:
      - rule: freshness
        mustBeLessThan: 24
        unit: hours
        description: "At least one entry must be less than 24 hours old."
```




## Consequences

> The consequences of this decision.

## References

> Prior art, inspiration, and other references you used to create this based on what's worked well before.

- [Great Expectations](https://greatexpectations.io/expectations/)
- [Soda Metrics and Checks](https://docs.soda.io/soda-cl/metrics-and-checks.html#list-of-sodacl-metrics-and-checks)
- [OpenMetadata Data Quality](https://docs.open-metadata.org/latest/how-to-guides/data-quality-observability/quality/tests-yaml)
- [Deequ](https://github.com/awslabs/deequ)
- [Data Contract CLI](https://github.com/datacontract/datacontract-specification/blob/main/datacontract.schema.json#L1797)

