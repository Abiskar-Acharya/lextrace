# LexTrace: a diagnostic evaluation harness for legal RAG retrieval

**Status:** corpus frozen (v1, 714 files). No pipeline or metrics code yet.

## What this is

Most RAG evaluation tools score a pipeline with one number. When that number is bad, nobody can say *why* — was the passage never in the corpus, did the retriever fetch the wrong contract, did the reranker bury the right chunk, or did the generator ignore evidence it had? Collapsing four different bugs into one score is why teams spend months tuning the wrong component.

LexTrace is an instrumented **evaluation harness**: a hybrid retrieval pipeline (BM25 + dense + reciprocal rank fusion) is run as the *subject under test*, every query is traced span-by-span through the full pipeline, and the trace is checked mechanically — not by an LLM judge — for whether the gold answer span was still reachable at each stage. The first stage where it stops being reachable is where the failure is attributed.

`gold_reach` — the trace field the whole harness runs on — is a reachability check performed at every handoff in the pipeline, the same function a legal chain-of-custody log performs for physical evidence.

## Why legal, why this corpus

The subject domain is legal contract retrieval, using the LegalBench-RAG benchmark (Pipitone & Houir Alami, 2024) — 6,889 expert-annotated queries with character-level gold spans over a 79M-character corpus spanning CUAD, MAUD, ContractNLI, and PrivacyQA. This corpus hands over what most RAG evaluation setups lack: a frozen, reproducible, expert-authored answer key, so no metric here depends on regenerating queries at run time.

The benchmark's queries always name their target document by description before asking the question. An independent replication (Reuter et al. 2025, "Towards Reliable Retrieval in RAG Systems for Large Legal Datasets") found retrievers still pull chunks from the wrong document even when told which one to search — a phenomenon called Document-Level Retrieval Mismatch. LexTrace's central experiment runs the benchmark twice per system: once as published (document-conditioned) and once with the document description stripped from the query (document-agnostic), measuring the gap between them. That gap quantifies how much reported legal-retrieval performance is an artifact of being handed the answer's address.

## Related work

This isn't a new scalar RAG score — RAGAS, ARES, and RAGChecker already fill that role. It isn't a new stage-attribution method in the abstract either — Doctor-RAG and Legal RAG Bench (2026) already attribute legal RAG failures to retrieval/reasoning/hallucination stages, with Legal RAG Bench's headline finding being that most "hallucinations" in legal RAG are in fact retrieval failures. LexTrace's contribution is narrower and stated plainly: deterministic, non-judge attribution via gold-span reachability, run at full benchmark scale, plus the document-conditioning leakage experiment above, which nothing in the published literature has measured yet.

Legal-domain hallucination baselines this harness's numbers should be read against: Dahl et al. 2024 ("Large Legal Fictions") and Magesh et al. 2025 (Stanford RegLab), both finding production legal RAG tools hallucinate in the 17–34% range.

## Data and license

Corpus and gold labels: LegalBench-RAG (Pipitone & Houir Alami, 2024), distributed under CC BY 4.0, built on CUAD, MAUD, ContractNLI, and PrivacyQA. `corpus/manifest.json` and `corpus/v1.lock` fix the exact frozen copy this repo's results are computed against.

## Layout

| Path | Contents |
|---|---|
| `corpus/` | Frozen corpus + `manifest.json` (SHA-256 per file) + version lock. Raw text is fetched, not committed. |
| `benchmark/` | Queries, gold spans, document-agnostic query variants, scoring protocol. |
| `subjects/` | Retrieval systems under test, each implementing a common interface. |
| `traces/` | Per-run, per-query trace JSONL — the instrument's raw output. |
| `metrics/` | Pure functions over saved traces: retrieval, grounding, citation, calibration. |
| `probes/` | Counterfactual corpus perturbations (delete gold chunk, inject distractor, shuffle context, paraphrase gold, strip document description). |
| `analysis/` | Failure-attribution tables and plots generated from trace + metric output. |
| `notebooks/` | Exploration notebooks. |
| `report/` | Generated, reproducible write-up. |
| `tests/` | Unit tests for metric functions and the trace schema. |
