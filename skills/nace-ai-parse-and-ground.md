---
name: nace-ai-parse-and-ground
description: Parse a document to markdown with Nace.AI NDI, then ground specific values back to their precise page coordinates for citation/verification.
api: NDI Service
base_url: https://ndi-api.nace.ai/api/v1
auth: X-API-Key header (ndi_sk_...)
operations:
- upload_file_api_v1_upload_post
- parse_async_api_v1_parse_async_post
- ground_async_api_v1_ground_async_post
- get_job_api_v1_jobs__job_id__get
generated: '2026-07-20'
method: generated
source: openapi/nace-ai-ndi-openapi.json
---

# Parse a document and ground values to coordinates (NDI)

Convert a document to high-fidelity markdown, then locate specific values and
get back the page + bounding-box coordinates where they appear — useful for
citations, human review, and audit trails.

## Steps

1. **Stage the file.** `POST /api/v1/upload` → `file_id`. Reference as
   `source_url: "ndi://file/<file_id>"`.
2. **Parse.** `POST /api/v1/parse_async` with
   `{ "file": { "source_url": "ndi://file/<file_id>" } }` → `202 { job_id }`.
   Poll `GET /api/v1/jobs/{job_id}`; on success `result.item` (a `ParsedFile`)
   gives `markdown_url`, `page_count`, `char_count`.
3. **Ground values.** `POST /api/v1/ground_async` with the same `file` plus
   `items` (max 30): each `{ "item_id", "query", "semantic": true|false }`.
   Set `semantic: true` when `query` describes what to find (e.g. "total
   liabilities") rather than literal text. Optionally pass the prior
   `markdown_url` under `derived` to speed grounding.
   - Tune with `options.confidence_threshold` (default 0.75) and
     `options.return_cropped_images`.
4. **Read matches.** Poll the ground job; `result.results[]` holds one
   `GroundItemResult` per `item_id` with `matches[]` — `visual_region`
   (page + x/y/w/h, normalized 0–1), `text_range`, `spreadsheet_range`, etc.

## Rules

- Auth: `X-API-Key: ndi_sk_...` on every call.
- Use `idempotency_key` (≤128 chars) on the POSTs to make retries safe.
- Errors: `{ "error": { "code", "message", "request_id" } }`; branch on
  `error.code`.
- Sync variants (`/parse`, `/ground`) accept `?timeout_seconds` (1–300); a
  `408`/`202` means poll `GET /jobs/{job_id}`.
