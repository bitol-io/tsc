# TBD

## Context

Data consumers come to the data contract for information on the types.
Data consumers might not be too tech-savy. They might not understand what a `VARCHAR(2)` is.

## Decision

TBD

## Options

### Option 1: Logical Type System

| Type   | Options            | Example   |
|--------|--------------------| -----------   |
| string | length<br>encoding | |
| integer| min<br>max<br>digits | |
| float  | min<br>max<br>digits<br>decimals | |
| character | encoding | |
| boolean | encoding | |
| binary | length | |
| datetime | format<br>timezone | MM/dd/yyyy hh:mm:ss tt<br> -|
| date | format<br>timezone | MM/dd/yyyy<br> - |
| time | format<br>timezone | hh:mm:ss<br> - |
| file | type | |


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


