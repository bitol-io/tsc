# Columns are dead, long live fields

- Champion: Andrew Jones

We currently have the following structure:

```yaml
dataset:
  - table: orders
    columns:
      - column: order_id
  - table: line_items
    columns:
      - column: order_id
```

Found issues:
- Confusion whether this is a logical or physical data model. Table suggests a physical one. We assume, it means a logical data model.
- The term dataset is problematic as it has a very specific meaning in the context of GCP, and some fields in the ODCS are GCP specific and also use the term dataset (e.g., `datasetName`).

Open Questions:
- How do we capture other data, like unstructured data (scientific papers in form of PDFs), vectors, ... Do we want to specify unstructured or semistructured data in a data contract?

## Options

### Field and Name

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

#### Consequences

- Makes the structure more applicable for users for other data structures
- Nudges users more towards creating a logical model and less a physical model here
- This is a breaking change. Would be targeted for v3.
- We are very conscious in separating the logical representation of a dataset with the physical representation

### Other option

The structure is fine. We want different naming.

```yaml
dataset:
- dataobject: orders
  type: table
  fields:
    - name: order_id
- dataobject: line_items
  type: table
  fields:
    - name: order_id
```

### Yet another option

```yaml
models:
- model: orders
  type: table
  fields:
    - field: order_id
- model: line_items
  type: table
  fields:
    - field: order_id
```
