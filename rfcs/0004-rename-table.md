# Columns are dead, long live fields

- Champion: Andrew Jones

We currently have the following structure:

```yaml
dataset:
  tables:
  - table: orders
    columns:
      - column: order_id
  - table: line_items
    columns:
      - column: order_id
```

Found issues:
- Confusion whether this is a logical or physical data model. Table suggests a physical one. We assume, it means a logical data model.

## Decision

We want the following structure:

```yaml
dataset:
- name: orders
  fields:
    - name: order_id
- name: line_items
  fields:
    - name: order_id
```

## Consequences

- Makes the structure more applicable for users for other data structures
- Nudges users more towards creating a logical model and less a physical model here
- This is a breaking change. Would be targeted for v3.
- Simplified the nested "dataset.tables" structure with a single field "dataset" that has a list of "data-thingies".
- We are very conscious in separating the logical representation of a dataset with the physical representation

