---
name: Audit Ledge source freshness and ingestion health
description: >-
  Check every data source connected to a Ledge organization for stale datasets
  and ingestion failures before anyone trusts a reconciliation or closes a
  period. Answers "is my data actually current, and did anything fail to land?"
api: openapi/ledge-api-openapi.yml
operations:
  - getSources
generated: '2026-07-19'
method: generated
source: >-
  Generated from openapi/ledge-api-openapi.yml. Every operationId is verified
  verbatim in that description; cross-cutting rules cite
  conventions/ledge-conventions.yml, errors/ledge-problem-types.yml and
  authentication/ledge-authentication.yml.
---

# Audit Ledge source freshness and ingestion health

Ledge reconciles data pulled from banks, payment service providers, ERPs,
databases and file feeds. Every downstream answer is only as good as the last
successful fetch, so the first move in any Ledge workflow is to confirm the
sources are current and clean.

## Before you start

- **Get a token.** Ledge uses OAuth 2.0 `client_credentials`. POST to
  `https://goledge.us.auth0.com/oauth/token` with `grant_type=client_credentials`,
  your `client_id`, `client_secret` and `audience`, then send the returned
  `access_token` as `Authorization: Bearer <token>`. Tokens are short-lived
  (`expires_in` was 10800 seconds in the published example) — refresh on `401`
  rather than caching indefinitely. See `authentication/ledge-authentication.yml`.
- **Find your `orgId`.** It is the Organization UUID shown on the Ledge
  Developers page at `https://app.goledge.io/developers`. It is a required path
  segment on every operation.
- **All timestamps are epoch milliseconds**, not ISO 8601. Convert before you
  compare or display.

## Steps

### 1. List the sources

Call **`getSources`** — `GET /v1/api/{orgId}/sources` on `https://api.goledge.io`.

It returns an array of `Source` objects, each with a `datasets[]` array. There
are no query parameters and no pagination on this operation.

### 2. Triage each dataset

For every dataset in every source, read these fields:

- **`status`** — one of `pending`, `active`, `deactivated`. Only `active`
  datasets are feeding reconciliation. A dataset sitting in `pending` or flipped
  to `deactivated` is a silent gap; surface it explicitly.
- **`lastReceived`** — when new data last landed. Compare against now: a feed
  that should arrive daily but has not received data in several days is stale,
  regardless of whether anything errored.
- **`latestTimestamp`** — the "as-of" point, the timestamp of the newest
  transaction in the dataset. This, not `lastReceived`, is the date through
  which you can honestly reconcile. A fetch can succeed while delivering only
  old records.
- **`failures[]`** — each entry has a `type` and a `count`. Any non-empty
  `failures` array means records were dropped or misparsed. Report the type and
  count; do not average them away.
- **`lastFile`** — `name` and `fetchTime` of the last file received, useful for
  file- and SFTP-based feeds where you need to name the artifact to the customer.
- **`revision`** — increments when the dataset changes shape. A recent bump can
  explain new parse failures.

### 3. Report

Produce a per-source summary that names, for each dataset: status, as-of
timestamp (`latestTimestamp`), time since `lastReceived`, and any failure types
with counts. Lead with the problems — stale, deactivated, or failing datasets —
because those are what block a close.

Do not proceed to reconciliation analysis on a dataset whose `latestTimestamp`
falls short of the period you are closing. Say so instead.

## Error handling

Errors come back as `{"error": {"code", "status", "request", "message"}}` — a
proprietary envelope, not RFC 9457 `problem+json`.

- **401** — token expired or wrong credentials. Request a fresh token and retry once.
- **403** — the caller's role denies this resource. Ledge uses role-based
  fine-grained permissions (`Administrator`, `Full member`, `View-only`, plus
  custom roles); a custom role can deny specific resources. This is not
  retryable — escalate.
- **404** — check the `orgId`.
- **500 / 503** — Ledge-side. Retry with backoff and check `https://status.ledge.co`.

Every error carries a `request` UUID. **Quote it** when you escalate to
`support@ledge.co` — it is the correlation id Ledge support will ask for.

## Notes

- This operation is read-only and safe to run unattended.
- Ledge documents no rate limits and no idempotency keys; there is nothing to
  set, and nothing to rely on. Be conservative with polling frequency.
