# TBD

## Context

As a data consumer, when looking at a data contract, I want to access the actual data. I want to know where the data is.

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
  - environment: production
    type: BigQuery
    project: acme_orders_prod
    dataset: bigquery_orders_latest_npii_v1
    usage: "Low latency queries here"
  - environment: dev
    type: BigQuery
    project: acme_orders_dev
    dataset: bigquery_orders_latest_npii_v1
  - environment: production
    type: S3
    location: s3://some-url/
    usage: "Use this for batch processing"
  - environment: production
    type: JDBC
    connection: "jdbc:postgresql://localhost:5432/database/orders"
```

We want to put only so much information in here, so the interested data consumer can find the data.

*Note: secrets should not be part of the contract! (Passwords are secrets!)*

*Open Questions*

- What about (additional) links?
- What about having environment-specific pricing, quality checks, etc.? Do we want to support this?

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

