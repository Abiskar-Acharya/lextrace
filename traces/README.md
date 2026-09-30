# traces/

One JSONL file per query per run. This is the instrument's spine — every metric in `metrics/` is a pure function over what's recorded here, nothing more.

Per-query record: ordered spans `parse → chunk_lookup → bm25 → dense → fuse → rerank → assemble → generate → cite`, each storing `{stage, t_start, t_end, inputs_hash, outputs, scores, ranks, config_hash}`, plus a top-level `gold_reach` field: the first span where the gold character span became unreachable, computed by set intersection against `benchmark/gold.jsonl` — no LLM judge in this computation.

Bulk run output is gitignored (see root `.gitignore`); a small `sample_run/` is kept in git to pin the schema.
