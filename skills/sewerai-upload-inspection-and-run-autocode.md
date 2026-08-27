---
generated: '2026-08-27'
method: generated
name: Upload an inspection video and run AutoCode
description: Take a CCTV inspection video from a local file to AI-coded NASSCO observations in SewerAI PIONEER — create the asset, project and inspection, register and upload the video via presigned S3 POST, wait for encoding, then start an AutoCode run.
api: openapi/sewerai-swagger.json
operations: [v1_organizations_create, v1_assets_create, v1_projects_create, v1_inspections_create, v1_videos_create, v1_videos_read, v1_inspections_AutoCode, v1_inspections_observations_list]
source: >-
  Grounded in openapi/sewerai-swagger.json, harvested verbatim 2026-08-27 from
  https://api.sewerai.com/swagger.json. Every operationId verified verbatim in that contract. The
  step sequence follows the Quick Start and the worked Python at https://docs.sewerai.com/. Auth per
  authentication/sewerai-authentication.yml; errors per errors/sewerai-problem-types.yml; retry,
  pagination and reversibility rules per conventions/sewerai-conventions.yml and
  rate-limits/sewerai-rate-limits.yml.
---

# Upload an inspection video and run AutoCode

This is SewerAI's marquee flow: raw CCTV footage in, NASSCO-coded observations out. It is a stateful, ordered sequence — each object must exist before the next one can reference it.

## Auth

- Base URL: `https://api.sewerai.com/v1`
- Send `Authorization: X-SAI {API_KEY}`, or `Authorization: Bearer {JWT}` using a token from `token_create` (`POST /token/`).
- **Keys are not self-serve.** SewerAI issues them on request to `info@sewerai.com`, and you must already be a registered PIONEER user. See `authentication/sewerai-authentication.yml`.

## Steps

1. **Create the organization** (optional) — `v1_organizations_create` (`POST /v1/organizations/`). The organization is the *owner* of assets. Skip it if the assets already belong to one. Keep the returned `sid`.
2. **Create the asset** — `v1_assets_create` (`POST /v1/assets/`). Required: `name`, `city`. Set `kind` to one of `mainline`, `lateral`, `maintenance-hole`, `other` — this is the NASSCO class discriminator and it decides which inspection shape applies later (PACP / LACP / MACP). Set `owner` to the organization `sid`. Keep the returned `sid`.
3. **Create the project** — `v1_projects_create` (`POST /v1/projects/`). Keep the returned `sid`.
4. **Create the inspection** — `v1_inspections_create` (`POST /v1/inspections/`). Required: `asset`. Also set `projects` (an array), `inspection_datetime` (ISO 8601), `city`, and — importantly — `key`, your own unique identifier for this inspection. If you omit `key` SewerAI generates a UUID and you lose your only handle for reconciling a retry. Keep the returned `sid`.
5. **Register the video** — `v1_videos_create` (`POST /v1/videos/`) with `{"inspection": <inspection_sid>, "path": "<local path>"}`. The response carries `sid`, `video_name` and `presigned_upload_data`.
6. **Upload the bytes to S3, not to SewerAI** — `POST` a `multipart/form-data` body to `presigned_upload_data.url`, carrying `presigned_upload_data.fields.key`, `.AWSAccessKeyId`, `.policy` and `.signature` alongside the `file` part. This request goes to AWS, carries no SewerAI Authorization header, and its failures are S3's, not SewerAI's.
7. **Wait for the upload to settle** — poll `v1_videos_read` (`GET /v1/videos/{sid}/`) until `stage` is `uploaded`. The docs advise waiting at least 5 minutes before the first poll. Do not proceed while `stage` is anything else.
8. **Run AutoCode** — `v1_inspections_AutoCode` (`POST /v1/inspections/AutoCode/`) with `{"inspections": [<sid>, ...], "projects": [<sid>, ...]}`. Either array may be null. A 200 means the run was accepted, not finished.
9. **Wait, then read the results** — the docs advise 15–20 minutes. Poll `v1_videos_read` and watch `autocode_complete` / `autocode_complete_date` on the inspection, then read the coded output with `v1_inspections_observations_list` (`GET /v1/inspections/{inspection_sid}/observations/`).

## Rules an agent must follow

- **AutoCode is billable, non-idempotent and irreversible.** SewerAI prices AutoCode on usage (`plans/sewerai-plans-pricing.yml`), the API publishes no idempotency key, and there is no cancel, abort or refund operation anywhere in the 476-operation contract. A blind retry of step 8 after a timeout is a second billable run over the same footage. If step 8 does not return cleanly, re-read the inspection state with `v1_inspections_read` and check `autocode_complete` before firing again.
- **Nothing here is idempotent.** No `Idempotency-Key` header exists. Steps 1–5 are all plain `POST`s. Before retrying any of them, list first (`v1_assets_list`, `v1_inspections_list`) and match on your own `key`, or you will create duplicates. See `conventions/sewerai-conventions.yml`.
- **There is no pagination.** List endpoints return an unbounded JSON array with no `limit`, `offset`, `page` or `cursor` parameter, despite response schemas named `Paginated*List`. Filter with `created_after` / `created_before` / `updated_after` / `updated_before` / `kind` rather than trying to page.
- **There is no rate-limit signal.** No `RateLimit-*`, `X-RateLimit-*` or `Retry-After` header is returned, and no limit is published. Pace polling conservatively — minutes, not seconds — because you have no way to know when you are close to a ceiling. See `rate-limits/sewerai-rate-limits.yml`.
- **Errors come in three shapes, none of them RFC 9457.** `{"detail": "..."}`, a field-keyed validation map like `{"username": ["This field is required."]}`, and a gateway `{"message": "Not Found (404)"}`. Parse all three. Note that SewerAI returns **403, not 401**, for missing credentials, and sends no `WWW-Authenticate` header. See `errors/sewerai-problem-types.yml`.
- **Capture `x-amzn-requestid`** from every response. It is the only correlation identifier the API returns; nothing in the error body identifies the request.
- **Related objects come back as URLs, not ids.** `owner`, `asset`, `account`, `created_by` and friends are absolute URIs. To recover a `sid`, take the trailing path segment — the worked example in the SewerAI docs does exactly this (`v['inspection'].split('/')[-2]`).
- **Deletes are soft.** `v1_inspections_delete`, `v1_assets_delete` and `v1_videos_delete` set `deleted` / `deleted_by` rather than erasing. A restore operation exists (`videos_Restore`, `POST /videos/{sid}/Restore/`) but only for videos, only on the undocumented legacy namespace, and with no stated retention window. Do not assume a delete can be taken back.
