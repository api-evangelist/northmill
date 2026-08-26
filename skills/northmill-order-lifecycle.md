---
name: northmill-order-lifecycle
description: Create, look up, update and cancel orders in Northmill Flo using the v3 order API and a client-owned external reference.
api: Northmill Flo API
base_url: https://api.moreflo.com
spec: openapi/northmill-flo-api-swagger.json
operations:
  - OrdersV3_Create
  - OrdersV3_List
  - OrdersV3_Get
  - OrdersV3_GetByExternalReference
  - OrdersV3_Update
  - Customers_Find
  - Customers_Get
  - Customers_GetNextAvailableCustomerNumber
generated: '2026-08-26'
method: generated
source: >-
  Grounded in openapi/northmill-flo-api-swagger.json (fetched from
  https://api.moreflo.com/swagger/docs/v2 on 2026-08-26). State and PaymentStatus enum
  members are quoted verbatim from the v3 Order definition.
---

# Run an order through Northmill Flo

Use the **v3** order surface. v2 orders (`Orders_Create`, `Orders_Update`) are still published and
are **not** marked deprecated, but v3 is the current shape and it is the only order form that
supports listing.

## 1. Own the key

`ExternalReference` is **required on create** — the contract calls it "the external reference or
order number from another system". It is your key, and it is what makes writes safe:

- `UpdateOnExisting: true` on `OrderCreate` means a retried create with the same reference updates
  the existing order instead of creating a second one.
- `OrdersV3_GetByExternalReference` — `GET /v3/orders/byExternalReference/{externalReference}` —
  lets you look the order up again without storing Northmill's `Id`.

There is no `Idempotency-Key` header. Choose a stable `ExternalReference` and this is your
idempotency.

## 2. Attach the customer

- `Customers_Find` — `POST /v2/customers/find`, body `query`
- `Customers_Get` — `GET /v2/customers/get/{customerNumber}`
- `Customers_GetNextAvailableCustomerNumber` — `GET /v2/customers/nextcustomernumber` when you need
  to allocate a new number before `Customers_Create`.

## 3. Create the order

`OrdersV3_Create` — `POST /v3/orders`, body parameter `orders` (an array — this API is bulk-first;
every create and update takes a list).

The v3 `Order` carries `Customer`, `BillingAddress`, `ShippingAddress`, `Rows` (`OrderRow`),
`AttachedFiles`, plus `Discount`, `TotalExclVat`, `TotalInclVat`, `TotalVat`, `PaymentOption`,
`PaymentReference`, `PaymentOrigin`, `DeliveryOption` and `BookableResourceId`.
`AdjustStockIfOpenState` controls whether creating the order moves stock.

## 4. Track state

`State` (v3 enum, verbatim): `Created`, `Saved`, `Started`, `WaitingCustomerResp`,
`ReadyForHandOut`, `Waiting`, `Cancelled`, `Delivered`.

`PaymentStatus` (verbatim): `NotSet`, `NotPaid`, `FullyPaid`, `Refunded`.

## 5. Poll for changes

`OrdersV3_List` — `GET /v3/orders?modifiedSince=...&page=1&pageSize=50`.
**`modifiedSince` is required** on this operation — you cannot list every order, only the delta.

## 6. Reverse it

There is no cancel *operation*. To cancel, `OrdersV3_Update` (`PUT /v3/orders`) with `State` set to
`Cancelled`.

- **No cancellation window is stated** for orders. Do not assume one.
- Booking orders are different and better documented: `BookingOrder` carries
  `CancellationLatestTime`, `ClientCancellationPossible` and `CancellationMinimumTimeInAdvance`
  (seconds, on the article's booking settings), and Flo stamps `CancellationSource`
  (e.g. `"Staff"`, `"ExternalApi"`) when the state moves to `Cancelled`. Read those fields off the
  object rather than assuming a policy.
- A payment reversal is not an order operation at all — it is a refund receipt. See
  `skills/northmill-subscribe-to-receipts.md`.

## 7. Check `Success`, not the status code

Every response is declared `200 OK`; no 4xx or 5xx is documented on any operation. The real outcome
is in `Success` and `ExtraInformation` on the result envelope. Read them.

## Related

- `conventions/northmill-conventions.yml` — the reversibility profile in full
- `data-model/northmill-data-model.yml` — Order to OrderRow, Customer, Address, AttachedFile
