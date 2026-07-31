---
name: nace-ai-extract-document-data
description: Extract schema-shaped structured JSON from a document with Nace.AI NDI — upload once, then run an extraction against a caller-defined JSON Schema.
api: NDI Service
base_url: https://ndi-api.nace.ai/api/v1
auth: X-API-Key header (ndi_sk_...)
operations:
- upload_file_api_v1_upload_post
- extract_async_api_v1_extract_async_post
- get_job_api_v1_jobs__job_id__get
generated: '2026-07-20'
method: generated
source: openapi/nace-ai-ndi-openapi.json
---

# Extract structured data from a document (NDI)

Turn an unstructured document (PDF, DOCX, XLSX, image, email, …) into structured
JSON that matches a schema you define.

## Steps

1. **Stage the file.** `POST /api/v1/upload` (multipart, field `file`).
   Response `UploadResponse` gives a `file_id`. Reference it later as
   `source_url: "ndi://file/<file_id>"`. The handle lives ~6 days
   (`expires_at`).
2. **Start the extraction.** `POST /api/v1/extract_async` with body
   `{ "file": { "source_url": "ndi://file/<file_id>" }, "schema": { ... } }`.
   - `schema` is a JSON Schema whose root is an object with properties. Nesting
     is bounded; `$ref`/`anyOf`/`oneOf`/`allOf`/`not` are not supported. Field
     `description`s directly improve extraction accuracy.
   - Pass an `idempotency_key` (≤128 chars) to make retries safe — a repeat with
     the same key returns the original job instead of starting a new one.
   - Returns `202 { job_id }`.
3. **Poll for the result.** `GET /api/v1/jobs/{job_id}` until
   `status` is `succeeded` or `failed`. On success, `result` (tagged
   `result_type: extract`) carries `data` (values matching your schema; absent
   fields are null / empty arrays) and `json_url`.

## Rules

- Auth: send `X-API-Key: ndi_sk_...` on every call.
- Errors are `{ "error": { "code", "message", "request_id" } }`; branch on
  `error.code`, and quote `request_id` (== `X-Request-Id`) to support.
- Prefer async for large documents; the sync `/extract` may return `408`/`202`
  meaning "poll `GET /jobs/{job_id}`".
- Cancel a stuck job with `POST /api/v1/jobs/{job_id}/cancel`.
