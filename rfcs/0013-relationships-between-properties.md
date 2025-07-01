# Relationships between Properties (Foreign Keys)

Champion: Simon Harrer

[Slack channel](https://data-mesh-learning.slack.com/archives/C08BUU150LF). Some early discussions were in the [WG channel](https://data-mesh-learning.slack.com/archives/C089S376YGM).

## Summary

## Motivation

We want to express relationships between properties. Think of foreign keys in SQL. 

Examples:
- one property references another table.column
- two columns reference two other columns in another table

The relationships might not be implemented by foreign keys as foreign keys are not available in the storage. 
Think of CSV files in an AWS S3 storage. There is no mechanism to define those foreign key constraint. 
But relationships still exists.

We would like to check those relationships automatically via tooling, generate in SQL DDL, and visualize them in a diagram (think ER-diagram).

## Design and examples

A property can have a list of relationships. Each relationship is defined by a reference to another property. We use the reference mechanism from RFC 9.

```yaml
schema:
- name: users
  properties:
  - name: id
    # NEW
    relationships:
    - type: foreignKey
      to: users.account_number
      # from: users.id (implied)
    - to: accounts.account_number
      from: users.id
      # type: foreignKey (default)
    - type: foreignKey
      to: accounts.address.street # (nested)
      from: users.id
      customProperties:
        description: "This is a foreign key to the street address of the account."
        cardinality: "one-to-many"
        label: "is-part-of"
  - name: account_number
  relationships:
  - from: users.id
    to: users.account_number
    type: foreignKey
- name: accounts
  properties:
  - name: account_number
  - name: address
    properties:
      - name: street
```

### Fields

| Key          | UX Label   | Required                                           | Description                                                                                                                                                                                                                                    |
|--------------|------------|----------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| relationships | Relationships | optional                                           | Array. A list of relationships to other properties. Each relationship can have a `to` and `type` field.                                                                                                                                        |
| relationships.to | To         | yes                                                | A reference to a property in the same or another schema. Use the shorthand notation `<schema_name>.<property_name>` Shorthand can be nested.                                                                                                   |
| relationships.type | Type       | no (defaults to `foreignKey`)                      | The type of relationship. Defaults to `foreignKey`. Current options are: `foreignKey`                                                                                                                                                          |
| relationships.from | From       | yes (not required on property-level relationships) | A reference to a property in the same or another schema. Use the shorthand notation `<schema_name>.<property_name>`. Shorthand can be nested. Shorthand could be only the `<property_name>`. Is optional when specified at the property level. |
| relationships.customProperties | Custom Properties | optional                                           | Any additional properties that are not part of the standard schema. This can be used to add metadata or other information relevant to the relationship.                                                                                        |

### Shorthand

The shorthand notation allows for a more concise way to define relationships. It can be used in the `to` and `from` fields of a relationship.
The shorthand notation is `<schema_name>.<property_name>`.

## FAQ

Do we need additional info to mark a relationship as one-to-many or one-to-one?
- No, we can derive it from the data contract based on the required/unique properties.

What about m:n?
- Data Contracts are not a modeling tool. That's why we leave it out.

Why calling it relationships?
- Used by other tools (dbt)
- More general than foreign keys

Why we want an abbreviated syntax?
- Easier to read

Why do we skip composite keys?
- Simpler
- Not necessary as we can have multiple references

What about lineage?
- TBD

What's the difference to authoritative definitions?
- Authoritative definitions point to something outside of data contracts. 
- Relationships point to properties within data contracts, and stays within the "metadata" model defined by all data contracts of an organization.

What about business definitions/field?
- We define the business definitions as part of the data contract. Therefore, relationships
- authoritativeDefinitions vs. relationships?

Out of scope
- Relationships between data contracts (e.g., lineage)

## Alternatives

- A more general solution with support for lineage or business definitions. One could think of adding a type attribute for each relationship.

## Decision

TBD

## Consequences

TBD

## References

- How it is implemented in the Data Contract Specification: https://datacontract.com/#field-object
