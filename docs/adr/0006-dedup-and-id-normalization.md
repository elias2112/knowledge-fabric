# ADR 0006 — Deduplication and ID Normalization

**Status:** Accepted  
**Date:** 2025-09-24

## Context
Documents may include chunked variants (`*_chunk_*`). Appending multiple times across reloads can introduce duplicates.

## Decision
When appending to `Docs_All` and `SearchIndex`, use:
- `AND NOT Exists(id, [_id])` to deduplicate.
- Derive `doc_base_id`, `chunk_no`, `is_chunk` by pattern on `_id`.
This keeps a coherent “base doc + chunks” model.

## Alternatives Considered
- Dedup only by `doc_base_id` → risks dropping valid chunks.
- No dedup → ballooning tables and inconsistent UI.

## Consequences
- Positive: Stable incremental loads, predictable counts.
- Negative: Requires consistent `_id` conventions in the backend.

## References
- `__AppendFromMaster` and `__RunSearch` appends.
