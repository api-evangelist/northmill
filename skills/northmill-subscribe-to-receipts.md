---
name: northmill-subscribe-to-receipts
description: Receive Northmill Flo point-of-sale events in near real time by creating a webhook subscription, then fetch the full receipt through the REST API.
api: Northmill Flo API
base_url: https://api.moreflo.com
spec: openapi/northmill-flo-api-swagger.json
operations:
  - WebHookCategories_Get
  - WebHooks_Create
  - WebHooks_Get
  - WebHooks_Update
  - WebHooks_Delete
  - Receipts_Get
generated: '2026-08-26'
method: generated
source: >-
  Grounded in openapi/northmill-flo-api-swagger.json (fetched from
  https://api.moreflo.com/swagger/docs/v2 on 2026-08-26). Category names and the
  ReceiptCreated payload example are quoted verbatim from the WebHookCategory
  definition in that document.
---

# Subscribe to Northmill Flo events

Flo uses a **thin-payload / fat-fetch** pattern: the callback tells you what happened and gives you
an id, and you read the detail back over REST.

## 1. List the categories you may subscribe to

`WebHookCategories_Get` — `GET /v2/webhookcategories?page=1&pageSize=50`

Categories are server-defined. The Swagger document names two:

| Category | Fires when |
|---|---|
| `ArticleStockChanged` | "A WebHook will be called when there are any stock changes." |
| `ReceiptCreated` | "A WebHook will be called when a receipt is created." |

Do not hard-code that list — call the endpoint and read the live set. Each `WebHookCategory` carries
`Id`, `Name`, `Description`, `IsEnabled`, `CreatedTime`, `ModifiedTime`. You need the **`Id`**.

## 2. Create the subscription

`WebHooks_Create` — `POST /v2/webhooks`, body parameter `webHooks` (an array of `WebHookCreate`).

Required fields: `WebHookCategoryId` and `HttpMethod` (one of `Get`, `Post`, `Put`).
Also set `RemoteUrl` (your endpoint, max 2048 chars) and `IsEnabled: true`.
Set `UpdateOnExisting: true` so a retried create does not leave you with duplicate subscriptions.

**Read the operation summary before you call it.** It states: *"User remote ip address must be in
list of IPs resolved from host part in URL of webhook."* Your calling IP must resolve from the
hostname in `RemoteUrl`, or the create is rejected. The same constraint is on `WebHooks_Update`.

## 3. Handle the callback

For `ReceiptCreated` the body is documented in the contract as:

```json
{
  "receiptId": "6c30a0c6-44fd-473a-b7a7-ed164ba9083d",
  "receiptType": "CashierReceipt",
  "isRefund": false
}
```

Then call `Receipts_Get` — `GET /v2/receipts/{id}` — with `receiptId` to pull the whole receipt:
merchant, tax `ControlUnit` (`UnitId` + `ControlCode`), rounding, customers, articles, payments,
per-rate `Vat`, order references, discounts, recipients, text rows, client details and voucher.

`isRefund: true` marks a refund. In this model a refund is expressed *as a receipt* with negative
`UnitPrice` and `TotalPriceIncVat` on the article rows (quantity stays positive) — there is no refund
operation and no stated refund window. Do not assume one.

## 4. Know what is not guaranteed

The contract documents **no** signature header, shared secret, retry policy, backoff, ordering or
replay for webhook delivery. Practically:

- You cannot cryptographically verify a callback came from Northmill. Treat the payload as an
  untrusted hint and re-fetch over the authenticated REST API before acting on it.
- Assume at-most-once, out-of-order delivery. Reconcile with a periodic poll:
  `Receipts_Get` at `GET /v2/receipts?createdAfterUTC=...` for receipts, and
  `Articles_Get` with `QuantityUpdatedSince` for stock.

## 5. Clean up

`WebHooks_Delete` — `DELETE /v2/webhooks/{Id}`. **This is terminal** — there is no restore or
undelete in this API. To pause instead of remove, `WebHooks_Update` with `IsEnabled: false`.

## Related

- `asyncapi/northmill-flo-webhooks.yml` — the full webhook catalog and its documented gaps
- `conventions/northmill-conventions.yml` — reversibility, in-band error signalling
