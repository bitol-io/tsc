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

Closed Questions:
- How do we capture other data, like unstructured data (scientific papers as PDFs), vectors, more... Do we want to specify unstructured or semistructured data in a data contract? Via metadata for now.

## Updated syntax

### Basic example

```yaml
schema:
  - name: MyObjectName
    logicalType: object 
    physicalType: table|view|topic
    properties:
      - name: id
        logicalType: id
        physicalType: SERIAL
      - name: shipment_date
        logicalType: date
      - name: name
        logicalType: string
```

### An invoice stored hierarchically

```yaml
schema:
  - name: Invoice
    logicalType: object 
    physicalType: table
    properties:
      - name: id
        logicalType: id
        physicalType: SERIAL
      - name: shipment_date
        logicalType: date
      - name: comment
        logicalType: string
      - name: LineItem
          logicalType: object 
          physicalType: table
          properties:
          - name: position
            logicalType: numeric
            physicalType: integer
      - name: Address
          logicalType: object 
          physicalType: table
          properties:
          - name: city
            logicalType: string
            physicalType: varchar(45)
```

### An invoice stored linearly

```yaml
schema:
  - name: orders
    description: One record per order. Includes cancelled and deleted orders.
    logicalType: table
    properties:
      - name: order_id
        logicalType: string
      - name: order_timestamp
        description: The business timestamp in UTC when the order was successfully registered in the source system and the payment was successful.
        logicalType: timestamp
      - name: order_total
        description: Total amount the smallest monetary unit (e.g., cents).
        logicalType: long
      - name: customer_id
        description: Unique identifier for the customer.
        logicalType: string
      - name: customer_email_address
        description: The email address, as entered by the customer. The email address was not verified.
        logicalType: string
      - name: processed_timestamp
        description: The timestamp when the record was processed by the data platform.
        logicalType: timestamp
        physicalType: string
  - name: line_items:
    description: A single article that is part of an order.
    logicalType: table
    properties:
      - name: lines_item_id
        logicalType: text
        description: Primary key of the lines_item_id table
      - name: order_id
      - name: sku
        description: The purchased article number
```

### An array

```yaml
schema:
  - name: MyObjectName
    logicalType: object 
    properties:
      - name: shipment_date
        logicalType: date
      - name: shipment_address
        logicalType: object 
        properties:
          - name: postal_code
            logicalType: string 
            physicalType: VARCHAR(15)
          - name: street_lines
            logicalType: array 
            items:
              logicalType: string

          - name: x
            logicalType: array 
            items: 
              logicalType: object
              properties:
                - name: id
                  logicalType: uuid 
                  physicalType: VARCHAR(40)
                - name: b
                  logicalType: string 
                  physicalType: VARCHAR(15)
```

The structure and naming of keys are agreed on.

Lesson from the Avro schema: confusion around physical & logical type: let's explicitly call them `logicalType` and `physicalType`.

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

Status: rejected.

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

Status: rejected.

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

Status: rejected.

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

Status: rejected.

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

Status: rejected.

```yaml
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

### Option J

Status: rejected.

```yaml
schema:
  - object: MyObjectName
    logicalType: object # defaults to object, can be left away
    physicalType: table|view|topic
    properties:
      - attribute: id
        logicalType: id
        physicalType: SERIAL
      - attribute: shipment_date
        logicalType: date
      - attribute: name
        logicalType: string
```

```yaml
schema:
  - object: MyObjectName
    logicalType: object # defaults to object, can be left away
    properties:
      - attribute: shipment_date
        logicalType: date
      - object: shipment_address
        logicalType: object # defaults to object, can be left away
        properties:
          - attribute: postal_code
            logicalType: string 
            physicalType: VARCHAR(15)
          - attribute: street_lines
            logicalType: array 
            items: # BE AWARE, THIS IS DIFFERENT FOR ARRAY
              logicalType: string

          - attribute: x
            logicalType: array 
            items: # BE AWARE, THIS IS DIFFERENT FOR ARRAY
              logicalType: object
              properties:
                - attribute: id
                  logicalType: uuid 
                  physicalType: VARCHAR(40)
                - attribute: b
                  logicalType: string 
                  physicalType: VARCHAR(15)
```

## Decision

- 2024-06-18: Rejection of options A, B, C, D, E, F, G, and H. Approval of option I.
- 2024-07-16: Approval of final options w/ terms above.
- 2024-07-25: Rejected of option J, validation of current syntax.

## Consequences

- Makes the structure more applicable for users for other data structures
- Nudges users more towards creating a logical model and less a physical model here
- This is a breaking change. Targeted for v3.
- We are very conscious in separating the logical representation of a dataset with the physical representation

## References

* At GoCardless [we use name](https://medium.com/gocardless-tech/implementing-data-contracts-at-gocardless-3b5c49074d13), and it's working well for both tables and streams. It has not caused any confusion in 3 years of production, despite most users being familiar with the underlying technology and its terms (BigQuery and Pub/Sub)
* [Protobuf calls them "fields"](https://protobuf.dev/programming-guides/proto3/), as [does Avro](https://avro.apache.org/docs/1.11.1/specification/).
* [OpenAPI uses "title"](https://spec.openapis.org/oas/latest.html), which could also work well instead of name
* [OpenLineage uses "Dataset"](https://openlineage.io/docs), so the current usage of dataset may be fine given that it is not exclusively used in GCP/BigQuery
* [XTable uses "datasets"](https://github.com/apache/incubator-xtable), so it is not uncommon to use dataset also for tabular data
* [AsyncAPI v3.0.0 uses "Channel" for conveying messages](https://www.asyncapi.com/docs/reference/specification/v3.0.0), so that may be appropriate to use for messaging/streaming data (e.g. Apache Kafka topic)
