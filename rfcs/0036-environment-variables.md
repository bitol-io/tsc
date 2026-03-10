# RFC-0036: Environment Variables in Servers

Champion: Jean-Georges Perrin

Authors: Jean-Georges Perrin, Guillaume Bodet

Slack: *TBD*

GitHub issue: *TBD*

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

Two options are presented for TSC consideration:

- **Option A** — `${VAR_NAME}` interpolation syntax in server values, accompanied by an optional top-level `variables` section that declares what the contract expects (name, description, default, required, sensitive).
- **Option B** — `${VAR_NAME}` interpolation syntax in server values only, with no `variables` section. Smaller surface area; resolution is entirely left to tooling.

### Option A vs Option B

| Concern                       | Option A                                 | Option B                                           |
| ----------------------------- | ---------------------------------------- | -------------------------------------------------- |
| Standard surface area         | Larger: new `variables` section          | Smaller: syntax only                               |
| Self-documenting contracts    | Yes: consumers can inspect requirements  | No: variable inventory lives outside the contract |
| Secret masking signal         | `sensitive: true` in the contract        | Left entirely to tooling                           |
| Required/default enforcement  | Declared in the contract                 | Left entirely to tooling                           |
| Alignment with guiding values | Favors interoperability                  | Favors a small standard                            |

### Shared: `${VAR_NAME}` interpolation syntax

Both options use the same reference syntax. Any string value within the `servers` section MAY contain one or more variable references using `${VAR_NAME}`. References MAY appear anywhere within a string value, including as substrings (e.g., `jdbc:postgresql://${DB_HOST}:${DB_PORT}/orders`).

Variable resolution is intentionally left to tooling. Common sources include OS environment variables, `.env` files, Vault, AWS Secrets Manager, Kubernetes secrets, and CI/CD pipeline variables.

---

## Option A — Interpolation with `variables` declaration

### Variable declaration: the `variables` section

A new optional top-level section `variables` declares the named variables a contract expects. Each variable entry supports:

| Field         | Type    | Required | Description                                                                                                                  |
| ------------- | ------- | -------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `name`        | string  | Yes      | Variable name, used in `${name}` references                                                                                  |
| `description` | string  | No       | Human-readable explanation of the variable's purpose                                                                         |
| `default`     | string  | No       | Default value if none is supplied by the environment                                                                         |
| `required`    | boolean | No       | When `true`, tooling MUST refuse to process the contract if no value is provided and no default exists. Defaults to `false`. |
| `sensitive`   | boolean | No       | When `true`, tooling MUST NOT log, display, or persist the resolved value in plain text. Defaults to `false`.                |

### Example A-1: Minimal — references only, no declaration

References can be used without a `variables` section when tooling has an established resolution convention:

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

### Example A-2: Full — explicit variable declarations

When the contract needs to communicate its requirements to consumers or enforce resolution:

```yaml
apiVersion: v3.2.0
kind: DataContract
id: 81a10b42-4ee9-4306-8824-34edea77e4b9
name: Orders
version: v1.0.0

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
  - name: S3_BUCKET
    description: "S3 bucket for raw data export"
    required: true

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

### Example A-3: Inline substring interpolation

```yaml
variables:
  - name: DB_HOST
    required: true
  - name: DB_PORT
    default: "5432"
  - name: DB_NAME
    required: true

servers:
  - server: prod_jdbc
    environment: prod
    type: jdbc
    connection: "jdbc:postgresql://${DB_HOST}:${DB_PORT}/${DB_NAME}"
```

### Tooling behavior under Option A (normative)

- Tools MUST resolve `${VAR_NAME}` references before using server values for any purpose.
- Tools MUST NOT store or log the resolved value of a variable declared `sensitive: true` in plain text.
- If a variable is declared `required: true` and no value or default is available, tools MUST surface an error and MUST NOT silently use an empty string.
- If a `variables` section is absent: 
  - Option A1: (or if a variable is not listed), tools SHOULD still attempt to resolve `${VAR_NAME}` references using their default resolution strategy (e.g., OS environment variables).
  - Option A2: fail if variables are used (not a breaking change).
- Tools SHOULD warn when a `${VAR_NAME}` reference appears in a server value but `VAR_NAME` is not declared in the `variables` section (Option A1).
- Tools MUST preserve unresolved `${VAR_NAME}` tokens verbatim when serializing a contract back to YAML (round-trip safety).

---

## Option B — Interpolation only, no `variables` section

Option B standardizes the `${VAR_NAME}` syntax in `servers` values without introducing any new top-level section. The contract makes no declaration about what variables it uses or whether they are required or sensitive. Tooling resolves values from whatever source it supports.

This option favors a smaller standard. It is sufficient when:
- Teams already manage variable inventories outside the contract (e.g., in infrastructure-as-code or CI/CD tooling).
- Contracts are not shared with external consumers who need to understand variable requirements.
- There is no need for the contract itself to enforce resolution.

### Example B-1: Minimal — DB connection

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

### Example B-2: Multiple servers and S3

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

### Example B-3: Inline substring interpolation

```yaml
servers:
  - server: prod_jdbc
    environment: prod
    type: jdbc
    connection: "jdbc:postgresql://${DB_HOST}:${DB_PORT}/${DB_NAME}"
```

### Tooling behavior under Option B (normative)

- Tools MUST resolve `${VAR_NAME}` references before using server values for any purpose.
- If a referenced variable cannot be resolved, tools SHOULD surface an error and MUST NOT silently use an empty string.
- Tools MUST preserve unresolved `${VAR_NAME}` tokens verbatim when serializing a contract back to YAML (round-trip safety).
- Tools MAY apply their own convention for resolution order (e.g., OS environment variables, then `.env` file).

## Alternatives

### Use `customProperties` for dynamic values

`customProperties` is a flat key/value extension mechanism. It could carry hints about secret references, but it has no semantic meaning to tooling and cannot be used to flag `sensitive` values or enforce `required` constraints. It also places the burden of defining a convention on every implementor independently.

### Rely on pre-processing outside the standard

Some teams use external templating tools (Helm, Jinja2, envsubst) to substitute values before passing the contract to tooling. This works but is invisible to the standard — consumers cannot inspect the contract and understand what variables it requires without running the pre-processor. Standardizing the `${VAR_NAME}` syntax (even under Option B) makes references machine-readable without requiring a pre-processor.

### Restrict to a defined list of secret backends

Coupling the standard to specific secret management systems (Vault, AWS Secrets Manager, etc.) would limit portability and create version drift as backends evolve. Both options keep resolution out of scope for the standard.

## Decision

> The decision made by the TSC.

## Consequences

### Positive (both options)

- Contracts can be stored in version control without exposing credentials.
- The same contract can be used across environments (prod, staging, dev) by supplying different variable values.
- Operational parameters can change without requiring consumers to re-approve the contract.

### Positive (Option A only)

- Tooling can inspect the `variables` section to validate that all required values are present before attempting a connection.
- `sensitive: true` provides a standard signal for tooling to protect secret values in logs and UIs.
- Contracts are self-documenting: consumers can see exactly what variables are required and what they mean.

### Negative

- Contracts with unresolved `${VAR_NAME}` references are not self-contained; consumers need access to a variable source to use them. (Both options.)
- Option A introduces a new concept (`variables`) that implementations must understand to process contracts correctly.

### Neutral

- Resolution order and priority (e.g., environment variable over default) are left to tooling under both options.
- This RFC does not cover variable references outside the `servers` section. A future RFC may extend the mechanism to other sections (e.g., quality rule parameters, SLA thresholds).

## References

- [RFC-0001: Servers](approved/odcs-v3.0.0/0001-servers.md) — establishes the `servers` section this RFC extends
- [Docker Compose variable substitution](https://docs.docker.com/compose/how-tos/environment-variables/variable-interpolation/) — prior art for `${VAR}` syntax in infrastructure configuration files
- [Kubernetes ConfigMaps and Secrets](https://kubernetes.io/docs/concepts/configuration/) — precedent for separating configuration declaration from runtime injection
- [The Twelve-Factor App — Config](https://12factor.net/config) — established principle of storing config in the environment, not in code
