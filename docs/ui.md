---
nav_order: 5
---

# UI & Interactions

## Search Input

- Bound to `vQueryPersist`. The script mirrors it into `vQuery` to keep the text visible after reload.

## Buttons

- **Search**
  - Action: `Set variable vDoSearch = 1` then **Reload**
- **Clear**
  - Actions:
    - `Set vQueryPersist = ''`
    - `Set vIndexSel = ''`
    - (Optional) `Set vDoSearch = 0`
    - **Reload**

## Result Panels

- **Results (Search_Results)** — preview rows per `/search`
- **Documents (Docs_All)** — consolidated deduped documents
- **Debug KPIs** — last timestamp, fresh row count, etc.
