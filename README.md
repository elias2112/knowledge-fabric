<<<<<<< HEAD
# Knowledge Fabric
This Qlik app integrates with multiple REST indexes to fetch, normalize, and store documents and metadata. It supports on-demand searches across single or multiple sources, builds searchable indexes, and manages results in structured tables for exploration, analysis, and downstream reporting.
=======
# Knowledge Fabric - Qlik App

Unified knowledge search across Qlik content sources, built entirely in Qlik Sense using REST connectors.

- **Live search** against the Knowledge Fabric backend (`/search`).
- **On-demand indexing** for multiple sources (`/documents`).
- **Local preview tables** to explore results and debug.
- **Minimal coupling**: backend URLs configurable via variables; per-index connections decoupled.

👉 Full documentation: https://elias2112.github.io/knowledge-fabric/

## Quick Start

1. Open the Qlik app (`.qvf`) in Qlik Sense (SaaS or Desktop).
2. Create REST connections (one per index) with the exact names used in the docs.
3. Update base variables (e.g. `vBaseUrl`) if needed.
4. Reload the app. Use the search input, pick an index, and hit **Search**.

> This repository contains documentation only. App scripts are captured in the docs for reference.
>>>>>>> origin/master
