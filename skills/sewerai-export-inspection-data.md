---
generated: '2026-08-27'
method: generated
name: Initiate and fetch an inspection export
description: Run the asynchronous export flow in SewerAI PIONEER — start an export job, poll it to completion, retrieve the output, and clean up — the pattern SewerAI's own docs single out as a worked example.
api: openapi/sewerai-swagger.json
operations: [v1_exports_create, v1_exports_list, v1_exports_read, v1_exports_delete, v1_inspections_list]
source: >-
  Grounded in openapi/sewerai-swagger.json, harvested verbatim 2026-08-27 from
  https://api.sewerai.com/swagger.json. Every operationId verified verbatim in that contract. The
  flow mirrors the "Initiate and Fetch an Export" example published at https://docs.sewerai.com/.
  Error and retry rules per errors/sewerai-problem-types.yml and conventions/sewerai-conventions.yml.
---

# Initiate and fetch an inspection export

Exports are how a large volume of inspection data leaves PIONEER as a file rather than as API responses — the path SewerAI's own docs give a named worked example to, and the one to prefer when the alternative would be an unbounded, unpaged list read.

## Auth

- Base URL: `https://api.sewerai.com/v1`
- `Authorization: X-SAI {API_KEY}` or `Authorization: Bearer {JWT}`. See `authentication/sewerai-authentication.yml`.

## Steps

1. **Decide the scope first.** Use `v1_inspections_list` (`GET /v1/inspections/`) with date-window filters to confirm which inspections fall inside the slice you intend to export. Exports are billed against a platform that publishes no quota, so knowing the size before you ask for it is the only cost control available to you.
2. **Start the export** — `v1_exports_create` (`POST /v1/exports/`). This is an asynchronous job: a successful response means accepted, not complete. Keep the returned `sid`.
3. **Poll for completion** — `v1_exports_read` (`GET /v1/exports/{sid}/`). Poll on a slow interval (tens of seconds at minimum). There is no webhook, no callback and no event stream anywhere in the SewerAI surface, so polling is the only completion signal that exists.
4. **Retrieve the output** when the export reports ready, following the location the export record carries.
5. **List what is outstanding** — `v1_exports_list` (`GET /v1/exports/`) to reconcile jobs you started but lost the `sid` for. Do this *before* re-creating an export you think failed.
6. **Clean up** — `v1_exports_delete` (`DELETE /v1/exports/{sid}/`) when the output has been retrieved and stored.

## Rules an agent must follow

- **Reconcile before you retry.** `v1_exports_create` carries no idempotency key. A timed-out create followed by a blind retry produces two jobs over the same data. Always `v1_exports_list` first and match on scope. See `conventions/sewerai-conventions.yml`.
- **There is no cancel.** Nothing in the contract stops a running export. `v1_exports_delete` removes the export record; it is not documented as aborting work in flight. Treat step 2 as a commitment.
- **No webhooks exist.** SewerAI publishes no webhook catalog, no AsyncAPI and no event surface of any kind. Do not wait for a callback that will never arrive.
- **Poll slowly.** No rate limit is published and no `Retry-After` is returned, so there is no signal telling you that you are polling too fast. See `rate-limits/sewerai-rate-limits.yml`.
- **Watch for the 500 on the schema endpoint, not on exports.** `GET /api/schema/` — the contract endpoint the docs and Swagger playground link to — currently returns HTTP 500. That is a documented provider defect (`errors/sewerai-problem-types.yml`); it does not indicate anything about export health. Use `https://api.sewerai.com/swagger.json` if you need the machine-readable contract.
- **Capture `x-amzn-requestid`.** For an asynchronous job with no correlation id in the payload, the AWS request id is the only handle support can trace.
