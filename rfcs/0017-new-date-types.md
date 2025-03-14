# Title

Champion: Peter Flook

## Summary

Include new date data types (datetime and time) to `logicalDataType`.

## Motivation

Users expect to have a refined version of the date data type to specify whether it is either date, datetime or time.

- **We favor a small standard over a large one**
  - We now introduce more options for `logicalType`
- **We favor interoperability over readability**
  - Improves interoperability as `logicalType` could more directly map to the `physicalType` in some circumstances
  - Referring to `format` in the `logicalTypeOptions` may not be easy to deduce a `physicalType`

## Design and examples

Current vs new logical data types:

| Current | New          | Example             | Note                                                                           |
|---------|--------------|---------------------|--------------------------------------------------------------------------------|
| string  | string       | 'abc123'            |                                                                                |
| number  | number       | 123.1               |                                                                                |
| integer | integer      | 123                 |                                                                                |
| boolean | boolean      | true                |                                                                                |
| array   | array        | [1,2,3]             |                                                                                |
| object  | object       | {name: 'hello'}     |                                                                                |
| date    | date         | 2020-12-31          |                                                                                |
| date    | **datetime** | 2020-12-31 01:01:01 | Date with time component(s) such as hours, minutes, seconds, millis, time zone |
| date    | **time**     | 01:01:01            | Only time components(s) such as hours, minutes, seconds, millis, time zone     |

## Alternatives

> Rejected alternative solutions and the reasons why.

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

### Data sources with no date data type

- [Protobuf data types](https://protobuf.dev/programming-guides/proto3/#scalar)
- [Avro data types](https://avro.apache.org/docs/1.11.1/specification/#primitive-types)
- [OpenAPI spec data types](https://swagger.io/docs/specification/v3_0/data-models/data-types/)

