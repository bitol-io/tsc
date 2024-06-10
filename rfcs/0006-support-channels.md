# Standard support channels

Champion: Jean-Georges Perrin (jgp).

## Needs
* Better description of communication channels.
* Not linked to a specific technology.
* Scope of the channel: interactive vs. announcements (Both at a global retailer, specific channel for automation to avoid noise).
* Support for public announcement channels.
* Not related to ownership.

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
  - channel: # Simple distribution list
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
  - channel: # all announcement
      - tool: teams
        scope: announcements
        url: https://bitol.io/teams/channel/all-announcements
  - channel:
      - tool: email
        scope: announcements
        url: datacontract-ann@bitol.io
```

### Allowed values

## Notes

* For Teams, we could have:
  * a link to the channel (by default after accepting the invitation) and
  * a link to the invitation.

* For Slack, we could have:
  * a link to the channel `url` and
  * a link to the invitation - `invitationUrl`. 

* Tools: email|slack|teams|discord|other



## Consequences

Breaking change for ODCS v2.2.1.
