# RFC-0053: Taxonomy-backed classification assignments

Champion: TBD — seeking a TSC champion

Authors: Thomas Brackin

Slack: `#rfc-0053-data-categorization-and-sensitivity`

GitHub issue: [#80](https://github.com/bitol-io/tsc/issues/80) (relates to [open-data-contract-standard#137](https://github.com/bitol-io/open-data-contract-standard/issues/137) and [open-data-contract-standard#282](https://github.com/bitol-io/open-data-contract-standard/issues/282))

Applies to:

* [x] ODCS - Open Data Contract Standard
* [ ] ODPS - Open Data Product Standard
* [ ] OORS - Open Observability Results Standard
* [ ] OOCS - Open Orchestration and Control Standard
* [ ] OMMS - Open Maturity Model Standard
* [ ] OMDS - Open Metadata Difference Standard
* [ ] OSDS - Open Semantic Definition Standard

## Summary

Allow ODCS `classification` to be either a string or an array of `{taxonomy, value}` objects, on schema objects and properties, including nested properties. Existing strings retain their meaning. The array form references taxonomies defined by the companion [RFC-0054](https://github.com/jarlbrak/tsc/blob/rfc-0054-odts/rfcs/0054-odts.md).

## Motivation

A column may have both a data category, such as contact information, and a handling tier, such as restricted. A single string cannot identify the source taxonomy or represent these independent assignments. A structured form lets tools check each value against its taxonomy while preserving existing contracts.

## Design and examples

### Classification

`classification` remains optional. It accepts a string or an array with the following entry fields:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `taxonomy` | string | Yes | ID of a root `authoritativeDefinitions` entry of type `Taxonomy`. |
| `value` | non-empty string | Yes | Exact term key in the referenced taxonomy. |
| `provenance` | object | No | Current attribution for the assignment's fields; see below. |

Each referenced declaration MUST have a unique ID and resolve to an ODTS `DataTaxonomy` through an immutable, version-specific URL. The URL and the artifact's `name` and `version` identify the taxonomy version. A policy-document link cannot serve as a taxonomy assignment target. Missing, duplicate, unresolved, or non-taxonomy references and unknown term values are invalid.

Array order has no meaning. Each element may have at most one assignment per taxonomy; duplicates are invalid even when their values match. An empty array explicitly records no assignments; an absent field makes no assertion. Assignments apply to their own element without cascading to children or changing parents. ODTS defines retired-term warnings and ranked aggregation checks.

### Example

```yaml
authoritativeDefinitions:
  - id: data-category
    type: Taxonomy
    url: https://governance.example.com/taxonomies/customer-data_v2_0_0.odts.yaml
  - id: handling
    type: Taxonomy
    url: https://governance.example.com/taxonomies/handling_v1_0_0.odts.yaml

schema:
  - name: customers
    logicalType: object
    classification:
      - taxonomy: data-category
        value: customer_content
      - taxonomy: handling
        value: restricted
    properties:
      - name: email
        logicalType: string
        classification:
          - taxonomy: data-category
            value: customer_content.contact_information
          - taxonomy: handling
            value: restricted
      - name: legacy_customer_id
        logicalType: string
        classification: confidential
```

The same forms apply to nested properties. The legacy string is preserved as written; tools do not infer a taxonomy from it.

### Current provenance

An optional `provenance` map records current attribution, keyed by immediate sibling field names. An assignment uses `provenance.value` or `provenance.taxonomy`; a scalar classification uses its owning element's `provenance.classification`.

```yaml
classification:
  - taxonomy: handling
    value: restricted
    provenance:
      value:
        method: manual
        author: urn:example:identity:org-a:user:11111111-1111-4111-8111-111111111111
        vendor: example-tool
        at: '2026-09-22T10:20:00.000Z'
        justification: Handling tier selected for this use.
```

Each provenance entry requires `method` (`manual`, `ai-assisted`, or `automated`), `vendor`, and `at`. A stable, authority-qualified `author` is required for `manual` and `ai-assisted`, and optional for `automated`. Optional `justification` must be nonblank and at most 4,000 UTF-8 bytes.

Trusted manual attribution marks a human pin; other methods remain unpinned. Tools establish trust through authorized authoring or a protected repository. The serialized claim alone does not authenticate its author. Provenance records the current value, without a separate pin flag, confidence, history, or derivation proof. RFC-0054 reuses this structure.

### Compatibility

Existing strings remain valid without changes to their content or meaning. Arrays and schema-object classification require the ODCS version adopting this RFC; earlier schemas remain unchanged. Consumers that do not support the new form must report unsupported input without coercing arrays to strings or discarding entries.

Converting a string requires an explicit taxonomy and term mapping. Tools must not split, prefix, or infer values. DCS `classification` continues to map to the string form; DCS `pii` requires an explicit mapping to an assignment.

## Alternatives

- **Separate `category` field:** adds a field for each classification dimension. The array represents multiple dimensions in one structure.
- **Root `taxonomies` block and prefixed strings:** duplicates the existing `authoritativeDefinitions` mechanism.
- **Tags or custom properties:** require private conventions for taxonomy references and values.
- **Fixed vocabulary or `pii` flag:** cannot express the range of organizational and regulatory taxonomies.
- **Rename or redefine `classification`:** changes existing meaning; the proposed union preserves it.

## Decision

To be completed by the TSC.

## Consequences

- The ODCS schema gains a string-or-array classification field on schema objects and properties.
- Tools can validate multiple assignments against shared taxonomy artifacts.
- Existing scalar contracts remain valid; structured assignments require consumer support.

## References

- [RFC-0038: Context](approved/odcs-v3.2.0/0038-context.md), which introduced taxonomy links in `authoritativeDefinitions`.
- [RFC-0054: Open Data Taxonomy Standard](https://github.com/jarlbrak/tsc/blob/rfc-0054-odts/rfcs/0054-odts.md), companion draft; numbering and filing remain provisional.
- [ODCS schema documentation](https://github.com/bitol-io/open-data-contract-standard/blob/main/docs/schema.md) and ODCS issues [#137](https://github.com/bitol-io/open-data-contract-standard/issues/137) and [#282](https://github.com/bitol-io/open-data-contract-standard/issues/282).
