# Clean infrastructure/implementation-specific properties

Champion: Jochen Christ.

Keep ODCS small and free from vendor-specific or platform-specific parameters. 

## Status

Proposed

## Decision

Top-Level

- Rename `uuid` to `id`
- Add `name` (did I miss anything?)
- Rename `quantumName` to `dataProduct` and make it optional
- Rename `datasetDomain` to `domain` (we avoid the dataset prefix)
- Drop `datasetKind` (example: `virtualDataset`, was optional, have not seen any usage)
- Drop `userConsumptionMode` (examples: `analytical`, was optional, already deprecated in v2.)
- Drop `sourceSystem` (example: `bigQuery`, information will be encoded in servers)
- Drop `sourcePlatform` (example: `googleCloudPlatform`, information will be encoded in servers)
- Drop `productSlackChannel` (will move to support channels)
- Drop `productFeedbackUrl` (will move to support channels)
- Drop `productDl` (will move to support channels)
- Drop `username` (credentials should not be stored in the data contract)
- Drop `password` (credentials should not be stored in the data contract)
- Drop `driverVersion` (will move to servers if needed)
- Drop `driver` (will move to servers if needed)
- Drop `project` (BigQuery-specific, will move to servers)
- Drop `datasetName` (BigQuery-specific, will move to servers)
- Drop `database` (BigQuery-specific, will move to servers)
- Drop `schedulerAppName` (not part of the contract)

Dataset
- Drop `dataset.table.priorTableName`
- Drop `dataset.table.dataGranularity`
- Rename `dataset.table.columns.column.partitionStatus` to `isPartitioned` (TBD use `is` prefix or not)
- Drop `dataset.table.columns.column.clusterStatus`
- Drop `dataset.table.columns.column.clusterKeyPosition`
- Drop `dataset.table.columns.column.encryptedColumnName` (not sure, shouldn't this be another column)
- Drop `dataset.table.columns.column.transformSourceTables` (would be part of RFC-11)
- Drop `dataset.table.columns.column.transformLogic` (would be part of RFC-11)
- Drop `dataset.table.columns.column.transformDescription` (would be part of RFC-11)
- Rename `dataset.table.columns.column.sampleValues` to `examples`
- Drop `dataset.table.columns.column.criticalDataElementStatus` or rename to `isCriticalDataElement`

## TBD
- For boolean values use `is` prefix or not (`isUnique` vs `partitionStatus`).
- Follow Kubernetes conventions for apiVersion? (e.g., `odcs.bitol.io/v3`) 

## Example

```yaml
apiVersion: odcs.bitol.io/v3 # TBD
kind: DataContract
id: 53581432-6c55-4ba2-a65f-72344a91553a # Renamed from uuid
name: Transactions 
version: 1.1.0
status: current
tenant: ClimateQuantumInc
domain: seller # Renamed from datasetDomain
dataProduct: my data product # Renamed from quantumName and made optional

description:
  purpose: Views built on top of the seller tables.
  limitations: null
  usage: null

dataset:
  - table: tbl
    physicalName: tbl_1 
    # Removed priorTableName
    description: Provides core payment metrics 
    authoritativeDefinitions: # NEW in v2.2.0, inspired by the column-level authoritative links
      - url: https://catalog.data.gov/dataset/air-quality 
        type: businessDefinition
      - url: https://youtu.be/jbY1BKFj9ec
        type: videoTutorial
    tags: null
    # Removed dataGranularity
    columns:
      - column: txn_ref_dt
        businessName: transaction reference date
        description: null
        logicalType: date
        physicalType: date
        isPrimaryKey: false # NEW in v2.1.0, Optional, default value is false, indicates whether the column is primary key in the table.
        primaryKeyPosition: -1
        isNullable: false
        isPartitioned: true # Renamed from partitionStatus
        partitionKeyPosition: 1
        # Removed clusterStatus
        # Removed clusterKeyPosition
        # Removed encryptedColumnName
        # Removed transformSourceTables
        # Removed transformLogic
        # Removed transformDescription
        examples: # Renamed from sampleValues
          - 2022-10-03
          - 2020-01-28
        # Removed criticalDataElementStatus
        tags: []
        classification: public
      - column: rcvr_id
        businessName: receiver id
        description: A description for column rcvr_id.
        logicalType: string
        physicalType: varchar(18)
        isPrimaryKey: true # NEW in v2.1.0, Optional, default value is false, indicates whether the column is primary key in the table.
        primaryKeyPosition: 1
        isNullable: false
        isPartitioned: false # Renamed from partitionStatus
        # Removed partitionKeyPosition
        # Removed clusterStatus
        # Removed clusterKeyPosition
        # Removed encryptedColumnName
        # Removed criticalDataElementStatus
        tags: []
        classification: restricted
      - column: rcvr_cntry_code
        businessName: receiver country code
        description: null
        logicalType: string
        physicalType: varchar(2)
        isPrimaryKey: false # NEW in v2.1.0, Optional, default value is false, indicates whether the column is primary key in the table.
        primaryKeyPosition: -1
        isNullable: false
        isPartitioned: false # Renamed from partitionStatus
        # Removed partitionKeyPosition
        # Removed clusterStatus
        # Removed clusterKeyPosition
        # Removed criticalDataElementStatus
        tags: []
        classification: public
        authoritativeDefinitions:
          - url: https://collibra.com/asset/742b358f-71a5-4ab1-bda4-dcdba9418c25
            type: businessDefinition
          - url: https://github.com/myorg/myrepo
            type: transformationImplementation
          - url: jdbc:postgresql://localhost:5432/adventureworks/tbl_1/rcvr_cntry_code
            type: implementation
        # Removed encryptedColumnName

```


## Consequences

- Breaking change
- Migration path: move to custom fields (could be supported by an automated migration tool)

