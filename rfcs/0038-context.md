# RFC-0038: Context Block for AI and Semantic Interoperability

Champion: Jean-Georges Perrin

Authors/Reviewers: Diego Carvallo, Patrick Beitsma, Martin Meermeyer, Denis Arnaud, Jean-Georges Perrin.

Slack: TBD.

GitHub issue: [#52](https://github.com/bitol-io/tsc/issues/52).

Applies to:
* [x] ODCS - Open Data Contract Standard
* [x] ODPS - Open Data Product Standard
* [ ] OORS - Open Observability Results Standard
* [ ] OOCS - Open Orchestration and Control Standard
* [ ] OMMS - Open Maturity Model Standard
* [ ] OMDS - Open Metadata Difference Standard
---

## Summary

This RFC proposes adding a standard `context` block to both ODCS (targeting v3.2.0) and ODPS (targeting v1.1.0). The block provides structured, human- and machine-readable guidance that helps AI agents, LLMs, BI tools, and semantic layer platforms understand how to interpret and consume data governed by Bitol standards. It is optional at every level, additive, and non-breaking.

The name `context` is proposed as the primary option. Alternatives are discussed below.

---

## Motivation

### Why are we doing this?

ODCS and ODPS already capture schema structure, quality rules, ownership, and SLAs. What they do not yet capture is the *interpretive guidance* that AI agents and semantic tools need to work accurately with data.

Research and production deployments show that structured context dramatically improves AI accuracy:

- Adding semantic metadata (descriptions, synonyms, sample values) to schema definitions   improves LLM SQL generation accuracy by 20–27% (Tiger Data, 2026).
- Column type annotations alone improve accuracy by 8%; semantic descriptions by 12% (Mishra, 2025).
- Few-shot examples injected dynamically into prompts are the single most effective accuracy booster for text-to-SQL tasks (multiple sources, 2024–2025).
- Negative guidance ("do not use this column for X") prevents hallucinations and incorrect joins that positive descriptions alone do not prevent (Microsoft Fabric, 2025).

Without a standardized way to express this guidance, every team reinvents it in proprietary formats, defeating the interoperability goals of both ODCS and ODPS.

### Relationship to RFC-0041 (Synonyms)

> See also Appendix C for the full vendor detail survey.

Synonyms are **out of scope for this RFC**. [RFC-0041](0041-synonyms.md) is the sole authoritative source for `synonyms` in the Bitol standards. `synonyms` is defined by RFC-0041 at the schema level (on named objects such as properties, schemas, output ports, and data products) and is not duplicated inside the `context` block. Tools that need synonym-based disambiguation MUST read them from RFC-0041's `synonyms` field, not from `context`.

(Synonyms were originally defined inside RFC-0034 — Measures and Dimensions — and factored out into RFC-0041 following the TSC meeting on 2026-04-21.)

### Relationship to OSI

> See Appendix C.1 — for the full OSI analysis and comparison to other standards.

### Use cases

1. **Text-to-SQL accuracy**: An LLM generating queries against an ODCS-governed dataset reads  `context.instructions` on the schema object to understand how to join it, and reads   `context.constraints` on a property to learn it must never be used as a filter alone.
2. **Natural language querying**: A BI tool reads `synonyms` on a measure (as defined by [RFC-0041](0041-synonyms.md)) to resolve    "revenue", "sales", and "TO" to the same `total_turnover_euros` property. Synonyms are NOT part of the `context` block.
3. **AI agent onboarding**: A data product exposes `context.instructions` at the product level    so an AI agent knows which output port to use, what domain it covers, and what questions it    can answer.
4. **Verified Q&A grounding**: A contract owner pre-anchors frequent business questions via    `context.<name>` entries with both `question` and `answer`, preventing the LLM from fabricating answers to known questions.
5. **Sensitive data protection**: `context.constraints` on a property instructs AI agents not    to expose raw values, only aggregated results.

### Alignment with guiding values

- **Small standard over large**: The `context` block is a single optional object. No new  top-level sections. All sub-fields are optional.
- **Interoperability over readability**: The structure is a superset of OSI's `ai_context`,  ensuring import/export compatibility.
- **Non-breaking**: Absence of `context` leaves all existing contracts and products valid.

---

## Design and examples

### Core concept

`context` is an optional object applicable at multiple levels in ODCS and ODPS. All sub-fields are optional. The block may be provided as a plain string (equivalent to providing only `instructions`) or as a structured object.

### Example

Example #1: Full structured object

```yaml
apiVersion: v3.2.0
kind: DataContract
context:
  instructions: >
    This contract governs the turnover dataset for the EMEA sales domain.
    Use it for revenue analysis, order volume trends, and basket value
    benchmarking. Do not use it for individual customer PII queries.
  <name>:
    - question: "What was total revenue in France last quarter?"
    - question: "Show me the top 10 countries by order count this year."
    - question: "What was the total revenue last year?"
      answer: "Query ${total_turnover_euros} grouped by year using ${turnover_ts_dim}."
    - question: "What is the average basket value?"
      answer: "Use the ${average_basket_value} measure directly; it is pre-computed."
  constraints:
    - "Do not expose individual order details; always aggregate to at least country level."
    - "Do not join with customer PII tables without explicit data access approval."
```

> **Note:** `synonyms` are defined by [RFC-0041](0041-synonyms.md) at the schema level (on named objects such as properties, schemas, output ports, and data products), not inside `context`. See the "Relationship to RFC-0041" section above.

Example #2: Full structured object with external resource (authoritative definitions)

```yaml
apiVersion: v3.2.0
kind: DataContract
context:
  instructions: >
    This contract governs the turnover dataset for the EMEA sales
    domain. Use it for revenue analysis, order volume trends, and basket value
    benchmarking. Do not use it for individual customer PII queries.
  <name>:
    - question: What was total revenue in France last quarter?
    - question: Show me the top 10 countries by order count this year.
    - question: What was the total revenue last year?
      answer: Query ${total_turnover_euros} grouped by year using ${turnover_ts_dim}.
    - question: What is the average basket value?
      answer: Use the ${average_basket_value} measure directly; it is pre-computed.
  constraints:
    - Do not expose individual order details; always aggregate to at least
      country level.
    - Do not join with customer PII tables without explicit data access approval.
  authoritativeDefinitions:
    - url: https://example.com/MyGlobalAndMarvelousOntology
      type: Ontology
      description: Link to the onto
    - url: https://example.com/MySpecificAndWonderfulGlossary
      type: Glossary
      description: Link to the glossary
    - url: https://example.com/OneOfManyTaxonomy
      type: Taxonomy
      description: Link to the taxonomy
```

### Definitions

| Key                         | UX label     | Type               | Required | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| --------------------------- | ------------ | ------------------ | -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `context`                   | Context      | `object`           | No       | AI and semantic context block. Applicable at contract, schema object, and property level in ODCS; at product and output port level in ODPS. Input ports do not define `context` — refer to the linked ODCS contract instead.                                                                                                                                                                                                                                                                 |
| `context.instructions`      | Instructions | `string`           | No       | Natural language guidance for AI agents and tools on how to use this entity. Equivalent to a system prompt scoped to this level. Be specific: clarify business terminology, specify preferred analysis approaches, and state data source priorities.                                                                                                                                                                                                                                         |
| `context.<name>`            | (TBD)        | `array of objects` | No       | Merged field replacing `examples` and `verifiedAnswers`. Each entry anchors a canonical business question. Entries without `answer` act like the former `examples` (sample questions for disambiguation and text-to-SQL priming). Entries with `answer` act like the former `verifiedAnswers` (pre-anchored responses; agents should return the curated `answer` rather than generate a new one when a query is semantically close). The field name is pending — see "Merged block" section. |
| `context.<name>[].question` | Question     | `string`           | Yes      | The canonical question.                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| `context.<name>[].answer`   | Answer       | `string`           | No       | The expected response or result description. Optional — omit to signal an unanswered sample question.                                                                                                                                                                                                                                                                                                                                                                                        |
| `context.constraints`       | Constraints  | `array of strings` | No       | Negative guidance: what AI agents must NOT do with this entity. Safety hints and constraint guidance reduce hallucinations and incorrect joins in production deployments.                                                                                                                                                                                                                                                                                                                    |

In addition to custom properties, like `CanonicalUrl`:

* Ontology (or OntologyUrl?)
* Glossary (or GlossaryUrl?)
* Taxonomy (or TaxonomyUrl?)

### Applicable levels

#### ODCS

| Level                | Rationale                                                                                            |
| -------------------- | ---------------------------------------------------------------------------------------------------- |
| Contract (top level) | Sets overall AI context for the entire data contract: domain, purpose, known limitations             |
| Schema object        | Guides agents on how to use a specific table or API object: join hints, cardinality notes            |
| Property             | Field-level guidance: what a column means, how measures should be interpreted, what not to filter on |

Quality rules and SLA sections are excluded: they are machine-executable and self-describing; adding AI context there would conflate governance with interpretation.

#### ODPS

| Level               | Rationale                                                                                                                                                                                                                                                          |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Product (top level) | Sets overall AI context for the data product: which questions it can answer, which output port to use for what purpose                                                                                                                                             |
| Output port         | Guides agents on how to consume a specific port: access patterns, recommended query approach, format hints                                                                                                                                                         |
| Input port          | Input ports do not define their own `context`. AI agents consuming a data product should refer to the `context` defined in the ODCS data contract linked to the input port. The contract is the authoritative source of semantic guidance for the data flowing in. |

---

### Additional examples

#### Schema object context (ODCS)

```yaml
schema:
  - name: turnover
    physicalName: metrics_turnover
    description: "Turnover metrics for sales and financial analysis"
    context:
      instructions: >
        This table contains pre-aggregated turnover metrics at order granularity.
        Always filter by turnover_ts when querying time ranges.
        For cross-country comparisons, normalize using the currency_code dimension.
      <name>:
        - question: "What is the total revenue for Germany in Q1 2025?"
        - question: "How many orders were placed in the UK last month?"
      constraints:
        - "Do not query without a date range filter; the table is unbounded and queries without filters will be expensive."
        - "Do not compare turnover_euros values across different currency_code values without normalization."
```

#### Property-level context (ODCS)

```yaml
      - name: turnover_euros
        logicalType: number
        description: "TurnOver converted in Euros"
        context:
          instructions: >
            This column stores the transaction value converted to EUR at the
            time of the transaction. It is NOT adjusted for refunds. Use
            total_turnover_euros (a measure) for aggregated reporting.
          constraints:
            - "Do not SUM this column directly; use the total_turnover_euros measure instead."
            - "Do not compare raw values across different product_type values without weighting."
```

#### Product-level context (ODPS)

```yaml
apiVersion: v1.1.0
kind: DataProduct
id: turnover-product
name: Turnover Data Product

context:
  instructions: >
    This data product exposes governed EMEA turnover metrics for self-service
    analytics and AI agents. Use the 'metrics' output port for aggregated
    reporting. Use the 'raw' output port only for audit and reconciliation.
    Do not use this product for real-time pricing decisions; data latency is 4 hours.
  <name>:
    - question: "What was total revenue by country last quarter?"
    - question: "Show me order volume trends for the past 12 months."
  constraints:
    - "Do not use for real-time decisions; SLA latency is 4 hours."
    - "Do not expose output port credentials in shared environments."
```

#### Output port context (ODPS)

```yaml
outputPorts:
  - name: metrics
    type: SQL
    context:
      instructions: >
        Use this port for all aggregated reporting and AI-driven analytics.
        It exposes pre-computed measures and dimensions as defined in the
        linked ODCS contract. Prefer measures over raw columns for all
        aggregation queries.
      <name>:
        - question: "SELECT total_turnover_euros, country_code_dim FROM metrics_turnover GROUP BY 2"
      constraints:
        - "Do not write INSERT, UPDATE, or DELETE statements against this port."
        - "Do not query more than 90 days of data in a single request without pagination."
```

> The formal JSON Schema definition of the `context` block is provided in Appendix D.

---

## Cascading and inheritance

### The question

When `context` is defined at multiple levels simultaneously — for example, at the contract level AND on a specific schema object AND on a specific property — which takes precedence? Do they merge, override, or concatenate? Tools need a clear rule to behave consistently.

### Analogy: output port vs. contract

This is analogous to an existing design tension in ODPS/ODCS: an output port describes how to consume a data product, while the linked data contract describes what the data promises. They operate at different levels of specificity. The port is the consumer-facing surface; the contract is the producer-facing commitment. Neither overrides the other: they compose.

The same principle applies to `context`: each level adds specificity without invalidating the level above it.

### Hierarchy

The following diagram shows the cascading order from most general (top) to most specific (bottom), across both ODCS and ODPS:

```
ODPS (Product)
└── Data Product
    └── context  ← broadest scope: domain, overall purpose, top-level constraints
        └──  (links to ODCS contract)

ODCS (Contract)
└── Data Contract
    └── context  ← contract-wide scope: dataset purpose, known limitations
        └── Schema Object  (table / API object)
            └── context  ← object-specific: join hints, cardinality, time range guidance
                └── Property  (column / measure / dimension)
                    └── context  ← field-level: what this field means, what not to do with it
```

When traversing from ODPS down to ODCS properties, the full chain is:

```
Data Product context
  → Contract context
    → Schema Object context
      → Property context
```

### Cascading rules (normative)

**Lower levels are more specific and take precedence for the same sub-field.** Higher levels provide context that lower levels do not repeat unless overriding. Cascading is normative — tooling SHOULD follow these rules.

- **`instructions`**: Tools SHOULD present context from all levels, ordered from most general to most specific (product → port → contract → schema → property). Each level adds, rather than replaces, the level above it. Tools MAY concatenate them into a single prompt prefix.
- **`<name>` (merged former `examples` + `verifiedAnswers`)**: Tools SHOULD merge all entries across levels, keyed by `question`. If two levels define the same question, the more specific level (lower in the hierarchy) takes precedence. An entry with an `answer` at a lower level overrides an entry with only a `question` at a higher level for the same question.
- **`constraints`**: Tools SHOULD merge all constraint arrays across levels. No constraint is ever overridden by a higher level — the most restrictive set applies.

---

## Merged block: former `examples` + `verifiedAnswers`

> **Status:** decision adopted — `examples` and `verifiedAnswers` are merged into a single field. Only the field name is still open. In this document the placeholder `<name>` is used everywhere the final name will appear.

`examples` and `verifiedAnswers` are two points on the same spectrum: an `examples` entry is a question with no answer yet; a `verifiedAnswers` entry is the same question once an answer has been curated. Keeping them separate forced authors to pick a bucket upfront and migrate entries between arrays as they matured. They are now merged into a single field.

### Shape

Entries are **always objects** (like `verifiedAnswers` was). `answer` is optional — an entry with only a `question` replaces the former free-text `examples` entry; an entry with both `question` and `answer` replaces the former `verifiedAnswers` entry. Free-string entries are no longer accepted: all entries are objects, for interoperability and a uniform schema.

```yaml
context:
  <name>:
    - question: "What was total revenue in France last quarter?"
    - question: "What was the total revenue last year?"
      answer: "Query ${total_turnover_euros} grouped by year using ${turnover_ts_dim}."
```

### Naming (open)

The field name is **not yet chosen**. See **Discussion 2** in "Discussions and alternatives" for the current candidates (`statements`, `interactions`, `questions`, `affirmations`). The final name will be plural. Whichever name is selected replaces `<name>` throughout the RFC.

---

## Discussions and alternatives

Open topics for TSC and community input, together with alternatives that were considered and rejected.

### Discussion 1: Field name for the `context` block

The name `context` is proposed because:

- It is not AI-specific: useful for human readers, documentation tools, and data catalogs, not only LLMs.
- It is shorter and more neutral than `aiContext` or `semanticContext`.
- It aligns with broader industry use of "context" in RAG, agent frameworks, and metadata standards.

Alternatives considered:

| Name              | Assessment                                                    |
| ----------------- | ------------------------------------------------------------- |
| `context`         | Preferred: broad, neutral, future-proof                       |
| `aiContext`       | Too narrow: implies AI-only, may age poorly                   |
| `semanticContext` | Verbose: "semantic" is already implied by measures/dimensions |
| `guidance`        | Reasonable alternative if `context` feels too generic         |
| `agentContext`    | Too narrow: implies agentic use only                          |

Community input on the final name is welcome.

---

### Discussion 2: Name for the merged `<name>` block

The merge of `examples` and `verifiedAnswers` is decided (see the "Merged block" section). The **name** of the merged field is still open. The final name will be **plural** (the field is an array). The placeholder `<name>` appears throughout the RFC until this is resolved.

Current candidates:

| Candidate      | Notes                                                                                                         |
| -------------- | ------------------------------------------------------------------------------------------------------------- |
| `statements`   | Neutral; covers both questions and declarative entries. Doesn't imply an answer is required.                  |
| `interactions` | Frames each entry as an exchange (prompt + optional reply). Reads well when `answer` is present.              |
| `questions`    | Aligns with the sub-field `question`; may feel odd for entries that read as statements rather than questions. |
| `affirmations` | Emphasises the curated/authoritative nature of entries; reads oddly for unanswered entries.                   |

Additional candidates are welcome before the name is locked.

---

### Discussion 3: Can lower levels suppress higher-level instructions? (deferred)

Should a lower level be able to explicitly *suppress* a higher-level instruction (e.g., a property-level `context` that says "ignore contract-level instructions for this field")? If so, how is that expressed?

Not in scope for this RFC — deferred for future work.

---

### Alternative A: Add fields directly without a wrapper block (rejected)

Add `instructions`, `<name>`, `constraints` as top-level fields on schema objects and contracts rather than wrapping them in a `context` block.

Rejected because: a wrapper block keeps the namespace clean, makes the concept discoverable as a unit, and allows the string shorthand. It also mirrors OSI's design, supporting interoperability.

---

### Alternative B: Reuse `description` for `instructions` (rejected)

`description` already exists at all levels and is a free-text string.

Rejected because: `description` is for human documentation. `context.instructions` is for AI agent consumption and may contain guidance that is not appropriate in documentation (e.g., join constraints, negative guidance, cardinality warnings). Conflating them degrades both.

---

### Alternative C: Separate RFCs for ODCS and ODPS (rejected)

Define `context` in one RFC for ODCS and a separate RFC for ODPS.

Rejected because: the block is identical in structure across both standards. A single shared RFC avoids duplication, ensures consistency, and signals that Bitol standards are designed as an integrated suite.

---

## Compatibility with OSI

The Bitol `context` block covers most of OSI's `ai_context`, with `synonyms` delegated to [RFC-0041](0041-synonyms.md):

| OSI `ai_context` field | Bitol equivalent                                                                    |
| ---------------------- | ----------------------------------------------------------------------------------- |
| `instructions`         | `context.instructions`                                                              |
| `synonyms`             | RFC-0041 `synonyms` (schema level, not context)                                     |
| `examples`             | `context.<name>` entries with `question` only (no `answer`)                         |
| (not present)          | `context.<name>` entries with both `question` and `answer` (former verifiedAnswers) |
| (not present)          | `context.constraints`                                                               |

An OSI-to-ODCS converter can map `ai_context.synonyms` to RFC-0041's `synonyms` field on the appropriate named object, map each `ai_context.examples` string to a `context.<name>` entry `{question: <string>}`, and map the remaining `ai_context` fields to `context`. An ODCS-to-OSI exporter can project `context.instructions` into `ai_context.instructions`, project `context.<name>[].question` values (entries without `answer`) into `ai_context.examples`, drop the answered entries (OSI has no equivalent), and project RFC-0041 `synonyms` into `ai_context.synonyms`.

---

## Decision

> The decision made by the TSC.

TBD.

---

## Consequences

- **Non-breaking**: `context` is optional at every level. All existing contracts and products  remain valid.
- **Shared structure**: The same `context` block definition is reused across ODCS and ODPS,   reducing maintenance overhead and ensuring consistent tooling.
- **OSI compatibility**: Bitol `context` covers OSI `ai_context` for `instructions` and `examples`, and adds `verifiedAnswers` and `constraints`. OSI's `synonyms` maps to [RFC-0041](0041-synonyms.md)'s `synonyms` (schema level), not to `context`.
- **RFC-0041 relationship**: `synonyms` is owned exclusively by [RFC-0041](0041-synonyms.md) at the schema level (on named objects such as properties, schemas, output ports, and data products). RFC-0038 does not define `synonyms` and does not duplicate them inside `context`. Tools needing synonym disambiguation MUST read RFC-0041's `synonyms` field.
- **ODPS coverage**: This RFC lands in ODPS v1.1.0 and defines `context` at the product and output-port levels directly. Input ports do not define their own `context`; consumers refer to the `context` in the ODCS data contract linked from the input port.

---

## References

- [Gartner Magic Quadrant for Analytics and Business Intelligence Platforms](https://www.gartner.com/en/documents/5726363) —   June 2025 (ID G00822218). Used to identify the vendor landscape in Appendix C and   to source specific vendor capability assessments (metrics layer gaps, NLQ limitations,  synonym management burdens).
- [OSI Core Metadata Specification](https://github.com/open-semantic-interchange/OSI) —   defines `ai_context` as a string or object with `instructions`, `synonyms`, `examples`.
- [Microsoft Fabric Semantic Model Best Practices](https://learn.microsoft.com/en-us/fabric/data-science/semantic-model-best-practices) —  "Prep for AI": instructions, verified answers, AI data schema scoping.
- [Tiger Data: Self-describing Postgres for LLMs](https://www.tigerdata.com/blog/the-database-new-user-llms-need-a-different-database) —  27% SQL accuracy improvement from semantic catalog enrichment.
- [Sigma Computing: Curating Context for AI Agents](https://www.sigmacomputing.com/blog/curate-context-ai-agents) —  three-pillar context model: warehouse metadata, semantic definitions, user behavior.
- [Wren AI: Semantic Engine for LLMs](https://www.getwren.ai/post/how-we-design-our-semantic-engine-for-llms-the-backbone-of-the-semantic-layer-for-llm-architecture) —  MDL-based context: instructions, calculations, constraints, relationships.
- [LLMs and Data Querying: Serialization Survey](https://medium.com/@vijayshankar.mishra/when-tables-finally-started-making-sense-my-deep-dive-into-llms-and-data-querying-3e8cf439b623) —  column type annotations +8%, semantic descriptions +12%, hybrid metadata +20–25%.
- [RFC-0034: Measures and Dimensions](0034-measures-and-dimensions.md) —   introduces measures and dimensions on properties; references RFC-0038 for the broader context framework and RFC-0041 for synonyms.
- [RFC-0041: Synonyms](0041-synonyms.md) —   defines the `synonyms` field on named objects (properties, schemas, output ports, data products). Split out of RFC-0034 on 2026-04-21. RFC-0038 delegates all synonym definition to RFC-0041.
- [RFC-0013: Relationships](approved/odcs-v3.1.0/0013-relationships-between-properties.md) —   relationships between schema properties.

---

## Appendix A: Changelog

All substantive modifications to this RFC are tracked here. Each entry includes the
date, a short summary, rationale, and the scope of affected sections.

| Date       | Change                                                                                                                                                                                                                                                       | Rationale                                                                                                                                                                                                                                                                                                                                | Affected sections                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 2026-04-21 | Retargeted all `synonyms` delegation from RFC-0034 to [RFC-0041](0041-synonyms.md).                                                                                                                                                                          | Following the TSC meeting on 2026-04-21, `synonyms` was factored out of RFC-0034 into a new standalone RFC-0041 so it applies to any named object (columns, schemas, output ports, data products), not only measures and dimensions. RFC-0038 still delegates all synonym definition externally — only the target RFC changed. | "Relationship to RFC-0034" renamed to "Relationship to RFC-0041"; Use cases (#2); Example #1 note; OSI compatibility table row and surrounding paragraph; Consequences (OSI superset and former "RFC-0034 relationship" bullet, now "RFC-0041 relationship"); References (added RFC-0041 entry, updated RFC-0034 entry); Appendix B table row "Syn." legend and the Bitol RFC-0038 row value. |
| 2026-04-17 | Removed `synonyms` from the `context` block; delegated to RFC-0034.                                                                                                                                                                                          | Decision made by the Bitol Working Group on 2026-04-14 to exclude `synonyms` from `context`. Avoids duplicating `synonyms` across two RFCs. `synonyms` belongs at the schema level (measures/dimensions) per RFC-0034, not at every level where `context` is available. Enforces RFC-0034 as the sole authoritative source for synonyms. | Relationship to RFC-0034; Use cases (#2); Examples #1 and #2; Definitions table; Property-level example; Product-level example; JSON Schema definition; Cascading rule; Alternative A; Compatibility with OSI (table and paragraph); Consequences (OSI superset and RFC-0034 relationship bullets); Appendix B table row for Bitol RFC-0038 and abbreviation legend; summary paragraph after Appendix B.                                                                                                         |
| 2026-04-17 | Added this changelog as Appendix A; renumbered prior appendices.                                                                                                                                                                                             | Track every RFC modification going forward so reviewers can see what changed and why.                                                                                                                                                                                                                                                    | Appendix A (new); former Appendix A renamed to Appendix B (Summary Comparison); former Appendix B renamed to Appendix C (Vendor Detail Survey).                                                                                                                                                                                                                                                                                                                                                                  |
| 2026-04-17 | Added `apiVersion: v3.2.0` and `kind: DataContract` to Examples #1 and #2.                                                                                                                                                                                   | Anchor the canonical ODCS examples to the target version (v3.2.0) in which this RFC is expected to land.                                                                                                                                                                                                                                 | Design and examples (Example #1, Example #2).                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| 2026-04-17 | Moved the JSON Schema definition from the body to Appendix D.                                                                                                                                                                                                | Keep the normative prose focused; relegate the formal schema to an appendix for reference.                                                                                                                                                                                                                                               | Design and examples (removed inline JSON Schema section, replaced with pointer); new Appendix D.                                                                                                                                                                                                                                                                                                                                                                                                                 |
| 2026-04-17 | Merged `examples` and `verifiedAnswers` into a single object-typed field; `answer` is now optional. Field name is still open; the placeholder `<name>` is used throughout the RFC. Candidate names: `request`, `statement`, `question`, `example`, `prompt`. | An `examples` entry is effectively a question without an answer, and promoting it to `verifiedAnswers` was a manual migration. A single field with optional `answer` removes the split and the migration step. Free-string entries are no longer accepted — all entries are objects for schema uniformity.                               | "Merged block" section (replaces the prior discussion section); Use cases (#4); Definitions table (collapsed four rows into three; `answer` now optional); Example #1, Example #2, schema object example, product example, output port example (all YAML examples updated); Cascading rule (two bullets merged into one); Compatibility with OSI (table + paragraph); Appendix D JSON Schema (collapsed `examples` and `verifiedAnswers` into a single `<name>` array of objects with only `question` required). |
| 2026-04-17 | Scoped ODPS coverage directly into this RFC; target ODPS v1.1.0. Removed the "ODPS alignment" future-work bullet and replaced it with an "ODPS coverage" statement. Product example updated to `apiVersion: v1.1.0`.                                         | The prior Consequences bullet deferred ODPS product/port `context` to a follow-up RFC. That split is unnecessary since the block definition is identical across ODCS and ODPS; landing both together in RFC-0038 avoids a second RFC for a purely mechanical extension.                                                                  | Summary (now explicitly targets ODCS v3.2.0 and ODPS v1.1.0); Product-level example (`apiVersion: v1.1.0`); Consequences (replaced "ODPS alignment" bullet with "ODPS coverage").                                                                                                                                                                                                                                                                                                                                |
| 2026-04-17 | Consolidated all open discussions under a single top-level `## Discussion` section; replaced candidate names for the merged `<name>` block with `statements`, `interactions`, `questions`, `affirmations`.                                                   | Previously there were two sibling `## Discussion:` sections plus a naming table embedded in the "Merged block" decision section. Grouping them under one umbrella keeps decided content separate from open questions, and the naming candidates were trimmed/renamed to reflect the current shortlist.                                   | Replaced `## Discussion: field name` and `## Discussion: Context Inheritance and Cascading` with a single `## Discussion` containing three numbered sub-discussions (field name, cascading, merged-block name); moved the naming table out of the "Merged block" section into Discussion 3 and replaced its pointer; candidate list in the new Discussion 3 uses `statements`, `interactions`, `questions`, `affirmations`.                                                                                    |
| 2026-04-17 | Resolved most cascading discussions into a normative "Cascading and inheritance" section; collapsed and renumbered the Discussions section; merged Alternatives into Discussions.                                                                            | After review the cascading framing (former Discussion 2) and the three per-field cascading rules (former Discussions 3/4/5) were agreed as-is, and the normative/advisory question (former Discussion 6) was settled as "normative — SHOULD". The ODPS chain question (former Discussion 8) was dropped. Remaining open items are only the `context` name, the merged-block name, and the suppression question (deferred). Alternatives A/B/C are now in the same umbrella section as Discussions for a single "what was considered, what is still open" view. | New normative `## Cascading and inheritance` section (framing + three SHOULD rules). Old `## Discussions` section replaced with `## Discussions and alternatives` containing only: Discussion 1 (`context` name), Discussion 2 (merged `<name>` block — absorbs former Discussions 9/10/11; plural decided in-text), Discussion 3 (lower-level suppression, deferred — was Discussion 7), then Alternatives A/B/C. Merged block pointer updated from "Discussion 3" to "Discussion 2". Former Discussions 5 / 6 / 8 removed (absorbed or dropped).                                                                                                                                                                                                                                                                                                      |

---

## Appendix B: Summary Comparison

Vendors as rows, features as columns. See Appendix C for full detail on each vendor.
Abbreviations: Inst. = `instructions`, Syn. = `synonyms` (RFC-0041, not part of RFC-0038 `context`), Ex. = `examples`,
VA = `verifiedAnswers`, Con. = `constraints`, In-model = context defined inside the model spec,
OL = open license, IG = independent governance, RFC = formal RFC process,
NSB = neutral stewardship body, Gov. = governed with contracts,
Prop. = property level, Rel. = relationship level, Port. = portable across vendors,
Runtime = runtime-executable semantics.

| Vendor                    | Inst.    | Syn.     | Ex.    | VA      | Con.      | In-model | OL      | IG      | RFC     | NSB     | Gov. | Prop. | Rel. | Port.   | Runtime    |
| ------------------------- | -------- | -------- | ------ | ------- | --------- | -------- | ------- | ------- | ------- | ------- | ---- | ----- | ---- | ------- | ---------- |
| **Bitol RFC-0038**        | yes      | RFC-0041 | yes    | yes     | yes       | yes      | yes     | yes     | yes     | yes     | yes  | yes   | yes  | yes     | no         |
| Actian AI Analyst (Wobby) | yes      | partial  | no     | no      | yes exec. | yes      | no      | no      | no      | no      | no   | yes   | no   | no      | yes SemQL  |
| Amazon QuickSight         | no       | no       | no     | no      | no        | yes      | no      | no      | no      | no      | no   | yes   | no   | no      | yes        |
| AtScale SML               | no       | no       | no     | no      | no        | yes      | yes     | no      | no      | no      | no   | yes   | no   | yes     | yes        |
| Cube                      | no       | meta     | no     | no      | no        | yes      | yes     | no      | no      | no      | no   | yes   | no   | partial | yes SemSQL |
| Databricks Metric Views   | no       | yes      | no     | no      | no        | yes      | no      | no      | no      | no      | no   | yes   | no   | no      | yes        |
| dbt / MetricFlow          | no       | meta     | no     | no      | no        | yes      | yes     | no      | no      | no      | no   | yes   | no   | partial | yes        |
| Domo                      | no       | no       | no     | no      | no        | yes      | no      | no      | no      | no      | no   | yes   | no   | no      | yes        |
| GoodData                  | no       | no       | no     | no      | no        | yes      | partial | no      | no      | no      | no   | yes   | no   | yes     | yes        |
| IBM watsonx BI / Cognos   | auto     | auto     | no     | no      | no        | yes      | no      | no      | no      | no      | no   | yes   | no   | partial | yes        |
| Incorta                   | no       | manual   | no     | no      | no        | yes      | no      | no      | no      | no      | no   | yes   | no   | no      | yes        |
| Jaspersoft (TIBCO)        | no       | no       | no     | no      | no        | yes      | partial | no      | no      | no      | no   | yes   | no   | no      | yes        |
| Lightdash                 | no       | no       | no     | no      | no        | yes      | yes     | no      | no      | no      | no   | yes   | no   | partial | yes        |
| LookML (Looker / Google)  | external | external | no     | no      | no        | no       | no      | no      | no      | no      | no   | yes   | no   | no      | yes        |
| Metabase                  | no       | no       | no     | no      | no        | yes      | yes     | no      | no      | no      | no   | yes   | no   | no      | yes        |
| Microsoft Power BI        | yes      | partial  | via VA | yes     | partial   | yes      | no      | no      | no      | no      | no   | yes   | no   | no      | yes DAX    |
| Omni Analytics            | yes      | partial  | no     | no      | partial   | yes      | no      | no      | no      | no      | no   | yes   | no   | no      | yes        |
| OSI                       | yes      | yes      | yes    | no      | no        | yes      | yes     | partial | partial | no      | no   | yes   | yes  | yes     | no         |
| Oracle Analytics Cloud    | no       | no       | no     | no      | no        | yes      | no      | no      | no      | no      | no   | yes   | no   | no      | yes        |
| Pyramid Analytics         | no       | no       | no     | no      | no        | yes      | no      | no      | no      | no      | no   | yes   | no   | no      | yes        |
| Qlik                      | no       | no       | no     | no      | no        | yes      | no      | no      | no      | no      | no   | yes   | no   | no      | yes        |
| Salesforce / Tableau      | partial  | partial  | no     | no      | no        | yes      | no      | no      | no      | no      | no   | yes   | no   | no      | yes        |
| SAP Analytics Cloud       | no       | no       | no     | no      | no        | yes      | no      | no      | no      | no      | no   | yes   | no   | no      | yes        |
| SAS Viya                  | no       | catalog  | no     | no      | no        | yes      | no      | no      | no      | no      | no   | yes   | no   | no      | yes        |
| Sigma Computing           | no       | no       | no     | no      | no        | yes      | no      | no      | no      | no      | no   | yes   | no   | no      | yes        |
| Sisense                   | no       | no       | no     | partial | no        | yes      | no      | no      | no      | no      | no   | yes   | no   | no      | yes        |
| Snowflake Semantic Views  | yes      | yes      | yes    | no      | no        | yes      | no      | no      | no      | no      | no   | yes   | yes  | no      | yes        |
| Apache Superset           | no       | no       | no     | no      | no        | yes      | yes     | yes     | no      | yes ASF | no   | yes   | no   | no      | yes        |
| Tellius                   | no       | auto     | no     | no      | no        | yes      | no      | no      | no      | no      | no   | yes   | no   | no      | yes        |
| ThoughtSpot / Spotter     | partial  | yes      | no     | partial | partial   | yes      | no      | no      | no      | no      | no   | yes   | no   | partial | yes        |
| Strategy (MicroStrategy)  | no       | no       | no     | no      | no        | yes      | no      | no      | no      | no      | no   | yes   | no   | no      | yes        |
| Zoho Analytics            | no       | no       | no     | no      | no        | yes      | no      | no      | no      | no      | no   | yes   | no   | no      | yes        |

**Legend for non-binary values:**
- `external`: feature exists but is defined outside the model spec, not version-controlled with it
- `meta`: available via an unstructured `meta` escape hatch, not a standardized field
- `partial`: some support but incomplete or informal
- `auto`: generated automatically by the platform, not author-defined
- `manual`: requires manual management outside any standard mechanism
- `catalog`: available at the data catalog level, not the schema/model level
- `exec.`: executable (not just documentation)
- `partial`: present but limited (OSI governance is multi-vendor consortium, not a neutral foundation)
- ASF = Apache Software Foundation

**Reading the governance columns (OL, IG, RFC, NSB):**
- OSI has an open license and multiple vendors but no neutral stewardship body and no
  formal RFC process. It is a vendor consortium agreement, not a standard.
- dbt/MetricFlow, AtScale SML, Metabase, Lightdash are open source but controlled by
  single commercial entities or communities without formal standardization. They are
  open source projects, not open standards.
- Apache Superset has a neutral stewardship body (Apache Software Foundation) but no
  formal RFC process for its semantic layer specifically.
- Bitol ODCS/ODPS is the only entry that satisfies all four criteria: open license,
  independent governance via Linux Foundation TSC, formal RFC process, and neutral
  stewardship body.

RFC-0038 (combined with [RFC-0041](0041-synonyms.md) for `synonyms`) is the only open standard combining
`instructions`, `examples`, `verifiedAnswers`, and `constraints` inside the model spec,
alongside schema-level `synonyms`, with data governance via ODCS/ODPS. See Appendix C
for full vendor detail.

---

## Appendix C: Vendor Detail Survey

This appendix surveys how each semantic layer tool or initiative handles AI context,
instructions, synonyms, examples, and constraints in detail. It informs the design of
the `context` block and positions RFC-0038 relative to the broader ecosystem.
Vendors are listed alphabetically.

Source: Gartner Magic Quadrant for Analytics and Business Intelligence Platforms,
June 2025 (ID G00822218), was used to identify the vendor landscape. Not all vendors
below are evaluated in the Gartner MQ; some are independent semantic layer tools.

---

### B.1 Actian AI Analyst (Wobby)

**Type:** AI-native conversational analytics platform. SaaS. Proprietary.
**Note:** Actian AI Analyst was formerly known as Wobby until March 10, 2026, following
its acquisition by Actian (HCLSoftware). This is Actian's own product, making it the
only entry in this appendix with direct production relevance to Bitol's parent
organization.

Actian AI Analyst (Wobby) was built from the ground up around AI agent consumption of a
semantic layer. Its design choices are therefore the most directly informative for
RFC-0038 of any tool surveyed, as they reflect real production experience with what AI
agents actually need.

#### Semantic layer structure

The Actian AI Analyst semantic layer has six first-class components:

- **Models**: core business entities mapping to tables or views
- **Dimensions**: attributes for filtering and grouping
- **Measures**: pre-aggregated numerical calculations
- **Pre-defined Filters**: constrained query conditions scoped to a model
- **Relationships**: cross-model joins
- **Metrics**: model-spanning calculated KPIs
- **Glossary**: business term definitions (independent of models)

#### AI context fields

Actian AI Analyst defines two distinct free-text fields on models that map directly to RFC-0038
`context` sub-fields:

**`description`** (on models, dimensions, measures):
A concise explanation of what the entity represents. Intended for both human readers
and AI agents. Equivalent to ODCS `description`, already present in ODCS and ODPS.

**`guidance`** (on models):
Free-text context specifically for AI agents. Covers:
- Special business rules or constraints
- How the data should be interpreted
- When to use or avoid this model
- Important nuances about the data

Example from Actian AI Analyst docs:
```
Use this model to analyze customer behavior and segmentation.
Customers can be either B2B or B2C, with different pricing tiers for each type.
```

Actian AI Analyst's documentation describes `guidance` as: "instructions you'd give to a new
team member analyzing this data." This is conceptually identical to
`context.instructions` in RFC-0038.

#### SemQL: runtime-executable semantic references

Actian AI Analyst introduces SemQL (Semantic Query Language), a query language for AI agents that
references semantic layer components directly using double-brace syntax:

```sql
SELECT {{ organizations.user_count }}
FROM {{ organizations }}
WHERE {{ organizations.active }} = true
```

SemQL translates semantic references into executable SQL at runtime. This is the most
concrete implementation of the "agent-ready data" concept surveyed here: the `context`
block in RFC-0038 provides the interpretive layer; SemQL provides the execution layer
on top of it.

#### Steward: AI-maintained semantic layer

The Steward agent monitors AI analyst conversations, detects knowledge gaps in the
semantic layer, and proactively proposes updates. This is a direct consequence of having
a rich, structured `guidance` block: the Steward can compare what agents asked versus
what the semantic layer defines, and flag mismatches.

This has a direct implication for RFC-0038: a well-structured `context` block (with
`instructions`, `examples`, and `constraints`) creates the surface area for a Steward-
like agent to validate and maintain semantic definitions over time.

#### Pre-defined filters as `constraints` in practice

Actian AI Analyst's `pre-defined filters` component is the closest production equivalent to
`context.constraints` in any tool surveyed. Rather than free-text negative guidance,
Actian AI Analyst encodes constrained query conditions directly as executable filters:

```
filter: current_member_count > 0
label: "Active organizations only"
```

This is more structured than RFC-0038's `constraints` (which are free-text strings),
but serves the same purpose: preventing agents from querying data incorrectly.
RFC-0038's `constraints` field may evolve toward structured filter expressions in a
future RFC if production experience warrants it.

**Relationship context:** Actian AI Analyst defines relationships as a first-class semantic
component (cross-model joins) but does not document an AI context or guidance field on
them.

**Actian AI Analyst (Wobby) context fields mapped to RFC-0038:**

| RFC-0038 `context` field | Actian AI Analyst equivalent       | Level                     | Structured?    |
| ------------------------ | ---------------------------------- | ------------------------- | -------------- |
| `instructions`           | `guidance`                         | Model                     | yes free text  |
| `instructions`           | `description`                      | Model, dimension, measure | yes free text  |
| `synonyms`               | Glossary (separate component)      | Cross-model               | partial        |
| `examples`               | None documented                    | —                         | no             |
| `verifiedAnswers`        | None documented                    | —                         | no             |
| `constraints`            | `pre-defined filters` (executable) | Model                     | yes structured |

**Key gap vs. RFC-0038:** no `synonyms` at the field level, no `examples`, no
`verifiedAnswers`. The Glossary provides business term definitions but is decoupled
from individual models and properties.

**Key strength vs. RFC-0038:** `pre-defined filters` are executable constraints, not
just documentation. SemQL provides a runtime bridge from semantic definitions to SQL.
The Steward agent demonstrates that a rich semantic context enables automated maintenance.

---

### B.2 Amazon QuickSight

**Type:** BI-native analytics platform. Proprietary (AWS). Gartner MQ Challenger (2025).

QuickSight provides dashboards, NLQ via Amazon Q, and scenario modeling. The Gartner
MQ explicitly flags a key limitation: QuickSight's metrics layer is constrained because
"metrics are linked to specific datasets and can't be accessed separately," and its
"lack of native connections restricts its effectiveness as a semantic layer for integrating
various platforms." There is no dedicated AI context block.

**What QuickSight has that maps to `context`:**

| `context` field   | QuickSight equivalent      | Structured? |
| ----------------- | -------------------------- | ----------- |
| `instructions`    | None                       | no          |
| `synonyms`        | None                       | no          |
| `examples`        | None                       | no          |
| `verifiedAnswers` | None                       | no          |
| `constraints`     | None                       | no          |
| `description`     | Dataset field descriptions | yes         |

**Key gap:** Gartner identifies QuickSight's universal metric layer as a barrier. RFC-0038
would provide richer semantic context than QuickSight currently supports.

---

### B.3 AtScale SML (Semantic Modeling Language)

**Type:** Universal/headless semantic layer. Open-source (Apache 2.0). YAML-based.
Released September 2024. Joined OSI initiative January 2026.

SML is described as a superset of all other existing semantic modeling languages. It
supports metrics, dimensions, hierarchies, semi-additive measures, many-to-many
relationships, and multidimensional constructs. It is the most comprehensive open
semantic modeling language in terms of analytical capability.

SML does not define a dedicated AI context block. It relies on `description` fields
and the overall richness of its semantic definitions (hierarchies, named calculations,
explicit relationships) to provide AI grounding indirectly.

SML's key differentiator over OSI and dbt is its multidimensional heritage: it handles
hierarchies (country → region → city), semi-additive measures, and OLAP-style modeling
that OSI v0.1.1 does not yet cover.

**What SML has that maps to `context`:**

| `context` field   | SML equivalent               | Structured?              |
| ----------------- | ---------------------------- | ------------------------ |
| `instructions`    | None                         | no                       |
| `synonyms`        | None documented              | no                       |
| `examples`        | None                         | no                       |
| `verifiedAnswers` | None                         | no                       |
| `constraints`     | None                         | no                       |
| `description`     | `description` on all objects | yes                      |
| Hierarchies       | Full hierarchy support       | yes (unique to SML/OLAP) |

**Relationship context:** SML supports relationships but defines no AI context on them.

**Key gap:** SML is the richest analytical modeling language but has the weakest
AI-readiness features. Bitol `context` addresses the gap SML has not yet tackled.

---

### B.4 Cube

**Type:** Universal/headless semantic layer. Open-source core (Apache 2.0), commercial cloud. YAML and JavaScript. API-first.

Cube defines data models as "cubes" (business entities with measures, dimensions, and joins). It uses `description`, `title`, and an unstructured `meta` field for custom metadata. There is no dedicated AI context block.

Cube's AI grounding strategy relies on its meta API: AI agents call the API to download all semantic definitions as embeddings for RAG retrieval, rather than reading an inline context field. Cube also offers Semantic SQL, which extends standard SQL with a `MEASURE()` function, letting AI agents query through the semantic layer deterministically.

```yaml
cubes:
  - name: orders
    sql_table: orders
    description: "All customer orders"
    measures:
      - name: revenue
        type: sum
        sql: amount
        title: "Total Revenue"
        description: "Sum of all order amounts"
        meta:
          synonyms: ["sales", "income"]  # unstructured, tool-specific
```

**What Cube has that maps to `context`:**

| `context` field   | Cube equivalent                              | Structured? |
| ----------------- | -------------------------------------------- | ----------- |
| `instructions`    | None (via `meta` informally)                 | no          |
| `synonyms`        | `meta.synonyms` (unstructured)               | no          |
| `examples`        | None                                         | no          |
| `verifiedAnswers` | None                                         | no          |
| `constraints`     | None                                         | no          |
| `description`     | `description` on cubes, measures, dimensions | yes         |
| `title`           | `title` on all objects                       | yes         |

**Relationship context:** Cube defines joins inside cube definitions but has no AI context field on them.

**Key gap:** Like dbt, Cube relies on `meta` as an unstructured escape hatch. Bitol `context` standardizes what Cube leaves to conventions.

---

### B.5 Databricks Metric Views

**Type:** Platform-native semantic layer (Unity Catalog). Proprietary (Databricks).
YAML-based. Primary inspiration for RFC-0034.

Databricks Metric Views support `synonyms` and `display_name` on dimensions and measures
via a `semantic_metadata` block. There is no dedicated AI instructions or constraints field.

```yaml
dimensions:
  - name: country_code
    expr: country_code
    display_name: Country Code
    synonyms:
      - "ISO-3166-2 country code"
    comment: "Country where the order was made"
```

**What Databricks has that maps to `context`:**

| `context` field   | Databricks equivalent             | Structured? |
| ----------------- | --------------------------------- | ----------- |
| `instructions`    | None                              | no          |
| `synonyms`        | `synonyms` on dimensions/measures | yes         |
| `examples`        | None                              | no          |
| `verifiedAnswers` | None                              | no          |
| `constraints`     | None                              | no          |
| `description`     | `comment` on all objects          | yes         |

**Relationship context:** Databricks Metric Views defines joins but has no AI context
field on relationships.

**Note:** RFC-0034's `implementationType` and RFC-0038's `context.synonyms` are directly
inspired by the Databricks Metric Views structure.

---

### B.6 dbt Semantic Layer / MetricFlow

**Type:** Universal/headless semantic layer. Open-source (Apache 2.0). YAML-based.
Now part of OSI initiative.

dbt MetricFlow uses `description` on semantic models, entities, dimensions, and measures.
There is no dedicated AI context or instructions block.

The dbt community has explicitly flagged the gap: a GitHub discussion on the MetricFlow
spec proposal noted the need for a "more universal one-place for all semantic information"
and that `meta` is relied on heavily for "synonyms, formatting, natural language info, etc."
— but this is unstructured and tool-specific.

```yaml
semantic_models:
  - name: order_item
    description: |
      Items contained in each order. This table contains one row per order item.
    measures:
      - name: revenue
        description: "Total revenue in USD"
        agg: sum
        expr: revenue_usd
```

**What dbt has that maps to `context`:**

| `context` field   | dbt / MetricFlow equivalent            | Structured? |
| ----------------- | -------------------------------------- | ----------- |
| `instructions`    | None (community workaround via `meta`) | no          |
| `synonyms`        | None (community workaround via `meta`) | no          |
| `examples`        | None                                   | no          |
| `verifiedAnswers` | None                                   | no          |
| `constraints`     | None                                   | no          |
| `description`     | `description` on all objects           | yes         |

**Relationship context:** dbt uses entities (join keys) rather than explicit
relationship definitions. No AI context on join semantics.

**Key gap:** dbt relies on `meta` as an unstructured escape hatch for AI guidance. Bitol
standardizes this into a first-class, schema-validated block.

---

### B.7 Domo

**Type:** BI and analytics platform. Proprietary (Domo). Gartner MQ Challenger (2025).

Domo provides dashboards, data integration, and a basic metrics layer. Its AI vision
includes agentic workflows and agent builders. However, the Gartner MQ notes that "Domo's
natural language query features are limited, missing predictive suggestions and autotyping
capabilities." No dedicated AI context block is documented.

**What Domo has that maps to `context`:**

| `context` field   | Domo equivalent            | Structured? |
| ----------------- | -------------------------- | ----------- |
| `instructions`    | None                       | no          |
| `synonyms`        | None                       | no          |
| `examples`        | None                       | no          |
| `verifiedAnswers` | None                       | no          |
| `constraints`     | None                       | no          |
| `description`     | Dataset field descriptions | yes         |

**Key gap:** Domo's AI vision is agentic but not yet semantic-layer-oriented. No structured context mechanism.

---

### B.8 GoodData

**Type:** Headless BI and metrics layer. Open-source option + SaaS. Gartner MQ Niche Player (2025).

GoodData is the most philosophically aligned vendor with Bitol in the Gartner MQ. Its
"headless vision" focuses on a centralized metrics layer as a universal semantic framework.
Gartner highlights: "GoodData focuses on the market's need for a centralized metrics layer
that ensures consistency of metric definitions and maps to business objectives. Its open
semantic layer is accessible for various downstream analytics use cases, positioning
GoodData as an agnostic modeling tool in the metrics layer market."

GoodData uses an analytics-as-code approach with YAML-based metric definitions, Git
integration, and REST APIs. Its data catalog integrates metadata from competing BI tools.
There is no documented dedicated AI context block, but its metric-first architecture and
open semantic layer are closest in spirit to ODCS's approach.

**What GoodData has that maps to `context`:**

| `context` field   | GoodData equivalent               | Structured? |
| ----------------- | --------------------------------- | ----------- |
| `instructions`    | None documented                   | no          |
| `synonyms`        | None documented                   | no          |
| `examples`        | None                              | no          |
| `verifiedAnswers` | None                              | no          |
| `constraints`     | None                              | no          |
| `description`     | Metric and dimension descriptions | yes         |

**Key finding:** GoodData's headless, metrics-first, analytics-as-code philosophy mirrors
Bitol's positioning. An ODCS data contract could serve as the governed foundation for
a GoodData semantic layer. RFC-0038's `context` block would add AI-readiness to what
GoodData provides structurally.

---

### B.9 IBM Cognos Analytics / watsonx BI

**Type:** Enterprise BI platform (Cognos) + AI-native insight agent (watsonx BI).
Proprietary (IBM). Gartner MQ Visionary (2025).

IBM has two products relevant to RFC-0038. Cognos Analytics is the established enterprise
BI platform with a semantic modeling tool (Framework Manager) that defines dimensions,
hierarchies, joins, and calculated fields. watsonx BI (GA November 2025) is IBM's new
AI-native successor that imports Cognos models and adds conversational NL querying.

**watsonx BI is the most significant finding for RFC-0038:** it automatically enriches
data with "business synonyms and descriptions," builds semantic models, and suggests
metric definitions. This is the only vendor surveyed where the platform itself
auto-generates `context`-equivalent metadata rather than requiring authors to write it
manually.

Key capabilities of watsonx BI:
- Automatically enriches data with business synonyms and descriptions
- Imports Cognos Framework Manager models (dimensions, hierarchies, joins, filters)
- Governed semantic layer with centralized metrics catalog
- Can consume metrics from third-party semantic layers and publish its own
- Supports multiple LLMs for NL querying
- Business Glossary in IBM Information Catalog (since May 2024) provides semantic terms

**What IBM has that maps to `context`:**

| `context` field   | IBM equivalent                                        | Structured?         |
| ----------------- | ----------------------------------------------------- | ------------------- |
| `instructions`    | None (watsonx BI generates context automatically)     | no (auto-generated) |
| `synonyms`        | Auto-generated by watsonx BI; Business Glossary terms | yes (auto)          |
| `examples`        | None                                                  | no                  |
| `verifiedAnswers` | None                                                  | no                  |
| `constraints`     | None                                                  | no                  |
| `description`     | Auto-enriched descriptions; Glossary definitions      | yes (auto)          |

**Relationship context:** Cognos Framework Manager models include joins which are
imported into watsonx BI, but no AI context on relationships is documented.

**Key finding:** IBM watsonx BI demonstrates that the metadata RFC-0038 asks authors to
write manually can also be auto-generated by AI. This validates the RFC-0038 field
design while pointing to a future where `context` fields could be auto-populated from
ODCS schemas. The Steward agent in Actian AI Analyst pursues the same vision.

---

### B.10 Incorta

**Type:** Enterprise analytics platform focused on ERP data. Proprietary. Gartner MQ Niche Player (2025).

Incorta specializes in rapid integration with Oracle, Salesforce, and SAP. The Gartner
MQ notes a relevant AI context gap: "business-specific synonyms require manual management"
in Incorta Copilot — precisely the problem RFC-0038 solves by standardizing synonyms as
a first-class field. There is no dedicated AI context block.

**What Incorta has that maps to `context`:**

| `context` field   | Incorta equivalent                   | Structured?               |
| ----------------- | ------------------------------------ | ------------------------- |
| `instructions`    | None                                 | no                        |
| `synonyms`        | Manual synonym management in Copilot | no (manual, unstructured) |
| `examples`        | None                                 | no                        |
| `verifiedAnswers` | None                                 | no                        |
| `constraints`     | None                                 | no                        |
| `description`     | Field descriptions                   | yes                       |

**Key gap:** Gartner explicitly identifies the manual synonym management burden as a
limitation. RFC-0038's `context.synonyms` directly addresses this.

---

### B.11 Jaspersoft (TIBCO)

**Type:** Embedded reporting and analytics platform. Open-source core (AGPL), commercial.
Proprietary (TIBCO/Cloud Software Group). Not in Gartner MQ (developer/embedded focus).

Jaspersoft is a developer-focused embedded BI platform primarily designed for pixel-perfect
reporting and embedded analytics in applications. It has a "Domains" metadata layer that
provides business-friendly views of data, abstracting SQL complexity for end users. This
is a traditional semantic abstraction layer — it maps tables to business concepts — but
has no AI context, NL querying, instructions, or synonyms mechanism.

Jaspersoft was not included in the Gartner MQ for ABI Platforms. Its positioning is
explicitly developer/embedded, not business user or AI-agent facing.

**What Jaspersoft has that maps to `context`:**

| `context` field   | Jaspersoft equivalent                  | Structured? |
| ----------------- | -------------------------------------- | ----------- |
| `instructions`    | None                                   | no          |
| `synonyms`        | None                                   | no          |
| `examples`        | None                                   | no          |
| `verifiedAnswers` | None                                   | no          |
| `constraints`     | None                                   | no          |
| `description`     | Field labels in Domains metadata layer | yes         |

**Key gap:** Jaspersoft's semantic layer is traditional and reporting-focused with no
AI context mechanism. It is not a target consumer of RFC-0038.

---

### B.12 Lightdash

**Type:** Open-source BI tool built on top of dbt. Apache 2.0.

Lightdash exposes dbt semantic models directly as a BI layer — metrics and dimensions
defined in dbt YAML are automatically surfaced in Lightdash. It inherits dbt's
`description` and `meta` fields. There is no dedicated AI context block beyond what dbt
provides.

**What Lightdash has that maps to `context`:**

| `context` field   | Lightdash equivalent       | Structured? |
| ----------------- | -------------------------- | ----------- |
| `instructions`    | None (inherits dbt `meta`) | no          |
| `synonyms`        | None                       | no          |
| `examples`        | None                       | no          |
| `verifiedAnswers` | None                       | no          |
| `constraints`     | None                       | no          |
| `description`     | Inherits dbt `description` | yes         |

**Key gap:** Lightdash is a thin layer on top of dbt and inherits all of dbt's
limitations regarding AI context.

---

### B.13 LookML (Looker / Google)

**Type:** BI-native semantic layer. Proprietary (Google Cloud). Git-friendly YAML-like DSL. Gartner MQ Leader (2025).

LookML does not have a dedicated AI context block. Instead, it uses:
- `description` on views, explores, dimensions, and measures for human documentation
- Free-text NL instructions in "Conversational Analytics" (Gemini in Looker), specified
  outside the LookML model via a separate configuration UI

Looker's Conversational Analytics accepts instructions in the form of:
- Synonyms: "If someone says 'revenue', use the 'total_sales' field"
- Key fields: "The most important fields are order_id, customer_id, and order_date"
- Filtering guidance: "Unless stated otherwise, always filter data for the current year"
- Business definitions: "We consider 'loyal' customers to be those with more than 5 orders"

These instructions live outside the LookML model itself, not inside the schema definition.
This means they are not version-controlled alongside the model and not portable.

**What LookML has that maps to `context`:**

| `context` field   | LookML equivalent               | In-model?   |
| ----------------- | ------------------------------- | ----------- |
| `instructions`    | Conversational Analytics config | no external |
| `synonyms`        | Conversational Analytics config | no external |
| `examples`        | Not defined                     | no          |
| `verifiedAnswers` | Not defined                     | no          |
| `constraints`     | Not defined                     | no          |
| `description`     | `description` on any object     | yes         |

**Relationship context:** LookML defines joins inside `explore` blocks with `sql_on`
expressions. No AI context field on joins.

**Key gap:** LookML separates AI guidance from the model definition. Bitol embeds it
in the contract, making it version-controlled, governed, and portable by design.

---

### B.14 Metabase

**Type:** Open-source BI tool. AGPL license.

Metabase has a basic semantic layer via "Segments" (saved filters) and "Metrics" (saved
aggregations). There is no AI context block, no instructions field, no synonyms, and no
structured guidance for AI agents.

**What Metabase has that maps to `context`:**

| `context` field   | Metabase equivalent      | Structured?   |
| ----------------- | ------------------------ | ------------- |
| `instructions`    | None                     | no            |
| `synonyms`        | None                     | no            |
| `examples`        | None                     | no            |
| `verifiedAnswers` | None                     | no            |
| `constraints`     | None                     | no            |
| `description`     | Field descriptions in UI | yes (UI only) |

**Key gap:** Metabase has no AI context concept.

---

### B.15 Microsoft Power BI / Fabric "Prep for AI"

**Type:** BI-native semantic layer. Proprietary (Microsoft). Gartner MQ Leader (2025).

Microsoft Fabric's "Prep for AI" feature is the closest existing implementation to what
RFC-0038 proposes. It defines three AI-readiness components directly on semantic models:

1. **AI Instructions**: Free-text NL guidance, equivalent to `context.instructions`.
   Can include business terminology, preferred analysis approaches, data source priorities.
2. **Verified Answers**: Pre-anchored question/answer pairs, equivalent to
   `context.verifiedAnswers`. When an AI agent receives a semantically similar question,
   it returns the verified answer directly.
3. **AI Data Schema**: Scoping which tables, columns, and measures are visible to AI
   agents — a form of access governance not covered by RFC-0038 (left to ODCS roles/access).

Microsoft documentation explicitly states: "AI instructions are unstructured guidance that
the LLM interprets, but there's no guarantee it follows them exactly. Clear, specific
instructions are more effective than complex or conflicting ones."

**What Power BI "Prep for AI" has that maps to `context`:**

| `context` field        | Power BI equivalent                   | Structured?           |
| ---------------------- | ------------------------------------- | --------------------- |
| `instructions`         | AI Instructions                       | yes (free text)       |
| `synonyms`             | Via AI Instructions + column synonyms | partial               |
| `examples`             | Via Verified Answers                  | yes                   |
| `verifiedAnswers`      | Verified Answers (question + answer)  | yes                   |
| `constraints`          | Via AI Instructions (informal)        | partial               |
| AI data schema scoping | Prep for AI schema                    | yes (not in RFC-0038) |

**Relationship context:** Power BI defines relationships in the data model but has no
AI context field on them.

**Key gap:** Power BI's "Prep for AI" is proprietary, model-specific, and not
portable. RFC-0038 standardizes the same concepts in an open, vendor-neutral format.

---

### B.16 Omni Analytics

**Type:** BI-native semantic layer platform. Proprietary. OSI founding member.

Omni is the most notable BI-native tool for RFC-0038 purposes: it defines explicit "AI
context" at three levels — model, Topic (curated dataset), and field — using free-text
instructions. This independently mirrors RFC-0038's multi-level `context` block design.

Omni's documentation describes model-level AI context as: "The highest-level way to
guide how Omni's AI behaves across your entire workspace. Think of this as context
you'd give a new analyst on day one: what kind of business you are, which metrics
matter, and how answers should be delivered."

```yaml
# Omni model-level AI context
model:
  ai_context: |
    This model covers EMEA sales data. Revenue is always in EUR.
    Use total_turnover for top-line reporting. Do not combine
    B2B and B2C metrics without filtering by customer_type first.

# Topic-level AI context
topic:
  name: turnover
  ai_context: |
    Use this topic for revenue analysis and basket value benchmarking.

# Field-level AI context
field:
  name: total_turnover
  ai_context: "Always use this measure for total revenue. Do not sum raw turnover_euros directly."
```

**What Omni has that maps to `context`:**

| `context` field   | Omni equivalent                | Level               | Structured?   |
| ----------------- | ------------------------------ | ------------------- | ------------- |
| `instructions`    | `ai_context`                   | Model, Topic, Field | yes free text |
| `synonyms`        | Via AI instructions (informal) | Model               | partial       |
| `examples`        | None documented                | —                   | no            |
| `verifiedAnswers` | None documented                | —                   | no            |
| `constraints`     | Via AI instructions (informal) | Model               | partial       |
| `description`     | `description` on all objects   | yes                 | yes           |

**Relationship context:** Not documented.

**Key finding:** Omni independently arrived at a multi-level `ai_context` pattern
(model → topic → field) nearly identical to RFC-0038's `context` block hierarchy
(contract → schema → property). This validates the RFC-0038 design from production BI.

---

### B.17 Open Semantic Interchange (OSI)

**Backed by:** Snowflake, Salesforce, dbt Labs, Databricks, Omni, Preset, AtScale.
**Released:** January 27, 2026 (v0.1.1). Apache 2.0.

OSI defines `ai_context` as a `oneOf` — either a plain string or a structured object:

```yaml
ai_context:
  instructions: "Natural language guidance for AI agents"
  synonyms:
    - "alternative name"
  examples:
    - "What was total revenue last quarter?"
```

**Coverage by level:**

| OSI Entity            | Has `ai_context`? |
| --------------------- | ----------------- |
| `SemanticModel` (top) | yes               |
| `Dataset`             | yes               |
| `Field`               | yes               |
| `Metric`              | yes               |
| `Relationship`        | yes               |

**Fields defined:**
- `instructions` (string): NL guidance
- `synonyms` (array of strings): alternative names
- `examples` (array of strings): sample questions

**Relationship context:** OSI is the only surveyed standard that explicitly defines
`ai_context` on relationships. This allows authors to explain join semantics in natural
language: e.g., "This relationship connects orders to customers on a many-to-one basis;
always filter by date range before joining to avoid fan-out."

**What OSI lacks:** no `constraints` (negative guidance), no `verifiedAnswers`, no
governance, quality, or contract semantics of any kind.

**Bitol `context` vs. OSI `ai_context`:**

| Field                         | OSI                   | Bitol RFC-0038      |
| ----------------------------- | --------------------- | ------------------- |
| `instructions`                | yes                   | yes                 |
| `synonyms`                    | yes (in `ai_context`) | yes (in `context`)  |
| `examples`                    | yes                   | yes                 |
| `verifiedAnswers`             | no                    | yes                 |
| `constraints`                 | no                    | yes                 |
| Governance / contracts        | no                    | yes (via ODCS/ODPS) |
| Applied at property level     | yes                   | yes                 |
| Applied at relationship level | yes                   | open (see issue 6)  |
| Multi-dialect SQL             | yes                   | no (deferred)       |

Bitol `context` is a strict superset of OSI `ai_context`. An OSI-to-ODCS converter can
map `ai_context` to `context` losslessly.

---

### B.18 Oracle Analytics Cloud

**Type:** Enterprise BI and analytics platform. Proprietary (Oracle). Gartner MQ Leader (2025).

Oracle Analytics Cloud (OAC) provides NLQ via the Oracle Analytics AI Assistant, which
supports both Oracle's own GenAI model and third-party LLMs (OpenAI). It offers
comprehensive integration with Oracle Fusion Data Intelligence providing packaged semantic
models for finance, CRM, and supply chain.

OAC's semantic layer is primarily expressed through Oracle Analytics Server's data
modeling layer, which defines business objects, subject areas, and hierarchies. No
dedicated AI context block is documented.

**What OAC has that maps to `context`:**

| `context` field   | Oracle Analytics equivalent           | Structured? |
| ----------------- | ------------------------------------- | ----------- |
| `instructions`    | None                                  | no          |
| `synonyms`        | None structured                       | no          |
| `examples`        | None                                  | no          |
| `verifiedAnswers` | None                                  | no          |
| `constraints`     | None                                  | no          |
| `description`     | Object descriptions in semantic model | yes         |

**Key gap:** OAC's semantic layer is deep but proprietary and Oracle-ecosystem-bound.
RFC-0038 would provide portable, standard AI context across any data platform.

---

### B.19 Pyramid Analytics

**Type:** Modern ABI platform. Proprietary. Gartner MQ Visionary (2025).

Pyramid offers an AI-assisted metrics layer with PQL (Process Query Language) and MDX
support, plus REST API for external consumption. Gartner notes its "metrics layer
leverages AI-assisted processes to suggest metric code." This is similar to IBM watsonx BI's
auto-suggestion approach but less documented for AI context specifically.

**What Pyramid has that maps to `context`:**

| `context` field   | Pyramid equivalent  | Structured? |
| ----------------- | ------------------- | ----------- |
| `instructions`    | None                | no          |
| `synonyms`        | None documented     | no          |
| `examples`        | None                | no          |
| `verifiedAnswers` | None                | no          |
| `constraints`     | None                | no          |
| `description`     | Metric descriptions | yes         |

**Key gap:** AI-assisted metric creation but no AI context fields for guiding
consumption of those metrics.

---

### B.20 Qlik

**Type:** BI and analytics platform. Proprietary. Gartner MQ Leader (2025).

Qlik Cloud Analytics features an associative analytics engine, NLQ via Qlik Answers
(natural language querying on unstructured data), and a metrics layer. The metrics layer
allows metric creation through a visual interface. No dedicated AI context block is
documented. Gartner notes limited NLQ functional differentiation compared to peers.

**What Qlik has that maps to `context`:**

| `context` field   | Qlik equivalent    | Structured? |
| ----------------- | ------------------ | ----------- |
| `instructions`    | None               | no          |
| `synonyms`        | None documented    | no          |
| `examples`        | None               | no          |
| `verifiedAnswers` | None               | no          |
| `constraints`     | None               | no          |
| `description`     | Field descriptions | yes         |

**Key gap:** Despite being a Leader, Qlik has no structured AI context mechanism.

---

### B.21 Salesforce / Tableau Semantics

**Type:** BI-native semantic layer. Proprietary (Salesforce). OSI co-lead. Gartner MQ Leader (2025).

Tableau Semantics is Salesforce's AI-infused semantic layer integrated into Salesforce
Data Cloud. It supports "agent enrichment capabilities" and "Semantic Learning" (improved
agent knowledge management, added 2025.2). Salesforce co-leads OSI alongside Snowflake
and follows OSI's `ai_context` pattern for interoperability.

In 2025, Tableau introduced "Tableau Next," featuring an AI semantic layer that "uses
agents to suggest and curate semantics."

**What Tableau Semantics has that maps to `context`:**

| `context` field   | Tableau Semantics equivalent                       | Structured? |
| ----------------- | -------------------------------------------------- | ----------- |
| `instructions`    | Agent enrichment / Semantic Learning (proprietary) | partial     |
| `synonyms`        | Via OSI alignment                                  | partial     |
| `examples`        | Not documented                                     | no          |
| `verifiedAnswers` | Not documented                                     | no          |
| `constraints`     | Not documented                                     | no          |
| `description`     | Field and metric descriptions                      | yes         |

**Relationship context:** Not documented beyond OSI alignment.

**Key gap:** Proprietary and tightly coupled to the Salesforce ecosystem. RFC-0038
provides an open, vendor-neutral alternative.

---

### B.22 SAP Analytics Cloud

**Type:** Enterprise BI and planning platform. Proprietary (SAP). Now part of SAP BDC.
Gartner MQ Visionary (2025).

SAP Analytics Cloud (SAC), now part of SAP Business Data Cloud (BDC), provides semantic
models that harmonize SAP and non-SAP data. It includes SAP Business AI for insights.
SAC's semantic models define dimensions, hierarchies, and measures within the SAP
ecosystem. No dedicated AI context block beyond descriptions is documented.

**What SAP has that maps to `context`:**

| `context` field   | SAP equivalent                         | Structured? |
| ----------------- | -------------------------------------- | ----------- |
| `instructions`    | None                                   | no          |
| `synonyms`        | None                                   | no          |
| `examples`        | None                                   | no          |
| `verifiedAnswers` | None                                   | no          |
| `constraints`     | None                                   | no          |
| `description`     | Object descriptions in semantic models | yes         |

**Key gap:** SAC's semantic layer is deep but SAP-ecosystem-bound. RFC-0038 would
provide portable context independent of the SAP stack.

---

### B.23 SAS Viya

**Type:** End-to-end data and AI platform. Proprietary (SAS). Gartner MQ Visionary (2025).

SAS Viya is primarily a data science and ML platform with BI capabilities via SAS Visual
Analytics. Gartner explicitly flags: "SAS lags other vendors for metrics layer capabilities;
metrics are created in views and are not accessible independently."

Viya Copilot (GA 2025) is an AI assistant for SAS code generation and model building,
powered by Microsoft Azure AI Foundry. This is developer productivity tooling, not
semantic AI context for data consumers. The SAS Information Catalog (since May 2024)
includes a Business Glossary that "adds a semantic layer of knowledge in natural language"
— the closest SAS comes to `context.synonyms`.

**What SAS Viya has that maps to `context`:**

| `context` field   | SAS Viya equivalent                                          | Structured? |
| ----------------- | ------------------------------------------------------------ | ----------- |
| `instructions`    | None (Viya Copilot is code-generation, not semantic context) | no          |
| `synonyms`        | Business Glossary terms (catalog-level, not schema-level)    | partial     |
| `examples`        | None                                                         | no          |
| `verifiedAnswers` | None                                                         | no          |
| `constraints`     | None                                                         | no          |
| `description`     | Dataset and column descriptions                              | yes         |

**Key gap:** Gartner confirms the weak metrics layer. SAS's semantic capabilities are
governance-catalog focused rather than AI-consumption-ready. RFC-0038 addresses the gap
Gartner flags.

---

### B.24 Sigma Computing

**Type:** Cloud-native BI platform with spreadsheet-like UI. Proprietary. Gartner MQ Niche Player (2025).

Sigma offers reusable Data Models as a semantic layer, with direct integration with
external semantic layers (Cube, dbt). Its NLQ/NLG capabilities are noted as "less mature"
by Gartner. No dedicated AI context block is documented.

**What Sigma has that maps to `context`:**

| `context` field   | Sigma equivalent               | Structured? |
| ----------------- | ------------------------------ | ----------- |
| `instructions`    | None                           | no          |
| `synonyms`        | None                           | no          |
| `examples`        | None                           | no          |
| `verifiedAnswers` | None                           | no          |
| `constraints`     | None                           | no          |
| `description`     | Column and metric descriptions | yes         |

---

### B.25 Sisense

**Type:** Embedded analytics platform. Proprietary. Gartner MQ Niche Player (2025).

Sisense has a semantic layer via drag-and-drop data combination and its proprietary
Sisense Knowledge Graph that "captures user analytic inquiries and usage, serving as
organizational memory by aggregating query usage." The Knowledge Graph is the most
notable AI-context-adjacent feature — it learns from usage patterns rather than
requiring authors to write context manually. No dedicated AI context block is documented.

**What Sisense has that maps to `context`:**

| `context` field   | Sisense equivalent                                  | Structured? |
| ----------------- | --------------------------------------------------- | ----------- |
| `instructions`    | None                                                | no          |
| `synonyms`        | None                                                | no          |
| `examples`        | None                                                | no          |
| `verifiedAnswers` | Knowledge Graph (usage-derived, not author-defined) | partial     |
| `constraints`     | None                                                | no          |
| `description`     | Field and dataset descriptions                      | yes         |

---

### B.26 Snowflake Semantic Views

**Type:** Platform-native semantic layer (Snowflake). GA since Snowflake Summit 2025.
Co-leads OSI initiative. Gartner MQ Honorable Mention (2025).

Snowflake Semantic Views define metrics, dimensions, and facts as schema-level objects.
AI context is handled via OSI's `ai_context` block (Snowflake is a primary OSI contributor).
No additional context fields beyond what OSI specifies.

**Relationship context:** Follows OSI — `ai_context` is available on relationships.

---

### B.27 Apache Superset

**Type:** Open-source BI tool. Apache 2.0. Community-governed (Apache Software Foundation).

Apache Superset has a lightweight semantic layer via SQL-first "Datasets" with metrics
and dimensions. SIP-182 (Semantic Layer proposal) is under discussion but not yet
implemented. There is no AI context block.

**What Superset has that maps to `context`:**

| `context` field   | Superset equivalent            | Structured? |
| ----------------- | ------------------------------ | ----------- |
| `instructions`    | None                           | no          |
| `synonyms`        | None                           | no          |
| `examples`        | None                           | no          |
| `verifiedAnswers` | None                           | no          |
| `constraints`     | None                           | no          |
| `description`     | Field descriptions on datasets | yes         |

---

### B.28 Tellius

**Type:** NLQ-first ABI platform. Proprietary. Gartner MQ Visionary (2025).

Tellius features Kaiya, an AI assistant for NLQ, NLG, and agentic analytics. Gartner
notes that "AI-enabled data prep features are limited to suggestions for synonyms and
descriptions" — identifying the exact gap RFC-0038 addresses. No dedicated AI context
block is documented; synonym suggestions are auto-generated, not author-defined.

**What Tellius has that maps to `context`:**

| `context` field   | Tellius equivalent                  | Structured? |
| ----------------- | ----------------------------------- | ----------- |
| `instructions`    | None                                | no          |
| `synonyms`        | Auto-suggested (not author-defined) | partial     |
| `examples`        | None                                | no          |
| `verifiedAnswers` | None                                | no          |
| `constraints`     | None                                | no          |
| `description`     | Auto-suggested descriptions         | partial     |

---

### B.29 ThoughtSpot / Spotter Semantics

**Type:** AI-native agentic analytics platform. Proprietary. OSI participant.
Gartner MQ Leader (2025).
**Released:** Spotter Semantics launched March 12, 2026.

ThoughtSpot is built around natural language search using ThoughtSpot Modeling Language
(TML). Spotter Semantics is its latest semantic layer, explicitly designed for AI agents.
It incorporates business logic, security rules, metric definitions, and model instructions
in a machine-readable form. ThoughtSpot explicitly aligns with OSI for interoperability
and supports MCP (Model Context Protocol) for connecting its governed semantic layer to
any AI agent or LLM.

ThoughtSpot supports synonyms on model fields and uses a Metrics Catalog to govern
metric definitions and prevent "metric drift." Spotter Semantics uses a deterministic
search-token architecture rather than LLM text-to-SQL.

**What ThoughtSpot has that maps to `context`:**

| `context` field   | ThoughtSpot / TML equivalent                    | Structured? |
| ----------------- | ----------------------------------------------- | ----------- |
| `instructions`    | Model instructions (Spotter Semantics)          | partial     |
| `synonyms`        | yes TML supports synonyms on columns            | yes         |
| `examples`        | Not documented                                  | no          |
| `verifiedAnswers` | Metrics Catalog (governed definitions, not Q&A) | partial     |
| `constraints`     | Via model instructions (informal)               | partial     |
| `description`     | Field and metric descriptions                   | yes         |

**Relationship context:** Not documented beyond model joins.

**Key finding:** ThoughtSpot's deterministic search-token approach needs less AI context
than LLM text-to-SQL tools, but its `synonyms` support is the strongest among proprietary
BI tools in this survey. The Metrics Catalog aligns conceptually with RFC-0038's
`verifiedAnswers` at the metric definition level.

---

### B.30 Strategy (MicroStrategy)

**Type:** Enterprise BI and analytics. Proprietary. Gartner MQ Visionary (2025).

Strategy (formerly MicroStrategy) offers a robust metrics layer that Gartner describes as
"top-tier" for composable analytics, "facilitating the creation, definition, governance
and distribution of business metrics across downstream analytics, data science and
business applications." Strategy AI adds AI-driven analytics. No dedicated AI context
block is documented despite the strong metrics foundation.

**What Strategy has that maps to `context`:**

| `context` field   | Strategy equivalent            | Structured? |
| ----------------- | ------------------------------ | ----------- |
| `instructions`    | None                           | no          |
| `synonyms`        | None documented                | no          |
| `examples`        | None                           | no          |
| `verifiedAnswers` | None                           | no          |
| `constraints`     | None                           | no          |
| `description`     | Metric and object descriptions | yes         |

**Key finding:** Strategy has the strongest metrics governance foundation among
proprietary BI tools surveyed (after Power BI), but no AI context layer on top of it.
RFC-0038's `context` block would naturally complement a Strategy-like metrics layer.

---

### B.31 Zoho Analytics

**Type:** Cloud BI platform. Proprietary. Gartner MQ Niche Player (2025).

Zoho Analytics offers AI-driven suggestions for data types, relationships, and
enrichment via its Zia AI assistant. It features NLQ and agentic AI assistants. No
dedicated AI context block is documented.

**What Zoho has that maps to `context`:**

| `context` field   | Zoho equivalent                | Structured? |
| ----------------- | ------------------------------ | ----------- |
| `instructions`    | None                           | no          |
| `synonyms`        | None                           | no          |
| `examples`        | None                           | no          |
| `verifiedAnswers` | None                           | no          |
| `constraints`     | None                           | no          |
| `description`     | Field and dataset descriptions | yes         |

---

## Appendix D: JSON Schema definition

> **Note:** `<name>` is a placeholder for the merged field that replaces `examples` and `verifiedAnswers`. The final name is pending — see the "Merged block" section of this RFC.

```json
"Context": {
  "description": "Structured guidance for AI agents and semantic tools",
  "oneOf": [
    {
      "type": "string",
      "description": "Shorthand: equivalent to providing instructions only"
    },
    {
      "type": "object",
      "properties": {
        "instructions": {
          "type": "string",
          "description": "Natural language guidance on how to use this entity"
        },
        "<name>": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "question": { "type": "string" },
              "answer":   { "type": "string" }
            },
            "required": ["question"],
            "additionalProperties": false
          },
          "description": "Canonical questions (entries without `answer`) and verified Q&A pairs (entries with `answer`). Merges the former `examples` and `verifiedAnswers` fields."
        },
        "constraints": {
          "type": "array",
          "items": { "type": "string" },
          "description": "Negative guidance: what AI agents must NOT do with this entity"
        }
      },
      "additionalProperties": false
    }
  ]
}
```
