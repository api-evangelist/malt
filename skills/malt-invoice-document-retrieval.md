---
name: Retrieve a Malt invoice document as a PDF
description: >-
  Fetch a specific Malt invoice or service charge invoice and turn its base64-encoded payload into a
  PDF file on disk, handling Malt's JSON-wrapped document contract correctly.
api: openapi/malt-exposed-apis-openapi.yml
operations:
  - findInvoices
  - getInvoice
  - getInvoicePdf
  - findFeeInvoices
  - getFeeInvoice
  - getFeeInvoicePdf
generated: '2026-08-17'
method: generated
source: >-
  openapi/malt-exposed-apis-openapi.yml (operationIds verified against the spec),
  data-model/malt-data-model.yml, conventions/malt-conventions.yml
---

# Retrieve a Malt invoice document as a PDF

A short skill, but one worth packaging, because Malt's PDF contract is not what its operation names
suggest and a naive client writes a broken file.

## The thing to know first

`getInvoicePdf` and `getFeeInvoicePdf` do **not** return a binary PDF stream. Both declare their
200 response as `application/json` returning a `PDFInvoiceResource`:

```
{
  "id":  "<invoice id>",
  "pdf": "<base64-encoded PDF bytes>"
}
```

You must parse the JSON and base64-decode the `pdf` field before writing bytes to disk. Piping the
response body straight to a `.pdf` file produces a JSON text file with a PDF extension.

The corollary: **these responses are memory-expensive.** Base64 inflates the payload by roughly a
third and the whole document must be buffered — there is no streaming path. Fetch one document at a
time, for a document a human actually asked for. Do not loop a date range pulling every PDF.

## Before you start

- Freelancer account token from <https://www.malt.com/account/tokens>, sent bare in the
  `Authorization` header (no `Bearer ` prefix).
- Base URL `https://api.malt.com`.

## Steps

### 1. Find the invoice id

If you do not already hold the id, list the relevant window:

- `findInvoices` — `GET /freelancer/invoices?since={date}&until={date}` for client invoices
- `findFeeInvoices` — `GET /freelancer/fee-invoices?since={date}&until={date}` for Malt's own
  service charges

`since` is required on both. Both return unpaged arrays. Match on `title`, `creationDate`,
`externalId` or the embedded `customer.name` to identify the right document.

### 2. Confirm you have the right document

Optional but cheap, and it avoids decoding a large payload for the wrong invoice:

- `getInvoice` — `GET /freelancer/invoices/{id}`
- `getFeeInvoice` — `GET /freelancer/fee-invoices/{id}`

These return the structured `InvoiceResource` / `FeeInvoiceResource` — amounts, taxes, customer and
supplier — with no document payload. Show the human this summary and let them confirm.

### 3. Fetch and decode the document

- `getInvoicePdf` — `GET /freelancer/invoices/{id}/pdf`
- `getFeeInvoicePdf` — `GET /freelancer/fee-invoices/{id}/pdf`

Parse the JSON body, base64-decode `pdf`, write the bytes. Verify the result starts with the `%PDF-`
magic bytes before reporting success — that check catches both an undecoded payload and a truncated
response.

Name the file from data you already have (`id`, `creationDate`, `customer.name`) rather than from
anything in the PDF itself.

## Choosing the right endpoint

The two families are easy to confuse and mean opposite things financially:

| Endpoint family | What the document is | Who owes whom |
|---|---|---|
| `/freelancer/invoices/*` | The freelancer's invoice to a client | Client owes the freelancer |
| `/freelancer/fee-invoices/*` | Malt's service charge to the freelancer | Freelancer owes Malt |

If a payment record gave you the id, `LightInvoiceResource.type` on the payment's `invoices[]`
array tells you which family it belongs to — use that rather than guessing.

## Error handling

| Status | Meaning | What to do |
|---|---|---|
| 401 | Token missing, expired or revoked | Regenerate at `/account/tokens`; tokens display once |
| 403 | Wrong token type or insufficient scope | Use a **freelancer** token |
| 404 | Unknown id **or** the invoice belongs to another identity | Re-list to obtain valid ids — Malt does not distinguish the two cases |

Error bodies are unreliable: nothing in the contract declares one, and live 401s come back empty.
Decide from the status code.

## Notes for agents

- All operations here are `GET` and safe to retry.
- Never bulk-download PDFs. One document, on request.
- Do not attempt to parse the PDF's contents to answer a question the structured endpoints already
  answer — `getInvoice` gives you amounts, taxes and parties as data.
