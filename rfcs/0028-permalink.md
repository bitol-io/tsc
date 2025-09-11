# Title

Champion: Jean-Georges Perrin

Slack: N/A.

Jira: https://bitol-io.atlassian.net/browse/ODCS-57.

## Summary

Add optional `permalink` to the header/fundamentals of ODCS/ODPS.

## Motivation

Hint the consumer of the contract/descriptor where is its source of truth. 

## Design and examples

```yaml
# Data Contract
permalink: https://github.com/mydataproduct/bitol/06af81d9-94b4-4d07-be55-cfef22561f6d.odcs.yaml
```

```yaml
# Data Product
permalink: https://github.com/mydataproduct/bitol/be98f68f-2b3b-4392-9192-51a53d7c5c78.odps.yaml
```

## Alternatives

N/A

## Decision

> The decision made by the TSC.

## Consequences

None.

## References

N/A.