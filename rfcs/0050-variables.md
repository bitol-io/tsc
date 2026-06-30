# RFC-0050: Variables

Champion: Jean-Georges Perrin

Authors: Jean-Georges Perrin, Guillaume Bodet, Jochen Christ, Patrick Beitsma, Tom De Wolf, Diego Carvallo, Simon Harrer

Slack: [#bitol-wg](https://data-mesh-learning.slack.com/archives/C089S376YGM)

GitHub issue: [TSC #50](https://github.com/bitol-io/tsc/issues/50)

Applies to:
* [x] ODCS - Open Data Contract Standard
* [x] ODPS - Open Data Product Standard
* [x] OORS - Open Observability Results Standard
* [x] OOCS - Open Orchestration and Control Standard
* [x] OMMS - Open Maturity Model Standard
* [x] OMDS - Open Metadata Difference Standard

> [!NOTE]
> This RFC supersedes [RFC-0036: Environment Variables in Servers](archive/0036-environment-variables.md). It keeps a single idea — `${VAR_NAME}` interpolation — and drops everything else RFC-0036 explored (the `$var` annotation alternative, a `variables` declaration block, and parameterized `$import`).

## Summary

This RFC standardizes a single, minimal mechanism: any string value in an ODCS data contract MAY contain `${VAR_NAME}` references that are resolved at runtime by tooling. This lets contract authors keep secrets and environment-specific values out of the contract itself, without adding any new section or field to the standard.

## Motivation

Data contracts contain values that should not be hardcoded — credentials, hostnames, ports, bucket paths, and other environment-specific settings:

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

This causes two problems:

1. **Secrets land in the contract.** Passwords and tokens in plain text make contracts unsafe to store in version control, share with consumers, or move through CI/CD.
2. **Operational values are frozen.** A hostname or bucket path change forces a contract edit — and potentially re-approval by consumers — for something that is not a semantic change to the contract.

A standard interpolation syntax lets every tool resolve these values the same way, rather than each implementor inventing its own substitution convention. This aligns with Bitol's guiding value of **favoring interoperability**.

## Design and examples

Any string value anywhere in a data contract MAY contain one or more variable references of the form `${VAR_NAME}`. References MAY appear as a whole value or as a substring (for example, `jdbc:postgresql://${DB_HOST}:${DB_PORT}/orders`).

`VAR_NAME` is an identifier chosen by the contract author. This RFC does not constrain the naming convention beyond requiring that it be a valid string within `${...}`.

Resolution is intentionally left to tooling. Common sources include OS environment variables, `.env` files, Vault, AWS Secrets Manager, Kubernetes secrets, and CI/CD pipeline variables.

### Example 1: Server connection

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

### Example 2: Substring interpolation

```yaml
servers:
  - server: prod_jdbc
    environment: prod
    type: jdbc
    connection: "jdbc:postgresql://${DB_HOST}:${DB_PORT}/${DB_NAME}"
```

### Example 3: Outside `servers`

A variable reference is valid in any string value, not just `servers`. For example, in a quality rule query or an SLA threshold:

```yaml
quality:
  - type: sql
    query: "SELECT COUNT(*) FROM ${TARGET_TABLE} WHERE created_at > ${CUTOFF_DATE}"
```

### Tooling behavior (normative)

- Tools MUST resolve `${VAR_NAME}` references before using the value for any purpose.
- If a referenced variable cannot be resolved, tools SHOULD surface an error and MUST NOT silently substitute an empty string.
- Tools MUST preserve unresolved `${VAR_NAME}` tokens verbatim when serializing a contract back to YAML (round-trip safety).
- Tools MAY define their own resolution order across sources (for example, OS environment variable before `.env` file).

## Open decision: default values

A reference can optionally carry an inline default, so that a value is used when the variable is unset or empty. This avoids re-introducing a separate declaration block just to express defaults. The TSC must choose between two options:

**Option A — no default (bare references only).** A reference is exactly `${VAR_NAME}`. If the variable cannot be resolved, tooling errors (per the normative rules above). Smallest possible surface.

**Option B — adopt `${VAR_NAME:-default}`.** In addition to bare references, a reference MAY supply a default after `:-`. The text between `:-` and the closing `}` is the default value, used verbatim when the variable is unset or empty.

```yaml
servers:
  - server: prod_db
    environment: prod
    type: postgresql
    host: ${DB_HOST:-localhost}
    port: ${DB_PORT:-5432}
    database: orders_prod
    password: ${DB_PASSWORD}
```

The `${VAR:-default}` form is not invented here: it is POSIX shell parameter expansion (IEEE Std 1003.1, Shell Command Language, §2.6.2 "Parameter Expansion"), and is implemented identically by Docker Compose variable interpolation. This RFC would adopt **only** the `:-` form; the other POSIX operators (`-`, `:?`, `:+`) are out of scope.

If Option B is adopted, two normative points apply:

- A reference with a resolvable default MUST NOT error when the variable is unset; the default is used instead.
- The round-trip rule still holds: a tool that has not resolved a reference MUST preserve the full `${VAR_NAME:-default}` token verbatim, including the default.

Source: [The Open Group Base Specifications Issue 7, IEEE Std 1003.1-2017, §2.6.2 Parameter Expansion](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/V3_chap02.html#tag_18_06_02).

## Decision

TBD — to be put to a vote at a TSC meeting. The TSC must choose **Option A** (bare `${VAR_NAME}` only) or **Option B** (also adopt the POSIX `${VAR_NAME:-default}` form) from [Open decision: default values](#open-decision-default-values).

## Consequences

### Positive

- Contracts can be stored in version control without exposing credentials.
- One contract works across environments by supplying different variable values.
- Operational values can change without consumer re-approval.
- Minimal standard surface: one interpolation syntax, no new section or field.

### Negative

- Contracts with unresolved `${VAR_NAME}` references are not self-contained; consumers need access to a variable source to use them.
- There is no in-contract signal for which variables are required or sensitive. That responsibility lives in tooling unless a future RFC adds a declaration block.

### Neutral

- Resolution order and priority are left to tooling.

## References

- [RFC-0036: Environment Variables in Servers](archive/0036-environment-variables.md) — the superseded, broader proposal this RFC simplifies
- [RFC-0001: Servers](approved/odcs-v3.0.0/0001-servers.md) — establishes the `servers` section
- [Docker Compose variable substitution](https://docs.docker.com/compose/how-tos/environment-variables/variable-interpolation/) — prior art for `${VAR}` syntax
- [The Twelve-Factor App — Config](https://12factor.net/config) — storing config in the environment, not in code
