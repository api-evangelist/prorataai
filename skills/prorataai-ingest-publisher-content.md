---
name: prorataai-ingest-publisher-content
description: >-
  Push articles into the Gist Content Network as a publisher partner — single real-time
  submissions and bulk archive backfill — under the documented 10 req/min limit.
api: Gist Content API (Ingest)
base_url: https://api.gist.ai
operations:
  - POST /ingest/article
  - POST /ingest/multiple_articles
generated: '2026-08-26'
method: generated
source: https://platform.gist.ai/docs/gist-content-api
---

# Ingest publisher content into the Gist Content Network

> **This surface has no OpenAPI contract.** ProRata documents it in prose only, and the
> documentation publishes relative endpoint paths without naming a host. Everything below
> comes from https://platform.gist.ai/docs/gist-content-api. Confirm the base with ProRata
> before wiring it — do not assume it from this file.

Push-based ingestion is the recommended path: content lands as soon as it is published,
with no dependency on the Gist crawler's schedule.

## Authentication

`Authorization: Bearer <api-key>`. The key is issued at the **Publisher Group** level, so
every publication under the group submits with the same credential and identifies itself
per-article instead.

## Single article — `POST /ingest/article`

Required: `publication_external_id`, `publish_date` (ISO 8601), `title`, `content_type`
(`text` | `video` | `image`), `external_url`, and `content` (HTML) when `content_type` is
`text`. Optional: `article_thumbnail_url`, `canonical_url` (use it for syndicated pieces).

Success is `{"status":"success","message":"Ingest was successful."}`.

## Bulk archive — `POST /ingest/multiple_articles`

Takes a JSON **array**. Note the field names differ from the single-article call — this is
the most common integration mistake:

| Single article | Bulk article |
|---|---|
| `publication_external_id` | `publication_domain` |
| `publish_date` | `article_date` |
| `external_url` | `url` |

`title`, `content` and `canonical_url` keep their names.

## Limits and errors

- **10 requests per minute** by default. Higher limits are granted by support on request.
- `400` invalid or missing parameters · `401` unauthorized / invalid key ·
  `429` rate limit exceeded · `500` internal server error.
- Sanitize submitted HTML before sending. The docs explicitly ask for well-formed content
  free of scripts or embedded content (XSS), and state ProRata reviews ingested content
  for security risks.

## Irreversibility

No delete, retract or takedown endpoint is documented. Once an article is ingested there
is no published way to pull it back out via the API — plan corrections as re-submissions
and raise removals with ProRata directly.
