# probes/

Counterfactual corpus perturbations, run against the frozen benchmark to test *why* retrieval succeeds or fails, not just whether it did.

- P1 — delete the gold chunk from the corpus. Unchanged answer ⇒ the subject answered from parametric memory, not retrieval.
- P2 — inject a lexically-similar wrong-document distractor.
- P3 — shuffle retrieved-context order (positional bias).
- P4 — paraphrase the gold span (lexical over-reliance check).
- P5 — strip the document description from the query (document-conditioning leakage — the project's primary novel experiment; see root README).
