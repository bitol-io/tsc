# Named objects

Champion: Andrea Gioia

Slack: NA

Jira: NA

## Summary

This RFC proposes standardizing the representation of collections of named sub-entities as objects, where each key corresponds to the sub-entity's name and the value is the sub-entity itself. 

## Motivation

> Why are we doing this? What use cases does it support? What is the expected outcome?


This approach is favored over using arrays of sub-entities with internal name properties. Using named-object mappings improves clarity, enables more efficient lookups, simplifies validation, and avoids redundancy by leveraging the object structure to encode identity. More in detail, the expected outcomes are: 

1. **Improved Clarity:** Object keys immediately identify the entity
1. **Faster Lookups:** Direct access via key instead of array iteration
1. **Simplified Validation:** Object structure inherently validates uniqueness
1. **Reduced Redundancy:** Eliminates need for internal name properties
1. **Better Developer Experience:** More intuitive API for accessing specific entities

> How does it align with our guiding values?
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

The following example shoes how the previous yaml block will look like after refactoring...

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

Each refactored collection should include:
- `additionalProperties: false` to prevent arbitrary keys
- Appropriate `patternProperties` to validate key naming conventions
- Updated descriptions reflecting the named-object approach


### Recommended Refactorings

From the analysis of the [JSON Schema](https://github.com/bitol-io/open-data-contract-standard/blob/main/schema/odcs-json-schema-v3.0.2.json) of version 3.0.2 the following are the sections impacted by this RFC

#### 1. Servers Collection

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

**Impact:** Server configurations can be accessed directly by name (e.g., `servers.production`, `servers.staging`) instead of searching through an array.

#### 2. Schema Collection

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

**Impact:** Schema objects can be referenced directly by name, improving readability and reducing lookup complexity.

#### 3. Team Collection

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

**Impact:** Team members can be accessed by username/identifier, eliminating the need to iterate through arrays to find specific team members.

#### 4. Roles Collection

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

**Impact:** Roles can be referenced directly by name, simplifying access control management and role lookups.

#### 5. SLA Properties Collection

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

**Impact:** SLA properties can be accessed by property name, making SLA management more intuitive and efficient.

#### 6. Schema Properties within Objects

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

**Impact:** Object properties can be accessed directly by property name, aligning with standard JSON Schema patterns.

#### 7. Data Quality Checks Collection

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

**Impact:** Quality checks can be referenced by name, improving maintainability and making it easier to manage specific quality rules.

#### 8. Support Channels Collection

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

**Impact:** Support channels can be accessed by channel name, making support configuration more organized and accessible.


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

> Prior art, inspiration, and other references you used to create this based on what's worked well before.
