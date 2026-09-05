# Advanced Agentic RAG with Corrective RAG and Self-RAG

An industry-oriented **Agentic Retrieval-Augmented Generation (RAG)** implementation built with **LangGraph, LangChain, Pinecone, Tavily, Pydantic, and Groq**.

This project demonstrates how a traditional RAG pipeline can be evolved into an adaptive, evaluation-driven system using:

* **Agentic RAG** for intelligent workflow orchestration
* **Corrective RAG (CRAG)** for retrieval evaluation and recovery
* **Self-RAG** for generated-answer evaluation
* **Query rewriting** for failed retrieval
* **Private Knowledge Base retrieval** using Pinecone
* **Web fallback** using Tavily
* **Evidence grading** for KB and web results
* **Groundedness, completeness, and citation evaluation**
* **LangGraph conditional routing and feedback loops**

---

## Architecture

The current notebook implements the following workflow:

```text
                         ┌─────────────────────┐
                         │     USER QUERY      │
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │   AGENTIC ROUTER    │
                         │      LangGraph      │
                         └──────────┬──────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
                   KB                            Direct
                    │                               │
                    ▼                               ▼
          ┌──────────────────┐             ┌─────────────────┐
          │ Pinecone         │             │   Direct LLM    │
          │ KB Retrieval     │             │    Response     │
          └────────┬─────────┘             └────────┬────────┘
                   │                                │
                   ▼                                ▼
          ┌──────────────────┐                     END
          │ KB Evidence      │
          │ Grader           │
          └────────┬─────────┘
                   │
             ┌─────┴─────┐
             │           │
           GOOD         WEAK
             │           │
             ▼           ▼
      ┌────────────┐  ┌─────────────────┐
      │ KB Answer  │  │ Query Rewriter  │
      │ Generator  │  │     (CRAG)      │
      └─────┬──────┘  └────────┬────────┘
            │                  │
            │                  ▼
            │           ┌──────────────┐
            │           │ KB Retrieval │
            │           │    Retry     │
            │           └──────┬───────┘
            │                  │
            │                  ▼
            │           ┌──────────────┐
            │           │ KB Evidence  │
            │           │    Grader     │
            │           └──────┬───────┘
            │                  │
            │             Still Weak
            │                  │
            │                  ▼
            │           ┌──────────────┐
            │           │ Web Search   │
            │           │    Tavily    │
            │           └──────┬───────┘
            │                  │
            │                  ▼
            │           ┌──────────────┐
            │           │ Web Evidence │
            │           │    Grader     │
            │           └──────┬───────┘
            │                  │
            │                  ▼
            │           ┌──────────────┐
            │           │ Web Answer   │
            │           │  Generator   │
            │           └──────┬───────┘
            │                  │
            └────────┬─────────┘
                     │
                     ▼
             ┌───────────────────┐
             │    SELF-RAG       │
             │   ANSWER GRADER   │
             │                   │
             │ Groundedness      │
             │ Completeness      │
             │ Citation Quality  │
             └────────┬──────────┘
                      │
                ┌─────┴─────┐
                │           │
              PASS         RETRY
                │           │
                ▼           ▼
               END    Rewrite / Retrieve
```

See [`architecture_diagram.md`](architecture_diagram.md) for the architecture documentation.

---

# 1. Project Overview

Traditional RAG generally follows:

```text
User Query
    ↓
Retrieve Documents
    ↓
Generate Answer
```

This approach can fail when:

* Retrieved documents are irrelevant
* The knowledge base does not contain the answer
* The query is poorly formulated
* Web information is required
* The generated answer contains unsupported claims
* The answer is incomplete
* Citations are insufficient

This project addresses these problems by introducing **agentic decision-making, corrective retrieval, and answer-level self-evaluation**.

The resulting workflow is:

```text
User Query
    ↓
Agentic Routing
    ↓
Private KB Retrieval
    ↓
Evidence Evaluation
    ↓
Corrective Retrieval
    ↓
Web Fallback
    ↓
Answer Generation
    ↓
Self-RAG Evaluation
    ↓
PASS / RETRY
```

---

# 2. Key Capabilities

## Agentic RAG

The system uses **LangGraph** to dynamically control the RAG workflow.

The router determines whether a query should:

* Use the private knowledge base
* Be answered directly by the LLM

The workflow then dynamically decides what action should happen next based on evidence quality.

---

## Corrective RAG (CRAG)

The system evaluates retrieved KB evidence before generating an answer.

```text
Retrieve
   ↓
Grade Evidence
   ↓
 ┌───────────────┐
 │               │
Good            Weak
 │               │
 ▼               ▼
Generate     Rewrite Query
                 │
                 ▼
              Retrieve
                 │
                 ▼
              Grade
                 │
                 ▼
           Still insufficient
                 │
                 ▼
             Web Search
```

The evidence grader evaluates:

* Relevance
* Sufficiency
* Relevance score
* Reason for the decision

Example:

```text
Grade: weak
Score: 0.20

Reason:
The retrieved documents do not contain information
about Self-RAG and are insufficient to answer the query.
```

This allows the system to recover from poor retrieval rather than blindly generating an answer.

---

# 3. Self-RAG

After generating an answer, the system evaluates the answer itself.

The Self-RAG evaluator produces:

```text
Groundedness Score
Completeness Score
Citation Score
Decision
Feedback
```

Example:

```text
Groundedness: 0.90
Completeness: 0.90
Citation: 0.80
Decision: pass
```

The decision determines whether the answer can be returned or whether another retrieval/generation cycle should be attempted.

```text
Generated Answer
       ↓
Self-RAG Evaluator
       ↓
 ┌─────┴─────┐
 │           │
PASS        RETRY
 │           │
 ▼           ▼
END      Retrieve Again
```

This introduces an evaluation-driven feedback loop instead of assuming that every generated answer is correct.

---

# 4. Retrieval Strategy

The current implementation uses **Pinecone** as the private knowledge-base vector store.

```text
User Query
    ↓
Embedding
    ↓
Pinecone
    ↓
Top-K Documents
    ↓
Evidence Grader
```

The current retriever is configured to retrieve multiple relevant chunks from the private knowledge base.

---

# 5. Web Fallback

When the private knowledge base cannot provide sufficient evidence, the system performs web search using **Tavily**.

```text
Private KB
    ↓
Evidence Grader
    ↓
Weak
    ↓
Query Rewrite
    ↓
KB Retry
    ↓
Still Weak
    ↓
Tavily Web Search
    ↓
Web Evidence Grader
    ↓
Generate Answer
```

This creates a hybrid knowledge strategy:

```text
Private Knowledge
       +
External Web Knowledge
       ↓
Evidence-driven Answer Generation
```

---

# 6. Query Rewriting

When retrieval produces weak evidence, the system automatically rewrites the query.

Example:

```text
Original:

How does Self-RAG improve Agentic RAG?
```

Rewritten:

```text
How does Self-RAG enhance the retrieval-augmented generation
pipeline of Agentic RAG, specifically in terms of relevance
scoring, context integration, and hallucination mitigation?
```

The rewritten query is then sent back through the retrieval pipeline.

This creates an iterative retrieval loop:

```text
Query
 ↓
Retrieve
 ↓
Grade
 ↓
Weak
 ↓
Rewrite
 ↓
Retrieve Again
```

---

# 7. Evidence Grading

Both private KB and web evidence are evaluated before generation.

## KB Evidence Grader

```text
Question
   +
Retrieved KB Context
   ↓
Evidence Grader
   ↓
grade
score
reason
```

Possible grades:

```text
good
weak
```

## Web Evidence Grader

```text
Question
   +
Web Search Results
   ↓
Evidence Grader
   ↓
grade
score
reason
```

This prevents the generator from blindly trusting retrieved information.

---

# 8. Answer Generation

The project supports multiple answer-generation paths.

### Private KB

```text
KB Evidence
    ↓
KB Generator
    ↓
Answer
```

### Web

```text
Web Evidence
    ↓
Web Generator
    ↓
Answer
```

### Direct

```text
Simple Query
    ↓
Direct LLM
    ↓
Answer
```

---

# 9. LangGraph Workflow

The complete workflow is implemented using LangGraph.

Conceptually:

```text
START
  ↓
route_question
  ↓
 ┌───────────────────┐
 │                   │
KB                  Direct
 │                   │
 ▼                   ▼
retrieve_kb       direct_answer
 │                   │
 ▼                   ▼
grade_kb              END
 │
 ├── good ───────────────► generate_from_kb
 │                              │
 ├── rewrite ─► rewrite_query   │
 │                    │         │
 │                    ▼         │
 │                retrieve_kb   │
 │                              │
 └── exhausted ─────► search_web
                           │
                           ▼
                    grade_web_evidence
                           │
                    ┌──────┴──────┐
                    │             │
                  good           weak
                    │             │
                    ▼             ▼
             generate_web     rewrite_query
                    │
                    ▼
               grade_answer
                    │
              ┌─────┴─────┐
              │           │
            pass         retry
              │           │
              ▼           ▼
             END       rewrite_query
```

---

# 10. Technology Stack

| Component               | Technology           |
| ----------------------- | -------------------- |
| Programming Language    | Python               |
| LLM                     | Groq                 |
| LLM Model               | `openai/gpt-oss-20b` |
| Agent Orchestration     | LangGraph            |
| RAG Framework           | LangChain            |
| Vector Database         | Pinecone             |
| Web Search              | Tavily               |
| Structured Validation   | Pydantic             |
| Development Environment | VS Code / Jupyter    |
| Version Control         | Git / GitHub         |

---

# 11. Project Structure

Current project:

```text
advanced-rag-notebooks/
│
├── agentic_rag.ipynb
│
├── architecture_diagram.md
│
└── README.md
```

The notebook currently serves as the **working prototype and experimentation environment** for the Agentic RAG + CRAG + Self-RAG architecture.

The implementation can later be refactored into a production-oriented Python package once the architecture is stable.

---

# 12. Example Execution

Example query:

```text
How does Self-RAG improve Agentic RAG?
```

The system executes:

```text
[Router]
Decision: kb

        ↓

[KB Retriever]
Retrieved: 4 chunks

        ↓

[KB Grader]
Grade: weak
Score: 0.20

        ↓

[Query Rewriter]

        ↓

[KB Retriever]
Retrieved: 4 chunks

        ↓

[KB Grader]
Grade: weak
Score: 0.20

        ↓

[Tavily Search]

        ↓

[Web Grader]
Grade: good
Score: 0.95

        ↓

[Web Generator]

        ↓

[Self-RAG Grader]
Groundedness: 0.90
Completeness: 0.90
Citation: 0.80
Decision: pass

        ↓

[FINAL ANSWER]
```

This demonstrates that the system does not simply generate an answer after the first retrieval attempt.

Instead, it:

1. Retrieves from the private KB
2. Evaluates the retrieved evidence
3. Rewrites the query when evidence is weak
4. Retries retrieval
5. Falls back to web search when required
6. Grades the web evidence
7. Generates an answer
8. Evaluates the generated answer
9. Returns the answer only after passing the Self-RAG evaluation

---

# 13. Why This Is Different From Basic RAG

| Basic RAG                     | This Project                                        |
| ----------------------------- | --------------------------------------------------- |
| Retrieve → Generate           | Retrieve → Evaluate → Correct → Generate → Evaluate |
| Fixed workflow                | Conditional workflow                                |
| No retrieval validation       | KB/Web evidence grading                             |
| Usually one retrieval attempt | Query rewrite + retry                               |
| No fallback                   | Web fallback                                        |
| No answer evaluation          | Self-RAG evaluation                                 |
| Limited error recovery        | Feedback-driven recovery                            |
| Simple chain                  | LangGraph state machine                             |

---

# 14. Design Principles

The implementation follows several important RAG engineering principles:

### Evidence before generation

The system evaluates retrieved information before trusting it.

### Retrieval is not always correct

A vector database can return semantically similar but insufficient documents.

### Query rewriting is a recovery mechanism

Poor retrieval can sometimes be improved by reformulating the query.

### External search is a fallback

The private KB remains the preferred source, while web search provides additional coverage.

### Generation should be evaluated

A successful retrieval does not guarantee a correct answer.

### Bounded retries

The workflow uses retry limits to prevent uncontrolled loops.

---

# 15. Current Limitations

This notebook is an advanced prototype rather than a complete production deployment.

Current areas for future enhancement include:

* Cross-encoder / semantic reranking
* Hybrid vector + keyword retrieval
* Metadata-based filtering
* Improved citation tracking
* Separate retrieval and answer retry counters
* Automated evaluation datasets
* Retrieval precision / recall metrics
* Hallucination benchmarks
* LangSmith observability
* Knowledge Graph / Neo4j integration
* Databricks-based ingestion pipeline
* API deployment
* Authentication and access control
* Production monitoring

These are planned extensions rather than components claimed to be implemented in the current notebook.

---

# 16. Future Roadmap

```text
Current
   │
   ├── Agentic RAG                    ✅
   ├── CRAG                           ✅
   ├── Self-RAG                       ✅
   ├── Query Rewriting                ✅
   ├── KB Evidence Grading            ✅
   ├── Web Evidence Grading           ✅
   └── Web Fallback                   ✅
          │
          ▼
Phase 2
   ├── Reranking
   ├── Metadata Filtering
   ├── Citation Validation
   └── Advanced Retrieval
          │
          ▼
Phase 3
   ├── LangSmith Observability
   ├── Evaluation Dataset
   ├── RAG Metrics
   └── Automated Testing
          │
          ▼
Phase 4
   ├── Hybrid Retrieval
   ├── Neo4j Knowledge Graph
   └── Graph + Vector RAG
          │
          ▼
Phase 5
   ├── Databricks Medallion Architecture
   ├── Production Data Pipeline
   ├── API
   └── Cloud Deployment
```

---

# 17. Learning Objectives

This project demonstrates practical understanding of:

* Retrieval-Augmented Generation
* Advanced RAG patterns
* Agentic AI
* LangGraph state machines
* Conditional graph routing
* Query transformation
* Evidence evaluation
* Corrective RAG
* Self-RAG
* Vector databases
* Web-augmented RAG
* Structured LLM output
* Retry and recovery strategies
* Grounded answer generation
* RAG evaluation concepts

---

# 18. Git Commit Milestone

Current implementation checkpoint:

```text
Commit:
42472c6

Message:
feat: implement agentic RAG with CRAG and Self-RAG
```

This commit represents the working **Agentic RAG + Corrective RAG + Self-RAG** baseline.

---

# 19. Conclusion

This project demonstrates an evolution from a traditional RAG pipeline toward an **adaptive, evaluation-driven Agentic RAG architecture**.

The core design is:

```text
             AGENTIC RAG
                  │
                  ▼
             CRAG LOOP
                  │
       ┌──────────┴──────────┐
       │                     │
   Evidence Good        Evidence Weak
       │                     │
       │                Query Rewrite
       │                     │
       │                Retry Retrieval
       │                     │
       │                 Web Fallback
       │                     │
       └──────────┬──────────┘
                  │
                  ▼
             GENERATION
                  │
                  ▼
              SELF-RAG
                  │
           ┌──────┴──────┐
           │             │
         PASS           RETRY
           │             │
           ▼             └──────► Retrieval Loop
          END
```

The project is intentionally designed to evolve from a notebook-based prototype into a production-oriented RAG platform through incremental improvements in retrieval quality, evaluation, observability, knowledge representation, data engineering, and deployment.
