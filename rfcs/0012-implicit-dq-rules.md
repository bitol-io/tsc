# Implicit Data Quality Rules

> The champion is the person who is primarily supporting the RFC.

Champion: Jochen Christ.

[Slack channel](https://data-mesh-learning.slack.com/archives/C08C9G9PDA7).

## Summary

> One paragraph explanation of the RFC.

This RFC proposes to add a list of predefined data quality rules to the standard.

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


## Alternative 1: Metrics-Style


### Design Principles

- Simple: The rule/metric should be simple and easy to understand.
- Executable: The rules should be executable by data quality tools
- Metrics-based: The rules should be defined in a way to result in a numeric value that can be compared to a threshold
- Threshold-based: The rule should have a threshold that defines the acceptable value. Tools can report how many rows/objects are violating the rule. Thresholds can be skipped if it is implicit from the rule.
- Percentage and absolute values: The rule should be able to express both percentage and absolute values.

### Quality Type

The quality type is `library`, which is also the default, if a `rule` (TBD: or `metric`) is defined.

```yaml
quality:
  - type: library 
    rule: nullValues # or metric: nullValues?
```

is the same as:

```yaml
quality:
  - type: library 
    rule: nullValues # or metric: nullValues?
```

### Predefined Rules/Metrics

- Schema
  - Row Count (`rowCount`)
  - Unique/duplicates (`duplicateValues`)
- Property
  - Null values (`nullValues`)
  - Missing values (empty strings, etc.) (`missingValues`)
  - Unique/duplicates (`duplicateValues`)
  - Valid values (enum, regex, etc.) (`invalidValues`)
  - Statistical
    - count (`count`)  
    - avg (`avg`)
    - sum (`sum`)
    - stddev (`stddev`)
    - median (`median`)


### Examples

#### Null values

```yaml
properties:
  - name: order_id
    quality:
      - rule: nullValues 
        mustBe: 0
        unit: rows
        description: "There must be no null values in the column."
      - rule: nullValues
        mustBeLessThan: 10
        unit: rows
        description: "There must be less than 10 null values in the column."
      - rule: nullValues
        mustBeLessThan: 1
        unit: percent
        description: "There must be less than 1% null values in the column."
```

#### Missing values

```yaml
properties:
  - name: email_address
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
     - rule: invalidValues
       validValues: ['pounds', 'kg']
       mustBeLessThan: 5
       unit: rows
```

Using a pattern:

```yaml
properties:
  - name: iban
    quality:
     - rule: noInvalidValues
       mustBe: 0
       description: "The value must be an IBAN."
       pattern: '^[A-Z]{2}[0-9]{2}[A-Z0-9]{4}[0-9]{7}([A-Z0-9]?){0,16}$'
```



#### Duplicate values

Level: Property

```yaml
quality:
  - rule: duplicateValues 
    mustBe: 0 
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
  - rule: duplicateValues
    mustBe: 0 
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
    description: "There must be more than 1 million rows in the table."
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

same for median, max, min, sum?


## Alternative 2: Verb-Style

### Design Principles

- Simple: The rule should be simple and easy to understand.
- Executable: The rules should be executable by data quality tools
- Verbs: Rules shoud start with a verb, such as has..., is...
- Rules have a boolean result: true or false
- Rules can have `arguments` (an object of key value entries)


### Quality Type

The quality type is `library`, which is also the default, if a `rule` is defined.

```yaml
quality:
  - type: library
    rule: hasNoNullValues
```

is the same as:

```yaml
quality:
  - rule: hasNoNullValues
```

### Predefined Rules

| Level    | Rule                  | Arguments                        |
|----------|------------------------|-----------------------------------|
| Schema   | `hasRowCountGreaterThan` | `value` (numeric)                 |
| Schema   | `hasRowCountLessThan`    | `value` (numeric)                 |
| Schema   | `isCompoundUnique`       | `properties` (list of columns)    |
| Property | `isUnique`               | –                                 |
| Property | `hasNoNullValues`        | –                                 |
| Property | `hasNoMissingValues`     | `missingValues`                   |
| Property | `hasValidValue`               | `validValues`                     |
| Property | `isOlderThan`            | `days`, `seconds`                 |
| Property | `isNotOlderThan`         | `days`, `seconds`                 |



### Examples

#### Row count

```yaml
schema:
- name: employees
  quality:
  - rule: hasRowCountGreaterThan
    arguments:
      value: 1000
    description: "The dataset must contain more than 1000 rows."
```

#### Compound Uniqueness

```
schema:
- name: orders
  quality:
  - rule: isCompoundUnique
    arguments:
      properties: ["tenant_id", "order_id"]
    description: "The combination of tenant_id and order_id must be unique across the dataset."
```

#### Null values

```yaml
properties:
  - name: order_id
    quality:
      - rule: hasNoNullValues
        description: "There must be no null values in the column."
```

#### Allowed values

```yaml
properties:
  - name: status
    quality:
      - rule: hasValidValue
        arguments:
          validValues: ["OPEN", "SUCCESS", "CANCELED"]
        description: "Status must be one of the predefined values."
```

#### Uniqueness

```yaml
properties:
  - name: customer_id
    quality:
      - rule: isUnique
        description: "Customer IDs must be unique."
```

#### Age checks

```yaml
properties:
  - name: created_at
    quality:
      - rule: isOlderThan
        arguments:
          days: 30
        description: "Records must be older than 30 days."
      - rule: isNotOlderThan
        arguments:
          days: 365
        description: "Records must not be older than 1 year."
```

#### Aggregations

```yaml
properties:
  - name: price
    quality:
      - rule: hasAvgGreaterThan
        arguments:
          value: 50.0
        description: "The average price must be greater than 50."

  - name: quantity
    quality:
      - rule: hasSumEqualTo
        arguments:
          value: 10000
        description: "The total quantity must equal 10,000."

  - name: salary
    quality:
      - rule: hasMedianNotEqualTo
        arguments:
          value: 0
        description: "The median salary must not be 0."
```

## Optional A: Value Checks

### Values checks

We could allow simple value checks without an explicit rule. These operators already exists, but can now be used to compare the numeric values in a numeric property.

  - `mustBe`
  - `mustNotBe`
  - `mustBeGreaterThan`
  - `mustBeGreaterOrEqualTo`
  - `mustBeLessThan`
  - `mustBeLessOrEqualTo`
  - `mustBeBetween`
  - `mustNotBeBetween`

#### Example

```yaml
properties:
  - name: price
    quality:
      - mustBeGreaterThan: 1.0
        description: "The minimum price is $1"
```

## Consequences

> The consequences of this decision.

## References

> Prior art, inspiration, and other references you used to create this based on what's worked well before.

- [Great Expectations](https://greatexpectations.io/expectations/)
- [Soda Metrics and Checks](https://docs.soda.io/soda-cl/metrics-and-checks.html#list-of-sodacl-metrics-and-checks)
- [OpenMetadata Data Quality](https://docs.open-metadata.org/latest/how-to-guides/data-quality-observability/quality/tests-yaml)
- [Deequ](https://github.com/awslabs/deequ)
- [dqx](https://databrickslabs.github.io/dqx/docs/reference/quality_rules/)
- [Data Contract CLI](https://github.com/datacontract/datacontract-specification/blob/main/datacontract.schema.json#L1797)

