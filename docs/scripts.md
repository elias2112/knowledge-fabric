---
nav_order: 7
---

# Core Scripts

Below are the **canonical** script blocks used in the app. They are provided for reference/documentation and can be copied into Qlik as needed.

## Header (variables & empty tables)

// -----------------------------------------------------------------------------
// Script purpose
// -----------------------------------------------------------------------------
// This header script initializes variables and base tables for a search pipeline.
// It sets default config values (API limits, index filters), handles UI-driven
// variables (selected index, persisted query, latch behavior), validates flags,
// and prepares helper variables for ad-hoc fetches. It also ensures that empty
// base tables exist (0 rows) if they haven’t been loaded yet, preventing script
// errors in downstream steps. Finally, it emits a DEBUG trace of key UI vars.
// -----------------------------------------------------------------------------

// =================== HEADER (variables + base tables) ===================

// ====== DEV Config ======
LET vBaseUrl         = 'https://hmxbpq4fmt.us-east-2.awsapprunner.com';
LET vApiMaxSize      = 20;
LET vSearchBatchSize = 20;
LET vSearchWant      = Min($(vApiMaxSize),$(vSearchBatchSize));

LET vUseIndexFilter  = 1; // sends index_name
LET vIndexList = 'stitch_docs|stitch_conversations|qlik_dev|qlik_youtube_videos|sfdc_knowledge_articles|help_pages';

// ---------- UI variables ----------
IF Len(Trim('$(vIndexSel)'))=0 THEN 
LET vIndexSel = ''; END IF;

// Ensure the persisted query variable exists without overwriting it
IF Len(Trim('$(vQueryPersist)'))=0 THEN 
LET vQueryPersist = ''; END IF;

// Keep the search visible in the input / latch
IF Len(Trim('$(vQuery)'))=0 THEN 
LET vQuery = '$(vQueryPersist)'; END IF;
IF $(vDoSearch)=1 AND Len(Trim('$(vQueryPersist)'))>0 THEN 
LET vQuery = '$(vQueryPersist)'; END IF;

// If missing or not numeric, default to 0 (avoid calling IsNum() with empty)
IF Len(Trim('$(vDoSearch)'))=0 THEN
  LET vDoSearch = 0;
ELSE
  // Num#() on the string; Alt() falls back to 0 if parsing fails
  LET vDoSearch = Alt( Num#('$(vDoSearch)'), 0 );
END IF


SET ErrorMode=1;

// Helper for one-off fetch (leave empty by default)
IF Len(Trim('$(vFetchId)'))=0 THEN 
LET vFetchId = ''; END IF;

// --- Empty base tables (0 rows) if they don't exist ---
IF TableNumber('Docs_All') < 0 THEN
  Docs_All:
  LOAD
    '' AS id, '' AS index_name, '' AS doc_base_id, Null() AS chunk_no, 0 AS is_chunk,
    '' AS content_title, '' AS content_text, '' AS chunk_text, '' AS content_html,
    '' AS qlik_product, '' AS qlik_version, '' AS content_type, '' AS content_format,
    '' AS content_language, '' AS content_version, '' AS content_author, '' AS content_author_type,
    '' AS source, '' AS source_url, '' AS source_doc_id, '' AS source_ingestion_method,
    '' AS source_canonical_url, '' AS parent_id, '' AS security_authority, '' AS security_access_tier,
    0 AS security_pii_flag, 0 AS security_dlp_flag, Null() AS char_start, Null() AS char_end,
    Null() AS chunk_index, '' AS created_at, '' AS updated_at,
    '' AS loaded_at, 0 AS loaded_at_num, 0 AS loaded_from_search
  AUTOGENERATE 0;
END IF

IF TableNumber('SearchIndex') < 0 THEN
  SearchIndex:
  LOAD '' AS id, '' AS index_name, '' AS search_text, '' AS content_title, '' AS content_text
  AUTOGENERATE 0;
END IF

IF TableNumber('Search_Results') < 0 THEN
  Search_Results:
  LOAD '' AS sr_id, '' AS sr_index, '' AS sr_title, '' AS sr_snippet, '' AS sr_type, '' AS sr_url
  AUTOGENERATE 0;
END IF

TRACE [DEBUG] vDoSearch=$(vDoSearch) vQueryPersist='$(vQueryPersist)' vQuery='$(vQuery)';



## `__ConnectByIndex`

// -----------------------------------------------------------------------------
// Subroutine purpose
// -----------------------------------------------------------------------------
// __ConnectByIndex maps a given index name to a pre-created REST connection,
// validates that a matching connection exists, and then performs LIB CONNECT.
// On success, it sets vLastConnSet to the connection name. If no mapping is
// found, it logs a TRACE, clears vLastConnSet, and exits the subroutine.
// -----------------------------------------------------------------------------

// =================== SUB __ConnectByIndex ===================
SUB __ConnectByIndex(pIndex)
  // Map each index to the name of the existing REST connection
  LET vConn = '';

  IF '$(pIndex)'='stitch_docs'          THEN 
  LET vConn = 'Knowledge Fabric:REST_stitch-docs-documents'; END IF
  IF '$(pIndex)'='stitch_conversations' THEN 
  LET vConn = 'Knowledge Fabric:REST_stitch-conversations-documents'; END IF
  IF '$(pIndex)'='qlik_dev'             THEN 
  LET vConn = 'Knowledge Fabric:REST_qlik-dev-documents'; END IF
  IF '$(pIndex)'='qlik_youtube_videos'  THEN 
  LET vConn = 'Knowledge Fabric:REST_qlik-youtube-videos-documents'; END IF
  IF '$(pIndex)'='sfdc_knowledge_articles' THEN 
  LET vConn = 'Knowledge Fabric:REST_sfdc-knowledge-articles-documents'; END IF
  IF '$(pIndex)'='help_pages'           THEN 
  LET vConn = 'Knowledge Fabric:REST_help-pages-documents'; END IF

  // If no connection was set, stop here
  IF Len(Trim('$(vConn)'))=0 THEN
    TRACE [ConnectByIndex] ABORT: no hay conexión para index $(pIndex);
    LET vLastConnSet = '';
    EXIT SUB;
  END IF

  LIB CONNECT TO '$(vConn)';
  LET vLastConnSet = '$(vConn)';
END SUB

## Stitch Docs

// -----------------------------------------------------------------------------
// Script purpose
// -----------------------------------------------------------------------------
// This script connects to the REST source for the 'stitch_docs' index, queries
// the total number of available documents, and defines utilities to page through
// the /documents endpoint. It loads records into two models:
//   - Docs_All: normalized document/chunk fields
//   - SearchIndex: prebuilt lowercased search text for efficient lookups
// It first fetches the total, then (in Section A) loads up to vListSize items,
// appending them into the base tables via __AppendFromMaster.
// -----------------------------------------------------------------------------

// ---------- CONNECTION ----------
LIB CONNECT TO 'Knowledge Fabric:REST_stitch-docs-documents';

// ======================== CONFIG ========================
LET vBaseList    = 'https://hmxbpq4fmt.us-east-2.awsapprunner.com/documents';
LET vBaseSearch  = 'https://hmxbpq4fmt.us-east-2.awsapprunner.com/search';
LET vIndex       = 'stitch_docs';
LET vListSize    = 10;

// ===================== READ 'total' ======================
RestConnectorMasterTable:
SQL SELECT 
  "total","__KEY_root",
  (SELECT "_id" FROM "documents" FK "__FK_documents")
FROM JSON (wrap on) "root" PK "__KEY_root"
WITH CONNECTION (
  URL "$(vBaseList)",
  QUERY "index_name" "$(vIndex)",
  QUERY "from_" "0",
  QUERY "size" "1"
);

__TotalTmp:
LOAD total RESIDENT RestConnectorMasterTable WHERE NOT IsNull(total);
LET vTotal = Alt(Peek('total',0,'__TotalTmp'),0);
DROP TABLE __TotalTmp;
DROP TABLE RestConnectorMasterTable;

// =============== SUB: APPEND FROM MASTER ===============
SUB __AppendFromMaster
  IF TableNumber('RestConnectorMasterTable') >= 0 THEN

    // ---- Docs_All (create or implicitly append) ----
    Docs_All:
    LOAD
      [_id] AS id,
      '$(vIndex)' AS index_name,
      If(WildMatch([_id],'*_chunk_*'), SubField([_id],'_chunk_',1), [_id]) AS doc_base_id,
      If(WildMatch([_id],'*_chunk_*'), Num(SubField([_id],'_chunk_',2)), Null()) AS chunk_no,
      If(WildMatch([_id],'*_chunk_*'), 1, 0) AS is_chunk,
      content_title, content_text, chunk_text, content_html,
      qlik_product, qlik_version, content_type, content_format, content_language,
      content_version, content_author, content_author_type,
      source, source_url, source_doc_id, source_ingestion_method, source_canonical_url, parent_id,
      security_authority, security_access_tier, security_pii_flag, security_dlp_flag,
      char_start, char_end, chunk_index,
      created_at, updated_at
    RESIDENT RestConnectorMasterTable
    WHERE NOT IsNull([__FK_documents]) AND NOT Exists(id, [_id]);

    // ---- SearchIndex ----
    SearchIndex:
    LOAD
      [_id] AS id,
      '$(vIndex)' AS index_name,
      Lower( Alt(content_title,'') & ' ' & Alt(chunk_text,'') & ' ' & Alt(content_text,'') ) AS search_text,
      content_title,
      content_text
    RESIDENT RestConnectorMasterTable
    WHERE NOT IsNull([__FK_documents]) AND NOT Exists(id, [_id]);

    DROP TABLE RestConnectorMasterTable;
  END IF
END SUB

// =============== SUB: /documents ========================
SUB LoadPage_List(pFrom, pSize)
  SET ErrorMode=0;
  RestConnectorMasterTable:
  SQL SELECT 
    "total","__KEY_root",
    (SELECT 
      "_id","content_title","content_text","chunk_text","content_html",
      "qlik_product","qlik_version","content_type","content_format","content_language",
      "content_version","content_author","content_author_type",
      "source","source_url","source_doc_id","source_ingestion_method","source_canonical_url","parent_id",
      "security_authority","security_access_tier","security_pii_flag","security_dlp_flag",
      "char_start","char_end","chunk_index",
      "created_at","updated_at",
      "__FK_documents"
    FROM "documents" FK "__FK_documents")
  FROM JSON (wrap on) "root" PK "__KEY_root"
  WITH CONNECTION (
    URL "$(vBaseList)",
    QUERY "index_name" "$(vIndex)",
    QUERY "from_" "$(pFrom)",
    QUERY "size" "$(pSize)"
  );
  SET ErrorMode=1;
END SUB

// ================= SECTION A: only 10 ====================
LET vFrom = 0;
LET vCapA = RangeMin($(vListSize), $(vTotal));
IF $(vCapA) > 0 THEN
  CALL LoadPage_List($(vFrom), $(vCapA));
  CALL __AppendFromMaster
END IF

## SFCD Knolwdge Articles

// -----------------------------------------------------------------------------
// Script purpose
// -----------------------------------------------------------------------------
// Connects to the REST source for the 'sfdc_knowledge_articles' index, retrieves
// the total number of documents, and defines utilities to page through the
// /documents endpoint. It appends records into:
//   - Docs_All: normalized document/chunk fields plus extracted SFDC fields
//   - SearchIndex: lowercased searchable text including extracted SFDC fields
// Section A loads up to vListSize items and appends them via __AppendFromMaster.
// -----------------------------------------------------------------------------

// ---------- CONNECTION ----------
LIB CONNECT TO 'Knowledge Fabric:REST_sfdc_knowledge_articles-documents';

// ======================== CONFIG ========================
LET vBaseList    = 'https://hmxbpq4fmt.us-east-2.awsapprunner.com/documents';
LET vIndex       = 'sfdc_knowledge_articles';
LET vListSize    = 10;

// ===================== READ 'total' ======================
RestConnectorMasterTable:
SQL SELECT 
  "total","__KEY_root",
  (SELECT "_id" FROM "documents" FK "__FK_documents")
FROM JSON (wrap on) "root" PK "__KEY_root"
WITH CONNECTION (
  URL "$(vBaseList)",
  QUERY "index_name" "$(vIndex)",
  QUERY "from_" "0",
  QUERY "size" "1"
);

__TotalTmp:
LOAD total RESIDENT RestConnectorMasterTable WHERE NOT IsNull(total);
LET vTotal = Alt(Peek('total',0,'__TotalTmp'),0);
DROP TABLE __TotalTmp;
DROP TABLE RestConnectorMasterTable;

// =============== SUB: APPEND FROM MASTER ===============
SUB __AppendFromMaster
  IF TableNumber('RestConnectorMasterTable') >= 0 THEN

    Docs_All:
    LOAD
      [_id] AS id,
      '$(vIndex)' AS index_name,
      If(WildMatch([_id],'*_chunk_*'), SubField([_id],'_chunk_',1), [_id]) AS doc_base_id,
      If(WildMatch([_id],'*_chunk_*'), Num(SubField([_id],'_chunk_',2)), Null()) AS chunk_no,
      If(WildMatch([_id],'*_chunk_*'), 1, 0) AS is_chunk,
      content_title, content_text, chunk_text, content_html,
      Replace(Alt(content_text, chunk_text),'\"','"') AS content_text_unescaped,
      TextBetween(Replace(Alt(content_text, chunk_text),'\"','"'), '"cause": "',       '", "description"') AS sfdc_cause,
      TextBetween(Replace(Alt(content_text, chunk_text),'\"','"'), '"description": "', '", "resolution"') AS sfdc_description,
      TextBetween(Replace(Alt(content_text, chunk_text),'\"','"'), '"resolution": "',  '"}')               AS sfdc_resolution,
      qlik_product, qlik_version, content_type, content_format, content_language,
      content_version, content_author, content_author_type,
      source, source_url, source_doc_id, source_ingestion_method, source_canonical_url, parent_id,
      security_authority, security_access_tier, security_pii_flag, security_dlp_flag,
      char_start, char_end, chunk_index,
      created_at, updated_at
    RESIDENT RestConnectorMasterTable
    WHERE NOT IsNull([__FK_documents]) AND NOT Exists(id, [_id]);

    SearchIndex:
    LOAD
      [_id] AS id,
      '$(vIndex)' AS index_name,
      Lower(
        Alt(content_title,'') & ' ' &
        Alt(content_text,'') & ' ' &
        Alt(chunk_text,'') & ' ' &
        Alt(TextBetween(Replace(Alt(content_text, chunk_text),'\"','"'), '"description": "', '", "resolution"'),'') & ' ' &
        Alt(TextBetween(Replace(Alt(content_text, chunk_text),'\"','"'), '"resolution": "',  '"}'),'') & ' ' &
        Alt(TextBetween(Replace(Alt(content_text, chunk_text),'\"','"'), '"cause": "',       '", "description"'),'')
      ) AS search_text,
      content_title,
      Alt(content_text, chunk_text) AS content_text
    RESIDENT RestConnectorMasterTable
    WHERE NOT IsNull([__FK_documents]) AND NOT Exists(id, [_id]);

    DROP TABLE RestConnectorMasterTable;
  END IF
END SUB

// =============== SUB: /documents ========================
SUB LoadPage_List(pFrom, pSize)
  SET ErrorMode=0;
  RestConnectorMasterTable:
  SQL SELECT 
    "total","__KEY_root",
    (SELECT 
      "_id","content_title","content_text","chunk_text","content_html",
      "qlik_product","qlik_version","content_type","content_format","content_language",
      "content_version","content_author","content_author_type",
      "source","source_url","source_doc_id","source_ingestion_method","source_canonical_url","parent_id",
      "security_authority","security_access_tier","security_pii_flag","security_dlp_flag",
      "char_start","char_end","chunk_index",
      "created_at","updated_at",
      "__FK_documents"
    FROM "documents" FK "__FK_documents")
  FROM JSON (wrap on) "root" PK "__KEY_root"
  WITH CONNECTION (
    URL "$(vBaseList)",
    QUERY "index_name" "$(vIndex)",
    QUERY "from_" "$(pFrom)",
    QUERY "size" "$(pSize)"
  );
  SET ErrorMode=1;
END SUB

// ================= SECTION A: only 10 ====================
LET vFrom = 0;
LET vCapA = RangeMin($(vListSize), $(vTotal));
IF $(vCapA) > 0 THEN
  CALL LoadPage_List($(vFrom), $(vCapA));
  CALL __AppendFromMaster
END IF

## Qlik Youtube

// -----------------------------------------------------------------------------
// Script purpose
// -----------------------------------------------------------------------------
// Connects to the REST source for the 'qlik_youtube_videos' index, retrieves
// the total number of documents, and provides utilities to page through the
// /documents endpoint. It appends records into:
//   - Docs_All: normalized document/chunk fields (handling content_text/content)
//   - SearchIndex: lowercased searchable text (title + content variants + chunks)
// Section A loads up to vListSize items and appends them via __AppendFromMaster.
// -----------------------------------------------------------------------------

// ---------- CONNECTION ----------
LIB CONNECT TO 'Knowledge Fabric:REST_qlik-youtube-videos-documents';

// ======================== CONFIG ========================
LET vBaseList    = 'https://hmxbpq4fmt.us-east-2.awsapprunner.com/documents';
LET vIndex       = 'qlik_youtube_videos';
LET vListSize    = 10;

// ===================== READ 'total' ======================
RestConnectorMasterTable:
SQL SELECT 
  "total","__KEY_root",
  (SELECT "_id" FROM "documents" FK "__FK_documents")
FROM JSON (wrap on) "root" PK "__KEY_root"
WITH CONNECTION (
  URL "$(vBaseList)",
  QUERY "index_name" "$(vIndex)",
  QUERY "from_" "0",
  QUERY "size" "1"
);

__TotalTmp:
LOAD total RESIDENT RestConnectorMasterTable WHERE NOT IsNull(total);
LET vTotal = Alt(Peek('total',0,'__TotalTmp'),0);
DROP TABLE __TotalTmp;
DROP TABLE RestConnectorMasterTable;

// =============== SUB: APPEND FROM MASTER ===============
SUB __AppendFromMaster
  IF TableNumber('RestConnectorMasterTable') >= 0 THEN

    Docs_All:
    LOAD
      [_id] AS id,
      '$(vIndex)' AS index_name,
      If(WildMatch([_id],'*_chunk_*'),SubField([_id],'_chunk_',1),[_id]) AS doc_base_id,
      If(WildMatch([_id],'*_chunk_*'),Num(SubField([_id],'_chunk_',2)),Null()) AS chunk_no,
      If(WildMatch([_id],'*_chunk_*'),1,0) AS is_chunk,
      content_title,
      Alt(content_text, content, chunk_text) AS content_text,
      chunk_text, content_html,
      qlik_product, qlik_version, content_type, content_format, content_language,
      content_version, content_author, content_author_type,
      source, source_url, source_doc_id, source_ingestion_method, source_canonical_url, parent_id,
      security_authority, security_access_tier, security_pii_flag, security_dlp_flag,
      char_start, char_end, chunk_index,
      created_at, updated_at
    RESIDENT RestConnectorMasterTable
    WHERE NOT IsNull([__FK_documents]) AND NOT Exists(id, [_id]);

    SearchIndex:
    LOAD
      [_id] AS id,
      '$(vIndex)' AS index_name,
      Lower(Alt(content_title,'') & ' ' & Alt(content_text,'') & ' ' & Alt(content,'') & ' ' & Alt(chunk_text,'')) AS search_text,
      content_title,
      Alt(content_text, content, chunk_text) AS content_text
    RESIDENT RestConnectorMasterTable
    WHERE NOT IsNull([__FK_documents]) AND NOT Exists(id, [_id]);

    DROP TABLE RestConnectorMasterTable;
  END IF
END SUB

// =============== SUB: /documents ========================
SUB LoadPage_List(pFrom, pSize)
  SET ErrorMode=0;
  RestConnectorMasterTable:
  SQL SELECT 
    "total","__KEY_root",
    (SELECT 
      "_id","content_title","content_text","content","chunk_text","content_html",
      "qlik_product","qlik_version","content_type","content_format","content_language",
      "content_version","content_author","content_author_type",
      "source","source_url","source_doc_id","source_ingestion_method","source_canonical_url","parent_id",
      "security_authority","security_access_tier","security_pii_flag","security_dlp_flag",
      "char_start","char_end","chunk_index",
      "created_at","updated_at",
      "__FK_documents"
    FROM "documents" FK "__FK_documents")
  FROM JSON (wrap on) "root" PK "__KEY_root"
  WITH CONNECTION (
    URL "$(vBaseList)",
    QUERY "index_name" "$(vIndex)",
    QUERY "from_" "$(pFrom)",
    QUERY "size" "$(pSize)"
  );
  SET ErrorMode=1;
END SUB

// ================= SECTION A: only 10 ====================
LET vFrom = 0;
LET vCapA = RangeMin($(vListSize), $(vTotal));
IF $(vCapA) > 0 THEN
  CALL LoadPage_List($(vFrom), $(vCapA));
  CALL __AppendFromMaster
END IF

## Help Pages

// 
//  Script purpose
//  
//  Connects to the REST source for the 'help_pages' index, retrieves the total
//  number of documents, and defines utilities to page through the /documents
//  endpoint. It appends records into:
//    - Docs_All: normalized document/chunk fields (includes conversation_length)
//    - SearchIndex: lowercased searchable text (title + text + chunks + HTML)
//  Section A loads up to vListSize items and appends them via __AppendFromMaster.
//

// ---------- CONNECTION ----------
LIB CONNECT TO 'Knowledge Fabric:REST_help-pages-documents';

// ======================== CONFIG ========================
LET vBaseList    = 'https://hmxbpq4fmt.us-east-2.awsapprunner.com/documents';
LET vIndex       = 'help_pages';
LET vListSize    = 10;

// ===================== READ 'total' ======================
RestConnectorMasterTable:
SQL SELECT 
  "total","__KEY_root",
  (SELECT "_id" FROM "documents" FK "__FK_documents")
FROM JSON (wrap on) "root" PK "__KEY_root"
WITH CONNECTION (
  URL "$(vBaseList)",
  QUERY "index_name" "$(vIndex)",
  QUERY "from_" "0",
  QUERY "size" "1"
);

__TotalTmp:
LOAD total RESIDENT RestConnectorMasterTable WHERE NOT IsNull(total);
LET vTotal = Alt(Peek('total',0,'__TotalTmp'),0);
DROP TABLE __TotalTmp;
DROP TABLE RestConnectorMasterTable;

// =============== SUB: APPEND FROM MASTER ===============
SUB __AppendFromMaster
  IF TableNumber('RestConnectorMasterTable') >= 0 THEN

    Docs_All:
    LOAD
      [_id] AS id,
      '$(vIndex)' AS index_name,
      If(WildMatch([_id],'*_chunk_*'),SubField([_id],'_chunk_',1),[_id]) AS doc_base_id,
      If(WildMatch([_id],'*_chunk_*'),Num(SubField([_id],'_chunk_',2)),Null()) AS chunk_no,
      If(WildMatch([_id],'*_chunk_*'),1,0) AS is_chunk,
      content_title, content_text, chunk_text, content_html,
      qlik_product, qlik_version, content_type, content_language,
      content_author, content_author_type,
      source, source_url, source_doc_id, source_ingestion_method, source_canonical_url, parent_id,
      security_authority, security_access_tier, security_pii_flag, security_dlp_flag,
      conversation_length,
      char_start, char_end, chunk_index,
      created_at, updated_at
    RESIDENT RestConnectorMasterTable
    WHERE NOT IsNull([__FK_documents]) AND NOT Exists(id, [_id]);

    SearchIndex:
    LOAD
      [_id] AS id,
      '$(vIndex)' AS index_name,
      Lower(Alt(content_title,'') & ' ' & Alt(content_text,'') & ' ' & Alt(chunk_text,'') & ' ' & Alt(content_html,'')) AS search_text,
      content_title,
      content_text
    RESIDENT RestConnectorMasterTable
    WHERE NOT IsNull([__FK_documents]) AND NOT Exists(id, [_id]);

    DROP TABLE RestConnectorMasterTable;
  END IF
END SUB

// =============== SUB: /documents ========================
SUB LoadPage_List(pFrom, pSize)
  SET ErrorMode=0;
  RestConnectorMasterTable:
  SQL SELECT 
    "total","__KEY_root",
    (SELECT 
      "_id","content_title","content_text","chunk_text","content_html",
      "qlik_product","qlik_version","content_type","content_language",
      "content_author","content_author_type",
      "source","source_url","source_doc_id","source_ingestion_method","source_canonical_url","parent_id",
      "security_authority","security_access_tier","security_pii_flag","security_dlp_flag",
      "conversation_length",
      "char_start","char_end","chunk_index",
      "created_at","updated_at",
      "__FK_documents"
    FROM "documents" FK "__FK_documents")
  FROM JSON (wrap on) "root" PK "__KEY_root"
  WITH CONNECTION (
    URL "$(vBaseList)",
    QUERY "index_name" "$(vIndex)",
    QUERY "from_" "$(pFrom)",
    QUERY "size" "$(pSize)"
  );
  SET ErrorMode=1;
END SUB

// ================= SECTION A: only 10 ====================
LET vFrom = 0;
LET vCapA = RangeMin($(vListSize), $(vTotal));
IF $(vCapA) > 0 THEN
  CALL LoadPage_List($(vFrom), $(vCapA));
  CALL __AppendFromMaster
END IF

## Qlik Dev

// -----------------------------------------------------------------------------
// Script purpose
// -----------------------------------------------------------------------------
// Connects to the REST source for the 'qlik_dev' index, retrieves the total
// number of documents, and defines utilities to page through the /documents
// endpoint. It appends records into:
//   - Docs_All: normalized document/chunk fields plus Qlik Dev metadata
//   - SearchIndex: lowercased searchable text (title + text + chunks + HTML/meta)
// Section A loads up to vListSize items and appends them via __AppendFromMaster.
// -----------------------------------------------------------------------------

// ---------- CONNECTION ----------
LIB CONNECT TO 'Knowledge Fabric:REST_qlik-dev-documents';

// ======================== CONFIG ========================
LET vBaseList    = 'https://hmxbpq4fmt.us-east-2.awsapprunner.com/documents';
LET vIndex       = 'qlik_dev';
LET vListSize    = 10;

// ===================== READ 'total' ======================
RestConnectorMasterTable:
SQL SELECT 
  "total","__KEY_root",
  (SELECT "_id" FROM "documents" FK "__FK_documents")
FROM JSON (wrap on) "root" PK "__KEY_root"
WITH CONNECTION (
  URL "$(vBaseList)",
  QUERY "index_name" "$(vIndex)",
  QUERY "from_" "0",
  QUERY "size" "1"
);

__TotalTmp:
LOAD total RESIDENT RestConnectorMasterTable WHERE NOT IsNull(total);
LET vTotal = Alt(Peek('total',0,'__TotalTmp'),0);
DROP TABLE __TotalTmp;
DROP TABLE RestConnectorMasterTable;

// =============== SUB: APPEND FROM MASTER ===============
SUB __AppendFromMaster
  IF TableNumber('RestConnectorMasterTable') >= 0 THEN

    Docs_All:
    LOAD
      [_id] AS id,
      '$(vIndex)' AS index_name,
      If(WildMatch([_id],'*_chunk_*'),SubField([_id],'_chunk_',1),[_id]) AS doc_base_id,
      If(WildMatch([_id],'*_chunk_*'),Num(SubField([_id],'_chunk_',2)),Null()) AS chunk_no,
      If(WildMatch([_id],'*_chunk_*'),1,0) AS is_chunk,
      content_title, content_text, chunk_text, content_html,
      _http_status, _meta_canonical, _meta_description, _meta_robots,
      qlik_product, qlik_version, content_type, content_format, content_language,
      content_version, content_author, content_author_type,
      source, source_url, source_doc_id, source_ingestion_method, source_canonical_url, parent_id,
      security_authority, security_access_tier, security_pii_flag, security_dlp_flag,
      char_start, char_end, chunk_index,
      created_at, updated_at,
      content_citation_title, content_keywords, content_topics, content_is_validated,
      intent_categories, intent_difficulty_level, intent_main_audience, vector_embedding_model, vector_dim
    RESIDENT RestConnectorMasterTable
    WHERE NOT IsNull([__FK_documents]) AND NOT Exists(id, [_id]);

    SearchIndex:
    LOAD
      [_id] AS id,
      '$(vIndex)' AS index_name,
      Lower( Alt(content_title,'') & ' ' & Alt(content_text,'') & ' ' & Alt(chunk_text,'') & ' ' & Alt(content_html,'') & ' ' & Alt(_meta_description,'') ) AS search_text,
      content_title,
      content_text
    RESIDENT RestConnectorMasterTable
    WHERE NOT IsNull([__FK_documents]) AND NOT Exists(id, [_id]);

    DROP TABLE RestConnectorMasterTable;
  END IF
END SUB

// =============== SUB: /documents ========================
SUB LoadPage_List(pFrom, pSize)
  SET ErrorMode=0;
  RestConnectorMasterTable:
  SQL SELECT 
    "total","__KEY_root",
    (SELECT 
      "_id","content_title","content_text","chunk_text","content_html",
      "_http_status","_meta_canonical","_meta_description","_meta_robots",
      "qlik_product","qlik_version","content_type","content_format","content_language",
      "content_version","content_author","content_author_type",
      "source","source_url","source_doc_id","source_ingestion_method","source_canonical_url","parent_id",
      "security_authority","security_access_tier","security_pii_flag","security_dlp_flag",
      "char_start","char_end","chunk_index",
      "created_at","updated_at",
      "content_citation_title","content_keywords","content_topics","content_is_validated",
      "intent_categories","intent_difficulty_level","intent_main_audience","vector_embedding_model","vector_dim",
      "__FK_documents"
    FROM "documents" FK "__FK_documents")
  FROM JSON (wrap on) "root" PK "__KEY_root"
  WITH CONNECTION (
    URL "$(vBaseList)",
    QUERY "index_name" "$(vIndex)",
    QUERY "from_" "$(pFrom)",
    QUERY "size" "$(pSize)"
  );
  SET ErrorMode=1;
END SUB

// ================= SECTION A: only 10 ====================
LET vFrom = 0;
LET vCapA = RangeMin($(vListSize), $(vTotal));
IF $(vCapA) > 0 THEN
  CALL LoadPage_List($(vFrom), $(vCapA));
  CALL __AppendFromMaster
END IF


## Helper

// -----------------------------------------------------------------------------
// Subroutine purpose
// -----------------------------------------------------------------------------
// __ConnectByIndex performs a LIB CONNECT to the proper pre-defined REST
// connection based on the provided index name (pIndex). It uses a simple
// conditional mapping (IF/ELSEIF chain) to select the target connection.
// No variables are set; it only issues the LIB CONNECT statement.
// -----------------------------------------------------------------------------

SUB __ConnectByIndex(pIndex)
  IF '$(pIndex)'='stitch_docs'               THEN 
    LIB CONNECT TO 'Knowledge Fabric:REST_stitch-docs-documents';
  ELSEIF '$(pIndex)'='stitch_conversations'  THEN 
    LIB CONNECT TO 'Knowledge Fabric:REST_stitch-conversations-documents';
  ELSEIF '$(pIndex)'='qlik_dev'              THEN 
    LIB CONNECT TO 'Knowledge Fabric:REST_qlik-dev-documents';
  ELSEIF '$(pIndex)'='qlik_youtube_videos'   THEN 
    LIB CONNECT TO 'Knowledge Fabric:REST_qlik-youtube-videos-documents';
  ELSEIF '$(pIndex)'='sfdc_knowledge_articles' THEN 
    LIB CONNECT TO 'Knowledge Fabric:REST_sfdc-knowledge-articles-documents';
  ELSEIF '$(pIndex)'='help_pages'            THEN 
    LIB CONNECT TO 'Knowledge Fabric:REST_help-pages-documents';
  ENDIF
END SUB

## Run Search

// -----------------------------------------------------------------------------
// Subroutine purpose
// -----------------------------------------------------------------------------
// __RunSearch executes a search query against the API for a given index. It:
//   - Normalizes the query (removes stray quotes, checks for empty)
//   - Validates and sets effective size
//   - Optionally includes the index_name parameter
//   - Calls the REST API /search endpoint
//   - Counts hits and logs them
//   - Appends results to:
//       • Search_Results (preview table)
//       • Docs_All (detailed docs, avoiding duplicates)
//       • SearchIndex (local search index, avoiding duplicates)
// If no results are found, it exits without appending anything.
// -----------------------------------------------------------------------------

// =================== SUB __RunSearch ===================
SUB __RunSearch(pIndex, pQ, pSize)

    // normalize query (removes single quotes) and validate empty
    LET vQ = Trim(Replace('$(pQ)','''',''));
    IF Len(Trim('$(vQ)'))=0 THEN
        TRACE [RunSearch] ABORT: empty query for index $(pIndex);
        EXIT SUB;
    END IF

    // effective size (avoid size="")
    LET vEffSize = $(vApiMaxSize);
    IF Len(Trim('$(pSize)'))>0 THEN
      // Only parse when we know it's non-empty
      LET vEffSize = Alt( Num#('$(pSize)'), $(vEffSize) );
    END IF


    // optional index_name parameter
    LET vIndexParam = '';
    IF $(vUseIndexFilter)=1 THEN
      LET vIndexParam = 'QUERY "index_name" "' & '$(pIndex)' & '",';
    END IF

    TRACE [RunSearch] index=$(pIndex) q="$(vQ)" size=$(vEffSize);

    // REST call (overrides URL)
    SET ErrorMode=0;
    RestConnectorMasterTable:
    SQL SELECT 
        "total",
        "__KEY_root",
        (SELECT 
            "_id","content_title","content_text","content","chunk_text","content_html",
            "qlik_product","qlik_version","content_type","content_language","content_author",
            "source","source_url","parent_id","__FK_documents"
         FROM "documents" FK "__FK_documents")
    FROM JSON (wrap on) "root" PK "__KEY_root"
    WITH CONNECTION (
        URL "$(vBaseUrl)/search",
        $(vIndexParam)
        QUERY "q"      "$(vQ)",
        QUERY "from_"  "0",
        QUERY "size"   "$(vEffSize)"
    );
    SET ErrorMode=1;

    // count hits
    __Cnt:
    LOAD Count([__FK_documents]) AS n
    RESIDENT RestConnectorMasterTable
    WHERE NOT IsNull([__FK_documents]);
    LET vGot = Alt(Peek('n',0,'__Cnt'),0);
    DROP TABLE __Cnt;
    TRACE [RunSearch] node=__FK_documents | hits=$(vGot);

    IF $(vGot)=0 THEN
        DROP TABLE RestConnectorMasterTable;
        EXIT SUB;
    END IF

    // append to Search_Results (preview)
    CONCATENATE (Search_Results)
    LOAD
        [_id]            AS sr_id,
        '$(pIndex)'      AS sr_index,
        content_title    AS sr_title,
        If(Len(Trim(content_text))>0, Left(content_text,300),
           If(Len(Trim(chunk_text))>0, Left(chunk_text,300),
              If(Len(Trim(content))>0, Left(content,300), Null()))) AS sr_snippet,
        content_type     AS sr_type,
        source_url       AS sr_url
    RESIDENT RestConnectorMasterTable
    WHERE NOT IsNull([__FK_documents]);

    // append to Docs_All (avoids duplicates)
    CONCATENATE (Docs_All)
    LOAD
        [_id] AS id,
        '$(pIndex)' AS index_name,
        If(WildMatch([_id],'*_chunk_*'), SubField([_id],'_chunk_',1), [_id])         AS doc_base_id,
        If(WildMatch([_id],'*_chunk_*'), Num(SubField([_id],'_chunk_',2)), Null())   AS chunk_no,
        If(WildMatch([_id],'*_chunk_*'), 1, 0)                                       AS is_chunk,
        content_title,
        If(Len(Trim(content_text))>0, content_text,
           If(Len(Trim(chunk_text))>0, chunk_text, content))                         AS content_text,
        chunk_text, content_html,
        qlik_product, qlik_version, content_type, content_language, content_author,
        source, source_url, parent_id,
        Timestamp(Now()) AS loaded_at,
        Num(Now())       AS loaded_at_num,
        1                AS loaded_from_search
    RESIDENT RestConnectorMasterTable
    WHERE NOT IsNull([__FK_documents])
      AND NOT Exists(id, [_id]);

    // append to SearchIndex
    CONCATENATE (SearchIndex)
    LOAD
        [_id] AS id,
        '$(pIndex)' AS index_name,
        Lower( Alt(content_title,'') & ' ' & Alt(content_text,'') & ' ' & Alt(content,'') & ' ' & Alt(chunk_text,'') ) AS search_text,
        content_title,
        If(Len(Trim(content_text))>0, content_text, If(Len(Trim(chunk_text))>0, chunk_text, content)) AS content_text
    RESIDENT RestConnectorMasterTable
    WHERE NOT IsNull([__FK_documents])
      AND NOT Exists(id, [_id]);

    DROP TABLE RestConnectorMasterTable;

END SUB


## __FetchById

// -----------------------------------------------------------------------------
// Subroutine purpose
// -----------------------------------------------------------------------------
// __FetchById fetches a specific document from the API by its ID for a given
// index. It:
//   - Ensures the correct REST connection for the index
//   - Normalizes and validates the input id
//   - Optionally adds index_name as a filter
//   - Calls /search with q = id (backend supports id search)
//   - If no hits are found, it exits
//   - Otherwise, appends results into:
//       • Docs_All (with flag 2 = fetched by id)
//       • SearchIndex (local search index, avoiding duplicates)
// -----------------------------------------------------------------------------

SUB __FetchById(pIndex, pId)
  // Ensure proper REST connection for the target index
  CALL __ConnectByIndex('$(pIndex)');

  // Normalize and validate the document id
  LET vDocId = Trim(Replace('$(pId)','''',''));
  IF Len(Trim('$(vDocId)'))=0 THEN
    TRACE [FetchById] ABORT: empty id for index $(pIndex);
    EXIT SUB;
  END IF

  // Optional index_name parameter
  LET vIndexParam = '';
  IF $(vUseIndexFilter)=1 THEN
    LET vIndexParam = 'QUERY "index_name" "' & '$(pIndex)' & '",';
  END IF

  // Call /search with q = id (backend supports matching by id via search)
  SET ErrorMode=0;
  RestConnectorMasterTable:
  SQL SELECT 
    "total","__KEY_root",
    (SELECT 
      "_id","content_title","content_text","content","chunk_text","content_html",
      "qlik_product","qlik_version","content_type","content_language","content_author",
      "source","source_url","parent_id","__FK_documents"
    FROM "documents" FK "__FK_documents")
  FROM JSON (wrap on) "root" PK "__KEY_root"
  WITH CONNECTION (
    URL "$(vBaseUrl)/search",
    $(vIndexParam)
    QUERY "q" "$(vDocId)",
    QUERY "from_" "0",
    QUERY "size" "1"
  );
  SET ErrorMode=1;

  // Guard: no hits -> cleanup and exit
  __Cnt:
  LOAD Count([__FK_documents]) AS n
  RESIDENT RestConnectorMasterTable
  WHERE NOT IsNull([__FK_documents]);
  LET vGot = Alt(Peek('n',0,'__Cnt'),0);
  DROP TABLE __Cnt;
  TRACE [FetchById] hits=$(vGot);

  IF $(vGot)=0 THEN
    DROP TABLE RestConnectorMasterTable;
    EXIT SUB;
  END IF

  // Append to Docs_All (avoid duplicates)
  CONCATENATE (Docs_All)
  LOAD
    [_id] AS id,
    '$(pIndex)' AS index_name,
    If(WildMatch([_id],'*_chunk_*'), SubField([_id],'_chunk_',1), [_id]) AS doc_base_id,
    If(WildMatch([_id],'*_chunk_*'), Num(SubField([_id],'_chunk_',2)), Null()) AS chunk_no,
    If(WildMatch([_id],'*_chunk_*'), 1, 0) AS is_chunk,
    content_title,
    If(Len(Trim(content_text))>0, content_text,
       If(Len(Trim(chunk_text))>0, chunk_text, content)) AS content_text,
    chunk_text, content_html,
    qlik_product, qlik_version, content_type, content_language, content_author,
    source, source_url, parent_id,
    Timestamp(Now()) AS loaded_at,
    Num(Now())       AS loaded_at_num,
    2                AS loaded_from_search   // 2 = fetched by id
  RESIDENT RestConnectorMasterTable
  WHERE NOT IsNull([__FK_documents])
    AND NOT Exists(id, [_id]);

  // Append to local SearchIndex (avoid duplicates)
  CONCATENATE (SearchIndex)
  LOAD
    [_id] AS id,
    '$(pIndex)' AS index_name,
    Lower( Alt(content_title,'') & ' ' & Alt(content_text,'') & ' ' & Alt(content,'') & ' ' & Alt(chunk_text,'') ) AS search_text,
    content_title,
    If(Len(Trim(content_text))>0, content_text,
       If(Len(Trim(chunk_text))>0, chunk_text, content)) AS content_text
  RESIDENT RestConnectorMasterTable
  WHERE NOT IsNull([__FK_documents])
    AND NOT Exists(id, [_id]);

  DROP TABLE RestConnectorMasterTable;
END SUB

## End Loading Data

// -----------------------------------------------------------------------------
// Script purpose
// -----------------------------------------------------------------------------
// Dispatcher for on-demand searches. It checks the search trigger (vDoSearch),
// clears prior results, runs __RunSearch either on a single index or across all
// indices, and stores fresh results into Search_Results. It also tracks the
// timestamp of the last search and the number of rows retrieved. Finally, it
// resets the trigger to avoid loops and exits the script.
// -----------------------------------------------------------------------------

// =================== END LOADING DATA (dispatcher on-demand) ===================
TRACE [SearchTrigger:PRE] vDoSearch=$(vDoSearch) vIndexSel='$(vIndexSel)' vQuery='$(vQuery)' vQueryPersist='$(vQueryPersist)';

IF $(vDoSearch)=1 AND Len(Trim('$(vQueryPersist)'))>0 THEN

  // clear previous results and recreate empty structure
  IF TableNumber('Search_Results') >= 0 THEN 
  DROP TABLE Search_Results;
  END IF
  Search_Results:
  LOAD '' AS sr_id, '' AS sr_index, '' AS sr_title, '' AS sr_snippet, '' AS sr_type, '' AS sr_url
  AUTOGENERATE 0;

  // single index or multi-index
  IF Len(Trim('$(vIndexSel)'))>0 THEN
      TRACE [SearchTrigger] single index=$(vIndexSel) q='$(vQueryPersist)';
      CALL __RunSearch('$(vIndexSel)', '$(vQueryPersist)', $(vSearchWant));
  ELSE
      LET vIdxCount = SubStringCount('$(vIndexList)','|') + 1;
      FOR vPos = 1 TO $(vIdxCount)
        LET vThisIndex = SubField('$(vIndexList)','|', $(vPos));
        TRACE [SearchTrigger] running index=$(vThisIndex) q='$(vQueryPersist)';
        CALL __RunSearch('$(vThisIndex)', '$(vQueryPersist)', $(vSearchWant));
      NEXT vPos
  END IF

  LET vLastSearchTS = Timestamp(Now());
  LET vFreshRows    = Alt(NoOfRows('Search_Results'),0);
END IF

// reset trigger (avoids loops)
LET vDoSearch = 0;

// final debug
TRACE Last search timestamp = '$(vLastSearchTS)' | fresh rows = $(vFreshRows);

EXIT Script;
