# Improve Support Channels

Champion: Simon Harrer

## Summary

Enhance the support channels specification to make URL optional, add custom properties support, extend scope examples with notifications, and include Google Chat as a supported tool.

## Motivation

The current support channels specification requires a URL for every support channel, which may not always be applicable or available. Some support channels might be managed through alternative identification methods or might not have a direct URL.

Additionally, organizations often need to store custom metadata about their support channels that isn't covered by the standard fields. The ability to add custom properties would provide flexibility for organization-specific requirements.

The current scope examples could benefit from including "notifications" as a common use case, and Google Chat should be included as a supported tool alongside existing options like Slack, Teams, and Discord.

This RFC aligns with our guiding values of flexibility and extensibility while maintaining backward compatibility.

## Design and examples

### Key Changes

1. **URL becomes optional**: The `url` field is no longer required, allowing for support channels that don't have direct URLs
2. **Custom Properties**: Add support for `customProperties` to store organization-specific metadata
3. **Extended scope**: Add "notifications" to the allowed scope values
4. **Google Chat support**: Add "googlechat" to the allowed tool values

### Minimalist Example
```yaml
support:
  - channel: help-desk
    tool: googlechat
    scope: interactive
  - channel: notifications-only
    tool: email
    scope: notifications
    customProperties:
      priority: high
      region: us-west
```

### Complex Example
```yaml
support:
  - channel: interactive-support
    tool: googlechat
    scope: interactive
    url: https://chat.google.com/room/AAAAxxx
    description: "Primary support channel for interactive help"
    customProperties:
      businessHours: "9am-5pm EST"
      escalationLevel: "L1"
  - channel: system-notifications
    tool: slack
    scope: notifications
    url: https://workspace.slack.com/archives/C1234567890
    description: "Automated system notifications and alerts"
    customProperties:
      alertType: "system"
      severity: "info"
  - channel: announcements
    tool: email
    scope: announcements
    url: mailto:announcements@company.com
    customProperties:
      distribution: "all-users"
      frequency: "weekly"
  - channel: legacy-support
    description: "Legacy phone-based support"
    customProperties:
      phone: "+1-800-555-0123"
      hours: "24/7"
```

## Updated Schema

**Required:**
- `channel`

**Optional:**
- `url` (previously required, now optional)
- `description`
- `tool`: email|slack|teams|discord|ticket|googlechat|other
- `scope`: interactive|announcements|issues|notifications
- `invitationUrl`
- `customProperties`: object with arbitrary key-value pairs

## Decision

> The decision made by the TSC.

## Consequences

### Positive
- Increased flexibility for organizations with diverse support channel structures
- Backward compatibility maintained (existing configurations remain valid)
- Support for organization-specific metadata through custom properties
- Better alignment with modern communication tools (Google Chat)
- More comprehensive scope coverage with notifications

### Potential Concerns
- Optional URL might reduce discoverability of support channels
- Custom properties could lead to inconsistent implementations across organizations

## References

- RFC 0006: Standard support channels (original specification)
- Google Chat API documentation for URL structure patterns
- Industry best practices for support channel categorization
