# Lineage

> The champion is the person who is primarily supporting the RFC.

Champion: Dirk Van de Poel

## Summary

Add lineage information to the data contract by referencing upstream dependencies.

## Motivation

Lineage information is quite common and data contracts are well suited to express upstream dependencies. 
This allows for verifying downstream impacted data contracts (hence data products) on data contract changes (that reference the changed data contract).

## Design and examples

To be decided where the upstream depedencies are added, either as table under the fundamentals section or a seperate new Lineage section.
References can be the uuids of referenced upstream data contracts or an alternative based upon version-based compatiblity to prevent requiring updating references on each data contract change.
The level of granularity is also to be defined, this can be on data contract level, dataset level or even more fine-grained (e.g. column level lineage).

## Alternatives

> Rejected alternative solutions and the reasons why.

## Decision

> The decision made by the TSC.

## Consequences

> The consequences of this decision.

## References

> Prior art, inspiration, and other references you used to create this based on what's worked well before.
