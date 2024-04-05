# Data Quality

Champion: Jean-Georges Perrin

## Summary

Build more genericity around the definition of the data quality rules.

## Motivation

* The current (v2.x) implementation of data quality rules is too close to the PayPal tooling.

## Design and examples

* Assumes the new schema structure (RFC 0004, option D).
* Standard block that can be applied at the table (object) level or at the column (field) level.
* Support for SQL directly in the contract.
* Support for `customProperties`.

Changes:

* `templateName` is now a custom property.
* `toolName` is optional.

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

> The decision made by the TSC.

## Consequences

> The consequences of this decision.

## References

> Prior art, inspiration, and other references you used to create this based on what's worked well before.
