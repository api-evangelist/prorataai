---
name: prorataai-ask-and-attribute
description: >-
  Ask the Gist Answers engine a question on behalf of a publisher, stream the answer, and
  then read back the citations and the fractional attribution credit split for that turn.
  This is ProRata's marquee flow — the answer and the credit for the answer are two
  separate calls.
api: Prorata API Service
base_url: https://api.gist.ai
operations:
  - POST /v1/chat
  - GET /v1/chat/response/{threadId}/{turnId}
  - GET /v1/chat/citations/{threadId}/{turnId}
  - GET /v1/chat/attributions/{threadId}/{turnId}
generated: '2026-08-26'
method: generated
source: openapi/prorataai-openapi.json
---

# Ask Gist Answers and read the attribution

> The published OpenAPI declares **no `operationId` on any operation**, so every step below
> is addressed by method + path exactly as the contract publishes it. Stable operationIds
> are proposed non-destructively in `overlays/prorataai-openapi-overlay.yaml`; do not
> assume the provider honors them.

## Before you start

- Send `Authorization: Bearer <api-key>` on every call. Keys are issued at the
  **Publisher Group** level during onboarding — there is no self-serve signup.
- `POST /v1/chat` additionally requires the header **`X-User-ID`** (a unique identifier for
  the end user). It is `required: true` in the contract; omitting it is a 400.
- There is **no idempotency key**. Every `POST /v1/chat` creates a new turn, so a retry
  after a timeout creates a duplicate. Decide before you call whether a duplicate is
  acceptable.

## 1. Create the chat

`POST /v1/chat`

Body: `user_prompt` (required). Optional `thread_id` to continue an existing conversation —
send an empty string or omit it to start a new one. `inclusion_list` narrows the corpus.
`temperature` is accepted but the contract states it is **currently not supported and will
be ignored**; do not build behaviour on it.

The 200 returns `thread_id`, `turn_id`, `thread_title` and a `citations` object. Keep
`thread_id` and `turn_id` — every subsequent step is addressed by that pair.

## 2. Stream the answer

`GET /v1/chat/response/{threadId}/{turnId}`

Server-sent events. In a browser, consume with `EventSource`. `POST /v1/chat/completions`
is an alternative streaming entry point, but its own summary marks it
**(Experimental)** — prefer the pair above for anything durable.

## 3. Read the citations

`GET /v1/chat/citations/{threadId}/{turnId}`

Returns the sources behind the answer. A 404 here means the thread, the turn, or the
citations are not (yet) available — not that the answer was uncited.

## 4. Read the attribution credit split

`GET /v1/chat/attributions/{threadId}/{turnId}`

Returns four parallel maps of identifier → percentage:

| Field | Keyed on |
|---|---|
| `credit_dist` | source URL |
| `domain_credit_dist` | domain (e.g. `example.com`) |
| `document_credit_dist` | document ID (SHA-256 hash) |
| `publisher_credit_dist` | publisher UUID |

This is the commercial payload — it is what the Gist Content Network revenue share is
computed against. Note that it is a **vendor shape**: it carries no estimator id/version,
no `model_residual` and no granularity, so it is not the Content Telemetry fractional
attribution document ProRata published separately (see
`conformance/prorataai-conformance.yml`).

## Errors and limits

- `401` — missing or invalid key, or a missing `Authorization` header. Observed live as
  `{"error":"Unauthorized","message":"Missing or invalid Authorization header","statusCode":401}`.
- `429` — every step in this flow is rate limited per organization. Read
  `X-RateLimit-Remaining` and back off until the unix timestamp in `X-RateLimit-Reset`.
  There is **no `Retry-After`**.
- `400` — most commonly a missing `user_prompt` or a malformed `threadId`/`turnId`.
- `500` — retry with backoff.
- Error bodies are **not** RFC 9457 and come in three different shapes. See
  `errors/prorataai-problem-types.yml` before writing a parser.

## Cleaning up

`DELETE /v1/threads/{threadId}` **permanently** deletes the thread and all its turns.
There is no restore, no soft-delete and no stated retention window. Treat it as
irreversible.
