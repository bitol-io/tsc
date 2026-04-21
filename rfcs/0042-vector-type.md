# Vector Type

Champion: Jean-Georges Perrin.

Authors: Jean-Georges Perrin.

Slack: TBD.

GitHub issue: TBD.

Applies to:
* [x] ODCS - Open Data Contract Standard
* [ ] ODPS - Open Data Product Standard
* [ ] OORS - Open Observability Results Standard
* [ ] OOCS - Open Orchestration and Control Standard
* [ ] OMMS - Open Maturity Model Standard
* [ ] OMDS - Open Metadata Difference Standard

## Summary

This RFC introduces a new `vector` value for `logicalType` in ODCS, together with a dedicated set of `logicalTypeOptions` (`dimensions`, `elementType`, `distanceMetric`, `normalized`, `embeddingModel`, `embeddingModelVersion`). Vectors are dense arrays of numbers used to represent embeddings produced by AI models (text, image, audio, multimodal). Standardizing the type lets data contracts describe embedding columns consistently across vector databases, warehouses, and AI tooling, so downstream consumers can run similarity search, retrieval-augmented generation (RAG), and semantic matching without reverse-engineering the producer's conventions.
