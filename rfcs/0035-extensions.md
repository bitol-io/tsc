# Vendor Attribution for ODCS Custom Properties

**Status:** Draft  
**Champion:** Jean-Georges Perrin  
**Authors:** Gene Azad  

Applies to:
* [x] ODCS - Open Data Contract Standard
* [ ] ODPS - Open Data Product Standard
* [ ] OORS - Open Observability Results Standard
* [ ] OOCS - Open Orchestration and Control Standard
* [ ] OMMS - Open Maturity Model Standard
* [ ] OMDS - Open Metadata Difference Standard

## Summary

This RFC adds an optional `vendor` field to ODCS `customProperties`.

The `vendor` field allows tools and users to associate a custom property with a specific vendor, provider, or external system while continuing to use the existing `customProperties` structure.

## Motivation

ODCS already supports `customProperties` and allows `value` to contain structured data.

However, there is currently no standard way to indicate which vendor or external system a custom property belongs to. As a result:

- vendor-specific custom properties are inconsistent across tools
- tooling cannot reliably identify vendor-owned metadata
- round-tripping vendor-specific data is harder than necessary

Adding a `vendor` field solves this without introducing a new extension mechanism.

## Goals

- Reuse existing `customProperties`
- Add a simple way to identify vendor-specific properties
- Improve interoperability and round-trip safety
- Avoid introducing new extensibility structures

## Non-goals

- Defining vendor-specific schemas
- Standardizing all possible vendor property names
- Creating a centralized vendor registry

## Proposal

Add an optional `vendor` field to ODCS `customProperties`.

### Structure

```yaml
customProperties:
  - property: string
    value: any
    vendor: string   # optional
```

### Semantics

- `property` identifies the custom property name
- `value` contains the custom property value
- `vendor` identifies the vendor or external system associated with the property

If `vendor` is omitted, the property is treated as a generic custom property.

## Vendor rules

When present, `vendor`:

- SHOULD be a stable identifier
- SHOULD be lowercase
- SHOULD follow `^[a-z0-9][a-z0-9-]*$`

Examples:

- `confluent`
- `zeenea`
- `atlan`
- `soda`

Tools MUST preserve unknown vendor values.

## Tooling behavior

- Tools MUST NOT rewrite the `vendor` value
- Tools MUST preserve unknown vendors when possible
- Tools MAY use `vendor` to group, validate, or process vendor-specific custom properties
- Tools SHOULD continue to treat `value` as opaque unless they explicitly support that vendor

## Examples

### Example 1: Vendor-specific structured custom property

```yaml
customProperties:
  - id: theNewOne
    vendor: myPropGovObject
    value:
      myObject:
        mo.x: abc
        mo.y: 123
        mo.array:
          - 1
          - 2
          - 3
    tags:
      - finance
```

### Example 2: Vendor property with authoritative definitions

This example shows how `authoritativeDefinitions` can be used in tandem with the `vendor` property.

```yaml
customProperties:
  - id: data_proc_cluster_name
    property: dataprocClusterName
    value:
      myObject:
        mo.x: abc
        mo.y: 123
        mo.array:
          - 1
          - 2
          - 3
    description: Cluster name for specific applications
    vendor: Confluent
    tags:
      - finance
    authoritativeDefinitions:
      - url: https://catalog.data.gov/vendors.txt
        type: vendorList
        description: List of vendors.
      - url: https://catalog.data.gov/vendor-schema-extension.json
        type: validationRule
        description: Validation rule for the specific data.
  - property: GlossaryRefs
    vendor: Zeenea
    value:
      - category: KPI
        def: Number of Delivered Doses of Vaccine
```

## Alternatives

### Introduce a new extensions container

Rejected because ODCS already has `customProperties`, which is sufficient for vendor-specific values.

## Decision

TBD by TSC

## Consequences

### Positive

- Minimal change to ODCS
- Keeps extensibility in the existing `customProperties` model
- Makes vendor-specific properties easier to identify and preserve

### Trade-offs

- Vendor-specific interpretation remains tooling-dependent
- This RFC does not standardize vendor-specific property schemas

