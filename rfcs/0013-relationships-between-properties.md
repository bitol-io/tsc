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

A property can have a list of relationships. Each relationship is defined by a reference to another property. Properties are referenced through dot notation, keeping the context in mind.

```yaml
schema:
- name: users
  properties:
  - name: id
    # NEW
    relationships:
    - ref: account_number # property, refers to users.account_number
      type: foreignKey
    - ref: accounts.account_number # object.property
      type: foreignKey
    - ref: accounts.address.street # object.property.property
      type: foreignKey
```

Examples:

```yaml
schema:
- name: users
  properties:
  - name: id
    type: integer
    primaryKey: true
  - name: supervisor_id
    type: integer
    relationships:
    - ref: id # same schema
      type: foreignKey
- name: posts
  properties:
  - name: id
    type: integer
    primaryKey: true
  - name: user_id
    type: integer
    relationships:
    - ref: users.id # other schema
      type: foreignKey
```

```yaml
schema:
- name: account
  properties:
  - name: account_number
    type: integer
    primaryKey: true
  - name: account_type
    type: integer
    primaryKey: true
- name: sub_accounts
  properties:
  - name: sub_account_number
    type: integer
    primaryKey: true
  - name: ref_number
    type: integer
    relationships:
    - ref: account.account_number
      type: foreignKey
  - name: ref_type
    type: integer
    relationships:
    - ref: account.account_type
      type: foreignKey
```

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
