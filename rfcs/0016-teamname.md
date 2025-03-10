# Team Name

Champion: Simon Harrer

## Motivation

Teams and any team memberships have a different lifecycle than a data contract, and are typically managed separately - but still need to be linked.

How to identify the owning team in a data contract?

Because if we can identify the owning team, we can externalize the definition of the team itself.
And one team can own multiple data contracts, and adding a team member to a team would affect the `team` members list of its owned data contracts.

## Status Quo / Workaround

This is a workaround created by a vendor and a end-user customer:

```yaml
# identify owning team via organization-specific identifier "my-team"
customProperties:
- property: owner
  value: "my-team"
# include team members based on that
team:
- username: ceastwood
  role: Data Scientist
  dateIn: 2022-08-02
  dateOut: 2022-10-01
  replacedByUsername: mhopper
- username: mhopper
  role: Data Scientist
  dateIn: 2022-10-01
- username: daustin
  role: Owner
  comment: Keeper of the grail
  name: David Austin
  dateIn: 2022-10-01
```


## Design and examples

Team name is unique within an organization (context).

### Option A: Simple solution (Non-breaking Change: v3.1)

Proposed solution: a new field to identify the owning team as a first class citizen.

```yaml
teamName: my-team
# alternatives
# teamId: my-team
# owner: my-team

# include team based on teamName
team:
- username: ceastwood
  role: Data Scientist
  dateIn: 2022-08-02
  dateOut: 2022-10-01
  replacedByUsername: mhopper
- username: mhopper
  role: Data Scientist
  dateIn: 2022-10-01
- username: daustin
  role: Owner
  comment: Keeper of the grail
  name: David Austin
  dateIn: 2022-10-01
```

Recommendation to derive a teamId from the teamName using some form of uri.

### Option B: Introduce a team object (Breaking Change: v4)

```yaml
team:
  name: my-team
  # potential to add more properties to the team (name, description, ...)
  # lookup members via a system
  members:
  - username: ceastwood
    role: Data Scientist
    dateIn: 2022-08-02
    dateOut: 2022-10-01
    replacedByUsername: mhopper
  - username: mhopper
    role: Data Scientist
    dateIn: 2022-10-01
  - username: daustin
    role: Owner
    comment: Keeper of the grail
    name: David Austin
    dateIn: 2022-10-01
```

### Option C: Do nothing

Recommend the workaround.

## References

- The Data Contract Specification uses an `owner` field at the top level.
