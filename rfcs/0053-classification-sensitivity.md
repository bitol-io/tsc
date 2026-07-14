# RFC-0053: Data category as a distinct dimension

Champion: *TBD — seeking a TSC champion*

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

ODCS's `classification` field is settled practice for sensitivity — the handling tier of an element. What a contract cannot carry today is the other governance dimension: what the data *is*. This RFC adds an optional `category` field holding a single data-category term, at property and schema-object level, and a contract-level `taxonomies` block that identifies, by reference, the schemes governing `category` and `classification` terms. It also makes `classification` available on schema objects, which is what issue [#282](https://github.com/bitol-io/open-data-contract-standard/issues/282) asks for. The change is additive only: no field changes meaning, nothing is removed, every existing contract remains valid. Neither field imposes a vocabulary — a three-rung identification ladder makes terms resolvable against ISO/IEC 19944-1, the IAB Tech Lab Privacy Taxonomy, or any internal taxonomy.

## Motivation

Category ("this is customer contact information") and sensitivity ("restricted") are different dimensions with different owners, change cadences, and consumers — and the category is where regulatory meaning attaches: in GDPR, CCPA, and ISO/IEC 19944-1, obligations follow from what the data is. ODCS has a settled home for sensitivity (`classification`, inherited from the first version of the standard) and no home for category, so category terms get pressed into whatever slot exists. Approved [RFC-0026b](approved/odcs-v3.1.0/0026b-internal-references.md) carries `classification: pii` — a category term in the sensitivity field — next to `classification: restricted` in the same contract. [RFC-0015](0015-business-definitions.md)'s pending example fuses both dimensions into compound tokens (`classification: 02_GDPR_direct`). Organizations that keep the dimensions separate do it in customProperties, which no other party's tooling can read.

The ecosystem already models the two dimensions separately: Snowflake assigns classified columns both a semantic category and a privacy category; Microsoft Purview separates classifications from sensitivity labels. And no single vocabulary fits all parties — ISO/IEC 19944-1 is scoped to the cloud-services ecosystem, and its most prominent public adopter maps an internal taxonomy to the standard's clauses rather than adopting its terms. This RFC therefore defines mechanics, not vocabulary: the slot, the term syntax, and scheme identification. The organization brings the taxonomy.

Guiding values: one optional field and one optional block; scheme identification makes terms portable across organizational boundaries; nothing breaks; terms with declared schemes are computable metadata a CI tool or an agent can check.

## Design and examples

### Fields

| Key | UX label | Type | Required | Description |
|---|---|---|---|---|
| `category` | Data Category | string | No | New. A single data-category term identifying what the data is, drawn from a data taxonomy (internal or published). Property and schema-object level. |
| `classification` | Classification | string | No | Existing; unchanged in meaning (the element's sensitivity per the organization's scheme). This RFC adds schema-object-level availability. |
| `taxonomies` | Taxonomies | array | No | Contract-level. Named, versioned identification of the schemes governing this contract's `category` and `classification` terms — by reference only, never defined inline. |
| `taxonomies[].name` | Name | string | Yes | Short name, usable as a term prefix (e.g. `corp-edt`). |
| `taxonomies[].version` | Version | string | No | Version the contract's terms were authored against. |
| `taxonomies[].url` | URL | string | No | Where the taxonomy is published. |
| `taxonomies[].description`, `taxonomies[].authoritativeDefinitions` | | | No | Human context; standard block (e.g. a published mapping to ISO/IEC 19944-1). |

Both fields may appear on schema objects and on properties; each level is an independent statement about its own element. Cardinality is one term per field per element — cross-scheme equivalence is the taxonomy's responsibility, published once as a mapping, not repeated per column.

### Term resolution — the three-rung ladder

Every rung is valid ODCS; each adds resolvability; none is required.

1. **Bare term.** `category: customer_content`, no declaration — meaningful inside one governance boundary, exactly like a bare `classification` value today.
2. **Anchored.** An `authoritativeDefinitions` entry of `type: Taxonomy` (introduced by approved [RFC-0038](approved/odcs-v3.2.0/0038-context.md)) identifies the governing taxonomy; bare terms SHOULD be terms of it.
3. **Named.** The `taxonomies` block declares one or more schemes; terms MAY carry a `name:` prefix (`iso-19944-1:customer_content`) — needed only when multiple schemes are in play.

A prefixed term resolves to the declared taxonomy of that name; an unprefixed term with exactly one applicable declaration SHOULD be a term of it. Validators MAY verify term membership when the referenced taxonomy is machine-readable; the standard imposes no enforcement. This is what puts the checking on the tool and the CI process: given a declared name, version, and url, a validator can confirm a cited term exists, is not deprecated, and — for ordered sensitivity schemes — rolls up correctly from properties to objects.

### Example

```yaml
taxonomies:
  - name: iso-19944-1
    version: "2020"
    description: ISO/IEC 19944-1 data categories (keys shortened).
  - name: tiers
    version: "1.0.0"
    url: https://governance.example.com/schemes/tiers
    description: public < internal < confidential < restricted

schema:
  - name: transactions
    logicalType: object
    category: iso-19944-1:customer_content
    classification: restricted
    properties:
      - name: customer_email
        logicalType: string
        category: iso-19944-1:customer_content
        classification: restricted
      - name: account_id
        logicalType: string
        category: iso-19944-1:account_data
        classification: confidential
      - name: usage_events
        logicalType: string
        category: iso-19944-1:derived_data
        classification: internal
```

Rung 1 is the same picture with bare terms and no `taxonomies` block.

### Migration and compatibility

- Nothing changes meaning and nothing is removed; every existing contract validates unchanged, and existing `classification` values stay exactly where they are.
- The `classification` description's "can be anything" wording can be aligned with its settled usage in a separate documentation change; this RFC does not depend on it.
- DCS `classification` (defined there as a sensitivity level) maps directly to ODCS `classification`; DCS `pii: true` maps to a `category` term rather than a stringified customProperty.

## Alternatives

1. **Redefine `classification` as the category term** (this RFC's own first draft) or **rename it** ([#137](https://github.com/bitol-io/open-data-contract-standard/issues/137)). Rejected: `classification` as sensitivity is settled practice inherited from the standard's first version; redefining it is a semantic break for every existing user, and removal to avoid misuse would be worse. Adding the missing dimension breaks nobody.
2. **Add a `pii` boolean** (DCS parity). Rejected: "PII" is jurisdiction-specific, identifiability is a mutable spectrum, and the bit is derivable from any serious category scheme while the reverse is not.
3. **Standardize an enum** (e.g. ISO/IEC 19944-1 categories). Rejected: 19944-1 is scoped to the cloud-services ecosystem, ISO/IEC 19944-2 itself expects extension, and any imposed vocabulary is wrong for someone. Mechanics, not vocabulary.
4. **Links only** (`authoritativeDefinitions` without a term field). Rejected: a link is documentation; a term is data. Pointing at a taxonomy does not let a column carry a resolvable value from it.
5. **Tags with structured values** (e.g. `gdpr:r1`, context plus specifier). The prefix idea is right — it is the same syntax as rung 3 — but a tag's context is not declared anywhere, so a CI tool has nothing to resolve it against: no version, no url, no term list. The `taxonomies` block is that missing declaration; customProperties have the same limitation with private names.
6. **Array of category terms per element.** Rejected: dual-annotating every column duplicates what a taxonomy-level mapping states once, and blurs which term governs handling.

## Decision

*To be completed by the TSC.*

## Consequences

- The JSON schema adds `category` (property and object level), `taxonomies` (contract level), and object-level `classification`. Existing contracts are untouched; resolves [#282](https://github.com/bitol-io/open-data-contract-standard/issues/282) directly and [#137](https://github.com/bitol-io/open-data-contract-standard/issues/137)'s underlying confusion without the rename.
- Platform importers/exporters (Snowflake, Purview, DCS tooling) gain unambiguous targets for their dual constructs.
- The same mechanism extends later to allowed and disallowed data-use stipulations at element and product level — a future RFC citing a declared data-use taxonomy through the same block.
- This RFC specifies only how contracts *reference* taxonomies. The natural follow-on is a small ancillary standard — an Open Data Taxonomy Standard (ODTS) — defining the machine-readable taxonomy artifact, and with it a CLI that validates contracts and products against their declared taxonomies in CI. The `taxonomies` block's `name`/`version`/`url` triple is designed as that seam.

## References

- Bitol: issues [#137](https://github.com/bitol-io/open-data-contract-standard/issues/137), [#282](https://github.com/bitol-io/open-data-contract-standard/issues/282); [RFC-0026b](https://github.com/bitol-io/tsc/blob/main/rfcs/approved/odcs-v3.1.0/0026b-internal-references.md) (mixed `restricted`/`pii` example); [RFC-0015](https://github.com/bitol-io/tsc/blob/main/rfcs/0015-business-definitions.md) (compound tokens); [RFC-0038](https://github.com/bitol-io/tsc/blob/main/rfcs/approved/odcs-v3.2.0/0038-context.md) (`Taxonomy` link type).
- ISO/IEC 19944-1:2020 and 19944-2:2022; Microsoft's [Windows diagnostic-data mapping](https://learn.microsoft.com/en-us/windows/privacy/optional-diagnostic-data) to 19944-1 clauses (equivalence mapping, not adoption).
- [Snowflake classification](https://docs.snowflake.com/en/user-guide/classify-intro) (semantic + privacy categories); [Purview classifications vs. sensitivity labels](https://learn.microsoft.com/en-us/purview/data-map-sensitivity-labels-faq).
- [Fideslang / IAB Tech Lab Privacy Taxonomy](https://github.com/IABTechLab/fideslang) (85 hierarchical categories; separate sensitivity scoring); [Data Contract Specification](https://github.com/datacontract/datacontract-specification) (`classification` = sensitivity level; `pii` boolean).
