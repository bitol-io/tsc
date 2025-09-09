# Title

> The champion is the person who is primarily supporting the RFC.

Champion: Vishwas Balakrishna, Gene Azad

[Slack](https://data-mesh-learning.slack.com/team/U09APEKU21J)

Jira: *replace this text with the link to the dedicated Jira ticket*.

## Summary

> Both the Open Data Contract Standard (ODCS) and Open Data Product Standard (ODPS) provide frameworks for defining and managing data quality, including support for unstructured data. While both standards were originally designed for structured data, there is a need to accommodate the growing importance of unstructured data in modern data ecosystems.

### ODCS Enhancements for Unstructured Data

- Enhanced schema structure that now supports not only relational databases but also hierarchical and complex data storage, streaming messages, and unstructured data
- Expanded infrastructure definitions that better capture metadata around systems, applications, and platforms handling unstructured data
- Improved data quality support compatible with virtually any data quality tool, making it easier to integrate unstructured data quality frameworks

### ODPS Ehnacements Unstructured Data Features

- Machine-readable metadata definitions that can describe unstructured data products
- Flexible data quality profiles that can be applied to various data types including text, documents, and multimedia
- Everything-as-Code philosophy enabling automated quality validation for unstructured content


## Motivation

> Why are we doing this? What use cases does it support? What is the expected outcome?

- Unstructured data (e.g., text, documents, multimedia) now constitutes the majority of enterprise data, but traditional data quality frameworks often fall short in addressing it.
    - We are doing this RFC to:
        - Extend the use of ODCS and ODPS for unstructured data.
        - Provide consistency in how unstructured data quality is defined, validated, and monitored.
        - Enable automation of quality checks across different unstructured data types.

    - Expected outcomes:
        - Shared definitions of unstructured data quality dimensions (accuracy, completeness, relevance, etc.)
        - Reusable quality profiles and schemas that align with ODCS/ODPS standards.
        - Improved governance and observability of unstructured data quality.

> How does it align with our guiding values?
    - This aligns with to guiding values of interoperability, automation ("everything-as-code"), and trust in data assets

## Design and examples

### Defining Unstructured Data Quality Dimensions 
Core Quality Dimensions for Unstructured Data
Based on research and industry practices, both ODCS and ODPS can leverage established unstructured data quality dimensions:

Primary Dimensions:
- Accuracy: Correctness of the unstructured content compared to real-world entities
- Completeness: Whether all necessary unstructured data elements are present
- Consistency: Uniformity of format and content across unstructured data sources
- Relevance: How pertinent the unstructured data is to specific tasks or applications
- Timeliness: Currency and freshness of the unstructured content
- Interpretability: How well the data can be understood by both human and machine consumers

**Additional Dimensions:**
- Validity: Adherence to expected formats and standards for unstructured content
- Uniqueness: Absence of duplicate or redundant unstructured data
- Conformity: Alignment with required standards, syntax, or domain values


### Implementation Approaches


### Using ODCS for Unstructured Data Quality
#### 1. Schema Definition for Unstructured Data

```yaml
schema:
- name: document_content
  logicalType: object
  physicalType: text
  description: Unstructured document content
  properties:
  - name: content_text
    logicalType: string
    physicalType: text
    description: Raw text content of documents
    quality:
    - type: library
      rule: completeness
      mustBeGreaterThan: 95
      unit: percent
    - type: custom
      engine: textQuality
      implementation: |
        readability_score: flesch_kincaid
        min_score: 6.0
        max_score: 12.0
```
#### 2. Data Quality Rules for Text Content
```yaml
quality:
- type: text
  description: Content must be relevant to business domain and free of PII
- type: library
  rule: accuracy
  objective: 90
  unit: percent
- type: sql
  query: |
    SELECT COUNT(*) FROM ${object}
    WHERE LENGTH(${property}) > 0 AND ${property} IS NOT NULL
  mustBeGreaterThan: 95
- type: custom
  engine: NLP
  implementation: |
    checks:
      - sentiment_coherence
      - language_detection
      - topic_relevance

```

### Using ODPS for Unstructured Data Quality

#### 1. Data Quality Profiles for Unstructured Content

```yaml
dataQuality:
  declarative:
    default:
      displaytitle:
        en: Basic Unstructured Data Quality
      dimensions:
      - dimension: accuracy
        displaytitle:
          en: Content Accuracy (percent)
        objective: 85
        unit: percentage
      - dimension: completeness
        displaytitle:
          en: Document Completeness (percent)
        objective: 90
        unit: percentage
      - dimension: validity
        displaytitle:
          en: Format Validity (percent)
        objective: 95
        unit: percentage

  executable:
  - dimension: accuracy
    type: Custom
    reference: 'https://docs.textquality.org/'
    spec:
      language_model: en_core_web_sm
      accuracy_checks:
      - spell_check
      - grammar_check
      - fact_verification

  - dimension: completeness
    type: SodaCL
    spec: |
      checks for document_table:
        - missing_count(content_field) = 0
        - invalid_count(content_field) < 100
```
#### 2. Pricing Plans with Quality Tiers



## Alternatives

> Rejected alternative solutions and the reasons why.

## Decision

> The decision made by the TSC.

## Consequences

> The consequences of this decision.

## References

> Prior art, inspiration, and other references you used to create this based on what's worked well before.
