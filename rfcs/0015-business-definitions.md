# Business Definitions

Champions: Martin, Simon

Status: VERY EARLY WIP

## Example


### Option A: Business Definitions as separate entity

```yaml
version: "1.0.0"
apiVersion: "v3.0.1"
id: "6aeafdc1-ed62-4c8f-bf0a-da1061c98cdb"
kind: "BusinessDefinition"
domain: "orders"

name: "departmentid"
logicalType: "number"
description: "Primary key for Department records."
criticalDataElement: false
---
version: "1.0.0"
apiVersion: "v3.0.1"
id: "6aeafdc1-ed62-4c8f-bf0a-da1061c98cdc"
kind: "BusinessDefinition"

domain: "fulfillment"
name: "employee"
logicalType: "object"
properties:
  - name: "employeeid"
    logicalType: "number"
    description: "Primary key for Employee records."
  - name: "name"
    logicalType: "string"
    description: "Name of the employee."
```

### Option B: Business Definitions as a data contract

```yaml
version: "1.0.0"
apiVersion: "v3.0.1"
id: "6aeafdc1-ed62-4c8f-bf0a-da1061c98cdb"
kind: "DataContract"
schema:  
  - name: "department"
    description: "Lookup table containing the departments within the Adventure Works\
      \ Cycles company."
    logicalType: "object"
    properties:
    - name: "departmentid"
      logicalType: "number"
      description: "Primary key for Department records."
      criticalDataElement: false
    - name: "name"
      logicalType: "string"
      description: "Name of the department"
      criticalDataElement: false
    - name: "addresses"
      logicalType: "array"
      description: "Name of the department"
      criticalDataElement: false
      items:
        logicalType: "object"
        description: "address of a department"
        properties:
          - name: street
            logicalType: "string"
            description: "Street of the address"
          - name: zipcode
            logicalType: "string"
            description: "Zipcode of the address"
  - name: "employee"
    logicalType: "object"
    properties:
    - name: "employeeid"
      logicalType: "number"
      description: "Primary key for Employee records."
    - name: "departmentid"
      logicalType: "number"
      description: "Foreign key to the department that the employee belongs to."
    - name: "name"
      logicalType: "string"
      description: "Name of the employee."
    - name: "jobtitle"
      logicalType: "string"
      description: "Job title of the employee."
    - name: "birthdate"
      logicalType: "date"
      description: "Birthdate of the employee."
    
```
