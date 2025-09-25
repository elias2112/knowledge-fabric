# ADR 0009 — Synthetic Keys Strategy

**Status:** Accepted  
**Date:** 2025-09-24

## Context
Qlik auto-creates synthetic keys when multiple tables share fields. Our design concatenates into `Docs_All` and `SearchIndex` without joining, but synthetic keys may still occur if fields align across residual loads.

## Decision
- Prefer concatenation over joins.
- Keep `Search_Results` minimal (id/title/snippet/url/type).
- Avoid loading partial replicas of `Docs_All` fields into other tables.
- If a future join is necessary, use `QUALIFY/UNQUALIFY` or explicit key fields to prevent synthetic keys explosion.

## Alternatives Considered
- Accept synthetic keys everywhere → hard to reason about model.
- Over-qualify all fields → noisy model.

## Consequences
- Positive: Simpler model, stable associations.
- Negative: Requires vigilance when adding new loaders.

## References
- Field lists in `__AppendFromMaster` and `__RunSearch`.
