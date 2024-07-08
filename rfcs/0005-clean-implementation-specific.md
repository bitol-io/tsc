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
- Rename `dataset.table.columns.column.sampleValues` to `examples`
- Drop `dataset.table.columns.column.criticalDataElementStatus` or rename to `isCriticalDataElement`
- Rename `dataset.table.columns.column.isNullable` to `required` (and negate!)

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

```


## Consequences

- Breaking change
- Migration path: move to custom fields (could be supported by an automated migration tool)

