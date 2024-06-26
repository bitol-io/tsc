# Generic, hierarchical tables

Champion: Andrew Jones & Jean-Georges Perrin (jgp).

## Summary

The current structure uses `dataset`, `name` and `columns` to define a data contract. However, these are all database terms - or specially, BigQuery terms, and they suggest very clearly an implementation (tables in a data warehouse). We want the data contract to be more generic than that.

Moreover, there is a need for a hierarchical representation to represent complex data types.

## Motivation

Many of us are using data contracts for more than just the data warehouse. The second most popular technology used is streaming, for example Kafka or Google Pub/Sub. But as a concept, data contracts are not tied to any particular technology - it's more of a pattern to describe data, no matter where that data happens to be.

We want to make the standard reflect that. It should be generic, so as technology changes, the standard remains relevant.

This aligns with our guiding value of interoperability.

## Design and examples

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
- How do we capture other data, like unstructured data (scientific papers as PDFs), vectors, more... Do we want to specify unstructured or semistructured data in a data contract?

## Current proposition

```
schema:
  - object|name: MyObjectName
    type|logicalType: object # defaults to object, can be left away
    attributes|properties|fields|objects:
      - object|field|name|attribute: shipment_date
        type|logicalType: date
      - object|name: shipment_address
        type|logicalType: object # defaults to object, can be left away
        attributes|properties|fields|objects:
          - object|field|name|attribute: postal_code
            type|logicalType: string 
          - object|field|name|attribute: street_lines
            type|logicalType: array 
            items|objects: # BE AWARE, THIS IS DIFFERENT FOR ARRAY
              field|name: street_line
              type|logicalType: string
```

The structure is agreed on. Naming og properties is still open.

- Perhaps go alongside JSON schema?
- Lesson from the Avro schema: confusion around physical & logical type.



## Rejected alternatives

### Option A - Field and Name

Status: rejected.

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

### Option B - Genericity

Status: rejected.

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

### Option C - Genericity #2

Status: rejected.

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

### Option D - Genericity with hierarchy

```yaml
schema:
- object: MyTable
  kind: table
```

```yaml
schema:
- object: MyTopic
  kind: topic
```

```yaml
schema:
- object: Transactions
  kind: table
  physicalName: trx_v1
  description: Contains transactions.
  attrtibutes:
  - field: Identifier
    physicalName: id
```

```yaml
schema:
- object: Transactions
  kind: table
  physicalName: trx_v1
  description: Contains transactions.
  attrtibutes:
  - field: Identifier
    physicalName: id
```

```yaml
schema:
- object: Transactions
  kind: table
  physicalName: trx_v1
  description: Contains transactions.
  attrtibutes:
  - field: Identifier
  - object: TransactionDetail
    kind: table
    physicalName: trx_details_v1
    description: Contains transactions.
    attributes:
    - field: date
      physicalName: trx_ts
      description: Timestamp of the transaction.
```

### Option E = Option D + Support for Array

TBD

### Option F = More generic tags
```yaml
schema:
- object: Transactions
  kind: table
  physicalName: trx_v1
  description: Contains transactions.
  attrtibutes:
  - object: Identifier
    kind: field
    physicalName: id_whatever
    description: Identifier for the table in this instance.
  - object: TransactionDetail
    kind: table
    physicalName: trx_details_v1
    description: Contains transactions.
    attributes:
    - field: date
      physicalName: trx_ts
      description: Timestamp of the transaction.
```
Alternative: Leaf is always an attribute. If there is non-trivial substructure it is an object.
```yaml
schema:
- object: Transactions
  kind: table
  physicalName: trx_v1
  description: Contains transactions.
  objects:
  - attribute: Identifier
    kind: field
    physicalName: id_whatever
    description: Identifier for the table in this instance.
  - object: TransactionDetail
    kind: table
    physicalName: trx_details_v1
    description: Contains transactions.
    attributes:
    - attribute: date
      kind: field
      physicalName: trx_ts
      description: Timestamp of the transaction.
```

### Option G - JSON Schema-inspired

https://json-schema.org/learn/getting-started-step-by-step

```yaml
schema:
- Transactions: table # `Transactions` is the name of the table
  physicalName: trx_v1
  description: Contains transactions.
  properties:
  - Identifier: string
    physicalName: id_whatever
    description: Identifier for the table in this instance.
  - TransactionDetail: table
    physicalName: trx_details_v1
    description: Contains transactions.
    properties:
    - trx_ts: timestamp
      physicalName: trx_ts
      description: Timestamp of the transaction.
    - transactions: array
      items:
        - type: string
      description: List of transactions.
```

Discussion:
* The tech changes the schema

### Option H - JSON Schema-inspired with names

```yaml
schema:
- name: Transactions
  type: table
  physicalName: trx_v1
  description: Contains transactions.
  properties:
  - name: Identifier
    type: string
    physicalName: id_whatever
    description: Identifier for the table in this instance.
  - name: TransactionDetail
    type: table
    physicalName: trx_details_v1
    description: Contains transactions.
    properties:
    - name: trx_ts
      type: timestamp
      physicalName: trx_ts
      description: Timestamp of the transaction.
    - name: transactions
      type: array
      items:
        - type: string
      description: List of transactions.
```

### Option I

```
schema:
  - object|name: MyObjectName
    type|logicalType: object # defaults to object, can be left away
    attributes|properties|fields|objects:
      - object|field|name|attribute: shipment_date
        type|logicalType: date
      - object|name: shipment_address
        type|logicalType: object # defaults to object, can be left away
        attributes|properties|fields|objects:
          - object|field|name|attribute: postal_code
            type|logicalType: string 
          - object|field|name|attribute: street_lines
            type|logicalType: array 
            items|objects: # BE AWARE, THIS IS DIFFERENT FOR ARRAY
              field|name: street_line
              type|logicalType: string
```

Structure is agreed on. Naming is still open. Perhaps go alongside json schema?

## Decision

- 2024-06-18: Rejection of options A, B, C, D, E, F, G, and H. Approval of option I.

## Consequences

- Makes the structure more applicable for users for other data structures
- Nudges users more towards creating a logical model and less a physical model here
- This is a breaking change. Would be targeted for v3.
- We are very conscious in separating the logical representation of a dataset with the physical representation

## References

* At GoCardless [we use name](https://medium.com/gocardless-tech/implementing-data-contracts-at-gocardless-3b5c49074d13), and it's working well for both tables and streams. It has not caused any confusion in 3 years of production, despite most users being familiar with the underlying technology and its terms (BigQuery and Pub/Sub)
* [Protobuf calls them "fields"](https://protobuf.dev/programming-guides/proto3/), as [does Avro](https://avro.apache.org/docs/1.11.1/specification/).
* [OpenAPI uses "title"](https://spec.openapis.org/oas/latest.html), which could also work well instead of name
* [OpenLineage uses "Dataset"](https://openlineage.io/docs), so the current usage of dataset may be fine given that it is not exclusively used in GCP/BigQuery
* [XTable uses "datasets"](https://github.com/apache/incubator-xtable), so it is not uncommon to use dataset also for tabular data
* [AsyncAPI v3.0.0 uses "Channel" for conveying messages](https://www.asyncapi.com/docs/reference/specification/v3.0.0), so that may be appropriate to use for messaging/streaming data (e.g. Apache Kafka topic)
