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

| Current | New           | Example             | Note                                                                           |
| ------- | ------------- | ------------------- | ------------------------------------------------------------------------------ |
| string  | string        | 'abc123'            |                                                                                |
| number  | number        | 123.1               |                                                                                |
| integer | integer       | 123                 |                                                                                |
| boolean | boolean       | true                |                                                                                |
| array   | array         | [1,2,3]             |                                                                                |
| object  | object        | {name: 'hello'}     |                                                                                |
| date    | date          | 2020-12-31          |                                                                                |
| date    | **datetime**  | 2020-12-31 01:01:01 | Date with time component(s) such as hours, minutes, seconds, millis, time zone |
| date    | **timestamp** | 2020-12-31 01:01:01 | Same as datetime                                                               |
| date    | **time**      | 01:01:01            | Only time components(s) such as hours, minutes, seconds, millis, time zone     |

## Alternatives

| Alternative                                                                            | Description                                  |
| -------------------------------------------------------------------------------------- | -------------------------------------------- |
| [A: Keep single 'date' type](#alternative-a-keep-single-date-type-with-format-options) | Use format options for variants              |
| [B: Add specific types (proposed)](#alternative-b-add-specific-types-proposed)         | Add datetime, timestamp, time                |
| [C: Timezone-aware types](#alternative-c-timezone-aware-types)                         | Add timestamp, timestamp with timezone, time |
| [D: Full ISO 8601 alignment](#alternative-d-full-iso-8601-alignment)                   | Match all ISO 8601 variants                  |

### Alternative A: Keep single 'date' type with format options

Using a single 'date' type with format options in logicalTypeOptions.

**Logical Types:**

- date (with format options)

```yaml
- name: created_at
  logicalType: date
  logicalTypeOptions:
    format: yyyy-MM-dd HH:mm:ss
```

**Pros:**

- Maintains simplicity with fewer logical types
- Format options provide flexibility
- Consistent with JSON Schema approach

**Cons:**

- Requires additional configuration
- Less direct mapping to physical types
- Format options may be overlooked
- Inconsistent with common data systems that have distinct types

### Alternative B: Add specific types (proposed)

Add distinct logical types for different date/time variants as proposed in this RFC.

**Logical Types:**

- date
- datetime
- timestamp
- time

```yaml
- name: created_date
  logicalType: date # e.g. 2020-12-31
- name: created_at
  logicalType: datetime # e.g. 2020-12-31 01:01:01
- name: start_time
  logicalType: time # e.g. 01:01:01
```

**Pros:**

- Clear semantic distinction between different date/time concepts
- Direct mapping to common physical types in many systems
- Consistent with industry standards (Postgres, Iceberg, Cassandra, Avro)
- No additional configuration needed

**Cons:**

- Increases number of logical types
- Potential confusion between datetime and timestamp types
- May require conversion logic in some implementations
- Lacks explicit timezone handling

### Alternative C: Timezone-aware timestamp types

Add distinct logical types with explicit timezone handling capabilities.

**Logical Types:**

- date
- timestamp
- timestamptz
- time
- timetz

```yaml
- name: created_date
  logicalType: date # e.g. 2020-12-31
- name: created_at
  logicalType: timestamp # e.g. 2020-12-31 01:01:01
- name: event_time
  logicalType: timestamptz # e.g. 2020-12-31 01:01:01+00:00
- name: start_time
  logicalType: time # e.g. 01:01:01
- name: meeting_time
  logicalType: timetz # e.g. 01:01:01+00:00
```

**Pros:**

- Explicit timezone handling
- Aligns with systems like Postgres, Iceberg that distinguish timestamp vs timestamptz
- More precise semantics for time-sensitive data
- Better support for global/distributed applications

**Cons:**

- Further increases number of logical types
- More complex mapping to systems without timezone support
- Requires additional knowledge about timezone handling
- May introduce conversion challenges

### Alternative D: Full ISO 8601 alignment

Adopt all ISO 8601 date and time format variations as distinct logical types.

**Logical Types:**

- date
- year-month
- year
- datetime
- datetime-with-timezone
- time
- time-with-timezone
- duration

```yaml
- name: created_at
  logicalType: datetime-with-timezone
```

**Pros:**

- Complete coverage of all date/time scenarios
- Precise specification for each variant
- Strong international standard alignment

**Cons:**

- Excessive number of logical types
- Increased complexity for users
- Many types may rarely be used
- Difficult to map cleanly to all physical systems

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
