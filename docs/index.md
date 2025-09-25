---
nav_order: 1
---

# Knowledge Fabric - Qlik App

**Unified knowledge search for Qlik**. This app queries a backend (“Knowledge Fabric”) to search and aggregate documents from multiple content indices (e.g., `stitch_docs`, `qlik_dev`, `help_pages`), presents results in a unified view, and keeps a local preview for validation and troubleshooting.

- **Tech:** Qlik Sense + REST Connector
- **Endpoints:** `/search` and `/documents`
- **Core tables:** `Docs_All`, `SearchIndex`, `Search_Results`
- **Key variables:** `vBaseUrl`, `vDoSearch`, `vQueryPersist`, `vIndexSel`

> This site documents architecture, configuration, scripts, and operations. It is meant for engineers and analytics developers.
