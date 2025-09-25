---
nav_order: 3
---

# Setup

## Prerequisites

- Qlik Sense Desktop or Qlik Cloud (SaaS)
- Network access to the backend base URL
- Permission to create Qlik REST connections

## Create REST Connections (per index)

Create these Qlik connections (names **must** match):

- `Knowledge Fabric:REST_stitch-docs-documents`
- `Knowledge Fabric:REST_stitch-conversations-documents`
- `Knowledge Fabric:REST_qlik-dev-documents`
- `Knowledge Fabric:REST_qlik-youtube-videos-documents`
- `Knowledge Fabric:REST_sfdc-knowledge-articles-documents`
- `Knowledge Fabric:REST_help-pages-documents`

> Each points to the base `/documents` endpoint. Authentication and headers as required by your environment. Do **not** store secrets in the repo.

## Import the App

- Open the `.qvf` in Qlik Sense.
- Ensure connections exist and test them.
- Reload the app.

## GitHub Pages

- Configure Pages to build from `/docs` using Jekyll.
- Place `docs/assets/logo.png` with your Qlik logo.
