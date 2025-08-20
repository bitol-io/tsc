# Extensions to customProperties and authoritativeDefinitions

Champion: Jean-Georges Perrin

Slack: https://data-mesh-learning.slack.com/archives/C089S376YGM (We use [#bitol-wg](https://data-mesh-learning.slack.com/archives/C089S376YGM)).

Jira: https://bitol-io.atlassian.net/browse/ODCS-54.

## Summary

Add more fields to the customProperties and authoritativeDefinitions block, used accross ODCS and ODPS.

## Motivation

Alignment between the two standards.

## Design and examples

### Custom Properties

| Key                          | UX label             | Required | Description                                                                                                        |
|------------------------------|----------------------|----------|--------------------------------------------------------------------------------------------------------------------|
| customProperties             | Custom Properties    | No       | A list of key/value pairs for custom properties.                                                                   |
| customProperties.property    | Property             | No       | The name of the key. Names should be in camel case–the same as if they were permanent properties in the contract.  |
| customProperties.value       | Value                | No       | The value of the key.                                                                                              |
| customProperties.description | Description          | No       | Optional description. *NEW*                                                                                        |

### Authoritative definitions

| Key                                  | UX label          | Required | Description                                                                                                                                                   |
|--------------------------------------|-------------------|----------|---------------------------------------------------------------------------------------------------------------------------------------------------------------|
| authoritativeDefinitions             | Link              | No       | A list of type/link pairs for authoritative definitions.                                                                                                      |
| authoritativeDefinitions.type        | Definition type   | Yes      | Type of definition for authority.  Valid values are: `businessDefinition`, `transformationImplementation`, `videoTutorial`, `tutorial`, and `implementation`. |
| authoritativeDefinitions.url         | URL to definition | Yes      | URL to the authority.                                                                                                                                         |
| authoritativeDefinitions.description | Description       | No       | Optional description. *NEW*                                                                                                                                   |


## Alternatives

None.

## Decision

Expected in August for inclusion in ODCS v3.0.3.
Approved by the TSC on 2025-08-19, will go in v3.1.0.


## Consequences

None.

## References

None.
