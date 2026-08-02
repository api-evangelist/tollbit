---
name: tollbit-index-property-catalog
description: >-
  Enumerate a publisher property's content catalog and retrieve each page for
  indexing/analysis using TollBit crawl (indexing) tokens. Use when building a
  knowledge base or search index over licensed publisher content rather than
  displaying it to end users.
api: TollBit API
base_url: https://gateway.tollbit.com
auth: API key in the TollbitKey header; single-use crawl token for content
generated: '2026-07-21'
method: generated
source: openapi/tollbit-openapi.json + docs.tollbit.com/docs/content
operations:
- ContentService_getCatalog
- TokensService_createCrawlAccessToken
- ContentService_getContent
- ContentService_getRates
---

# Index a Property Catalog

Crawl a publisher's content for indexing intent (not end-user display).

## Steps

1. **List the catalog** — `ContentService_getCatalog`
   `GET /dev/v2/content/{propertyDomain}/catalog/list` with `TollbitKey`.
   Returns `pages[]` (`pageUrl`, `lastMod`, `propertyId`), ordered by last
   modified descending. Paginate with `pageToken` (and `pageSize`).

2. **(Optional) Check rates** — `ContentService_getRates`
   Indexing may still incur a rate; check `GET /dev/v2/rates/{contentUrl}` before
   fetching if cost control matters.

3. **Mint a crawl token** — `TokensService_createCrawlAccessToken`
   `POST /dev/v2/tokens/crawl` with `TollbitKey` and body `{ "url", "userAgent" }`.
   Returns `{ "token" }`. The publisher controls whether indexing access is
   permitted for a given page.

4. **Fetch for indexing** — `ContentService_getContent`
   `GET /dev/v2/content/{contentUrl}` with `TollbitToken: <crawl_token>` and
   `User-Agent: <AgentID>`. Same path as content retrieval; the token type
   (crawl vs content) sets the intent. Response has `content.body` + `metadata`
   (typically without a `rate` block for indexing).

## Rules
- Use **indexing** intent only when building indexes / running analysis; use the
  content-token flow (`tollbit-licensed-search-to-content`) when displaying or
  summarizing content for users or using it for training data.
- Tokens are single-use and expire in 5 minutes — mint per page.
- Errors follow RFC 9457 Problem Details; `403`/`400` may indicate the provider
  disallows indexing for that page.
