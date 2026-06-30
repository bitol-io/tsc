# File Object Storage Layout via `logicalType: blob`

Champion: TBD (TSC member needed).

Authors: Damien Maresma

Slack: Damien Maresma

GitHub issue: https://github.com/bitol-io/open-data-contract-standard/issues/276

Applies to:
* [x] ODCS - Open Data Contract Standard
* [ ] ODPS - Open Data Product Standard
* [ ] OORS - Open Observability Results Standard
* [ ] OOCS - Open Orchestration and Control Standard
* [ ] OMMS - Open Maturity Model Standard
* [ ] OMDS - Open Metadata Difference Standard

## Summary

This RFC introduces `logicalType: blob` as a schema-level marker that activates file-properties mode on any ODCS schema object. When set, the `properties` array describes file-metadata attributes (naming patterns, size limits, content types, tiering, freshness) rather than data columns — replacing ad hoc storage audits with contract-driven, CI/CD-ready validation across Azure Blob Storage, ADLS Gen2, AWS S3, SFTP, and local filesystems.

## Motivation

Raw and processed assets (JSON exports, Parquet snapshots, CSV feeds, ML artifacts, ingestion logs) stored in object stores carry the same data-quality obligations as structured datasets, yet no ODCS mechanism covers them. `logicalType: blob` closes that gap by reusing existing ODCS constructs (`logicalType`, `quality`, `properties`) with a thin layer of file-specific semantics — no new top-level keys, no server-type constraints.

Alignment with ODCS guiding values:

- **Interoperability over readability** — the file-properties mapping targets tooling, not just humans.
- **Small standard** — reuses `logicalType` and `quality`; zero new structural keys.
- **Everything-as-code** — file-layout policies ship as version-controlled, reviewable artifacts.

## Design and examples

### Activation condition

File-schema mode is activated by a single schema-level condition, independent of server type:

| Condition | Location in ODCS | Value |
| --------- | ---------------- | ----- |
| Schema object represents a file set | `schema[*].logicalType` | `blob` |

When set, `properties` is interpreted as a **file-properties schema** instead of a data-column schema. All other ODCS features (quality rules, SLA, lineage) remain unchanged.

### File-properties mapping

Each property `name` maps to a standard file-metadata attribute. ⚡ = available on all backends; ☁ = object-store platforms only (Azure Blob, ADLS Gen2, AWS S3).

| ODCS property `name` | Underlying attribute | Logical type | Availability | Description |
| -------------------- | -------------------- | ------------ | ------------ | ----------- |
| `name` | File / blob name (relative to root) | `string` | ⚡ All | Full path of the file within its storage root, including any virtual directory or folder prefix. |
| `prefix` | Path prefix / virtual directory | `string` | ⚡ All | Directory path or object-store prefix (e.g. `year=2024/month=01/`). Used to declare the expected hierarchy. |
| `size` | `Content-Length` / file size | `integer` | ⚡ All | Size of the file in bytes. |
| `contentType` | `Content-Type` / MIME type | `string` | ⚡ All | MIME type (e.g. `application/json`, `application/octet-stream`). |
| `lastModified` | Last-modified timestamp | `date` | ⚡ All | UTC timestamp of the last write operation. Used for freshness constraints. |
| `createdOn` | Creation timestamp | `date` | ⚡ All | UTC timestamp of file creation. |
| `etag` | Entity tag | `string` | ☁ Object stores | Opaque string for optimistic concurrency and integrity checks. |
| `contentMD5` | MD5 / checksum | `string` | ⚡ All | Base64-encoded MD5 hash (or equivalent checksum) of the file content. |
| `contentEncoding` | `Content-Encoding` | `string` | ☁ Object stores | Transfer encoding (e.g. `gzip`). |
| `blobType` | Storage class / blob type | `string` | ☁ Azure only | Azure blob type: `BlockBlob`, `AppendBlob`, `PageBlob`. |
| `tier` | Access tier / storage class | `string` | ☁ Object stores | Azure: `Hot`, `Cool`, `Cold`, `Archive`. AWS S3: `STANDARD`, `INTELLIGENT_TIERING`, `GLACIER`, etc. |
| `leaseState` | Lease / lock state | `string` | ☁ Azure only | Azure lease state: `available`, `leased`, `expired`, `breaking`, `broken`. |
| `metadata` | User-defined key-value metadata | `object` | ☁ Object stores | Platform metadata tags attached to the file or blob. |

Implementations MAY support additional platform-specific attributes via ODCS `customProperties`.

### Directory hierarchy (`prefix` property)

The `prefix` property documents and enforces the virtual directory layout via its `pattern` option (regex). Common use cases: Hive-style partitioning (`year=YYYY/month=MM/`), environment segregation (`raw/`, `processed/`), team namespacing, or version tagging.

### Quality rules on file properties

Standard ODCS quality rules apply to every file-property. Common patterns:

| Goal | Quality rule |
| ---- | ------------ |
| Maximum file size | `mustBeLessThan: <bytes>` on `size` |
| Allowed MIME types | `mustBeIn: [...]` on `contentType` |
| Freshness / expiry | `mustBeGreaterThan: <ISO date>` on `lastModified` |
| Required blob type | `mustBe: BlockBlob` on `blobType` |
| Required access tier | `mustBeIn: [Hot, Cool]` on `tier` |
| Naming pattern | `pattern: <regex>` on `name` or `prefix` |
| Integrity check | `mustNotBeNull: true` on `contentMD5` |
| Empty directory (no stray files) | `mustBe: 0` on a count quality rule |

---

### Example 1 — Minimal: JSON export landing zone on AWS S3

Nightly JSON export on S3. `logicalType: blob` is the sole activation condition; the server `type` has no effect on schema mode.

```yaml
apiVersion: v3.1.0
kind: DataContract
name: nightly_json_export

servers:
  production:
    type: s3
    location: s3://my-bucket/exports/nightly/

schema:
  - name: nightly_export_files
    logicalType: blob
    description: Nightly JSON export files for downstream consumers.
    properties:
      - name: name
        logicalType: string
        quality:
          - type: text
            rule: pattern
            pattern: "^exports/nightly/[0-9]{4}-[0-9]{2}-[0-9]{2}/.*\\.json$"

      - name: contentType
        logicalType: string
        quality:
          - type: text
            rule: mustBe
            mustBe: "application/json"

      - name: size
        logicalType: integer
        description: Must not exceed 500 MB.
        quality:
          - type: library
            rule: mustBeLessThan
            mustBeLessThan: 524288000
```

---

### Example 2 — Structured: ML artifact store on Azure ADLS Gen2

Enforces prefix hierarchy, blob type, access tier, content type, max age, and naming conventions for ML training checkpoints and rendered images.

```yaml
apiVersion: v3.1.0
kind: DataContract
name: ml_artifact_store

servers:
  production:
    type: azure
    location: az://ml-storage-account/ml-artifacts/

schema:
  - name: model_checkpoints
    logicalType: blob
    description: Training checkpoints under raw/<project>/<run-id>/.
    properties:
      - name: prefix
        logicalType: string
        required: true
        quality:
          - type: text
            rule: pattern
            pattern: "^raw/[a-z0-9_-]+/[a-f0-9-]{36}/$"

      - name: name
        logicalType: string
        required: true
        quality:
          - type: text
            rule: pattern
            pattern: "^raw/[a-z0-9_-]+/[a-f0-9-]{36}/.*\\.(ckpt|pt)$"

      - name: size
        logicalType: integer
        description: Must not exceed 2 GB.
        quality:
          - type: library
            rule: mustBeLessThan
            mustBeLessThan: 2147483648

      - name: contentType
        logicalType: string
        quality:
          - type: text
            rule: mustBe
            mustBe: "application/octet-stream"

      - name: blobType
        logicalType: string
        quality:
          - type: text
            rule: mustBe
            mustBe: "BlockBlob"

      - name: tier
        logicalType: string
        quality:
          - type: text
            rule: mustBeIn
            mustBeIn: ["Hot", "Cool"]

      - name: lastModified
        logicalType: date
        description: Must have been written within the last 90 days.
        quality:
          - type: library
            rule: mustBeGreaterThan
            mustBeGreaterThan: "$now-90d"

      - name: contentMD5
        logicalType: string
        required: true
        quality:
          - type: library
            rule: mustNotBeNull

      - name: metadata
        logicalType: object
        description: Must include project and runId keys.
        properties:
          - name: project
            logicalType: string
            required: true
          - name: runId
            logicalType: string
            required: true

  - name: rendered_images
    logicalType: blob
    description: PNG images under images/<model-version>/<run-id>/.
    properties:
      - name: name
        logicalType: string
        required: true
        quality:
          - type: text
            rule: pattern
            pattern: "^images/v[0-9]+/[a-f0-9-]{36}/.*\\.png$"

      - name: contentType
        logicalType: string
        quality:
          - type: text
            rule: mustBe
            mustBe: "image/png"

      - name: size
        logicalType: integer
        description: Must not exceed 10 MB.
        quality:
          - type: library
            rule: mustBeLessThan
            mustBeLessThan: 10485760

      - name: lastModified
        logicalType: date
        description: Must have been produced within the last 7 days.
        quality:
          - type: library
            rule: mustBeGreaterThan
            mustBeGreaterThan: "$now-7d"

quality:
  - type: text
    description: No blob outside the raw/ or images/ prefixes is allowed.
    rule: pattern
    mustMatch: "^(raw|images)/"
```

---

### Example 3 — Data quality gate: quarantine log detection in an ingestion pipeline

In a data ingestion pipeline, files that fail validation are moved to a `quarantine/` folder and a `.log` file is written alongside them. A healthy pipeline produces **no quarantine files**. The contract below enforces that invariant: any `.log` file found under `quarantine/` is a contract violation that fails the CI/CD quality gate.

```
ingestion pipeline
  ├── landing/          ← raw inbound files
  ├── processed/        ← successfully validated files
  └── quarantine/       ← rejected files + error logs  ← monitored here
        ├── 2024-06-30_crm_feed_REJECTED.csv
        └── 2024-06-30_crm_feed_REJECTED.log   ← must NOT exist
```

```yaml
apiVersion: v3.1.0
kind: DataContract
name: crm_ingestion_pipeline

servers:
  production:
    type: azure
    location: az://ingestion-storage/crm/

schema:
  - name: quarantine_logs
    logicalType: blob
    description: >
      Quarantine error logs written by the CRM ingestion pipeline.
      Any file matching this schema signals an upstream validation failure.
      The quality rule below asserts the quarantine folder must stay empty.
    properties:
      - name: prefix
        logicalType: string
        description: All quarantine files live under the quarantine/ prefix.
        required: true
        quality:
          - type: text
            rule: pattern
            pattern: "^quarantine/$"

      - name: name
        logicalType: string
        description: Log file name written by the ingestion engine.
        quality:
          - type: text
            rule: pattern
            pattern: "^quarantine/[0-9]{4}-[0-9]{2}-[0-9]{2}_.*\\.log$"

      - name: size
        logicalType: integer
        description: Log file size in bytes.

      - name: lastModified
        logicalType: date
        description: Last write timestamp — used for incident age tracking.

    quality:
      - type: library
        description: >
          CRITICAL — The quarantine folder must contain zero log files.
          A non-zero count means records were rejected during ingestion
          and need immediate review before downstream consumers proceed.
        rule: mustBe
        mustBe: 0
        dimension: completeness
        severity: error
```

**How it works at runtime:** The contract engine counts all objects under `quarantine/` that match the `*.log` pattern. Any count above `0` fails the rule with severity `error`, blocking promotion to the `processed/` layer and giving operators a clear, contract-driven signal — no custom monitoring scripts required.

---

### Interaction with existing server fields

| Field | Interaction |
| ----- | ----------- |
| `type` | Any server type is supported (`azure`, `s3`, `sftp`, `local`). No restriction. |
| `format` | Retains its payload-format meaning; does **not** activate blob-schema mode. |
| `encoding` (RFC-0043) | Applicable for text payloads; may be omitted for opaque binary files. |
| `location` | Defines the storage root. `prefix` refines the sub-directory layout inside that root. |
| `delimiter` | Not meaningful in file-property schemas. |

### JSON Schema impact

For ODCS JSON Schema, a single change is required:

1. **Schema object**: add `blob` to the allowed enum values of the `logicalType` field.

No change is required to any server definition. The blob-schema mode is activated entirely by the schema-side `logicalType`.

```json
"logicalType": {
  "type": "string",
  "description": "Logical type of the schema object. Use 'blob' to declare a file-properties schema applicable to any object store or file server.",
  "examples": ["table", "view", "object", "blob"]
}
```

## Alternatives

- **Dedicated `blobPolicy:` top-level section** — rejected: overlaps with `schema`/`quality`; increases the learning surface without added expressiveness.
- **Constraints on the `servers` section only** — rejected: the server section describes an access point, not a dataset shape. `schema` is the right home for data-shape declarations.
- **`logicalType: file` instead of `blob`** — rejected: `file` already refers to a scalar upload property in OpenAPI 2.0 and original ODCS (RFC-0002). Using `blob` avoids the collision while remaining readable across all backends.
- **New `fileProperties:` sub-object on schema** — rejected: the existing `properties` model covers all necessary semantics without a structural extension.
- **Multiple activation conditions (server type + format)** — rejected: tying the feature to a single server type limits adoption; overloading `format` with a schema-mode switch conflates unrelated concerns. A single schema-side marker is sufficient.

## Decision

TBD by TSC.

## Consequences

### Positive

- Brings file-layer governance into the data contract, removing the need for out-of-band blob audit scripts.
- Reuses existing ODCS constructs: no new top-level keys, no new quality rule syntax.
- Fully backward-compatible: contracts without `logicalType: blob` are unaffected.
- The file-properties mapping table gives tooling authors a stable, documented interface.
- Enables CI/CD validation of file layouts alongside schema and data-quality checks.
- Works across all server types (Azure, S3, SFTP, local) with a single, uniform declaration.

### Trade-offs

- Tooling must detect `logicalType: blob` at the schema level and switch interpretation mode accordingly.
- The `$now-Nd` / `$now-Nh` shorthand for freshness rules is informal; a future RFC should standardize relative time expressions.
- Property names in the mapping table are opinionated: teams with different conventions should use `customProperties` for non-standard attributes.
- Some attributes (e.g., `blobType`, `leaseState`, `tier`) are platform-specific; contracts targeting SFTP or local servers SHOULD omit them.

## References

- Azure Blob Storage BlobProperties REST reference: <https://learn.microsoft.com/en-us/rest/api/storageservices/get-blob-properties>
- Azure SDK for Python — `BlobProperties`: <https://learn.microsoft.com/en-us/python/api/azure-storage-blob/azure.storage.blob.blobproperties>
- ADLS Gen2 hierarchical namespace: <https://learn.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-namespace>
- AWS S3 Object metadata reference: <https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingMetadata.html>
- AWS S3 Storage classes: <https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-class-intro.html>
- RFC-0002 Data Types (original `file` logical type in OpenAPI context): `rfcs/approved/odcs-v3.0.0/0002-types.md`
- RFC-0043 Physical Data Encoding (server-level `encoding` field): `rfcs/0043-physical-data-encoding.md`
- RFC-0042 Vector Type (precedent for extending `logicalType` with platform-specific semantics): `rfcs/0042-vector-type.md`
- RFC-0027 Unstructured Data Quality (prior art for non-tabular schema governance): `rfcs/0027-unstructured-data-quality.md`
- OpenAPI 2.0 `file` type: <https://swagger.io/docs/specification/2-0/file-upload/>
- ODCS infrastructure servers documentation: <https://bitol-io.github.io/open-data-contract-standard/latest/infrastructure-servers/>
