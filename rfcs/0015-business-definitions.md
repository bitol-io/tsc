# Business Definitions

Champions: Martin, Simon, jgp

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
apiVersion: "v3.1.0"
id: "6aeafdc1-ed62-4c8f-bf0a-da1061c98cdb"
kind: "DataContract"
schema:  
  - name: "department"
    id: department
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


### Option C: Business Definitions as a data contract using id fields
No changes to the standard itself required . Basic idea is that the defintions are abstracted in the sense that these do not mimic the data structure. Instead these are structured using logical objects. The defintions are referenced using the id fields introduced in v3.1.0. Note that the names are unsuitable for technical implementations but are human readable and self explanatory to the greatest extent given their parent object. Hence, these could be directly employed as labels in reports or in data catalogs.

```yaml
apiVersion: v3.1.0
kind: DataContract
name: Business Definitions Departments
version: 1.0.0
id: 6aeafdc1-ed62-4c8f-bf0a-da1061c98cdb

schema:
  ## Schema object for business defintions for department data
  - name: Department Data
    id: department
    description: "Business defintions for department data within the Adventure Works Cycles company."
    logicalType: object
    properties:
    - name: Id
      id: id
      description: "Primary key for Department records."
      logicalType: number
      criticalDataElement: false
      classification: 00_GDPR_none
    - name: Name
      id: name
      description: "Name of the department"
      logicalType: string
      criticalDataElement: false
      classification: 00_GDPR_none
    - name: Street and Number
      id: street_no
      logicalType: string
      description: "Street and house number of the department"
      criticalDataElement: false
      classification: 00_GDPR_none
    - name: ZIP Code
      id: zipcode
      logicalType: string
      description: "Zipcode of of the department"
      criticalDataElement: false
      classification: 00_GDPR_none
    - name: City
      id: city
      logicalType: string
      description: "Zipcode of of the department"
      criticalDataElement: false
      classification: 00_GDPR_none

  ## Schema object for business defintions for employee data
  - name: Employee
    id: employee
    description: "Business defintions for employee data in  the Adventure Works Cycles company."
    logicalType: object
    properties:
    - name: Id
      id: id
      logicalType: number
      description: "Primary key for Employee records."
      classification: 01_GDPR_indirect
    - name: departmentid
      id: departmentid
      logicalType: number
      description: "Foreign key to the department that the employee belongs to."
      classification: 00_GDPR_none
    - name: Name
      id: name
      logicalType: string
      description: "Name of the employee."
      classification: 02_GDPR_direct
    - name: Job Title
      id: jobtitle
      logicalType: string
      description: "Job title of the employee."
      classification: 02_GDPR_direct
    - name: Birthdate
      id: birthdate
      logicalType: date
      description: "Birthdate of the employee."
      classification: 02_GDPR_direct
    - name: IBAN
      id: iban
      logicalType: string
      description: "IBAN for salary transfer"
      classification: 03_GDPR_critical
    
```
