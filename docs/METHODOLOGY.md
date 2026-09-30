# Methodology

Status: specified, not yet executed. Nothing below is a result — it is the design the results will be checked against.

## M1 — Problem and research questions

RQ1 (instrument validity): does deterministic gold-span reachability produce failure attribution that agrees with hand-inspection on a held-out sample?
RQ2 (leakage): how much does stripping the document description from a LegalBench-RAG query (document-agnostic condition) degrade retrieval, relative to the published document-conditioned condition?
RQ3 (attribution split): across a fixed subject and corpus, what fraction of failures attribute to chunking, sparse retrieval, dense retrieval, fusion, rerank, context assembly, or generation?
RQ4 (intervention effect): does clause-boundary chunking (using CUAD's own annotated spans) reduce Document-Level Retrieval Mismatch relative to fixed-length chunking, on the same benchmark?

## M2 — Prior art and positioning

The general RAG-evaluation-harness space is occupied: RAGAS, ARES, RAGChecker, CCRS, RAGVUE, Q-CARE, DoRAG all provide scalar or fine-grained scoring, and RAGVUE already frames itself as "diagnostic and explainable." Stage-level failure attribution specifically is also occupied: Doctor-RAG (arXiv 2604.00865) attributes agentic RAG trajectory failures to Format/Reasoning/Retriever/Search error via a distilled diagnosis model; Legal RAG Bench (arXiv 2603.01710) runs a full factorial design over legal RAG and attributes errors to hallucination/retrieval/reasoning, with the headline finding that most hallucinations in legal RAG are in fact retrieval failures.

What is not occupied: (a) attribution computed deterministically from gold character spans rather than from a judge or distilled classifier, removing judge bias from the attribution step itself; (b) the document-conditioning leakage experiment on LegalBench-RAG specifically — the benchmark's own limitations section admits every query names its target document, and Reuter et al. 2025 (arXiv 2510.06999) found retrievers still mismatch documents even when told which one to search (Document-Level Retrieval Mismatch), but no one has measured the gap between conditioned and unconditioned performance on this benchmark; (c) scale — 6,889 expert-annotated queries vs. Legal RAG Bench's 100 hand-crafted ones.

Honest scope statement: this is a strong first-year PhD contribution or a strong MSc thesis. It is not a full PhD by itself. Do not oversell it in any proposal.

## M3 — Corpus construction and freezing

Corpus = LegalBench-RAG's four source datasets (CUAD, MAUD, ContractNLI, PrivacyQA), fetched via the Dropbox link in the upstream repo, not regenerated (regenerating uses an LLM and will not reproduce the same benchmark — see upstream README). Each file gets a SHA-256 in `corpus/manifest.json`. Once `corpus/v1.lock` is written, the corpus is immutable; any correction is `v2`, never a silent edit to `v1`.

## M4 — Benchmark design

Queries and gold spans are LegalBench-RAG's own `QAGroundTruth` objects (`benchmark_types.py` in the upstream repo): `{query: str, snippets: [{file_path, span: (start, end)}]}`. Reused directly rather than re-authored, since re-authoring would discard the only thing this benchmark offers for free — expert-verified gold spans. A second query set, `variants/document_agnostic.jsonl`, strips the `"Consider <document description>; "` prefix each query is templated with, for the RQ2 experiment. Fixed k per experiment, character-level precision/recall as LegalBench-RAG itself defines it, Wilson confidence intervals given the query count (6,889 is enough for genuinely tight intervals — use them, don't just report point estimates).

## M5 — Harness architecture

Trace schema, one JSONL record per query per run:

```
{
  "query_id": str,
  "spans": [
    {"stage": "parse", ...},
    {"stage": "chunk_lookup", ...},
    {"stage": "bm25", "top_k": [...], "scores": [...]},
    {"stage": "dense", "top_k": [...], "scores": [...]},
    {"stage": "fuse", "top_k": [...], "source_list": [...]},
    {"stage": "rerank", "top_k": [...], "scores": [...]},
    {"stage": "assemble", "final_context": [...], "truncated": bool},
    {"stage": "generate", "answer": str, "model": str, "prompt_hash": str},
    {"stage": "cite", "citations": [{"claim_span": ..., "cited_chunk_id": ...}]}
  ],
  "gold_reach": {"stage": str, "reachable": bool}
}
```

Every span carries `{t_start, t_end, inputs_hash, config_hash}` in addition to its stage-specific fields, so a run is fully replayable in principle even without re-running the subject. Subjects implement one interface (`subjects/base.py`); the harness never depends on a subject's internals, only on what it emits into the trace. This is what makes it a harness and not an app with an eval script attached — TruLens's OTEL-based span/attribute model (`~/third-party/trulens/src/otel/semconv`) is the instrumentation-library reference for how to make spans queryable, not a dependency we take on.

`gold_reach` is computed by set intersection between each stage's candidate character ranges and `benchmark/gold.jsonl`'s spans — the first stage at which the intersection becomes empty is where the answer stopped being reachable. No LLM is involved in computing this field.

## M6 — Metric definitions

**Retrieval:** recall@k, nDCG@10, MRR, character-level precision/recall, Document-Level Retrieval Mismatch (Reuter et al. 2025's definition: proportion of top-k chunks not from the gold document), and the document-conditioned vs. document-agnostic delta (this project's primary metric).

**Grounding:** atomic claim decomposition, four-way partition — grounded / in-corpus-but-not-retrieved (retrieval coverage failure) / nowhere-in-corpus-but-true (parametric leakage) / nowhere-and-false (hallucination) — plus deterministic numeric/date exact-match after normalisation, avoiding LLM-judge fuzziness for the checks that can be checked exactly.

**Citation:** three named failure modes — misattribution (real clause, wrong document), phantom citation (clause doesn't exist), unsupported citation (clause exists, doesn't support the claim).

**Calibration:** N-sample judge mean ± std, Cohen's κ against a human-labelled subset, position-bias check (does reordering identical context change the judge's score).

## M7 — Attribution algorithm

Formal rule: attribute a failure to the earliest stage `k` such that `gold_reach.stage == k` and `gold_reach.reachable == False`. Compound failures (multiple stages that could each explain the miss) are attributed to the earliest one, since every downstream stage inherits the upstream fault as its premise — an unreachable answer at `bm25` makes `rerank`'s behaviour on that query uninformative, not a second failure.

## M8 — Counterfactual probes

P1 delete gold chunk (tests parametric-memory reliance) · P2 inject lexically-similar wrong-document distractor (tests DRM directly) · P3 shuffle context order (positional bias) · P4 paraphrase gold span (lexical over-reliance) · P5 strip document description from query (RQ2 — the project's primary novel experiment).

## M9 — Experiment matrix

Factorial: subject (`hybrid_rrf` / `dense_only` / `bm25_only` / `clause_aware`) × query condition (conditioned / agnostic) × probe (none / P1–P4), fixed k and fixed corpus version throughout.

## M10 — Validity and threats

LegalBench-RAG's own admitted limitation: every query is answerable from exactly one document, so nothing here tests cross-document reasoning. Gold is span-level, not answer-level — a system can retrieve the gold span and still generate a wrong answer, which is why grounding and retrieval are scored separately, never as one number. Single-corpus generalisation: results here are about LegalBench-RAG's four datasets (NDAs, M&A agreements, commercial contracts, privacy policies), not about legal retrieval as a whole. Judge validity: any LLM-judge component is checked against the deterministic scorer and reported with disagreement, not treated as ground truth by default.

## M11 — Reproducibility

A reviewer needs: `corpus/manifest.json` + `corpus/v1.lock`, `benchmark/` as committed, and one subject's code to reproduce any reported number. Runtime and query count are reported alongside every result table so "how long does this take to check" has an answer.

## M12 — Ethics and licensing

Corpus is CC BY 4.0 via LegalBench-RAG / CUAD / MAUD / ContractNLI / PrivacyQA — redistribution and reuse permitted with attribution, already given in the root README. This project does not produce legal advice and makes no claim about deployment readiness for real contract review. UK Find Case Law is explicitly out of scope until (and unless) a computational-analysis licence is filed with The National Archives — their Open Justice Licence excludes NLP/vector-database/ML-training use without one.
