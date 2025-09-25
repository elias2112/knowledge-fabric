# ADR 0008 — Baseline Preload (/documents) vs On-Demand Search

**Status:** Accepted  
**Date:** 2025-09-24

## Context
The UI must show something even before the first search, and we also need fresh results on demand.

## Decision
- Load a small baseline from `/documents` per index (e.g., `vListSize=10`).
- On user action (`vDoSearch=1`), hit `/search` with cleaned query and safe size, append results, and mark `loaded_from_search=1`.

## Alternatives Considered
- Search-only flow → empty UI on first load.
- Full preload → slow reload, heavy backend impact.

## Consequences
- Positive: Good initial UX, responsive searches, controlled volume.
- Negative: Requires two code paths and careful dedup.

## References
- Per-index loaders + `__RunSearch`.
