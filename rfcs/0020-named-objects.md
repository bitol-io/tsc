# Named objects

Champion: Andrea Gioia

Slack: NA

Jira: NA

## Summary

This RFC proposes standardizing the representation of collections of named sub-entities as objects, where each key corresponds to the sub-entity's name and the value is the sub-entity itself. This approach is favored over using arrays of sub-entities with internal name properties.

## Motivation

Using named-object mappings instead of arrays improves clarity, enables more efficient lookups, simplifies validation, and avoids redundancy by leveraging the object structure to encode identity. 

The expected outcomes are: 

1. **Improved Clarity:** Object keys immediately identify the entity
1. **Faster Lookups:** Direct access via key instead of array iteration
1. **Simplified Validation:** Object structure inherently validates uniqueness
1. **Reduced Redundancy:** Eliminates need for internal name properties
1. **Better Developer Experience:** More intuitive API for accessing specific entities (see [RFC-0009](https://github.com/bitol-io/tsc/blob/main/rfcs/0009-external-internal-reference.md))

This refactoring aligns the ODCS with modern JSON Schema best practices and significantly improves the developer experience when working with data contracts.

## Design and examples
This section outlines recommended refactorings for the Open Data Contract Standard JSON Schema to align with the RFC proposal for standardizing collections of named sub-entities as objects rather than arrays.

### Simple example
Access roles are now listed using an array. The name of each role is specified using the property `role` as shown in the example

```yaml
roles:
  - role: microstrategy_user_opr
    access: read
    firstLevelApprovers: Reporting Manager
    secondLevelApprovers: 'mandolorian'
  - role: bq_queryman_user_opr
    access: read
    firstLevelApprovers: Reporting Manager
    secondLevelApprovers: na
  - role: risk_data_access_opr
    access: read
    firstLevelApprovers: Reporting Manager
    secondLevelApprovers: 'dathvador'
  - role: bq_unica_user_opr
    access: write
    firstLevelApprovers: Reporting Manager
    secondLevelApprovers: 'mickey'
```

The following example shows how the previous YAML block will look after refactoring...

```yaml
roles:
  microstrategy_user_opr:
    access: read
    firstLevelApprovers: Reporting Manager
    secondLevelApprovers: 'mandolorian'
  bq_queryman_user_opr:
    access: read
    firstLevelApprovers: Reporting Manager
    secondLevelApprovers: na
  risk_data_access_opr:
    access: read
    firstLevelApprovers: Reporting Manager
    secondLevelApprovers: 'dathvador'
  bq_unica_user_opr:
    access: write
    firstLevelApprovers: Reporting Manager
    secondLevelApprovers: 'mickey'
```


### Recommended Refactorings

From the analysis of the [JSON Schema](https://github.com/bitol-io/open-data-contract-standard/blob/main/schema/odcs-json-schema-v3.0.2.json) of ODCS-v3.0.2, the following are the sections impacted by this RFC

#### 1. Schema Object Collection

This [section](https://bitol-io.github.io/open-data-contract-standard/latest/#schema) describes the schema of the data contract.

**Current Implementation:**
```json
"schema": {
  "type": "array",
  "description": "A list of elements within the schema to be cataloged.",
  "items": {
    "$ref": "#/$defs/SchemaObject"
  }
}
```

**Recommended Refactor:**
```json
"schema": {
  "type": "object",
  "description": "Named collection of schema elements to be cataloged.",
  "patternProperties": {
    "^[a-zA-Z0-9_-]+$": {
      "$ref": "#/$defs/SchemaObject"
    }
  },
  "additionalProperties": false
}
```

**Impact:** Schema objects can be accessed directly by name instead of searching through an array. The name of the schema object is equal to the property `name` 

#### 2. Elements within a Schema Object

**Current Implementation:**
```json
"properties": {
  "type": "array",
  "description": "A list of properties for the object.",
  "items": {
    "$ref": "#/$defs/SchemaProperty"
  }
}
```

**Recommended Refactor:**
```json
"properties": {
  "type": "object",
  "description": "Named collection of properties for the object.",
  "patternProperties": {
    "^[a-zA-Z0-9_-]+$": {
      "$ref": "#/$defs/SchemaProperty"
    }
  },
  "additionalProperties": false
}
```

**Impact:** Object properties can be accessed directly by property name, aligning with standard JSON Schema patterns. The name of the object schema element is equal to the property `name` 

#### 3. Servers Collection

This [section](https://bitol-io.github.io/open-data-contract-standard/latest/#infrastructure-and-servers) describes where the data protected by this data contract is physically located.

**Current Implementation:**
```json
"servers": {
  "type": "array",
  "description": "List of servers where the datasets reside.",
  "items": {
    "$ref": "#/$defs/Server"
  }
}
```

**Recommended Refactor:**
```json
"servers": {
  "type": "object",
  "description": "Named collection of servers where the datasets reside.",
  "patternProperties": {
    "^[a-zA-Z0-9_-]+$": {
      "$ref": "#/$defs/Server"
    }
  },
  "additionalProperties": false
}
```

**Impact:** Server configurations can be accessed directly by name instead of searching through an array. The name of the server is equal to the property `server` 



#### 4. Team Collection

This [section](https://bitol-io.github.io/open-data-contract-standard/latest/#team) lists team members and the history of their relationship with this data contract

**Current Implementation:**
```json
"team": {
  "type": "array",
  "items": {
    "$ref": "#/$defs/Team"
  }
}
```

**Recommended Refactor:**
```json
"team": {
  "type": "object",
  "description": "Named collection of team members.",
  "patternProperties": {
    "^[a-zA-Z0-9_@.-]+$": {
      "$ref": "#/$defs/Team"
    }
  },
  "additionalProperties": false
}
```

**Impact:** Team members can be accessed directly by name instead of searching through an array. The name of the team member is equal to the property `username` 

#### 5. Roles Collection

This [section](https://bitol-io.github.io/open-data-contract-standard/latest/#roles) lists the roles that a consumer may need to access the dataset, depending on the type of access they require.

**Current Implementation:**
```json
"roles": {
  "type": "array",
  "description": "A list of roles that will provide user access to the dataset.",
  "items": {
    "$ref": "#/$defs/Role"
  }
}
```

**Recommended Refactor:**
```json
"roles": {
  "type": "object",
  "description": "Named collection of roles that provide user access to the dataset.",
  "patternProperties": {
    "^[a-zA-Z0-9_-]+$": {
      "$ref": "#/$defs/Role"
    }
  },
  "additionalProperties": false
}
```

**Impact:** Roles can be accessed directly by name instead of searching through an array. The name of the role is equal to the property `role` 

#### 6. SLA Properties Collection

This [section](https://bitol-io.github.io/open-data-contract-standard/latest/#service-level-agreement-sla) describes the service-level agreements (SLA).

**Current Implementation:**
```json
"slaProperties": {
  "type": "array",
  "description": "A list of key/value pairs for SLA specific properties.",
  "items": {
    "$ref": "#/$defs/ServiceLevelAgreementProperty"
  }
}
```

**Recommended Refactor:**
```json
"slaProperties": {
  "type": "object",
  "description": "Named collection of SLA specific properties.",
  "patternProperties": {
    "^[a-zA-Z0-9_-]+$": {
      "$ref": "#/$defs/ServiceLevelAgreementProperty"
    }
  },
  "additionalProperties": false
}
```

**Impact:** SLA properties can be accessed directly by name instead of searching through an array. The name of the SLA properties object is equal to the property `property` 

#### 7. Data Quality Checks Collection

This [section](https://bitol-io.github.io/open-data-contract-standard/latest/#data-quality) describes data quality rules & parameters.

**Current Implementation:**
```json
"DataQualityChecks": {
  "type": "array",
  "description": "Data quality rules with all the relevant information for rule setup and execution.",
  "items": {
    "$ref": "#/$defs/DataQuality"
  }
}
```

**Recommended Refactor:**
```json
"DataQualityChecks": {
  "type": "object",
  "description": "Named collection of data quality rules for setup and execution.",
  "patternProperties": {
    "^[a-zA-Z0-9_-]+$": {
      "$ref": "#/$defs/DataQuality"
    }
  },
  "additionalProperties": false
}
```

**Impact:** Data Quality Checks can be accessed directly by name instead of searching through an array. The name of the Data Quality Check is equal to the property `rule` 

#### 8. Support Channels Collection

This [section](https://bitol-io.github.io/open-data-contract-standard/latest/#support-and-communication-channels) describes support and communication channels that can be used by consumers to find help regarding their use of the data contract.

**Current Implementation:**
```json
"Support": {
  "type": "array",
  "description": "Top level for support channels.",
  "items": {
    "$ref": "#/$defs/SupportItem"
  }
}
```

**Recommended Refactor:**
```json
"Support": {
  "type": "object",
  "description": "Named collection of support channels.",
  "patternProperties": {
    "^[a-zA-Z0-9_-]+$": {
      "$ref": "#/$defs/SupportItem"
    }
  },
  "additionalProperties": false
}
```

**Impact:** Channels can be accessed directly by name instead of searching through an array. The name of the Channel is equal to the property `channel` 


## Alternatives

> Rejected alternative solutions and the reasons why.

## Decision

> The decision made by the TSC.

## Consequences

These RFC represent breaking changes that will require:

1. **Data Contract Updates**: Existing data contracts using array structures will need to be migrated
2. **Tooling Updates**: Any tools that parse ODCS contracts will need to be updated
3. **Documentation Updates**: All examples and documentation will need to reflect the new structure

## References

OpenAPI and AsyncAPI follow the approach proposed in this RFC.
