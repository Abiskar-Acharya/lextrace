# metrics/

Pure functions over saved traces (`traces/run_*/*.jsonl`). No metric reads live from a subject — everything is computed post-hoc from what was recorded, so a new metric re-scores every historical run for free.

- `retrieval.py` — recall@k, nDCG@10, MRR, character-level precision/recall (reusing LegalBench-RAG's span-overlap definition), Document-Level Retrieval Mismatch (Reuter et al. 2025), doc-conditioned vs. doc-agnostic delta.
- `grounding.py` — four-way atomic claim partition: grounded / in-corpus-but-not-retrieved / parametric-leakage / hallucination.
- `citation.py` — citation correctness taxonomy: misattribution, phantom citation, unsupported citation.
- `calibration.py` — judge-score variance across N samples, Cohen's κ against a human-labelled subset, position-bias check.
