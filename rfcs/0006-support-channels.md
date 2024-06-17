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
      - url: https://aidaug.slack.com/archives/C05UZRSBKLY
  - channel: # Simple distribution list
      - url: mailto:datacontract-ann@bitol.io
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
        invitationUrl: https://bitol.io/teams/channel/my-data-contract-announcements-invit
  - channel: # all announcement
      - name: All announcement for all data contracts
        tool: teams
        scope: announcements
        url: https://bitol.io/teams/channel/all-announcements
  - channel:
      - tool: email
        scope: announcements
        url: mailto:datacontract-ann@bitol.io
  - channel:
      - tool: ticket
        url: https://bitol.io/ticket/my-product
```

### Allowed values

Requires: 
* `support`
* `channel`
* `url`

Optional:
* `name`
* `tool`: email|slack|teams|discord|ticket|other
* `scope`: interactive|announcements|issues
* `invitationUrl`

### Out of scope

* Hours of support: can be implemented as an extension if needed for now.

## Notes

* For Teams, we could have:
  * a link to the channel `url` and
  * a link to the invitation `invitationUrl` (if you're already connected, it brings you to the default channel).

* For Slack, we could have:
  * a link to the channel `url` and
  * a link to the invitation - `invitationUrl`.

## Consequences

Breaking change for ODCS v2.2+: replaces:

```YAML
productDl: product-dl@ClimateQuantum.org
productSlackChannel: '#product-help'
productFeedbackUrl: https://product-feedback.com/sellers
```
