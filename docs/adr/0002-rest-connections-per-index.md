# ADR 0002 — REST Connections per Index with URL Override

**Status:** Accepted  
**Date:** 2025-09-24

## Context
Qlik’s REST connectors are stateful and sometimes flaky if reused across different endpoints/params. We load baseline content from `/documents` per index and later reuse the same connection with `WITH CONNECTION` URL override for `/search`.

## Decision
Create one named REST connection per index (e.g., `Knowledge Fabric:REST_stitch-docs-documents`). For searches, keep using those connections and override the URL with `WITH CONNECTION (URL "$(vBaseUrl)/search")`. Avoid a single generic connection for everything.

## Alternatives Considered
- A single global REST connection → increased “connection not found” and caching issues.
- One connection per endpoint and per index → more management overhead without clear benefits.

## Consequences
- Positive: Stability, predictable authentication/headers, easier troubleshooting.
- Negative: A few more connections to maintain across environments.

## References
- `__ConnectByIndex` subroutine, per-index loaders, `WITH CONNECTION` usage.
