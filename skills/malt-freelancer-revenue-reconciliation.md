---
name: Reconcile Malt freelancer revenue for a period
description: >-
  Pull a freelancer's Malt invoices, Malt's service charges and the payments actually received over
  a date range, then reconcile them into a net-revenue picture for bookkeeping or tax filing.
api: openapi/malt-exposed-apis-openapi.yml
operations:
  - findInvoices
  - findFeeInvoices
  - findPayments
  - getInvoice
  - getFeeInvoice
generated: '2026-08-17'
method: generated
source: >-
  openapi/malt-exposed-apis-openapi.yml (operationIds verified against the spec),
  conventions/malt-conventions.yml, errors/malt-problem-types.yml
---

# Reconcile Malt freelancer revenue for a period

This is the flow the Malt API is actually built for: a freelancer (or their accountant) closing a
month or a quarter.

## Before you start

- You need a **freelancer account token**, created at
  <https://www.malt.com/account/tokens> (My Account > API Keys) with the relevant permission
  scopes. The token is displayed **once** — capture it at creation.
- Base URL: `https://api.malt.com`
- Send the token as a **bare** value in the `Authorization` header. Malt's guidelines show
  `Authorization: your-api-token-here` with no `Bearer ` prefix. (The OpenAPI also declares an
  unused bearer scheme; follow the prose.)
- There are **no published rate limits and no rate-limit response headers**. Pace yourself
  conservatively and back off on any non-2xx.

## Steps

### 1. List the invoices you issued

`findInvoices` — `GET /freelancer/invoices?since={ISO date}&until={ISO date}`

- `since` is **required**. `until` is optional and defaults to open-ended.
- The response is an **unpaged array** of `InvoiceResource`. There is no cursor, page or limit
  parameter, so a wide window on a long history returns everything in one response. Choose windows
  deliberately — one month at a time is safer than one year.
- Each invoice carries `id`, `title`, `creationDate`, `expectedPaymentDate`, `externalId`,
  `amountWithoutTaxes`, `amountAllTaxesIncluded`, a `taxes[]` array, and embedded `customer` and
  `supplier` objects.
- **Watch out:** `InvoiceResource` has **no currency field**. If the freelancer bills in more than
  one currency you cannot tell from this response which is which — you will have to infer it from
  the matching payment in step 3.

### 2. List Malt's service charges against you

`findFeeInvoices` — `GET /freelancer/fee-invoices?since={ISO date}&until={ISO date}`

- Same `since`/`until` contract, same unpaged array shape.
- These are Malt's own commission invoices — the platform's cut, modelled as first-class invoices.
- **Watch out:** `FeeInvoiceResource` omits `creationDate` and `expectedPaymentDate` entirely. You
  filtered on a date range and the objects you got back carry no date. If you need to place a fee
  invoice in time, correlate it through step 3 (`findPayments` returns `LightInvoiceResource`
  entries whose `type` discriminates invoice from fee invoice) or through the enclosing period you
  queried.

### 3. List payments actually received

`findPayments` — `GET /freelancer/payments?since={ISO date}&until={ISO date}`

- Returns `PaymentResource` objects with `id`, `date`, `amount`, `currency` and `wireRef`, each
  carrying an `invoices[]` array of `LightInvoiceResource` (`id`, `externalId`, `type`).
- This is the **only** object in the API that states a currency. Use it as the authority.
- `LightInvoiceResource.id` is the same identifier `getInvoice` and `getFeeInvoice` accept, and its
  `type` tells you which of the two to call. That is your join key.
- **Watch out:** payments have an `id` but there is **no** `GET /freelancer/payments/{id}`. If you
  need a payment later, persist it now — you cannot re-fetch one by id.

### 4. Build the reconciliation

For each payment in step 3, walk its `invoices[]` entries and match each `id` back to the invoice
from step 1 or the fee invoice from step 2, dispatching on `type`. Where an id was not in your
windows (a payment settling an older invoice), fetch it directly:

- `getInvoice` — `GET /freelancer/invoices/{id}`
- `getFeeInvoice` — `GET /freelancer/fee-invoices/{id}`

Then compute: gross billed (sum of `amountAllTaxesIncluded` from step 1), platform charges (sum
from step 2), tax collected (sum the `taxes[]` `amount` values), and cash received (sum of payment
`amount`). Invoices in step 1 with no matching payment are outstanding — compare
`expectedPaymentDate` against today to age them.

## Error handling

Discriminate on HTTP status only — Malt returns **no machine-readable error codes**, and observed
401 responses have an **empty body** despite the documented `{timestamp, status, error, path}`
envelope.

| Status | Meaning | What to do |
|---|---|---|
| 400 | `since`/`until` not a parseable date, or `since` missing | Send ISO dates; `since` is required |
| 401 | Token missing, malformed, expired or revoked | Regenerate at `/account/tokens` — tokens show once |
| 403 | Token valid but wrong identity or missing scope | Confirm you are using a **freelancer** token, not a client team or organization token |
| 404 | Invoice id unknown **or** belongs to another identity | Malt does not distinguish the two; re-list to get valid ids |

There is no documented `429` and no `5xx` in the contract. Treat any unexpected status as
retry-with-backoff, and remember all five operations here are `GET` — safe to retry.

## Notes for agents

- Every operation in this skill is **read-only**. Nothing here mutates Malt state.
- Do not attempt to retrieve invoice PDFs in bulk. `getInvoicePdf` and `getFeeInvoicePdf` return
  `PDFInvoiceResource` — the document **base64-encoded inside a JSON body**, declared
  `application/json`, not a binary stream. Fetch a PDF only when a human asked for that specific
  document.
- Nothing in this API exposes profiles, missions, offers, availability or rates. If asked for
  those, say they are not in Malt's published contract rather than reaching for a scraper.
