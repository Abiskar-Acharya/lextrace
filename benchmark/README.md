# benchmark/

Frozen queries + gold answers + scoring protocol. Once versioned, immutable — adding a metric must never require touching this folder, and fixing this folder means a new version, not a silent edit.

- `queries.jsonl` — LegalBench-RAG queries as-published (document-conditioned): `"Consider <document description>; <interrogative>"`.
- `variants/document_agnostic.jsonl` — the same queries with the document description stripped, for the DRM/leakage experiment (see root README and `docs/METHODOLOGY.md` M8, probe P5).
- `gold.jsonl` — gold answer spans, reusing LegalBench-RAG's `Snippet(file_path, span=(start, end))` shape directly (see `~/third-party/legalbenchrag/legalbenchrag/benchmark_types.py`).
- `protocol.md` — fixed k, scoring rule, reporting rule (Wilson intervals given the query count).
