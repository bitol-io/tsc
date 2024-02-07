# Removal of stakeholders

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

1. We remove the section for v3.
2. We add a required, top level field named `owner`

```yaml
# use email addresses, team names, any name you like
owner: daustin
```

## Consequences

- This is a major breaking change. Can only be part of v3.
- A migration is necessary for existing contracts to convert the current owner to the top level field `owner`
- Users that really need this information need to use the extension mechanism to describe them. But they'll might have to duplicate the information about the current owner.
- Tooling can now easily find out the current owner of a data contract, instead of computing it based on dateIn/dateOut and the Role = `"Owner"` convention.

## Alternatives

### Option: Add Owner as an Alternative to Stakeholders

#### Decision

We could add "owner" at the top level and decide that it's superseed by the block if the block is here and this is a v2.3 change that is not breaking.

#### Consequences

- Is a nonbreaking change that can be introduced quickly
- The superseding mechanism makes the standard more complicated
