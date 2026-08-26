---
name: northmill-sync-catalog-and-stock
description: Keep an external system's article catalogue and stock levels in sync with a Northmill Flo point-of-sale account, using incremental pulls and mirror-image stock adjustments.
api: Northmill Flo API
base_url: https://api.moreflo.com
sandbox_url: https://test.api.moreflo.com
spec: openapi/northmill-flo-api-swagger.json
operations:
  - Articles_Get
  - Articles_Create
  - Articles_Update
  - Articles_IncreaseStock
  - Articles_DecreaseStock
  - Stores_Get
generated: '2026-08-26'
method: generated
source: >-
  Grounded in openapi/northmill-flo-api-swagger.json (fetched from
  https://api.moreflo.com/swagger/docs/v2 on 2026-08-26). Every operationId, parameter
  and field name below is verified against that document. Northmill publishes no
  AGENTS.md or skill of its own.
---

# Sync a catalogue and stock with Northmill Flo

## Before you start

- Authentication is **HTTP Basic**. There is exactly one scheme (`securityDefinitions.basic`).
  Credentials come from the Flo back office at https://apps.moreflo.com/ — there is no self-service
  key creation.
- Develop against `https://test.api.moreflo.com`, which serves the identical 125-path contract.
  Environments are separated by **host**, not by key prefix.
- Decide your scope. Every call below exists in two forms:
  `/v2/articles` (merchant-wide) and `/v2/stores/{externalStoreId}/articles` (one store).
  `externalStoreId` is also accepted as a *query* parameter on the merchant-wide form. Pick one form
  and stay on it.

## 1. Discover the stores

`Stores_Get` — `GET /v2/stores?page=1&pageSize=50`

Read `list`, `Page`, `Total`, `PageSize` from the response. **`Total` is the number of PAGES, not the
number of stores** — this is the field most often misread in this API.

## 2. Pull the catalogue incrementally

`Articles_Get` — `GET /v2/articles`

Useful query parameters, all optional:

| Parameter | Use |
|---|---|
| `ArticleInfoUpdatedSince` | only articles whose descriptive data changed since a timestamp |
| `QuantityUpdatedSince` | only articles whose stock changed since a timestamp |
| `ArticleType`, `ArticleGroup` | narrow by type or group |
| `page`, `pageSize` | paginate |

Poll with the `*UpdatedSince` filters rather than re-reading the whole catalogue. For a single
article use `Articles_Get` at `GET /v2/articles/{ArticleNumber}`, or the v3 form
`GET /v3/articles/article?ArticleNumber=...`.

## 3. Push articles safely

`Articles_Create` — `POST /v2/articles`, body parameter `articles`.

Set **`UpdateOnExisting: true`** on each `ArticleCreate`. The contract states it "indicates whether
to update an existing article if one with the same article number already exists". `ArticleNumber` is
your stable key, so a retried create is an update rather than a duplicate. **There is no
`Idempotency-Key` header in this API** — this flag plus a stable key is the whole mechanism, and
Northmill documents no replay window or stored response.

Use `Articles_Update` (`PUT /v2/articles`) when you know the article exists.

## 4. Adjust stock

- `Articles_IncreaseStock` — `PUT /v2/articles/increasestock`, body `articleQuantityInfos`
- `Articles_DecreaseStock` — `PUT /v2/articles/decreasestock`, body `articleQuantityInfos`

These are **deltas, not absolute levels**, and they are exact mirrors of each other: an increase of
*n* undoes a decrease of *n*. That is the only reversal available here — there is no
"set stock to X" operation and no transaction log to roll back. Because they are deltas they are
**not** naturally idempotent: a retried decrease decrements twice. Track your own request outcomes
before retrying.

## 5. Check every response, even a 200

Every `*Result` envelope carries:

- `Success` — boolean, "indicates whether the API operation was successful"
- `ExtraInformation` — array of strings, "can include error messages, warnings, or informational notes"

**HTTP 200 does not mean the operation worked.** The Swagger document declares exactly one response
per operation — `200 OK` — and no 4xx or 5xx anywhere, so status codes carry almost no information.
Read `Success` and log `ExtraInformation` on every call.

## 6. Errors and limits

- No rate limits are published for the Flo API and no `X-RateLimit-*`, `RateLimit-*`, `Retry-After`
  or `429` appears in the contract. Be conservative and back off on any non-200.
- Unauthenticated requests return `401` with a Basic challenge — even `GET /health`.
- There is no request-id or correlation-id header, so you have no provider-side handle to quote to
  support. Keep your own request log.

## Related

- `conventions/northmill-conventions.yml` — pagination, upsert, incremental sync, reversibility
- `data-model/northmill-data-model.yml` — how Article relates to ArticleSet, Barcode and Group
- `errors/northmill-problem-types.yml` — the measured absence of an error catalog
