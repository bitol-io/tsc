# Quality Check Identifier

Champion: Simon Harrer

[Slack channel](https://data-mesh-learning.slack.com/archives/C08CN5BCBS5). Some early discussions were in the [WG channel](https://data-mesh-learning.slack.com/archives/C089S376YGM).

## Summary

## Motivation

A quality check is hard to identify at the moment. 

```yaml
quality:
  - type: sql 
    query: |
      SELECT COUNT(*) FROM ${object} WHERE ${property} IS NOT NULL
    mustBeLessThan: 3600    
```

- We could use "name" (but can quickly become hard to stay unchanged),
- or use a custom property (not natively supported, harder to parse),
- or the path in the YAML (but this could change quickly when restructuring the YAML, resorting, ...).

But all are not good solutions.

## Design and examples

Add an optional ID field with type string. We can recommend using UUIDs, but should not restrict it to only them.

```yaml
quality:
  - type: sql 
    query: |
      SELECT COUNT(*) FROM ${object} WHERE ${property} IS NOT NULL
    mustBeLessThan: 3600
    id: 160512f7-358b-487d-8d26-5222534264e4
```

## Alternatives

- Use name for the ID. Make it explicit that it has this property.
- Use UUID for the ID.

## Decision

TBD

## Consequences

TBD

## References

- TBD
