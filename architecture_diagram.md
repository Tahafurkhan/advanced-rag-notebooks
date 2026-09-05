                         USER QUESTION
                              │
                              ▼
                    ┌──────────────────┐
                    │  ROUTE QUESTION  │
                    └────────┬─────────┘
                             │
                    ┌────────┴────────┐
                    │                 │
                 DIRECT               KB
                    │                 │
                    ▼                 ▼
              Direct Answer      Retrieve KB
                                      │
                                      ▼
                             ┌─────────────────┐
                             │  CRAG EVALUATOR │
                             └────────┬────────┘
                                      │
                         ┌────────────┴────────────┐
                         │                         │
                       GOOD                      WEAK
                         │                         │
                         │                  Rewrite Query
                         │                         │
                         │                    Retrieve KB
                         │                         │
                         │                    Grade Again
                         │                         │
                         │                    ┌────┴────┐
                         │                    │         │
                         │                  GOOD      WEAK
                         │                    │         │
                         │                    │       Web Search
                         │                    │         │
                         │                    │    Grade Web
                         │                    │         │
                         └────────────────────┴─────────┘
                                      │
                                      ▼
                              GENERATE ANSWER
                                      │
                                      ▼
                            ┌──────────────────┐
                            │  SELF-RAG CHECK  │
                            └────────┬─────────┘
                                     │
                           ┌─────────┴─────────┐
                           │                   │
                         PASS                FAIL
                           │                   │
                           ▼                   ▼
                          END             SELF-CORRECT
                                               │
                                      ┌────────┴────────┐
                                      │                 │
                                Rewrite Query     Improve Retrieval
                                      │                 │
                                      └────────┬────────┘
                                               │
                                               ▼
                                         Retrieve Again
                                               │
                                               ▼
                                            Generate
                                               │
                                               ▼
                                         Self-RAG Check
                                               │
                                      max retries reached?
                                               │
                                               ▼
                                              END