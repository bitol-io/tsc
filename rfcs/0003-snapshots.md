# TBD

## Context

As a data consumer, when looking at a data contract, I want to access temporal immutable snapshot of underlying dataset.

Snapshots are immutable and temporal state capture of dataset. Being immutable, they allow consumers to pin they analysis for better reproducibility. For mutable data sets, snapshots may be physical copy of data captured using specific strategy. For append only data sets, snapshots may contain dynamic strategy to calculate snapshot from underlying data on demand.   
### Example

* Daily snapshots
* Snapshot by certain unique value

## Decision

TBD

## Options

### Option 1: Included

```YAML
snapshots:
  - name: daily_snapshot
    dataset: tab1
    unique_key: rcvr_id
    strategy: daily_update
    updated_at: 11:59 PM
```

We want to put only so much information in here, so the interested data consumer can find the snapshot.

#### Consequences
- Ability to define snapshots including composed keys
- Snapshot spec flexible enough to calculate acroos multiple datasets

### Option 2: Referenced

```yaml
- column: txn_ref_dt
      businessName: Transaction reference date
      logicalType: date
      physicalType: date
      description: Reference date for the transaction. Use this date in reports and aggregation rather than txn_mystical_dt, as it is slightly too mystical.
      sampleValues:
        - 2022-10-03
        - 2025-01-28
      snapshots:
        - name: daily_snapshot
          strategy: daily_update
          updated_at: 11:59 PM
```

#### Consequences
- Less verbose option
- Reduced readability considering already over-loaded column configuraitons
- No option for including composed keys in snapshot calculations



