# Single Artifact — embedding referenced documents for delivery

Champion: Jean-Georges Perrin.

Authors: Jean-Georges Perrin.

Slack: *replace this text with the link to the dedicated Slack channel*.

GitHub issue: *replace this text with the link to the dedicated GitHub issue*.

Applies to:
* [x] ODCS - Open Data Contract Standard
* [x] ODPS - Open Data Product Standard
* [ ] OORS - Open Observability Results Standard
* [ ] OOCS - Open Orchestration and Control Standard
* [ ] OMMS - Open Maturity Model Standard
* [ ] OMDS - Open Metadata Difference Standard
* [x] OSDS - Open Semantic Definition Standard

> **Style.** Be concise. The body says what changes and why, and nothing else. Anything that needs explaining — rationale, background, prior art, comparisons — goes in an appendix.

## Summary

A Bitol delivery is several documents: a product, the contracts its ports expose, the semantic definitions those contracts bind to. Several files is right while they are authored — each is owned, versioned and reviewed by a different team. It is wrong at a system boundary, where the delivery has to be one file a consumer can read, validate and verify without fetching anything.

This RFC opens the question of letting one document **embed the documents it references**, so a product plus its contracts plus its definitions travel as a single, signable artifact. It presents three candidate shapes and asks the TSC which to specify; it does not pick one.

## Motivation

Ownership splits files. Delivery wants one.

- **References stop resolving at the boundary.** A `contractId`, a file path, an RFC-0032 locator all resolve in the repository that holds them. None of them resolves in a Kafka message, an API response, an email attachment, a customer's air-gapped environment, or a catalog that ingested the file two years ago.
- **A signature covers one document.** [RFC-0062](0062-signatures.md) signs the canonical bytes of the document it sits in. A product whose three ports reference three contracts by id is signable, but the signature attests to the ids, not to what they resolve to — a consumer who verifies it learns nothing about the contracts they actually received. One artifact, one signature, one verification.
- **RFC-0032 does not reach this far.** Its flattening resolves a reference *into the element that carries it*, within one document. Embedding a whole document inside another is a different operation, and today it is impossible: ODPS ports carry `contractId` and `version`, never a contract.
- **Every ecosystem that got delivery right ships one file.** A container image manifest, a `package-lock.json`, a CycloneDX SBOM. Each is a bundle whose whole point is that the consumer does not have to go and fetch the parts.

## Design and examples

Three shapes. Each is sketched with the same delivery: a data product with two output ports, both exposing contracts, one contract binding to a semantic definition.

### Shape 1 — Inline at the reference site

The referencing element carries the referenced document.

```yaml
apiVersion: v1.1.0
kind: DataProduct
id: acme.sales.customer-360
version: 2.3.0
name: Customer 360
outputPorts:
  - id: c360_tabular
    name: tabular
    contract:                      # the whole ODCS document, inline
      apiVersion: v3.2.0
      kind: DataContract
      id: acme.sales.customer-360.tabular
      version: 1.4.0
      schema:
        - id: customers_tbl
          name: customers
          properties:
            - id: cust_id
              name: id
              logicalType: string
```

Obvious to read and to write. Two ports sharing one contract embed it twice, and an OSDS document, which no port references, has nowhere to go.

### Shape 2 — A document bundle at the root

A top-level array carries whole documents, each keeping its own envelope. Existing references resolve against the bundle first, then outward.

```yaml
apiVersion: v1.1.0
kind: DataProduct
id: acme.sales.customer-360
version: 2.3.0
name: Customer 360
outputPorts:
  - id: c360_tabular
    name: tabular
    contractId: acme.sales.customer-360.tabular
    version: 1.4.0
  - id: c360_events
    name: events
    contractId: acme.sales.customer-360.events
    version: 1.1.0

documents:
  - apiVersion: v3.2.0
    kind: DataContract
    id: acme.sales.customer-360.tabular
    version: 1.4.0
    schema:
      - id: customers_tbl
        name: customers
        properties:
          - id: cust_clv
            name: clv
            physicalType: decimal(18,2)
            authoritativeDefinitions:
              - type: semanticDefinition
                url: acme.sales.semantics@1.2.0#/definitions/customer-lifetime-value

  - apiVersion: v3.2.0
    kind: DataContract
    id: acme.sales.customer-360.events
    version: 1.1.0
    schema:
      - id: events_tbl
        name: events

  - apiVersion: v1.0.0
    kind: SemanticDefinition
    id: acme.sales.semantics
    version: 1.2.0
    definitions:
      - id: customer-lifetime-value
        name: Customer Lifetime Value
        appliesTo:
          - property
        logicalType: number
        description: Net margin expected over the customer relationship.

signatures:
  - algorithm: ES256
    canonicalization: JCS-RFC8785
    created: 2026-09-01T14:30:00Z
    signer:
      name: Acme Data Governance
    key:
      uri: https://acme.com/.well-known/bitol-keys.json#governance-2026
    value: MEUCIQDf1x9nQ8mZ2K3hV0pQ7yJ8Lx4aB6cE9dG2fH5iK1mNoA==
```

One copy per document, whatever references it and however often. It covers every reference kind, not only ports — the OSDS document is reachable because the bundle is addressed by id, not by position. Strip `documents:` and what remains is exactly the product as authored. The cost is a new top-level section in every standard that can carry a bundle.

### Shape 3 — An envelope kind

A new `kind: Bundle` holding a list of documents and naming the root one. No change to any existing standard, but every tool must learn a new document kind, and the product stops being the artifact — what is delivered is a box with a product in it.

### Resolution order

Whichever shape is chosen, resolution is: **the bundle first, then the outside world.** An id present in the bundle resolves there and no fetch happens. An id present in the bundle at a version other than the one requested is an error, never a silent fallback to the network. The same id and version twice in a bundle is an error.

This composes with [RFC-0032](0032-imports.md) rather than replacing it: RFC-0032 says how a reference is written and resolved, this says where the resolver looks first.

### Is an embedded copy a cache or the artifact?

The question the TSC has to answer, because everything else follows from it.

- **A cache** — the canonical document wins whenever it is reachable, and the embedded copy is a convenience for when it is not.
- **The artifact** — what was bundled, signed and delivered is what is true. A later divergence from the canonical source is a governance event to be reported, not an override to be applied silently.

This RFC recommends **the artifact**. A signed delivery whose content can be changed by something it references has not been delivered, and a consumer cannot verify what they were given.

### Signing

Bundle, then sign. Never the reverse: bundling a signed document changes its bytes, and RFC-0062 forbids lenient verification, so the signature would correctly read as `invalid`.

With shape 2, one signature over the outer document covers every embedded document, because they are part of the canonical bytes. RFC-0062's `covers` stays the tool for what you deliberately do not embed — a large SBOM, a rendered PDF — binding it by digest instead of by value.

## Alternatives

- **Do nothing; resolve at read time.** The status quo. It works in a repository and fails at every boundary that matters.
- **A sidecar archive (tar, zip) holding the document set.** Loses the single-file property that makes a Bitol document readable, greppable and schema-validated, and the sidecar is the thing that gets lost.
- **RFC-0062 `covers` digests alone.** Proves what a referenced document was at signing time, which is real value, but the consumer still has to fetch it — the boundary problem is untouched.
- **An OCI artifact or image manifest.** The right shape from the right ecosystem, and worth revisiting for distribution, but it moves the answer out of the YAML that every other Bitol standard lives in.

## Decision

> Pending. This RFC is deliberately a discussion opener: it names the problem and three shapes, and asks the TSC which to specify.

## Consequences

- Additive and optional in every standard it touches. A document with no bundle is unchanged.
- Duplication is the cost, accepted deliberately — the same trade RFC-0032 Option B makes, for the same reason.
- Bundled deliveries are large. Tooling has to stream rather than assume a document fits comfortably in memory.
- Version drift between an embedded copy and its canonical source becomes visible and reportable, where today it is invisible.
- RFC-0062 signatures become meaningful at the delivery boundary: what is signed is what the consumer got, in full.
- A bundle is a snapshot. Deciding it is the artifact rather than a cache means a consumer can be reading a definition its owning domain has already superseded — which is the point, and has to be said out loud.

## References

- [RFC-0032 (Imports)](0032-imports.md) — reference notation, resolution and flattening within a document; this RFC extends the idea between documents.
- [RFC-0044 (OSDS)](0044-osds.md) — the semantic definitions a delivery embeds alongside its contracts.
- [RFC-0062 (Signatures)](0062-signatures.md) — the signature a single artifact makes meaningful, and the `covers` mechanism for what stays outside.
- [RFC-0061 (SBOM metadata)](0061-sbom-metadata.md) — SBOMs are the canonical example of something to bind by digest rather than embed.
- ODPS output and input ports: `contractId` and `version`, the reference this RFC would let a delivery resolve in place.
- `package-lock.json`, OCI image manifests, CycloneDX: prior art for shipping the dependency set with the thing that depends on it.
