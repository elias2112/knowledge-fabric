---
nav_order: 6
---

# Data Model

## Docs_All

Union of documents across indices.

- Keys: `id`, `index_name`, `doc_base_id`, `chunk_no`
- Core fields: `content_title`, `content_text`, `chunk_text`, `content_html`, `source_url`, timestamps, tags
- Search audit: `loaded_at`, `loaded_at_num`, `loaded_from_search`

## SearchIndex

Local text index.

- `id`, `index_name`, `search_text`, `content_title`, `content_text`

## Search_Results

Preview of last on-demand search.

- `sr_id`, `sr_index`, `sr_title`, `sr_snippet`, `sr_type`, `sr_url`

## Notes

- Avoid synthetic keys by consistent field naming and deduplication via `Exists(id, [_id])`.
- Per-index loaders add optional/extra fields (e.g., `sfdc_*`, meta tags).
