# RFC-0060: Open Data Quality Dimensions

Champion: Patrick Beitsma.

Authors: Patrick Beitsma, Yassin Chibrani-derks, Diego Carvallo

Slack: [rfc-0060-dq-Dimensions](https://data-mesh-learning.slack.com/archives/C0BSSV2GKQF)

GitHub issue: https://github.com/bitol-io/tsc/issues/105

Applies to:
* [x] ODCS - Open Data Contract Standard
* [ ] ODPS - Open Data Product Standard
* [x] OORS - Open Observability Results Standard
* [ ] OOCS - Open Orchestration and Control Standard
* [ ] OMMS - Open Maturity Model Standard
* [ ] OMDS - Open Metadata Difference Standard
* [?] OSDS - Open Semantic Definition Standard

## Summary

Change the `dimension` property in a quality block from a fixed enum to an array of open-strings. Existing and established data quality dimensions values remain valid and are documented as recommended examples, but users may provide dimensions from their own quality model.

## Motivation

A fixed vocabulary prevents a quality rule from expressing dimensions that are required by a domain, organization, or quality framework but are not included in the current enum. This creates a choice between using an imprecise existing value and using organization-established but currently not supported dimensions.

An array of open strings preserves interoperability for commonly used dimensions while allowing organizations and tools to adopt additional dimensions without waiting for a standard revision. This supports the Bitol guiding values of interoperability, extensibility, and practical standardization: the standard provides shared vocabulary without making it exhaustive.

The change applies to quality blocks in ODCS and the corresponding quality-dimension field in OORS. Other standards, including OSDS, are affected if they reuse this quality-block definition and should align their schema descriptions accordingly.


## Design and examples

Quality `dimension` remains optional. When present, its value is an array of non-empty identifiers matching `^[a-z_.-]+$`: lowercase ASCII letters, underscores, dots (usable as a separator to indicate hierarchy), and dashes only. Values are not restricted to an enum, so producers may define additional dimensions using this syntax. Consumers MUST preserve unknown dimension values and MUST NOT reject a document solely because its dimension is not one of the recommended examples.

The current ODCS enum values remain recommended examples:

- `accuracy`
- `completeness`
- `conformity`
- `consistency`
- `coverage`
- `timeliness`
- `uniqueness`

The following additional DAMA data quality dimensions are also recommended examples:

- `validity`

These lists are examples, not a replacement enum. Framework-specific dimensions such as `freshness`, `relevance`, or `accessibility` are valid when they are meaningful in the producer's quality model. A producer may document the meaning of a dimension in the quality rule's `description`, through an authoritative definition, or in an accompanying quality vocabulary.

### Example 1: Minimal

```yaml
quality:
  - metric: nullValues
    mustBe: 0
    dimension: 
      - completeness
```

### Example 2: Framework-specific dimensions

```yaml
quality:
  - name: Address validity
    metric: invalidValues
    mustBe: 0
    dimension:
      - intrinsic.validity
    description: Every address conforms to the agreed postal-address format.
  - name: Service accessibility
    metric: availability
    mustBeGreaterOrEqualTo: 99.9
    dimension:
      - operational.accessibility
      - operational.validity
    description: The service is reachable during the agreed operating window.
```

### Example 3: Current enum values in one quality block

```yaml
quality:
  - metric: invalidValues
    mustBe: 0
    dimension:
      - conformity
  - metric: duplicateValues
    mustBe: 0
    dimension:
      - uniqueness
  - metric: nullValues
    mustBe: 0
    dimension:
      - completeness
  - metric: freshness
    mustBeLessThan: 3600
    dimension:
      - timeliness
```

The same array of open-strings rule applies when the property is represented in an OORS result or a quality block reused by another standard. Existing values remain valid, and a consumer that does not recognize a newly introduced value can still process the surrounding quality result.

## Proposed schema changes (breaking change)

The following changes to the ODCS and OORS schemas are proposed as part of this RFC. They are **not** yet applied to the standards; they will be applied to the target schema(s) if and when the TSC approves this RFC.

1. Replace the ODCS quality `dimension` enum with an array of non-empty strings:

```json
"dimension": {
  "type": "array",
  "items": {
    "type": "string",
    "minLength": 1,
    "pattern": "^[a-z_.-]+$",
    "description": "The data quality dimension measured by the rule. The value is open and may use an organization- or framework-specific vocabulary.",
    "examples": ["accuracy", "completeness", "conformity", "consistency", "coverage", "timeliness", "uniqueness", "validity"]
  },
  "minItems": 1
}
```

The existing `enum` containing `accuracy`, `completeness`, `conformity`, `consistency`, `coverage`, `timeliness`, and `uniqueness` is removed. The field remains optional.

2. Define the OORS result dimension using the same array of open-strings constraint:

```json
"dimension": {
  "type": "array",
  "items": {
    "type": "string",
    "minLength": 1,
    "pattern": "^[a-z_.-]+$",
    "description": "The data quality dimension measured by the rule. The value is open and may use an organization- or framework-specific vocabulary.",
    "examples": ["accuracy", "completeness", "conformity", "consistency", "coverage", "timeliness", "uniqueness", "validity"]
  },
  "minItems": 1
}
```
## Alternative schema changes (non-breaking change)

### Option: Non-breaking alternative with single string and array of strings

To avoid breaking existing consumers, one could consider introducing a non-breaking alternative that allows both a single open string value (representing the old enum) and the new array of open-strings. This allows to maintain backward compatibility and continue the 3.x.x line of the standard. Deprecated single string usage should be removed when going to the 4.x.x line of the standard.

```json
"dimension": {
  "oneOf": [
    {
      "type": "string",
      "minLength": 1,
      "pattern": "^[a-z_.-]+$",
      "deprecated": true,
      "description": "The single string form is deprecated in favor of the array of strings."
    },
    {
      "type": "array",
      "items": {
        "type": "string",
        "minLength": 1,
        "pattern": "^[a-z_.-]+$"
      },
      "minItems": 1
    }
  ]
}
```

### Option: Non-breaking alternative with both the old enum and array of strings

Option: keep the enum instead of changing it into a single open string forcing consumers to adapt the array-based approach sooner when they need alternative or custom dimensions.

```json
"dimension": {
  "oneOf": [
    {
      "type": "string",
      "description": "The key performance indicator (KPI) or dimension for data quality. (Deprecated in favor of the array of strings.)",
      "enum": ["accuracy", "completeness", "conformity", "consistency", "coverage", "timeliness", "uniqueness"],
      "deprecated": true
    },
    {
      "type": "array",
      "items": {
        "type": "string",
        "minLength": 1,
        "pattern": "^[a-z_.-]+$"
      },
      "minItems": 1
    }
  ]
}
```

## Alternatives

### Just allow any non-empty string

Rejected. Although it offers easy backward compatibility for existing consumers, it lacks a way for representing multiple dimensions simultaneously.

### Extend the enum

Rejected because every new quality framework or domain-specific dimension would require another standard revision, while existing consumers would still be unable to process the new value without an update.

### Keep the enum and use custom properties

Rejected because the quality dimension is a first-class semantic property and should remain directly discoverable, queryable, and portable across standards. Encoding it in custom properties would fragment interoperability and make it harder to discover and process quality dimensions in a standard way.

### Replace the enum with a centrally managed vocabulary

Deferred. A shared vocabulary may be useful later, but it should not be a prerequisite for allowing valid dimensions that are outside the current list.

## Decision

> The decision made by the TSC.

## Consequences

Producers can represent organization-specific and framework-specific quality dimensions without changing the standard. Consumers must accept and preserve unknown dimension values and should not assume that the recommended examples are exhaustive.

Validation becomes less restrictive: implementations may validate that the value is an array of non-empty strings, but must not validate it against a closed list. Tooling that currently offers enum-based suggestions should present the example values as suggestions rather than enforced values.

The meaning of a custom dimension is no longer guaranteed by the field name alone. Producers should document custom dimensions and use stable spelling and casing. Existing documents using the current enum will be valid after migrating to the new array format without the need to change the value.

## Additional Considerations

The (quality)`dimension` property is not the only property that may benefit from being represented as an array of open strings. Other properties like `classification` (see [RFC 0053 Classification-sensitive for details](0053-classification-sensitivity.md)) and `domain` could also adopt a similar approach if there is a need for extensibility beyond a fixed enum or single open-string value.

Organizations often use their own quality dimensions, classification schemes and domain definitions, but also are looking for ways to enforce the (array) values to comply with their internal standards, such as through controlled vocabularies/taxonomies, and/or governance processes. Currently, the standard does not provide mechanisms for enforcing such compliance, leaving it to individual organizations and tools to implement these controls as needed.

Leveraging `authoritativeDefinitions` block for these properties is currently not supported as these properties do not support referencing external authoritative definitions. But _misusing_ authoritative definitions for validation or enforcement purposes is discouraged, as it deludes the intended purpose of authoritative definitions as a mechanism to provide additional information (and externally defined additional properties and values).

## References

- [RFC-0007: Data Quality](approved/odcs-v3.0.0/0007-data-quality.md) - defines the current seven-value ODCS dimension vocabulary.
- [RFC-0018: Open Observability Results Standard (OORS)](0018-oors.md) - defines the OORS result dimension as a string and gives data quality examples.
- [RFC-0027: Unstructured Data Quality for ODCS & ODPS](0027-unstructured-data-quality.md) - demonstrates quality dimensions beyond the original fixed vocabulary.
- [DAMA-DMBOK: Data Management Body of Knowledge](https://www.dama.org/content/body-knowledge) - source of the commonly used DAMA data quality dimensions.

## Appendix A: Compatibility and implementation guidance

The change is NOT fully backward compatible for serialized documents: every value accepted before remains accepted but the structure of the data has changed (from a single string to an array of strings).

Implementations should expose the recommended values as autocomplete or documentation examples, while allowing arbitrary non-empty strings in the array. Implementations should compare values consistently and avoid silently normalizing casing; if a product chooses to normalize values for search or reporting, it should retain the original serialized value.
