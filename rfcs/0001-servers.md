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

### Option 1: Included

```yaml
dataset:
  tables:
  - table: orders
  - table: line_items
servers:
  - environment: prod_bigquery
    type: BigQuery
    project: acme_orders_prod
    dataset: bigquery_orders_latest_npii_v1
    description: "Low latency queries here"
    custom:
      roles:
        - role: acme-order-prod-reader
        - role: acme-order-prod-pii-reader
  - environment: dev
    type: BigQuery
    project: acme_orders_dev
    dataset: bigquery_orders_latest_npii_v1
    custom:
      - username: my-username # migration from 'username' field
      - password: secret # migration from 'password' field
  - environment: prod_s3
    type: S3
    location: s3://some-url/
    description: "Use this for batch processing"
  - environment: prod_jdbc
    type: JDBC
    connection: "jdbc:postgresql://localhost:5432/database/orders"
  - environment: legacy-dev
    type: sftp
    location: my-sftp-location-dev
    filetype: csv
```


**Should we enumerate all possible types in the standard?** We want to have an "extension" for this, not hardcoded in the standard. We support the three main server types sql, file, message.

**How should we handle "standard" technical properties?** TBD

**How should we handle custom properties?** TBD


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



### Option 2: Referenced

```yaml

```

#### Consequences
- Link is necessary to another place.


### Option 3: Out of scope

#### Consequences
- Put it somewhere else.

