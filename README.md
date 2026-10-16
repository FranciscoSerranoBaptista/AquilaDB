# AquilaDB

**A multi-model graph database for research on people.** AquilaDB keeps documents, a property graph, full-text search and vector search in one Rust engine, behind one query language. The privacy rules live inside the query compiler, not in the application around it.

AquilaDB grew out of one need: psychological and coaching research that must answer questions about a population, over time, without exposing any one person. The same engine also serves a second purpose. It gives a language model graph context with provenance, so the model can read evidence but cannot invent it.

This repository is the public face of the project: the description and the public issues. The source code is private.

## Why it matters to an organisation

An organisation that invests in coaching or leadership development must answer three questions. Each one is a business risk if the answer is weak.

**1. Can we prove that it works?**

Most programmes measure what is easy: attendance, satisfaction, a score at the end. These numbers do not show that behaviour changed. AquilaDB keeps each record of development in one graph over time, so a programme can be read for change and not only for activity.

- **Change over time:** trend models over sessions show how each person moves, and how the population moves.
- **What works for whom:** effectiveness readouts rank interventions and coaching styles by the change that follows them.
- **Evidence that holds:** a pattern counts only when it is stable across runs. A study is pre-registered, so nobody searches the data until a good result appears.

This is the impact half of a return on investment. AquilaDB measures the change. The cost side stays in the finance systems. The readouts describe change and do not prove cause, unless the study design supports a causal claim.

**2. Can we do it without exposing anyone?**

Data about people's development is sensitive. A leak or a re-identified person damages trust, and it can be a breach of GDPR or of an agreement with a works council.

- **Privacy by design:** the engine enforces the rules, not the analyst. No result comes back for a group of fewer than five persons. Identity fields are encrypted and are never returned without write rights.
- **Consent:** a person without consent is removed from each analysis.
- **Audit:** every data change goes to a change log, and every study keeps its plan, its approval and the hash of its results.

**3. Can we use AI on it safely?**

A language model makes the data easy to question. It also invents numbers and quotes. In AquilaDB, a model reads only through the privacy rules, every number comes from the engine, and every quote is checked against its source.

## Heritage

AquilaDB is a re-implementation of the core of [ArangoDB](https://arangodb.com) in Rust. It keeps what ArangoDB does well and changes what a research graph cannot accept.

**What it keeps from ArangoDB:**

- **AQL.** The query language has broad parity with ArangoDB AQL: `FOR`, `FILTER`, `COLLECT`, `SORT`, `LIMIT`, graph traversals, `SHORTEST_PATH`, `K_SHORTEST_PATHS`, and the builtin function library. Each builtin follows the ArangoDB semantics, edge cases included. The engine checks this against the ArangoDB source, not against memory.
- **The multi-model idea.** A record is a document, a graph vertex, a search hit and a vector neighbour at the same time. You do not copy data between a document store, a graph store and a search engine.
- **The HTTP shape.** The API follows the ArangoDB layout: `/_api/cursor`, `/_api/document`, `/_api/collection`, `/_api/gharial` for named graphs.
- **An open grammar.** The AQL [tree-sitter grammar](https://github.com/FranciscoSerranoBaptista/tree-sitter-aql) is public.

**What it changes:**

| ArangoDB | AquilaDB | Why |
|---|---|---|
| C++, RocksDB, VelocyPack | Rust, [fjall](https://github.com/fjall-rs/fjall) LSM store, MessagePack | Memory safety, no C++ FFI, one build toolchain. |
| Cluster first | Single node, one writer | One writer is the throughput ceiling that research ingestion actually reaches. |
| Vectors stored with the documents | Vectors in a separate [usearch](https://github.com/unum-cloud/usearch) HNSW file, built in bulk | An HNSW insert in a B-tree or LSM store multiplies the write volume by about 100. |
| Full-text index by view | [tantivy](https://github.com/quickwit-oss/tantivy) BM25, opt-in per field | A full-text rebuild over all text fields caused out-of-memory failures at scale. |
| Access control per collection | Encryption tiers per field, and a cohort floor in the query compiler | Research on people needs rules on the result, not only on the collection. |

AquilaDB contains no ArangoDB source code. ArangoDB is a behavioural reference only.

## What makes it different

Most graph databases are general-purpose tools. They leave privacy, time, confidence and research method to the application. AquilaDB puts them in the engine.

| Concern | A general-purpose graph database | AquilaDB |
|---|---|---|
| **Small groups** | The application must remember to hide small counts. | The query compiler adds a **cohort floor** to every grouped `COLLECT` over clinical data. A group of fewer than 5 distinct persons does not come back, for every caller, admin included. |
| **Sensitive fields** | Access is per collection, or the application encrypts. | Each field has an **encryption tier**: `None`, `Coded`, `Narrative` or `Identity`. Narrative text is encrypted at rest with AES-256-GCM. Identity fields never leave without write rights. The compiler enforces the tiers before the query runs. |
| **Consent** | Not modelled. | A collection declares a consent field. A person without consent is removed from each analysis before it starts. The gate fails closed. |
| **Time** | Timestamps are plain properties. | Each node has a `TemporalContext`: creation time, validity window and a **decay function** (half-life, linear, step, sequence). A traversal can rank by recency and prune expired edges. |
| **Confidence** | A number on an edge, with no origin. | Each confidence value declares its basis: a human ruling, a validated judge, or counted evidence. **A language model never writes a confidence number.** `JUSTIFY(id)` returns the evidence chain and its weakest link. |
| **Retrieval for an LLM** | Vector search in one system, graph in another, glue code between them. | `GRAPH_RAG(...)` in one AQL call: vector seeds above a calibrated floor, graph expansion, reciprocal rank fusion, and rows that carry their provenance path. |
| **Novelty** | Not modelled. | `SURPRISE(id)` ranks neighbours that are close in meaning but far in the graph. These are the candidate bridges between two bodies of work. |
| **Research method** | Any caller can query until a result looks good. | A study is **pre-registered**. The plan is hashed before any evidence exists, an admin approves that hash, and the study runs once. A researcher key reaches only the approved studies. |
| **Analytics** | A community appears on each run, stable or not. | Community detection runs with **consensus stability** and **lineage** across runs. A pattern that does not hold across runs is reported as unstable, not as a finding. |

## Example: the cohort floor

A collection `coachees` holds six coachees in tech and one in pharma. An admin runs a grouped query:

```aql
FOR c IN coachees
  COLLECT industry = c.industry
  RETURN industry
```

The query does not ask for a count, but the compiler adds one. The pharma group has one person, so it does not come back. The response says why:

```json
{
  "result": ["tech"],
  "extra": {
    "warnings": [
      {
        "code": 2202,
        "message": "minimum-cohort (k-anonymity) floor of 5 applied: groups smaller than 5 were suppressed (#202)"
      }
    ]
  }
}
```

The same rule applies to the admin, to a rewritten query with a subquery, and to a vertex that a graph traversal reaches.

## The use case: psychological research under compliance

AquilaDB holds two kinds of data and treats them differently.

- **The research corpus** is one connected graph: works, chapters, ideas, concepts, persons and traditions, linked by citations, bridges, close matches and contradictions. Graph methods apply here: centrality, communities, paths, gaps between clusters.
- **Clinical data** is analysed through tables, time windows and trajectories, always under the cohort floor and the consent gate.

The compliance rules do not depend on a careful caller:

- **The cohort floor counts persons, not rows.** Five records from one client are a cohort of one, and the engine suppresses it.
- **Two phrasings give one answer.** The floor follows rows through subqueries and through graph traversals. A rewritten query does not escape it.
- **A suppressed result is visible.** The response carries a warning, so an empty group and a hidden group are different.
- **The floor can only go up.** Only a key scoped to one coach, with row access to that coach's own records, can lower it for that coach alone.
- **Every data change goes to a change log.** The log supports point-in-time restore and a live changefeed.

## A graph that gives context to a language model

AquilaDB ships an [MCP](https://modelcontextprotocol.io) server. A model reads the graph through tools such as `graph_rag`, `query`, `describe`, `justify`, `surprise`, `lit_search`, `lit_coverage`, `lit_citations`, `verify_quote` and `ground_quote`.

The rules for a model are strict:

- **The model articulates. It does not assert.** A model can describe a result in words. Every number, status and confidence comes from the engine or from a validation gate.
- **A quote must exist.** `ground_quote` binds a quoted passage to its source. `verify_quote` checks it again against the live source.
- **The context carries its source.** `GRAPH_RAG` admits a seed only above a calibrated similarity floor. Each row it returns carries its seed, its depth and the path from the seed.
- **The model works inside the floor.** A model with a researcher key sees only the results of approved studies, never the rows.

## The stack

| Part | Role |
|---|---|
| **AquilaDB** | The engine: the records, their relations over time, and the privacy rules. |
| **[KestrelAnalytics](https://github.com/FranciscoSerranoBaptista/KestrelAnalytics)** | Graph and population analytics: communities with stability, centrality, statistics, pre-registered studies. |
| **[PeacockUI](https://github.com/FranciscoSerranoBaptista/PeacockUI)** | The research console: start from a question, read ranked answers, keep the question with its finding. |

## Status

`v1.1.1`, the latest Aquila release (2026-10-12). Single node. In daily use for the author's own coaching research since May 2026.

Not in scope for v1: a distributed cluster, multiple concurrent writers, and the ArangoDB features that a single-node research graph does not need.

## Issues

Use the issues here for questions, ideas and collaboration requests. The working backlog is private.

**Do not post personal, clinical or coaching data in an issue.** An issue is public. To report a vulnerability, see [SECURITY.md](SECURITY.md).

© 2026 Francisco Serrano Baptista. All rights reserved.
