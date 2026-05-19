# RFC-0036: Environment Variables in Servers

Champion: Jean-Georges Perrin

Authors: Jean-Georges Perrin, Guillaume Bodet

Slack: [#bitol-wg](https://data-mesh-learning.slack.com/archives/C089S376YGM)

GitHub issue: [TSC #50](https://github.com/bitol-io/tsc/issues/50)

Applies to:
* [x] ODCS - Open Data Contract Standard
* [ ] ODPS - Open Data Product Standard
* [ ] OORS - Open Observability Results Standard
* [ ] OOCS - Open Orchestration and Control Standard
* [ ] OMMS - Open Maturity Model Standard
* [ ] OMDS - Open Metadata Difference Standard

## Summary

This RFC introduces a standardized mechanism to reference named variables within the `servers` section of a data contract, enabling server configuration values to be injected at runtime rather than hardcoded. Variables are resolved from environment variables, secrets managers, or any external source at runtime.

## Motivation

Data contracts describe how data is accessed, including server connection details such as hostnames, ports, database names, and credentials. Today, these values must be hardcoded directly in the contract:

```yaml
servers:
  - server: prod_db
    environment: prod
    type: postgresql
    host: db.prod.acme.com
    port: 5432
    database: orders_prod
    password: supersecret
```

This creates two concrete problems:

**1. Secrets are stored in the contract.**
Passwords, API keys, and tokens may in plain text, making it difficult to store contracts in version control, share them with consumers, or pass them through CI/CD pipelines without exposing credentials.

**2. Operational parameters cannot be changed without editing the contract.**
Connection strings, endpoints, and S3 bucket paths may change as infrastructure evolves. Consumers who have agreed to a contract should not need to re-approve it because a hostname changed.

This RFC aligns with the guiding value of **favoring interoperability**: tools that provision infrastructure, enforce policies, or consume contracts need a standard way to resolve dynamic values without each implementing its own substitution convention.

## Design and examples

The working group recommends standardizing **`${VAR_NAME}` interpolation in server values only**, with no in-contract `variables` declaration section. This keeps the standard surface small — the contract simply marks where substitution happens, and resolution is left entirely to tooling.

A richer variant — adding a top-level `variables` declaration block (name, description, default, required, sensitive) — was considered and is **deferred to a future RFC**; see [Future considerations](#future-considerations).

### `${VAR_NAME}` interpolation syntax

Any string value within the `servers` section MAY contain one or more variable references using `${VAR_NAME}`. References MAY appear anywhere within a string value, including as substrings (e.g., `jdbc:postgresql://${DB_HOST}:${DB_PORT}/orders`).

Variable resolution is intentionally left to tooling. Common sources include OS environment variables, `.env` files, Vault, AWS Secrets Manager, Kubernetes secrets, and CI/CD pipeline variables.

### Example 1: Minimal — DB connection

```yaml
apiVersion: v3.2.0
kind: DataContract
id: 81a10b42-4ee9-4306-8824-34edea77e4b9
name: Orders
version: v1.0.0

servers:
  - server: prod_db
    environment: prod
    type: postgresql
    host: ${DB_HOST}
    port: ${DB_PORT}
    database: orders_prod
    password: ${DB_PASSWORD}
```

### Example 2: Multiple servers and S3

```yaml
apiVersion: v3.2.0
kind: DataContract
id: 81a10b42-4ee9-4306-8824-34edea77e4b9
name: Orders
version: v1.0.0

servers:
  - server: prod_db
    environment: prod
    type: postgresql
    host: ${DB_HOST}
    port: ${DB_PORT}
    database: orders_prod
    password: ${DB_PASSWORD}

  - server: dev_db
    environment: dev
    type: postgresql
    host: ${DB_HOST}
    port: ${DB_PORT}
    database: orders_dev
    password: ${DB_PASSWORD}

  - server: prod_s3
    environment: prod
    type: s3
    location: s3://${S3_BUCKET}/orders/
```

### Example 3: Inline substring interpolation

```yaml
servers:
  - server: prod_jdbc
    environment: prod
    type: jdbc
    connection: "jdbc:postgresql://${DB_HOST}:${DB_PORT}/${DB_NAME}"
```

### Tooling behavior (normative)

- Tools MUST resolve `${VAR_NAME}` references before using server values for any purpose.
- If a referenced variable cannot be resolved, tools SHOULD surface an error and MUST NOT silently use an empty string.
- Tools MUST preserve unresolved `${VAR_NAME}` tokens verbatim when serializing a contract back to YAML (round-trip safety).
- Tools MAY apply their own convention for resolution order (e.g., OS environment variables, then `.env` file).

## Alternatives

### Use `customProperties` for dynamic values

`customProperties` is a flat key/value extension mechanism. It could carry hints about secret references, but it has no semantic meaning to tooling and cannot be used to flag `sensitive` values or enforce `required` constraints. It also places the burden of defining a convention on every implementor independently.

### Rely on pre-processing outside the standard

Some teams use external templating tools (Helm, Jinja2, envsubst) to substitute values before passing the contract to tooling. This works but is invisible to the standard — consumers cannot inspect the contract and understand what variables it requires without running the pre-processor. Standardizing the `${VAR_NAME}` syntax makes references machine-readable without requiring a pre-processor.

### Restrict to a defined list of secret backends

Coupling the standard to specific secret management systems (Vault, AWS Secrets Manager, etc.) would limit portability and create version drift as backends evolve. This RFC keeps resolution out of scope for the standard.

## Open Discussion: Alternative Reference Syntax

### Existing `${...}` usage across the standards

The `${VAR_NAME}` interpolation syntax is not new to this RFC. It is already used in several places:

| Location                                 | Usage                      | Example                                                     |
| ---------------------------------------- | -------------------------- | ----------------------------------------------------------- |
| **ODCS JSON schema** (all versions)      | SQL quality rule examples  | `SELECT COUNT(*) FROM ${table} WHERE ${column} IS NOT NULL` |
| **RFC-0007** (data quality, approved)    | SQL quality rule queries   | `${table}`, `${column}`                                     |
| **RFC-0027** (unstructured data quality) | SQL quality rule queries   | `${object}`, `${property}`                                  |
| **RFC-0035** (extensions)                | Extension naming templates | `${physicalName}-${scope}`                                  |
| **RFC-0038** (context)                   | Query answer templates     | `${total_turnover_euros}`                                   |
| **example.yaml** (v2.x reference)        | Server credentials         | `${env.username}`, `${env.password}`                        |

The `${...}` syntax is a de facto convention in ODCS for any kind of runtime substitution — SQL templates, extension naming, and environment values. This RFC would formalize and standardize what is already an established pattern.

ODPS does not currently use `${...}` in any standard documents or schema.

### Alternative: `$var` annotation syntax

This RFC proposes the `${VAR_NAME}` interpolation syntax. An alternative approach — consistent with RFC-0032 Option B's `$import` annotation — would use a `$var` field-level annotation instead of string interpolation.

### `$var` annotation syntax

Instead of embedding `${VAR_NAME}` inside string values, each field that needs variable resolution carries a `$var` annotation alongside its materialized value:

**Example — `${VAR_NAME}` syntax (current proposal):**
```yaml
servers:
  - server: prod_db
    environment: prod
    type: postgresql
    host: ${DB_HOST}
    port: ${DB_PORT}
    database: orders_prod
    password: ${DB_PASSWORD}
```

**Example — `$var` annotation syntax (alternative):**
```yaml
servers:
  - server: prod_db
    environment: prod
    type: postgresql
    host:
      $var: DB_HOST
      value: db.prod.acme.com
    port:
      $var: DB_PORT
      value: 5432
    database: orders_prod
    password:
      $var: DB_PASSWORD
      value: ""
```

### Trade-offs

| Concern                       | `${VAR_NAME}` syntax                                          | `$var` annotation syntax                                                    |
| ----------------------------- | ------------------------------------------------------------- | --------------------------------------------------------------------------- |
| **Self-contained contract**   | No — unresolved `${...}` tokens are not valid values          | Yes — `value` always holds a usable default or last-known value             |
| **Readability**               | Compact, familiar to DevOps teams                             | More verbose, but explicit about what is variable vs fixed                  |
| **Substring interpolation**   | Natural: `jdbc:postgresql://${DB_HOST}:${DB_PORT}/${DB_NAME}` | Not supported — each field is either fully variable or fully fixed          |
| **Consistency with RFC-0032** | Different mechanism (`${...}` vs `$import`)                   | Same pattern — `$` prefix annotation on fields, declaration side + use side |
| **Tooling complexity**        | String parsing required to find/replace tokens                | Structured field — tooling reads `$var` key directly                        |

### Combined example with the deferred `variables` declaration block

```yaml
variables:
  - name: DB_HOST
    description: "Hostname of the PostgreSQL database server"
    required: true
  - name: DB_PORT
    description: "Port for the PostgreSQL database server"
    default: "5432"
  - name: DB_PASSWORD
    description: "Password for the database service account"
    required: true
    sensitive: true

servers:
  - server: prod_db
    environment: prod
    type: postgresql
    host:
      $var: DB_HOST
      value: db.prod.acme.com
    port:
      $var: DB_PORT
      value: 5432
    database: orders_prod
    password:
      $var: DB_PASSWORD
      value: ""
```

The `variables` section (deferred — see [Future considerations](#future-considerations)) would declare what the contract expects. The `$var` annotation marks which fields are variable-resolved, while `value` holds the materialized/default content — making the contract self-contained even before resolution.

The TSC should decide whether the familiarity and compactness of `${VAR_NAME}` outweighs the self-containment and structural consistency of `$var`.

## Future considerations

### Future 1 — Top-level `variables` declaration block

A natural follow-up is to add a top-level `variables` section that declares the variables a contract expects (description, default value, required, sensitive). This trades a slightly larger standard surface for self-documenting contracts: consumers can inspect requirements, tooling can validate completeness before connecting, and `sensitive: true` becomes a standard signal for secret masking.

Sketch:

```yaml
variables:
  - name: DB_HOST
    description: "Hostname of the PostgreSQL database server"
    required: true
  - name: DB_PORT
    description: "Port for the PostgreSQL database server"
    default: "5432"
  - name: DB_PASSWORD
    description: "Password for the database service account"
    required: true
    sensitive: true

servers:
  - server: prod_db
    type: postgresql
    host: ${DB_HOST}
    port: ${DB_PORT}
    password: ${DB_PASSWORD}
```

Open design points to settle in the follow-up RFC:

- **Identifier key** for each entry — candidates are `name` (consistent with the rest of ODCS/ODPS), `variable` (self-describing inside a `variables[]` array), or `id` (emphasizes identifier role, but may clash with ODCS's use of `id` for UUID-like stable identifiers).
- **Behavior when a `${VAR_NAME}` reference is used without a matching declaration** — warn, error, or silently fall through to the runtime resolution strategy.
- **Whether the section is optional** (additive over this RFC) or **mandatory** (every `${...}` reference must be declared).
- **Sensitivity propagation** into logs, diffs, and tool UIs.

This is deferred so this RFC can stay minimal. The `${VAR_NAME}` syntax adopted here is forward-compatible with any of the design choices above.

### Future 2 — Variables + `$import`: parameterized fragments

This RFC scopes variable references to the `servers` section. A natural extension is to combine variables with **RFC-0032 `$import`** so that an imported fragment becomes a parameterized template — effectively **turning a data contract (or part of one) into a function**.

#### Sketch: parameterizing an imported SLA block

Today, SLA properties are hardcoded in a contract. With variables + `$import`, a reusable SLA fragment could be imported and bound at the call site:

```yaml
# slas/gold-tier.odcs.yaml — reusable, parameterized fragment
slaProperties:
  - property: latency
    value: ${SLA_LATENCY_MS}
    unit: ms
  - property: retention
    value: ${SLA_RETENTION_DAYS}
    unit: d
  - property: availability
    value: ${SLA_AVAILABILITY_PCT}
    unit: "%"
```

```yaml
# orders.odcs.yaml — caller binds the parameters
apiVersion: v3.2.0
kind: DataContract
id: orders
name: Orders

variables:
  - name: SLA_LATENCY_MS
    default: "500"
  - name: SLA_RETENTION_DAYS
    default: "90"
  - name: SLA_AVAILABILITY_PCT
    default: "99.9"

$import: ./slas/gold-tier.odcs.yaml
```

The imported fragment behaves like a function body; the `variables` (or inline overrides at the import site) behave like the call arguments. The same fragment can produce a "gold", "silver", or "bronze" SLA contract by varying the bindings, without duplicating the fragment.

#### What this would unlock

- **Reusable SLA tiers, quality rule packs, and server templates** shared across many contracts.
- **Environment specialization** of a single canonical contract (prod vs. dev parameters injected at import time).
- **Policy-as-code patterns**: a central governance team publishes parameterized building blocks; product teams import and bind.
- **Composition**: a contract becomes a small program that imports parameterized pieces and supplies bindings — a step toward treating data contracts as first-class composable artifacts.

#### What needs to be worked out (out of scope here)

- **Scoping rules**: do variables defined in the caller flow into the imported fragment, or must bindings be passed explicitly (e.g., an `with:` block on the `$import`)?
- **Required vs. optional parameters** at the fragment boundary, and how to surface missing bindings as errors.
- **Override precedence** between caller-defined variables, fragment defaults, and the runtime environment.
- **Sensitivity propagation** — a `sensitive: true` flag on a caller variable must follow the value into the imported fragment.
- **Round-trip and resolution semantics** when a fragment is rendered without bindings.

These are explicitly **deferred to a follow-up RFC** so this RFC can stay small. The intent here is only to record that the design space exists and that the `${VAR_NAME}` (or `$var`) mechanism chosen by this RFC should be forward-compatible with that future.

## Decision

TBD. With the `variables` declaration block deferred to a follow-up RFC (see [Future considerations](#future-considerations)), the only remaining item open for the TSC is:

1. **Reference syntax** — `${VAR_NAME}` interpolation vs. `$var` field-level annotation (see "Open Discussion: Alternative Reference Syntax").

The WG recommends the `${VAR_NAME}` form. This item will be put to a vote at the next TSC meeting.

## Consequences

### Positive

- Contracts can be stored in version control without exposing credentials.
- The same contract can be used across environments (prod, staging, dev) by supplying different variable values.
- Operational parameters can change without requiring consumers to re-approve the contract.
- Small standard surface: a single interpolation syntax, no new top-level section.

### Negative

- Contracts with unresolved `${VAR_NAME}` references are not self-contained; consumers need access to a variable source to use them.
- No in-contract signal for which variables are required or sensitive — that responsibility lives in tooling until the `variables` declaration block in [Future considerations](#future-considerations) is adopted.

### Neutral

- Resolution order and priority (e.g., environment variable over default) are left to tooling.
- This RFC does not cover variable references outside the `servers` section. A future RFC may extend the mechanism to other sections (e.g., quality rule parameters, SLA thresholds — see [Future 2](#future-2--variables--import-parameterized-fragments)).

## References

- [RFC-0001: Servers](approved/odcs-v3.0.0/0001-servers.md) — establishes the `servers` section this RFC extends
- [Docker Compose variable substitution](https://docs.docker.com/compose/how-tos/environment-variables/variable-interpolation/) — prior art for `${VAR}` syntax in infrastructure configuration files
- [Kubernetes ConfigMaps and Secrets](https://kubernetes.io/docs/concepts/configuration/) — precedent for separating configuration declaration from runtime injection
- [The Twelve-Factor App — Config](https://12factor.net/config) — established principle of storing config in the environment, not in code
