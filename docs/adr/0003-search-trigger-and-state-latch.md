# ADR 0003 — Search Trigger and State Latch (vQueryPersist & vDoSearch)

**Status:** Accepted  
**Date:** 2025-09-24

## Context
Qlik variables are evaluated at reload time and can be cleared or drift between interactions. We need the input text to persist across reloads and to avoid accidental empty searches.

## Decision
Use `vQueryPersist` as the canonical input persisted by the UI. On search, set `vDoSearch = 1`. In the script, if `vDoSearch=1`, copy `vQueryPersist` into `vQuery`, run the search, then **always reset** `vDoSearch = 0`. Never override `vQueryPersist` in the script.

## Alternatives Considered
- Driving solely from `vQuery` → input cleared frequently; race conditions.
- Not resetting the trigger → repeated searches on subsequent reloads.

## Consequences
- Positive: Idempotent behavior, input remains visible, predictable UX.
- Negative: Requires discipline in button actions and variable wiring.

## References
- Header section: latch logic and defaulting rules.
