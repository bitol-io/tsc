# RFC-0060: Open Data Quality Dimensions

Champion: Patrick Beitsma.

Authors: Patrick Beitsma, Yassin Chibrani-derks.

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

Change the `dimension` property in a quality block from a fixed enum to an open string. Existing and established data quality dimensions remain valid and are documented as recommended examples, but users may provide dimensions from their own quality model.

## Motivation

A fixed vocabulary prevents a quality rule from expressing dimensions that are required by a domain, organization, or quality framework but are not included in the current enum. This creates a choice between using an imprecise existing value and using organization-established but currently not supported dimensions.

An open string preserves interoperability for commonly used dimensions while allowing organizations and tools to adopt additional dimensions without waiting for a standard revision. This supports the Bitol guiding values of interoperability, extensibility, and practical standardization: the standard provides shared vocabulary without making it exhaustive.

The change applies to quality blocks in ODCS and the corresponding quality-dimension field in OORS. Other standards, including OSDS, are affected if they reuse this quality-block definition and should align their schema descriptions accordingly.

## Design and examples

Quality `dimension` remains optional. When present, its value is a non-empty identifier matching `^[a-z_-]+$`: lowercase ASCII letters, underscores, and dashes only. Values are not restricted to an enum, so producers may define additional dimensions using this syntax. Consumers MUST preserve unknown dimension values and MUST NOT reject a document solely because its dimension is not one of the recommended examples.

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
    dimension: completeness
```

### Example 2: Framework-specific dimensions

```yaml
quality:
  - name: Address validity
    metric: invalidValues
    mustBe: 0
    dimension: validity
    description: Every address conforms to the agreed postal-address format.
  - name: Service accessibility
    metric: availability
    mustBeGreaterOrEqualTo: 99.9
    dimension: accessibility
    description: The service is reachable during the agreed operating window.
```

### Example 3: Current enum values in one quality block

```yaml
quality:
  - metric: invalidValues
    mustBe: 0
    dimension: conformity
  - metric: duplicateValues
    mustBe: 0
    dimension: uniqueness
  - metric: nullValues
    mustBe: 0
    dimension: completeness
  - metric: freshness
    mustBeLessThan: 3600
    dimension: timeliness
```

The same open-string rule applies when the property is represented in an OORS result or a quality block reused by another standard. Existing values remain valid, and a consumer that does not recognize a newly introduced value can still process the surrounding quality result.

## Proposed schema changes

The following changes to the ODCS and OORS schemas are proposed as part of this RFC. They are **not** yet applied to the standards; they will be applied to the target schema(s) if and when the TSC approves this RFC.

1. Replace the ODCS quality `dimension` enum with a non-empty string:

```json
"dimension": {
  "type": "string",
  "minLength": 1,
  "pattern": "^[a-z_-]+$",
  "description": "The data quality dimension measured by the rule. The value is open and may use an organization- or framework-specific vocabulary.",
  "examples": ["accuracy", "completeness", "conformity", "consistency", "coverage", "timeliness", "uniqueness", "validity"]
}
```

The existing `enum` containing `accuracy`, `completeness`, `conformity`, `consistency`, `coverage`, `timeliness`, and `uniqueness` is removed. The field remains optional.

2. Define the OORS result dimension using the same open-string constraint:

```json
"dimension": {
  "type": "string",
  "minLength": 1,
  "pattern": "^[a-z_-]+$",
  "description": "The data quality dimension being observed. The value is open and may use an organization- or framework-specific vocabulary.",
  "examples": ["accuracy", "completeness", "conformity", "consistency", "coverage", "timeliness", "uniqueness", "validity"]
}
```

## Alternatives

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

Validation becomes less restrictive: implementations may validate that the value is a non-empty string, but must not validate it against a closed list. Tooling that currently offers enum-based suggestions should present the example values as suggestions rather than enforced values.

The meaning of a custom dimension is no longer guaranteed by the field name alone. Producers should document custom dimensions and use stable spelling and casing. Existing documents using the current enum remain valid without migration.

## References

- [RFC-0007: Data Quality](approved/odcs-v3.0.0/0007-data-quality.md) - defines the current seven-value ODCS dimension vocabulary.
- [RFC-0018: Open Observability Results Standard (OORS)](0018-oors.md) - defines the OORS result dimension as a string and gives data quality examples.
- [RFC-0027: Unstructured Data Quality for ODCS & ODPS](0027-unstructured-data-quality.md) - demonstrates quality dimensions beyond the original fixed vocabulary.
- [DAMA-DMBOK: Data Management Body of Knowledge](https://www.dama.org/content/body-knowledge) - source of the commonly used DAMA data quality dimensions.

## Appendix A: Compatibility and implementation guidance

The change is backward compatible for serialized documents: every value accepted before remains accepted. It is a schema-validation compatibility change because validators must replace an enum constraint with a string constraint.

Implementations should expose the recommended values as autocomplete or documentation examples, while allowing arbitrary non-empty strings. Implementations should compare values consistently and avoid silently normalizing casing; if a product chooses to normalize values for search or reporting, it should retain the original serialized value.
