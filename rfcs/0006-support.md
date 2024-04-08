# Standard support channels

Champion:Jean-Georges Perrin.

## Needs
* Better desciption of communication channel.
* Not linked to a specific technology.
* Scope of the channel: interactive vs. announcements (Both at a global retailer, specific channel for automation to avoid noise).
* Support for public announcement channels.

## Support
The updated system should support several tools and scope.

## Decision

### Examples

#### Minimalist
```YAML
support:
  - channel: # Simple Slack communication channel
      - tool: slack
        url: https://aidaug.slack.com/archives/C05UZRSBKLY
  - channel: # Simple distrubution list
      - tool: email
        url: datacontract-ann@bitol.io
```

#### Complex
```YAML
support:
  - channel:
      - tool: teams
        scope: interactive
        url: https://bitol.io/teams/channel/my-data-contract-interactive
  - channel:
      - tool: teams
        scope: announcements
        url: https://bitol.io/teams/channel/my-data-contract-announcements
  - channel:
      - tool: email
        scope: announcements
        url: datacontract-ann@bitol.io
```

### Allowed values


## Consequences

Breaking change for ODCS v2.2.1.
