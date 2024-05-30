# Data Types

Champion: Manuel Destouesse

## Summary

Define data types for fields so that both consumers and producers have a common understanding of the data types expected.

## Motivation

Data consumers can come to the data contract for information on the types.
Data consumers might not be too tech-savy. They might not understand what a `VARCHAR(2)` is.

- **We favor a small standard over a large one**
  - Define minimal set of data types required
  - Allow for additional information about data types to be optional (i.e. what type of integer int32, int64, etc.)

- **We favor interoperability over readability**
  - Common understanding of data types for producers and consumers of the data contract
  - Retain information about source system data types to allow for direct interactions with data sources to still occur

## Design and examples

### Option 1: Logical Type System

| Type      | Options                            | Example                       |
|-----------|------------------------------------|-------------------------------|
| string    | minLength<br>maxLength<br>encoding | "hello","world"               |
| integer   | min<br>max                         | 1,2,3                         |
| number    | min<br>max<br>precision<br>scale   | -1.23,9.876                   |
| character | encoding                           | 'a','b','c'                   |
| boolean   |                                    | true/false                    |
| binary    | minLength<br>maxLength             |                               |
| datetime  | format<br>timezone                 | MM/dd/yyyy hh:mm:ss tt<br> -  |
| date      | format<br>timezone                 | MM/dd/yyyy<br> -              |
| time      | format<br>timezone                 | hh:mm:ss<br> -                |
| file      | type                               | csv,json,parquet              |
| object    | properties<br>required             | {"users": {"name": "pflook"}} |


#### Example
```yaml
- column: name
  logicalType: string
  options:
    min: 5
    max: 25

- column: date_of_birth
  logicalType: date
  options:
    format: YYYY-MM-DD
            
- column: last_connection
  logicalType: datetime
  options:
    format: MM/dd/yyyy hh:mm:ss tt

- column: opt_in_sms
  logicalType: boolean
```

#### Consequences
- Ease of use
- Technology agnostic

### Option 2: Physical Type System

Types defined by the physical type system.

#### Consequences
- TBD

### Option 3: Both Physical and Logical (OpenApi / Swagger)

#### Logical type
- [OpenAPI/Swagger data types](https://swagger.io/docs/specification/data-models/data-types/)
- New datetime type
 

| Type                    | Options                                                                                     | Example                      |
|-------------------------|---------------------------------------------------------------------------------------------|------------------------------|
| string                  | minLength<br>maxLength<br>pattern<br>format                                                 |                              |
| number                  | format<br>multipleOf<br>minimum<br>maximum<br>exclusiveMinimum<br>exclusiveMaximum          |                              |
| integer                 | format<br>multipleOf<br>minimum<br>maximum<br>exclusiveMinimum<br>exclusiveMaximum          |                              |
| object                  | format<br>properties<br>required<br>minProperties<br>maxProperties<br>readOnly<br>writeOnly |                              |
| array                   | items<br>minItems<br>maxItems<br>uniqueItems                                                |                              |
| boolean                 |                                                                                             |                              |
| file (OpenAPI 2.0 only) |                                                                                             |                              |
| datetime                | format<br>minimum<br>maximum                                                                |                              |
|

##### string / number / integer / object /array boolean 
Same behaviour as for OpenApi 

##### datetime
format: this should remain open so both options below are allowed: 


###### Option 1: formated string following [RFC 3339, section 5.6,](https://datatracker.ietf.org/doc/html/rfc3339#section-5.6)

examples:

 | Description                                                               | Value                    | Example                      |
|---------------------------------------------------------------------------|--------------------------|------------------------------|
| Year                                                                      | yyyy                     | 2024                         |
| Year and month                                                            | yyyy-MM                  | 2024-05                      |
| Complete date                                                             | yyyy-MM-dd               | 2025-05-13                   |
| Complete date, hours and minutes                                          | yyyy-MM-ddThh:mmTZD      | 2024-05-13T19:20+01:00       |
| Complete date, hours, minutes and seconds                                 | YYYY-MM-ddThh:mm:ssTZD   | 2024-05-13T19:20:30+01:00    |
| Complete date, hours, minutes, seconds and a decimal fraction of a second | yyyy-MM-ddThh:mm:ss.sTZD | 2024-05-13T19:20:30.45+01:00 |


###### Option 2: The format could also be a custom sting like:
- yyyy/MM/dd 
- MM/dd/yyyy
- using separator '/'



#### Physical type
- Types defined by the physical type system (Postgres, Mysql, ...).

#### Consequences
- TBD

### Option 4: Align with JSON Schema

[JSON Schema data types](https://json-schema.org/understanding-json-schema/reference/type)

| Type     | Options                                                                                                                                         | Example                      |
|----------|-------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------|
| string   | minLength<br>maxLength<br>pattern<br>format                                                                                                     |                              |
| number   | multipleOf<br>minimum<br>maximum<br>exclusiveMinimum<br>exclusiveMaximum                                                                        |                              |
| integer  | multipleOf<br>minimum<br>maximum<br>exclusiveMinimum<br>exclusiveMaximum                                                                        |                              |
| object   | properties<br>patternProperties<br>additionalProperties<br>unevaluatedProperties<br>required<br>propertyNames<br>minProperties<br>maxProperties |                              |
| array    | items<br>prefixItems<br>unevaluatedItems<br>contains<br>minContains<br>maxContains<br>minItems<br>maxItems<br>uniqueItems                       |                              |
| boolean  |                                                                                                                                                 |                              |
| null     |                                                                                                                                                 |                              |

#### Example

```yaml
- column: name
  logicalType: string
  logicalTypeOptions:
    minLength: 5
    maxLength: 25
    pattern: "[a-z]{5,25}"

- column: date_of_birth
  logicalType: string
  logicalTypeOptions:
    format: yyyy-MM-dd
            
- column: last_connection
  logicalType: string
  logicalTypeOptions:
    format: MM/dd/yyyy hh:mm:ss tt

- column: opt_in_sms
  logicalType: boolean

- column: details
  logicalType: object
  logicalTypeOptions:
    properties:
      - column: sum_amount
        logicalType: number
      - column: sum_amount_today
        logicalType: number
        logicalTypeOptions:
          minimum: 0

- column: previous_transactions
  logicalType: array
  logicalTypeOptions:
    items: number
```

### Option 5: OpenAPI/Swagger with Date

[OpenAPI/Swagger data types](https://swagger.io/docs/specification/data-models/data-types/)

| Type                           | Options                                                                           | Example                      |
|--------------------------------|-----------------------------------------------------------------------------------|------------------------------|
| string                         | minLength<br>maxLength<br>pattern<br>format                                       |                              |
| number                         | multipleOf<br>minimum<br>maximum<br>exclusiveMinimum<br>exclusiveMaximum          |                              |
| integer                        | multipleOf<br>minimum<br>maximum<br>exclusiveMinimum<br>exclusiveMaximum          |                              |
| object                         | properties<br>required<br>minProperties<br>maxProperties<br>readOnly<br>writeOnly |                              |
| array                          | items<br>minItems<br>maxItems<br>uniqueItems                                      |                              |
| boolean                        |                                                                                   |                              |
| file (OpenAPI 2.0 only)        |                                                                                   |                              |
| date (new type not in OpenAPI) | format<br>minimum<br>maximum<br>exclusiveMinimum<br>exclusiveMaximum              |                              |

#### Example

```yaml
- column: name
  logicalType: string
  logicalTypeOptions:
    minLength: 5
    maxLength: 25
    pattern: "[a-z]{5,25}"

- column: date_of_birth
  logicalType: date
  logicalTypeOptions:
    format: "yyyy-MM-dd"
    example: 2024-01-25
            
- column: last_connection
  logicalType: date
  logicalTypeOptions:
    format: "yyyy-MM-dd'T'HH:mm:ss'Z'"
    example: 2024-01-25T15:23:45Z

- column: opt_in_sms
  logicalType: boolean

- column: details
  logicalType: object
  logicalTypeOptions:
    properties:
      - column: sum_amount
        logicalType: number
      - column: sum_amount_today
        logicalType: number
        options:
          minimum: 0

- column: previous_transactions
  logicalType: array
  logicalTypeOptions:
    items: number
```


## Alternatives

> Rejected alternative solutions and the reasons why.

## Decision

> The decision made by the TSC.

## Consequences

> The consequences of this decision.

## References

> Prior art, inspiration, and other references you used to create this based on what's worked well before.


## Sketchpad

```
- column: name
    logicalType: string
        options:
            min: 5
            max: 25
    physicalType: varchar[25]
        toolingContext:
            database: Oracle
    physicalType: char[25]
        toolingContext:
            database: dBase III

```

Needs for both logical & physical type:
* Table generation.
   * Need for multiple tools/context in the same contract.
* Data quality as validating the schema/schema changes.
   * Need for multiple tools/context in the same contract.
* Creation of a reader dashboard (or any data-consuming application).
* Pushing metadata to data catalogs.
* Security aspects: masking, etc. work on logical type.
* Compatibility with OpenAPI.
* Support for object-type column/data element.
* Data contract -> Kafka registry/schema, etc.

Notes/ideas:
* Bindings like in AsyncAPI.
* Should we support on SQL? NO.

Exclusions:
* Consumer use cases -- TBD.
