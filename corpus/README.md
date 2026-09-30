# corpus/

Frozen corpus for a run. Contains:

- `manifest.json` — one entry per file: `sha256`, `char_count`, `source_dataset` (cuad / maud / contractnli / privacyqa), `indexed_at`.
- `v1.lock` — the manifest's own hash, written once and never edited. If the corpus needs to change, that's `v2`, not an edit to `v1`.
- `raw/` — the actual text files (gitignored; fetched by `scripts/fetch_corpus.py` from the LegalBench-RAG Dropbox link, see repo root README).

Nothing in `benchmark/`, `subjects/`, or `metrics/` should read a file that isn't in `manifest.json`. That invariant is what makes a run reproducible — the corpus a metric was scored against is always named by hash, never by "whatever was on disk that day".
