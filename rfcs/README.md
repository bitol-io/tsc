# RFCs

The RFC (request for comments) process provides a consistent, controlled path for changes to the Bitol standards — ODCS, ODPS, OORS, OOCS, OMMS, OMDS, OSDS — so that all stakeholders can be confident about the project's direction.

This page doubles as a **dashboard**. Open RFCs under discussion are listed first, followed by everything that has been approved (and in which standard and version), and finally the archived proposals. Every row links to its document.

## Contents

- [Open RFCs](#open-rfcs) — proposed / in discussion / awaiting a vote
- [Approved RFCs](#approved-rfcs) — accepted by the TSC, grouped by standard and version
- [Archived RFCs](#archived-rfcs) — not approved as-is (may have morphed into other RFCs)
- [The RFC process](#the-rfc-process)

## Open RFCs

Proposed or under discussion — not yet approved by the TSC. These are what remains to be worked on.

| RFC | Title |
| --- | --- |
| [0011](0011-lineage.md) | Lineage |
| [0015](0015-business-definitions.md) | Business Definitions |
| [0018](0018-oors.md) | Open Observability Results Standard (OORS) |
| [0019](0019-global-context.md) | Global Context |
| [0020](0020-named-objects.md) | Named objects |
| [0027](0027-unstructured-data-quality.md) | Unstructured Data Quality for ODCS & ODPS |
| [0031](0031-relatesTo.md) | Informational Relationship Types |
| [0032](0032-imports.md) | Imports — Reusing Definitions Across Contracts |
| [0037](0037-omms.md) | Open Maturity Model Standard (OMMS) |
| [0039](0039-omds.md) | Open Metadata Difference Standard (OMDS) |
| [0040](0040-oocs.md) | Open Orchestration and Control Standard (OOCS) |
| [0044](0044-osds.md) | Open Semantic Definition Standard (OSDS) |
| [0048](0048-geometry-geography.md) | Geometry and Geography Data Types |
| [0052](0052-blob-storage-file-logical-type.md) | Schema's Blob logicalType |
| [0056](0056-oaas.md) | Open Access Agreement Standard (OAAS) |

## Approved RFCs

Accepted by the TSC and shipped in the standard/version shown. Each links to its stub here in `rfcs/`, which points to the canonical specification under [`approved/`](approved/). An RFC shared across standards appears under each version it is part of.

### ODCS v3.0.0

| RFC | Title |
| --- | --- |
| [0001](0001-servers.md) | Servers |
| [0002](0002-types.md) | Data Types |
| [0003](0003-replacement-of-stakeholders.md) | Replacing stakeholers with productTeam structure |
| [0004](0004-hierarchical-tables.md) | Generic, hierarchical tables |
| [0005](0005-clean-implementation-specific.md) | Clean infrastructure/implementation-specific properties |
| [0006](0006-support-channels.md) | Standard support channels |
| [0007](0007-data-quality.md) | Data Quality |

### ODCS v3.1.0

| RFC | Title |
| --- | --- |
| [0009a](0009a-ref-identify-external-contract.md) | Identify an external contract |
| [0009b](0009b-ref-internal-properties.md) | Identify an element within a contract |
| [0012](0012-implicit-dq-rules.md) | Implicit Data Quality Rules |
| [0013](0013-relationships-between-properties.md) | Relationships between Properties (Foreign Keys) |
| [0016](0016-teamname.md) | Team Name |
| [0017](0017-new-date-types.md) | New Date Types |
| [0021](0021-drop-slaDefaultElement.md) | Drop slaDefaultElement |
| [0022](0022-add-description-to-sla.md) | Add description to SLA |
| [0023](0023-improve-support-channels.md) | Improve Support Channels |
| [0024](0024-extensions-to-customproperties-authoritativedefinitions.md) | Extensions to customProperties and authoritativeDefinitions |
| [0025](0025-scheduling-sla-checks.md) | Scheduling SLA Checks |
| [0026a](0026a-reference-id.md) | Adding ID Field for Stable References |
| [0026b](0026b-internal-references.md) | Internal References Using Foreign Keys |
| [0028](0028-permalink.md) | Permalink |

### ODCS v3.2.0

| RFC | Title |
| --- | --- |
| [0030](0030-maps.md) | Support Map Property Type |
| [0033](0033-enum.md) | Enum |
| [0034](0034-measures-and-dimensions.md) | Measures and Dimensions |
| [0035](0035-extensions.md) | Vendor Attribution for Custom Properties |
| [0038](0038-context.md) | Context Block for AI and Semantic Interoperability |
| [0041](0041-synonyms.md) | Synonyms |
| [0042](0042-vector-type.md) | Vector Type |
| [0043](0043-physical-data-encoding.md) | Physical Data Encoding |
| [0045](0045-hana-server-type.md) | HANA server type |
| [0046](0046-sla-custom-properties-and-authoritative-definitions.md) | Custom properties and authoritative definitions for SLAs |
| [0047](0047-relationship-id.md) | Adding `id` to Relationship Objects |
| [0049](0049-iceberg-server-type.md) | Apache Iceberg server type |
| [0050](0050-variables.md) | Variables |
| [0051](0051-deprecated-flag.md) | Deprecated flag for schema and properties |

### ODPS v0.9.0

| RFC | Title |
| --- | --- |
| [0010](0010-odps.md) | Open Data Product Standard (ODPS) |

### ODPS v1.0.0

| RFC | Title |
| --- | --- |
| [0028](0028-permalink.md) | Permalink |

### ODPS v1.1.0

| RFC | Title |
| --- | --- |
| [0029](0029-data-product-type.md) | Data Product Type Field |
| [0035](0035-extensions.md) | Vendor Attribution for Custom Properties |
| [0038](0038-context.md) | Context Block for AI and Semantic Interoperability |
| [0041](0041-synonyms.md) | Synonyms |
| [0050](0050-variables.md) | Variables |
| [0051](0051-deprecated-flag.md) | Deprecated flag for schema and properties |

### OORS v1.0.0

| RFC | Title |
| --- | --- |
| [0035](0035-extensions.md) | Vendor Attribution for Custom Properties |
| [0050](0050-variables.md) | Variables |

## Archived RFCs

Not approved in their original form. A rejected RFC may have morphed into other accepted RFCs.

| RFC | Title | Note |
| --- | --- | --- |
| [0008](0008-business-rules-versioning-attribute-level.md) | Support for business rules & versioning at the attribute level |  |
| [0009](0009-external-internal-reference_deprecated.md) | External & Internal References  | split into 0009a / 0009b / 0009c |
| [0009c](0009c-ref-contract-structure.md) | Reference Internal and External Structure |  |
| [0014](0014-quality-check-id.md) | Quality Check Identifier |  |
| [0036](0036-environment-variables.md) | Environment Variables in Servers | superseded by [0050](0050-variables.md) |

## The RFC process

Although there is no single way to prepare for submitting an RFC, it is generally a good idea to pursue feedback from other project developers beforehand, to ascertain that the RFC may be desirable; having a consistent impact on the project requires concerted effort toward consensus-building. The most common preparations include talking the idea over on our Slack channel and using a Google Doc to create the draft.

- Fork the RFC repository.
  - Copy [`0000-template.md`](0000-template.md) to `xxxx-my-feature.md`, where "my-feature" is descriptive and the number is the next in sequence.
  - Fill in the RFC.
  - Submit a pull request. As a pull request the RFC will receive design feedback from the larger community, and the author should be prepared to revise it in response.
  - Build consensus and integrate feedback. RFCs that have broad support are much more likely to make progress than those that don't receive any comments. Feel free to reach out to the RFC assignee in particular to get help identifying stakeholders and obstacles.
  - RFCs rarely go through this process unchanged, especially as alternatives and drawbacks are shown. You can make edits, big and small, to the RFC to clarify or change the design, but make changes as new commits to the pull request, and leave a comment on the pull request explaining your changes. Specifically, do not squash or rebase commits after they are visible on the pull request.
  - When ready, RFCs will be voted on by the TSC at its monthly meeting.
  - Update the RFC with the decision taken by the TSC.
  - Once approved, the RFC is soft-moved into [`approved/<standard>-v<version>/`](approved/) and a short stub is left here pointing to it, so this dashboard stays a complete index.

## Acknowledgments

- [Rust RFCs](https://github.com/rust-lang/rfcs) — Rust RFC process.
- [ghostinthewires](https://github.com/ghostinthewires/Rfcs-Template/tree/master) — an RFC process template.
