# Custom properties and authoritative definitions for SLAs

Champion: Simon Harrer

Applies to:
* [x] ODCS - Open Data Contract Standard
* [ ] ODPS - Open Data Product Standard
* [ ] OORS - Open Observability Results Standard
* [ ] OOCS - Open Orchestration and Control Standard
* [ ] OMMS - Open Maturity Model Standard
* [ ] OMDS - Open Metadata Difference Standard

## Summary

Allow `customProperties` and `authoritativeDefinitions` on SLA elements (`slaProperties[]`), consistent with how they already work on other ODCS objects.

## Motivation

SLAs often need vendor-specific metadata (e.g. an alerting policy id) and links to external definitions (e.g. the formal SLA document or measurement method). Other ODCS objects already support `customProperties` and `authoritativeDefinitions`; SLAs are an inconsistent exception. Adding them keeps the standard uniform and avoids forcing users into the `jdbc`-style workaround of stuffing context into free-text fields.

## Design and examples

Add two optional fields to each entry in `slaProperties[]`:

| Field                      | Type  | Required | Description                                                   |
| -------------------------- | ----- | -------- | ------------------------------------------------------------- |
| `customProperties`         | array | No       | Vendor-specific key/value pairs, as defined for other objects. |
| `authoritativeDefinitions` | array | No       | Links to external definitions, as defined for other objects.  |

### Example 1: Minimal

```yaml
slaProperties:
  - property: availability
    value: 99.9
    unit: "%"
    customProperties:
      - property: alertPolicyId
        value: pol-123
```

### Example 2: Structured

```yaml
slaProperties:
  - property: latency
    value: 100
    unit: ms
    authoritativeDefinitions:
      - url: https://wiki.acme.com/sla/latency
        type: businessDefinition
    customProperties:
      - property: alertPolicyId
        value: pol-123
      - property: owner
        value: data-platform-team
```

## Alternatives

Keep this metadata in `description` or top-level `customProperties`. Rejected: it detaches the metadata from the specific SLA element and prevents structured tooling.

## Decision

**Approved by the TSC on 2026-06-30** for inclusion in ODCS v3.2.0.

## Consequences

- nonbreaking change: adds two optional fields to SLA elements.

## References

- [RFC 0035 - Extensions / customProperties](0035-extensions.md)
- [RFC 0015 - Business definitions / authoritativeDefinitions](0015-business-definitions.md)
