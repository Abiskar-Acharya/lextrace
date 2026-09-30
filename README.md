# LexTrace: a diagnostic evaluation harness for legal RAG retrieval

**Status:** architecture and methodology specified; no code written, no corpus frozen, no run executed yet. This README will be rewritten once results exist — right now it documents the plan, not a claim.

## What this is

Most RAG evaluation tools score a pipeline with one number. When that number is bad, nobody can say *why* — was the passage never in the corpus, did the retriever fetch the wrong contract, did the reranker bury the right chunk, or did the generator ignore evidence it had? Collapsing four different bugs into one score is why teams spend months tuning the wrong component.

LexTrace is not another RAG chatbot and not another scalar RAG score. It is an instrumented **evaluation harness**: a hybrid retrieval pipeline (BM25 + dense + reciprocal rank fusion + cross-encoder rerank) is run as the *subject under test*, every query is traced span-by-span through the full pipeline, and the trace is checked mechanically — not by an LLM judge — for whether the gold answer span was still reachable at each stage. The first stage where it stops being reachable is where the failure is attributed.

## Why legal, why this corpus

The subject domain is legal contract retrieval, using [LegalBench-RAG](https://arxiv.org/abs/2408.10343) (Pipitone & Houir Alami, 2024) — 6,889 expert-annotated queries with character-level gold spans over a 79M-character corpus spanning CUAD, MAUD, ContractNLI, and PrivacyQA. This corpus was chosen over building a bespoke one because it hands over exactly the thing most RAG evaluation setups lack: a frozen, reproducible, expert-authored answer key, so no metric here depends on regenerating queries at run time.

LegalBench-RAG's own limitations section states its queries are always answerable from exactly one document, and each query names that document by description. An independent replication ([Reuter et al. 2025, "Towards Reliable Retrieval in RAG Systems for Large Legal Datasets"](https://arxiv.org/abs/2510.06999)) found retrievers still pull chunks from the wrong document even when told which one to search — Document-Level Retrieval Mismatch (DRM). ArXivRAG's original contribution is running the benchmark twice per system, once as published (document-conditioned) and once with the document description stripped from the query (document-agnostic), and measuring the gap. That gap quantifies how much reported legal-retrieval performance is an artifact of being handed the answer's address.

## What this is not

Not a new scalar RAG score (RAGAS, ARES, RAGChecker, CCRS, RAGVUE, Q-CARE already occupy that space, and RAGVUE already does diagnostic/explainable scoring). Not a new stage-attribution method in the abstract (Doctor-RAG and Legal RAG Bench already attribute failures to retrieval/reasoning/hallucination stages on their own corpora). This project's claim to novelty is narrow and stated honestly in `docs/METHODOLOGY.md` section M2: deterministic (non-judge) attribution via gold-span reachability, run at LegalBench-RAG's full scale (6,889 queries vs. Legal RAG Bench's 100 hand-crafted ones), plus the document-conditioning leakage experiment nobody has run on this benchmark yet.

The name: `gold_reach` — the trace field this harness lives or dies on — is a reachability check performed at every handoff in the pipeline, the same function a chain-of-custody log performs for physical evidence. LexTrace names the mechanism, not the domain.

## Layout

| Path | Contents |
|---|---|
| `corpus/` | Frozen corpus + `manifest.json` (SHA-256 per file) + version lock. Raw text is fetched, not committed — see `corpus/README.md`. |
| `benchmark/` | Queries, gold spans, document-agnostic query variants, scoring protocol. Frozen once versioned. |
| `subjects/` | Retrieval systems under test, each implementing a common `retrieve()` interface. First subject: hybrid BM25+dense+RRF+rerank, adapted from the [ArXivMind](https://github.com/Abiskar-Acharya/Research-Assistant) pipeline with a legal clause-boundary chunker replacing its arXiv-section chunker. |
| `traces/` | Per-run, per-query trace JSONL — the instrument's raw output. |
| `metrics/` | Pure functions over saved traces: retrieval, grounding, citation, calibration. |
| `probes/` | Counterfactual corpus perturbations (delete gold chunk, inject distractor, shuffle context, paraphrase gold, strip document description). |
| `analysis/` | Failure-attribution tables and plots generated from trace + metric output. |
| `notebooks/` | Exploration notebooks (data shape, prior-art reproduction checks, early plots). |
| `report/` | Generated, reproducible write-up. |
| `tests/` | Unit tests for metric functions and the trace schema. |

See `docs/METHODOLOGY.md` for the full research design (M1–M12).

## Prior art consulted

- [LegalBench-RAG](https://github.com/zeroentropy-ai/legalbenchrag) — corpus/benchmark source, `Snippet`/`QAGroundTruth` gold-span format reused directly.
- [TruLens](https://github.com/truera/trulens) — OpenTelemetry-based span/trace instrumentation reference (`src/otel/semconv`).
- RAGAS, ARES, RAGChecker, CCRS, RAGVUE, Q-CARE — scalar/diagnostic RAG evaluators, surveyed for what already exists.
- Doctor-RAG (arXiv 2604.00865), Legal RAG Bench (arXiv 2603.01710) — stage-level failure attribution, closest adjacent work.
- Dahl et al. 2024 "Large Legal Fictions", Magesh et al. 2025 (Stanford RegLab) — legal hallucination rate baselines this harness's numbers should be read against.
