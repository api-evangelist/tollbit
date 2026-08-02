---
name: tollbit-licensed-search-to-content
description: >-
  Discover licensable publisher content with TollBit Licensed Search, check the
  rate/license options for a result, mint a one-time content access token, and
  retrieve the licensed content as markdown. Use when an agent needs authorized,
  paid access to first-party publisher content instead of scraping.
api: TollBit API
base_url: https://gateway.tollbit.com
auth: API key in the TollbitKey header (management calls); single-use TollbitToken for content
generated: '2026-07-21'
method: generated
source: openapi/tollbit-openapi.json + docs.tollbit.com/docs/search-to-content
operations:
- SearchService_search
- ContentService_getRates
- TokensService_createContentAccessToken
- ContentService_getContent
- ReportingService_reportContentUsage
---

# Licensed Search to Content

End-to-end flow for finding, licensing, and fetching publisher content through TollBit.

## Prerequisites
- A TollBit organization API key (Developer Dashboard > Access at https://hack.tollbit.com).
- A registered AgentID used as your `User-Agent`.

## Steps

1. **Search for content** — `SearchService_search`
   `GET /dev/v2/search?q={query}&size={<=20}` with header `TollbitKey: <api_key>`.
   Optionally pass `properties` (comma-separated domains to boost) and
   `readyToLicense=true` to restrict to licensable results. Paginate with
   `next-token`. Each `items[]` result has `url`, `publisher`, and
   `availability.readyToLicense`.

2. **Check rates** — `ContentService_getRates`
   `GET /dev/v2/rates/{contentUrl}` (URL-encode the content URL) with `TollbitKey`.
   Returns an array of options, each with a `license` (`licenseType`,
   `permissions`) and a `price` (`priceMicros`, `currency`). Remember
   1 USD = 1,000,000 micros. For many URLs use `ContentService_batchGetRates`
   (`POST /dev/v2/rates/batch`).

3. **Mint a content token** — `TokensService_createContentAccessToken`
   `POST /dev/v2/tokens/content` with `TollbitKey` and body:
   `{ "url", "userAgent", "maxPriceMicros", "currency": "USD", "licenseType" }`.
   Set `maxPriceMicros` at or above the chosen option's `priceMicros`, or the
   request is rejected (400). For `CUSTOM_LICENSE`, include `licenseCuid`.
   Returns `{ "token": "<JWT>" }`. Tokens are single-use and expire in 5 minutes.

4. **Fetch the content** — `ContentService_getContent`
   `GET /dev/v2/content/{contentUrl}` with headers `TollbitToken: <token>`,
   `User-Agent: <your AgentID>`, and optionally
   `Tollbit-Accept-Content: text/markdown` (default) or `text/html`.
   Returns `content.body` (markdown), `metadata`, and the applied `rate`.

5. **Self-report usage (recommended)** — `ReportingService_reportContentUsage`
   `POST /dev/v2/transactions/selfReport` with `TollbitKey` and a body carrying
   a unique `idempotencyId` and a `usage[]` array (`url`, `timesUsed`,
   `licenseType`). The `idempotencyId` makes retries safe (no double billing).

## Error handling
- Responses use RFC 9457 Problem Details (`type/title/status/detail/instance`).
- `400` invalid license / price too low; `401` bad/missing `TollbitKey` or token;
  `403` not permitted; `404` unknown content; `500` retry with backoff.

## Notes
- Never send your API key to a `tollbit.<publisher>.com` subdomain — mint a token instead.
- Use crawl/indexing intent (`TokensService_createCrawlAccessToken`) when building
  indexes rather than displaying content to users.
