# Operations Runbook

## Overview
What the app does and the key data flows.

## Prerequisites
Qlik version, REST connections, network access, env vars.

## Routine Operations
- Baseline reload: steps, expected row counts
- Run a search: steps, expected results
- Clear/reset: steps

## Monitoring & Logs
- Where to see TRACE logs
- What to watch (e.g., 422/500, connection not found)

## Troubleshooting (First 15 Minutes)
Symptom → Checks → Fix
- No search results → validate non-empty query, size integer, connectivity
- Input clears → verify vQueryPersist latch and vDoSearch reset
- Duplicates → ensure `AND NOT Exists(id, [_id])`
- Synthetic keys → review field naming and concatenations

## Known Errors
- 422 size parsing → ensure effective size is numeric (no empty string)
- 500 empty query → normalize quotes and trim before `/search`
- Connection not found → verify exact connection names in scripts
- 404 on /documents → confirm base URL and index_name

## Rollback / Bypass
- Disable `/search` path and rely on `/documents` baseline
- Reduce `vApiMaxSize` under backend stress

## Configuration
- vBaseUrl, vApiMaxSize, vUseIndexFilter, vIndexList, etc.

## Limits & SLA Hints
- Expected reload time, P95 search latency (if measured)
