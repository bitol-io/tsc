# Signatures — integrity and provenance for every Bitol document

Champion: Jean-Georges Perrin.

Authors: Jean-Georges Perrin.

Slack: *replace this text with the link to the dedicated Slack channel*.

GitHub issue: *replace this text with the link to the dedicated GitHub issue*.

Applies to:
* [x] ODCS - Open Data Contract Standard
* [x] ODPS - Open Data Product Standard
* [x] OORS - Open Observability Results Standard
* [x] OOCS - Open Orchestration and Control Standard
* [x] OMMS - Open Maturity Model Standard
* [x] OMDS - Open Metadata Difference Standard
* [x] OSDS - Open Semantic Definition Standard

## Summary

Add one optional top-level `signatures` array, defined once and identical in every Bitol standard, so that any Bitol document can carry one or more detached-key, enveloped digital signatures — and so that any tool can verify one without knowing which vendor produced it. The array is excluded from the signing input, which makes signatures composable: several parties can sign the same document, in any order, without invalidating each other.

## Motivation

A data contract is a promise between a producer and a consumer, and an observability result is evidence about whether that promise was kept. Both travel as files: through pull requests, catalogs, APIs, object stores, and email attachments. Today nothing in any Bitol standard lets a consumer answer two questions that matter more than any field in the document: *who committed to this*, and *has it been altered since*.

The standards are silent on this. `signature` appears in no ODCS, ODPS, or OORS schema and in no page of their documentation. Implementations that need it therefore invent it in user space — one shipping implementation embeds a signature block as an unprefixed `customProperties` entry named `signature` ([Appendix A](#appendix-a-what-one-implementation-does-today)). That works, and it proves the shape is sound, but it is the wrong home: `customProperties` is vendor and user space, where RFC 0035 makes `vendor` the discriminator and where an unprefixed name is a collision waiting to happen; nothing about the block is schema-validated; and a single-valued entry cannot express two signers. Every vendor that solves this independently produces a signature no other vendor can check, which is the exact opposite of what a signature is for.

The need is not ODCS-specific, which is why this RFC applies to all seven standards at once:

- ODCS / ODPS — a steward attests to the contract or product they publish; a consumer verifies the file they fetched is the file that was approved.
- OORS — results are machine-produced evidence. A signature binds a result set to the engine that produced it, which is what makes it usable in an audit rather than merely informative.
- OMMS / OMDS — an assessment and a diff are conclusions someone will be held to.
- OOCS / OSDS — control documents and semantic definitions are executed and depended upon; tampering with them is an attack, not an inconvenience.

This aligns with the guiding values: one mechanism, defined once, reused across the family (consistency), sitting in the document itself so it survives every transport (simplicity), and specified so independent implementations agree byte for byte (interoperability).

## Design and examples

Add an optional `signatures` array at the root of every Bitol document. It is defined once as a shared `$defs.Signature` and mirrored verbatim into each standard's JSON schema, the way `customProperties`, `tags`, and `authoritativeDefinitions` already are.

### The signature object

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `id` | string | No | Stable identifier for this signature, as elsewhere in the standards. |
| `algorithm` | string | Yes | JWA algorithm name. This RFC registers `RS256`, `ES256`, `ES384`, and `EdDSA`. |
| `canonicalization` | string | Yes | Canonicalization profile. This RFC registers exactly one value: `JCS-RFC8785`. |
| `created` | string | Yes | RFC 3339 UTC instant, asserted by the signer. |
| `role` | string | No | Why this party signed: `author`, `approver`, `steward`, `publisher`, `producer`, or `auditor`. |
| `signer` | object | Yes | Claimed identity: `name` (required), `email`, `url`, `scope` (`user`, `workspace`, or `service`). |
| `key` | object | Yes | How to obtain the verification key. Exactly one of `certificateChain` (array of PEM strings, leaf first), `jwk` (RFC 7517 public key), or `uri` (dereferenceable key location). |
| `covers` | array | No | Digests of external artifacts this signature also binds. Each entry: `url` or `id`, plus `digest` in `<alg>:<hex>` form. |
| `expires` | string | No | RFC 3339 UTC instant after which the signer no longer stands behind the document. |
| `value` | string | Yes | Base64 signature over the canonical bytes. |

The document is valid with `signatures` absent, with one entry, or with many.

### Signing

1. Parse the document into its JSON data model. YAML surface syntax — quoting, key order, comments, line width — is irrelevant, because only the data model is signed.
2. Remove the top-level `signatures` member, if present, and nothing else.
3. Serialize the result to UTF-8 bytes per RFC 8785 (JCS).
4. Sign those bytes with the private key, using `algorithm`.
5. Append the signature object to `signatures`, creating the array if needed.

### Verifying

Steps 1 to 3 are identical, then, for each entry independently: resolve the key from `key`, verify `value` over the canonical bytes, and — when `covers` is present — fetch or recompute each referenced artifact's digest and compare.

An implementation MUST report one of four outcomes per signature, and MUST NOT collapse the last two: `valid`, `invalid` (the key resolved and the signature did not match — the document was altered, or `covers` disagrees), `unverifiable` (unknown `algorithm` or `canonicalization`, or the key could not be resolved), and, for the document as a whole, `unsigned`.

Verification MUST NOT normalize values before hashing. If a tool rewrote `version: 1.0.0` as `version: v1.0.0`, the document changed and the signature is `invalid` — that is the mechanism working, not a bug in it ([Appendix B](#appendix-b-why-verification-must-not-be-lenient)).

### Example 1: minimal

```yaml
apiVersion: v3.2.0
kind: DataContract
id: 53581432-6c55-4ba2-a65f-72344a91553a
version: 1.0.0
name: Transactions
signatures:
  - algorithm: ES256
    canonicalization: JCS-RFC8785
    created: 2026-09-01T14:30:00Z
    signer:
      name: Jean-Georges Perrin
    key:
      jwk:
        kty: EC
        crv: P-256
        x: f83OJ3D2xF1Bg8vub9tLe1gHMzV76e8Tus9uPHvRVEU
        y: x_FEzRu9m36HLN_tue659LNpXW6pCyStikYjKIWI5a0
    value: MEUCIQDf1x9nQ8mZ2K3hV0pQ7yJ8Lx4aB6cE9dG2fH5iK1mNoA==
```

### Example 2: structured — two signers, a certificate chain, and a covered SBOM

```yaml
apiVersion: v1.1.0
kind: DataProduct
id: fbe8d147-28db-4f1d-bedf-a3fe9f458427
version: 2.3.0
name: Customer 360
outputPorts:
  - name: transactions
    sbom:
      - id: sbom-runtime
        type: external
        url: https://sbom.acme.com/transactions/runtime.cdx.json
signatures:
  - id: sig-author
    algorithm: ES256
    canonicalization: JCS-RFC8785
    created: 2026-09-01T14:30:00Z
    role: author
    signer:
      name: Jean-Georges Perrin
      email: jgp@acme.com
      scope: user
    key:
      uri: https://acme.com/.well-known/bitol-keys.json#jgp-2026
    covers:
      - url: https://sbom.acme.com/transactions/runtime.cdx.json
        digest: sha-256:9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08
    value: MEUCIQDf1x9nQ8mZ2K3hV0pQ7yJ8Lx4aB6cE9dG2fH5iK1mNoA==
  - id: sig-approver
    algorithm: RS256
    canonicalization: JCS-RFC8785
    created: 2026-09-02T09:12:44Z
    role: approver
    signer:
      name: Acme Data Governance
      email: governance@acme.com
      scope: workspace
    key:
      certificateChain:
        - |
          -----BEGIN CERTIFICATE-----
          MIIDXTCCAkWgAwIBAgIJAKL0UG+mRkSPMA0GCSqGSIb3DQEBCwUAMEUxCzAJBgNV
          -----END CERTIFICATE-----
        - |
          -----BEGIN CERTIFICATE-----
          MIIDdzCCAl+gAwIBAgIEAgAAuTANBgkqhkiG9w0BAQUFADBaMQswCQYDVQQGEwJJ
          -----END CERTIFICATE-----
    value: SFlt8xO2rQ9wZ1cV6nP0kM3jH7gB4dF5eA8sT2uY1oXqL9bN0cR7vK4mJ6hG3fD2
```

Both entries sign the same bytes, because the whole `signatures` array is excluded from canonicalization. The approver's signature was added without touching, and without invalidating, the author's.

### Scope of this RFC

In scope: the syntax, the canonicalization profile, the verification algorithm, and the outcome vocabulary. Out of scope, and deliberately so: which certificate authorities or key registries to trust (deployment policy), key distribution and revocation, trusted timestamping (RFC 3161), counter-signatures over other signatures, and signatures on nested objects rather than the document root.

## Alternatives

1. **Keep it in `customProperties`,** as at least one implementation does today. Rejected: it is vendor and user space, nothing is schema-validated, an unprefixed `signature` key collides with anyone else's, and a single entry cannot carry two signers. It also leaves every vendor's signature unreadable to every other vendor, which defeats the purpose.
2. **Detached sidecar files (`contract.odcs.yaml.sig`).** Rejected as the primary form: Bitol documents travel as a single file through catalogs, pull requests, and API responses, and the sidecar is the thing that gets lost. Nothing here prevents a sidecar as an additional transport.
3. **Wrap the whole document in a JWS.** Rejected: the artifact stops being a readable contract — every tool, including `grep` and a code reviewer's eyes, would have to unwrap it first. We reuse JWA algorithm names and nothing else.
4. **W3C Verifiable Credentials Data Integrity `proof` blocks.** Rejected as a requirement: it pulls JSON-LD contexts and DID resolution into a standard that has neither. The field set here is deliberately close enough that a mechanical mapping to a `proof` remains possible later.
5. **Rely on Git commit signing or Sigstore.** Rejected as the only mechanism: they attest to a repository event, not to a document, and a contract served from an API has no commit. Either can later be added as a `key` binding mode.
6. **A separate standard — an "Open Signature Standard".** Rejected: this is one optional array, not a document kind. A new standard would need its own versioning, governance, and adoption curve to deliver a field.

## Decision

Pending TSC vote.

## Consequences

- Non-breaking and additive in every standard: `signatures` is optional, and every currently valid document stays valid.
- Schema: add a shared `$defs.Signature` and the root `signatures` property to the ODCS, ODPS, and OORS JSON schemas, and to OOCS, OMMS, OMDS, and OSDS as each is published. Because `signatures` is removed before canonicalization, the root staying `additionalProperties: false` costs nothing.
- Docs: one shared page per standard, plus a signed example file and a negative fixture (a tampered document that must verify as `invalid`).
- Interoperability: the TSC publishes a set of test vectors — a document, a key pair, the expected canonical bytes, and the expected signature — as the conformance bar. Without them, independent implementations will disagree on canonicalization and every signature will be unverifiable across tools. This is the single most important deliverable after the schema change.
- Migration: implementations carrying a `customProperties`-based signature should read both forms for one release and write only `signatures`.
- Tooling: `covers` gives producers a way to bind SBOMs, attachments, and rendered documents to the signed contract, so "signed" means the whole delivery and not only the YAML.

## References

- [RFC 8785](https://www.rfc-editor.org/rfc/rfc8785) — JSON Canonicalization Scheme (JCS), the canonicalization profile registered here.
- [RFC 7515](https://www.rfc-editor.org/rfc/rfc7515) / [RFC 7518](https://www.rfc-editor.org/rfc/rfc7518) — JSON Web Signature and the JWA algorithm names reused for `algorithm`.
- [RFC 7517](https://www.rfc-editor.org/rfc/rfc7517) — JSON Web Key, the `key.jwk` form.
- [W3C Verifiable Credential Data Integrity 1.0](https://www.w3.org/TR/vc-data-integrity/) — the enveloped-proof pattern this design follows without adopting its context machinery.
- [RFC 0035 - Vendor Attribution for Custom Properties](approved/odcs-v3.2.0/0035-extensions.md) — why `customProperties` is the wrong home for a standard field.
- [RFC 0061 - Tags, custom properties and authoritative definitions for SBOMs](0061-sbom-metadata.md) — the SBOM entries that `covers` binds.
- [RFC 0024 - Extensions to customProperties and authoritativeDefinitions](0024-extensions-to-customproperties-authoritativedefinitions.md)

## Appendix A: What one implementation does today

Measured against a shipping implementation of ODCS and ODPS signing, to show the shape is proven and to name what has to change.

It embeds a single block as an unprefixed `customProperties` entry:

```yaml
customProperties:
  - property: signature
    value:
      algorithm: RS256
      canonicalization: JCS-RFC8785
      timestamp: 2026-04-08T14:30:00Z
      signer: { name: Jean-Georges Perrin, email: jgp@example.com, scope: user }
      certificateChain: ["-----BEGIN CERTIFICATE-----..."]
      digest: <base64 signature>
```

What it already gets right, and what this RFC keeps: JCS RFC 8785 canonicalization over the parsed data model; the enveloped pattern, with the signature stripped before canonicalization; RS256 and ES256 auto-detected from the key type; a lenient certificate check against the signing time rather than the verification time, which is the standard practice for code signing; and public, unauthenticated verification by document id.

What has to change, and why:

| Today | This RFC | Why |
| --- | --- | --- |
| `customProperties` entry named `signature` | Top-level `signatures` | Vendor space, unvalidated, collides |
| One block, replaced on re-sign | An array | Author, approver, and auditor all need to sign |
| `digest` | `value` | It holds a signature, not a digest |
| `timestamp` | `created` | Consistent with the family's field naming |
| Certificate chain only | `certificateChain` \| `jwk` \| `uri` | Self-signed certificates are a key transport, not a trust model |
| Nothing binds referenced files | `covers` | Signing the YAML does not sign the SBOM it points at |

## Appendix B: Why verification must not be lenient

The same implementation carries a special case: when verification fails, it retries once with the leading `v` of the top-level `version` toggled, so that a document signed as `1.0.0` still verifies after some tool rewrote it as `v1.0.0`.

It is an honest fix for a real interoperability problem — one downstream catalog matches versions verbatim and needs the `v` — but it is exactly the wrong layer. A signature answers one question: are these the bytes that were signed? Every value normalization added to a verifier is a small, permanent hole in that answer, and each one has to be implemented identically by every other verifier or the same document verifies in one tool and fails in another.

This RFC therefore forbids value normalization in verification, and pushes the problem to where it belongs: the producer signs the document in the form it will be published in, and a tool that rewrites a signed document is expected to re-sign it or to leave the signature `invalid`. That is the mechanism working.
