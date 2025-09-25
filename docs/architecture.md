---
nav_order: 2
---

# Architecture

## Components

- **Qlik Sense App**
  - Loads lists of documents via `/documents`.
  - Performs on-demand search via `/search`.
  - Maintains preview tables for exploration and QA.

- **REST Connectors (per index)**
  - One Qlik REST connection per logical index (e.g., `Knowledge Fabric:REST_stitch-docs-documents`).
  - URLs are overridden at runtime in `WITH CONNECTION` for `/search`.

- **Backend (App Runner)**
  - Base URL (configurable): `https://hmxbpq4fmt.us-east-2.awsapprunner.com`.
  - Endpoints:
    - `GET /documents?index_name=…&from_=…&size=…`
    - `GET /search?q=…&index_name=…&from_=…&size=…`

## Data Flow

1. **Base load (per index):** call `/documents` to fetch canonical fields and populate:
   - `Docs_All` (deduplicated by `_id`)
   - `SearchIndex` (lowercased concatenation for local search)
2. **On-demand search:** when the user triggers **Search**, call `/search` with `vQueryPersist` and (optionally) `vIndexSel`, then:
   - Append to `Search_Results` (preview rows)
   - Append to `Docs_All` and `SearchIndex` if new IDs are found
3. **UI Binding:** KPIs and lists read from `Search_Results` and `Docs_All`.

## Tables

- **Docs_All**: normalized document store (union across indices)
- **SearchIndex**: local search text (for light client-side filtering)
- **Search_Results**: preview of last on-demand search
