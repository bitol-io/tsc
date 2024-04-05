# TBD

Champion: Manuel Destouesse

## Context

Data consumers come to the data contract for information on the types.
Data consumers might not be too tech-savy. They might not understand what a `VARCHAR(2)` is.

## Decision

TBD

## Options

### Option 1: Logical Type System

| Type      | Options                            | Example                      |
|-----------|------------------------------------|------------------------------|
| string    | minLength<br>maxLength<br>encoding | "hello","world"              |
| integer   | min<br>max                         | 1,2,3                        |
| number    | min<br>max<br>precision<br>scale   | -1.23,9.876                  |
| character | encoding                           | 'a','b','c'                  |
| boolean   |                                    | true/false                   |
| binary    | minLength<br>maxLength             |                              |
| datetime  | format<br>timezone                 | MM/dd/yyyy hh:mm:ss tt<br> - |
| date      | format<br>timezone                 | MM/dd/yyyy<br> -             |
| time      | format<br>timezone                 | hh:mm:ss<br> -               |
| file      | type                               | csv,json,parquet             |


##### Example:
```
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

### Option 3: Both

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

### Option 5: OpenAPI/Swagger

[OpenAPI/Swagger data types](https://swagger.io/docs/specification/data-models/data-types/)

| Type                    | Options                                                                           | Example                      |
|-------------------------|-----------------------------------------------------------------------------------|------------------------------|
| string                  | minLength<br>maxLength<br>pattern<br>format                                       |                              |
| number                  | multipleOf<br>minimum<br>maximum<br>exclusiveMinimum<br>exclusiveMaximum          |                              |
| integer                 | multipleOf<br>minimum<br>maximum<br>exclusiveMinimum<br>exclusiveMaximum          |                              |
| object                  | properties<br>required<br>minProperties<br>maxProperties<br>readOnly<br>writeOnly |                              |
| array                   | items<br>minItems<br>maxItems<br>uniqueItems                                      |                              |
| boolean                 |                                                                                   |                              |
| file (OpenAPI 2.0 only) |                                                                                   |                              |


# Sketchpad

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


