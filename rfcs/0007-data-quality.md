# Data Quality

* Champion: Jean-Georges Perrin
* Discusssion channel: [Slack](https://aidaug.slack.com/archives/C06TC4X4KD1).

## Summary

Build more genericity around the definition of the data quality rules.

## Motivation

* Support for multiple rules engine/data quality providers (support for some edge cases).
* The current (v2.x) implementation of data quality rules is too close to the PayPal tooling.

## Working group

* Jean-Georges Perrin
* Jochen Christ
* Eugene Stakhov
* Martin Meermeyer
* Tom Baeyens 
* Todd Nemanich 
* Manuel Destouesse 
* Peter Flook 

## Design and examples

* Assumes the new schema structure (RFC 0004, option D).
* Standard block that can be applied at the table (object) level or at the column (field) level.
* Support for SQL directly in the contract.
* Support for `customProperties`.
* Easier scheduling.

## Option 1

Changes:

* `templateName` is now a custom property.
* `toolName` is optional.
* `scheduleCronExpression` is now replaced by `schedule` and `scheduler`. `scheduleCronExpression: 0 20 * * *` becomes `schedule: 0 20 * * *` and `scheduler: cron`.

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

## Option 2: Adopt Soda's Data Contract Check Reference

Design principles:
- Quality attributes can be defined on table or column level.
- Support different levels/stages of data quality attributes
  - __Text:__ A human-readable text that describes the quality of the data.
  - __Predefined types:__ A maintained library of commonly-used predefined quality attributes such as `row_count`, `unique`, `freshness`
  - __SQL:__ An individual SQL query that returns a value that can be compared.
  - __Custom:__ Quality attributes that are vendor-specific, such as Great Expectations, dbt tests, or Montecarlo monitors.
- The predefined types should be based on Soda's proposal of [Data contract check reference](https://docs.soda.io/soda/data-contracts-checks.html)


__Text__

A human-readable text that describe the quality of the data. Later in the development process, these might be translated into an executable check (such as `sql`), or checked through an AI engine.

```yaml
quality:
  - type: text
    description: The email address was verified by the system
```

__Predefined types__

We should support a list of predefined types that are commonly used in data quality checks. These should be executable in all common data quality engines.

This makes live simpler for data engineers, as they don't have to write the SQL query themselves.

The experimental [Data contract check reference](https://docs.soda.io/soda/data-contracts-checks.html) are a good starting point.
Instead of creating a new library, I propose to use these as a reference.

Column-level
```yaml
quality:
- type: no_duplicate_values
- type: duplicate_count
  must_be_less_than: 10
  name: Fewer than 10 duplicate names
- type: duplicate_percent
  must_be_less_than: 1
- type: no_invalid_values
  valid_values: ['pounds']
  filter_sql: country = 'UK'
- type: invalid_percent
  must_be_less_than: 3
  valid_values_reference_data:
    dataset: countryID
    column: id
```

Table-level

```yaml
quality:
  - type: row_count
    must_be_between: [100, 120]
    name: Verify row count range
```

I would suggest some minor changes to the Soda reference:
- Instead `checks:` use `quality:` as the key for the list of quality attributes.
- Rename `metric_query` to `sql`.
- Drop `metric_expression`
- In `valid_values_reference_data` adopt `dataset` and `column` to the terms used in ODCS

__SQL__

_(This actually represents the `metric_query` type in the Soda reference)_

An individual SQL query that returns a single number or boolean value that can be compared. 
The SQL query must be in the SQL dialect of the provided server.

For comparison, the Soda [List of threshold keys](https://docs.soda.io/soda/data-contracts-checks.html#list-of-threshold-keys) should be adopted.

```yaml
quality:
  - type: sql
    query: SELECT COUNT(*) FROM ${table} WHERE ${column} IS NOT NULL
    must_be_less_than: 3600    
```

__Custom__

We should also support vendor specific checks, such as Great Expectations, dbt-tests or Montecarlo:
Any properties should be acceptable here.

```yaml
quality:
- type: custom
  engine: great-expectations
  expectation_type: expect_table_row_count_to_be_between
  kwargs:
    min_value: 10000
    max_value: 50000
```

## Alternatives

### Current Spec v2.2.1

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

## Decision

### Working Group

* Support for multiple engines or data quality providers: yes.
* Default engine: yes.
* No need to specify engine when one engine: yes.
* No need to specify engine when default engine: yes.
* Support for DQ rules at the field/attribute level: yes.
* Support for DQ rules at the object/field level: yes.
* Support for DQ rules for multiple tables/objects or cross table/object: TBD.
* Support for a core set of rules at the standard level: TBD, could be inspired by [Soda's implementation](https://docs.soda.io/soda/data-contracts-checks.html).

### TSC

TBD

## Consequences

> The consequences of this decision.

## References

> Prior art, inspiration, and other references you used to create this based on what's worked well before.
