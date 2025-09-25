# ADR 0010 — Fetch by ID via /search

**Status:** Accepted  
**Date:** 2025-09-24

## Context
We occasionally need to pull a specific document by its ID for inspection or UI deep-link. The backend supports only `/search` with "q", not a direct `/documents/{id}` route.

## Decision
Implement `__FetchById(index, id)` that calls `/search` with `q=id` and `size=1`, then appends to `Docs_All`/`SearchIndex` if not present. Sanitize input to avoid empty queries.

## Alternatives Considered
- Build a dedicated `/by-id` endpoint → not available at the time.
- Recompute full index → too heavy.

## Consequences
- Positive: Simple, reuses existing pipeline.
- Negative: Requires unique IDs and exact match behavior in the backend search.

## References
- `__FetchById` subroutine.
