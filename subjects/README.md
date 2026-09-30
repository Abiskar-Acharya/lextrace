# subjects/

Retrieval systems under test. Each subject implements a common interface (`base.py`) so the harness can run any of them against the same frozen benchmark without caring how they work internally — this is what makes it a harness and not a single app with an eval script bolted on.

- `base.py` — the `retrieve(query, k) -> list[Passage]` contract every subject must satisfy.
- `hybrid_rrf/` — BM25 + dense + reciprocal rank fusion + cross-encoder rerank, adapted from ArXivMind's `HybridRetriever`. The arXiv-paper section-header chunker is replaced here with a clause-boundary chunker suited to contract structure; nothing in the original `glm-rag-pipeline` repo is modified.
- `dense_only/`, `bm25_only/` — ablation subjects, same interface, single retriever each.
- `clause_aware/` — chunker variant using CUAD's own annotated clause boundaries as chunk boundaries, to test whether structure-aware chunking reduces Document-Level Retrieval Mismatch.
