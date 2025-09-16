# Unstructured Data Quality for ODCS & ODPS

> The champion is the person who is primarily supporting the RFC.

Authors: Vishwas Balakrishna, Gene Azad

Champion: Jean-Georges Perrin

[Slack](https://data-mesh-learning.slack.com/team/U09APEKU21J)

Jira: **

## Summary
> One paragraph explanation of the RFC.

Both the Open Data Contract Standard (ODCS) and Open Data Product Standard (ODPS) provide frameworks for defining and managing data quality. While both standards were originally designed for structured data, there is a need to accommodate the growing importance of unstructured data in modern data ecosystems.

<br> 

### ODCS Enhancements for Unstructured Data

- Enhanced schema structure that now supports not only relational databases but also hierarchical and complex data storage, streaming messages, and unstructured data
- Expanded infrastructure definitions that better capture metadata around systems, applications, and platforms handling unstructured data
- Improved data quality support compatible with virtually any data quality tool, making it easier to integrate unstructured data quality frameworks

<br> 

### ODPS Enhancements for Unstructured Data 

- Machine-readable metadata definitions that can describe unstructured data products
- Flexible data quality profiles that can be applied to various data types including text, documents, and multimedia
- Everything-as-Code philosophy enabling automated quality validation for unstructured content

<br> 

## Motivation

> Why are we doing this? What use cases does it support? What is the expected outcome? How does it align with our guiding values?


- Unstructured data (e.g., text, documents, multimedia) now constitutes the majority of enterprise data, but traditional data quality frameworks often fall short in addressing it.
    - We are doing this RFC to:
        - Extend the use of ODCS and ODPS for unstructured data.
        - Provide consistency in how unstructured data quality is defined, validated, and monitored.
        - Enable automation of quality checks across different unstructured data types.

    - Expected outcomes:
        - Shared definitions of unstructured data quality dimensions (accuracy, completeness, relevance, etc.)
        - Reusable quality profiles and schemas that align with ODCS/ODPS standards.
        - Improved governance and observability of unstructured data quality.


- This aligns with to guiding values of interoperability, automation ("everything-as-code"), and trust in data assets

<br> 

## Design and examples

> This is the bulk of the RFC. Explain the design in enough detail for somebody familiar with data contracts and the standard to understand. This should get into specifics and corner-cases, and include examples of how this is to be used. Offer at least two examples, one is minimalist, one is more structured.



### Defining Unstructured Data Quality Dimensions

**Core Quality Dimensions for Unstructured Data** - Based on research and industry practices, both ODCS and ODPS can leverage established unstructured data quality dimensions:

**Primary Dimensions:**
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

<br> 

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

#### Example ODCS Quality Implementation
```yaml
quality:
- name: Text Readability Check
  type: custom
  engine: textstat
  implementation: |
    import textstat
    def check_readability(text):
        score = textstat.flesch_kincaid_grade(text)
        return 6.0 <= score <= 12.0
  mustBe: true

- name: Content Completeness
  type: library
  rule: completeness
  mustBeGreaterThan: 90
  unit: percent

- name: Language Consistency
  type: custom
  engine: langdetect
  implementation: |
    from langdetect import detect
    def check_language(text):
        return detect(text) == 'en'
  mustBe: true

```

<br> 

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
    type: IBM
    spec: |
      checks for document_table:
        - missing_count(content_field) = 0
        - invalid_count(content_field) < 100
```
#### 2. Pricing Plans with Quality Tiers

```yaml
pricingPlans:
  declarative:
    en:
    - name: Premium Text Analytics
      priceCurrency: USD
      price: 100
      billingDuration: month
      unit: recurring
      dataQuality:
        $ref: '#/product/dataQuality/declarative/premium'
      offering:
      - High-accuracy text processing
      - Advanced NLP quality validation
      - Real-time content scoring
```

#### 3. Automated Quality Pipeline:

```yaml
# ODPS executable quality example
executable:
- dimension: accuracy
  type: prometheus
  spec: |
    accuracy_score =
      sum(text_quality_checks_passed) /
      sum(text_quality_checks_total)
  scheduler: cron
  schedule: "0 2 * * *"

```

<br> 

## Notes 

### Text-Specific Quality Metrics
**Document-Level Metrics:**
- Readability scores (Flesch-Kincaid, SMOG index)
- Language detection confidence
- Topic coherence measurements
- Sentiment consistency analysis
**Content-Level Metrics:**
- Spelling and grammar accuracy
- Entity recognition precision
- Information extraction completeness
- Duplicate content detection


<br> 

### Integration with Quality Tools

Both ODCS and ODPS will support integration with various quality tools:
- **Text Analytics Platforms:**
    - IBM for document-level quality checks
    - Custom NLP pipelines for content validation
    - Machine learning models for quality scoring
- **Quality Monitoring Systems:**
    - Lightup AI for unstructured data observability
    - Ataccama for document quality automation
    - Monte Carlo for unstructured data monitoring


<br> 

### Best Practices and Recommendations
**1. Define Clear Quality Objectives**
- Establish baseline quality metrics for your unstructured data types
- Set realistic targets based on data usage requirements
- Align quality dimensions with business objectives and regulatory requirements

**2. Implement Layered Quality Checks**
- Structural validation: File format, encoding, basic completeness
- Content validation: Language detection, readability, topic relevance  
- Semantic validation: Entity extraction, relationship consistency, factual accuracy

**3. Leverage Referencing Capabilities**
- Create reusable quality profiles for different document types
- Use ODPS referencing to maintain consistency across data products
- Implement tiered quality levels for different service offerings

**4. Monitor and Iterate**
- Implement continuous monitoring of quality metrics
- Use feedback loops to improve quality rules over time
- Track quality trends to identify degradation patterns

<br> 

### Challenges and Considerations

- **Scale and Complexity:** Unstructured data quality assessment is computationally intensive and requires specialized tools

- **Subjectivity:** Many quality dimensions for text are subjective and context-dependent

- **Tool Integration:** Limited standardization in unstructured data quality tools requires custom implementations


<br> 

## Alternatives

> Rejected alternative solutions and the reasons why.

- Continue using custom NLP pipelines without standardization → rejected due to lack of interoperability.

- Define a new unstructured-data-only framework → rejected to avoid duplication, since ODCS/ODPS already evolve to cover this domain


## Decision

> The decision made by the TSC.

## Consequences

> The consequences of this decision.

- Positive: Standardized approach across teams, easier automation, stronger governance, and interoperability with industry tools.

- Negative: Requires initial investment in defining reusable profiles, integrating tools, and adjusting existing pipelines.

- Risk: Subjectivity in quality metrics for text data remains a challenge


## References

> Prior art, inspiration, and other references you used to create this based on what's worked well before.



