# ADR 0004 — Search Dispatcher and Optional Index Filter

**Status:** Accepted  
**Date:** 2025-09-24

## Context
The app may search multiple indices or a single selected index. Some backends require `index_name` explicitly for performance and scoping.

## Decision
Implement a dispatcher that:
- If `vIndexSel` is set, search only that index.
- Else iterate over `vIndexList`.
Gate the inclusion of `index_name` with `vUseIndexFilter=1` (send it by default).

## Alternatives Considered
- Always search all indices → unnecessary load, slower responses.
- Hardcode per-screen index → less flexible.

## Consequences
- Positive: Flexible behavior, good performance when filtering.
- Negative: Requires maintaining `vIndexList`.

## References
- End-of-script search trigger, `__RunSearch` parameterization.
