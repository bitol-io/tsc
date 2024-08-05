# Data Quality

* Champion: Jean-Georges Perrin
* Discusssion channel: [Slack](https://aidaug.slack.com/archives/C06TC4X4KD1).


## Summary

Build more genericity around the definition of the data quality rules.


## Motivation

* Support for multiple rules engine/data quality providers (support for some edge cases).
* The current (v2.x) implementation of data quality rules is too close to the PayPal tooling.


## Working group

(order is not significant)

* Chris Foyer
* Enrique Catala
* Eugene Stakhov
* Jean-Georges Perrin
* Jochen Christ
* Keith John Ellis
* Manuel Destouesse 
* Martin Meermeyer
* Peter Flook
* Simon Harrer
* Todd Nemanich 
* Tom Baeyens 
* Alpesh Pandya


## Design principles

* Assumes the new schema structure (RFC 0004, approved July 2024).
* Standard block that can be applied at the table (object) level or at the column (property) level.
* Support for SQL directly in the contract.
* Support for `customProperties`.
* Easier scheduling.
* Quality attributes can be defined on table or column level (replaced by object or property in ODCS v3).
* Support different levels/stages of data quality attributes
  - __Text__: A human-readable text that describes the quality of the data.
  - __Implicit__ rules: A maintained library of commonly-used predefined quality attributes such as `rowCount`, `unique`, `freshness`, and more.
  - __SQL__: An individual SQL query that returns a value that can be compared. Can be extended to `Python` or other.
  - __Custom__: Quality attributes that are vendor-specific, such as Great Expectations, dbt tests, or Montecarlo monitors.
* The predefined types should be based on Soda's proposal of [Data contract check reference](https://docs.soda.io/soda/data-contracts-checks.html)

### Text

Status: this is approved as part of ODCS v3.

A human-readable text that describes the quality of the data. Later in the development process, these might be translated into an executable check (such as `sql`), an implicit rule, or checked through an AI engine.

```yaml
quality:
  - type: text
    description: The email address was verified by the system.
```

### Implicit (or predefined) data quality rules

Status: this is approved as part of ODCS v3.

We should support a list of predefined rules that are commonly used in data quality checks. These should be executable in all common data quality engines.

This makes life simpler for data engineers, as they don't have to write the SQL query themselves.


#### Column-level

##### Duplicate count on rows

```yaml
quality:
- type: implicit # optional and default value
  rule: duplicateCount
  mustBeLessThan: 10
  name: Fewer than 10 duplicate names
  unit: rows
```

##### Duplicate count on %

```yaml
quality:
- rule: duplicateCount
  mustBeLessThan: 1
  unit: percent
```

##### Valid values

```yaml
quality:
- rule: validValues
  validValues: ['pounds']
```

##### Invalid values

```yaml
quality:
- rule: invalid
  mustBeLessThan: 3
  unit: percent
  object: country
  property: id
```

#### Table-level

##### Row count

```yaml
quality:
  - rule: rowCount
    mustBeBetween: [100, 120]
    name: Verify row count range
```

#### Soda influence

The experimental [Data contract check reference](https://docs.soda.io/soda/data-contracts-checks.html) is a good starting point.

However, some minor changes to the Soda reference:
- Use lowerCamelCase instead of snake_case
- Instead `checks:` use `quality:` as the key for the list of quality attributes.
- Drop `metric_expression`
- In references, such as `valid_values_reference_data` adopt `dataset` and `column` to the terms used in ODCS

### SQL

Status: this is approved as part of ODCS v3.

An individual SQL query that returns a single number or boolean value that can be compared. 
The SQL query must be in the SQL dialect of the provided server.

For comparison, the Soda [List of threshold keys](https://docs.soda.io/soda/data-contracts-checks.html#list-of-threshold-keys) should be adopted.

```yaml
quality:
  - type: sql 
    query: |
      SELECT COUNT(*) FROM ${table} WHERE ${column} IS NOT NULL
    mustBeLessThan: 3600    
```

Note: _This actually represents the `metric_query` type in the Soda reference._

### Custom

Status: this is **NOT YET** part of ODCS v3.

This enables vendor-specific checks, such as Great Expectations, dbt-tests, or Montecarlo. Any properties should be acceptable here, whether the property is written in YAML, JSON, XML, or even uuencoded binary.

#### Soda Example

```yaml
quality:
- type: custom
  engine: soda
  implementation: |
        type: duplicate_percent  # Block
        columns:                 # passed as-is
          - carrier              # to the tool
          - shipment_numer       # (Soda in this situation)
        must_be_less_than: 1.0   #
```

#### Great Expectation Example

```yaml
quality:
- type: custom
  engine: great-expectations
  implementation: |
    type: expect_table_row_count_to_be_between # Block
    kwargs:                                    # passed as-is
      minValue: 10000                          # to the tool
      maxValue: 50000                          # (GX in this situation)
```


## General changes

* `templateName` is now a custom property.
* `toolName` is obsolete, replaced by `type=custom; engine: <engine name>`.
* `scheduleCronExpression` is replaced by `schedule` and `scheduler`. `scheduleCronExpression: 0 20 * * *` becomes `schedule: 0 20 * * *` and `scheduler: cron`.


## Common elements to each rule

* `name`: optional; a short name
* `description`: optional; a description of what the rule does/applies to.
* `dimension`: optional; valid values are case insensitive (based on the 7 dimensions from Data QoS & EDM Council):
  * `Accuracy` (synonym `Ac`),
  * `Completeness` (synonym `Cp`),
  * `Conformity` (synonym `Cf`),
  * `Consistency` (synonym `Cs`),
  * `Coverage` (synonym `Cv`),
  * `Timeliness` (synonym `Tm`),
  * `Uniqueness` (synonym `Uq`).
* `method`: optional; values are open and include `reconciliation`.
* `severity`: optional; values are open and include and can be `error,` `warning,` or `info`.
* `businessImpact`: optional; values are open and include: `operational` and `regulatory`.
* `tags`: optional.
* `customProperties`: optional.
* `authoritativeDefinitions`: optional.


## Decision

### Working Group

#### Proposed options by working group

##### Option 1

Description: Using YAML multi-line string to specify "implementation" that can be executed by assigned engine (like Soda, Great Expectations etc)
Example:
```yaml
quality:
- type: custom
  engine: soda
  implementation: |
        type: duplicate_percent  # Block
        columns:                 # passed as-is
          - carrier              # to the tool
          - shipment_numer       # (Soda in this situation)
        must_be_less_than: 1.0   #
```
###### Pros

* Contract remains self contained with quality rules defined as part of implementation section
* Quality rules remain an integral part of the contract

###### Cons

* Some of the engines require rule specifications that are non-YAML (JSON etc) mixing different formats in the contract may hamper readability of the contract
* Some engines have very verbose specification that may increase length/size of the contract
* Change in quality rules is interpreted as change in overall contract

##### Option 2

Description: Using URI to external quality rules specification that can be executed by assigned engine (like Soda, Great Expectations etc)
Example:
```yaml
quality:
- type: custom
  engine: soda
  implementationURI: s3://location-of-the-soda-quality-rules-spec.yml
```
###### Pros

* Length/size of the contract remains in check and is predictable
* Quality rules specification remains reusable as a part of contract and directly on execution engine (not sure if this would a pro really)

###### Cons

* Quality rules may be modified without the knowledge of contract maintainers and may impact usability of data

---

* Support for multiple engines or data quality providers: yes.
* Default engine: yes.
* No need to specify engine when one engine: yes.
* No need to specify engine when default engine: yes.
* Support for DQ rules at the field/attribute level: yes.
* Support for DQ rules at the object/field level: yes.
* Support for DQ rules for multiple tables/objects or cross table/object: No.
* Support for a core set of rules at the standard level: Yes, could be inspired by [Soda's implementation](https://docs.soda.io/soda/data-contracts-checks.html).

### TSC

TBD

## Consequences

Breaking change over v2.x.

## References

N.A

## Rejected options

### Keep current specifications v2.2.1

#### Table-level

```YAML
dataset:
  - table: tab1
# ...
    quality:
      - code: countCheck                                       # Required, name of the rule
        templateName: CountCheck                               # NEW in v2.1.0 Required
        description: Ensure row count is within expected range # Optional
        toolName: Elevate                                      # Required
        toolRuleName: DQ.rw.tab1.CountCheck                    # NEW in v2.1.0 Optional (Available only to the users who can change in source code edition)
        dimension: completeness                                # Optional
        type: reconciliation                                   # Optional NEW in v2.1.0 default value for column level check - dataQuality and for table level reconciliation
        severity: error                                        # Optional NEW in v2.1.0, default value is error
        businessImpact: operational                            # Optional NEW in v2.1.0
        scheduleCronExpression: 0 20 * * *                     # Optional NEW in v2.1.0 default schedule - every day 10 a.m. UTC
        customProperties:                                      # Optional
          - property: FIELD_NAME
            value: rcvr_id
          - property: FILTER_CONDITIONS
            value: 1>0
```

#### Column-level

```YAML
dataset:
  - table: tab1
      - column: rcvr_id
        isPrimary: true # NEW in v2.1.0, Optional, default value is false, indicates whether the column is primary key in the table.
        businessName: receiver id
# ...
        quality:
          - code: nullCheck
            templateName: NullCheck
            description: column should not contain null values
            toolName: Elevate
            toolRuleName: DQ.rw.tab1_2_0_0.rcvr_cntry_code.NullCheck
            dimension: completeness # dropdown 7 values
            type: dataQuality
            severity: error
            businessImpact: operational
            scheduleCronExpression: 0 20 * * *
            customProperties:
              - property: FIELD_NAME
                value:
              - property: COMPARE_TO
                value:
              - property: COMPARISON_TYPE
                value: Greater than
```

### Example 1 - Implicit call to rule

```YAML
quality:
- code: required
```

* Applied on a column: it will apply a rule called `required` to this column.
* Applied on a table: it will apply a rule called `required` to all the required columns in the table.

### Example 2 - Named parameters

```YAML
quality:
- code: required
  parameters:
  - parameter: tolerance
    value: 2%
```

```YAML
quality:
- code: invalid_count
  engine: soda # namespace, library
  parameters:
  - parameter: valid_values
    value: ['S', 'M', 'L']
  - parameter: must_be_greater_than_or_equal
    value: 10
```

If there is less than 2% of NULL in this field, then the rule is valid.

### Example 3 - Support for SQL 

```YAML
schema: # ex dataset
- object: StockTable
  physicalName: stock_v1
  attributes: # ex columns
  - field: Quantity
    physicalName: qty
    quality:
    - code: sql
      query: SELECT count(*) FROM ${table} WHERE ${column} >= 0;
```

In this situation, the rule will pass if the result of the `SELECT count(*) FROM stock_v1 WHERE qty >= 0;` is true.

```YAML
schema: # ex dataset
- object: StockTable
  physicalName: stock_v1
  quality:
  - code: sql
    query: SELECT count(*) FROM ${table} WHERE ${column} >= 0;
```

In this situation, the rule should fail: ${column} cannot be identified.
