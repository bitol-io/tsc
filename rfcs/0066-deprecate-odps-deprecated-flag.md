# Deprecate the `deprecated` flag on ODPS data products and ports

Champion: Simon Harrer

Authors: Simon Harrer

Applies to:
* [ ] ODCS - Open Data Contract Standard
* [x] ODPS - Open Data Product Standard
* [ ] OORS - Open Observability Results Standard
* [ ] OOCS - Open Orchestration and Control Standard
* [ ] OMMS - Open Maturity Model Standard
* [ ] OMDS - Open Metadata Difference Standard
* [ ] OSDS - Open Semantic Definition Standard

## Summary

Deprecate the `deprecated` boolean on the ODPS data product and on input, output and management ports in ODPS v1.2.0, and remove it in ODPS v2.0.0.

## Motivation

[RFC-0051](approved/odcs-v3.2.0/0051-deprecated-flag.md) added a `deprecated` flag to schema objects and properties. Adding it to the ODPS data product and its ports in v1.1.0 was an oversight. The RFC was written for schema elements, and ODPS already has a way to say this:

- **Data product:** `status: deprecated` already marks a data product as deprecated. A second field means two sources of truth that can disagree, e.g. `status: active` with `deprecated: true`.
- **Input and output ports:** a port points to a data contract through `contractId`, and the contract's `status` (`deprecated`, `retired`, …) already says whether its data should still be used.
- **Management port:** a management port is an operational endpoint (discoverability, observability, control). One that should no longer be used is removed; a notice in `description` covers the transition.

We cannot remove the field in a minor version without breaking v1.1.0 documents, so we deprecate it now and remove it in the next major version.

## Design and examples

### ODPS v1.2.0

- `deprecated` on the data product (root), `inputPorts[]`, `outputPorts[]` and `managementPorts[]` stays valid but is marked deprecated.
- The JSON Schema marks each with the `deprecated: true` annotation, so editors and validators can warn.
- The documentation marks the field as deprecated and names its replacement:

| Element         | Deprecated field | Use instead                                          |
| --------------- | ---------------- | ---------------------------------------------------- |
| Data product    | `deprecated`     | `status: deprecated`                                 |
| Input port      | `deprecated`     | `status: deprecated` on the referenced data contract |
| Output port     | `deprecated`     | `status: deprecated` on the referenced data contract |
| Management port | `deprecated`     | remove the port; announce it in `description`        |

- Implementations SHOULD warn when the field is used, and SHOULD treat a data product with `deprecated: true` as `status: deprecated`.

### ODPS v2.0.0

- Remove `deprecated` from the data product, `inputPorts[]`, `outputPorts[]` and `managementPorts[]`.

### Example: data product

Before (v1.1.0):

```yaml
apiVersion: v1.1.0
kind: DataProduct
id: 064c4630-8aad-4dc0-ba95-0f69940e6b18
name: customer-360
status: active
deprecated: true
```

After:

```yaml
apiVersion: v1.2.0
kind: DataProduct
id: 064c4630-8aad-4dc0-ba95-0f69940e6b18
name: customer-360
status: deprecated
description:
  purpose: "DEPRECATED: use 'customer-360-v2' instead."
```

### Example: output port

Before (v1.1.0):

```yaml
outputPorts:
  - name: legacy-export
    version: 1.0.0
    contractId: customer-360-legacy-export
    deprecated: true
  - name: tables
    version: 2.0.0
    contractId: customer-360-tables
```

After, the port no longer carries the flag:

```yaml
outputPorts:
  - name: legacy-export
    version: 1.0.0
    contractId: customer-360-legacy-export
  - name: tables
    version: 2.0.0
    contractId: customer-360-tables
```

and the referenced data contract carries the status:

```yaml
apiVersion: v3.1.0
kind: DataContract
id: customer-360-legacy-export
version: 1.0.0
status: deprecated
description:
  purpose: "DEPRECATED: use the 'tables' output port instead."
```

### Example: management port

Before (v1.1.0):

```yaml
managementPorts:
  - name: legacy-status
    content: observability
    type: rest
    url: https://example.com/legacy/status
    deprecated: true
```

After, until the port is removed:

```yaml
managementPorts:
  - name: legacy-status
    content: observability
    type: rest
    url: https://example.com/legacy/status
    description: "DEPRECATED: use 'status' instead. Will be removed on 2027-01-31."
```

Input ports follow the output port example: drop the flag and rely on the referenced contract's `status`.

## Alternatives

- **Keep the flag.** Rejected: two fields for one fact on the data product, and input and output ports duplicate their contract's status.
- **Remove the flag in v1.2.0.** Rejected: breaks valid v1.1.0 documents in a minor version.
- **Add a `status` field to ports.** Rejected for now: the contract status already covers it. Can be proposed separately if a port needs a lifecycle of its own.

## Decision

> The decision made by the TSC.

## Consequences

- ODPS v1.2.0: nonbreaking. The field stays valid and is marked deprecated.
- ODPS v2.0.0: breaking. Documents that still use the field must move to `status`.
- `deprecated` on ODCS schema objects and properties (RFC-0051) is unchanged.

## References

- [RFC-0051: Deprecated flag for schema and properties](approved/odcs-v3.2.0/0051-deprecated-flag.md)
- [ODPS `deprecated` documentation](https://github.com/bitol-io/open-data-product-standard/blob/main/docs/deprecated.md)
- [JSON Schema 2020-12 `deprecated` annotation](https://json-schema.org/draft/2020-12/json-schema-validation.html#name-deprecated)
