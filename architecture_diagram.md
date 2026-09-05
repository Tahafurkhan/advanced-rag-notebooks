                    ┌──────────────────────────────┐
                    │         USER QUERY           │
                    │                              │
                    │ "How does Self-RAG improve  │
                    │       Agentic RAG?"          │
                    └──────────────┬───────────────┘
                                   │
                                   ▼
                    ┌──────────────────────────────┐
                    │       AGENTIC ROUTER          │
                    │          LangGraph            │
                    │                              │
                    │  KB  ───────────────┐        │
                    │  Direct              │        │
                    └──────┬───────────────┘        │
                           │                        │
                           │ KB                     │ Direct
                           ▼                        ▼
              ┌──────────────────────┐    ┌──────────────────────┐
              │    KB RETRIEVER      │    │    DIRECT LLM        │
              │                      │    │      RESPONSE        │
              │      Pinecone        │    └──────────┬───────────┘
              └──────────┬───────────┘               │
                         │                           ▼
                         ▼                          END
              ┌──────────────────────┐
              │   KB EVIDENCE GRADER │
              │                      │
              │ Grade: good / weak   │
              │ Score: 0 → 1         │
              │ Reason                │
              └──────────┬───────────┘
                         │
                 ┌───────┴────────┐
                 │                │
               GOOD              WEAK
                 │                │
                 ▼                ▼
       ┌─────────────────┐  ┌──────────────────────┐
       │ GENERATE FROM   │  │   CRAG QUERY         │
       │      KB         │  │      REWRITER        │
       └────────┬────────┘  └──────────┬───────────┘
                │                      │
                │                      ▼
                │               ┌──────────────┐
                │               │ KB RETRIEVAL │
                │               │    RETRY     │
                │               └──────┬───────┘
                │                      │
                │               ┌──────▼────────────┐
                │               │ KB EVIDENCE      │
                │               │     GRADER        │
                │               └──────┬────────────┘
                │                      │
                │                 Still weak
                │                      │
                │                      ▼
                │               ┌──────────────┐
                │               │  WEB SEARCH  │
                │               │    Tavily    │
                │               └──────┬───────┘
                │                      │
                │                      ▼
                │               ┌──────────────┐
                │               │ WEB EVIDENCE │
                │               │    GRADER    │
                │               └──────┬───────┘
                │                      │
                │                     GOOD
                │                      │
                │                      ▼
                │               ┌──────────────┐
                │               │ WEB ANSWER   │
                │               │  GENERATOR   │
                │               └──────┬───────┘
                │                      │
                └──────────┬───────────┘
                           │
                           ▼
                ┌──────────────────────────┐
                │      SELF-RAG GRADER     │
                │                          │
                │  Groundedness Score      │
                │  Completeness Score      │
                │  Citation Score          │
                │  Decision: PASS / RETRY  │
                └────────────┬─────────────┘
                             │
                     ┌───────┴────────┐
                     │                │
                   PASS              RETRY
                     │                │
                     ▼                ▼
                  ┌─────┐       Query Rewrite /
                  │ END │       Retrieve Again
                  └─────┘
