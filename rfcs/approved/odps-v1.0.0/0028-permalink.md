# Permalink

Champion: Jean-Georges Perrin

Slack: https://data-mesh-learning.slack.com/archives/C089S376YGM.

Jira: https://bitol-io.atlassian.net/browse/ODCS-57.

## Summary

Add **optional** `permalink` to the header/fundamentals of ODCS/ODPS.

## Motivation

Hint the consumer of the contract/descriptor where is its source of truth. This would apply to both ODCS and ODPS.

## Design and examples

```yaml
# Data Contract
kind: DataContract
authoritativeDefinitions:
- type: canonical
  url: https://github.com/mydataproduct/bitol/06af81d9-94b4-4d07-be55-cfef22561f6d.odcs.yaml
  description: Canonical URL to the latest version of the contract.
```

```yaml
# Data Product
kind: DataProduct
authoritativeDefinitions:
- type: canonical
  url: https://github.com/mydataproduct/bitol/be98f68f-2b3b-4392-9192-51a53d7c5c78.odps.yaml
  description: Canonical URL to the latest version of the product descriptor.
```

## Alternatives

N/A

## Decision

2025-09-16 Go with authoritativeDefinitions.

## Consequences

None.

## References

N/A.
