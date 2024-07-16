# Replacing stakeholers with productTeam structure

Champion: Simon Harrer

The ODCS standard v2.2.1 contains a section about stakeholders.

```yaml
stakeholders:
  - username: ceastwood
    role: Data Scientist
    dateIn: 2022-08-02
    dateOut: 2022-10-01
    replacedByUsername: mhopper
  - username: mhopper
    role: Data Scientist
    dateIn: 2022-10-01
    dateOut: null
    replacedByUsername: null
  - username: daustin
    role: Owner
    dateIn: 2022-10-01
    dateOut: null
    replacedByUsername: null
```

### Reasoning

- The stakeholder section has potential for confusion because it is unclear how to use it: which roles to define, which stakeholders to add, and how that information can be used.
- The stakeholder section captures very niche use cases, such as history of any kind of stakeholder
- The only major use case is defining the current owner. Determining the current owner requires computation based on dateIn/dateOut as well as the role field.
- It's unclear how a list of any kind of stakeholder is helpful for the contract. Other systems are better suited for stakeholder maps or team membership management.
- The history should be given by appropriate tooling and version control.
- This violates our guiding value "We favor a small standard over a large one".

## Status

To be discussed.

## Decision

```yaml
# use email addresses, team names, any name you like
team:
  - username: ceastwood
    role: Product Owner
    contact: ceastwood@sample-org.com
  - username: mhopper
    role: Tech Lead
    contact: @mhopper [Slack]
  - username: apandya
    role: Data Engineer
    contact: @apandya [Teams]
  - username: gdavis
    role: Marketing Data Analyst
    contact: +1 800 201 8904 [Telephone]
```

## Discarded Options

### Option 2

```yaml
# use email addresses, team names, any name you like
productTeam:
  - productOwner: ceastwood
    contact: ceastwood@sample-org.com
  - techLead: mhopper
    contact: @mhopper [Slack]
  - engineer: apandya
    contact: @apandya [Teams]
  - analyst: gdavis
    contact: +1 800 201 8904 [Telephone]
```

## Consequences

- This is a major breaking change. Can only be part of v3.
- A migration is necessary for existing contracts to convert the current stakeholders list to productTeam list
- Tooling can now easily find out the current product team, instead of computing it based on dateIn/dateOut and the Role.
- More flexibility for tooling to interpret productTeam data to meet various persona needs

## External References
- Geospatial Metadata standards (ISO-19115-1:2014) defines `CI_RoleCode` (cf. https://wiki.earthdata.nasa.gov/pages/viewpage.action?pageId=35851076). These are used across several concepts such as responsible party, citation, user, processor, etc (as the role depends on the context ofc).
