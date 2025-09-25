---
nav_order: 2.5
---

# Walkthrough

This short, end-to-end guide shows how to run the app from a clean state, perform a search, and validate results.

## Prerequisites
- Qlik Sense (SaaS or Desktop).
- Working REST connections (one per index) with the exact names listed in [Setup](./setup.md).
- Network access to the backend base URL.

## 1. First Load (documents baseline)
1. Open the app.
2. Reload.  
   - The per-index loaders call `/documents` and populate:
     - **Docs_All** (deduped union across sources)
     - **SearchIndex** (local search text)

> Tip: Use the data model viewer to confirm `Docs_All` has rows and fields as expected.

## 2. Run a Search
1. In the **Search input**, type a query (e.g., `Taboola`).
2. (Optional) Select a single index in the **Index selector**.
3. Click **Search**. The button sets `vDoSearch=1` and triggers a reload.
4. After reload:
   - **Search_Results** shows the preview rows from `/search`.
   - **Docs_All** is appended with any new IDs returned, avoiding duplicates.

## 3. Clear & Reset
- Click **Clear** to wipe the query (`vQueryPersist=''`) and index (`vIndexSel=''`), then reload.
- The input remains stable and the app returns to the baseline state.

## 4. Validate the Results
- Compare `Search_Results.sr_id` with `Docs_All.id` to confirm deduping.
- Check `loaded_from_search=1` in `Docs_All` for rows sourced by `/search`.

## 5. Troubleshooting Fast Path
- If `/search` returns 500: ensure the query is not empty and quotes are normalized.
- If 422 about `size`: ensure effective size is never an empty string (see `vEffSize`).
- If “Connection not found”: verify REST connection names exactly match those in scripts.

## Appendix: UI Cheatsheet
- **Search**: sets `vDoSearch=1` → reload → appends results.
- **Clear**: resets `vQueryPersist`, `vIndexSel` → reload → baseline.

> For deeper details, see [Architecture](./architecture.md) and [Scripts](./scripts.md).
