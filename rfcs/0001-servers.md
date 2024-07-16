# Servers

Champion: Simon Harrer

## Summary

We introduce a way to specify the "physical" location of the data protected by the contract. The contract protect data on different environments (prod, preprod, ...) and technologies (bigquery, kafka,  s3, ...). 

## Motivation

Use Cases:
- As a data consumer, when looking at a data contract, I want to access the actual data. I want to know where the data is and have details on how to get info.
- The other way around, when having access to given datasets I need to be able to correlate them to a given data contract.

Status Quo:
- Currently, there are already a number of fields defined in ODCS which are related to they "physical" location that are BigQuery/JDBC dependent.
- RFC 5 removes them which are scheduled to be removed via RFC 5.

Assumption:
- A data contract can protect data on different technologies and environments
- Role-Based Security is the 80% case

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

- `server`: REQUIRED identifier
- `type`: REQUIRED supported server types: a union of SODA server types and Data Contract Specification server types along with their properties.
- type-dependent REQUIRED and OPTIONAL parameters
- `description`: OPTIONAL textual description, only an info block
- `environment`: OPTIONAL use for "prod", "preprod", ...
- `roles`: OPTIONAL list of offered roles, similar to the roles on top level of the data contract, but local to a specific server
- `customProperties`: OPTIONAL anything custom

## Consequences

- Standardization of connection details is more feasible
- Data producers can more easily communicate the connection details to data consumers (due to standardization)
- There are no "server-specific" quality checks, SLAs, etc. If those are different per server, one needs to create separate data contracts.
- Support for "monoserver" is indirectly available by simply adding only a single server to the list

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
