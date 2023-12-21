# TBD

## Context

Data consumers come to the data contract for information on the types.
Data consumers might not be too tech-savy. They might not understand what a `VARCHAR(2)` is.

## Decision

TBD

## Options

### Option 1: Logical Type System

```
String : Length, Encoding ( ASCII, UTF-8, EBCDIC,…)
Integer : Min value, Max value, Number of digits (if applicable)
Float : Min value, Max value, Number of digits (if applicable), Number of decimals (if applicable)
Character : Encoding (ASCII, UTF-8, EBCDIC,…)
Boolean :True/False
Binary: Length
File: File type (png, jpg, …)
DateTime: Format, Time zone (if applicable)
Date: Format, Time zone (if applicable)
Time: Format, Time Zone (if applicable)
```

#### Consequences
- TBD

### Option 2: Physical Type System

Types defined by the physical type system.

#### Consequences
- TBD

### Option 3: Both

#### Consequences
- TBD

