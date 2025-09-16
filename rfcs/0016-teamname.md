# Team Name

Champion: Simon Harrer

Slack: https://data-mesh-learning.slack.com/archives/C096UHZFC6A 

## Motivation

Teams and any team memberships have a different lifecycle than a data contract, and are typically managed separately - but still need to be linked.

How to identify the owning team in a data contract?

Because if we can identify the owning team, we can externalize the definition of the team itself.
And one team can own multiple data contracts, and adding a team member to a team would affect the `team` members list of its owned data contracts.

## Status Quo / Workaround

This is a workaround created by a vendor and an end-user customer:

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

### Option A: Simple solution

Proposed solution: a new field to identify the owning team as a first class citizen.

```yaml
owner: my-team
```

Recommendation to derive a teamId from the teamName using some form of uri.

### Option B: Introduce a team object with members

```yaml
team:
  name: my-team
  description: The team owning the data contract
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
    name: David Austin
    dateIn: 2022-10-01
```

While this structure would be a breaking change, we can keep conformity with v3.0, by adding this structure as an alternative to JSON Schema, using `oneOf`/`anyOf`.

### Option C: Do nothing

Recommend the workaround.

### Option D: 

* Default element is of type `member`.
* Add type `team`.
* Practical organizational recommendation: Do not mix type `team` and type `member`.
* Add support for `customProperties` and `authoritativeDefinitions` blocks.
* `description` is optional but universal.

```yaml
team:
  - type: team
    name: my-team
    role: xyz
    description: xxxx
    customProperties: ...
    authoritativeDefinitions: ...
    members:
    - username: ...
    - username: ...
  - username: ceastwood # type: member
    role: Data Scientist
    dateIn: 2022-08-02
    dateOut: 2022-10-01
    replacedByUsername: mhopper
  - username: mhopper # type: member
    role: Data Scientist
    dateIn: 2022-10-01
  - username: daustin # type: member
    role: Owner
    description: Keeper of the grail
    name: David Austin
    dateIn: 2022-10-01
```

### Option E:

```yaml
team:
  - authoritativeDefinitions:
     - type: teamDefinition
       url: my-team
```

### Option F:

```yaml
team:
- team: my-team
  role: owner
- username: ceastwood
  role: Data Scientist
  dateIn: 2022-08-02
  dateOut: 2022-10-01
  replacedByUsername: mhopper
```

## References

- The Data Contract Specification uses an `owner` field at the top level.
