# ADR 0005 — Size and Pagination Guardrails

**Status:** Accepted  
**Date:** 2025-09-24

## Context
Passing `size=""` to `/search` returns 422 errors (“int parsing”). Empty or non-numeric size must be sanitized. We also need to cap default size to protect the backend.

## Decision
Compute `vEffSize` as a safe integer:
- Default to `vApiMaxSize`.
- If a user value exists and parses as a number, use it.
Never call `/search` with an empty size.

## Alternatives Considered
- Blindly forwarding user input → fragile, frequent 422s.
- Hardcoded fixed size → less flexible.

## Consequences
- Positive: No 422s from empty size, backend protected.
- Negative: Slightly more logic in the search subroutine.

## References
- `__RunSearch` size normalization.
