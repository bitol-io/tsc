# Servers

Champion: Simon Harrer

## Summary

We introduce a way to specify the "physical" location of the data protected by the contract. A data contract can protect data on different environments (prod, preprod, ...) and technologies (bigquery, kafka,  s3, ...).

## Motivation

**Use Cases**
- As a data consumer, when looking at a data contract, I want to access the actual data. I want to know where the data is accessible to me and have details on how to get access.
- As a platform engineer, when automating access, I need to know the offered roles per technology and environment so I can set up the appropriate ACL for the data marketplace.

**Status Quo**
- We have a number of top-level attributes on "physical" location that are BigQuery/JDBC dependent. RFC5 removes it.
- We have support for roles at the top-level.
- We have physicalType at the column/table level.

**Assumptions**
- A data contract can protect data on different technologies and environments: one contract for data that is available on BigQuery and Kafka, one contract for data that is available on the PROD and the NONPROP Snowflake account, ...

## Decision

TBD

## Explicit Servers

```yaml
dataset:
  tables:
  - table: orders
  - table: line_items
servers:
  - server: prod_bigquery
    environment: prod
    type: bigquery
    project: acme_orders_prod
    dataset: bigquery_orders_latest_npii_v1
    description: "Low latency queries here"
    roles:
      - role: acme-order-prod-reader
        description: just the reader role
      - role: acme-order-prod-pii-reader
        description: the PII reader role # optional
  - server: dev
    environment: dev
    type: bigquery
    project: acme_orders_dev
    dataset: bigquery_orders_latest_npii_v1
    customProperties:
      - username: my-username # migration from 'username' field
      - password: secret # migration from 'password' field
  - server: prod_s3
    environment: prod
    type: S3
    location: s3://some-url/
    description: "Use this for batch processing"
  - server: prod_jdbc
    environment: prod
    type: JDBC
    connection: "jdbc:postgresql://localhost:5432/database/orders"
  - server: legacy-dev
    environment: dev
    type: sftp
    location: my-sftp-location-dev
    format: csv
  - server: kafka-dev
    environment: dev
    type: kafka
    topic: orders
    format: json
```

- `server`: STRING REQUIRED identifier of the server
- `type`: STRING REQUIRED supported server types: a union of SODA server types and Data Contract Specification server types along with their properties. (examples: kafka, bigquery, s3, ...).
- type-dependent REQUIRED and OPTIONAL parameters (examples (kafka): topic, host; examples (bigquery): project, dataset) 
- `description`: STRING OPTIONAL textual description, only an info block
- `environment`: STRING OPTIONAL use for "prod", "preprod", ...
- `roles`: ARRAY OPTIONAL list of offered roles, similar to the roles on top level of the data contract, but local to a specific server
- `customProperties`: MAP OPTIONAL anything custom

**Open Questions**
- What about the properties of SODA and DCS: which naming convention should we apply here?
- Possible conflict with `physicalType` at the table/column level

## Consequences

- Standardization of connection details is more feasible
- Data producers can more easily communicate the connection details to data consumers (due to standardization)
- There are no "server-specific" quality checks, SLAs, etc. *Workaround*: create separate data contracts or add additional limitations via the `description` for `customProperties` to each server.
- Support for "monoserver" is not available on the top level. *Workaround*: have a servers list with one server

## Discarded Options

### Option 2: Generic server types (DISCARDED)

```yaml
dataset:
  tables:
  - table: orders
  - table: line_items
servers:
  - environment: prod_bigquery
    type: dbms
    host: https://www.googleapis.com/bigquery/v2
    port: 443
    dialect: bigquery
    database: acme_orders_prod
    description: Low latency queries here
    custom:
      project: acme_orders_prod
      dataset: bigquery_orders_latest_npii_v1
      roles:
        - role: acme-order-prod-reader
        - role: acme-order-prod-pii-reader
      username: my-username # migration from 'username' field
      password: secret # migration from 'password' field
  - environment: prod_s3
    type: objectstorage
    location: s3://some-url/
    format: json
    delimiter: newline
    description: Use this for batch processing
  - environment: prod_jdbc
    type: dbms
    host: localhost
    port: 5432
    database: database
    schema: orders
    dialect: postgresql
  - environment: legacy-dev
    type: objectstorage
    location: sftp://my-sftp-location-dev
    format: csv
    delimiter: newline
  - environment: kafka-dev
    type: messaging
    channel: orders
    format: json
    host: my-kafka-cluster 
    custom:
      partitions: 10
      partitionKey: orderId
```

### Option 3: Generic structure (DISCARDED)

Same as option 1, but all fields are basically custom.

Not helpful for automation based on that.

### Option 4: Do nothing (DISCARDED)

Do not include the servers.
