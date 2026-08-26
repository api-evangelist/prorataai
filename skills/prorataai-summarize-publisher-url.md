---
name: prorataai-summarize-publisher-url
description: >-
  Create and stream a Gist summary of a specific article URL, respecting ProRata's exact
  domain-authorization rule — the URL's domain must be on your publisher group, and
  subdomains do not inherit.
api: Prorata API Service
base_url: https://api.gist.ai
operations:
  - GET /v1/publishers
  - POST /v1/summaries
  - GET /v1/summaries/{summaryId}
generated: '2026-08-26'
method: generated
source: openapi/prorataai-openapi.json
---

# Summarize a publisher URL

> The published OpenAPI declares no `operationId`s; steps are addressed by method + path.

## 1. Confirm what you are allowed to summarize

`GET /v1/publishers` returns the publisher group your key belongs to — group id, name,
description and the list of publisher IDs. `GET /v1/publishers/{id}` returns one
publisher's detail. Both are **Redis-cached with a 1-hour TTL**, so a publisher added
minutes ago may not appear yet; the response carries cache headers.

Neither publisher endpoint is rate limited in the published contract.

## 2. Create the summarization request

`POST /v1/summaries`

| Field | Required | Values |
|---|---|---|
| `url` | yes | the article to summarize (`format: uri`) |
| `length` | no | `short` \| `medium` (default) \| `long` |
| `medium` | no | `text` (default) \| `audio` |
| `style` | no | `paragraph` (default) \| `bullets` |

**The domain rule is the thing that breaks integrations.** The contract states the URL's
domain must *exactly* match one of the publisher domains on your organization's publisher
group, and that **subdomains are not automatically allowed**. A mismatch is a `403` with
`{"error":"Domain not authorized for this organization"}`. Check the domain before you
call rather than catching the 403.

A `404` here means the URL's content could not be retrieved.

Accepts either a public or a secret API key. The difference between the two tiers is not
documented anywhere in the hub — if it matters to your deployment, ask ProRata directly.

## 3. Stream the summary

`GET /v1/summaries/{summaryId}` using the `summaryId` from step 2. Server-sent events;
the docs direct browser clients to use `EventSource`. A malformed id is a `400`; an
unknown id is a `404`.

## Irreversibility

There is no delete, cancel or retract for a created summary. Nothing in the contract takes
this action back.
