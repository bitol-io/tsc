# Scheduling SLA Checks

Champion: 

Slack: https://data-mesh-learning.slack.com/archives/C089S376YGM

Jira: https://bitol-io.atlassian.net/browse/ODCS-56

## Summary

Define scheduling for SLAs as we have for DQ.

## Motivation

Alignment with DQ.

## Design and examples

### Example

```YAML
slaDefaultElement: tab1.txn_ref_dt # Optional, default value is partitionColumn. ## SEE RCF 0021
slaProperties:
  - property: latency # Property, see list of values in DP QoS
    value: 4
    unit: d # d, day, days for days; y, yr, years for years
    element: tab1.txn_ref_dt # This would not be needed as it is the same table.column as the default one
    scheduler: cron
    schedule: 0 30 * * *
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

### Definitions

| Key                      | UX label                 | Required                       | Description                                                                                           |
|--------------------------|--------------------------|--------------------------------|-------------------------------------------------------------------------------------------------------|
| slaDefaultElement        | Default SLA element(s)   | No                             | Element (using the element path notation) to do the checks on.                                        |
| slaProperties            | SLA                      | No                             | A list of key/value pairs for SLA specific properties. There is no limit on the type of properties.   |
| slaProperties.property   | Property                 | Yes                            | Specific property in SLA, check the Data QoS periodic table. May requires units.                      |
| slaProperties.value      | Value                    | Yes                            | Agreement value. The label will change based on the property itself.                                  |
| slaProperties.valueExt   | Extended value           | No - unless needed by property | Extended agreement value. The label will change based on the property itself.                         |
| slaProperties.unit       | Unit                     | No - unless needed by property | **d**, day, days for days; **y**, yr, years for years, etc. Units use the ISO standard.               |
| slaProperties.element    | Element(s)               | No                             | Element(s) to check on. Multiple elements should be extremely rare and, if so, separated by commas.   |
| slaProperties.driver     | Driver                   | No                             | Describes the importance of the SLA from the list of: `regulatory`, `analytics`, or `operational`.    |
| slaProperties.scheduler  | Scheduler                | No                             | Name of the scheduler, can be `cron` or any tool your organization support.                           |
| slaProperties.schedule   | Scheduler Configuration  | No                             | Configuration information for the scheduling tool, for `cron` a possible value is `0 20 * * *`.       |

### Recommended Values for Property

For a higher compatibility between products, we recommend using mainly those values for `property`, as defined by [Data QoS](https://medium.com/data-mesh-learning/what-is-data-qos-and-why-is-it-critical-c524b81e3cc1).

**Availability** (Av)
The proportion of time during which the data source is operational and capable of establishing a successful connection via the designated access protocol (e.g., JDBC connect() method), expressed as a percentage over a defined measurement period.

**Throughput** (Th)
The sustained rate at which data can be read from or written to the system, measured in standardized units (e.g., bytes per second or records per second) over a defined interval under specified conditions.

**Error Rate** (Er)
The ratio of erroneous data records or transactions to the total processed records or transactions within a defined time period, expressed as a percentage, and constrained by an agreed-upon maximum tolerance level.

**General Availability** (Ga)
The date or milestone at which a data product or dataset is declared production-ready, fully functional, stable, and supported for consumption by all intended users, as defined by version release policies.

**End of Support** (Es)
The date after which no maintenance, defect correction, or user support will be provided for the dataset or data product, even if the data remains accessible; a successor or replacement version is expected for continued consumption.

**End of Life** (El)
The date after which the dataset or data product will be permanently withdrawn from service, rendering it inaccessible and unsupported.

**Retention** (Re)
The duration for which dataset records or associated documentation are preserved and accessible, determined by contractual, operational, or regulatory requirements.

**Frequency of Update** (Fy)
The regular interval at which the dataset is refreshed or updated (e.g., daily, weekly, monthly), including the scheduled time of availability where applicable.

**Latency** (Ly)
The elapsed time between the generation of a data element and the point at which it becomes available for consumption by downstream systems or users.

**Time to Detect** (Td)
The maximum allowable duration between the occurrence of an issue affecting the dataset and its identification by monitoring or operational processes.

**Time to Notify** (Tn)
The maximum allowable duration between detection of an issue and formal notification to all affected stakeholders or consumers.

**Time to Repair** (Tr)
The maximum allowable duration between detection of an issue and restoration of normal operation, including data accuracy, availability, or performance to agreed service levels.

## Alternatives

TBD

## Decision

Approved by the TSC on 2025-08-19.

## Consequences

Minor change.

## References

DQ in ODCS.