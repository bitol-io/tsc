# New Date Types

Champion: Peter Flook

## Summary

Include new date data types (datetime, timestamp and time) to `logicalDataType`.

## Motivation

Users expect to have a refined version of the date data type to specify whether it is either date, datetime or time.

- **We favor a small standard over a large one**
  - We now introduce more options for `logicalType`
- **We favor interoperability over readability**
  - Improves interoperability as `logicalType` could more directly map to the `physicalType` in some circumstances
  - Referring to `format` in the `logicalTypeOptions` may not be easy to deduce a `physicalType`

## Design and examples

Current vs new logical data types:

| Current | New           | Example             | Note                                                                       |
| ------- | ------------- | ------------------- | -------------------------------------------------------------------------- |
| string  | string        | 'abc123'            |                                                                            |
| number  | number        | 123.1               |                                                                            |
| integer | integer       | 123                 |                                                                            |
| boolean | boolean       | true                |                                                                            |
| array   | array         | [1,2,3]             |                                                                            |
| object  | object        | {name: 'hello'}     |                                                                            |
| date    | date          | 2020-12-31          |                                                                            |
| date    | **timestamp** | 2020-12-31 01:01:01 | Date with time component(s)                                                |
| date    | **time**      | 01:01:01            | Only time components(s) such as hours, minutes, seconds, millis, time zone |

New logical type options:

- timezone
- defaultTimezone (default: Etc/UTC)

Add distinct logical types for different date/time variants as proposed in this RFC.

```yaml
- name: created_date
  logicalType: date # e.g. 2020-12-31
- name: created_ts
  logicalType: timestamp # e.g. 2020-12-31 01:01:01, 2020-12-31 01:01:01+09:00
- name: created_time
  logicalType: time # e.g. 01:01:01
- name: created_ts_with_tz
  logicalType: timestamp # e.g. 2020-12-31 01:01:01+09:00
  logicalTypeOptions:
    timezone: true
    defaultTimezone: Asia/Tokyo
- name: created_ts_with_ntz
  logicalType: timestamp # e.g. 2020-12-31 01:01:01
  logicalTypeOptions:
    timezone: false
- name: created_ts_with_tz_format
  logicalType: timestamp # e.g. 2020-12-31T01:01:01+09:00
  logicalTypeOptions:
    timezone: true
    defaultTimezone: Asia/Tokyo
    format: 'yyyy-MM-dd'T'HH:mm:ssZ'
- name: created_time_with_tz
  logicalType: time # e.g. 01:01:01+09:00
  logicalTypeOptions:
    timezone: true
    defaultTimezone: Asia/Tokyo
- name: created_time_with_ntz
  logicalType: time # e.g. 01:01:01
  logicalTypeOptions:
    timezone: false
```

## Alternatives

Alternatives have been rejected. Alternatives were:

- Keep single 'date' type with format options
- Timezone-aware types (i.e. timestamp, time, timestamptz, timetz)
- Full ISO 8601 alignment

## Decision

> The decision made by the TSC.

## Consequences

> The consequences of this decision.

## References

> Prior art, inspiration, and other references you used to create this based on what's worked well before.

### Data sources with date types

- [Postgres date like data type](https://www.postgresql.org/docs/current/datatype-datetime.html)
  - date
  - time
  - time with time zone
  - timestamp
  - timestamp with time zone
  - [Postgres date format options](https://www.postgresql.org/docs/current/functions-formatting.html#FUNCTIONS-FORMATTING-DATETIME-TABLE)
- [Iceberg data types](https://iceberg.apache.org/spec/#primitive-types)
  - date
  - time
  - timestamp
  - timestamptz
  - timestamp_ns
  - timestamptz_ns
- [Cassandra data types](https://cassandra.apache.org/doc/stable/cassandra/cql/types.html)
  - date
  - time
  - timestamp
- [Avro date types](https://avro.apache.org/docs/1.11.1/specification/#logical-types)
  - date
  - time-millis
  - time-micros
  - timestamp-millis
  - timestamp-micros
  - local-timestamp-millis
  - local-timestamp-micros

### Data sources with no date data type

- [Protobuf data types](https://protobuf.dev/programming-guides/proto3/#scalar)
  - [It has Timestamp but not Date](https://protobuf.dev/reference/protobuf/google.protobuf/#timestamp)
- [OpenAPI spec data types](https://swagger.io/docs/specification/v3_0/data-models/data-types/)
