# MCP server type

Champion: Jean-Georges Perrin

Authors: Jean-Georges Perrin

GitHub issue: https://github.com/bitol-io/tsc/issues/133

Applies to:
* [x] ODCS - Open Data Contract Standard
* [ ] ODPS - Open Data Product Standard
* [ ] OORS - Open Observability Results Standard
* [ ] OOCS - Open Orchestration and Control Standard
* [ ] OMMS - Open Maturity Model Standard
* [ ] OMDS - Open Metadata Difference Standard
* [ ] OSDS - Open Semantic Definition Standard

## Summary

Add `mcp` as a server `type` to describe data served through a Model Context Protocol (MCP) server.

## Motivation

AI agents increasingly read data through MCP servers rather than through a database or an API directly. ODCS has no server type for them, so users fall back to `custom` and put the transport, command or URL in `customProperties`, where nothing is validated. A first-class `mcp` type lets tooling connect to the server, and lets a contract say which of the server's tools serve its data.

## Design and examples

Add `mcp` to the list of allowed server `type` values, with the following fields:

| Field             | Type             | Required        | Description                                                                                          |
| ----------------- | ---------------- | --------------- | ---------------------------------------------------------------------------------------------------- |
| `type`            | string           | Yes             | `mcp`                                                                                                |
| `transport`       | string           | Yes             | `stdio` (a local process) or `http` (Streamable HTTP).                                               |
| `url`             | string (uri)     | If `http`       | Endpoint of the server.                                                                              |
| `headers`         | map of strings   | No, `http` only | HTTP headers sent with every request.                                                                |
| `command`         | string           | If `stdio`      | Executable that starts the server.                                                                   |
| `args`            | array of strings | No, `stdio` only | Arguments passed to `command`.                                                                     |
| `env`             | map of strings   | No, `stdio` only | Environment variables set for the process.                                                         |
| `tools`           | array of strings | No              | Names of the server's tools that serve this contract's data. Absent means the contract does not restrict them. |
| `protocolVersion` | string           | No              | MCP protocol revision the server is known to support, e.g. `2025-06-18`.                             |

Credentials are never written in the contract. `headers` and `env` values that carry secrets use variable references (`${VAR_NAME}`, [RFC 0050](approved/odcs-v3.2.0/0050-variables.md)), resolved by tooling at runtime.

Minimal, a remote server:

```yaml
servers:
  - server: prod
    type: mcp
    transport: http
    url: https://mcp.example.com/mcp
```

Structured, a local server limited to two tools, with a secret resolved at runtime:

```yaml
servers:
  - server: local
    environment: dev
    type: mcp
    transport: stdio
    command: npx
    args: ["-y", "@example/orders-mcp"]
    env:
      ORDERS_API_TOKEN: ${ORDERS_API_TOKEN}
    tools: [list_orders, get_order]
    protocolVersion: "2025-06-18"
```

JSON schema, added to `ServerSource` and to the `type` enum, with the matching `if`/`then` in `Server`:

```json
"McpServer": {
  "type": "object",
  "title": "McpServer",
  "properties": {
    "transport": { "type": "string", "enum": ["stdio", "http"] },
    "url": { "type": "string", "format": "uri" },
    "headers": { "type": "object", "additionalProperties": { "type": "string" } },
    "command": { "type": "string" },
    "args": { "type": "array", "items": { "type": "string" } },
    "env": { "type": "object", "additionalProperties": { "type": "string" } },
    "tools": { "type": "array", "items": { "type": "string" }, "uniqueItems": true },
    "protocolVersion": { "type": "string" }
  },
  "required": ["transport"],
  "allOf": [
    { "if": { "properties": { "transport": { "const": "http" } } }, "then": { "required": ["url"] } },
    { "if": { "properties": { "transport": { "const": "stdio" } } }, "then": { "required": ["command"] } }
  ]
}
```

## Alternatives

- Keep using `custom` with `customProperties`. Rejected: no validation, and every tool invents its own keys.
- Use the `api` type. Rejected: `api` has a single `location` and no notion of transport, process or tools.
- Describe MCP servers in a separate standard. Rejected: an MCP server is a place data is served from, which is what `servers` describes.

## Decision

> The decision made by the TSC.

## Consequences

- Nonbreaking change: adds a new optional server type.

## References

- [Model Context Protocol specification](https://modelcontextprotocol.io/specification)
- [RFC 0050: Variables](approved/odcs-v3.2.0/0050-variables.md)
