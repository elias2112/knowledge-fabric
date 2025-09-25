# ADR 0007 — Error Handling and Logging (ErrorMode, TRACE)

**Status:** Accepted  
**Date:** 2025-09-24

## Context
We observed 404, 422, 500 and “connection not found” errors. We need quick diagnosis and safe cleanup when a segment returns no rows.

## Decision
- Use `SET ErrorMode=0` around REST calls; restore `ErrorMode=1` afterwards.
- Always create/drop a temporary `RestConnectorMasterTable` and clean it when hits are zero.
- Use consistent `TRACE` tags: `[SearchTrigger]`, `[RunSearch]`, `[DEBUG]` to make logs scannable.

## Alternatives Considered
- Global ErrorMode=0 → hides real issues.
- No TRACE discipline → slow troubleshooting.

## Consequences
- Positive: Faster triage, cleaner failure modes.
- Negative: Slight verbosity in logs.

## References
- `__RunSearch`, per-index loaders, debug counters.
