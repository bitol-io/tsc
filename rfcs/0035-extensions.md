# ODCS Extensions

**Status:** Draft  
**Champion:** TBD  
**Authors:** Gene Azad  

## Summary

This RFC standardizes a **extension mechanism** for ODCS.  

Many vendors (e.g., Confluent, Atlan, Soda, etc.) define data contracts with vendor-specific attributes that are not part of the core ODCS specification. The proposed extension mechanism enables organizations to incorporate these attributes directly into ODCS contracts without losing fidelity.



**Why not customProperties?**

Right now, customProperties are a flat string key/value that loses structure and semantics.

```yaml
customProperties:
  - id: rfc_ruleset_name
    property: refRulesetName
    value: gcsc.ruleset.name

```

With Custom_extensions, it is easier to round-trip nested, typed data (objects/arrays) from different tools.

```yaml
custom_extensions:
  - vendor: xxx
    data:
      metadata:
        tags:
          "xx":
            - "xx"
      ruleSet:
        rules:
          - name: "x"
```

If this can be achieved already with customProperties, I would suggest adding example snippets how different vendors can integrate.  


As a first example, this document defines an optional Confluent Schema Registry (SR) binding built on the extension framework. This binding allows ODCS contracts to include Schema Registry–specific registration fields, including:

- metadata (external tags + properties)  
- ruleSet (domainRules + migrationRules)  
- minimal SR registration context needed to apply them consistently (endpoint/serverRef, subject, schemaType, scope, references, optional desired compatibility)




## Motivation

ODCS already models Kafka topics and schemas, but it does not standardize:

- Confluent-specific contract parts (tags, metadata properties, domain rules, migration rules), and  
- the Schema Registry identifiers required to publish/enforce those parts (e.g., subject, schemaType, key/value scope, and often compatibility).

In Confluent Schema Registry, a “data contract” is effectively schema + tags + metadata + rules attached to a subject and registered via a payload that includes metadata and ruleSet.

Without a standardized mechanism:

- tools cannot reliably publish SR tags/metadata/rules from ODCS,  
- SR contracts cannot be round-tripped into ODCS without ad-hoc fields.

## Goals

- Keep ODCS core platform-agnostic while enabling vendor mappings via a consistent extension pattern.  
- Define a lossless, machine-operational mapping for Confluent SR metadata and ruleSet.  
- Define minimal SR context required for publish/validate behavior.

## Non-goals

- A universal rules language or execution semantics (Confluent-defined).  
- Guaranteeing compatibility can be set (treat as desired state only).

## Proposal

### A) Standard ODCS extensions container (generic)

ODCS objects MAY include a vendor extensions container.  
This RFC defines `custom_extensions` as allowed at:

- `servers[]` entries (optional defaults), and  
- schema objects (topic/event/table/etc.) (preferred location).

#### Structure

```yaml
custom_extensions:
  - vendor: string
    data: object
```

`custom_extensions` is an array of extension entries.  
Each extension entry MUST contain:

- `vendor` (string): vendor identifier  
- `data` (object): vendor-specific structured data (YAML/JSON object)

#### Vendor identifier rules

- `vendor` MUST be lowercase.  
- `vendor` MUST match `^[a-z0-9][a-z0-9-]*$` (e.g., `confluent`, `atlan`, `soda`).  
- Tooling SHOULD treat the vendor string as a stable identifier and MUST NOT rewrite it.  
- Tools MUST ignore unknown vendor values but SHOULD preserve them on read/modify/write operations when possible (round-trip safety).  

Note: A future RFC MAY define a governed “vendors registry/enum.” This RFC does not require centralized governance but defines a strict naming convention to prevent fragmentation.

#### Uniqueness

Within a given object, there MUST be at most one entry per vendor in `custom_extensions`.  
If multiple entries appear for the same vendor, tools SHOULD treat the document as invalid (or apply last-one-wins if operating in a lenient mode; strictness MAY be policy-driven).

#### Placement, precedence, and merge semantics

Extensions MAY appear at both the server level and schema-object level.

Precedence: schema-object vendor extensions override server defaults for the same vendor.

Merge semantics (normative):  
For a given vendor:

- Maps/objects SHOULD be deep-merged (schema-level wins on key conflicts).  
- Arrays MUST be replaced (schema-level array replaces server-level array; no concatenation).  
- Scalars MUST be replaced by schema-level values.  

This provides deterministic behavior for drift detection and avoids accidental rule/reference accumulation.

### B) Confluent binding: `custom_extensions[vendor=confluent].data` (optional)

When an extension entry has `vendor: confluent`, its `data` object MAY contain:

- `schemaRegistry` — SR binding context (publish/validate)  
- `metadata` — maps 1:1 to SR `metadata`  
- `ruleSet` — maps 1:1 to SR `ruleSet`

```yaml
custom_extensions:
  - vendor: confluent
    data:
      schemaRegistry: {}
      metadata: {}
      ruleSet: {}
```

## Field Definitions

### confluent.data.schemaRegistry

- `endpoint` (string, optional): Base URL for Schema Registry.  
  Implementations SHOULD support an alternative `serverRef` pattern if ODCS already has a servers registry (preferred for secrets/auth reuse).  
- `serverRef` (string, optional): Reference to a Schema Registry server configuration (if available in the ODCS servers registry). Preferred over `endpoint` when supported.  
  If both `serverRef` and `endpoint` are present, tools SHOULD prefer `serverRef` and treat `endpoint` as fallback.  
- `subject` (string, required for publishing): Schema Registry subject to register against.  
  Implementations MAY support templating (e.g., `${physicalName}-${scope}`) to accommodate naming strategy outputs.  
- `schemaType` (string, default `AVRO`): Schema type used by SR registration.  
  Known values include `AVRO`, `JSON`, `PROTOBUF`.  
  Tools SHOULD preserve unknown strings for forward compatibility.  
- `scope` (enum, default `value`): `key` | `value` | `both`  
  If `both`, tooling publishes/validates two subjects (key + value).  
- `compatibility` (string, optional): desired SR compatibility mode (e.g., `BACKWARD`, `FORWARD`, `FULL`, transitive variants, `NONE`).  
  Tooling MUST NOT assume it can set compatibility (RBAC may block it).  
  It SHOULD be treated as desired state for validation and drift detection when readable.  
- `references` (array, optional): Referenced schema dependencies used during schema registration. Each entry:  
  - `name` (string, required): reference name used in schema text  
  - `subject` (string, required): SR subject of the referenced schema  
  - `version` (integer, required): SR version number  

Note: These are SR registration references. They are not intended to replace any ODCS-native concept of schema dependency; tools MAY derive SR references from ODCS schema refs, but this RFC does not require it.

#### Subject resolution rules (normative)

- If `scope` is `key` or `value`: `subject` is the exact SR subject to use.  
- If `scope` is `both`: `subject` is treated as a base subject and tools SHOULD publish/validate:  
  - `${subject}-key` for key schema, and  
  - `${subject}-value` for value schema,  
  unless tool configuration provides an explicit override mechanism.

### confluent.data.metadata

- `tags` (map<string, string[]>, optional):  
  Keys are Confluent external tag path strings; values are arrays of tags.  
  Tools MUST treat tag-path keys as opaque strings and MUST NOT rewrite them.  
  Supports wildcard paths (including `*` and `**`) as defined by Confluent external tags.  
- `properties` (map<string, string>, optional): Free-form metadata properties.  

Note: Inline tags exist in Avro/JSON/Protobuf syntax, but this binding standardizes external tags because they round-trip cleanly via `metadata.tags` and support wildcard pathing not consistently represented via inline tags.

### confluent.data.ruleSet

Rules map 1:1 to SR rule objects inside `ruleSet.domainRules` and `ruleSet.migrationRules`.  
Each rule object SHOULD support (mirroring SR concepts):

- `name` (string, required)  
- `doc` (string, optional)  
- `kind` (string, required): typically `CONDITION` | `TRANSFORM` (tools SHOULD preserve unknown values)  
- `type` (string, optional but strongly recommended): executor type (e.g., `CEL`, `CEL_FIELD`, `JSONATA`, etc.)  
- `mode` (string, required):  
  - Domain: typically `WRITE` | `READ` | `WRITEREAD`  
  - Migration: typically `UPGRADE` | `DOWNGRADE` | `UPDOWN`  
  Tools SHOULD preserve unknown values  
- `tags` (string[], optional): tags the rule applies to  
- `params` (map<string, string>, optional): rule parameters  
- `expr` (string, optional): rule expression body  
- `onSuccess` / `onFailure` (string, optional): action types (e.g., `NONE`/`ERROR`/`DLQ`/custom)  
- `disabled` (bool, optional)

## Mapping to Confluent Schema Registry REST Payload

When publishing to SR, tooling constructs the registration payload:

```json
{
  "schemaType": "...",
  "schema": "...",
  "references": [...],
  "metadata": {
    "tags": { "...": ["..."] },
    "properties": { "...": "..." }
  },
  "ruleSet": {
    "domainRules": [ ... ],
    "migrationRules": [ ... ]
  }
}
```

Mapping:

- `confluent.data.metadata` maps 1:1 to SR `metadata`.  
- `confluent.data.ruleSet` maps 1:1 to SR `ruleSet`.  
- `confluent.data.schemaRegistry` provides publish/validate context (endpoint/serverRef, subject resolution, schemaType, scope, references, desired compatibility).

## Tooling Behavior

- Root schema rule execution: Confluent rule execution is defined on the schema registered under the subject (root). Tooling MUST NOT imply rules are enforced on referenced schemas.  
- Compatibility as desired state: Tooling MUST NOT fail publishing solely because it cannot set compatibility. Tooling SHOULD validate compatibility drift when readable.  
- Capability-aware mode: If rules/tags are unsupported in a deployment/edition/version, tools SHOULD publish what is supported and report unsupported features (strict failure optional via policy).  
- Round-trip safety: Tools SHOULD preserve unknown vendor extensions and unknown enum strings under known vendors whenever possible.

## Validation Requirements (recommended minimum)

For publish-capable tooling:  
If `vendor: confluent` extension is present and `data.schemaRegistry` is present:

- either `serverRef` or `endpoint` MUST be present,  
- `subject` MUST be present,  
- `scope` defaults to `value`,  
- `schemaType` defaults to `AVRO`.

For read-only validators:  
tools MAY allow omission of `endpoint`/`serverRef` and treat the entry as descriptive-only.

## Examples

### Example A: Minimal (subject + external tags)

```yaml
apiVersion: v3.1.0
kind: DataContract
id: orders
name: Orders Event Stream
version: 0.0.1

servers:
  - server: my-kafka
    type: kafka
    host: kafkabroker1:9092
    format: avro

schema:
  - name: Orders
    physicalName: orders
    physicalType: topic
    custom_extensions:
      - vendor: confluent
        data:
          schemaRegistry:
            endpoint: https://schemaregistry.acme.com
            schemaType: AVRO
            scope: value
            subject: orders-value
          metadata:
            tags:
              "**.ssn": ["PII", "PRIVATE"]
```

### Example B: Subject templating (naming-strategy friendly)

```yaml
schema:
  - name: Orders
    physicalName: orders
    physicalType: topic
    custom_extensions:
      - vendor: confluent
        data:
          schemaRegistry:
            endpoint: https://schemaregistry.acme.com
            scope: value
            subject: "${physicalName}-${scope}"   # -> orders-value
```

### Example C: Full (metadata + rules + compatibility + references)

```yaml
schema:
  - name: Orders
    physicalName: orders
    physicalType: topic
    custom_extensions:
      - vendor: confluent
        data:
          schemaRegistry:
            endpoint: https://schemaregistry.acme.com
            schemaType: AVRO
            scope: value
            subject: orders-value
            compatibility: BACKWARD
            references:
              - name: com.acme.Customer
                subject: customer-value
                version: 3

          metadata:
            tags:
              "**.ssn": ["PII", "PRIVATE"]
            properties:
              owner: "Bob Jones"
              email: "bob@acme.com"

          ruleSet:
            domainRules:
              - name: validateAge
                doc: "Age must be positive"
                kind: CONDITION
                type: CEL
                mode: WRITEREAD
                expr: "message.age > 0"
                onFailure: "ERROR"
            migrationRules:
              - name: v1_to_v2
                kind: TRANSFORM
                type: JSONATA
                mode: UPGRADE
                expr: "/* jsonata expression */"
```

### Example D: Key + value (scope=both)

```yaml
schema:
  - name: Orders
    physicalName: orders
    physicalType: topic
    custom_extensions:
      - vendor: confluent
        data:
          schemaRegistry:
            endpoint: https://schemaregistry.acme.com
            scope: both
            subject: orders
            # resolves to orders-key and orders-value by default
```

## Alternatives

### 1) Use customProperties only

ODCS `customProperties` is intentionally generic (flat key/value), which makes it a poor fit for Confluent’s nested structures:

- `metadata.tags` is a structured map of paths → arrays  
- rules are arrays of objects with enumerated kind/mode plus optional params/actions  

Using only `customProperties` hinders interoperability and round-tripping because tooling can’t reliably infer structure.

### 2) Use authoritativeDefinitions only (link out)

Linking to an external artifact is human-navigable, but not machine-operational for:

- publishing tags/metadata/rules into SR,  
- validating ODCS ↔ SR equivalence,  
- round-tripping back from SR,  
- enforcing rule modes/actions in a consistent representation.

## Decision

TBD by TSC

## Consequences

### Positive

- Enables Confluent Schema Registry contract publishing and enforcement from ODCS.  
- Standardizes a portable representation of Confluent tags/metadata/rules across ODCS tooling.

### Trade-offs / risks

- Introduces vendor-specific surface area (mitigated by being explicitly optional and namespaced by vendor id).

## References

- ODCS supports custom properties having any data type of the value (as defined [in the JSON schema](https://github.com/bitol-io/open-data-contract-standard/blob/main/schema/odcs-json-schema-latest.json#L2717))
- Confluent docs: “Data Contracts for Schema Registry” (tags, metadata, rules, payload shape, modes)  
- Confluent docs: “Schema Registry API Reference” (schema registration request fields; compatibility types)  
- Open Semantic Interchange supports a similar [extension pattern](https://github.com/open-semantic-interchange/OSI/blob/main/core-spec/spec.md#vendors)

