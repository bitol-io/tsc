
# Lineage

Champion: Dirk Van de Poel

## Summary

This RFC proposes to add data lineage information to the data contract by adding references to upstream data contracts.

## Motivation

Lineage information collection and reporting is quite common and data contracts are well suited to express upstream dependencies. 
This allows for verifying downstream impacted data contracts (hence data products) on data contract changes.
It also enables linking data contracts to existing (fine-grained) operational data linesage information to create a complete view on the data processing pipeline(s).
The objective is to express course-grained data contract level dependencies and not add additional fine-grained operational details on the processing logic.


## Design and examples

In most cases data product developers will have a view on which data is ingested and processed in order to deliver the data product according to the data contract, but they will now have a view on the data consumers that may use the data today or in the future.
That is why the focus of adding lineage information to ODCS is on listing upstream dependencies, input datasets that the data product relies on.
When all data contracts in the pipeline express their upstream dependencies, there will be an end-to-end view on the data product level lineage.

Currently the ODCS specification (v2.3.0) defines dataset.table.columns.column.transformSourceTables to reference upstream depedencies but this has a number of limitations:
1. It provides fine-grained (column level) lineage with may not be required or possible in all cases, this proposal focusses on supporting course-grained lineage in between data contracts
2. It applies exclusively to table output datasets that are the result of transformations of table input datasets, hence it does not apply to non tabular data
3. It references by table name without an indication in which domain/namepace database this table can be found, unless table names are globally unique it does not express dependencies across data products

### Design

This proposal aims to be compatible with OpenLineage and focusses on so-called design lineage.  OpenLineage focsuses on operational lineage aimed at tracebility of each data processing job execution, this is more fine-grained and provides data processing pipeline information on each processing step also within a data product.
Adding lineage information at the data contract level is complementary to the OpenLineage lineage information, it extends the existing OpenLineage information with the formal data contract information for the data product output datasets.

In order to provide product level lineage, a table of references is added to reference all input data contracts as upstream dependencies.
In addition to referencing upstream data contracts, a link is added to the output datasets of this Data Contract in order to be able to correlate this to OpenLineage events.

| Key                              | UX label                 | Required | Description                                                                                                                                                             |
|----------------------------------|--------------------------|----------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| lineage                          | Object                   | No       | Object. Data lineage related properties.                                                                                                                                |                  
| inputDataContracts               | Input Data Contracts     | No       | Array. A list of input data contracts on which this data contract hence data product depends.                                                                           |                   
| inputDataContracts.UUID          | Input Data Contract UUID | No       | UUID of the referenced Data Contract that defines the datasets that servce as input for this Data Product.                                                              |
| outputDatasets                   | Output Datasets          | No       | Array. A list of Data Contract output datasets to link the Data Contract to OpenLineage for operational data lineage and data pipeline tracebility.                     |
| outputDatasets.namespace         | Output Dataset namespace | No       | Dataset namespace according to OpenLinedage naming convention: https://openlineage.io/docs/spec/naming/, links Data Contrac to OpenLineage Dataset.                     |                   
| outputDatasets.name              | Output Dataset name      | No       | Dataset name according to OpenLinedage naming convention: https://openlineage.io/docs/spec/naming/, links Data Contrac to OpenLineage Dataset.                          |                                                                      

                                                                    
In line ith the guiding value of keeping the standard small, if the proposal to add data contract level lineage information is accepted, it is also proposed to deprecate the former dataset.table.columns.column.transformSourceTables. 

### Example

![Data Products Example](https://klarr.io/wp-content/uploads/odcs-lineage-v2.png)

The picture above illustrates an example topology with a number of data products (Data Product), their data contracts (DC), data processing jobs (Job) and data sets (DS) in de pipeline(s).

Data Product Z has a single Data Contract (DC Z), internally it leverages three jobs (Jobs D, E & F) to ingest and transform data from Data Product Y and an external API (e.g. SaaS REST).
The three processing jobs emit OpenLineage events for each run referencing their input and output datasets.
Job F uses dataset DSZ1 as input and DSZ2 as output, Job D uses the external API as input and DSZ1 as output while Job E uses DSY1 as input and DSZ1 as output.

Data Contract Z (DC Z) adds a reference to the upstream Data Contract Y (DC Y) it depends on.
This will provide high level lineage information across the pipeline so it because visible and changes to DC Y will have an impact on Data Product Z and its Data Contract DC Z.

In addition, providing the optional OpenLineage namespace and name for the output dataset which the Data Contract covers, enables using OpenLineage for tracing the Data Product internal pipeline including the ingest from the external API.
This output datasets information is optional and not all Data Contracts are expected to have this information.

```YAML
# Data Contract Z
quantumName: Data Product Z 
version: 1.1.0 
uuid: 3c17473d-94c3-4456-98ad-81eb316df788
lineage:
  inputDataContracts:
    - UUID: 789ecec3-69c7-4d0e-9038-6cb97965e679
  outputDatasets:
    - name: DSZ2-name # output dataset name corresponding to OpenLineage event output dataset emitted by Job F
      namespace: DSZ2-namespace # output dataset namespace corresponding to OpenLineage event output dataset emitted by Job F

# Data Contract Y
quantumName: Data Product Y
version: 2.0.0 
uuid: 789ecec3-69c7-4d0e-9038-6cb97965e679
lineage:
  inputDataContracts:
    - UUID: 251f4798-63e9-457d-aa57-d14e61f1ab40
    - UUID: 512e9887-9fec-43f2-afe6-d8d7d891a2ff

# Data Contract X
quantumName: Data Product X
version: 1.8.0 
uuid: 251f4798-63e9-457d-aa57-d14e61f1ab40

# Data Contract W1
quantumName: Data Product W
version: 3.0.7 
uuid: eb640198-cb5e-4620-aca7-fd2f303fbb77

# Data Contract W2
quantumName: Data Product W
version: 3.0.4 
uuid: 512e9887-9fec-43f2-afe6-d8d7d891a2ff

```

## Alternatives

### Alternative 1:

Locate the link between the Data Contract dataset and OpenLineage under the dataset schema definition.
This depends on the outcome of rfc-0004 hierchical dataset structure and naming.

PRO:
- More accurate relation in between Data Contract and OpenLineage datasets

CON:
- It may not be required in most cases and spreads this information over the hierchical (nested) structure hence adds complexity

### Alternative 2:

Define an alternative way or referencing external (other) Data Contracts, e.g. based upon the outcome of rfc-0009 External & Internal References.

PRO:
- Uniform way or referencing other data contracts

CON:
- Given that a data lineage tool will most likely ingest all data contracts, matching by unique identifier probably is the simplest

### Alternative 3:

Define more fine-grained upstream references.

PRO:
- More meaningful lineage information in between specific datasets (e.g. column level lineage for tabular data)

CON:
- The main reason for adding lineage information to data contracts is to identify downstream impact on data contract changes, fine-grained lineage information is not required for this
- Overlap with OpenLineage or other operational lineage solutions


### Alternative 4: 

Define both upstream and downstream references.

PRO:
- When not all data contract contain lineage information there is a higher chance of covering all pipelin dependencies

CON:
- Data product developer are likely not aware of all current and future data consumers (downstream data products)

### Alternative 5:

Only add a mapping in between ODCS and OpenLineage output datasets and completely rely upon OpenLinage for dependencies. 

PRO:
- Leverage the existing OpenLineage definition

CON:
- Not all data contract users use a data lineage solution

## Decision

> The decision made by the TSC.

## Consequences

> The consequences of this decision.

## References

### OpenLineage

https://openlineage.io/

OpenLineage is an Open Standard for lineage metadata collection designed to record metadata for a job in execution.
It is a standard API for capturing lineage events. Pipeline components - like schedulers, warehouses, analysis tools, and SQL engines - can use this API to send data about runs, jobs, and datasets to a compatible OpenLineage backend for further study.

The main job update information that is published is on:
* Run 
* Job
* Dataset

Example OpenLineage event:

```json
	{
	  "eventType": "START",
	  "eventTime": "2020-12-28T19:52:00.001+10:00",
	  "run": {
		"runId": "d46e465b-d358-4d32-83d4-df660ff614dd"
	  },
	  "job": {
		"namespace": "workshop",
		"name": "process_taxes"
	  },
	  "inputs": [{
		"namespace": "postgres://workshop-db:None",
		"name": "workshop.public.taxes"
	  }],  
	  "producer": "https://github.com/OpenLineage/OpenLineage/blob/v1-0-0/client"
	}
	
	{
	  "eventType": "COMPLETE",
	  "eventTime": "2020-12-28T20:52:00.001+10:00",
	  "run": {
		"runId": "d46e465b-d358-4d32-83d4-df660ff614dd"
	  },
	  "job": {
		"namespace": "workshop",
		"name": "process_taxes"
	  },
	  "outputs": [{
		"namespace": "postgres://workshop-db:None",
		"name": "workshop.public.unpaid_taxes"
	  }],     
	  "producer": "https://github.com/OpenLineage/OpenLineage/blob/v1-0-0/client"
	}	
```
### Egeria

https://egeria-project.org/features/lineage-management/overview/

Egeria embraces the OpenLineage standard.  THe project defines Design Lineage vs. Opertaional Lineage.

__Design lineage__ describes all the digital resources and their linkages. Some tools, such as ETL engines, produce design lineage in their tools as part of their design process. Other technologies rely on design lineage captured in the dev-ops pipeline or the automatic cataloguing of digital resources as they are added to the pre-production or production environment.

__Operational lineage__ is the lineage information produced by a data processing engine when it runs processes. It enables an organization to validate that processes run at the right time, using the right data and produce the right results. It primarily focuses on capturing the dynamic aspects of lineage, but may also identify parts of the digital landscape that have not yet been catalogued.

