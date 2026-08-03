---
title: Design RAG System
date: 2026-08-04 00:11:35
tags: [system-design, RAG, AI]
categories: [RAG]
---

An enterprise-grade, production-ready knowledge base system leveraging **Agentic GraphRAG** combines semantic vector search, structured graph relationships, and autonomous multi-agent reasoning. This architecture eliminates context drift, handles multi-hop reasoning, and scales efficiently across millions of documents.

---

## 1. System Architecture Blueprint

{% mermaid graph TB %}
    A[Raw Documents<br/>PDF, Code, Wikis]
    B[Ingestion & Parsing Pipeline<br/>Layout-aware chunking<br/>Unstructured/LlamaParse]
    C[Entity Extraction]
    D[Hierarchical Chunking]
    E[Graph DB<br/>Neo4j]
    F[Vector DB<br/>Milvus/Qdrant]
    G[Agentic Orchestration Layer<br/>State Graph / Router / Planner]
    H[Graph Traversal]
    I[Vector Similarity Search]
    J[Cross-Encoder Reranker & Guardrails]
    K[Final Grounded Output + Citations]

    A --> B
    B --> C
    B --> D
    C --> E
    D --> F
    E --> G
    F --> G
    G --> H
    G --> I
    H --> J
    I --> J
    J --> K
{% endmermaid %}

---

## 2. Core Components & Detailed Key Points

### Component A: Ingestion & Knowledge Graph Construction

* **Layout-Aware Parsing:** Extracting text, nested tables, and markdown structures while maintaining parent-child document hierarchies.
* **Entity & Relation Extraction:** Using structured extraction schemas (via instructor or Pydantic with LLMs) to map out nodes (entities, concepts, components) and edges (dependencies, ownership, temporal relations).
* **Community Detection:** Running algorithms like Leiden or Louvain on the graph to cluster subgraphs and generating hierarchical summaries for broad corpus queries.

#### 💡 Interview Questions

1. **Chunking Strategies:** How do you handle document chunking for complex files containing nested tables and code blocks without breaking semantic context?
2. **Entity Resolution:** What strategies do you use for entity deduplication and disambiguation when multiple documents refer to the same entity using different aliases?
3. **Graph Summarization:** Explain how you implement hierarchical community summaries (similar to Microsoft GraphRAG) to handle global, corpus-level queries instead of narrow fact retrieval.
4. **Incremental Updates:** How do you handle real-time document updates, modifications, and deletions in both the graph database and vector index without requiring a costly full re-index?

---

### Component B: Hybrid Storage & Indexing Layer

* **Multi-Model Persistence:** Combining a Graph Database (e.g., Neo4j, NebulaGraph) for structural traversal, a Vector Database (e.g., Milvus, Qdrant) for dense semantic search, and a relational metadata store (Postgres) for ACL/RBAC permissions.
* **ID Synchronization:** Maintaining atomicity and consistency between vector chunks and graph node IDs during concurrent write operations.

#### 💡 Interview Questions

5. **Consistency Challenges:** What synchronization and consistency models do you use between your graph database and your vector index when updates occur?
6. **Metadata Filtering & RBAC:** How do you integrate user-level access control lists (ACLs) into the hybrid retrieval layer so users only retrieve authorized graph nodes and chunks?
7. **Scaling Graph Queries:** How do you optimize Cypher or Gremlin traversal queries to prevent memory exhaustion and latency spikes on deep multi-hop lookups?

---

### Component C: Agentic Orchestration & Retrieval Strategy

* **Stateful Agentic Workflows:** Utilizing frameworks like LangGraph or custom state machines where an LLM router dynamically decides the retrieval path.
* **Query Decomposition & Rewriting:** Breaking complex user prompts into sequential sub-tasks (e.g., separating temporal constraints from semantic entities).
* **Multi-Tool Execution:** The agent dynamically chooses between Vector Search (semantic similarity), Graph Traversal (structural relationships), and Full-Text Search (exact keyword IDs).

#### 💡 Interview Questions

8. **Agentic Loops & Latency:** Walk through an Agentic RAG workflow. How do you prevent infinite routing loops, excessive API calls, and runaway latency?
9. **Dynamic Routing:** How do you design a router classifier to decide when to trigger a graph traversal versus a dense vector search?
10. **Tool-Use Failures:** How do you handle errors or hallucinated parameters when an agent attempts to query structured APIs or SQL databases alongside the knowledge graph?
11. **Self-Correction (Self-RAG):** Explain how you implement critique loops to evaluate retrieved context sufficiency before passing it to the generator.

---

### Component D: Generation, Reranking & Guardrails

* **Cross-Encoder Reranking:** Filtering noisy chunks using models like BGE-Reranker or Cohere Rerank to optimize context window efficiency.
* **Context Window Optimization:** Mitigating the "lost in the middle" phenomenon by placing the most relevant retrieved nodes/chunks at the extreme edges of the context window.
* **Deterministic Guardrails:** Employing frameworks like NeMo Guardrails to block prompt injection, data exfiltration, and toxic generations.

#### 💡 Interview Questions

12. **Context Window Management:** How do you mitigate the "lost in the middle" phenomenon when passing dozens of graph neighborhoods and vector chunks into the context window?
13. **Hallucination Mitigation:** What mechanisms do you enforce to ensure the LLM strictly cites explicit graph node references or chunk IDs for every factual claim?
14. **Prompt Injection & Security:** How do you design an enterprise-grade security layer to prevent indirect prompt injection embedded inside unstructured ingestion documents?

---

### Component E: Observability & Automated Evaluation

* **The RAG Triad:** Continuously monitoring Context Relevance, Groundedness, and Answer Relevance using frameworks like Ragas, TruLens, or Phoenix.
* **Tracing & Debugging:** Tracking agent decision paths, token costs, and latency breakdowns per execution step.

#### 💡 Interview Questions

15. **Evaluation Metrics:** What automated metrics do you prioritize in a CI/CD pipeline to evaluate retrieval quality independently from generation quality?
16. **Cost & Latency Optimization:** How do you balance caching strategies (e.g., semantic caching of query embeddings and agent routing paths) with real-time freshness requirements?
