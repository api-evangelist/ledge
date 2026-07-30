---
name: Find and investigate unreconciled Ledge transactions
description: >-
  Pull the open items out of a Ledge dataset for a period — the transactions
  with no match or only a partial match — and assemble the exception list a
  finance team works through to close the books.
api: openapi/ledge-api-openapi.yml
operations:
  - getSources
  - queryTransactions
generated: '2026-07-19'
method: generated
source: >-
  Generated from openapi/ledge-api-openapi.yml. Every operationId is verified
  verbatim in that description; the filter grammar is transcribed from
  https://docs.ledge.co/api-reference/transactions/querying and cross-cutting
  rules cite conventions/ledge-conventions.yml and errors/ledge-problem-types.yml.
---

# Find and investigate unreconciled Ledge transactions

The month-end exception list is the core Ledge workflow: for a given account or
feed and a given period, which transactions have not fully reconciled, and why.

## Before you start

- **Authenticate.** OAuth 2.0 `client_credentials` against
  `https://goledge.us.auth0.com/oauth/token`; send the `access_token` as
  `Authorization: Bearer <token>`. See `authentication/ledge-authentication.yml`.
- **Get your `orgId`** from `https://app.goledge.io/developers`.
- **All times are epoch milliseconds.** Convert your period boundaries before
  building the request.

## Steps

### 1. Resolve the dataset

Call **`getSources`** — `GET /v1/api/{orgId}/sources` — and pick the `id` of the
dataset you want from the `datasets[]` array of the relevant source. That value
is the `datasetId` for the next call.

Check the dataset's `latestTimestamp` first: if it stops short of your period
end, the data is incomplete and the exception list will be misleading. See
`ledge-audit-source-freshness.md`.

### 2. Query the period

Call **`queryTransactions`** — `POST /v1/api/{orgId}/transactions`.

Despite the POST verb this operation only reads; the body carries the filter
grammar because it is too structured for a query string.

```json
{
  "datasetId": "8872aec1-6a21-423b-a7ad-e813f4419f75",
  "from": 1704067200000,
  "to": 1706745600000,
  "limit": 100,
  "offset": 0
}
```

- `from` / `to` bound the period, in epoch milliseconds.
- `limit` sets the page size; `offset` walks the pages.
- `columns[]` optionally narrows the response to the fields you need.

### 3. Page through the results

Increment `offset` by `limit` until a page comes back short or empty. There is
no `has_more` flag, no cursor and no total count — the response is a bare JSON
array, so a page smaller than `limit` is your only end-of-results signal.

**The maximum `offset` is 500.** This is a hard ceiling, not a suggestion: you
cannot page past it. If the result set is larger, you must partition the query
by narrowing `from`/`to` into smaller windows, or by adding filters, and merge
the results yourself. Detect this case and split rather than silently returning
a truncated exception list.

If a query returns **504**, it exceeded the server timeout — reduce `limit` and
narrow the window. That is the only backpressure signal Ledge publishes.

### 4. Filter to the exceptions

Filters go in `search[]` as `SearchQuery` objects, ANDed together:

```json
{
  "search": [
    { "path": "Description", "search": { "type": "string", "value": "INV-340174" } }
  ]
}
```

`path` is the transaction field name; `search` is a typed filter — `string`,
`boolean`, `number`, `money` or `date`. Numbers accept a single value, a list, a
`NumberExpression` (`operator` one of `ne`, `ge`, `gt`, `le`, `lt`) or a
`NumberRange` (`start`, `end`, `includeStart`, `includeEnd`). Money filters
additionally accept a `currencies[]` list of ISO 4217 codes. Date filters use a
`NumberRange` in milliseconds.

### 5. Classify by reconciliation status

Every returned transaction carries `reconciliationStatus`, one of:

- **`none`** — no match at all. The core exception: unapplied cash, a missing
  bank record, a payment that never landed.
- **`partial`** — matched for part of its value. Usually fees, FX drift, short
  pays or split settlements. Look at the match set to find the residual.
- **`full`** — reconciled. Exclude from the exception list.
- **`informative`** — recorded for context, not expected to reconcile.
- **`out of scope`** — explicitly excluded from reconciliation.

Build the exception list from `none` and `partial` only. Report `informative`
and `out of scope` counts separately so the numbers reconcile back to the total —
never fold them into the exceptions.

### 6. Follow the matches

`incomingMatches[]` lists matches where this transaction is the **origin**;
`outgoingMatches[]` lists matches where it is the **target**. Use the direction
to describe the relationship correctly (a payment matched *to* an invoice reads
differently from an invoice matched *from* a payment).

Note that Ledge documents the `Match[]` type but does not publish its field
schema, so treat match objects as opaque: report their presence, count and
direction, and do not assume field names inside them.

## Error handling

Errors use the `{"error": {"code", "status", "request", "message"}}` envelope —
not RFC 9457.

- **400** — malformed body. The usual cause is a filter whose `type` does not
  match the value shape (e.g. a `number` filter given a string).
- **401** — refresh the token and retry once.
- **403** — role denies the resource; not retryable.
- **404** — check `orgId` and `datasetId`.
- **504** — reduce `limit`, narrow `from`/`to`.
- **500 / 503** — retry with backoff; check `https://status.ledge.co`.

Quote the `request` UUID from the envelope in any support escalation to
`support@ledge.co`.

## Notes

- Both operations are read-only; the published Ledge API exposes no writes, so
  there is nothing here that can alter the ledger.
- Ledge supports no idempotency keys and publishes no rate limits — retries are
  safe here only because every operation is a read.
