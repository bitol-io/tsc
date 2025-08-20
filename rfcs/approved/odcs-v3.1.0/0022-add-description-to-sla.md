# Add description to SLA

Champion: Simon Harrer

## Summary

Introduce an optional `description` field for SLA (Service Level Agreement) entries in ODCS to provide human-readable context or clarification about the SLA property.

## Motivation

SLA properties in ODCS currently define measurable quality and lifecycle characteristics (such as latency, availability, retention) in a strictly structured format.  

While these properties are precise, they can lack context for business users, compliance officers, or operational staff who are not deeply familiar with the data product.  
An optional description would allow authors to document the rationale, scope, or assumptions behind the SLA, improving transparency and communication.

Example use cases:
- Clarifying if the latency SLA applies to business days only
- Explaining the legal or regulatory reason for a retention period
- Documenting operational constraints affecting time of availability

## Design and examples

Proposed schema addition:

```yaml
slaProperties:
  - property: latency
    value: 4
    unit: d
    element: tab1.txn_ref_dt
    description: "Data must be available for analytics within 4 business days after transaction date."
  - property: retention
    value: 3
    unit: y
    element: tab1.txn_ref_dt
    description: "Retention period required by financial regulation XYZ."
```

### Details
- **Type**: `string`
- **Optional**: Yes  
- **Purpose**: Provide a free-text explanation or context for the SLA property.
- **Breaking change**: None (fully backward-compatible).

## Decision

Approved by the TSC on 2025-08-19.


## Consequences

- Improves clarity and usability of SLA definitions for a broader audience.
- No impact on systems that ignore the `description` field.
- Minimal implementation effort required.
