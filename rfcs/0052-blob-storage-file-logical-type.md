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

This RFC introduces a first-class contract mechanism for declaring, documenting, and enforcing file-storage policies on any object store or file server — including Azure Blob Storage, ADLS Gen2, AWS S3, SFTP servers, and local filesystems. When a schema object declares `logicalType: blob`, its `properties` array is interpreted as file-metadata attributes rather than data columns. This lets data teams specify naming patterns, size limits, content-type constraints, tiering requirements, freshness thresholds, and prefix-based directory hierarchies directly inside the data contract — regardless of the underlying storage technology — turning ad-hoc file audits into automated, contract-driven validation.

## Motivation

As organisations increasingly store raw and processed assets — JSON exports, Parquet snapshots, CSV feeds, ML artefacts — in object stores such as Azure Blob Storage, ADLS Gen2, AWS S3, or on SFTP and local filesystems, a governance gap emerges: these files are neither tables nor streams, yet they carry the same data-quality obligations as any structured dataset. Introducing `logicalType: blob` as a first-class citizen of the Open Data Contract Standard fills that gap by letting data teams declare, document, and enforce a file-storage policy directly inside the data contract.

Each schema property maps to a standard file-metadata attribute — `size`, `lastModified`, `contentType`, `etag`, and more — giving platform engineers a single, version-controlled source of truth to assert that no file in a prefix is oversized, expired, incorrectly typed, or missing altogether. The result is a shift from ad-hoc storage audits to automated, contract-driven validation that runs in CI/CD pipelines alongside structural schema checks, bringing the same observability and accountability to the file layer that data contracts already provide for tables and event streams.

This aligns with the guiding values of the standard:

- **We favour interoperability over readability** — the file-properties mapping table is designed to be consumed by tooling, not only humans.
- **We favour a small standard over a large one** — the RFC reuses existing ODCS constructs (`logicalType`, `quality`) and adds only a thin layer of file-specific semantics without any dependency on server type.
- **Everything-as-code** — file-layout policies become version-controlled, reviewable artefacts in the same repository as the data contract.

## Design and examples

### Activation condition

The file-schema mode is activated by a single condition:

| Condition | Location in ODCS | Value |
| --------- | ---------------- | ----- |
| Schema object represents a file set | `schema[*].logicalType` | `blob` |

When this condition is satisfied, the `properties` array of the schema object is interpreted as a **file-properties schema** rather than a data-column schema. The condition is **independent of the server type**: it applies equally to `azure`, `s3`, `sftp`, `local`, and any other server type defined in the contract. Every other ODCS feature (quality rules, SLA, lineage, etc.) continues to work as normal.

### File-properties mapping

Each property `name` in a `logicalType: blob` schema maps to a standard file-metadata attribute. The table below is the normative mapping. Attributes marked with ⚡ are universally available across all supported storage backends; attributes marked with ☁ are specific to object-store platforms (Azure Blob, ADLS Gen2, AWS S3) and MAY be omitted for SFTP or local servers.

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

Implementations MAY support additional platform-specific attributes by using ODCS `customProperties` on the property definition.

### Directory hierarchy (`prefix` property)

The `prefix` property is the primary tool for specifying the expected virtual directory layout. Its `pattern` option (a regular expression) documents and enforces the naming convention of the directory tree. This enables:

- **Hive-style partitioning** (`year=YYYY/month=MM/day=DD/`)
- **Environment segregation** (`raw/`, `processed/`, `curated/`)
- **Team or domain namespacing** (`finance/fx-rates/`)
- **Version tagging** (`v1/`, `v2/`)

### Quality rules on file properties

Standard ODCS quality rules apply to every file-properties property. The most common patterns are:

| Goal | Quality rule |
| ---- | ------------ |
| Maximum file size | `mustBeLessThan: <bytes>` on `size` |
| Allowed MIME types | `mustBeIn: [...]` on `contentType` |
| Freshness / expiry | `mustBeGreaterThan: <ISO date>` on `lastModified` |
| Required blob type | `mustBe: BlockBlob` on `blobType` |
| Required access tier | `mustBeIn: [Hot, Cool]` on `tier` |
| Naming pattern | `pattern: <regex>` on `name` or `prefix` |
| Integrity check | `mustNotBeNull: true` on `contentMD5` |

---

### Example 1 — Minimal: JSON export landing zone on AWS S3

A contract for a nightly JSON export dropped in an S3 bucket prefix. Only the essential properties are declared. Note that `logicalType: blob` is the sole activation condition; the server type (`s3`) imposes no constraint on the schema mode.

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
        description: Object key within the bucket.
        quality:
          - type: text
            rule: pattern
            pattern: "^exports/nightly/[0-9]{4}-[0-9]{2}-[0-9]{2}/.*\\.json$"

      - name: contentType
        logicalType: string
        description: MIME type must be application/json.
        quality:
          - type: text
            rule: mustBe
            mustBe: "application/json"

      - name: size
        logicalType: integer
        description: File size in bytes. Must not exceed 500 MB.
        quality:
          - type: library
            rule: mustBeLessThan
            mustBeLessThan: 524288000
```

---

### Example 2 — Structured: ML artefact store on Azure ADLS Gen2 with tiering and freshness

A detailed contract for a machine-learning artefact store that enforces prefix hierarchy, blob type, access tier, content type, maximum age, and file-naming conventions. The contract covers raw training checkpoints and PNG images rendered by the pipeline. The server uses `type: azure`; `logicalType: blob` on the schema objects activates file-property mode independently of any server-level field.

```yaml
apiVersion: v3.1.0
kind: DataContract
name: ml_artefact_store

info:
  title: ML Artefact Store Contract
  version: 1.2.0
  description: >
    Governs all ML model artefacts persisted on ADLS Gen2.
    Covers raw training checkpoints and PNG images rendered by the pipeline.

servers:
  production:
    type: azure
    location: az://ml-storage-account/ml-artefacts/
    encoding: UTF-8

  staging:
    type: azure
    location: az://ml-storage-account-staging/ml-artefacts/
    encoding: UTF-8

schema:
  - name: model_checkpoints
    logicalType: blob
    description: >
      Intermediate model checkpoints written during training.
      Stored under the raw/ prefix, partitioned by project and run ID.
    properties:
      - name: prefix
        logicalType: string
        description: Virtual directory. Must follow raw/<project>/<run-id>/ convention.
        required: true
        quality:
          - type: text
            rule: pattern
            pattern: "^raw/[a-z0-9_-]+/[a-f0-9-]{36}/$"

      - name: name
        logicalType: string
        description: Blob name, including prefix. Must end with .ckpt or .pt.
        required: true
        quality:
          - type: text
            rule: pattern
            pattern: "^raw/[a-z0-9_-]+/[a-f0-9-]{36}/.*\\.(ckpt|pt)$"

      - name: size
        logicalType: integer
        description: Checkpoint size. Individual files must not exceed 2 GB.
        quality:
          - type: library
            rule: mustBeLessThan
            mustBeLessThan: 2147483648

      - name: contentType
        logicalType: string
        description: Blob MIME type must be application/octet-stream.
        quality:
          - type: text
            rule: mustBe
            mustBe: "application/octet-stream"

      - name: blobType
        logicalType: string
        description: Must be stored as a BlockBlob.
        quality:
          - type: text
            rule: mustBe
            mustBe: "BlockBlob"

      - name: tier
        logicalType: string
        description: Checkpoints may sit in Cool tier to reduce storage costs.
        quality:
          - type: text
            rule: mustBeIn
            mustBeIn: ["Hot", "Cool"]

      - name: lastModified
        logicalType: date
        description: Checkpoint must have been written within the last 90 days.
        quality:
          - type: library
            rule: mustBeGreaterThan
            mustBeGreaterThan: "$now-90d"

      - name: contentMD5
        logicalType: string
        description: MD5 hash must be present to guarantee upload integrity.
        required: true
        quality:
          - type: library
            rule: mustNotBeNull

      - name: metadata
        logicalType: object
        description: User metadata must include project and runId keys.
        properties:
          - name: project
            logicalType: string
            required: true
          - name: runId
            logicalType: string
            required: true

  - name: rendered_images
    logicalType: blob
    description: >
      PNG images rendered by the ML pipeline and stored under the images/ prefix,
      partitioned by model version and run ID.
    properties:
      - name: prefix
        logicalType: string
        description: Virtual directory. Must follow images/<model-version>/<run-id>/ convention.
        required: true
        quality:
          - type: text
            rule: pattern
            pattern: "^images/v[0-9]+/[a-f0-9-]{36}/$"

      - name: name
        logicalType: string
        description: Blob name must end with .png.
        required: true
        quality:
          - type: text
            rule: pattern
            pattern: "^images/v[0-9]+/[a-f0-9-]{36}/.*\\.png$"

      - name: contentType
        logicalType: string
        description: MIME type must be image/png.
        quality:
          - type: text
            rule: mustBe
            mustBe: "image/png"

      - name: size
        logicalType: integer
        description: Each PNG file must not exceed 10 MB.
        quality:
          - type: library
            rule: mustBeLessThan
            mustBeLessThan: 10485760

      - name: tier
        logicalType: string
        description: Rendered images may be stored in Hot or Cool tier.
        quality:
          - type: text
            rule: mustBeIn
            mustBeIn: ["Hot", "Cool"]

      - name: lastModified
        logicalType: date
        description: Images must have been produced within the last 7 days.
        quality:
          - type: library
            rule: mustBeGreaterThan
            mustBeGreaterThan: "$now-7d"

quality:
  - type: text
    description: >
      No blob in the ml-artefacts/ container root is allowed outside
      the raw/ or images/ prefixes.
    rule: pattern
    mustMatch: "^(raw|images)/"
```

---

### Example 3 — Prefix hierarchy only: CSV feed directory tree on a local filesystem

A contract that focuses exclusively on enforcing a multi-level directory tree for CSV feeds stored on a local server, without object-store-specific attributes.

```yaml
apiVersion: v3.1.0
kind: DataContract
name: csv_feed_directory

servers:
  production:
    type: local
    location: /data/feeds/

schema:
  - name: csv_feeds
    logicalType: blob
    description: >
      Daily CSV feeds partitioned by source system and date.
      Layout: feeds/<source>/<YYYY>/<MM>/<DD>/<filename>.csv
    properties:
      - name: prefix
        logicalType: string
        required: true
        quality:
          - type: text
            rule: pattern
            pattern: "^feeds/[a-z_]+/[0-9]{4}/[0-9]{2}/[0-9]{2}/$"

      - name: name
        logicalType: string
        required: true
        quality:
          - type: text
            rule: pattern
            pattern: "^feeds/[a-z_]+/[0-9]{4}/[0-9]{2}/[0-9]{2}/[a-z0-9_-]+\\.csv$"

      - name: contentType
        logicalType: string
        quality:
          - type: text
            rule: mustBeIn
            mustBeIn: ["text/csv", "text/plain"]

      - name: lastModified
        logicalType: date
        description: Each partition must be refreshed within the last 25 hours.
        quality:
          - type: library
            rule: mustBeGreaterThan
            mustBeGreaterThan: "$now-25h"
```

---

### Interaction with existing server fields

`logicalType: blob` is a schema-level declaration that is fully independent of server-level fields. The table below clarifies how it coexists with common ODCS server fields:

| Field | Interaction |
| ----- | ----------- |
| `type` | Any server type is supported: `azure`, `s3`, `sftp`, `local`, etc. No restriction is imposed. |
| `format` | The server `format` field (e.g. `csv`, `parquet`) retains its meaning as a hint about the payload format. It does **not** activate or deactivate blob-schema mode; `logicalType: blob` on the schema object is the sole trigger. |
| `encoding` (RFC-0043) | Still applicable when files contain text payloads. May be omitted for opaque binary files. |
| `location` | Specifies the storage root URL or path. The `prefix` property within the schema refines the sub-directory layout inside that root. |
| `delimiter` | Not meaningful for file-property schemas; MAY be present on the server if the files are also described as tabular data in a separate schema object. |

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

### Dedicated top-level `blobPolicy` section

A separate `blobPolicy:` top-level section dedicated to blob-storage governance was considered. This would isolate file-layout rules from the schema section entirely.

Rejected because it introduces a new top-level key with overlapping concerns with `schema` and `quality`. Reusing the schema model is consistent with how ODCS already extends to new data paradigms (cf. RFC-0042 for vector types) and keeps the learning surface small.

### Extending the `servers` section only

Placing all file constraints (naming, size, type) directly on the server definition was also evaluated, alongside RFC-0043's approach for encoding.

Rejected because the server section describes a single physical access point, not a dataset with multiple objects and properties. The schema section is the right home for describing the shape of data — even when that shape is a file tree.

### `logicalType: file` instead of `logicalType: blob`

`file` is already recognised in OpenAPI 2.0 and in the original ODCS type table (RFC-0002), which makes it a natural candidate. However, `file` in both OpenAPI and ODCS has historically referred to a scalar property that holds a file upload, not to a schema object that describes a set of stored files. Using `blob` avoids this ambiguity: it signals an object-store or binary-large-object paradigm without colliding with the existing scalar `file` type. Teams targeting non-Azure platforms (S3, SFTP, local) can still use `logicalType: blob` because the name refers to the storage pattern, not to the Azure-specific "blob" concept.

### New `fileProperties` sub-object on schema

Rather than using ODCS property names to mirror BlobProperties attributes, a new sub-object `fileProperties:` could have been added alongside `properties:`. This would have been more explicit but would have required a new structural extension, while the existing `properties` model already supports all necessary semantics.

### Requiring server-type or server-format conditions in addition to `logicalType: blob`

An earlier draft of this RFC required the server to carry `type: azure` and `format: binary` as additional activation conditions. This was rejected for three reasons. First, file-layout governance is equally valuable for AWS S3, SFTP, and local filesystems, and tying the feature to a single server type would artificially limit adoption. Second, `format` is a description of the payload (CSV, Parquet, etc.) and conflating it with a schema-mode switch introduces an overloaded meaning. Third, requiring multiple conditions across separate ODCS sections increases the cognitive burden for contract authors. A single schema-side marker — `logicalType: blob` — is sufficient, explicit, and cross-platform.

## Decision

TBD by TSC.

## Consequences

### Positive

- Brings file-layer governance into the data contract, eliminating the need for out-of-band blob audit scripts.
- Reuses existing ODCS constructs: no new top-level keys, no new quality rule syntax.
- Backward-compatible: existing contracts without `logicalType: blob` are unaffected.
- The file-properties mapping table provides a stable, documented interface for tooling authors.
- Enables CI/CD validation of file layouts alongside schema and data-quality checks.
- `logicalType: blob` works across all server types (Azure, S3, SFTP, local) with a single, uniform declaration.

### Trade-offs

- Tooling must detect `logicalType: blob` at the schema level and switch interpretation mode accordingly.
- The `$now-Nd` / `$now-Nh` shorthand used in freshness quality rules is informal; a future RFC may wish to standardise relative time expressions in quality rules.
- Property names in the file-properties mapping table are opinionated: teams with different naming conventions will need to use `customProperties` for non-standard attributes.
- Some attributes (e.g. `blobType`, `leaseState`, `tier`) are platform-specific and have no equivalent on SFTP or local servers; contracts targeting those backends SHOULD omit such properties.

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
