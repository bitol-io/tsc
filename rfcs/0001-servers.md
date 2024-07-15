# Servers

Champion: Simon Harrer

## Context

As a data consumer, when looking at a data contract, I want to access the actual data. I want to know where the data is.

The other way around, when having access to given datasets I need to be able to correlate them to a given data contract.

Currently there are already a number of fields defined in ODCS which are related to they "physical" location.  From the ODCS  example:

```yaml
# Physical parts / GCP / BigQuery specific
sourcePlatform: googleCloudPlatform
sourceSystem: bigQuery
datasetProject: edw # BQ dataset
datasetName: access_views # BQ dataset

# Physical access
driver: null
driverVersion: null
server: null
database: pypl-edw.pp_access_views
username: '${env.username}'
password: '${env.password}'
schedulerAppName: name_coming_from_scheduler
```

The above fields may be replaced or moved.

*Note: secrets should not be part of the contract!*

## Decision

TBD

## Options

### Option 1: Explicit Servers

```yaml
dataset:
  tables:
  - table: orders
  - table: line_items
servers:
  - server: prod_bigquery
    environment: prod
    type: BigQuery
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
    type: BigQuery
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

We want to put only so much information in here, so the interested data consumer can find the data.

*Note: secrets should not be part of the contract! (Passwords are secrets!)*

*Open Questions*

- What about (additional) links? We currently don't plan to add them.
- What about having environment-specific pricing, quality checks, etc.? Do we want to support this? If really are a lot of environment specifics, should it be a second contract?
- Where does one get access?
- What about the roles section? The roles section uses a monoserver approach whereas this option would support multiserver approach.
- How to add custom properties?

#### Consequences
- Standardization of connection details is more feasible
- Data producers can more easily communicate the connection details to data consumers (due to standardization)

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
