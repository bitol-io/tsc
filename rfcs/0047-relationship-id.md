# Adding `id` to Relationship Objects

Champion: *TBD (TSC member)*.

Authors: Jean-Georges Perrin.

Slack: *TBD*.

GitHub issue: https://github.com/bitol-io/open-data-contract-standard/issues/277

Applies to:
* [x] ODCS - Open Data Contract Standard
* [ ] ODPS - Open Data Product Standard
* [ ] OORS - Open Observability Results Standard
* [ ] OOCS - Open Orchestration and Control Standard
* [ ] OMMS - Open Maturity Model Standard
* [ ] OMDS - Open Metadata Difference Standard

## Summary

Add an optional `id` field to relationship objects (`RelationshipBase`, surfaced as `RelationshipSchemaLevel` and `RelationshipPropertyLevel`) in ODCS, completing the work started in [RFC-0026a](./approved/odcs-v3.1.0/0026a-reference-id.md).

## Motivation

RFC-0026a added an optional, stable `id` to referenceable array-item objects (servers, schema objects, properties, support, roles, SLA properties, custom properties, etc.). Relationship objects were omitted from its scope and are currently the only array-item object in ODCS without an `id`.

This is inconsistent and prevents relationships from being addressed by stable references — useful for tooling, diffs ([OMDS](./0039-omds.md)), and observability results that need to point at a specific relationship that survives reordering and name changes.

## Design and examples

Add an optional `id` to `RelationshipBase`, following exactly the rules of RFC-0026a: optional, unique within its containing array, stable across versions, and restricted to the same character set.

Minimal example:

```yaml
schema:
  - name: orders
    properties:
      - name: customer_id
        relationships:
          - id: fk_order_customer
            to: customers.id
```

Schema-level example:

```yaml
schema:
  - name: orders
    relationships:
      - id: fk_order_customer
        from: customer_id
        to: customers.id
        type: foreignKey
```

JSON Schema: add to `RelationshipBase.properties`:

```json
"id": {
  "type": "string",
  "description": "Optional stable identifier for the relationship, unique within its containing array. See RFC-0026a."
}
```

Document the field in `references.md` alongside the other relationship fields.

## Alternatives

Keep relying on positional/name-based identification of relationships — rejected as inconsistent with RFC-0026a and brittle under reordering.

## Decision

> The decision made by the TSC.

## Consequences

Optional and additive; no impact on existing contracts. Brings relationship objects in line with every other referenceable object in the standard.

## References

* [RFC-0026a: Adding ID Field for Stable References](./approved/odcs-v3.1.0/0026a-reference-id.md)
* [ODCS issue #277](https://github.com/bitol-io/open-data-contract-standard/issues/277)
