# Tags, custom properties and authoritative definitions for SBOMs

Champion: Jean-Georges Perrin.

Authors: Jean-Georges Perrin.

Slack: *replace this text with the link to the dedicated Slack channel*.

GitHub issue: https://github.com/bitol-io/tsc/issues/108

Applies to:
* [ ] ODCS - Open Data Contract Standard
* [x] ODPS - Open Data Product Standard
* [ ] OORS - Open Observability Results Standard
* [ ] OOCS - Open Orchestration and Control Standard
* [ ] OMMS - Open Maturity Model Standard
* [ ] OMDS - Open Metadata Difference Standard
* [ ] OSDS - Open Semantic Definition Standard

## Summary

Allow `tags`, `customProperties` and `authoritativeDefinitions` on `outputPorts[].sbom[]` entries, consistent with how they already work on every other nested object in ODPS.

## Motivation

`sbom` is a closed object (`additionalProperties: false`) carrying only `id`, `type` and `url`. In the same schema, the product root plus `inputPorts`, `outputPorts`, `managementPorts`, `support`, `team` and `team[].members` all carry the three metadata fields. SBOM is the exception, and because the object is closed there is no user-space escape hatch: the document fails validation.

`sbom` is an array, so a product may publish several entries — one per runtime image, one per library set. Today none of them can be labelled, none can carry a vendor-specific identifier, and none can link to the authoritative build record. Hoisting that metadata to the containing output port is the only workaround, and it is ambiguous as soon as there is more than one entry.

Adding the three fields keeps the standard uniform (consistency) and lets tools attach their own context without a standard revision (extensibility). See [Appendix A](#appendix-a-current-state-in-odps-v110) for the current state.

## Design and examples

Add three optional fields to each entry in `outputPorts[].sbom[]`:

| Field                      | Type  | Required | Description                                                       |
| -------------------------- | ----- | -------- | ----------------------------------------------------------------- |
| `tags`                     | array | No       | Tags, as defined for other objects.                               |
| `customProperties`         | array | No       | Vendor-specific key/value pairs, as defined for other objects.    |
| `authoritativeDefinitions` | array | No       | Links to external definitions, as defined for other objects.      |

No existing field changes. The object stays closed; only its property list grows.

### Example 1: Minimal

```yaml
outputPorts:
  - name: transactions
    sbom:
      - type: external
        url: https://mysbomserver/mysbom
        tags: ["runtime"]
```

### Example 2: Structured

```yaml
outputPorts:
  - name: transactions
    sbom:
      - id: sbom-runtime
        type: external
        url: https://mysbomserver/transactions/runtime.cdx.json
        tags: ["runtime", "cyclonedx"]
        customProperties:
          - property: sbomFormat
            value: CycloneDX 1.6
          - property: buildId
            value: build-8842
        authoritativeDefinitions:
          - type: implementation
            url: https://ci.acme.com/builds/8842
      - id: sbom-model
        type: external
        url: https://mysbomserver/transactions/model.spdx.json
        tags: ["model", "spdx"]
        customProperties:
          - property: sbomFormat
            value: SPDX 2.3
```

## Alternatives

1. **Put the metadata on the containing output port.** Rejected: it detaches the metadata from a specific SBOM entry and is ambiguous when a port publishes more than one.
2. **Open the object (`additionalProperties: true`).** Rejected: every other ODPS object is closed, and it would let arbitrary unvalidated keys in rather than reusing the structures tooling already understands.
3. **Overload `type` or the `url` query string.** Rejected: free text that no tool can read structurally.

## Decision

Pending TSC vote.

## Consequences

- Non-breaking change: adds three optional fields to SBOM entries. Every valid v1.1.0 document stays valid.
- Schema: extend `$defs.SBOM` in the ODPS JSON schema, reusing the existing `Tags`, `CustomProperty` and `AuthoritativeDefinition` definitions.
- Docs: three rows in the `outputPorts[].sbom[]` table in `docs/product-information.md`, and an example.

## References

- [RFC 0046 - Custom properties and authoritative definitions for SLAs](approved/odcs-v3.2.0/0046-sla-custom-properties-and-authoritative-definitions.md) — the same argument, applied to ODCS `slaProperties[]`.
- [RFC 0024 - Extensions to customProperties and authoritativeDefinitions](0024-extensions-to-customproperties-authoritativedefinitions.md)
- [RFC 0035 - Extensions](0035-extensions.md)
- [RFC 0010 - ODPS](approved/odps-v0.9.0/0010-odps.md) — where `sbom` was introduced.

## Appendix A: Current state in ODPS v1.1.0

Measured against `schema/odps-json-schema-v1.1.0.json` on `dev-v1.1.0`.

`sbom` exists only on `outputPorts` — not on `inputPorts`, not on `managementPorts`, not at the product root. Widening that placement is out of scope for this RFC.

Which objects carry the three metadata fields:

| Object | `tags` | `customProperties` | `authoritativeDefinitions` |
| --- | --- | --- | --- |
| root (data product) | yes | yes | yes |
| `inputPorts[]` | yes | yes | yes |
| `outputPorts[]` | yes | yes | yes |
| `managementPorts[]` | yes | yes | yes |
| `support[]` | yes | yes | yes |
| `team[]` / `team[].members[]` | yes | yes | yes |
| `description` | no | yes | yes |
| `synonyms[]` | no | yes | no |
| **`outputPorts[].sbom[]`** | **no** | **no** | **no** |
| `outputPorts[].inputContracts[]` | no | no | no |
| `context[]` | no | no | no |
