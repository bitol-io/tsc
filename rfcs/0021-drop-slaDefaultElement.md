# Drop slaDefaultElement

Champion:  Jochen Christ

## Summary

Drop `slaDefaultElement` in ODCS 4.0, deprecate in 3.1.

## Motivation

This is a single top-level element, next to `slaProperties`.

I don't know what it is, when to use it. It feels not to be used much.

## Design and examples

```
slaDefaultElement: tab1.txn_ref_dt # Optional, default value is partitionColumn.
slaProperties:
  - property: latency # Property, see list of values in DP QoS
    value: 4
    unit: d # d, day, days for days; y, yr, years for years
    element: tab1.txn_ref_dt # This would not be needed as it is the same table.column as the default one
  - property: generalAvailability
    value: 2022-05-12T09:30:10-08:00
  - property: endOfSupport
    value: 2032-05-12T09:30:10-08:00
  - property: endOfLife
    value: 2042-05-12T09:30:10-08:00
  - property: retention
    value: 3
    unit: y
    element: tab1.txn_ref_dt
  - property: frequency
    value: 1
    valueExt: 1
    unit: d
    element: tab1.txn_ref_dt
  - property: timeOfAvailability
    value: 09:00-08:00
    element: tab1.txn_ref_dt
    driver: regulatory # Describes the importance of the SLA: [regulatory|analytics|operational|...]
  - property: timeOfAvailability
    value: 08:00-08:00
    element: tab1.txn_ref_dt
    driver: analytics
```

Recommendation: remove this field with ODCS 4.0, deprecate it with ODCS 3.1.


## Decision

> The decision made by the TSC.

## Consequences

- Breaking change?

