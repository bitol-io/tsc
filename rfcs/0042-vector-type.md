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

## Motivation

### Why are we doing this?

Vector embeddings have become a first-class data shape in modern analytics and AI stacks. Virtually every major database and warehouse now ships a native vector type — PostgreSQL/pgvector, Snowflake `VECTOR`, Databricks Vector Search, DuckDB VSS, Oracle `VECTOR`, SQL Server, MongoDB, Redis, Pinecone, Weaviate, Qdrant, Milvus, Chroma, Elasticsearch `dense_vector`, and OpenSearch `knn_vector`. Yet each exposes the type with different syntax, different element types, and different metric conventions.

ODCS today has no standard way to describe an embedding column. Authors can either use `logicalType: array` (losing dimensionality, element type, and metric) or stuff everything into `customProperties` (losing interoperability). Both defeat the interoperability goal of the standard.

### Use cases

1. **RAG pipelines**: A data contract declares an embedding column with its dimensions, element type, and the model that produced it, so a retriever knows how to query and which model to use for the query embedding.
2. **Vector database portability**: Teams migrate a dataset from pgvector to Snowflake `VECTOR`, Qdrant, or Databricks Vector Search. A standardized ODCS vector type lets tools generate the target DDL automatically.
3. **Similarity search**: A consumer discovers from the contract that an embedding uses cosine distance and is L2-normalized, so it can pick the matching index configuration without testing.
4. **Model provenance**: The contract records which embedding model and version produced the vectors, so that stale embeddings (produced by a retired model) can be detected and regenerated.
5. **AI-ready data products**: An ODCS contract used by an AI agent exposes not only structured columns but also embeddings for semantic retrieval, directly consumable by LLM-based tooling.

### Alignment with guiding values

- **Small standard over large**: This RFC adds one `logicalType` value and a handful of optional `logicalTypeOptions`. No new top-level structures.
- **Interoperability over readability**: The shape maps cleanly onto pgvector, Snowflake `VECTOR`, Databricks, Qdrant, Weaviate, Milvus, Pinecone, Elasticsearch `dense_vector`, and OpenSearch `knn_vector`.
- **Non-breaking**: `vector` is a new optional `logicalType` value. Existing contracts are unaffected.

## Design and examples

### Core concept

A vector is a fixed-dimension array of numeric values. The standard captures the three pieces of information every vector database requires: the dimensionality, the numeric element type, and (optionally) the intended similarity metric. Metadata about the model that produced the embedding is also carried so consumers can select a compatible query embedder.

### New `logicalType` value

Add `vector` alongside the existing logical types.

| Value    | Description                                                                      |
| -------- | -------------------------------------------------------------------------------- |
| `vector` | A fixed-dimension dense numeric array used for embeddings and similarity search. |

### `logicalTypeOptions` for `vector`

| Option                                    | Required | Type    | Description                                                                                                                                                                                                                                                                                |
| ----------------------------------------- | -------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `dimensions` \| `size` \| other?          | Yes      | integer | The fixed length of the vector. Positive integer. Examples: `384`, `768`, `1024`, `1536`, `3072`. Final field name is a discussion item — see "Discussion: naming of `dimensions`" below.                                                                                                  |
| `distanceMetric`                          | No       | string  | Intended similarity metric. One of `cosine`, `dotProduct`, `euclidean`, `hamming`, `manhattan`. Advisory — the physical index may differ.                                                                                                                                                  |
| `elementType` \| `physicalType` \| other? | No       | string  | Numeric type of each element. One of `bfloat16`, `binary`, `float16`, `float32` (default), `float64`, `int8`, `uint8`. `binary` means each element is one bit (used by binary-quantized vectors). Final field name is a discussion item — see "Discussion: naming of `elementType`" below. |
| `embeddingModel`                          | No       | string  | Identifier of the model used to produce the vectors (e.g., `cohere/embed-english-v3.0`, `openai/text-embedding-3-small`, `sentence-transformers/all-MiniLM-L6-v2`).                                                                                                                        |
| `embeddingModelVersion`                   | No       | string  | Version or revision of the embedding model, when the model identifier does not already carry one.                                                                                                                                                                                          |
| `normalized`                              | No       | boolean | `true` if vectors are L2-normalized before storage, which makes `cosine` and `dotProduct` equivalent. Default `false`.                                                                                                                                                                     |

All options follow the existing `logicalTypeOptions` pattern: camelCase keys, single lowercase words for enum values (with camelCase for multi-word values such as `dotProduct`), per the convention audited across the published standard.

### Example 1: Minimal — a single embedding column

```yaml
apiVersion: v3.2.0
kind: DataContract
id: product-catalog-embeddings
name: Product Catalog Embeddings

schema:
  - name: products
    physicalName: product_embeddings
    properties:
      - name: product_id
        logicalType: string
        primaryKey: true
        required: true
      - name: description
        logicalType: string
      - name: description_embedding
        logicalType: vector
        required: true
        logicalTypeOptions:
          dimensions|size|other?: 1536
```

### Example 2: Detailed — a normalized OpenAI embedding with cosine similarity

```yaml
apiVersion: v3.2.0
kind: DataContract
id: support-ticket-embeddings
name: Support Ticket Embeddings

schema:
  - name: tickets
    physicalName: ticket_embeddings
    description: "Text embeddings for customer support tickets, used by the RAG copilot."
    properties:
      - name: ticket_id
        logicalType: string
        primaryKey: true
        required: true
      - name: body
        logicalType: string
        required: true
        description: "Raw ticket body."
      - name: body_embedding
        logicalType: vector
        physicalType: vector(1536)
        required: true
        description: "Embedding of the ticket body."
        logicalTypeOptions:
          dimensions|size|other?: 1536
          elementType|physicalType|other?: float32
          distanceMetric: cosine
          normalized: true
          embeddingModel: openai/text-embedding-3-small
          embeddingModelVersion: "2024-01-25"
```

### Example 3: Multimodal and quantized vectors on the same contract

```yaml
schema:
  - name: media
    physicalName: media_embeddings
    properties:
      - name: media_id
        logicalType: string
        primaryKey: true
        required: true
      - name: image_embedding
        logicalType: vector
        required: true
        logicalTypeOptions:
          dimensions|size|other?: 768
          elementType|physicalType|other?: float32
          distanceMetric: cosine
          embeddingModel: openai/clip-vit-base-patch32
      - name: image_embedding_int8
        logicalType: vector
        description: "INT8-quantized copy for faster approximate search."
        logicalTypeOptions:
          dimensions|size|other?: 768
          elementType|physicalType|other?: int8
          distanceMetric: cosine
      - name: image_embedding_binary
        logicalType: vector
        description: "Binary-quantized copy for Hamming-distance search."
        logicalTypeOptions:
          dimensions|size|other?: 768
          elementType|physicalType|other?: binary
          distanceMetric: hamming
```

The primary `image_embedding` is marked `required: true` because the contract guarantees every media record has one. The two quantized copies (`image_embedding_int8`, `image_embedding_binary`) omit `required` — they are derived optimizations that may be populated on a lag, so an absent value is legitimate.

### Nullability and `required`

Marking a vector column `required: true` states that **every row must carry a non-null embedding**. This is the recommended default whenever the embedding is the basis for similarity search or RAG retrieval: a null vector silently excludes the row from matches, which is almost always unintended.

Leave `required: false` (the default) only when embeddings are populated asynchronously and null values are an expected, visible state to downstream consumers.

### Physical type mapping

The existing `physicalType` field continues to carry the target system's native syntax. The logical type describes shape and intent; the physical type describes the on-disk column type.

| Target                     | Typical `physicalType`                                                                                                                                                     |
| -------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Actian Vector / Ingres     | `FLOAT[1536]` (native array) + ANN extension                                                                                                                               |
| Apache Cassandra 5         | `VECTOR<FLOAT, 1536>`                                                                                                                                                      |
| Azure Cosmos DB            | JSON array property + `vectorEmbeddingPolicy` (`dimensions`, `dataType`, `distanceFunction`)                                                                               |
| BigQuery                   | `ARRAY<FLOAT64>` (+ `VECTOR_SEARCH`)                                                                                                                                       |
| Chroma                     | No column type — per-collection `dimension` and `hnsw:space` metadata.                                                                                                     |
| ClickHouse                 | `Array(Float32)` + `VECTOR INDEX ... TYPE hnsw` with `DIMENSION` and `DISTANCE`                                                                                            |
| Databricks (Delta / Unity) | `ARRAY<FLOAT>` (+ Vector Search index)                                                                                                                                     |
| DuckDB VSS                 | `FLOAT[1536]`                                                                                                                                                              |
| Elasticsearch              | `dense_vector` with `dims: 1536`, `element_type`, `similarity`                                                                                                             |
| IBM Db2                    | `VECTOR(1536, FLOAT32)` (Db2 13.2+)                                                                                                                                        |
| IBM watsonx.data / Milvus  | Collection field `FloatVector` / `Float16Vector` / `BinaryVector` + field-level `dim`                                                                                      |
| LanceDB                    | Arrow `FixedSizeList<float, 1536>` + index with `metric`.                                                                                                                  |
| Marqo                      | No column type — per-index `model` + `treat_urls_and_pointers_as_images` / `distance_metric`.                                                                              |
| Milvus / Zilliz            | Field-level type `FloatVector` (or `BFloat16Vector`, `BinaryVector`, `Float16Vector`, `Int8Vector`, `SparseFloatVector`) with `dim` parameter; `metric_type` on the index. |
| MongoDB Atlas              | Regular array field + `vectorSearch` index (`numDimensions`, `similarity`, `type`)                                                                                         |
| OpenSearch                 | `knn_vector` with `dimension: 1536`, `method`, `space_type`                                                                                                                |
| Oracle 23ai                | `VECTOR(1536, FLOAT32)`                                                                                                                                                    |
| Pinecone                   | No column type — per-index config: `dimension` and `metric` set when the index is created; upserts send the raw array.                                                     |
| PostgreSQL / pgvector      | `vector(1536)`                                                                                                                                                             |
| Qdrant                     | No column type — per-collection config: `vectors: { size: 1536, distance: Cosine }` (optionally named/multi-vector).                                                       |
| Redis                      | Hash/JSON field + `VECTOR` schema (`DIM`, `TYPE`, `DISTANCE_METRIC`)                                                                                                       |
| SingleStore                | `VECTOR(1536)` (or `BLOB` pre-9.0)                                                                                                                                         |
| Snowflake                  | `VECTOR(FLOAT, 1536)`                                                                                                                                                      |
| SQL Server 2025            | `VECTOR(1536)`                                                                                                                                                             |
| Vespa                      | `tensor<float>(x[1536])` (1-D) or higher-rank tensor fields with `distance-metric` in the schema.                                                                          |
| Weaviate                   | No column type — per-class config: `vectorIndexConfig.distance` + `vectorizer` (or bring-your-own `vectorizer: none`).                                                     |

### Relationship to existing types

`vector` is deliberately *not* modelled as `array` with a numeric element type. An array is variable-length and has no inherent similarity semantics; a vector is fixed-length with a required `dimensions` and an optional `distanceMetric`. Treating vectors as a distinct logical type:

- preserves interoperability with native `VECTOR` types across every modern database,
- lets tools validate dimensionality at contract time,
- avoids overloading `array` with vector-specific options.

### Discussion: value of `logicalType`

`vector` is the proposal. Alternatives considered:

- `densevector` — spelled out but verbose; inconsistent with single-word logical types.
- `embedding` — narrower; excludes non-AI vectors (feature hashes, sparse-to-dense projections).
- `ndarray` — too broad; implies multi-dimensional tensors, which this RFC does not cover.

`vector` is short, single-word (consistent with every other `logicalType` value in the published standard), and matches the nomenclature used by every database surveyed.

### Discussion: naming of `elementType`

Two candidate names are on the table for the field that captures the numeric type of each vector element:

- **`elementType`** — what this RFC currently uses. Unambiguous inside `logicalTypeOptions`; reads naturally ("each element of the vector is of type X"). No collision with existing ODCS fields.
- **`physicalType`** — reuses the ODCS field name already used at the property level (e.g., `physicalType: varchar(255)`). Lets authors who already know `physicalType` transfer the concept directly, and arguably what `float32`/`int8`/`binary` describe *is* the physical storage type of each element.

Trade-offs:

- `elementType` keeps `logicalTypeOptions` self-contained and avoids the same key name meaning two different things depending on nesting level.
- `physicalType` (inside `logicalTypeOptions`) could be confusing next to the outer `physicalType` on the property, which carries the target-system column syntax (e.g., `vector(1536)`). Readers may wonder which one wins.

The working group leans toward `elementType` for clarity but the TSC will pick one before approval. The YAML examples use the placeholder key `elementType|physicalType|other?` to make the open question visible; tools will of course only accept the key the TSC finally picks.

### Discussion: naming of `dimensions`

The field that captures the fixed length of a vector has several candidate names across the ecosystem, and the ODCS name is not yet settled:

- **`dimensions`** — what this RFC currently uses. Plural, matches the domain term ("a 1536-dimensional vector"), and is used by Elasticsearch (`dims`), Microsoft Azure AI Search (`dimensions`), Azure Cosmos DB (`dimensions`), and OpenSearch (`dimension`).
- **`size`** — used by Qdrant (`vectors: { size: 1536 }`) and implied by `VECTOR(1536)` / `FLOAT[1536]` in SQL dialects. Shorter, but less descriptive (size of what?).
- **`length`** — accurate (a vector is a 1-D array of this length) but unusual for embeddings.
- **`numDimensions`** — MongoDB Atlas uses this. More explicit than `dimensions` but longer.
- **`dim`** — Milvus and Redis use this abbreviation. Shortest, but ODCS prefers full words.

Trade-offs:

- `dimensions` is the clearest for readers who treat the embedding as a point in an N-dimensional space, which is the standard mental model in ML/retrieval.
- `size` is shorter and matches Qdrant, but in ODCS "size" could easily be misread as "byte size" or "row count."
- `numDimensions` would be the most explicit but adds noise where the surrounding context (`logicalType: vector`) already makes "dimensions of what" obvious.

The working group leans toward `dimensions` but the TSC will pick one before approval. The YAML examples use the placeholder key `dimensions|size|other?` to make the open question visible.

## Alternatives

### Alternative A: Reuse `array` with `logicalTypeOptions` (rejected)

Model vectors as `logicalType: array` with `dimensions`, `elementType`, etc., in `logicalTypeOptions`.

**Rejected because**: arrays are variable-length and have no similarity semantics. Consumers would have to inspect `logicalTypeOptions` to discover that a given array is really a vector, and validators could not cheaply distinguish the two. Native vector types in target systems are distinct from arrays; the logical type should reflect that.

### Alternative B: Rely entirely on `customProperties` (rejected)

Keep ODCS silent about vectors and let each tool namespace vector metadata under `customProperties`.

**Rejected because**: vector columns are a ubiquitous and permanent feature of the modern data stack. Leaving them outside the standard defeats the interoperability goal — every tool would re-invent its own conventions, exactly the problem ODCS exists to solve.

### Alternative C: Sparse vectors (deferred)

Sparse vectors (common in learned-sparse retrieval such as SPLADE) are a separate shape: pairs of `(index, value)` rather than a dense array.

**Deferred because**: a separate RFC will add a `sparseVector` logical type if there is demand. The `vector` type in this RFC is dense by definition.

### Alternative D: Multi-dimensional tensors (rejected for now)

ML feature stores sometimes need 2-D or 3-D tensors (e.g., image patches, time-series windows).

**Rejected for now because**: the primary demand is 1-D vectors for similarity search. Tensors can be addressed by a future RFC (`tensor` with a `shape` option) if needed.

## Decision

> The decision made by the TSC.

TBD.

## Consequences

- **Non-breaking**: `vector` is a new optional `logicalType` value; no existing contract is affected.
- **Interoperable**: Maps cleanly onto every major vector database and warehouse vector type surveyed.
- **Composable with other RFCs**: Works with [RFC-0034](0034-measures-and-dimensions.md) (a vector column can coexist with measures and dimensions), [RFC-0038](0038-context.md) (`context.instructions` can describe how an AI agent should use the embedding), and [RFC-0041](0041-synonyms.md) (synonyms on the embedding column help natural-language discovery).
- **Validator impact**: Contract validators SHOULD require `dimensions` whenever `logicalType: vector`.
- **Future extensibility**: Sparse vectors, tensors, and index-specific options (HNSW parameters, IVF list counts, quantization details) can be layered on via follow-up RFCs or `customProperties`.

## References

- [BigQuery VECTOR_SEARCH](https://cloud.google.com/bigquery/docs/reference/standard-sql/search_functions#vector_search) — embeddings stored as `ARRAY<FLOAT64>`.
- [ChromaDB](https://docs.trychroma.com/) — collection-level dimensions and distance.
- [Databricks Vector Search](https://docs.databricks.com/en/generative-ai/vector-search.html) — embedding columns and managed indexes.
- [DuckDB VSS extension](https://duckdb.org/docs/extensions/vss) — `FLOAT[N]` with HNSW indexes.
- [Elasticsearch dense_vector](https://www.elastic.co/guide/en/elasticsearch/reference/current/dense-vector.html) — `dense_vector` with `dims`, `similarity`, `element_type`.
- [Milvus data types](https://milvus.io/docs/schema.md) — `BFloat16Vector`, `BinaryVector`, `Float16Vector`, `FloatVector`, `SparseFloatVector`.
- [MongoDB Atlas Vector Search](https://www.mongodb.com/docs/atlas/atlas-vector-search/) — `vectorSearch` index with `numDimensions`, `similarity`, `type`.
- [OpenSearch k-NN](https://opensearch.org/docs/latest/search-plugins/knn/knn-index/) — `knn_vector`.
- [Oracle 23ai VECTOR](https://docs.oracle.com/en/database/oracle/oracle-database/23/vecse/index.html) — `VECTOR(dim, format)`.
- [Pinecone data model](https://docs.pinecone.io/docs/indexes) — index-level dimensions and metric.
- [PostgreSQL pgvector](https://github.com/pgvector/pgvector) — `vector(N)` column type with cosine, L2, inner-product operators.
- [Qdrant Collections](https://qdrant.tech/documentation/concepts/collections/) — `vectors: { size, distance }`.
- [Redis Stack vector search](https://redis.io/docs/latest/develop/interact/search-and-query/advanced-concepts/vectors/) — `VECTOR` field with `DIM`, `DISTANCE_METRIC`, `TYPE`.
- [Snowflake VECTOR data type](https://docs.snowflake.com/en/sql-reference/data-types-vector) — `VECTOR(type, dim)`.
- [SQL Server 2025 vector type](https://learn.microsoft.com/en-us/sql/t-sql/data-types/vector-data-type) — `VECTOR(N)`.
- [Weaviate Schema](https://weaviate.io/developers/weaviate/config-refs/schema) — class-level `vectorizer` and `vectorIndexConfig`.
- [RFC-0002 Data Types](approved/odcs-v3.0.0/0002-types.md) — ODCS logical/physical types.
- [RFC-0017 New Date Types](approved/odcs-v3.1.0/0017-new-date-types.md) — model for adding new logical types.

## Appendix: Vendor Landscape for Vector Types

A non-exhaustive catalog of vector-type implementations, demonstrating the breadth of native support in databases, warehouses, and dedicated vector stores.

### Warehouses, lakehouses, and cloud data platforms

| Vendor           | Product                                                                                                                           | Native vector type                                | Dimensions | Element types                  | Metrics                            |
| ---------------- | --------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------- | ---------- | ------------------------------ | ---------------------------------- |
| **AWS**          | [Amazon Aurora PostgreSQL / RDS (pgvector)](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_PostgreSQL.html#pgvector) | `vector(N)`                                       | ≤ 16000    | `float4`, fp16, binary, sparse | cosine, Hamming, inner product, L2 |
| **AWS**          | [Amazon MemoryDB / Kendra / Neptune Analytics](https://aws.amazon.com/memorydb/) — various ANN offerings                          | Collection/index-level                            | varies     | `float32`                      | cosine, dot product, L2            |
| **AWS**          | [Amazon OpenSearch Service](https://opensearch.org/docs/latest/search-plugins/knn/)                                               | `knn_vector`                                      | ≤ 16000    | `binary`, `byte`, `float`      | cosine, hamming, inner product, L2 |
| **Databricks**   | [Mosaic AI Vector Search](https://docs.databricks.com/en/generative-ai/vector-search.html)                                        | `ARRAY<FLOAT>` column + Delta Sync / Direct index | any        | `float32`                      | cosine, dot product, L2            |
| **Google Cloud** | [BigQuery `VECTOR_SEARCH`](https://cloud.google.com/bigquery/docs/reference/standard-sql/search_functions#vector_search)          | `ARRAY<FLOAT64>` column + IVF / TreeAH index      | any        | `FLOAT64`                      | cosine, dot product, L2            |
| **Google Cloud** | [Vertex AI Vector Search (Matching Engine)](https://cloud.google.com/vertex-ai/docs/vector-search/overview)                       | Separate index service; JSONL upserts             | any        | `float32`                      | cosine, dot product, L2            |
| **Snowflake**    | [Snowflake Cortex + VECTOR](https://docs.snowflake.com/en/sql-reference/data-types-vector)                                        | `VECTOR(FLOAT, N)` / `VECTOR(INT, N)`             | ≤ 4096     | `FLOAT`, `INT`                 | cosine, inner product, L2          |

### Operational and general-purpose databases

| Vendor          | Product                                                                                                                                                                                 | Native vector type                                              | Dimensions | Element types                                                | Metrics                                       |
| --------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------- | ---------- | ------------------------------------------------------------ | --------------------------------------------- |
| **Actian**      | [Actian Vector](https://www.actian.com/databases/vector/) / [Actian Data Platform](https://www.actian.com/data-platform/)                                                               | Native `FLOAT[N]` (Vector) / ANN extensions                     | any        | `float32`                                                    | cosine, dot product, L2                       |
| **Cassandra**   | [Apache Cassandra 5](https://cassandra.apache.org/doc/latest/cassandra/developing/cql/vector-search.html)                                                                               | `VECTOR<FLOAT, N>`                                              | any        | `FLOAT`                                                      | cosine, dot product, euclidean                |
| **ClickHouse**  | [ClickHouse ANN](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/annindexes)                                                                                      | `Array(Float32)` + `VECTOR INDEX ... TYPE hnsw`                 | any        | `Float32`, `Float64`                                         | cosine, L2                                    |
| **CockroachDB** | [CockroachDB Vector Search](https://www.cockroachlabs.com/docs/stable/vector-search)                                                                                                    | `VECTOR(N)`                                                     | any        | `float32`                                                    | cosine, inner product, L2                     |
| **Couchbase**   | [Couchbase Capella / Server 7.6+](https://docs.couchbase.com/server/current/vector-search/vector-search.html)                                                                           | JSON array + Vector Search index                                | ≤ 4096     | `float32`                                                    | cosine, dot product, L2                       |
| **DataStax**    | [DataStax Astra DB (JVector)](https://www.datastax.com/products/datastax-astra)                                                                                                         | `VECTOR<FLOAT, N>` (Cassandra-based)                            | any        | `FLOAT`                                                      | cosine, dot product, euclidean                |
| **DuckDB**      | [VSS extension](https://duckdb.org/docs/extensions/vss)                                                                                                                                 | `FLOAT[N]`                                                      | any        | `FLOAT`                                                      | cosine, inner product, L2                     |
| **IBM**         | [IBM Cloudant / Cloud Databases for Elasticsearch](https://www.ibm.com/cloud/databases-for-elasticsearch)                                                                               | `dense_vector`                                                  | ≤ 4096     | `bit`, `byte`, `float`                                       | cosine, dot product, L2                       |
| **IBM**         | [IBM Db2 13.2](https://www.ibm.com/docs/en/db2/13.2?topic=search-vector)                                                                                                                | `VECTOR(N, datatype)`                                           | ≤ 32000    | `BINARY`, `FLOAT32`, `FLOAT64`, `INT8`                       | cosine, dot product, euclidean                |
| **IBM**         | [IBM watsonx.data (with Milvus)](https://www.ibm.com/docs/en/watsonx/w-and-w/2.0.x?topic=data-vector-database)                                                                          | Milvus collections (see below)                                  | see Milvus | see Milvus                                                   | see Milvus                                    |
| **Microsoft**   | [Azure AI Search](https://learn.microsoft.com/en-us/azure/search/vector-search-overview)                                                                                                | Field of type `Collection(Edm.Single)` / `Collection(Edm.Half)` | ≤ 4096     | `float16`, `float32`, `int8`, `uint8`, packed `bit`          | cosine, dot product, euclidean                |
| **Microsoft**   | [Azure Cosmos DB for MongoDB / PostgreSQL](https://learn.microsoft.com/en-us/azure/cosmos-db/mongodb/vcore/vector-search)                                                               | Mongo/PG vector search                                          | ≤ 16000    | `float32`                                                    | cosine, dot product, euclidean                |
| **Microsoft**   | [Azure Cosmos DB for NoSQL](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/vector-search)                                                                                      | JSON array + `vectorEmbeddingPolicy`                            | ≤ 4096     | `float32`, `int8`, `uint8`                                   | cosine, dot product, euclidean                |
| **Microsoft**   | [Microsoft Fabric](https://learn.microsoft.com/en-us/fabric/database/sql/vector-search-overview)                                                                                        | Fabric SQL `VECTOR(N)`                                          | ≤ 1998     | `float32`                                                    | cosine, dot product, L2                       |
| **Microsoft**   | [SQL Server 2025 / Azure SQL](https://learn.microsoft.com/en-us/sql/t-sql/data-types/vector-data-type)                                                                                  | `VECTOR(N)`                                                     | ≤ 1998     | `float32`                                                    | cosine, dot product, L2                       |
| **MongoDB**     | [MongoDB Atlas Vector Search](https://www.mongodb.com/docs/atlas/atlas-vector-search/)                                                                                                  | Array field + `vectorSearch` index                              | ≤ 4096     | `float32`, `int1`, `int8`                                    | cosine, dotProduct, euclidean                 |
| **Oracle**      | [Oracle Database 23ai AI Vector Search](https://docs.oracle.com/en/database/oracle/oracle-database/23/vecse/)                                                                           | `VECTOR(dim, type)`                                             | ≤ 65535    | `BINARY`, `FLOAT32`, `FLOAT64`, `INT8`                       | cosine, dot, euclidean, hamming, manhattan    |
| **Oracle**      | [Oracle MySQL HeatWave (Lakehouse)](https://dev.mysql.com/doc/heatwave/en/mys-hw-lakehouse-vector-stores.html)                                                                          | `VECTOR` column                                                 | ≤ 16383    | `float32`                                                    | cosine, dot product, L2                       |
| **PostgreSQL**  | [pgvector](https://github.com/pgvector/pgvector)                                                                                                                                        | `vector(N)`, `halfvec(N)`, `bit(N)`, `sparsevec(N)`             | ≤ 16000    | `bit`, `float4`, `halfvec` (fp16), `sparsevec`               | cosine, Hamming, inner product, Jaccard, L2   |
| **PostgreSQL**  | [pgvectorscale](https://github.com/timescale/pgvectorscale)                                                                                                                             | `vector(N)` + StreamingDiskANN index                            | ≤ 16000    | `float4`                                                     | cosine, inner product, L2                     |
| **Redis**       | [Redis Stack / Redis 8](https://redis.io/docs/latest/develop/interact/search-and-query/advanced-concepts/vectors/)                                                                      | `VECTOR` schema on hash/JSON fields                             | any        | `BFLOAT16`, `FLOAT16`, `FLOAT32`, `FLOAT64`, `INT8`, `UINT8` | cosine, inner product, L2                     |
| **Rockset**     | [Rockset Vector Search](https://docs.rockset.com/documentation/docs/vector-search)                                                                                                      | `array<float>`                                                  | any        | `float32`                                                    | cosine, dot product, L2                       |
| **SAP**         | [SAP HANA Cloud Vector Engine](https://help.sap.com/docs/hana-cloud-database/sap-hana-cloud-sap-hana-database-vector-engine-guide/sap-hana-cloud-sap-hana-database-vector-engine-guide) | `REAL_VECTOR(N)`                                                | ≤ 65000    | `float32`                                                    | cosine, euclidean                             |
| **SingleStore** | [SingleStoreDB](https://docs.singlestore.com/cloud/reference/sql-reference/data-types/vector-type/)                                                                                     | `VECTOR(N)` (+ legacy `BLOB` encoding)                          | any        | `F32`, `F64`, `I8`, `I16`, `I32`, `I64`                      | cosine, dot product, euclidean, inner product |
| **Teradata**    | [Teradata VantageCloud](https://docs.teradata.com/r/Enterprise_IntelliFlex_VMware_and_AWS/Teradata-VantageCloud-Lake)                                                                   | `ARRAY(FLOAT)` + `VectorDistance` functions                     | any        | `FLOAT`                                                      | cosine, euclidean, manhattan                  |
| **Yugabyte**    | [YugabyteDB + pgvector](https://docs.yugabyte.com/preview/explore/ysql-language-features/pg-extensions/extension-pgvector/)                                                             | `vector(N)`                                                     | ≤ 16000    | `float4`, binary, fp16, sparse                               | cosine, inner product, L2                     |

### Search engines with vector support

| Vendor          | Product                                                                                                           | Native vector type                      | Dimensions | Element types                 | Metrics                                    |
| --------------- | ----------------------------------------------------------------------------------------------------------------- | --------------------------------------- | ---------- | ----------------------------- | ------------------------------------------ |
| **Apache Solr** | [Solr DenseVectorField](https://solr.apache.org/guide/solr/latest/query-guide/dense-vector-search.html)           | `DenseVectorField`                      | any        | `byte`, `float32`             | cosine, dot product, euclidean             |
| **Elastic**     | [Elasticsearch `dense_vector`](https://www.elastic.co/guide/en/elasticsearch/reference/current/dense-vector.html) | `dense_vector`                          | ≤ 4096     | `bit`, `byte`, `float`        | cosine, dot product, L2, max inner product |
| **Meilisearch** | [Meilisearch Vector Search](https://www.meilisearch.com/docs/learn/ai_powered_search/vector_search)               | `_vectors` object per document          | any        | `float32`                     | cosine                                     |
| **OpenSearch**  | [OpenSearch k-NN](https://opensearch.org/docs/latest/search-plugins/knn/knn-index/)                               | `knn_vector`                            | ≤ 16000    | `binary`, `byte`, `float`     | cosine, hamming, inner product, L2         |
| **Typesense**   | [Typesense Vector Search](https://typesense.org/docs/latest/api/vector-search.html)                               | `float[]` field with `num_dim`          | any        | `float32`                     | cosine, inner product, L2                  |
| **Vespa**       | [Vespa tensor fields](https://docs.vespa.ai/en/tensor-user-guide.html)                                            | `tensor<float>(x[N])` (and higher-rank) | any        | `bfloat16`, `float32`, `int8` | cosine, dot product, euclidean             |

### Dedicated vector stores

| Vendor          | Product                                                                    | Storage unit                                          | Element types                                                                                       | Metrics                                               |
| --------------- | -------------------------------------------------------------------------- | ----------------------------------------------------- | --------------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| **Chroma**      | [Chroma](https://docs.trychroma.com/)                                      | Collection with `dimension` and `hnsw:space` metadata | `float32`                                                                                           | cosine, inner product, L2                             |
| **LanceDB**     | [LanceDB](https://lancedb.github.io/lancedb/)                              | Arrow table with `FixedSizeList<float, N>`            | `float16`, `float32`                                                                                | cosine, dot product, L2                               |
| **Marqo**       | [Marqo](https://docs.marqo.ai/latest/)                                     | Index with model + distance metric                    | `float32`                                                                                           | cosine, dot product, euclidean                        |
| **Milvus**      | [Milvus / Zilliz Cloud](https://milvus.io/docs/schema.md)                  | Collection with typed vector fields (`dim`)           | `BFloat16Vector`, `BinaryVector`, `Float16Vector`, `FloatVector`, `Int8Vector`, `SparseFloatVector` | cosine, hamming, inner product, jaccard, L2           |
| **Pinecone**    | [Pinecone](https://docs.pinecone.io/docs/indexes)                          | Index (serverless or pod) with per-index `dimension`  | `float32`, `int8`                                                                                   | cosine, dot product, euclidean                        |
| **Qdrant**      | [Qdrant](https://qdrant.tech/documentation/concepts/collections/)          | Collection with named `vectors: { size, distance }`   | binary (`bool`), `float32`, multi-vector, `uint8`                                                   | cosine, dot, euclidean, manhattan                     |
| **Turbopuffer** | [Turbopuffer](https://turbopuffer.com/docs)                                | Namespace with per-namespace dimension and distance   | `float32`                                                                                           | cosine, dot product, euclidean                        |
| **USearch**     | [Unum USearch](https://github.com/unum-cloud/usearch)                      | Single-file index                                     | binary, `float16`, `float32`, `float64`, `int8`                                                     | cosine, hamming, inner product, jaccard, L2, tanimoto |
| **Vald**        | [Vald (Yahoo! Japan)](https://vald.vdaas.org/docs/overview/configuration/) | Cluster with NGT-based index                          | `float32`, `uint8`                                                                                  | cosine, hamming, inner product, jaccard, L2           |
| **Weaviate**    | [Weaviate](https://weaviate.io/developers/weaviate/config-refs/schema)     | Class with `vectorizer` + `vectorIndexConfig`         | binary, `float32`, `int8`, multi-vector                                                             | cosine, dot product, euclidean, hamming, manhattan    |

### Summary

Every target system surveyed agrees on the three pieces of metadata this RFC standardizes: **dimensions**, **element type**, and **distance metric**. The disagreement is purely syntactic. A `logicalType: vector` with `logicalTypeOptions` covering these three fields (plus optional `normalized`, `embeddingModel`, `embeddingModelVersion`) is sufficient to round-trip with every vector store surveyed, and positions ODCS as the portable interchange format for AI-ready data.

## Appendix: Changelog

All notable changes to this RFC are recorded here. Dates are `YYYY-MM-DD`. Entries are listed newest-first.

| Date | Author | Change |
| ---- | ------ | ------ |
