# Knowledge and Evaluation Data

The RAG evaluation system works with four distinct categories of data. Understanding what each is, where it lives, and which services can access it is key to understanding how the system produces meaningful evaluation scores.

See also: [Evaluation Flow](evaluation-flow.md) for how the system uses this data during evaluation.

---

## 1. Knowledge Base

**Purpose:** The ingested documents that the RAG system draws on to answer questions. This is the system under test — the quality of RAG answers depends directly on the quality and consistency of the knowledge base.

**Stored in:** PostgreSQL + pgvector (chunks and embeddings), AWS S3 (source documents)
**Managed by:** Data Service

### Characteristics

| Aspect | Details |
|--------|---------|
| **Source** | Documents uploaded by a knowledge manager via the UI Service |
| **Ingestion** | Data Service chunks documents, generates embeddings via Amazon Titan Embed Text v2, and stores both the chunk text and embedding vector in PostgreSQL |
| **Original files** | Stored in S3; the PostgreSQL record holds the chunk and embedding, not the full document |
| **Search** | Data Service exposes a RAG search API — given a query, it returns the most semantically relevant chunks using vector similarity |

### Service access

| Service | Access | Purpose |
|---------|:------:|---------|
| Data Service | Read/Write | Ingests documents, stores chunks and embeddings, serves RAG search API |
| Runtime Service | Read (via API) | Calls the Data Service RAG search API during evaluation to retrieve relevant chunks |
| UI Service | Read (via API) | Displays knowledge base contents and ingestion status |

---

## 2. Knowledge Snapshots

**Purpose:** An immutable, point-in-time view of the knowledge base. Evaluations run against a snapshot — not against live, mutable knowledge base state.

**Stored in:** AWS S3 (snapshot metadata and document references)
**Managed by:** Data Service

### Characteristics

| Aspect | Details |
|--------|---------|
| **Created** | When a knowledge manager triggers ingestion after uploading new content |
| **Immutability** | Once created, a snapshot is never modified — subsequent uploads create new snapshots |
| **Purpose** | Ensures that evaluation results are reproducible and attributable to a specific state of the knowledge base |
| **Lifecycle** | A snapshot is created, optionally evaluated, and then either activated (making it the live knowledge base) or discarded |

### Why snapshots matter

Without snapshots, running an evaluation before and after a knowledge base update could produce different results simply because the underlying data changed between runs. Snapshots pin the evaluation to a specific state, making scores directly comparable across runs.

---

## 3. Evaluation Dataset

**Purpose:** The set of questions and known-good answers (ground truths) that form the test suite for judging a knowledge base snapshot.

A meaningful evaluation requires this dataset to be curated in advance. The quality of the evaluation is bounded by the quality of the test dataset — poor questions or inaccurate ground truths will produce unreliable scores regardless of the judge model or rubric.

**Managed by:** UI Service

### Characteristics

| Aspect | Details |
|--------|---------|
| **Questions** | Representative queries that a user of the RAG system would ask |
| **Ground truths** | The assumed ideal or correct answer for each question |
| **Scope** | Questions should be answerable from the knowledge base being evaluated |
| **Reuse** | The same dataset can be used across multiple evaluation runs and multiple snapshots, enabling trend comparison over time |

### Semantic equivalence

Because RAG responses are generative, ground truths are not expected to match responses verbatim. The LLM judge evaluates semantic equivalence guided by the rubric — "The capital of France is Paris" and "Paris is the capital city of France" should score identically.

---

## 4. Evaluation Results

**Purpose:** The scored outcomes of each evaluation run — persisted per question, per rubric, and per judge model.

**Stored in:** MongoDB
**Managed by:** Runtime Service

### Characteristics

| Aspect | Details |
|--------|---------|
| **Granularity** | One result record per `(run_id, question_id, rubric_id, model_id)` combination |
| **Contents** | Question, ground truth, RAG-retrieved answer, rubric used, judge model, score, and reasoning |
| **Aggregation** | Scores can be averaged across questions to produce a run-level quality score per rubric-model combination |
| **Retention** | Results accumulate across runs, enabling trend analysis — tracking whether knowledge base quality improves or degrades over time |

### Service access

| Service | Access | Purpose |
|---------|:------:|---------|
| Runtime Service | Read/Write | Persists results during evaluation; serves results via REST API |
| UI Service | Read (via API) | Displays scores, reasoning, and trends on the evaluation dashboard |
| Data Service | — | Not involved in evaluation results |

---

## Storage and Retrieval Summary

| Data | Stored In | Managed By | Access Pattern |
|---|---|---|---|
| Knowledge chunks & embeddings | PostgreSQL + pgvector | Data Service | Vector similarity search at evaluation time |
| Source documents | AWS S3 | Data Service | Retrieved by the Data Service during ingestion |
| Knowledge snapshots | AWS S3 | Data Service | Referenced by run_id when scoping RAG search |
| Evaluation dataset (questions & ground truths) | MongoDB | UI / Runtime Service | Loaded at evaluation start; passed to the queue worker |
| Evaluation results | MongoDB | Runtime Service | Written per question × rubric × model; read for display and aggregation |
